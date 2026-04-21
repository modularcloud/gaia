# gaia — Git-as-IPC for Agents

## 1. Purpose

`gaia` is a CLI that runs a Claude Code session on a remote machine
(Anthropic's Claude Code cloud VM) and makes the call feel like `claude -p`
to the local shell. The caller needs no `claude` install, no cloud-VM
login, and no messaging integration — everything crosses the boundary
through git.

```
$ gaia -p "refactor the auth module to use JWT and update the tests"
Refactored the auth middleware to use JWT. Added token validation on every
protected route. Updated tests. All 47 tests pass.
$ gaia -c -p "keep the legacy session-cookie path in parallel for now"
Added a session-cookie fallback. JWT is primary. Tests pass.
$ gaia --resume session_01HJK... -p "revert the last change"
```

The syntax mirrors Claude Code's: `-p` for print mode, `-c` / `--continue`
for most-recent continuation, `--resume <session_id>` for named-session
continuation. What runs under the hood is a routine-fired Claude Code
cloud session; what returns is the agent's final message plus whatever
code changes the session merged into the caller's current branch.

## 2. Non-goals (v1)

- **No in-session messaging in the stable surface.** Clarification in v1
  stable happens by re-invoking with `-c`. No Telegram, Slack, or other
  notifier integrations. An *experimental* mode (`--experimental-mode`)
  that keeps a single session alive by blocking the `Stop` hook and
  absorbing user replies through stdin is described in §15 and gated
  behind a warning; it is not part of the stable contract.
- **No true session resume.** Anthropic's `/fire` endpoint does not
  support resuming a prior `session_id` — every fire is a fresh session.
  `gaia -c` / `gaia --resume` is transcript-replay continuation: the
  prior conversation is re-injected into the new session's payload.
- **No protocol-level subcommands.** `gaia run fire`, `gaia session ask`,
  etc. are out of scope for v1; they belong to a future multi-session API.
- **No per-workflow prompt templates.** The prompt is the argument to
  `gaia`. There is no `.gaia-prompts/` directory, no `--workflow` flag.
- **No support for non-GitHub hosts** — required by the underlying
  routines spawner.
- **No Zero Data Retention support** — blocked upstream.
- **No local Claude invocation.** v1 always goes through a cloud routine.

## 3. Architecture

Everything ships as one npm package: `gaia`.

```
npm i -g gaia                                  # on driver machine AND in the VM
```

The package installs three binaries to `$PATH`:

| Binary | Where it runs | Purpose |
|---|---|---|
| `gaia` | Driver (local) | Entry point: `gaia -p`, `gaia -c -p`, `gaia --resume <id> -p`, `gaia init`, `gaia config`, `gaia history`. |
| `gaia-hook-user-prompt-submit` | VM | Claude Code `UserPromptSubmit` hook. Parses `GAIA_*` headers, fetches and checks out the work branch. |
| `gaia-hook-stop` | VM | Claude Code `Stop` hook. Commits any unstaged changes, extracts the final assistant message and conversation transcript, pushes the sentinel commit. |

The VM runs an unmodified `claude` binary. gaia contributes exactly two
hooks to the session side — no custom CLI surface the agent has to
remember.

## 4. Terminology

| Term | Meaning |
|---|---|
| Driver | The local `gaia` process. |
| Session | One remote Claude Code session, identified by the `session_id` Claude Code passes to hooks. This is the id `--resume` targets. |
| Run | One user-facing `gaia` invocation. Identified by `GAIA_RUN_ID` (ULID). Wraps exactly one session in v1. |
| User branch | The git branch the caller was on when `gaia` was invoked. Changes merge back here. |
| Work branch | `gaia/$RUN_ID/work` — descends from the user branch; holds the session's real code commits. |
| State branch | `gaia/$RUN_ID/state` — orphan; holds only `.gaia-state/` files; never merges anywhere. Used as a transport for per-session artifacts. |
| Sentinel | A commit on the state branch with message `GAIA_DONE: <session_id>`. Marks session completion. |
| Spawner | Plugin that dispatches a session to an agent platform. v1 bundles `claude-code-routine`. |

## 5. Prerequisites

**Driver machine.** Node.js 20+, git, push access to the origin repo.

**Anthropic account.** Pro / Max / Team / Enterprise with Claude Code on
the web enabled. ZDR orgs are blocked.

**Per-project, one-time setup** (walked through by `gaia init`):

1. A Claude Code cloud environment with:
   - Network access including `registry.npmjs.org`.
   - Setup script:
     ```bash
     #!/bin/bash
     set -euxo pipefail
     npm i -g gaia@<pinned-version>
     ```
     This script is cached after first run (Claude Code environment
     caching).
2. A Claude Code routine attached to that environment, with the saved
   prompt from §9 and an API trigger.
3. `GAIA_ROUTINE_URL` and `GAIA_ROUTINE_TOKEN` stored locally via
   `gaia config set`.
4. `.claude/settings.json` committed to the repo root, containing the
   hooks block from §8. `gaia init` writes this.
5. `.gitignore` entry for `.gaia/` (local state only). `gaia init`
   appends this.
6. The Claude GitHub App installed on the repo, with the `gaia/*` branch
   namespace permitted for push. `gaia init` prints the exact repo
   setting change.

**Per-invocation.**

- The caller is on a non-detached branch. `gaia` refuses otherwise.
- Working tree is clean. `gaia` refuses otherwise. No auto-stash.
- `git fetch origin <user-branch>` succeeds.

## 6. Usage

### 6.1 One-shot (`-p`)

```
gaia -p "<prompt>"
```

The `-p` flag is required for the one-shot "print" form, mirroring
`claude -p`. Without a mode flag, `gaia` prints usage and exits
non-zero (v1 has no REPL; `-p` signals intent unambiguously and keeps
command shapes aligned with Claude Code's own syntax).

Flow:

1. Generate `GAIA_RUN_ID`. Push `gaia/$RUN_ID/work` seeded from the user
   branch, and `gaia/$RUN_ID/state` as an orphan root.
2. Compose the payload (§7.2) with `"<prompt>"` as `CONTEXT`.
3. Fire the spawner. Receive a `session_id`.
4. Poll `gaia/$RUN_ID/state` for a `GAIA_DONE: <session_id>` commit.
5. Fetch the work and state branches.
6. Copy the prior transcript artifacts to `.gaia/transcripts/<session_id>.md`
   locally.
7. Merge the work branch into the user branch with `--no-ff`. Push if
   the user branch tracks a remote.
8. Print `.gaia-state/<session_id>.md` (the final assistant message)
   to stdout.
9. Append to `.gaia/history.jsonl`.
10. Delete `gaia/$RUN_ID/{work,state}` from origin (unless
    `--keep-branches` or the run failed).

Exit codes:

| Code | Meaning |
|---|---|
| 0 | Session completed; stdout is the final message. |
| 2 | Precondition failed (no `-p`, dirty tree, detached HEAD, missing config, unknown `--resume` id). |
| 3 | Session timed out. Work + state branches preserved. |
| 4 | Merge conflicted. Work branch preserved; user branch untouched. Run id + work branch printed to stderr. |
| 5 | Spawner configuration error. |
| 10 | Spawner returned 401/403 (entitlement / token). |
| 11 | Spawner returned 429 after retries exhausted. |
| 12 | Spawner returned 400 non-retryable (paused routine, text-too-large). |

### 6.2 Continuation

```
gaia -c -p "<prompt>"                              # continue most recent run
gaia --resume <session_id> -p "<prompt>"           # resume a specific session
```

**`-c` / `--continue`** targets the most recent entry in
`.gaia/history.jsonl` for the current repo.

**`--resume <session_id>`** targets a specific prior session by its
Claude Code session id (the same id `--resume` takes in `claude` itself).
History records the session id alongside the run id so lookup is direct.

Both forms:

1. Locate the prior run in `.gaia/history.jsonl`.
2. Read `.gaia/transcripts/<session_id>.md` (the readable transcript
   captured by the `Stop` hook; §8.2).
3. Compose the payload with the transcript injected as a
   `PREVIOUS CONVERSATION:` block *above* the new prompt:
   ```
   --- BEGIN CONTEXT ---
   PREVIOUS CONVERSATION:
   <transcript>
   END PREVIOUS CONVERSATION

   <new prompt>
   --- END CONTEXT ---
   ```
4. Fire a fresh session (new `session_id`, new run) and proceed exactly
   like §6.1. The new history entry records `continued_from: <prior_session_id>`.

If the composed payload exceeds the spawner's 65,536-char cap, the
driver commits the overflow to `.gaia-state/context.md` on the new run's
state branch and references it via `--- CONTEXT FILE ---` (§7.2). Long
chains of continuation will hit this; the mechanism is transparent to
the user.

**Important caveat.** Because `/fire` does not support server-side
session resume, this is *replay* continuation — the new session is a
fresh session that has been shown the prior conversation, not a true
resumption. Tool-call reasoning, thinking tokens, and any internal
state not visible in the transcript are lost. Expect:

- Slightly less efficient than a true `claude -c` (the replay costs
  input tokens every time).
- Occasional small inconsistencies when the model re-reads its own
  prior output and "re-decides" marginal calls.

A future release may use Claude Code's native session resume if the
routines surface exposes it (§14).

### 6.3 Flags

| Flag | Default | Meaning |
|---|---|---|
| `-p` | (required) | Print mode / one-shot. Required for every non-admin invocation. |
| `-c` / `--continue` | off | Prepend the most recent session's transcript. |
| `--resume <session_id>` | off | Prepend a specific session's transcript. Mutually exclusive with `-c`. |
| `--no-merge` | off | Stop after "sentinel received." Leave work branch for manual review. |
| `--keep-branches` | off | Do not delete `gaia/$RUN_ID/*` on origin after merge. |
| `--spawner <name>` | from config | Override the spawner. |
| `--timeout <duration>` | `45m` | Per-session timeout. |
| `--json` | off | Print a single-line JSON result (`{run_id, session_id, summary, merge_sha, continued_from?}`) instead of bare text. |
| `--experimental-mode` | off | **Experimental** (§15). Keeps the session alive across Q/A turns via Stop-hook block-and-inject. Prints a warning on start. Falls back to stable fire-and-forget automatically on any failure. The caller still runs one command that returns one final result — the mode changes the underlying transport, not the command shape. |

### 6.4 Admin commands

```
gaia init                                      # interactive project wizard
gaia config set <key> <value>                  # secret-aware: *_TOKEN → keychain
gaia config get <key>
gaia config unset <key>
gaia history                                   # list recent runs: run_id, session_id, prompt
gaia history show <session_id>                 # print the transcript for a prior session
gaia gc [--max-age 14d] [--dry-run]            # sweep stale gaia/* branches on origin
gaia version
```

`gaia init`:
1. Prompts for routine URL + token; stores them securely.
2. Writes `gaia.config.json` to the repo root.
3. Writes `.claude/settings.json` to the repo root with gaia's hooks
   block. If one exists, merges the `hooks` key non-destructively.
4. Appends `.gaia/` to `.gitignore`.
5. Prints the saved routine prompt (§9) for the user to paste into the
   Claude Code routines UI.
6. Prints the one-line VM setup-script snippet.

There is no separate `gaia vm-install` step in this design. The hooks
travel with the repo; the VM setup script's only job is to install the
gaia binaries via `npm i -g gaia`.

## 7. Protocol

### 7.1 Branch topology

```
origin/<user-branch>          ●──●──●                   (untouched during the run)
                                     \
                                      \                 (driver pushes at start)
                                       \
origin/gaia/$RUN_ID/work      ●──●──●──●                (session commits)
                                       \
                                        \               (driver merges at run end)
                                         ▼
origin/<user-branch>                     ●              (merge commit)

origin/gaia/$RUN_ID/state   ○──○                        (orphan, never merges)
                            │  └── GAIA_DONE: <session_id>
                            │         adds:
                            │           .gaia-state/<session_id>.md
                            │           .gaia-state/<session_id>.transcript.md
                            └───── orphan root
```

Each run in v1 contains exactly one session, so the state branch carries
exactly one sentinel.

### 7.2 Payload

`/fire` text:

```
GAIA_RUN_ID: <ulid>
GAIA_WORK_BRANCH: gaia/<ulid>/work
GAIA_STATE_BRANCH: gaia/<ulid>/state

--- BEGIN CONTEXT ---
[PREVIOUS CONVERSATION:
<transcript text, verbatim from .gaia/transcripts/<prior_session_id>.md>
END PREVIOUS CONVERSATION]

<user's prompt>
--- END CONTEXT ---
```

The `PREVIOUS CONVERSATION:` block is present only for `-c` or
`--resume`.

**Overflow.** If the composed payload exceeds `max_payload_chars`
(65,536 for the Claude routines spawner), the driver writes the
overflow content to `.gaia-state/context.md` on the new run's state
branch *before* firing and replaces it in the payload with:

```
--- CONTEXT FILE ---
.gaia-state/context.md
--- END CONTEXT FILE ---
```

The `UserPromptSubmit` hook (§8.1) detects this marker, fetches the
state branch, and rewrites the prompt to inline the file contents
before Claude sees the final user message. This keeps the saved
routine prompt agnostic to which transport was used.

### 7.3 Sentinel and artifacts

The `Stop` hook emits exactly one commit on the state branch:

```
commit <sha>
    GAIA_DONE: <session_id>

A  .gaia-state/<session_id>.md                 (final assistant message)
A  .gaia-state/<session_id>.transcript.md      (readable conversation transcript)
```

Both files are plain Markdown. The driver reads the first for stdout
display and the second for future continuation.

## 8. Hooks

Installed by `gaia init` to `.claude/settings.json` at the **repo root**
(project-level, not user-level):

```json
{
  "hooks": {
    "UserPromptSubmit": [
      { "command": "gaia-hook-user-prompt-submit" }
    ],
    "Stop": [
      { "command": "gaia-hook-stop" }
    ]
  }
}
```

Project-level install is chosen over `~/.claude/settings.json` because:

- Routine definitions travel with the repo; hooks in the repo ensure
  every cloud VM that fires against this repo has them.
- `.claude/settings.local.json` is explicitly not loaded in routines, so
  it cannot be used.
- Committing hooks to the repo is the only way to guarantee they are
  active on a fresh cloud VM that has not run the setup script yet (or
  where the cache was invalidated).

Both hooks no-op silently when the prompt does not contain
`GAIA_RUN_ID:`, so checking hooks into a shared repo does not affect
non-gaia Claude Code usage on that repo.

### 8.1 `gaia-hook-user-prompt-submit`

Fires on every user prompt submission. Blocks until it returns.

**Input (stdin JSON, v1 known fields).**

```json
{
  "session_id": "session_01HJK...",
  "transcript_path": "/home/user/.claude/projects/.../<uuid>.jsonl",
  "prompt": "GAIA_RUN_ID: 01HXX...\n..."
}
```

**Behavior.**

1. Parse `GAIA_RUN_ID`, `GAIA_WORK_BRANCH`, `GAIA_STATE_BRANCH` from the
   first lines of `prompt`. If absent, exit 0 (non-gaia session — hook
   no-ops).
2. Write `/tmp/gaia/<session_id>.json` with
   `{run_id, work_branch, state_branch}` so the `Stop` hook can
   correlate by `session_id`.
3. Fetch and check out the work branch:
   ```
   git fetch origin "$GAIA_WORK_BRANCH"
   git checkout "$GAIA_WORK_BRANCH"
   ```
4. If the payload contains `--- CONTEXT FILE ---`, fetch the state
   branch and assert the referenced file exists. (Claude will see the
   reference and read the file via its own `Read` tool when it gets to
   the prompt.)
5. Exit 0.

Any git error aborts the hook with non-zero exit, blocking prompt
submission. The session fails visibly in the routine UI; the driver
times out with exit 3 and surfaces the session URL.

### 8.2 `gaia-hook-stop`

Fires when the assistant finishes its final response, before session
teardown.

**Input (stdin JSON, v1 known fields).**

```json
{
  "session_id": "session_01HJK...",
  "transcript_path": "/home/user/.claude/projects/.../<uuid>.jsonl"
}
```

**Behavior.**

1. Look up `/tmp/gaia/<session_id>.json`. If absent, exit 0 (non-gaia
   session).
2. **Final commit sweep.** If `git status --porcelain` is non-empty:
   ```
   git add -A
   git commit -m "gaia: final sweep"
   git push origin "HEAD:$GAIA_WORK_BRANCH"
   ```
3. **Extract the final assistant message** from `transcript_path` by
   reading the JSONL and scanning lines in reverse for the first line
   where `message.role === "assistant"` (or `type === "assistant"`).
   Extract text content:
   - If `message.content` is a string, use it directly.
   - If it is an array, concatenate all `type: "text"` blocks in order,
     ignoring tool-use and thinking blocks.
   - If parsing fails (schema has evolved), fall back to the raw last
     line — still human-readable, and the driver will print it.
4. **Build the readable transcript.** One forward pass through the
   JSONL. For each line where `message.role` is `user` or `assistant`,
   emit a Markdown section:
   ```
   ## USER (2026-04-21T10:31:04Z)
   <text content>

   ## ASSISTANT (2026-04-21T10:31:12Z)
   <text content, tool-use blocks summarized as `[tool: bash — git status]`>
   ```
   Tool-result entries are summarized as `[tool-result: ok]` or
   `[tool-result: error — <first line>]` inline under the preceding
   assistant message. Thinking blocks are dropped.
5. **Write both artifacts** via a temporary worktree on the state
   branch:
   ```
   tmp=$(mktemp -d)
   git fetch origin "$GAIA_STATE_BRANCH"
   git worktree add "$tmp" "origin/$GAIA_STATE_BRANCH"
   mkdir -p "$tmp/.gaia-state"
   printf '%s\n' "$final_message"  > "$tmp/.gaia-state/$session_id.md"
   printf '%s\n' "$transcript_text" > "$tmp/.gaia-state/$session_id.transcript.md"
   git -C "$tmp" add .gaia-state
   git -C "$tmp" commit -m "GAIA_DONE: $session_id"
   git -C "$tmp" push origin "HEAD:$GAIA_STATE_BRANCH"
   git worktree remove "$tmp"
   ```
6. Exit 0.

The sentinel push is the only way the driver learns the session is
done; the hook must not exit before completing step 5.

### 8.3 Schema fragility

The transcript JSONL schema is not fully documented. The hook:

- Treats any line whose `message.role` OR top-level `type` equals
  `"assistant"` as an assistant message.
- Extracts text defensively (string OR array-of-blocks).
- Falls back to raw-last-line output on unrecognized shapes.

`gaia doctor` (future; §14) will run a synthetic session and verify
extraction works on the user's current Claude Code version before the
user relies on the pipeline for real work.

### 8.4 Why hooks and not a session CLI

A prior draft had `gaia session ask` / `gaia session done` subcommands
the agent had to call explicitly. Hooks are strictly better:

- Agent cannot forget to emit the sentinel — `Stop` always fires.
- Agent cannot leave work uncommitted — final sweep guarantees it.
- The saved routine prompt shrinks to something static, so prompt drift
  across projects disappears.
- Sessions can run any prompt, including prompts authored without
  knowing gaia exists.

## 9. Saved routine prompt

Static. Emitted by `gaia init` for the user to paste once into the
Claude Code routines UI:

```
You are a Claude Code session. Your working tree has already been
prepared on a dedicated branch by a gaia hook — you do not need to
checkout, create, or switch branches.

Your task is in the user message below, inside --- BEGIN CONTEXT --- /
--- END CONTEXT --- delimiters.

If the context starts with PREVIOUS CONVERSATION: ... END PREVIOUS
CONVERSATION, treat that as the conversation so far and continue
from it. The new prompt follows the END PREVIOUS CONVERSATION marker.

If the context includes --- CONTEXT FILE --- <path> --- END CONTEXT
FILE ---, read that file and use its contents as the task.

Commit progress with git as you go. Push to the branch name recorded
in the GAIA_WORK_BRANCH variable:

  git push origin "HEAD:$GAIA_WORK_BRANCH"

Do not merge, open pull requests, or comment on GitHub.

Your final assistant message is captured automatically as the result
returned to the caller. Make it the thing the caller needs to read —
a concise summary of what you did, files changed, and any open
questions.
```

This prompt never needs to change per routine or per project.

## 10. Configuration

### 10.1 `gaia.config.json` (committed to repo root)

```json
{
  "spawner": "claude-code-routine",
  "spawner_config": {
    "routine_url_env": "GAIA_ROUTINE_URL",
    "routine_token_env": "GAIA_ROUTINE_TOKEN",
    "beta_header": "experimental-cc-routine-2026-04-01",
    "max_payload_chars": 65536
  },
  "defaults": {
    "session_timeout": "45m",
    "poll_interval": "10s",
    "history_file": ".gaia/history.jsonl",
    "transcripts_dir": ".gaia/transcripts",
    "keep_branches": false
  }
}
```

### 10.2 Secrets

Provided via (in precedence order):

1. `gaia config set <KEY> <VALUE>` — OS keychain where available,
   `.gaia/secrets.json` (0600, gitignored) fallback.
2. Ambient environment variable.
3. `.env` at repo root (gitignored; parsed by gaia on startup).

v1 secrets: `GAIA_ROUTINE_URL`, `GAIA_ROUTINE_TOKEN`. (The URL is not
strictly secret but stored alongside the token for simplicity.)

### 10.3 Local state

```
.gaia/                                         (gitignored)
  history.jsonl                                run summaries for -c/--resume
  transcripts/
    <session_id>.md                            readable conversation transcripts
  secrets.json                                 fallback store (0600)
  runs/
    <run_id>/
      meta.json                                metadata for gc / recovery
      fire.json                                spawner response
```

`history.jsonl` — one JSON object per line:

```json
{
  "run_id": "01HXX...",
  "session_id": "session_01HJK...",
  "timestamp": "2026-04-21T10:30:00Z",
  "user_branch": "feature/auth",
  "prompt": "refactor the auth module...",
  "summary": "Refactored the auth middleware...",
  "merge_sha": "abc1234",
  "continued_from": "session_01HAA..."
}
```

`continued_from` is the `session_id` of the prior run, allowing
chain-walking for long `-c` chains.

### 10.4 `.gitignore`

`gaia init` appends:

```
.gaia/
```

Only this. `.gaia-state/` lives on orphan state branches exclusively and
never touches reviewed branches.

## 11. Spawner plugin layer

### 11.1 Interface

```ts
interface Spawner {
  name: string;
  fire(payload: { text: string }): Promise<{
    sessionId: string;
    sessionUrl?: string;
  }>;
  maxPayloadChars: number;
  defaultTimeout: string;
}
```

Kept abstract for future spawners (§14).

### 11.2 `claude-code-routine` (v1)

- `POST https://api.anthropic.com/v1/claude_code/routines/<trig>/fire`
- Headers: `Authorization: Bearer <token>`,
  `anthropic-beta: <beta_header>`, `anthropic-version: 2023-06-01`.
- Honors 429 `Retry-After` verbatim.
- Exponential backoff on 503 up to 3 attempts.
- Non-recoverable on 401/403/400-paused/400-text-too-large.
- `maxPayloadChars: 65536`, `defaultTimeout: "45m"`.

## 12. Failure modes

| Failure | Detection | Policy |
|---|---|---|
| `-p` missing | Arg parsing | Exit 2 with usage. |
| Spawner HTTP error | `fire()` rejects | Retry per §11.2. Exhausted → exit 10–12. |
| No sentinel by timeout | Poll loop deadline | Exit 3. Work + state branches preserved. Session URL printed. User decides whether to re-run with `-c`. |
| Merge conflict | `git merge` fails | Abort merge. Exit 4. Work branch preserved; user branch untouched. |
| Dirty working tree at start | `git status --porcelain` non-empty | Exit 2. |
| Detached HEAD at start | No symbolic ref | Exit 2. |
| `UserPromptSubmit` hook fails | Hook non-zero | Session blocked by Claude Code. Driver hits no-sentinel timeout; session URL surfaced. |
| `Stop` hook fails before sentinel push | No sentinel on state branch | Driver times out. The final-sweep commit (if it ran) is preserved on the work branch for manual recovery. |
| Transcript parse fails (final message) | Unknown JSONL shape | Fall back to raw last line; run still succeeds; stdout is whatever the fallback produced. |
| Transcript parse fails (readable transcript) | Same | `.gaia-state/<session_id>.transcript.md` contains the raw JSONL; continuation still works (Claude can read it) but with more token cost. |
| Duplicate sentinel | Two `GAIA_DONE:` commits for the same session_id | Earliest wins. Warning logged. |
| `-c` with empty history | `.gaia/history.jsonl` missing or empty | Exit 2 with a message to run without `-c`. |
| `--resume <session_id>` unknown | No matching entry | Exit 2. |
| Continuation transcript missing | `.gaia/transcripts/<session_id>.md` absent | Exit 2 with a message. Can happen if transcripts directory was deleted manually. |

### 12.1 Cleanup

Default: delete `gaia/$RUN_ID/{work,state}` from origin immediately
after a successful merge. Transcripts have already been copied locally
to `.gaia/transcripts/` at step 6 of §6.1, so the state branch is
disposable at that point.

Exceptions (branches preserved on origin):

- `--keep-branches`.
- Any non-zero exit.

`gaia gc` sweeps `gaia/*` branches on origin whose reflog timestamp is
older than `--max-age` (default 14d).

## 13. Concurrency

Parallel runs are supported. Two or more `gaia` invocations can be in
flight simultaneously against the same repo, the same user branch,
and the same account.

**What makes this safe:**

- `GAIA_RUN_ID` is a ULID — globally unique across repos, users, and
  time. Every invocation gets fresh `gaia/<run_id>/{work,state}`
  branches, a fresh `.gaia/runs/<run_id>/` local dir, and a fresh
  `session_id` from the spawner. Nothing is shared across concurrent
  runs at the protocol level.
- `.gaia/history.jsonl` is written with `O_APPEND` and each entry is
  well under `PIPE_BUF` (typically 4096 bytes), so concurrent appends
  do not interleave-corrupt. Order in the file reflects completion
  order.
- The `Stop` hook correlates by `session_id` via `/tmp/gaia/<session_id>.json`,
  which the `UserPromptSubmit` hook writes per-session. Different
  sessions on the same VM (unusual, but possible) do not collide.

**What can still conflict:**

- **Merge-back at run-end.** Two parallel runs that both modify the
  same files on the user branch will conflict when the second merges.
  Handled per §12 exit 4: abort the merge, preserve the work branch,
  surface the run id, let the user resolve. This is the same behavior
  as any concurrent git work and is not specific to gaia.
- **Spawner quota.** The Claude Code routines daily cap is per-account.
  N parallel invocations draw N times the budget. 429 from the spawner
  is terminal for the current window per §11.2.
- **Same-terminal `--experimental-mode`.** Two `--experimental-mode`
  invocations from the same terminal conflict on stdin — only one
  process can read input. Run them in separate terminals, or serialize.
  Stable (non-experimental) runs have no stdin dependency and can
  share a terminal via backgrounding without issue.

**What happens when `-c` is invoked during in-flight runs:**

`-c` picks the most recent *completed* run (by `timestamp` in
`.gaia/history.jsonl`). In-flight runs are invisible to `-c` because
they have no history entry yet. If that's ambiguous for the caller
(e.g., two runs finished in the same second), use
`--resume <session_id>` for determinism.

A `.gaia/lock` advisory lock remains future work (§14) for callers
who want to prevent concurrent merges into a single user branch by
policy rather than by convention.

## 14. Open questions and future work

- **`gaia doctor`.** Runs a synthetic session end-to-end to validate:
  spawner config, GitHub-App push permissions, transcript JSONL schema,
  hook installation. Should be run after `gaia init` and on any Claude
  Code version change.
- **True session resume.** If Anthropic exposes a way to continue a
  prior `session_id` via `/fire` (or a successor endpoint), `-c` and
  `--resume` can drop the transcript replay in favor of server-side
  resumption. The user-facing flags do not change.
- **Transcript schema stability.** The Claude Code transcript JSONL
  schema is not a documented contract. gaia's parser is defensive and
  falls back gracefully, but a schema change could degrade the readable
  transcript format. Monitor via `gaia doctor`.
- **Auto-commit hook.** A `PostToolUse` hook on `Edit`/`Write` would
  strengthen crash-safety at the cost of a noisier git log. Deferred.
- **Protocol subcommands.** `gaia run fire`, `gaia run wait`,
  `gaia run finish` — the multi-session interface that tools like loopx
  will need. Layers on top of the same protocol described here.
- **Additional spawners.** `local-claude` (shells out to `claude -p`
  locally — no cloud VM required), `codex`, `github-action`.
- **Concurrent-branch lock.** `.gaia/lock` or an origin-side advisory
  lock on the user branch.
- **Interactive picker for `--resume`.** `gaia --resume -p "<prompt>"`
  without a session id could show the recent history and prompt for
  selection, matching `claude --resume` without an argument.
- **Cross-machine history sync.** v1 history + transcripts are
  machine-local. A future mode could commit them to a dedicated
  `gaia/history` branch so continuation works across machines.
- **Routine provisioning API.** If Anthropic exposes a routine-creation
  endpoint, `gaia init` can stop printing "paste this into the UI."
- **Graduating `--experimental-mode` to stable** (§15). Once VM
  idle-lifetime is characterized empirically and the `Stop`-hook JSON
  semantics are confirmed stable across Claude Code versions, the flag
  drops the "experimental" warning and/or the mode becomes the default
  transport for `-c` / `--resume`.

## 15. Experimental mode: `--experimental-mode`

**Status: experimental.** Ships in v1.x behind `--experimental-mode`.
Prints a warning on start. No stability guarantees. Automatically falls
back to the stable fire-and-forget path on any failure (§15.3). Intended
to graduate once the constraints in §15.4 are characterized.

### 15.1 Motivation

Stable `-c` / `--resume` continuation is correct but costly for
conversational workflows:

- Every turn is a cold VM (git clone, setup script, Claude cold-boot).
- Transcript replay makes input-token cost grow O(N²) across a long
  conversation.
- Internal session state (tool-use plans, thinking, short-term context
  not expressed in the transcript) is discarded between turns.

Interactive mode keeps a single session alive across multiple Q/A
exchanges by exploiting Claude Code's `Stop` hook's ability to return
`{"decision": "block", "reason": "<text>"}` — which prevents session
termination and injects `reason` as a system reminder Claude sees
before continuing. Anthropic's `/fire` endpoint offers no
send-message-to-session API publicly (§14, confirmed in docs research);
this hook-level mechanism is the only way to achieve in-session Q/A
without a notifier.

### 15.2 Mechanism

No messaging layer. No notifier plugin. The "messaging channel" is the
driver's stdin on one side and the `Stop` hook on the other, connected
through commits on the run's orphan state branch.

Flow:

1. Driver fires the routine with `GAIA_EXPERIMENTAL_MODE: 1` appended to the
   payload headers. Drops the state-branch poll interval to 2s for
   this run.
2. VM: Claude processes the initial task, produces a response
   (possibly ending in a question). `Stop` hook fires.
3. Hook detects `GAIA_EXPERIMENTAL_MODE=1`. Instead of the standard
   `GAIA_DONE:` sentinel, it pushes a commit to the state branch:
   ```
   GAIA_PENDING: <session_id> #<turn_n>

   A  .gaia-state/<session_id>/pending_<turn_n>.md    (Claude's latest response)
   ```
4. Hook starts polling (2s) `origin/<state_branch>` for a matching
   `GAIA_INPUT: <session_id> #<turn_n>` commit. Blocking-wait up to
   9 minutes (safely under the documented 10-min hook default).
5. Driver sees `GAIA_PENDING:`. Reads `pending_<turn_n>.md`. Prints
   Claude's response to stdout. Prompts the user:
   ```
   reply (ctrl-D to finish):
   ```
6. User types a reply on stdin (supports multi-line input; ends on
   EOF or blank-line + `.` sentinel — configurable).
7. Driver writes `.gaia-state/<session_id>/input_<turn_n>.md` with
   the reply and pushes `GAIA_INPUT: <session_id> #<turn_n>` to the
   state branch.
8. Hook sees the input commit. Returns:
   ```json
   {
     "decision": "block",
     "reason": "<user's reply, verbatim>",
     "additionalContext": "The user replied via gaia experimental mode. Treat this as their next user message and continue."
   }
   ```
   Claude resumes with the reply as system-reminder context.
9. Loop repeats (step 2 for turn `n+1`) until one of:
   - Claude produces a response the user accepts by entering EOF
     (ctrl-D) — driver signals "end" to the hook by pushing an empty
     `GAIA_INPUT:` commit with body `__GAIA_END__`; hook exits 0 and
     lets the session finalize normally.
   - A fallback trigger fires (§15.3).
10. Clean end: hook exits 0. Session terminates. Regular `Stop`-path
    runs — `GAIA_DONE: <session_id>` sentinel is pushed with the
    standard artifacts (`.gaia-state/<session_id>.md`,
    `.gaia-state/<session_id>.transcript.md`). The transcript includes
    every turn, not just the initial exchange.

### 15.3 Fallback

Interactive mode falls back to the standard stable path on any of:

| Trigger | What happens | Driver behavior |
|---|---|---|
| Hook timeout (9 min without `GAIA_INPUT`) | Hook exits 0. Session finalizes. Standard `GAIA_DONE:` sentinel emitted. | Driver reads the sentinel and prints the final assistant message. Exits 0. User may continue with `gaia -c -p "..."` (transcript replay). |
| Hook exits non-zero (git error, JSON schema drift) | Session terminates abnormally. No sentinel pushed. | Driver times out. Exits 3 with the session URL. User recovers with `gaia -c` against the preserved branches. |
| VM wall-clock timeout (undocumented Anthropic cap) | Session dies server-side. | Driver detects state-branch inactivity, exits 3. Same recovery as above. |
| Driver killed (user ctrl-C or SIGTERM) | Driver cleans up its local state and exits. Hook eventually times out on its next poll. | User runs `gaia -c` when ready; the final-sweep in the hook preserved the work-branch commits, and the pending response is visible in the eventual sentinel. |
| Explicit user abort (`:abort` at the reply prompt) | Driver pushes `__GAIA_ABORT__` as the input commit. Hook treats as end-of-session, exits 0. | Session finalizes normally. Same as EOF. |

In every fallback case:

- **No work is lost.** The hook's pre-finalize sweep (§8.2 step 2)
  commits and pushes any unstaged changes before the session ends in
  every clean-exit path. Unclean exits preserve the work branch on
  origin for `gaia -c` recovery.
- **No protocol divergence.** Fallback always ends with a standard
  `GAIA_DONE:` sentinel + the same two artifacts a v1 session would
  emit. Downstream tooling (merge, history.jsonl append, stdout) is
  identical.
- **Continuation still available.** The user's next `gaia -c -p "..."`
  replays the full transcript (including interactive turns) in a fresh
  session — just like continuing from any other prior run.

This is the "backup mechanism" promised by the mode's design: a failure
of `--experimental-mode` degrades cleanly to the stable path, not to a
broken state.

### 15.4 Constraints and known issues

- **Fidelity gap.** User replies arrive as plain-text `additionalContext`
  system reminders, not as native user turns. Claude handles system
  reminders well in practice, but a native `/fire` with a new user
  message would be higher fidelity. That API is not exposed publicly
  for routines.
- **Per-turn timeout.** Bounded by the `Stop`-hook default (10 min).
  The hook uses 9 min to leave a safety margin. `timeout_seconds` in
  `.claude/settings.json` is configurable but has no documented upper
  bound; raising it risks hitting the (also undocumented) VM
  wall-clock cap.
- **VM wall-clock ceiling unknown.** Anthropic's docs state idle
  compute is not billed but do not specify a max session lifetime.
  `gaia doctor` should measure this empirically before users rely on
  long `--experimental-mode` sessions in production.
- **Single input channel.** The driver's stdin is the only reply
  surface. No multi-device input; no replying from a phone. If that's
  wanted, re-add a notifier (§14).
- **Concurrency.** Covered in §13. Parallel runs are supported across
  terminals — each has a unique run_id, its own state branch, and its
  own session_id, so commits do not collide. The only constraint
  specific to this mode is stdin: two `--experimental-mode`
  invocations from the same terminal cannot both read input at once.
  Run them in separate terminals.
- **Transcript size.** Long interactive sessions accumulate large
  transcripts. The final `transcript.md` artifact may be multi-MB; the
  `Stop`-hook writer streams to disk to avoid memory pressure.
- **Subagents unchanged.** The `Task`-tool subagent flow still fires
  `SubagentStop`, not `Stop`, so subagent execution is unaffected by
  `--experimental-mode`. Only the main-session `Stop` is intercepted.

### 15.5 UX sketch

```
$ gaia --experimental-mode -p "refactor the auth module to use JWT"
[gaia] --experimental-mode is experimental. Falls back to -c on timeout or failure.
[gaia] run 01HXX0AB..., session session_01HJK..., polling at 2s

────────────────────────────────────────────────────────────────────
AGENT: I can start, but the current config doesn't specify signing
       method. HS256 (symmetric) or RS256 (asymmetric)?
────────────────────────────────────────────────────────────────────
reply (ctrl-D to finish):
HS256 with a 512-bit secret, rotated monthly
^D
[gaia] sending input (turn 1)...

────────────────────────────────────────────────────────────────────
AGENT: Got it. Implementing now — committing as I go.
────────────────────────────────────────────────────────────────────
(no reply prompt — this turn ended with no question)
[gaia] session continuing automatically...

────────────────────────────────────────────────────────────────────
AGENT: Done. Refactored middleware.ts, added HS256 signing, wired up
       rotate_secret.ts. 47 tests pass. Anything else?
────────────────────────────────────────────────────────────────────
reply (ctrl-D to finish):
^D
[gaia] finalizing...
[gaia] merged 4 commits into feature/auth
Refactored the auth middleware to use JWT (HS256, 512-bit secret,
monthly rotation). 47 tests pass.
$
```

Note "no reply prompt" in the second turn: the driver uses a simple
heuristic (assistant response does not end with a question mark /
does not contain typical question phrasings) OR a quick-timeout idle
pass (2 seconds of waiting on stdin with no input) to auto-continue
without prompting. If the heuristic is wrong, the user can
`gaia --experimental-mode -p` again with the same `--resume` id to
continue the conversation via the stable path.

### 15.6 Protocol additions

Interactive mode extends the §7 protocol with two new commit-message
shapes on the state branch. Neither appears in a non-interactive run.

| Message | Producer | Contents |
|---|---|---|
| `GAIA_PENDING: <session_id> #<n>` | VM `Stop` hook | `.gaia-state/<session_id>/pending_<n>.md` — Claude's response for turn `n`. |
| `GAIA_INPUT: <session_id> #<n>` | Driver | `.gaia-state/<session_id>/input_<n>.md` — user's reply for turn `n`. Body may be `__GAIA_END__` or `__GAIA_ABORT__` as control sentinels. |

The standard `GAIA_DONE: <session_id>` sentinel is still emitted
exactly once, at the end of a clean-finalize path, regardless of how
many pending/input turns preceded it. The final `transcript.md`
artifact is assembled from the full JSONL transcript — including the
system-reminder-injected turns — by the same reverse-scan + forward-pass
logic in §8.2, with a small extension: system reminders matching
gaia's injection pattern are rendered back as "USER (via --experimental-mode):" in the
Markdown transcript so the continuation path (§6.2) sees them as real
user turns, not as reminders.

### 15.7 Doctor coverage

`gaia doctor` (§14) gains three interactive-mode checks:

1. Fire a session with `GAIA_EXPERIMENTAL_MODE=1` and a known-trivial
   prompt. Confirm the `GAIA_PENDING:` commit appears within 30s.
2. Push a `GAIA_INPUT:` reply. Confirm the hook injects it and Claude
   produces a follow-up `GAIA_PENDING:` commit within 60s.
3. End the session with `__GAIA_END__`. Confirm a standard
   `GAIA_DONE:` sentinel is emitted within 30s and the final
   transcript includes the injected turn rendered as "USER (via --experimental-mode):".

Failure on any of the three → `--experimental-mode` is not usable on
this Claude Code version; `gaia doctor` reports which step failed and
whether it is likely a hook-timeout, JSON-semantics, or VM-lifetime
issue.
