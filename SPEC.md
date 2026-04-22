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
  notifier integrations. A *lab-quality* mode (`--experimental-mode`)
  that keeps a single session alive by blocking the `Stop` hook and
  absorbing user replies through stdin is described in §15 and gated
  behind a warning; it is explicitly not part of the stable contract.
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
- **No Zero Data Retention support** — blocked upstream. Claude Code on
  the web / cloud sessions are unavailable for ZDR-enabled orgs.
- **No local Claude invocation.** v1 always goes through a cloud routine.

## 3. Architecture

Everything ships as one npm package. The unscoped name `gaia` appears
to be taken on the public npm registry; v1 publishes under a scoped
name (placeholder: `@your-scope/gaia`, to be finalized before release)
that still installs a `gaia` binary:

```
npm i -g @your-scope/gaia                      # on driver machine AND in the VM
```

The package installs three binaries to `$PATH`:

| Binary | Where it runs | Purpose |
|---|---|---|
| `gaia` | Driver (local) | Entry point: `gaia -p`, `gaia -c -p`, `gaia --resume <id> -p`, `gaia init`, `gaia config`, `gaia history`, `gaia recover`, `gaia doctor`. |
| `gaia-hook-user-prompt-submit` | VM | Handler for Claude Code's `UserPromptSubmit` hook. Parses `GAIA_*` headers, validates the state-branch manifest, checks out the work branch, materializes overflow context. |
| `gaia-hook-stop` | VM | Handler for Claude Code's `Stop` and `StopFailure` hooks (same binary; branches on `hook_event_name`). Commits and pushes any unstaged work, extracts the final assistant message and conversation transcript, pushes the sentinel commit. |

The VM runs an unmodified `claude` binary. gaia contributes exactly two
hook handlers — no custom CLI surface the agent has to remember.

**Shim layer.** `.claude/settings.json` never references the global
`gaia-hook-*` binaries directly. Instead, `gaia init` commits small
shim scripts under `.claude/hooks/` that peek at hook stdin, no-op
silently when the session is not a gaia session, and delegate to the
global binary only when it is. This keeps the settings file safe on a
teammate's machine that does not have gaia installed (the shim is
always present in the repo clone; the global binary is only required
for gaia sessions). See §8 for the full shim / hook layout.

## 4. Terminology

| Term | Meaning |
|---|---|
| Driver | The local `gaia` process. |
| Session | One remote Claude Code session, identified by the `session_id` Claude Code passes to hooks. This is the id `--resume` targets. |
| Run | One user-facing `gaia` invocation. Identified by `GAIA_RUN_ID` (ULID). Wraps exactly one session in v1. |
| User branch | The git branch the caller was on when `gaia` was invoked. Changes merge back here. |
| Work branch | `claude/gaia/$RUN_ID/work` — descends from the user branch; holds the session's real code commits. |
| State branch | `claude/gaia/$RUN_ID/state` — orphan; holds only `.gaia-state/` files; never merges anywhere. Used as a transport for per-run artifacts and the run manifest. |
| Manifest | A `.gaia-state/manifest.json` file committed to the state branch by the driver *before* firing. Contains a per-run random nonce. The hook validates it before acting on any prompt-supplied branch names. |
| Sentinel | A commit on the state branch with message `GAIA_DONE: <session_id>` (success) or `GAIA_FAILED: <session_id>` (routine / API error). Also `GAIA_PENDING` / `GAIA_INPUT` in lab-quality mode (§15). |
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
     npm i -g @your-scope/gaia@<pinned-version>
     ```
     This script is cached after first run (Claude Code environment
     caching).
2. A Claude Code routine attached to that environment, with the saved
   prompt from §9 and an API trigger.
3. `GAIA_ROUTINE_URL` and `GAIA_ROUTINE_TOKEN` stored locally via
   `gaia config set`.
4. `.claude/settings.json` and `.claude/hooks/*.sh` committed to the
   repo root and **merged to the default branch** (see note below).
   `gaia init` writes these files and, if the current branch is not the
   default branch, either opens a PR on the user's behalf (if `gh` is
   available) or prints the exact git commands to run.
5. `.gitignore` entry for `.gaia/` (local state only). `gaia init`
   appends this.
6. The Claude GitHub App installed on the repo. Routines can push to
   any `claude/*`-prefixed branch by default; gaia's `claude/gaia/*`
   namespace stays inside that boundary, so no *Allow unrestricted
   branch pushes* setting is required. If the repo has narrower
   restrictions, `gaia init` prints the exact Claude-for-GitHub setting
   to adjust.

**Why the default branch matters.** Routines clone a repo at its
default branch and run `UserPromptSubmit` *before* any checkout. If the
hook files live only on a feature branch, the cloud session will not
see them and the entire protocol silently degrades. `gaia init` refuses
to claim it has completed setup until the hook files are present on
the default branch.

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

1. Generate `GAIA_RUN_ID` (ULID) and `GAIA_NONCE` (24 random bytes,
   base64).
2. Push `claude/gaia/$RUN_ID/work` seeded from the user branch, and
   `claude/gaia/$RUN_ID/state` as an orphan root containing
   `.gaia-state/manifest.json` (§7.4).
3. **Write a pending run record.** Create
   `.gaia/runs/<run_id>/meta.json` with `status: "pending"`, and append
   a matching line to `.gaia/history.jsonl`. This makes the run visible
   to `gaia history`, `gaia -c`, and `gaia recover` even if the driver
   crashes or times out.
4. Compose the payload (§7.2) with `"<prompt>"` as `CONTEXT`.
5. Fire the spawner. Receive `claude_code_session_id` and
   `claude_code_session_url`. Update `meta.json` with both.
6. Poll `claude/gaia/$RUN_ID/state` for a sentinel commit matching the
   `session_id`:
   - `GAIA_DONE: <session_id>` → success, proceed to step 7.
   - `GAIA_FAILED: <session_id>` → routine / API error (§7.3). Finalize
     history with `status: "failed"`; print the error body; exit 13.
7. Fetch the work and state branches.
8. Copy the transcript artifacts to
   `.gaia/transcripts/<session_id>.md` locally.
9. **Merge back transactionally** (§6.1.1).
10. Print `.gaia-state/<session_id>.md` (the final assistant message)
    to stdout.
11. Finalize the history entry: update `status` to `success`, write
    `summary`, `merge_sha`.
12. Delete `claude/gaia/$RUN_ID/{work,state}` from origin (unless
    `--keep-branches` or any non-zero exit — see §12.1).

#### 6.1.1 Transactional merge-back

After a `GAIA_DONE:` sentinel, the driver performs the merge as a
strict sequence and never force-pushes. Each step's failure surfaces
as a distinct, actionable exit code:

1. Verify HEAD is still the caller's original user branch. If the user
   switched branches mid-run, `git checkout` it back and re-verify the
   working tree is clean.
2. If the working tree is dirty, abort with exit 2 — gaia will not
   overwrite manual edits. Run id and work branch are printed to
   stderr.
3. `git fetch origin <user-branch>` to pick up any upstream advance.
4. If the user branch tracks upstream, fast-forward local to
   `origin/<user-branch>`. If fast-forward is impossible (the user
   committed locally during the run), exit 2 with a recovery message.
5. `git merge --no-ff claude/gaia/$RUN_ID/work`. Conflict →
   `git merge --abort`, exit 4; work branch preserved on origin.
6. If the user branch tracks upstream, `git push origin <user-branch>`.
   Never force. Rejection → exit 15 (merge local, push upstream) with
   instructions: investigate the upstream advance, then `git push`
   manually when safe. The local user branch already has the merge
   commit.
7. On full success, delete the remote work / state branches and
   finalize history.

Exit codes:

| Code | Meaning |
|---|---|
| 0 | Session completed; stdout is the final message. |
| 2 | Precondition failed (no `-p`, dirty tree, detached HEAD, missing config, unknown `--resume` id, user branch diverged non-fast-forward at merge). |
| 3 | Session timed out. Work + state branches preserved. `gaia recover <run_id>` offered. |
| 4 | Merge conflicted. Work branch preserved; user branch untouched. Recoverable via `gaia recover`. |
| 5 | Spawner configuration error. |
| 10 | Spawner returned 401/403 (entitlement / token). |
| 11 | Spawner returned 429 after retries exhausted. |
| 12 | Spawner returned 400 non-retryable (paused routine, text-too-large). |
| 13 | `GAIA_FAILED:` sentinel received (session aborted by routine / API error; details printed to stderr). |
| 15 | Merge succeeded locally but push to `origin/<user-branch>` was rejected. Local user branch has the merge commit; `git push` when ready. |

### 6.2 Continuation

```
gaia -c -p "<prompt>"                              # continue most recent successful run on current branch
gaia --resume <session_id> -p "<prompt>"           # resume a specific session (any branch)
```

**`-c` / `--continue`** targets the most recent entry in
`.gaia/history.jsonl` whose `user_branch` equals the current branch
**and whose `status` is `success`**. Scoping to the current branch
avoids surprising cross-branch continuation; scoping to successful
runs avoids silently replaying a broken transcript. If there is no
matching entry but there are failed / timed-out runs on the branch,
`gaia -c` exits 2 and points at `gaia recover`.

**`--resume <session_id>`** targets a specific prior session by its
Claude Code session id (the same id `--resume` takes in `claude`
itself). Works across branches; the caller's current branch becomes
the new user branch for the continuation run. History records the
session id alongside the run id so lookup is direct.

Both forms:

1. Locate the prior run in `.gaia/runs/*/meta.json` or
   `.gaia/history.jsonl`.
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
   like §6.1. The new history entry records
   `continued_from: <prior_session_id>`.

If the composed payload exceeds the spawner's 65,536-char cap, the
driver commits the overflow to `.gaia-state/context.md` on the new
run's state branch and references it via `--- CONTEXT FILE ---`
(§7.2). Long chains of continuation will hit this; the mechanism is
transparent to the user.

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
| `-c` / `--continue` | off | Prepend the most recent successful session's transcript. Scoped to current branch. |
| `--resume <session_id>` | off | Prepend a specific session's transcript. Mutually exclusive with `-c`. Works across branches. |
| `--no-merge` | off | Stop after "sentinel received." Leave work branch for manual review. |
| `--keep-branches` | off | Do not delete `claude/gaia/$RUN_ID/*` on origin after merge. |
| `--spawner <name>` | from config | Override the spawner. |
| `--timeout <duration>` | `45m` | Per-session timeout. |
| `--json` | off | Print a single-line JSON result (`{run_id, session_id, status, summary, merge_sha, continued_from?}`) instead of bare text. |
| `--experimental-mode` | off | **Lab-quality** (§15). Keeps the session alive across Q/A turns via Stop-hook block-and-inject. Prints a warning on start. Falls back to stable fire-and-forget automatically on any failure. Not part of the v1 stable contract. |

### 6.4 Admin commands

```
gaia init                                      # interactive project wizard
gaia config set <key> <value>                  # secret-aware: *_TOKEN → keychain
gaia config get <key>
gaia config unset <key>
gaia history                                   # list recent runs: run_id, session_id, status, prompt
gaia history show <session_id>                 # print the transcript for a prior session
gaia recover <run_id|session_id>               # finish a stuck run: fetch preserved state/work, surface the sentinel (if any), offer merge or manual review
gaia gc [--max-age 14d] [--dry-run]            # sweep stale claude/gaia/* branches on origin
gaia doctor                                    # run a synthetic end-to-end session to validate spawner, hooks, transcript schema, branch permissions
gaia version
```

`gaia init`:
1. Prompts for routine URL + token; stores them securely.
2. Writes `gaia.config.json` to the repo root.
3. Writes `.claude/settings.json` and `.claude/hooks/*.sh` to the repo
   root (§8). If `.claude/settings.json` exists, merges the `hooks`
   key non-destructively.
4. Appends `.gaia/` to `.gitignore`.
5. Prints the saved routine prompt (§9) for the user to paste into the
   Claude Code routines UI.
6. Prints the one-line VM setup-script snippet.
7. Verifies the hook files exist on the default branch. If they do
   not, offers to open a PR (via `gh`) or prints the git commands.
   Does not claim success until this step passes.
8. Offers to run `gaia doctor` before exit.

There is no separate `gaia vm-install` step in this design. The hooks
travel with the repo; the VM setup script's only job is to install
the gaia binaries via `npm i -g @your-scope/gaia`.

`gaia doctor` is part of v1, not future work. gaia's runtime
correctness depends on several moving Claude Code surfaces (routine
fire, hook schema, transcript JSONL format, cloud-branch permissions,
default hook timeouts, `Stop`-hook decision semantics); `gaia doctor`
exercises each before a user relies on the pipeline for real work.
See §8.4 for the doctor checklist.

`gaia recover <id>` resolves a stuck run without needing the original
invocation's process to still be alive:
- Looks up `.gaia/runs/<run_id>/meta.json` (or finds by session_id).
- Fetches the preserved `claude/gaia/$RUN_ID/{work,state}` branches.
- If a sentinel exists on the state branch, completes the merge
  transactionally (§6.1.1), finalizes history, and deletes remote
  branches.
- If not, prints the `session_url` and work-branch name for manual
  review, and offers to re-run `-c` via transcript replay against
  whatever transcript was captured pre-failure.

## 7. Protocol

### 7.1 Branch topology

```
origin/<user-branch>                   ●──●──●                     (untouched during the run)
                                              \
                                               \                   (driver pushes at start)
                                                \
origin/claude/gaia/$RUN_ID/work        ●──●──●──●                  (session commits)
                                                \
                                                 \                 (driver merges at run end)
                                                  ▼
origin/<user-branch>                              ●                (merge commit)

origin/claude/gaia/$RUN_ID/state    ○──○──○                        (orphan, never merges)
                                    │  │  └── GAIA_DONE:   <session_id>    (or GAIA_FAILED:)
                                    │  │         adds:
                                    │  │           .gaia-state/<session_id>.md
                                    │  │           .gaia-state/<session_id>.transcript.md
                                    │  └────── manifest commit
                                    │            adds:
                                    │              .gaia-state/manifest.json
                                    └───── orphan root
```

The `claude/gaia/*` prefix places every gaia branch inside Claude
Code's default allowlist for routine pushes — no *Allow unrestricted
branch pushes* toggle is required on the repo. Each run in v1 contains
exactly one session, so the state branch carries exactly one
completion sentinel (`GAIA_DONE:` or `GAIA_FAILED:`).

### 7.2 Payload

`/fire` text:

```
GAIA_RUN_ID: <ulid>
GAIA_WORK_BRANCH: claude/gaia/<ulid>/work
GAIA_STATE_BRANCH: claude/gaia/<ulid>/state
GAIA_NONCE: <base64-24>

--- BEGIN CONTEXT ---
[PREVIOUS CONVERSATION:
<transcript text, verbatim from .gaia/transcripts/<prior_session_id>.md>
END PREVIOUS CONVERSATION]

<user's prompt>
--- END CONTEXT ---
```

The `PREVIOUS CONVERSATION:` block is present only for `-c` or
`--resume`.

**Nonce and manifest validation.** The driver generates `GAIA_NONCE`
per run and commits it into `.gaia-state/manifest.json` (§7.4) *before*
`/fire`. The `UserPromptSubmit` hook fetches the state branch, reads
the manifest, and refuses to proceed if the nonce in the prompt does
not match the manifest, the declared branch names disagree with the
manifest, or the manifest is missing. This protects the protocol
against forged prompts that try to imitate gaia headers.

**Branch-name pattern.** Even with a verified manifest, the hook also
checks that every branch name matches
`^claude/gaia/[0-9A-HJKMNP-TV-Z]{26}/(work|state)$` (ULID + fixed
suffix). Mismatches abort the hook.

**Overflow.** If the composed payload exceeds `max_payload_chars`
(65,536 for the Claude routines spawner), the driver writes the
overflow content to `.gaia-state/context.md` on the new run's state
branch *before* firing and replaces it in the payload with:

```
--- CONTEXT FILE ---
.gaia/runs/<run_id>/context.md
--- END CONTEXT FILE ---
```

The `UserPromptSubmit` hook (§8.1) detects this marker, fetches the
state branch, copies `.gaia-state/context.md` into the **work-tree**
at `.gaia/runs/<run_id>/context.md` (gitignored, so the Stop hook's
`git add -A` will not commit it), and returns a `hookSpecificOutput`
that rewrites the submitted prompt — replacing the marker with inline
text instructing Claude to `Read .gaia/runs/<run_id>/context.md`
before acting. This is the mechanism Claude's model actually sees; the
saved routine prompt (§9) only has to know that a context-file
reference means "read that file."

Note: `additionalContext` from `UserPromptSubmit` hooks is truncated
at 10,000 characters in current Claude Code — too small for large
overflow. Materializing to a worktree file and rewriting the prompt
avoids the limit entirely.

### 7.3 Sentinels and artifacts

**Success.** The `Stop` hook emits one commit on the state branch:

```
commit <sha>
    GAIA_DONE: <session_id>

A  .gaia-state/<session_id>.md                 (final assistant message)
A  .gaia-state/<session_id>.transcript.md      (readable conversation transcript)
```

**Failure.** The `StopFailure` hook (§8.3) emits one commit on the
state branch when Claude Code reports an API error (rate-limit, auth,
overloaded, prompt-too-long, etc.):

```
commit <sha>
    GAIA_FAILED: <session_id>

A  .gaia-state/<session_id>.error.json         ({error, error_details, last_assistant_message})
A  .gaia-state/<session_id>.transcript.md      (whatever transcript exists at failure time)
```

Without the failure sentinel, the driver would wait the full timeout
(45 min) on every transient API error. `GAIA_FAILED:` lets it fail
fast (exit 13) and surface the real error message.

All artifacts are plain text (Markdown or JSON). The driver reads
`<session_id>.md` on success, `<session_id>.error.json` on failure,
and `<session_id>.transcript.md` for future continuation.

### 7.4 Manifest

Before firing, the driver commits `.gaia-state/manifest.json` to the
state branch:

```json
{
  "run_id": "01HXX...",
  "nonce": "<base64-24>",
  "work_branch": "claude/gaia/01HXX.../work",
  "state_branch": "claude/gaia/01HXX.../state",
  "base_sha": "<sha of user branch at fire time>",
  "user_branch": "feature/auth",
  "driver_version": "1.0.0",
  "issued_at": "2026-04-21T10:30:00Z"
}
```

The `UserPromptSubmit` hook validates the manifest before doing any
checkout or state change. This is the authoritative cross-check for
prompt-supplied branch names; the hook refuses any request whose
manifest is missing, whose nonce does not match, or whose declared
branches disagree with the manifest's contents.

## 8. Hooks

Installed by `gaia init` under the repo root:

```
.claude/
  settings.json
  hooks/
    gaia-user-prompt-submit.sh
    gaia-stop.sh
```

`.claude/settings.json`:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/gaia-user-prompt-submit.sh",
            "timeout": 120
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/gaia-stop.sh",
            "timeout": 600
          }
        ]
      }
    ],
    "StopFailure": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/gaia-stop.sh",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

Schema notes:

- Current Claude Code hook config uses three nesting levels:
  event → matcher group → `hooks` array of handlers. Each handler
  requires `type: "command"` and accepts a per-handler `timeout` in
  seconds. Defaults are 600 seconds; gaia sets tighter timeouts on
  `UserPromptSubmit` and `StopFailure` so a stuck hook cannot monopolize
  the VM.
- `StopFailure` is a separate Claude Code event emitted when the turn
  stops due to an API error — `Stop` does not fire in that case. gaia
  reuses the same shim and binary; behavior branches on
  `hook_event_name` from stdin.

Project-level install is chosen over `~/.claude/settings.json` because:

- Routine definitions travel with the repo; hooks in the repo ensure
  every cloud VM that fires against this repo has them.
- `.claude/settings.local.json` is explicitly not loaded in routines,
  so it cannot be used.
- Committing hooks to the repo is the only way to guarantee they are
  active on a fresh cloud VM that has not run the setup script yet (or
  where the cache was invalidated).
- The hook files must live on the default branch. Routines clone the
  default branch and run `UserPromptSubmit` *before* the hook has a
  chance to switch to the work branch — the hook itself has to be on
  the branch the clone starts from. `gaia init` enforces this (§5,
  §6.4).

### 8.0 Shim scripts

`.claude/hooks/gaia-user-prompt-submit.sh` and
`.claude/hooks/gaia-stop.sh` are small shell scripts committed to the
repo. Each:

1. Reads hook stdin (JSON) into memory.
2. For `UserPromptSubmit`: exits 0 silently if the prompt does not
   contain `GAIA_RUN_ID:` on its first line.
3. For `Stop` / `StopFailure`: exits 0 silently if
   `/tmp/gaia/<session_id>.json` does not exist (the
   `UserPromptSubmit` hook writes this file only when it recognizes a
   gaia session).
4. Otherwise, locates the global `gaia-hook-*` binary on `$PATH` and
   `exec`s it with the original stdin. If the binary is absent, emits
   a single-line warning to stderr and exits 0 so the hook does not
   block a teammate's non-gaia Claude Code session.

This means a teammate who opens Claude Code on the repo without gaia
installed sees no disruption: the shim runs, recognizes the session is
not a gaia session (or the binary is absent), and returns silently.

### 8.1 `gaia-hook-user-prompt-submit`

Fires on every user prompt submission. Blocks until it returns.

**Input (stdin JSON, v1 known fields).**

```json
{
  "hook_event_name": "UserPromptSubmit",
  "session_id": "session_01HJK...",
  "transcript_path": "/home/user/.claude/projects/.../<uuid>.jsonl",
  "cwd": "/home/user/repo",
  "prompt": "GAIA_RUN_ID: 01HXX...\n..."
}
```

**Behavior.**

1. Parse `GAIA_RUN_ID`, `GAIA_WORK_BRANCH`, `GAIA_STATE_BRANCH`,
   `GAIA_NONCE` from the first lines of `prompt`. If any is absent,
   exit 0 (non-gaia session — hook no-ops).
2. Validate branch-name patterns against
   `^claude/gaia/<ulid>/(work|state)$`. On mismatch, exit non-zero.
3. Fetch `origin/$GAIA_STATE_BRANCH` to a bare `.git/gaia-state-check`
   and read `.gaia-state/manifest.json` from it. Refuse if missing,
   if the manifest's `nonce` does not match the prompt's
   `GAIA_NONCE:`, or if any branch name in the manifest disagrees with
   the prompt. On mismatch, exit non-zero.
4. Persist the branch names so Claude's later Bash tool calls can
   read them without relying on hook-exported environment variables
   (current Claude Code does not inject hook env vars into the Bash
   tool's process for `UserPromptSubmit`):
   ```
   git config gaia.runId "$GAIA_RUN_ID"
   git config gaia.workBranch "$GAIA_WORK_BRANCH"
   git config gaia.stateBranch "$GAIA_STATE_BRANCH"
   git config gaia.experimentalMode "$GAIA_EXPERIMENTAL_MODE"   # if present
   ```
5. Write `/tmp/gaia/<session_id>.json` with
   `{run_id, work_branch, state_branch, experimental_mode}` so the
   `Stop` / `StopFailure` hook can correlate by `session_id`.
6. Fetch and check out the work branch:
   ```
   git fetch origin "$GAIA_WORK_BRANCH"
   git checkout "$GAIA_WORK_BRANCH"
   ```
7. If the payload contains `--- CONTEXT FILE ---`, fetch the state
   branch, copy `.gaia-state/context.md` into the worktree at
   `.gaia/runs/<run_id>/context.md` (the `.gaia/` rule in `.gitignore`
   keeps it out of any commit), and emit a JSON stdout payload
   rewriting the marker in the prompt to inline instructions:
   ```json
   {
     "hookSpecificOutput": {
       "hookEventName": "UserPromptSubmit",
       "updatedPrompt": "<prompt with marker replaced by `Read .gaia/runs/<run_id>/context.md before acting.`>"
     }
   }
   ```
8. Exit 0.

Any git error aborts the hook with non-zero exit, blocking prompt
submission. The session fails visibly in the routine UI; the driver
hits no-sentinel timeout (exit 3) and surfaces the session URL for
`gaia recover`.

### 8.2 `gaia-hook-stop` (Stop event)

Fires when the assistant finishes its final response, before session
teardown. The same binary handles `StopFailure` (§8.3); behavior
branches on `hook_event_name`.

**Input (stdin JSON, v1 known fields).**

```json
{
  "hook_event_name": "Stop",
  "session_id": "session_01HJK...",
  "transcript_path": "/home/user/.claude/projects/.../<uuid>.jsonl",
  "last_assistant_message": "…the final assistant text…",
  "stop_hook_active": false
}
```

Current Claude Code includes `last_assistant_message` on `Stop`
precisely so hooks do not have to re-parse the transcript just to
recover the final message. gaia uses it directly for the final-message
artifact. Transcript JSONL parsing is still used for the readable
full-transcript artifact, but the critical happy-path output no longer
depends on undocumented JSONL schema.

**Behavior.**

1. Look up `/tmp/gaia/<session_id>.json`. If absent, exit 0 (non-gaia
   session).
2. Read `gaia.workBranch` / `gaia.stateBranch` from git config. If
   either is missing, abort with a non-zero exit — the session was not
   properly initialized.
3. **Final commit sweep (conditional) then push (unconditional).**
   ```bash
   if [ -n "$(git status --porcelain)" ]; then
     git add -A
     git commit -m "gaia: final sweep"
   fi
   git push origin "HEAD:$GAIA_WORK_BRANCH"
   ```
   The push is **unconditional**. Earlier drafts only pushed when the
   working tree was dirty, which dropped clean local commits Claude
   had created but not pushed — those are now always synced to origin.
4. **Extract the final assistant message** from the stdin
   `last_assistant_message` field. If missing or empty, fall back to a
   reverse-scan of the JSONL transcript (looking for the last line
   where `message.role === "assistant"` or `type === "assistant"`,
   concatenating text blocks). Last-resort fallback: the raw last line
   of the JSONL — still human-readable and the driver will print it.
5. **Build the readable transcript** by one forward pass through the
   JSONL. For each line where `message.role` is `user` or
   `assistant`, emit a Markdown section:
   ```
   ## USER (2026-04-21T10:31:04Z)
   <text content>

   ## ASSISTANT (2026-04-21T10:31:12Z)
   <text content, tool-use blocks summarized as `[tool: bash — git status]`>
   ```
   Tool-result entries are summarized as `[tool-result: ok]` or
   `[tool-result: error — <first line>]` inline under the preceding
   assistant message. Thinking blocks are dropped.
6. **Write both artifacts** via a temporary worktree on the state
   branch:
   ```
   tmp=$(mktemp -d)
   git fetch origin "$GAIA_STATE_BRANCH"
   git worktree add "$tmp" "origin/$GAIA_STATE_BRANCH"
   mkdir -p "$tmp/.gaia-state"
   printf '%s\n' "$final_message"   > "$tmp/.gaia-state/$session_id.md"
   printf '%s\n' "$transcript_text" > "$tmp/.gaia-state/$session_id.transcript.md"
   git -C "$tmp" add .gaia-state
   git -C "$tmp" commit -m "GAIA_DONE: $session_id"
   git -C "$tmp" push origin "HEAD:$GAIA_STATE_BRANCH"
   git worktree remove "$tmp"
   ```
7. Exit 0.

The sentinel push is the only way the driver learns the session is
done; the hook must not exit before completing step 6.

### 8.3 `gaia-hook-stop` (StopFailure event)

When `hook_event_name === "StopFailure"`, stdin includes `error`,
`error_details`, and `last_assistant_message` fields. The hook:

1. Looks up `/tmp/gaia/<session_id>.json`. No-op if absent.
2. Best-effort commits and pushes any unsaved work to the work branch
   (same commit-then-push as §8.2 step 3). The API error may have
   damaged the VM state; failures here are logged but do not block the
   failure sentinel.
3. Writes `.gaia-state/<session_id>.error.json` containing
   `{error, error_details, last_assistant_message}` and
   `.gaia-state/<session_id>.transcript.md` (whatever transcript
   content exists) to the state branch via the same temporary-worktree
   flow.
4. Commits with message `GAIA_FAILED: <session_id>` and pushes.
5. Exits 0.

The driver reads `GAIA_FAILED:` and exits 13, printing the error body
to stderr. The failed run is recorded in history with
`status: "failed"` so `gaia history` surfaces it and `gaia recover`
can act on it.

### 8.4 Schema fragility and `gaia doctor`

The transcript JSONL schema is not a documented contract. gaia's
dependency on it is minimal — `last_assistant_message` from hook stdin
is authoritative for the final message; JSONL parsing only feeds the
readable transcript artifact. The parser:

- Treats any line whose `message.role` OR top-level `type` equals
  `"assistant"` as an assistant message.
- Extracts text defensively (string OR array-of-blocks).
- Falls back to raw-last-line output on unrecognized shapes.

`gaia doctor` (§6.4) runs an end-to-end synthetic session and verifies:

1. Spawner config is valid: 200 on `/fire`, correct response field
   names (`claude_code_session_id`, `claude_code_session_url`).
2. The Claude GitHub App can push to `claude/gaia/*` on this repo.
3. The `UserPromptSubmit` hook receives the expected fields.
4. `last_assistant_message` is present on the `Stop` input on this
   Claude Code version.
5. The JSONL transcript on this version parses into the expected
   forward-pass output.
6. `StopFailure` is a recognized event (empirically: issue a known-
   bad prompt such that the routine rejects it and verify the hook
   fires with the failure fields).
7. (If `--experimental-mode` is configured) the experimental-mode
   flow from §15.7 completes.

`gaia init` offers to run `gaia doctor` automatically before declaring
setup complete.

### 8.5 Why hooks and not a session CLI

A prior draft had `gaia session ask` / `gaia session done` subcommands
the agent had to call explicitly. Hooks are strictly better:

- Agent cannot forget to emit the sentinel — `Stop` / `StopFailure`
  always fire.
- Agent cannot leave work uncommitted — final sweep + unconditional
  push guarantees it.
- The saved routine prompt shrinks to something static, so prompt
  drift across projects disappears.
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

If the user message tells you to read a file under .gaia/runs/, read
it first — it holds the real task content that did not fit inline.

Commit your work with git as you go. You do not need to push — a gaia
hook will push your final state to the correct branch when you are
done. If you do need the current work branch name for any reason,
read it with:

  git config --get gaia.workBranch

Do not merge, open pull requests, or comment on GitHub.

Your final assistant message is captured automatically as the result
returned to the caller. Make it the thing the caller needs to read —
a concise summary of what you did, files changed, and any open
questions.
```

The prompt is authoritative that pushing is not Claude's
responsibility: the Stop hook's unconditional push is the protocol's
single source of truth for "the work branch on origin matches
session HEAD." If Claude pushes anyway, the Stop-hook push remains
idempotent.

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
  history.jsonl                                run summaries for -c / --resume
  history.lock                                 advisory lock for appends
  transcripts/
    <session_id>.md                            readable conversation transcripts
  secrets.json                                 fallback store (0600)
  runs/
    <run_id>/
      meta.json                                full per-run metadata (canonical)
      fire.json                                raw spawner response
      context.md                               overflow context materialized by the hook (only if needed)
```

**Source of truth.** `.gaia/runs/<run_id>/meta.json` is canonical.
`history.jsonl` is a derived append-only log optimized for fast
reverse-chronology scans. If the two disagree, `meta.json` wins; gaia
offers to rebuild `history.jsonl` from the runs directory.

`meta.json` schema:

```json
{
  "run_id": "01HXX...",
  "session_id": "session_01HJK...",
  "session_url": "https://claude.ai/code/sessions/session_01HJK...",
  "status": "pending" | "success" | "failed" | "timeout" | "push-rejected" | "conflict",
  "user_branch": "feature/auth",
  "work_branch": "claude/gaia/01HXX.../work",
  "state_branch": "claude/gaia/01HXX.../state",
  "base_sha": "abc1234",
  "merge_sha": "def5678",
  "continued_from": "session_01HAA...",
  "started_at": "2026-04-21T10:30:00Z",
  "finished_at": "2026-04-21T10:35:12Z",
  "prompt_first_line": "refactor the auth module...",
  "error": null
}
```

`history.jsonl` — one JSON object per line, a flat-summary view of
`meta.json`:

```json
{
  "run_id": "01HXX...",
  "session_id": "session_01HJK...",
  "timestamp": "2026-04-21T10:35:12Z",
  "status": "success",
  "user_branch": "feature/auth",
  "prompt": "refactor the auth module...",
  "summary": "Refactored the auth middleware...",
  "merge_sha": "abc1234",
  "continued_from": "session_01HAA..."
}
```

`status` takes the same set as `meta.json`. Failed, timed-out, and
push-rejected runs are first-class entries — visible to `gaia
history` and actionable by `gaia recover` — so the recovery story does
not leak through the UX.

**Concurrency.** Multiple parallel `gaia` processes may append to
`history.jsonl`. gaia uses a file lock (`.gaia/history.lock` via
`flock` where available; `LockFileEx` on Windows) around the append
window. Prior drafts claimed `O_APPEND` + `PIPE_BUF` prevents
corruption; that is wrong — `PIPE_BUF` is a pipe / FIFO guarantee, not
a regular-file guarantee, and Node's append helpers may not map to a
single `write()` syscall. If history is ever corrupted (partial write,
filesystem without working locks), gaia rebuilds from
`.gaia/runs/*/meta.json`.

### 10.4 `.gitignore`

`gaia init` appends:

```
.gaia/
```

Only this. `.gaia-state/` lives on orphan state branches exclusively
and never touches reviewed branches. Overflow context files written
to `.gaia/runs/<run_id>/context.md` are covered by the top-level
`.gaia/` rule, so the Stop hook's `git add -A` will never commit them
to the work branch.

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
- Response mapping: `claude_code_session_id` → `sessionId`,
  `claude_code_session_url` → `sessionUrl`.
- Honors 429 `Retry-After` verbatim.
- Exponential backoff on 503 up to 3 attempts.
- Non-recoverable on 401/403/400-paused/400-text-too-large.
- `maxPayloadChars: 65536`, `defaultTimeout: "45m"`.
- The endpoint returns immediately after session creation — it does
  not stream or wait. gaia's sentinel / polling design is the intended
  completion transport.

## 12. Failure modes

| Failure | Detection | Policy |
|---|---|---|
| `-p` missing | Arg parsing | Exit 2 with usage. |
| Spawner HTTP error | `fire()` rejects | Retry per §11.2. Exhausted → exit 10–12. |
| `GAIA_FAILED:` sentinel received | Poll loop sees failure sentinel | Exit 13. Error body printed. `meta.json.status = "failed"`. `gaia recover` offered. |
| No sentinel by timeout | Poll loop deadline | Exit 3. Work + state branches preserved. Session URL printed. `meta.json.status = "timeout"`. Recoverable via `gaia recover`. |
| Merge conflict | `git merge` fails | Abort merge. Exit 4. Work branch preserved; user branch untouched. `meta.json.status = "conflict"`. Recoverable via `gaia recover`. |
| Push rejected after local merge | `git push origin <user-branch>` fails | Exit 15. Local user branch has the merge commit. `meta.json.status = "push-rejected"`. Printed instructions: investigate upstream advance, then `git push` when ready. |
| User branch diverged non-FF at merge time | `git merge-base --is-ancestor` fails pre-merge | Exit 2 with recovery message. |
| Dirty working tree at start | `git status --porcelain` non-empty | Exit 2. |
| Detached HEAD at start | No symbolic ref | Exit 2. |
| `UserPromptSubmit` hook fails | Hook non-zero | Session blocked by Claude Code. Driver hits no-sentinel timeout; session URL surfaced. |
| `Stop` hook fails before sentinel push | No sentinel on state branch | Driver times out. Any pre-failure pushed commits on the work branch are preserved. `gaia recover` surfaces the session URL. |
| `StopFailure` hook fails | No `GAIA_FAILED:` commit | Driver times out (exit 3) instead of exiting 13. Same recovery path. |
| Transcript parse fails (final message) | `last_assistant_message` empty + JSONL shape unknown | Fallback chain: `last_assistant_message` → reverse-scan JSONL → raw last line. Run still succeeds; stdout is whatever the fallback produced. |
| Transcript parse fails (readable transcript) | Same | `.gaia-state/<session_id>.transcript.md` contains the raw JSONL; continuation still works (Claude can read it) with higher token cost. |
| Duplicate sentinel | Two `GAIA_DONE:` commits for the same session_id | Earliest wins. Warning logged. |
| `-c` with no matching run on current branch | No entries with matching `user_branch` and `status: "success"` | Exit 2 with a message listing recent failed / timed-out runs on this branch and suggesting `gaia recover`. |
| `--resume <session_id>` unknown | No matching entry | Exit 2. |
| Continuation transcript missing | `.gaia/transcripts/<session_id>.md` absent | Exit 2 with a message. Can happen if the transcripts directory was deleted manually. |

### 12.1 Cleanup

Default: delete `claude/gaia/$RUN_ID/{work,state}` from origin
immediately after a successful merge. Transcripts have already been
copied locally to `.gaia/transcripts/` at step 8 of §6.1, so the state
branch is disposable at that point.

Exceptions (branches preserved on origin):

- `--keep-branches`.
- Any non-zero exit (3, 4, 13, 15) — `gaia recover` needs the
  branches to exist.

`gaia gc` sweeps `claude/gaia/*` branches on origin whose **tip commit
committer-date** is older than `--max-age` (default 14d). Normal
clients cannot inspect the remote reflog, so committer-date on the
branch tip is the reliable cross-client signal. `gaia gc` also cross-
references against `.gaia/runs/<run_id>/meta.json` when available and
prefers its local metadata's `finished_at`, only falling back to
committer-date when no local metadata exists (runs fired from another
machine).

## 13. Concurrency

Parallel runs are supported. Two or more `gaia` invocations can be in
flight simultaneously against the same repo, the same user branch,
and the same account.

**What makes this safe:**

- `GAIA_RUN_ID` is a ULID — globally unique across repos, users, and
  time. Every invocation gets fresh `claude/gaia/<run_id>/{work,state}`
  branches, a fresh `.gaia/runs/<run_id>/` local directory, and a
  fresh `session_id` from the spawner. Nothing is shared across
  concurrent runs at the protocol level.
- `.gaia/history.jsonl` is guarded by a file lock
  (`.gaia/history.lock`; §10.3). Concurrent appends serialize at the
  lock, not at the filesystem. If the lock fails (filesystem without
  working `flock`), gaia still has
  `.gaia/runs/<run_id>/meta.json` as the canonical source and can
  rebuild history on next read.
- The `Stop` hook correlates by `session_id` via
  `/tmp/gaia/<session_id>.json`, which the `UserPromptSubmit` hook
  writes per-session. Different sessions on the same VM (unusual but
  possible) do not collide.

**What can still conflict:**

- **Merge-back at run-end.** Two parallel runs that both modify the
  same files on the user branch will conflict when the second merges.
  Handled per §12 exit 4: abort the merge, preserve the work branch,
  surface the run id, let the user resolve. This is the same behavior
  as any concurrent git work and is not specific to gaia.
- **Spawner quota.** The Claude Code routines daily cap is
  per-account. N parallel invocations draw N times the budget. 429
  from the spawner is terminal for the current window per §11.2.
- **Same-terminal `--experimental-mode`.** Two `--experimental-mode`
  invocations from the same terminal conflict on stdin — only one
  process can read input. Run them in separate terminals, or
  serialize. Stable (non-experimental) runs have no stdin dependency
  and can share a terminal via backgrounding without issue.

**What happens when `-c` is invoked during in-flight runs:**

`-c` picks the most recent `status: "success"` run on the current
branch. Pending runs have a history entry but are skipped. Failed or
timed-out runs are visible via `gaia history` and recoverable via
`gaia recover`, but are not continued by `-c` (which is only for
replaying successful conversations). For determinism across ambiguous
cases, use `--resume <session_id>`.

A `.gaia/lock` advisory lock remains future work (§14) for callers
who want to prevent concurrent merges into a single user branch by
policy rather than by convention.

## 14. Open questions and future work

- **True session resume.** If Anthropic exposes a way to continue a
  prior `session_id` via `/fire` (or a successor endpoint), `-c` and
  `--resume` can drop the transcript replay in favor of server-side
  resumption. The user-facing flags do not change.
- **Transcript schema stability.** gaia's parser is defensive and
  falls back gracefully, but a schema change could degrade the
  readable transcript format. `gaia doctor` (§6.4, §8.4) monitors
  this.
- **Public npm name.** `gaia` on the public npm registry appears
  taken; v1 ships scoped. If the project eventually owns an unscoped
  `gaia` package, `gaia init` can update its documented install line
  in a minor release without breaking anything on the wire.
- **Auto-commit hook.** A `PostToolUse` hook on `Edit` / `Write`
  would strengthen crash-safety at the cost of a noisier git log.
  Deferred.
- **Protocol subcommands.** `gaia run fire`, `gaia run wait`,
  `gaia run finish` — the multi-session interface that tools like
  loopx will need. Layers on top of the same protocol described here.
- **Additional spawners.** `local-claude` (shells out to `claude -p`
  locally — no cloud VM required), `codex`, `github-action`.
- **Concurrent-branch lock.** `.gaia/lock` or an origin-side advisory
  lock on the user branch.
- **Interactive picker for `--resume`.** `gaia --resume -p "<prompt>"`
  without a session id could show the recent history and prompt for
  selection, matching `claude --resume` without an argument.
- **Cross-machine history sync.** v1 history + transcripts are
  machine-local. A future mode could commit them to a dedicated
  `claude/gaia-history` branch so continuation works across machines.
- **Routine provisioning API.** If Anthropic exposes a routine-creation
  endpoint, `gaia init` can stop printing "paste this into the UI."
- **Graduating `--experimental-mode` to stable** (§15). Once VM
  idle-lifetime is characterized empirically and the `Stop`-hook
  decision semantics are confirmed stable across Claude Code versions,
  the flag drops the lab-quality warning and/or the mode becomes the
  default transport for `-c` / `--resume`.

## 15. Lab-quality mode: `--experimental-mode`

**Status: lab-quality. Not part of the v1 stable contract.** Ships in
v1.x behind `--experimental-mode`. Prints a warning on start. Depends
on current `Stop`-hook block-decision semantics, which are sparsely
documented and may change between Claude Code versions. Falls back
automatically to the stable fire-and-forget path on any failure
(§15.3). Intended to graduate once the constraints in §15.4 are
characterized and `gaia doctor` can certify them.

### 15.1 Motivation

Stable `-c` / `--resume` continuation is correct but costly for
conversational workflows:

- Every turn is a cold VM (git clone, setup script, Claude cold-boot).
- Transcript replay makes input-token cost grow O(N²) across a long
  conversation.
- Internal session state (tool-use plans, thinking, short-term context
  not expressed in the transcript) is discarded between turns.

Lab-quality mode keeps a single session alive across multiple Q/A
exchanges by exploiting Claude Code's `Stop` hook's ability to return
`{"decision": "block", "reason": "<text>"}` — which prevents session
termination and injects `reason` as a system reminder Claude sees
before continuing. Anthropic's `/fire` endpoint offers no
send-message-to-session API publicly (§14, confirmed in docs research);
this hook-level mechanism is the only way to achieve in-session Q/A
without a notifier.

### 15.2 Mechanism

No messaging layer. No notifier plugin. The "messaging channel" is
the driver's stdin on one side and the `Stop` hook on the other,
connected through commits on the run's orphan state branch.

Flow:

1. Driver fires the routine with a `GAIA_EXPERIMENTAL_MODE: 1` header
   in the payload. Drops the state-branch poll interval to 2s for
   this run. The `UserPromptSubmit` hook persists
   `gaia.experimentalMode = 1` to git config so the Stop hook sees it
   regardless of transcript state.
2. VM: Claude processes the initial task, produces a response
   (possibly ending in a question). `Stop` hook fires.
3. Hook first checks `stop_hook_active` from stdin. If `true`, the
   hook itself caused the previous block — short-circuit to exit 0 to
   prevent infinite continuation. Otherwise, detect experimental mode
   from git config. Instead of the standard `GAIA_DONE:` sentinel,
   push:
   ```
   GAIA_PENDING: <session_id> #<turn_n>

   A  .gaia-state/<session_id>/pending_<turn_n>.md    (Claude's latest response)
   ```
4. Hook starts polling (2s) `origin/<state_branch>` for a matching
   `GAIA_INPUT: <session_id> #<turn_n>` commit. Blocking-wait up to
   9 minutes (safely under the configured 10-min hook timeout in §8).
5. Driver sees `GAIA_PENDING:`. Reads `pending_<turn_n>.md`. Prints
   Claude's response to stdout. Prompts the user:
   ```
   reply (ctrl-D to finish):
   ```
6. User types a reply on stdin (supports multi-line; ends on EOF).
7. Driver writes `.gaia-state/<session_id>/input_<turn_n>.md` with
   the reply and pushes `GAIA_INPUT: <session_id> #<turn_n>` to the
   state branch. The input-file body is one of:
   - `GAIA_CONTINUE` — driver auto-continue (§15.5 heuristic).
   - `__GAIA_END__` — user ended the session with EOF after accepting
     Claude's response.
   - `__GAIA_ABORT__` — user typed `:abort` at the reply prompt.
   - Otherwise: the user's literal reply text.
8. Hook sees the input commit and branches on the body:
   - **Regular reply:** return the Stop-hook decision JSON
     ```json
     {
       "decision": "block",
       "reason": "<user's reply, verbatim>"
     }
     ```
     Claude resumes with the reply as a system-reminder user turn.
     Loop to step 2 for turn `n+1`.
   - **`GAIA_CONTINUE`:** return
     ```json
     {
       "decision": "block",
       "reason": "No user reply was provided. Continue the task autonomously."
     }
     ```
     Loop to step 2 for turn `n+1`.
   - **`__GAIA_END__` or `__GAIA_ABORT__`:** **the same hook
     invocation** performs the final sweep, writes the standard
     success artifacts, and pushes `GAIA_DONE: <session_id>` — *then*
     exits 0. This is required because `decision: "block"` is the only
     thing that keeps the session alive; exiting 0 without the block
     decision finalizes the session immediately, so no separate Stop
     hook will run later. All finalize work must happen in this
     invocation.

Notes on hook-decision semantics (current Claude Code docs):

- `Stop` hook decision-control fields are `decision` and `reason`.
  `additionalContext` is documented for `UserPromptSubmit`, not
  `Stop`, and is ignored on Stop. gaia places the user's reply in
  `reason`.
- `reason` is what Claude sees as system-reminder context on the next
  turn and is also shown in the CLI log line; gaia treats it as the
  authoritative inject channel.

### 15.3 Fallback

Lab-quality mode falls back to the standard stable path on any of:

| Trigger | What happens in the VM | Driver behavior |
|---|---|---|
| Hook timeout (9 min without `GAIA_INPUT`) | Hook runs the final-sweep + `GAIA_DONE:` path and exits 0. | Driver reads the sentinel and prints the final assistant message. Exits 0. User may continue with `gaia -c -p "..."` (transcript replay). |
| Hook exits non-zero (git error, JSON schema drift) | Session terminates abnormally. No sentinel pushed. | Driver times out. Exits 3. Recovery via `gaia recover <run_id>`. |
| `StopFailure` fires instead of `Stop` (API error) | `GAIA_FAILED:` sentinel pushed; no pending turn. | Driver exits 13. `gaia recover` surfaces the run; `gaia -c` replay possible with the transcript captured pre-failure. |
| VM wall-clock timeout (undocumented Anthropic cap) | Session dies server-side. | Driver detects state-branch inactivity, exits 3. Recovery via `gaia recover`. |
| Driver killed (user ctrl-C or SIGTERM) | Driver leaves `meta.json.status = "timeout"` (or `"pending"` if the kill happened before the first poll). Hook eventually times out and runs the fallback finalize. | User runs `gaia recover <run_id>` when ready. The final sweep in the hook preserved the work-branch commits; the pending response is visible in the eventual sentinel. |
| Explicit user abort (`:abort` at the reply prompt) | Driver pushes `__GAIA_ABORT__`. Hook finalizes with `GAIA_DONE:` in the same invocation. | Session finalizes normally. Same as EOF. |

In every fallback case:

- **No work is lost for clean exits.** The Stop hook's unconditional
  push (§8.2 step 3) syncs work-branch state to origin in every clean-
  exit path. Unclean exits preserve the work branch on origin for
  `gaia recover`.
- **Failed runs are first-class in history.** Because each run writes
  a `status: "pending"` entry before firing (§6.1 step 3), a crashed
  or timed-out lab-quality run is visible to `gaia history` and
  actionable by `gaia recover`. This closes the recovery gap the prior
  draft had, where `gaia -c` could silently skip a broken run and
  continue the one before it.
- **No protocol divergence.** Every fallback ends with either
  `GAIA_DONE:` or `GAIA_FAILED:` + the standard artifacts; downstream
  tooling (merge, history append, stdout) is identical.

### 15.4 Constraints and known issues

- **Fidelity gap.** User replies arrive as `reason` system-reminder
  text, not as native user turns. Claude handles system reminders
  well in practice, but a native `/fire`-with-new-user-message API
  would be higher fidelity. That API is not exposed publicly for
  routines.
- **Per-turn timeout.** Bounded by the `Stop`-hook timeout (§8
  configures 600s default). The hook internally uses 540s to leave a
  safety margin before Claude Code kills it. The per-handler
  `timeout` is configurable but has no documented upper bound; raising
  it risks hitting the (also undocumented) VM wall-clock cap.
- **Infinite-continuation guard.** The `stop_hook_active` check (§15.2
  step 3) is the only thing preventing a block → unblock → Stop →
  block loop. gaia short-circuits on `stop_hook_active === true` even
  when an input commit looks valid. An edge case where the driver
  dies between pending turns is covered by the fallback timeout.
- **VM wall-clock ceiling unknown.** Anthropic's docs state idle
  compute is not billed but do not specify a max session lifetime.
  `gaia doctor` should measure this empirically before users rely on
  long `--experimental-mode` sessions in production.
- **Single input channel.** The driver's stdin is the only reply
  surface. No multi-device input; no replying from a phone. If that
  is desired, re-add a notifier (§14).
- **Concurrency.** Covered in §13. Parallel runs are supported across
  terminals — each has a unique run_id, state branch, and session_id,
  so commits do not collide. The only constraint specific to this
  mode is stdin: two `--experimental-mode` invocations from the same
  terminal cannot both read input at once.
- **Transcript size.** Long interactive sessions accumulate large
  transcripts. The final `transcript.md` artifact may be multi-MB;
  the `Stop`-hook writer streams to disk to avoid memory pressure.
- **Subagents unchanged.** The `Task`-tool subagent flow fires
  `SubagentStop`, not `Stop`, so subagent execution is unaffected by
  `--experimental-mode`. Only the main-session `Stop` is intercepted.

### 15.5 UX sketch

```
$ gaia --experimental-mode -p "refactor the auth module to use JWT"
[gaia] --experimental-mode is lab-quality. Falls back to -c on timeout or failure.
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
[gaia] auto-continuing (GAIA_CONTINUE)...

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

The "no reply prompt" behavior in the second turn uses a simple
heuristic (assistant response does not end with a question mark and
does not contain typical question phrasings) plus a brief idle pass
(2 s of waiting on stdin with no input) to auto-continue without
prompting. When auto-continuing, the driver pushes `GAIA_CONTINUE` as
the input body so the hook replies with the autonomous-continue
reminder rather than a user reply. If the heuristic misfires, the
user can interrupt with ctrl-C and re-invoke with `--resume` to
continue via the stable path.

### 15.6 Protocol additions

Lab-quality mode extends the §7 protocol with two new commit-message
shapes on the state branch. Neither appears in a non-experimental run.

| Message | Producer | Contents |
|---|---|---|
| `GAIA_PENDING: <session_id> #<n>` | VM `Stop` hook | `.gaia-state/<session_id>/pending_<n>.md` — Claude's response for turn `n`. |
| `GAIA_INPUT: <session_id> #<n>` | Driver | `.gaia-state/<session_id>/input_<n>.md` — user's reply or control sentinel (`GAIA_CONTINUE`, `__GAIA_END__`, `__GAIA_ABORT__`) for turn `n`. |

The standard `GAIA_DONE: <session_id>` sentinel is still emitted
exactly once, at the end of a clean-finalize path, regardless of how
many pending / input turns preceded it. The final `transcript.md`
artifact is assembled from the full JSONL transcript — including the
system-reminder-injected turns — by the same reverse-scan +
forward-pass logic in §8.2, with one small extension: system reminders
matching gaia's injection pattern are rendered back as
`USER (via --experimental-mode):` in the Markdown transcript so the
continuation path (§6.2) sees them as real user turns, not as
reminders.

### 15.7 Doctor coverage

`gaia doctor` gains three lab-quality-mode checks (only run when
`--experimental-mode` is configured):

1. Fire a session with `GAIA_EXPERIMENTAL_MODE=1` and a known-trivial
   prompt. Confirm a `GAIA_PENDING:` commit appears within 30 s.
2. Push a `GAIA_INPUT:` reply. Confirm the hook's returned
   `{decision: "block", reason: <reply>}` takes effect and Claude
   produces a follow-up `GAIA_PENDING:` commit within 60 s.
3. End the session with `__GAIA_END__`. Confirm the same hook
   invocation pushes `GAIA_DONE:` within 30 s and the final transcript
   includes the injected turn rendered as `USER (via --experimental-mode):`.

Failure on any of the three → `--experimental-mode` is not usable on
this Claude Code version; `gaia doctor` reports which step failed and
whether it is likely a hook-timeout, block-decision-semantics, or
VM-lifetime issue.
