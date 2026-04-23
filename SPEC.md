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
- **No Zero Data Retention support.** ZDR is generally available on
  Claude Code Enterprise, but orgs with ZDR enabled cannot use
  cloud-session features — routines, `/web-setup`, and the endpoints
  gaia fires against are all blocked for them. The rest of Claude
  Code still works under ZDR; only the cloud-session transport gaia
  depends on is unavailable.
- **No coexistence with pre-existing project-level `UserPromptSubmit`,
  `Stop`, or `StopFailure` hooks.** `UserPromptSubmit` and `Stop` do
  not accept matchers at all; `StopFailure` accepts an `error_type`
  matcher but no per-session or per-run identity matcher gaia could
  use. In all three cases, every registered handler fires in parallel
  on the events gaia cares about, and when multiple handlers return
  decision objects the most restrictive wins. gaia cannot safely
  share any of those events with another hook that might race its
  checkout, block its run, or mutate the conversation. `gaia init`
  refuses to install into a repo that already defines any of the
  three (§5, §6.4, §8.7). Composing with pre-existing hooks is
  deferred to v1.x (§14) and is a non-breaking addition when it
  lands. The fourth event gaia owns — `PreToolUse` — *does* accept
  per-tool matchers, and gaia's matcher set is documented in §8.5
  so a project can add other `PreToolUse` handlers on disjoint
  matchers without conflict.
- **No local Claude invocation.** v1 always goes through a cloud routine.

## 3. Architecture

Everything ships as one npm package. The unscoped name `gaia` appears
to be taken on the public npm registry; v1 publishes under a scoped
name (placeholder: `@your-scope/gaia`, to be finalized before release)
that still installs a `gaia` binary:

```
npm i -g @your-scope/gaia                      # on driver machine AND in the VM
```

The package installs four binaries to `$PATH`:

| Binary | Where it runs | Purpose |
|---|---|---|
| `gaia` | Driver (local) | Entry point: `gaia -p`, `gaia -c -p`, `gaia --resume <id> -p`, `gaia init`, `gaia config`, `gaia history`, `gaia recover`, `gaia doctor`. |
| `gaia-hook-user-prompt-submit` | VM | Handler for Claude Code's `UserPromptSubmit` hook. Parses the `--- BEGIN GAIA CONTROL ---` block, validates the state-branch manifest, checks out the work branch, materializes overflow context. |
| `gaia-hook-pre-tool-use` | VM | Handler for Claude Code's `PreToolUse` hook. For gaia sessions, denies tool calls that would mutate gaia-managed protocol state (shim files, settings, state branches, branch-switch / hard-reset / forced pushes). |
| `gaia-hook-stop` | VM | Handler for Claude Code's `Stop` and `StopFailure` hooks (same binary; branches on `hook_event_name`). Commits and pushes any unstaged work, extracts the final assistant message and conversation transcript, pushes the sentinel commit. |

The VM runs an unmodified `claude` binary. gaia contributes three
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
| Sentinel | A commit on the state branch with message `GAIA_DONE: <session_id>` (success), `GAIA_FAILED: <session_id>` (routine / API error), or `GAIA_ABORTED: <session_id>` (user abort in lab-quality mode). Also `GAIA_PENDING` / `GAIA_INPUT` in lab-quality mode (§15). |
| Spawner | Plugin that dispatches a session to an agent platform. v1 bundles `claude-code-routine`. |

## 5. Prerequisites

**Driver machine.** Node.js 20+, git, push access to the origin repo.

**Teammate machines opening Claude Code locally in a gaia-initialized
repo.** Node.js 20+. This is a new, documented prerequisite for
gaia-initialized repos. It is *not* a Claude Code requirement —
Claude Code itself can be installed via native installer (Homebrew,
WinGet, official native binary) and run with no Node on `PATH`. The
prerequisite comes from gaia: the hook shims committed to
`.claude/hooks/` are Node scripts, and the `command` entries in
`.claude/settings.json` start with `node`. If Node is not on `PATH`
when Claude Code fires the `UserPromptSubmit` / `PreToolUse` /
`Stop` / `StopFailure` hook on a teammate's machine, the hook
command fails to start — disrupting their local Claude Code session
even though they are not running gaia. `gaia init` prints a banner
flagging this expectation so the repo owner can communicate it
before the hook files land on the default branch. See §8.0 for the
shim layout and §14 for tracking a future POSIX `.sh` opt-in for
repos that want to drop the Node dependency for teammates.

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
4. `.claude/settings.json` and `.claude/hooks/*.js` committed to the
   repo root and **merged to the default branch** (see note below).
   `gaia init` writes these files and, if the current branch is not the
   default branch, either opens a PR on the user's behalf (if `gh` is
   available) or prints the exact git commands to run. If the repo
   already defines any project-level `UserPromptSubmit`, `Stop`, or
   `StopFailure` hook, `gaia init` refuses to proceed and points at
   the remediation path in §8.7. Pre-existing `PreToolUse` handlers
   are tolerated when their matchers are disjoint from gaia's
   protected-file matcher set (§8.5).
5. `.gitignore` entry for `.gaia/` (local state only). `gaia init`
   appends this.
6. Claude Code cloud repository access configured for the repo, with
   push permission on the `claude/gaia/*` namespace. In practice today
   this means either the Claude GitHub App is installed (the path the
   product docs recommend and the path `gaia init` verifies) or an
   equivalent access grant via `/web-setup`. Routines can push to any
   `claude/*`-prefixed branch by default; gaia's `claude/gaia/*`
   namespace stays inside that boundary, so no *Allow unrestricted
   branch pushes* setting is required. If the repo has narrower
   restrictions, `gaia init` prints the exact Claude-for-GitHub setting
   to adjust. Requiring the GitHub App specifically is a gaia policy
   choice, not a fundamental constraint of the `/fire` endpoint — a
   future release may relax this once `gaia doctor` can reliably
   discriminate configurations that will and will not work.

**Why the default branch matters (but is not sufficient).** Routines
clone a repo at its default branch and run `UserPromptSubmit` *before*
any checkout. If the hook files do not live on the default branch, the
cloud session never runs them and the protocol silently degrades.
`gaia init` refuses to claim it has completed setup until the hook
files are present on the default branch.

The default-branch presence alone is **not enough**, though: the
`UserPromptSubmit` hook immediately checks out the work branch, and
every subsequent hook invocation (`PreToolUse`, `Stop`, `StopFailure`)
executes `$CLAUDE_PROJECT_DIR/.claude/hooks/gaia-*.js` from the
*work branch's* working tree. A caller on an old feature branch that
predates `gaia init` would otherwise run `UserPromptSubmit` from the
default-branch clone, check out a work branch whose tree has no
`gaia-stop.js` / `gaia-pre-tool-use.js`, and time out with no
sentinel — and worse, run the session unprotected with no
`PreToolUse` enforcement on gaia-managed files. The driver closes
this gap by synthesizing a `gaia: ensure hook shims on work branch`
commit on the work branch before firing (§6.1 step 2, §8.8).

**Per-invocation.**

- The caller is on a non-detached branch. `gaia` refuses otherwise.
- Working tree is clean. `gaia` refuses otherwise. No auto-stash.
- If the user branch tracks an upstream, `git fetch <upstream-remote>
  <upstream-branch>` succeeds. Local-only branches (no tracked
  upstream) are allowed; the work and state branches still push to
  `origin`, but the merge-back at run end skips the upstream push
  step (§6.1.1) and the run finishes with the merge commit only on
  the local user branch. `gaia init` confirms `origin` is configured
  so the work / state branches have somewhere to go regardless of
  whether the user branch itself tracks anything.

**Managed policy.** Enterprise Claude Code installs can set
`allowManagedHooksOnly` (or equivalent) to disable user / project
hooks entirely. `gaia init` and `gaia doctor` both probe whether
project-level hooks are honored on this machine; if they are
disabled by policy, gaia is unusable in that environment until the
policy is relaxed. This is a hard prerequisite, surfaced loudly
rather than silently degraded.

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
2. Build and push the run branches:
   - Seed a local `claude/gaia/$RUN_ID/work` from the user branch.
   - **Enforce the work-branch hook invariant** (§8.8): reconcile the
     gaia-managed files on the work branch against the default
     branch's copies (`.claude/hooks/gaia-user-prompt-submit.js`,
     `.claude/hooks/gaia-pre-tool-use.js`,
     `.claude/hooks/gaia-stop.js`, `.claude/settings.json` entries
     for the four gaia events, `.gitignore` `.gaia/` rule). If any
     differ, commit the reconciled tree with message
     `gaia: ensure hook shims on work branch`. If nothing differs,
     skip the commit. This guarantees the `Stop` / `StopFailure`
     shim the session checks out matches the version the driver
     expects, regardless of how old the user branch is.
   - Push `claude/gaia/$RUN_ID/work`.
   - Push `claude/gaia/$RUN_ID/state` as an orphan root containing
     `.gaia-state/manifest.json` (§7.4).
3. **Write a pending run record.** Create
   `.gaia/runs/<run_id>/meta.json` with `status: "pending"`. The
   `history.jsonl` event log is not touched yet (§10.3 — history is
   written only on terminal states).
4. Compose the payload (§7.2) with `"<prompt>"` as `CONTEXT`.
5. Fire the spawner. If `/fire` rejects the request before a session
   is created (4xx, 5xx, network failure), the driver transitions
   `meta.status` to `"fire-failed"` (a terminal status with
   `session_id: null`; §10.3) and appends a `"fire-failed"` event to
   `history.jsonl`. The exit code is one of 5/10/11/12 per §11.2;
   the work and state branches pushed in step 2 are deleted from
   origin (no session ever ran, so no recovery is possible).
   Otherwise, the spawner returns `claude_code_session_id` and
   `claude_code_session_url`. Update `meta.json` with both.
6. Poll `claude/gaia/$RUN_ID/state` for a sentinel commit matching the
   `session_id`:
   - `GAIA_DONE: <session_id>` → verify the commit contains both
     `.gaia-state/<session_id>.md` and
     `.gaia-state/<session_id>.transcript.md`. If either is missing,
     log a warning (suspected forgery or truncated hook run) and
     keep polling through the remaining timeout. If the commit is
     well-formed, proceed to step 7.
   - `GAIA_FAILED: <session_id>` → routine / API error (§7.3). Update
     `meta.json.status = "failed"`, append a terminal `"failed"`
     event to `history.jsonl`, print the error body, exit 13.
   - `GAIA_ABORTED: <session_id>` → caller aborted a lab-quality
     run (§15). Update `meta.json.status = "aborted"`, append a
     terminal `"aborted"` event, print the partial transcript hint,
     exit 0. Work branch is preserved for `gaia recover` promotion.
7. Fetch the work and state branches.
8. Copy the transcript artifacts to
   `.gaia/transcripts/<session_id>.md` locally.
9. **Merge back transactionally** (§6.1.1).
10. Print `.gaia-state/<session_id>.md` (the final assistant message)
    to stdout.
11. **Finalize the run.** Update `meta.json` with the terminal status
    (`success`, `merge_sha`, `pushed_upstream`, `finished_at`), then
    append a single corresponding event to `.gaia/history.jsonl`
    (with `summary` populated from the truncated final message) under
    the lock (§10.3). History is written once per terminal transition;
    readers treat the latest history record for a run_id as
    authoritative.
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
3. If the user branch tracks an upstream, run
   ```
   git fetch <upstream-remote> \
     "+refs/heads/<upstream-branch>:refs/remotes/<upstream-remote>/<upstream-branch>"
   ```
   to pick up any upstream advance. Local-only user branches skip
   this step (§5).
4. If the user branch tracks upstream, fast-forward local to
   `<upstream-remote>/<upstream-branch>`. If fast-forward is
   impossible (the user committed locally during the run), exit 2
   with a recovery message.
5. Fetch the work branch with an explicit refspec and merge it:
   ```
   git fetch origin \
     "+refs/heads/$GAIA_WORK_BRANCH:refs/remotes/origin/$GAIA_WORK_BRANCH"
   git merge --no-ff "refs/remotes/origin/$GAIA_WORK_BRANCH"
   ```
   Conflict → `git merge --abort`, exit 4; work branch preserved on
   origin.
6. If the user branch tracks upstream,
   `git push <upstream-remote> "<user-branch>:<upstream-branch>"`.
   Never force. Rejection → exit 15 (merge local, push upstream) with
   instructions: investigate the upstream advance, then `git push`
   manually when safe **or run `gaia recover <run_id>`** (which
   retries the push without re-merging; see §6.4). The local user
   branch already has the merge commit. Local-only user branches
   skip this step entirely; the run finishes with the merge commit
   on the local branch only and `meta.json` records
   `pushed_upstream: false` for clarity.
7. On full success, delete the remote work / state branches and
   finalize history.

Explicit refspecs are used throughout because gaia branches are
slash-heavy (`claude/gaia/<ulid>/work`) and ambiguous fetch / checkout
arguments can resolve to either a tag or a remote-tracking ref
depending on local refs. The `+refs/heads/...:refs/remotes/...` form
fetches exactly the named branch into the corresponding tracking ref
and nothing else.

#### 6.1.2 `--no-merge`

When the caller passes `--no-merge`, the driver executes §6.1 steps
1–8 (push, fire, poll, fetch, copy transcripts) and step 10 (print
the final assistant message to stdout), then **skips steps 9 and 12**
(merge-back, remote-branch deletion). Concretely:

- `meta.json.status` is set to `"no-merge"` and a matching terminal
  `"no-merge"` event is appended to `history.jsonl` (§10.3). This is
  a terminal state distinct from `"success"` (merged) and from
  `"conflict"` / `"timeout"` (failures — see §12).
- `claude/gaia/$RUN_ID/{work,state}` are preserved on origin, as if
  `--keep-branches` were also set.
- stdout carries the final assistant message, exactly as in a merged
  run.
- stderr carries one line with the work-branch name and a
  `gaia recover <run_id>` hint so the caller can complete the merge
  later.
- Exit code is `0`. `--no-merge` is an intentional caller request,
  not a failure.
- `gaia -c` scoped to the current branch **does not** pick up
  `"no-merge"` runs (it only continues `"success"`). Use
  `--resume <session_id>` to explicitly continue a no-merge run's
  transcript, or `gaia recover <run_id>` to complete the merge and
  promote the run to `"success"` first.

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

**`-c` / `--continue`** resolves the **most recent terminal event in
`.gaia/history.jsonl` whose `user_branch` equals the current branch**
— regardless of status — and then requires that event to be
`status: "success"`. If the most recent terminal event on the branch
is anything else (`failed`, `timeout`, `conflict`, `push-rejected`,
`no-merge`, `aborted`, `fire-failed`), `gaia -c` exits 2 with a
message naming the offending run and pointing at `gaia recover`.

Scoping to the current branch avoids surprising cross-branch
continuation. The "most recent terminal event must be success" rule
matches the user's mental model — `-c` continues the *last* thing
done on this branch, not an arbitrary older successful thing. An
older successful run can still be resumed deliberately with
`--resume <session_id>`. If there is no terminal entry at all on
the branch, `gaia -c` also exits 2.

**`--resume <session_id>`** targets a specific prior session by its
Claude Code session id (the same id `--resume` takes in `claude`
itself). Works across branches; the caller's current branch becomes
the new user branch for the continuation run. History records the
session id alongside the run id so lookup is direct.

**Resuming a run whose code did not merge.** When the resolved prior
run's `meta.status` is `"no-merge"`, `"conflict"`, `"timeout"`,
`"aborted"`, or `"push-rejected"`, the prior run's code changes do
not necessarily live on the caller's current branch. A naive replay
would hand Claude a transcript describing file edits that are not
actually in the new session's working tree — a guaranteed source of
"you said you changed X but X is not here" confusion.

Guard: before firing the continuation, the driver reads
`meta.work_branch` for the prior run and requires one of the
following to hold on the caller's current branch:
1. `git merge-base --is-ancestor <work_branch_tip> HEAD` succeeds —
   the current branch already contains the prior run's work tip
   (e.g., the caller ran `gaia recover` first, or merged it
   manually). Continue normally.
2. The caller passes `--resume-seed-from-work`, which tells the
   driver to seed the new run's work branch from the prior run's
   work-branch tip instead of the current user branch. The
   continuation inherits the code changes; the merge-back at the
   end of the new run lands both sets of changes onto the current
   user branch (may conflict per §12). This is a caller opt-in
   because it is a non-obvious history rewrite.
3. Otherwise the driver exits 2 with a message that offers
   `gaia recover <run_id>` (complete the prior merge first) or
   `--resume-seed-from-work` (carry the code forward as part of
   the new run) as the two supported paths.

Resuming a `"success"` run is unaffected — the prior code is
already on the user branch as a merge commit, so the current
checkout transitively contains it.

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
| `--no-merge` | off | After the `GAIA_DONE:` sentinel arrives, skip the merge-back in §6.1.1 entirely. See §6.1.2 for exact semantics (history status, branch retention, stdout behavior, exit code). |
| `--keep-branches` | off | Do not delete `claude/gaia/$RUN_ID/*` on origin after merge. |
| `--resume-seed-from-work` | off | When `--resume` / `-c` targets a prior run whose code did not merge (`no-merge`, `conflict`, `timeout`, `aborted`, `push-rejected`), seed the new run's work branch from the prior run's work-branch tip instead of the current user branch. See §6.2. |
| `--spawner <name>` | from config | Override the spawner. |
| `--timeout <duration>` | `45m` | Per-session timeout. |
| `--json` | off | Print a single-line JSON result instead of bare text. Schema: `{run_id, session_id, status, final_message, merge_sha?, continued_from?, transcript_path?, error?}`. `final_message` carries the same content the bare-text mode prints to stdout. On failure (any non-zero exit), `--json` writes the JSON object to **stdout** with `final_message: null` and `error: {kind, message}`; human-readable diagnostics still go to stderr. |
| `--experimental-mode` | off | **Lab-quality** (§15). Keeps the session alive across Q/A turns via Stop-hook block-and-inject. Prints a warning on start. Falls back to stable fire-and-forget automatically on any failure. Not part of the v1 stable contract. |
| `--auto-continue` | off | Experimental-mode only. Auto-reply with `GAIA_CONTINUE` when the assistant response does not appear to ask a question (heuristic in §15.5). Off by default so every turn prompts the user — a wrong auto-continue can turn a clarification into unsupervised work. |

### 6.4 Admin commands

```
gaia init                                      # interactive project wizard
gaia config set <key> <value>                  # secret-aware: *_TOKEN → keychain
gaia config get <key>
gaia config unset <key>
gaia history                                   # list recent runs: run_id, session_id, status, prompt. Scans .gaia/runs/*/meta.json (canonical) and folds in history.jsonl terminal events. Pending runs show as "pending (in-flight or crashed)".
gaia history show <session_id>                 # print the transcript for a prior session
gaia recover <run_id|session_id> [--merge]     # finish a stuck run: fetch preserved state/work, surface the sentinel (if any), offer merge or manual review. --merge opts aborted runs into a merge-back.
gaia gc [--max-age 14d] [--dry-run]            # sweep stale claude/gaia/* branches on origin
gaia doctor                                    # run a synthetic end-to-end session to validate spawner, hooks, transcript schema, branch permissions
gaia version
```

`gaia init`:
1. Prompts for routine URL + token; stores them securely.
2. Writes `gaia.config.json` to the repo root.
3. Writes `.claude/settings.json` and `.claude/hooks/*.js` to the repo
   root (§8). If `.claude/settings.json` exists **but does not already
   define project-level `UserPromptSubmit`, `Stop`, or `StopFailure`
   handlers**, merges the remaining `hooks` key non-destructively. If
   it does define any of those three events, `gaia init` aborts with
   the error documented in §8.7 (hook coexistence) and offers
   migration guidance. Notes that managed-policy or plugin-scope
   handlers for the same events are outside `gaia init`'s visibility
   (§8.7) and, if present, will still run alongside gaia's.
4. Appends `.gaia/` to `.gitignore`.
5. Prints the saved routine prompt (§9) for the user to paste into the
   Claude Code routines UI.
6. Prints the one-line VM setup-script snippet.
7. Prints the Node-prerequisite banner (§5) so the repo owner can
   communicate to teammates that opening Claude Code in this repo
   requires Node 20+ on `PATH`.
8. Verifies the hook files exist on the default branch. If they do
   not, offers to open a PR (via `gh`) or prints the git commands.
   Does not claim success until this step passes.
9. Offers to run `gaia doctor` before exit.

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
- **Always targets the `user_branch` recorded in `meta.json`**, not
  the caller's current checkout. If the current branch differs from
  the recorded one, `gaia recover` requires a clean working tree and
  checks out the recorded branch before merging; it aborts with exit
  2 if the tree is dirty. On success, the caller is left on the
  recorded `user_branch`. This guarantees that a run's changes land
  where the original caller intended, regardless of where the caller
  happens to be now.
- Branches on `meta.status`:
  - **`"push-rejected"`** (§6.1.1 step 6 failed after the local
    merge landed): the local user branch should already contain the
    recorded `merge_sha`. `gaia recover` verifies this with
    `git merge-base --is-ancestor <merge_sha> <user_branch>`. If
    true, it fetches `origin/<user_branch>` and retries
    `git push origin <user_branch>` without re-merging. If the
    upstream has advanced such that fast-forward is no longer
    possible, `gaia recover` stops and instructs the user to
    rebase/merge manually, printing both `merge_sha` and the
    current upstream tip. If the local branch no longer contains
    `merge_sha` (e.g., the user reset locally), `gaia recover`
    refuses and prints the work-branch name so the caller can
    decide whether to re-merge from scratch or abandon. Never
    force-pushes.
  - **`"no-merge"`**, **`"timeout"`**, **`"conflict"`**: fetches
    the preserved `claude/gaia/$RUN_ID/{work,state}` branches. If
    a `GAIA_DONE:` sentinel exists and validates (§6.1 step 6),
    completes the merge transactionally (§6.1.1) and promotes the
    status to `"success"`.
  - **`"aborted"`** (§15, user `:abort`): prints the work-branch
    name and the partial transcript. Merging is **not offered by
    default** — abort means "do not land this." The user can pass
    `gaia recover <id> --merge` to opt in to a merge-back if they
    change their mind; otherwise `gaia recover` exits leaving the
    status as `"aborted"` and the work branch preserved.
  - **`"failed"`**: prints the `session_url`, the error body from
    `.gaia-state/<session_id>.error.json`, and the work-branch
    name for manual review. Offers to re-run `-c` via transcript
    replay against whatever transcript was captured pre-failure.
  - **`"fire-failed"`**: the spawner rejected the request before
    any session existed (§6.1 step 5). There is no `session_id`,
    no work / state branches on origin (deleted at fire-failure
    time), and no recovery action available — `gaia recover` simply
    prints the recorded error body and exits 0. Listed for
    completeness so `gaia history` entries with this status are not
    silently ignored.
  - **`"pending"`** (driver crashed mid-run, no sentinel yet):
    prints the `session_url`, polls the state branch once for a
    late sentinel, and treats a subsequent `GAIA_DONE` /
    `GAIA_FAILED` / `GAIA_ABORTED` arrival like the corresponding
    terminal-state path above.

## 7. Protocol

### 7.1 Branch topology

```
origin/<user-branch>                   ●──●──●                     (untouched during the run)
                                              \
                                               \                   (driver pushes at start; §8.8 synthesis commit applied here if needed)
                                                \
origin/claude/gaia/$RUN_ID/work        ●──●──●──●──●               ([synthesis?] + session commits)
                                                   \
                                                    \              (driver merges at run end)
                                                     ▼
origin/<user-branch>                                 ●             (merge commit)

origin/claude/gaia/$RUN_ID/state    ○──○──○                        (orphan, never merges)
                                    │  │  └── GAIA_DONE:   <session_id>    (or GAIA_FAILED: or GAIA_ABORTED:)
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
terminal sentinel (`GAIA_DONE:`, `GAIA_FAILED:`, or — in experimental
mode — `GAIA_ABORTED:`). `GAIA_PENDING:` / `GAIA_INPUT:` commits
only appear in experimental-mode sessions and always precede a
terminal sentinel on the same branch.

### 7.2 Payload

`/fire` text:

```
--- BEGIN GAIA CONTROL ---
GAIA_RUN_ID: <ulid>
GAIA_WORK_BRANCH: claude/gaia/<ulid>/work
GAIA_NONCE: <base64-24>
--- END GAIA CONTROL ---

--- BEGIN CONTEXT ---
[PREVIOUS CONVERSATION:
<transcript text, verbatim from .gaia/transcripts/<prior_session_id>.md>
END PREVIOUS CONVERSATION]

<user's prompt>
--- END CONTEXT ---
```

The `PREVIOUS CONVERSATION:` block is present only for `-c` or
`--resume`.

**Why a delimited control block, not a header.** Anthropic's `/fire`
endpoint accepts a `text` field that is concatenated with the saved
routine prompt before reaching `UserPromptSubmit.prompt`, and the
exact concatenation order (saved-prompt first vs. text first) is not
guaranteed by the public docs. A header-style classifier ("prompt
begins with `GAIA_RUN_ID:`") would silently no-op if the saved
prompt was prepended. The hook instead scans the entire `prompt`
for an exact `--- BEGIN GAIA CONTROL ---` … `--- END GAIA CONTROL ---`
fence and parses only that block. `gaia doctor` (§8.4) verifies the
fence is reachable on the current Claude Code version before any
real run depends on it.

**State branch is not in the payload.** The state-branch name
(`claude/gaia/<ulid>/state`) is deliberately *not* shown to the
model. The hook derives it internally from `GAIA_RUN_ID`, which is
shown. This is defense-in-depth (§16.1): the model already has to
invent the state-branch naming convention to forge a sentinel, and
the saved routine prompt (§9) tells it explicitly not to touch any
`claude/gaia/*/state` branch. This is not cryptographic
authentication — a determined prompt injection can still forge the
sentinel — but it raises the bar above trivial accident.

**Nonce and manifest validation.** The driver generates `GAIA_NONCE`
per run and commits it into `.gaia-state/manifest.json` (§7.4) *before*
`/fire`. The `UserPromptSubmit` hook fetches the (derived) state
branch, reads the manifest, and refuses to proceed if the nonce in
the prompt does not match the manifest, the declared work-branch
name disagrees with the manifest, or the manifest is missing. This
protects against forged prompts that try to imitate gaia headers
against the wrong run_id.

**Branch-name pattern.** Even with a verified manifest, the hook also
checks that `GAIA_WORK_BRANCH` matches
`^claude/gaia/[0-9A-HJKMNP-TV-Z]{26}/work$` (ULID + fixed suffix) and
that the derived state branch is `<same-run-id>/state`. Mismatches
abort the hook.

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
state branch, and copies `.gaia-state/context.md` into the **work-tree**
at `.gaia/runs/<run_id>/context.md` (gitignored, so the Stop hook's
`git add -A` will not commit it). The hook then returns
`hookSpecificOutput.additionalContext` pointing Claude at the file;
Claude Code injects that string as a system reminder the model sees
alongside the original prompt.

Prompt-rewriting is deliberately not used. Current `UserPromptSubmit`
hook output supports `additionalContext` and `sessionTitle` but does
not expose a documented `updatedPrompt` field. The `--- CONTEXT
FILE ---` marker therefore stays in the submitted prompt verbatim —
it is a human-visible crumb; the load-bearing instruction travels via
the system-reminder injection. The saved routine prompt (§9) only has
to know that a context-file reference means "read that file."

Note on size: Claude Code automatically spills `additionalContext`
over 10,000 characters into a file and replaces the text with a
preview/path before the model sees it. gaia's injected context is a
short pointer ("read this file"), well under the limit, so the
overflow content remains in the git-tracked `context.md` under gaia's
control rather than going through Claude Code's internal spill path.

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

**Abort (experimental mode only).** In `--experimental-mode`, the
`Stop` hook can emit a third terminal sentinel if the user types
`:abort` at the reply prompt (§15):

```
commit <sha>
    GAIA_ABORTED: <session_id>

A  .gaia-state/<session_id>.md                 (last assistant message + aborted marker)
A  .gaia-state/<session_id>.transcript.md      (readable conversation transcript)
```

The driver reads `GAIA_ABORTED:` and exits 0 with
`status: "aborted"`, **without merging** the work branch. The work
branch is preserved on origin so `gaia recover <id> --merge` can
land the changes later if the user changes their mind. This
sentinel never appears in a stable (non-experimental) run.

All artifacts are plain text (Markdown or JSON). The driver reads
`<session_id>.md` on success or abort, `<session_id>.error.json` on
failure, and `<session_id>.transcript.md` for future continuation.

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
prompt-supplied identifiers; the hook refuses any request whose
manifest is missing, whose nonce does not match, or whose
`work_branch` / `run_id` disagree with the prompt headers. `state_branch`
is recorded in the manifest for driver-side bookkeeping but is not
supplied by the prompt (§7.2); the hook derives it internally and
cross-checks it against `manifest.state_branch` as a sanity check.
The manifest-check gives no protection against the session itself
forging sentinels (§16.1) — it only protects against external
prompt spoofing across runs.

## 8. Hooks

Installed by `gaia init` under the repo root:

```
.claude/
  settings.json
  hooks/
    gaia-user-prompt-submit.js
    gaia-pre-tool-use.js
    gaia-stop.js
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
            "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/gaia-user-prompt-submit.js\"",
            "timeout": 120
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit|Bash|NotebookEdit",
        "hooks": [
          {
            "type": "command",
            "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/gaia-pre-tool-use.js\"",
            "timeout": 30
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/gaia-stop.js\"",
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
            "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/gaia-stop.js\"",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

**Why `$CLAUDE_PROJECT_DIR`, not a relative path.** Hook commands
execute with the session's current working directory, which Claude may
have `cd`d into a subdirectory before `Stop` or `UserPromptSubmit`
fires. A bare relative path (`.claude/hooks/…`) would silently break
in those cases. `$CLAUDE_PROJECT_DIR` is expanded by Claude Code to
the repo root regardless of the current directory, so the shim always
resolves.

**Why Node shims, not `.sh`.** The gaia driver is already Node, so
the shim logic (inspect stdin, delegate to the global binary) is
trivial in Node and does not pull in a new runtime for the rest of
the gaia codebase. Node shims work unchanged on Linux, macOS, and
Windows — teammates on any OS who open Claude Code locally in the
repo see the shims no-op cleanly for non-gaia sessions instead of
hitting an unexecutable `.sh` on Windows. The trade-off is that
Node becomes a prerequisite on every machine that opens Claude Code
in a gaia-initialized repo (§5) — Claude Code itself does *not*
require Node (the native installer / Homebrew / WinGet binary works
fine without it), so this prereq is gaia-imposed. Future work (§14)
may add a POSIX `.sh` opt-in for repos that want to drop the Node
prereq for teammates and accept the loss of native Windows support.

Schema notes:

- Current Claude Code hook config uses three nesting levels:
  event → matcher group → `hooks` array of handlers. Each handler
  requires `type: "command"` and accepts a per-handler `timeout` in
  seconds. Defaults are 600 seconds; gaia sets tighter timeouts on
  `UserPromptSubmit`, `PreToolUse`, and `StopFailure` so a stuck hook
  cannot monopolize the VM.
- `UserPromptSubmit` and `Stop` do not accept matchers. `StopFailure`
  accepts an `error_type` matcher but no per-session or per-run
  matcher — gaia therefore registers it without a matcher and
  classifies inside the shim. `PreToolUse` does accept tool-name
  matchers (and `Bash` subcommand `if` rules); gaia uses
  `"Edit|Write|MultiEdit|Bash|NotebookEdit"` as a coarse first cut
  and refines inside the shim.
- `StopFailure` is a separate Claude Code event emitted when the turn
  stops due to an API error — `Stop` does not fire in that case. gaia
  reuses the same shim and binary for both; behavior branches on
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

`.claude/hooks/gaia-user-prompt-submit.js`,
`.claude/hooks/gaia-pre-tool-use.js`, and
`.claude/hooks/gaia-stop.js` are small Node scripts committed to the
repo. Each:

1. Reads hook stdin (JSON) into memory.
2. Classifies the session:
   - **`UserPromptSubmit`** is a gaia session iff the `prompt` field
     contains a well-formed `--- BEGIN GAIA CONTROL ---` …
     `--- END GAIA CONTROL ---` block (§7.2). The shim scans the
     entire prompt for the fence — header-style "begins with"
     classification would silently no-op if the routine prompt or
     a system prefix were prepended.
   - **`PreToolUse` / `Stop` / `StopFailure`** is a gaia session iff
     `<runtime-dir>/<session_id>.json` exists, where `<runtime-dir>`
     is `$XDG_RUNTIME_DIR/gaia/` when set, otherwise
     `/tmp/gaia-$uid/`. The directory is created with mode `0700`
     and per-session files are written with mode `0600`. This file
     is written by `UserPromptSubmit` (§8.1).
3. If the session is **not** gaia: exit 0 silently. No stderr, no
   state change. This is the common path for teammates running
   Claude Code locally in the repo, and it is the only branch that
   ever fails open.
4. If the session **is** gaia: **check protocol-version
   compatibility** before delegating. The committed shim hard-codes
   the protocol version it understands (a small integer baked in at
   `gaia init` time). Each global `gaia-hook-*` binary advertises a
   matching version. If the shim's version differs from the global
   binary's version (the routine VM may have an older
   environment-cached install per §5; teammates may have an older
   global install), the shim fails closed with the same path as
   "binary missing" below — the driver gets a fast, explicit
   failure rather than a silent protocol mismatch.
5. Otherwise locate the global `gaia-hook-*` binary on `$PATH` and
   spawn it with the original stdin, piping stdout/stderr through.
   Exit with the child's exit code.
6. **Fail-closed when the global binary is missing or version-skewed
   for a gaia session.** A gaia session that cannot reach a
   compatible finalizer must not silently succeed — that would push
   no sentinel and the driver would time out after 45 minutes with
   no useful error.
   - For `UserPromptSubmit`: emit a `decision: "block"` JSON
     output with a clear reason ("gaia hook binary not installed in
     this environment, or version mismatch — install
     `@your-scope/gaia@<expected>` globally") and exit 0, which
     blocks the prompt per Claude Code's `UserPromptSubmit`
     decision-control semantics. The driver times out and
     `gaia recover` surfaces the session URL for diagnosis.
   - For `PreToolUse`: emit a `decision: "deny"` JSON output with
     the same reason on every tool call from a gaia session. The
     session cannot make progress and terminates; the driver times
     out the same way. This is preferable to fail-open — a gaia
     session running without protection is exactly what `PreToolUse`
     exists to prevent.
   - For `Stop` / `StopFailure`: exit 1 (non-blocking failure —
     the session terminates normally; the driver times out rather
     than exits 13). A tiny in-shim last-resort attempt pushes a
     `GAIA_FAILED: <session_id>` sentinel carrying only
     `{error: "gaia-hook-binary-missing-or-version-skew"}` if git
     is reachable from the shim's environment; this lets the driver
     exit 13 fast with a clear error instead of waiting for the
     timeout. This last-resort path is best-effort — it has no
     transcript, no last_assistant_message, and failure to push
     simply falls through to the timeout path.

This means a teammate who opens Claude Code on the repo without gaia
installed sees no disruption for their own (non-gaia) prompts: the
shim runs, classifies the session as non-gaia, exits 0. A gaia
session in the same environment would fail explicitly rather than
silently — which is what the driver wants.

**Protocol-version invariant.** The committed shim's protocol
version, the manifest's `driver_version` (§7.4), and the global
`gaia-hook-*` binary version must all agree per run. The driver
records its own version into the manifest at fire time; the shim
reads the manifest after fetching the state branch and refuses to
proceed if `manifest.driver_version` is incompatible with the
shim's pinned protocol version. The `UserPromptSubmit` hook is the
authoritative gate — `PreToolUse` and `Stop` rely on the
session-tmp file having been created earlier in the session, which
implies the gate already passed.

**Node is a prerequisite for the shim, not for Claude Code.** Claude
Code itself can run with no Node on `PATH` (native installer / Homebrew
/ WinGet ship a binary that does not invoke Node). The Node prereq is
specific to gaia-initialized repos because the committed shims are
Node scripts and the `command` entries start with `node`. If Node is
not on `PATH`, the hook command fails to start — disrupting the
session regardless of whether the prompt was gaia or not. §5
documents this as a prerequisite for any environment opening Claude
Code in a gaia-initialized repo. `gaia init` prints a banner so repo
owners can communicate this expectation. Cross-platform support
(Windows, macOS, Linux) is delivered by the Node runtime rather than
by the shim itself — no Git Bash / WSL requirement for teammates on
Windows. A future opt-in for POSIX `.sh` shims is tracked in §14.

`gaia doctor` (§8.4) includes checks that a gaia-authored prompt
fails closed when the binary is absent and that the version invariant
holds end-to-end.

### 8.1 `gaia-hook-user-prompt-submit`

Fires on every user prompt submission. Blocks until it returns.

**Input (stdin JSON, v1 known fields).**

```json
{
  "hook_event_name": "UserPromptSubmit",
  "session_id": "session_01HJK...",
  "transcript_path": "/home/user/.claude/projects/.../<uuid>.jsonl",
  "cwd": "/home/user/repo",
  "prompt": "...--- BEGIN GAIA CONTROL ---\nGAIA_RUN_ID: 01HXX...\n..."
}
```

**Behavior.** Every `git` invocation in this section runs with
`-C "$CLAUDE_PROJECT_DIR"` so Claude's working-directory changes and
any nested submodules do not redirect the command.

1. Scan the entire `prompt` for the
   `--- BEGIN GAIA CONTROL ---` … `--- END GAIA CONTROL ---` fence
   and parse `GAIA_RUN_ID`, `GAIA_WORK_BRANCH`, `GAIA_NONCE` from
   inside it. If the fence is absent, exit 0 (non-gaia session —
   hook no-ops). The state branch is derived internally as
   `claude/gaia/$GAIA_RUN_ID/state`; it is not sent in the
   payload (§7.2, §16.1).
2. Validate `GAIA_WORK_BRANCH` matches
   `^claude/gaia/[0-9A-HJKMNP-TV-Z]{26}/work$` (Crockford-base32
   ULID + fixed suffix) and that the embedded ulid matches
   `GAIA_RUN_ID`. Validate `GAIA_NONCE` is base64-decodable and 24
   bytes. On mismatch, emit
   `{"decision": "block", "reason": "gaia control-block validation failed"}`
   to stdout and exit 0 (Claude Code processes JSON decision output
   only on exit 0; this is the documented blocking path for
   `UserPromptSubmit`).
3. Fetch the derived state branch with an explicit refspec into a
   bare check dir (`.git/gaia-state-check`) and read
   `.gaia-state/manifest.json`:
   ```
   git -C "$CLAUDE_PROJECT_DIR" fetch origin \
     "+refs/heads/$GAIA_STATE_BRANCH:refs/remotes/origin/$GAIA_STATE_BRANCH"
   ```
   Refuse if the manifest is missing, if `manifest.nonce` does not
   match `GAIA_NONCE`, if `manifest.work_branch` does not match the
   prompt's `GAIA_WORK_BRANCH`, if `manifest.run_id` does not match
   `GAIA_RUN_ID`, or if `manifest.driver_version` is incompatible
   with this shim's pinned protocol version (§8.0). Refusal uses
   the same `{"decision": "block", "reason": ...}` path as step 2.
4. Write `<runtime-dir>/<session_id>.json` with
   `{run_id, work_branch, state_branch, experimental_mode,
   driver_version}`, where `<runtime-dir>` is `$XDG_RUNTIME_DIR/gaia/`
   when set and `/tmp/gaia-$uid/` otherwise. The directory is
   created with mode `0700`; the file is written with mode `0600`,
   `O_NOFOLLOW`, and `session_id` is rejected if it does not match
   `^session_[0-9A-Za-z_-]+$` (defense against path traversal). The
   tmp file is the **operational source of truth** for the `Stop`,
   `StopFailure`, and `PreToolUse` hooks during the session — they
   key by `session_id` and read branch names from this file rather
   than from any git config the agent could overwrite. The file is
   **not secret** and is not tamper-proof against a hostile in-VM
   actor with shell access (§16.1); the directory permissions and
   `O_NOFOLLOW` are about preventing accidental cross-user reads
   and symlink attacks, not about authenticating the agent.
5. **Convenience-only** side channel for Claude's own Bash tool
   reads: persist the *work branch* (only) to git config so prompts
   that read it via `git config --get gaia.workBranch` (see §9) work
   without hook-exported environment variables (current Claude Code
   does not inject hook env vars into the Bash tool's process for
   `UserPromptSubmit`):
   ```
   git -C "$CLAUDE_PROJECT_DIR" config gaia.runId       "$GAIA_RUN_ID"
   git -C "$CLAUDE_PROJECT_DIR" config gaia.workBranch  "$GAIA_WORK_BRANCH"
   ```
   The state branch is deliberately *not* written to git config —
   Claude does not need it, and keeping it out of the repo's
   model-readable surface keeps the §16.1 trust model intact. The
   experimental-mode flag (if present) is recorded in the per-
   session tmp file only, for the same reason.
6. Fetch the work branch with an explicit refspec and check it out
   as a real local branch matching the remote name:
   ```
   git -C "$CLAUDE_PROJECT_DIR" fetch origin \
     "+refs/heads/$GAIA_WORK_BRANCH:refs/remotes/origin/$GAIA_WORK_BRANCH"
   git -C "$CLAUDE_PROJECT_DIR" checkout -B "$GAIA_WORK_BRANCH" \
     "refs/remotes/origin/$GAIA_WORK_BRANCH"
   ```
   Local-branch name matches the remote target so the Claude Code
   GitHub proxy's "push to current working branch" check (§8.2 step
   7) is satisfied for any subsequent push from this branch. After
   checkout, `.claude/hooks/gaia-*.js` and `.claude/settings.json`
   on the worktree are guaranteed by the driver's synthesis step
   (§6.1 step 2, §8.8).
7. If the payload contains `--- CONTEXT FILE ---`, fetch the state
   branch, copy `.gaia-state/context.md` into the worktree at
   `.gaia/runs/<run_id>/context.md` (the `.gaia/` rule in `.gitignore`
   keeps it out of any commit), and emit `additionalContext` pointing
   Claude at the file:
   ```json
   {
     "hookSpecificOutput": {
       "hookEventName": "UserPromptSubmit",
       "additionalContext": "The user prompt references .gaia/runs/<run_id>/context.md. Read that file before acting — it holds the task content that did not fit inline."
     }
   }
   ```
   Claude Code injects `additionalContext` as a system reminder
   alongside the original prompt. No prompt rewriting is attempted
   (`updatedPrompt` is not in the documented `UserPromptSubmit`
   output surface); the `--- CONTEXT FILE ---` marker remains visible
   in the prompt as a human-readable crumb.
8. Exit 0.

**Exit-code semantics.** Claude Code processes `UserPromptSubmit`
JSON output only on exit 0; exit 2 blocks the prompt with stderr
surfaced as feedback; other non-zero exits are non-blocking errors.
gaia's validation-failure path (steps 2–3) uses
`{"decision": "block", "reason": "..."}` + exit 0 because it wants
the failure to surface as explicit feedback rather than as an
unstructured error. Any unexpected exception (git failure, manifest
parse error) exits 2 so Claude Code blocks the prompt and stderr
carries the cause. The driver hits no-sentinel timeout (exit 3)
and surfaces the session URL for `gaia recover`.

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

**Behavior.** Every `git` invocation in this section runs with
`-C "$CLAUDE_PROJECT_DIR"` (or, for worktree-scoped operations,
`-C "$tmp"`). Claude may have `cd`d into a subdirectory or even a
nested git repo during the session; bare `git status` / `git push`
would silently hit the wrong repo.

1. Look up `<runtime-dir>/<session_id>.json` (§8.1 step 4). If
   absent, exit 0 (non-gaia session).
2. Read `work_branch`, `state_branch`, `run_id`, and
   `experimental_mode` directly from the JSON loaded in step 1. The
   session-scoped tmp file is the **operational source of truth**
   for these values — it was written by the `UserPromptSubmit` hook
   in the same session and is the only path that survives the
   agent overwriting `git config` or other repo state. It is not
   secret or tamper-proof against a hostile in-VM actor (§16.1);
   the trust model treats the prompt boundary as the trust
   boundary, and the `PreToolUse` hook (§8.5) blocks edits to the
   shim files and to `.gaia-state/`.
3. **Verify the current branch is the work branch.** If not,
   attempt `git -C "$cd" checkout -B "$GAIA_WORK_BRANCH"
   "refs/remotes/origin/$GAIA_WORK_BRANCH"` and re-verify with
   `git -C "$cd" symbolic-ref --short HEAD`. If the work branch
   cannot be re-acquired (Claude may have left the repo in an
   unexpected state — detached HEAD, missing branch ref), skip the
   final sweep, fall through to step 6 to push a `GAIA_FAILED:
   <session_id>` sentinel carrying
   `{error: "work-branch-not-current"}`, and exit 1. The
   `PreToolUse` hook (§8.5) blocks `git checkout` / `git switch` /
   `git reset --hard` for gaia sessions, so this fallback is
   defense-in-depth rather than a routine path.
4. **Final commit sweep (conditional) then push (unconditional).**
   Root every command at `$CLAUDE_PROJECT_DIR` and explicitly exclude
   `.gaia/` from staging — the agent may have modified `.gitignore`
   during the session, and the belt-and-suspenders pathspec excludes
   `.gaia/` regardless.
   ```bash
   cd="$CLAUDE_PROJECT_DIR"
   if [ -n "$(git -C "$cd" status --porcelain -- ':!.gaia')" ]; then
     git -C "$cd" add -A -- ':!.gaia'
     git -C "$cd" commit -m "gaia: final sweep"
   fi
   git -C "$cd" push origin "$GAIA_WORK_BRANCH:$GAIA_WORK_BRANCH"
   ```
   The push is **unconditional**. Earlier drafts only pushed when the
   working tree was dirty, which dropped clean local commits Claude
   had created but not pushed — those are now always synced to
   origin. The push uses an explicit `<src>:<dst>` refspec so the
   Claude Code GitHub proxy's "push to current working branch" rule
   sees a name-matched refspec from the same-named local branch
   (verified in step 3).
5. **Extract the final assistant message** from the stdin
   `last_assistant_message` field. If missing or empty, fall back to a
   reverse-scan of the JSONL transcript (looking for the last line
   where `message.role === "assistant"` or `type === "assistant"`,
   concatenating text blocks). Last-resort fallback: the raw last line
   of the JSONL — still human-readable and the driver will print it.
6. **Build the readable transcript** by one forward pass through the
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
7. **Write both artifacts** via a temporary worktree seeded at the
   state-branch tip as a **real local branch with the same name as
   the remote target**, then push with an explicit name-matched
   refspec. Claude Code's GitHub proxy "restricts git push
   operations to the current working branch for safety"; using a
   different local name (e.g., `gaia-state-$GAIA_RUN_ID`) and
   pushing to a differently-named remote ref is exactly the case
   the proxy is designed to reject. The `-B "$GAIA_STATE_BRANCH"`
   on `git worktree add` plus an explicit
   `<src>:<dst>` refspec where `src == dst` removes the ambiguity:
   ```bash
   cd="$CLAUDE_PROJECT_DIR"
   tmp=$(mktemp -d)
   git -C "$cd" fetch origin \
     "+refs/heads/$GAIA_STATE_BRANCH:refs/remotes/origin/$GAIA_STATE_BRANCH"
   git -C "$cd" worktree add -B "$GAIA_STATE_BRANCH" "$tmp" \
     "refs/remotes/origin/$GAIA_STATE_BRANCH"
   mkdir -p "$tmp/.gaia-state"
   printf '%s\n' "$final_message"   > "$tmp/.gaia-state/$session_id.md"
   printf '%s\n' "$transcript_text" > "$tmp/.gaia-state/$session_id.transcript.md"
   git -C "$tmp" add .gaia-state
   git -C "$tmp" commit -m "GAIA_DONE: $session_id"
   git -C "$tmp" push origin "$GAIA_STATE_BRANCH:$GAIA_STATE_BRANCH"
   git -C "$cd" worktree remove "$tmp"
   ```
   The local branch name *is* the remote branch name; no separate
   `branch -D` step is needed because `worktree remove` cleans up
   the worktree without deleting the now-current branch ref (and
   leaving the branch ref behind is harmless — it is gaia's own
   namespace).

   If the push is rejected as non-fast-forward (another writer has
   advanced the state branch since the fetch), the hook refetches
   `origin/$GAIA_STATE_BRANCH`, resets the local
   `$GAIA_STATE_BRANCH` branch to the new tip, re-applies the
   two `.gaia-state/*` writes, re-commits, and retries the push.
   Bounded at 3 retries. `gaia doctor` (§8.4) verifies this exact
   write path from inside a cloud session as a release gate — if
   the proxy rejects state-branch writes on this repo, doctor fails
   loudly and gaia is unusable on this repo until the proxy
   permissions are widened.
8. Exit 0.

The sentinel push is the only way the driver learns the session is
done; the hook must not exit before completing step 7.

**Exit-code semantics.** `Stop` processes JSON decision output on
exit 0 (see §15.2 for the experimental continuation path that uses
`{"decision": "block"}`). Exit 2 is documented to **prevent Claude
from stopping** — dangerous for the stable Stop hook because it
keeps the VM alive past the intended termination and blocks
finalization. The stable Stop hook therefore exits:
- `0` on full success (sentinel pushed);
- `1` on any failure after the final commit sweep — the session
  still terminates and the driver times out at exit 3, which is
  the intended recovery path via `gaia recover`. Never exit 2 from
  the stable Stop path.

### 8.3 `gaia-hook-stop` (StopFailure event)

When `hook_event_name === "StopFailure"`, stdin includes `error`,
`error_details`, and `last_assistant_message` fields. Claude Code
**ignores** `StopFailure` hook output and exit codes for
decision-control purposes — the session is already failing; the hook
cannot steer it. Side effects (file writes, git pushes) still run,
so gaia relies on those to surface the failure sentinel.

The hook:

1. Looks up `<runtime-dir>/<session_id>.json` (§8.1 step 4). No-op
   if absent.
2. Best-effort attempt at the §8.2 step 3 current-branch check and
   the §8.2 step 4 final-sweep + work-branch push. The API error
   may have damaged the VM state; failures here are logged but do
   not block the failure sentinel.
3. Writes `.gaia-state/<session_id>.error.json` containing
   `{error, error_details, last_assistant_message}` and
   `.gaia-state/<session_id>.transcript.md` (whatever transcript
   content exists) to the state branch via the same name-matched
   local-branch temporary-worktree flow as §8.2 step 7. The
   worktree's local branch is `$GAIA_STATE_BRANCH` (matching the
   remote name) and the push uses an explicit
   `$GAIA_STATE_BRANCH:$GAIA_STATE_BRANCH` refspec. Retry on
   non-fast-forward, bounded at 3 attempts.
4. Commits with message `GAIA_FAILED: <session_id>` and pushes.
5. Exits 0 on success, exits 1 on any error. The exit code does not
   change Claude Code's behavior, but a non-zero exit is logged
   and surfaces in the session's stderr capture.

The driver reads `GAIA_FAILED:` and exits 13, printing the error body
to stderr. The failed run is recorded with
`meta.json.status = "failed"` and a terminal `"failed"` event is
appended to `history.jsonl` so `gaia history` surfaces it and
`gaia recover` can act on it.

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

1. **Managed-policy / hook-honored check.** Project-level hooks fire
   on this Claude Code installation. Enterprise installs can set
   `allowManagedHooksOnly` (or equivalent) and silently disable
   `.claude/hooks/*`; doctor probes by registering a trivial test
   hook in a temp directory and confirming it fires. If hooks are
   disabled by managed policy, doctor fails loudly — gaia cannot
   work in that environment. Run on both the driver machine (so the
   teammate-local Claude Code surface works) and the cloud VM (so
   the gaia protocol works at all).
2. Spawner config is valid: 200 on `/fire`, correct response field
   names (`claude_code_session_id`, `claude_code_session_url`).
3. Cloud repository access is configured and the session can push to
   both `claude/gaia/<doctor_run_id>/work` **and**
   `claude/gaia/<doctor_run_id>/state` from inside the VM. This is
   the release gate for §8.2 step 7's name-matched local-branch
   state push — if the proxy rejects the state-branch write (e.g.,
   the repo has narrower `claude/*` permissions than the default,
   or the proxy's "current working branch" check rejects the
   write), doctor fails with the exact remediation and gaia is
   unusable on this repo until it is fixed.
4. **`UserPromptSubmit.prompt` content shape.** The doctor's `/fire`
   includes a unique fingerprint inside the `--- BEGIN GAIA CONTROL
   ---` block. The doctor hook records the *full* `prompt` field it
   receives and pushes a sample to a doctor-only state ref so the
   driver can verify whether Claude Code prepended the saved routine
   prompt, appended it, wrapped it, or passed `text` through
   verbatim. The control-block scanner in §8.0 step 2 is robust to
   all four orderings; this check confirms the assumption holds on
   the current Claude Code version rather than discovering a
   regression in production.
5. The work branch the session sees after `UserPromptSubmit` checkout
   contains `.claude/hooks/gaia-stop.js`, `.claude/hooks/
   gaia-pre-tool-use.js`, and `.claude/settings.json` at the current
   driver's expected version. Catches broken `gaia init` states and
   confirms §6.1 step 2's synthesis commit took effect.
6. **Protocol-version invariant.** The committed shim, the global
   `gaia-hook-*` binary on the driver machine, and the global
   binary on the VM all advertise the same protocol version. The
   driver records the manifest's `driver_version` and confirms the
   hook tmp file's `driver_version` matches. Catches the cloud
   environment-cache skew problem: setup scripts cache for ~7 days,
   so a freshly published gaia version may not reach the VM until
   the cache rolls over. The `UserPromptSubmit` shim's fail-closed
   path (§8.0 step 6) catches this in production; doctor catches it
   before any user run does.
7. The `UserPromptSubmit` shim **fails closed** when the global
   `gaia-hook-user-prompt-submit` binary is absent or version-
   skewed: a gaia-authored prompt emits a `{"decision": "block"}`
   output and the session does not proceed, while a non-gaia
   prompt in the same environment exits 0 silently. Doctor
   simulates the binary-missing and version-skew states by
   temporarily shadowing `$PATH`.
8. The `UserPromptSubmit` hook receives the expected stdin fields.
9. `last_assistant_message` is present on the `Stop` input on this
   Claude Code version.
10. The JSONL transcript on this version parses into the expected
    forward-pass output.
11. `StopFailure` is a recognized event (empirically: issue a known-
    bad prompt such that the routine rejects it and verify the hook
    fires with the failure fields).
12. **`PreToolUse` denials work end-to-end.** Doctor fires a session
    whose prompt instructs Claude to attempt one operation from
    each gaia-protected category (edit `.claude/hooks/gaia-stop.js`,
    `git checkout main`, `git push origin claude/gaia/<id>/state`)
    and confirms the `PreToolUse` shim's `decision: "deny"`
    response is honored — Claude does not perform the operation —
    and that the session can complete normally otherwise. If any
    denial is bypassed, the protection in §8.5 is not load-bearing
    on this Claude Code version.
13. No foreign project-level handler has appeared on
    `UserPromptSubmit`, `Stop`, or `StopFailure` since install
    (§8.7). For `PreToolUse`, doctor checks for matcher overlap
    with gaia's protected matchers and warns rather than fails.
14. (If `--experimental-mode` is configured) the experimental-mode
    flow from §15.7 completes.

`gaia init` offers to run `gaia doctor` automatically before declaring
setup complete.

### 8.5 `gaia-hook-pre-tool-use` (PreToolUse event)

Fires before every tool call from a session. For non-gaia sessions
the shim exits 0 immediately and the call proceeds normally. For
gaia sessions the shim **denies** any tool call that would mutate
gaia-managed protocol state, and **allows** everything else.

**Why this exists.** The `Stop` and `StopFailure` shims execute
`$CLAUDE_PROJECT_DIR/.claude/hooks/gaia-stop.js` from the *work
branch's* working tree (§8.8). Without `PreToolUse` enforcement,
Claude could `Edit` or `Write` over `gaia-stop.js`, replace the
finalizer with a no-op, and the protocol would silently break — the
driver would time out at 45 min with no useful sentinel. Similarly,
nothing structural prevents Claude from `git checkout main`-ing
out of the work branch mid-session, or pushing a forged sentinel
to `claude/gaia/<id>/state`. The saved routine prompt (§9) tells a
well-behaved session not to do these things; `PreToolUse` enforces
it for sessions that drift, ignore the prompt, or are prompt-
injected from repo content (§16.1).

**Input (stdin JSON, v1 known fields).**

```json
{
  "hook_event_name": "PreToolUse",
  "session_id": "session_01HJK...",
  "tool_name": "Edit",
  "tool_input": { "file_path": "...", "...": "..." }
}
```

**Behavior.**

1. Look up `<runtime-dir>/<session_id>.json` (§8.1 step 4). If
   absent, exit 0 (non-gaia session — every tool call is allowed).
2. Read `tool_name` and `tool_input` from stdin. Apply the
   protected-paths and protected-commands rules below. Resolve
   user-supplied paths against `$CLAUDE_PROJECT_DIR` and reject any
   path that escapes the project root or that resolves through a
   symlink to a protected location.
3. **Protected paths.** Any tool call (`Edit`, `Write`, `MultiEdit`,
   `NotebookEdit`, or `Bash` with a write-like subcommand —
   `rm`, `mv`, `cp`, `tee`, `truncate`, `chmod`, redirect-to,
   `dd of=`, `git rm`, etc.) that targets one of the following
   paths is denied:
   - `.claude/hooks/gaia-*.js`
   - `.claude/settings.json` — the file as a whole; the rule is
     paranoid because partial edits could disable individual hooks
   - `.gitignore` — only edits that remove or alter the `.gaia/`
     line; other `.gitignore` edits are allowed (Claude may
     legitimately add ignore rules during development)
   - `.gaia/**` (driver-local state; should not exist in the VM but
     is locked down for safety)
   - `.git/**`
4. **Protected commands.** `Bash` calls whose decoded subcommand
   matches one of these patterns are denied (the shim parses the
   `tool_input.command` as a shell argv; the patterns match the
   first non-env command and apply to chained subshells too):
   - `git checkout *`, `git switch *`, `git restore --source=* *`,
     `git reset --hard *`, `git reset --merge *`
   - `git branch -D *`, `git branch -d *` against any
     `claude/gaia/*` ref
   - `git push * claude/gaia/*/state*` (any push that targets a
     state branch)
   - `git push -f` / `git push --force*` (any force push,
     regardless of target — gaia never force-pushes; cf. §6.1.1)
   - `git config --unset gaia.*` / `git config --remove-section gaia`
   - Any command containing `git update-ref` or `git
     symbolic-ref` against a gaia ref
5. **Allow-list overrides.** Claude must still be able to commit on
   the work branch, push the work branch (the Stop hook does this,
   but agents may also push intermediate progress), read any of the
   protected files, and run `git status` / `git log` / `git diff`
   freely. The shim only blocks *write* and *branch-switch*
   operations — read paths are not in the protected set.
6. **Decision output.** When denying, emit:
   ```json
   {
     "decision": "deny",
     "reason": "gaia: <path or command> is gaia-managed; the saved routine prompt explains why this branch / file is off-limits"
   }
   ```
   and exit 0. Claude Code processes `PreToolUse` decision output
   on exit 0; the tool call is rejected and Claude sees the
   `reason` as feedback. When allowing, exit 0 with no decision
   object.

**Coexistence with other `PreToolUse` handlers.** `PreToolUse`
accepts per-tool matchers, so multiple project-level handlers can
coexist without races as long as their matchers are disjoint.
gaia's matcher set is documented in §8 (the `.claude/settings.json`
block above): `Edit|Write|MultiEdit|Bash|NotebookEdit`. Other
tools (e.g., `Read`, `Glob`, `Grep`, `Task`) are unmatched by gaia
and can have their own handlers without conflict. If a foreign
handler matches one of gaia's tools, doctor warns at §8.4 step 13.

**Failure modes.** If the shim crashes or times out, Claude Code's
default `PreToolUse` behavior is to allow the call (fail-open). gaia
explicitly does *not* try to convert this to fail-closed for non-
protected calls — failing closed on `Read` or `Glob` would brick
the session for unrelated reasons. For *protected* calls, the
binary-missing path in §8.0 step 6 fail-closes with
`decision: "deny"`. Combined, the gaia-protected surface fails
closed; the rest fails open as Claude Code defaults.

### 8.6 Why hooks and not a session CLI

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
- `PreToolUse` (§8.5) lets gaia enforce its protocol invariants
  against the agent's tool calls without depending on the agent
  voluntarily abstaining.

### 8.7 Hook coexistence policy

`UserPromptSubmit`, `PreToolUse`, `Stop`, and `StopFailure` are
the four events gaia registers handlers for. The three exclusive
events (`UserPromptSubmit`, `Stop`, `StopFailure`) cannot share
the project-settings layer with other handlers safely:

- `UserPromptSubmit` and `Stop` accept no matchers at all.
- `StopFailure` accepts an `error_type` matcher but no per-session
  or per-run matcher gaia could use to scope itself, so a foreign
  `StopFailure` handler still fires alongside gaia's on every API
  failure.
- All registered handlers on a given event fire in parallel, and
  when multiple handlers return decision objects the most
  restrictive wins. A sibling hook could block gaia's
  session-start, return a conflicting `decision` on `Stop`, or
  race gaia's worktree mutations.

`PreToolUse` is different: it accepts per-tool matchers, so
multiple handlers with disjoint matcher sets coexist cleanly.
gaia's matcher (`Edit|Write|MultiEdit|Bash|NotebookEdit`; §8.5) is
narrow enough that a project can register its own `PreToolUse`
handlers on other tools without conflict. Even within gaia's
matcher set, parallel handlers can coexist as long as none of them
denies a tool call gaia would have allowed (the most-restrictive
rule means denial wins, which is generally desirable for security
hooks).

**v1 policy: gaia owns the three exclusive events at the
project-settings layer, per repo, and shares `PreToolUse` with
disjoint-matcher handlers.** This is narrower than "gaia owns the
events globally." Claude Code composes hooks from multiple scopes —
user settings (`~/.claude/settings.json`), project settings
(`.claude/settings.json`), project-local settings (not loaded in
routines), managed policy settings, plugin and skill scopes, and
agent scopes. `gaia init` and `gaia doctor` can only see and
control the *project-settings* layer. Managed policy or plugin-
level handlers on any of the four gaia events may still run
alongside gaia's and interfere with it; gaia cannot prevent that
from inside the repo.

- `gaia init` reads the existing `.claude/settings.json` (if any) and
  scans for pre-existing handlers on `UserPromptSubmit`, `Stop`, or
  `StopFailure`. If it finds any handler it did not write itself (see
  detection below), it aborts before modifying any file. The printed
  error:
  - Names the conflicting event(s) and the first handler's `command`.
  - Offers three remediations: (1) remove the pre-existing handler;
    (2) inline whatever it does into the gaia shim yourself and rerun
    `gaia init --force-merge-hooks` (flag reserved for v1.x, not
    implemented in v1 — the message spells that out); (3) do not
    install gaia in this repo.
  - Exits non-zero.
- For `PreToolUse`, `gaia init` is more permissive: it merges its
  matcher entry alongside any pre-existing matcher entries. A
  pre-existing `PreToolUse` handler whose matcher *overlaps* with
  gaia's (`Edit|Write|MultiEdit|Bash|NotebookEdit`) prints a
  warning naming the overlap but does not block install — the
  most-restrictive rule means a denial from either handler is
  honored, which is the desired security composition. A
  pre-existing `PreToolUse` handler whose matcher is disjoint from
  gaia's installs cleanly with no warning.
- A handler is recognized as gaia's own if its `command` references
  `.claude/hooks/gaia-user-prompt-submit.js`,
  `.claude/hooks/gaia-pre-tool-use.js`, or
  `.claude/hooks/gaia-stop.js` (with or without the
  `$CLAUDE_PROJECT_DIR` prefix). `gaia init` treats these as safe
  to replace on rerun.
- On repos that have never registered any of the three exclusive
  events at the project layer, `gaia init` writes the block
  described in §8 wholesale.
- `gaia doctor` re-runs the same project-layer scan on every
  invocation and warns if a foreign handler appears on any of the
  three exclusive events post-install (e.g., another tool added one
  later); the run does not fail, but the warning surfaces the risk.
  For `PreToolUse`, doctor warns only on matcher overlap. Doctor
  **cannot reliably detect** managed-policy or plugin-level handlers
  from inside the repo and is explicit about that limitation — if
  an enterprise configuration or plugin attaches a `Stop` hook to
  every session, gaia's protocol may race or be overridden without
  visible warning.

**What this does not block.** `PostToolUse`, `SubagentStop`,
`Notification`, and any other event outside the four gaia owns are
unaffected. `PreToolUse` is shared, not exclusive (per the bullets
above). Repos can freely use any of these alongside gaia.

**When this policy will relax.** Composition is the right long-term
answer — a single shim dispatcher that chains to pre-existing
handlers when the session is not a gaia session, and runs gaia logic
otherwise. That design is deferred to v1.x (§14) because it is
non-breaking: today's refusal becomes tomorrow's success, and the
installed settings layout does not need to change to accommodate it.

### 8.8 Work-branch hook invariant

`UserPromptSubmit` runs against the default-branch clone and
immediately checks out the work branch. Every subsequent hook the
session fires — `PreToolUse`, `Stop`, `StopFailure`, and any
future event — runs
`$CLAUDE_PROJECT_DIR/.claude/hooks/gaia-*.js` resolved against
the **work branch's** working tree, not the default branch's. If
the work branch (seeded from the caller's user branch) does not
contain the expected hook shim files, the protection hook never
runs, the Stop hook never runs, no sentinel is pushed, and the
driver times out at 45 minutes with a symptom that looks like
network failure — and the session ran the entire time without
`PreToolUse` enforcement on gaia-managed files.

Because users run gaia from whichever feature branch they happen to
be on, the user branch often predates `gaia init` and is missing
`.claude/hooks/gaia-*.js`, the settings entries for the four gaia
events, or the `.gaia/` ignore rule — or has out-of-date versions.

**Invariant enforced by the driver** (§6.1 step 2): before `/fire`,
the work branch must contain the gaia-managed files at exactly the
versions the current driver expects. The driver enforces this by
synthesizing a setup commit on the work branch:

1. In a temporary worktree on the local
   `claude/gaia/$RUN_ID/work` branch (created with
   `git worktree add -B "$GAIA_WORK_BRANCH" "$tmp"
   "refs/remotes/origin/$GAIA_WORK_BRANCH"` — name-matched per
   §8.2 step 7), read each gaia-managed file from
   `refs/remotes/origin/<default-branch>` (fetched with an
   explicit refspec):
   - `.claude/hooks/gaia-user-prompt-submit.js`
   - `.claude/hooks/gaia-pre-tool-use.js`
   - `.claude/hooks/gaia-stop.js`
   - `.claude/settings.json` — only the four gaia event entries
     are reconciled (merged non-destructively with existing
     project-layer configuration for other events; §8.7)
   - `.gitignore` entry for `.gaia/` (appended if missing,
     existing line left alone if already present)
2. If any file differs from the work-branch copy, write the
   default-branch version into the work-branch tree and stage it.
3. If anything was staged, commit with message
   `gaia: ensure hook shims on work branch` and push. If the tree
   was already clean against the default branch, skip the commit.

The commit flows back to the user branch via the standard merge-
back in §6.1.1, so after the first successful gaia run on a stale
feature branch, that branch carries the latest hook files and
future runs skip the synthesis step.

**What the driver does not synthesize.** If `gaia init` has never
run on this repo (default branch lacks the files), synthesis has
nothing to copy from and exits 2 with a clear error before firing.
`gaia init` is still a prerequisite; §8.8 only rescues stale
feature branches, not un-initialized repos.

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

**Never inspect, checkout, commit to, or push any branch matching
the pattern `claude/gaia/*/state`.** Those branches are gaia's
internal IPC transport; writing to them would corrupt the protocol.
The hook will write to them on your behalf at the appropriate time.
Never emit commits whose messages begin with GAIA_DONE:,
GAIA_FAILED:, GAIA_ABORTED:, GAIA_PENDING:, or GAIA_INPUT: on any
branch — those are gaia's protocol sentinels.

A protection hook (PreToolUse) will deny attempts to edit
.claude/hooks/gaia-*.js, .claude/settings.json, the .gitignore
".gaia/" rule, .git/**, or to run `git checkout`, `git switch`,
`git reset --hard`, force pushes, or pushes to claude/gaia/*/state.
If you see a denial from that hook, do not try to work around it —
it means you are touching gaia-managed state that is supposed to be
off-limits for this session. Read the denial reason and pick a
different approach.

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

The "do not touch state branches" and "do not emit GAIA_* commits"
instructions are defense-in-depth per §16.1. gaia cannot
cryptographically authenticate sentinel commits against a hostile
agent given the VM's shared credentials, so the prompt instructs
well-behaved sessions away from the protocol ref names and the
driver validates sentinel artifacts (§6.1 step 6) as the
belt-and-suspenders check.

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
  history.jsonl                                append-only terminal-event log for -c / --resume
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
`history.jsonl` is a derived append-only event log — one record per
**terminal** state transition per run. If the two disagree,
`meta.json` wins; gaia offers to rebuild `history.jsonl` from the
runs directory.

`meta.json` schema:

```json
{
  "run_id": "01HXX...",
  "session_id": "session_01HJK..." | null,
  "session_url": "https://claude.ai/code/sessions/session_01HJK..." | null,
  "status": "pending" | "success" | "no-merge" | "failed" | "fire-failed" | "timeout" | "push-rejected" | "conflict" | "aborted",
  "user_branch": "feature/auth",
  "work_branch": "claude/gaia/01HXX.../work",
  "state_branch": "claude/gaia/01HXX.../state",
  "pushed_upstream": true,
  "base_sha": "abc1234",
  "merge_sha": "def5678",
  "continued_from": "session_01HAA...",
  "driver_version": "1.0.0",
  "started_at": "2026-04-21T10:30:00Z",
  "finished_at": "2026-04-21T10:35:12Z",
  "prompt_first_line": "refactor the auth module...",
  "error": null
}
```

`meta.json` is rewritten under an exclusive in-process lock on
`.gaia/runs/<run_id>/.lock` during a run — a fire path transitions
it through `pending` → terminal. The `state_branch` field is
internal bookkeeping only and is not consumed by any code path
that runs in the cloud VM (§7.2, §16.1).

**Statuses.** `pending` is the only non-terminal status; everything
else is terminal. `fire-failed` represents the case where `/fire`
rejected the request before any session was created (§6.1 step 5):
`session_id` and `session_url` are `null`, the work and state
branches were deleted from origin (no recovery is possible without
a session), and `gaia history` lists the run with the recorded
error so it does not silently disappear. `pushed_upstream` is
`false` for local-only user branches that skipped the upstream
push step (§6.1.1 step 6).

`history.jsonl` — one JSON object per line, a flat-summary view
appended **only when a run reaches a terminal status**:

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

**Write model.** The driver writes history **exactly once per run
per terminal transition**. No mid-run mutation of existing history
records. Terminal transitions that can each produce one history
event are: `success`, `no-merge`, `failed`, `timeout`,
`conflict`, `push-rejected`, `aborted`. A stuck `pending` run
therefore has no history entry until recovery finalizes it — that
is deliberate so that crashed/abandoned runs never look like
successful continuation targets to `-c`.

If a run transitions again (e.g., `gaia recover` promotes a
`push-rejected` run whose push eventually succeeded, or lands an
`aborted` run via `--merge`), the driver appends a **new** event
with the new terminal status. Readers treat the **last record for
each run_id** as authoritative. `gaia history` surfaces all
events; `-c` considers only the most recent `"success"` event per
user_branch.

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
- HTTP error mapping (all of these surface as `meta.status =
  "fire-failed"` per §10.3 because no session is created):
  | Code | `error.type` | gaia behavior |
  |---|---|---|
  | 400 | `invalid_request_error` (text too large, paused, malformed) | Non-retryable. Exit 12. |
  | 401 | `authentication_error` | Non-retryable. Exit 10. |
  | 403 | `permission_error` (entitlement) | Non-retryable. Exit 10. |
  | 404 | `not_found_error` (routine id wrong / deleted) | Non-retryable. Exit 12. |
  | 429 | `rate_limit_error` | Honor `Retry-After`. After retries exhausted, exit 11. |
  | 500 | `api_error` | Exponential backoff up to 3 attempts. Then exit 12. |
  | 503 | `overloaded_error` | Exponential backoff up to 3 attempts. Then exit 12. |
  | network failure (DNS, TCP RST, TLS) | n/a | Exponential backoff up to 3 attempts. Then exit 5. |
- `maxPayloadChars: 65536`, `defaultTimeout: "45m"`.
- The endpoint returns immediately after session creation — it does
  not stream or wait. gaia's sentinel / polling design is the intended
  completion transport.

## 12. Failure modes

| Failure | Detection | Policy |
|---|---|---|
| `-p` missing | Arg parsing | Exit 2 with usage. |
| Spawner HTTP error before session creation | `fire()` rejects with non-2xx or network error | Retry per §11.2 where applicable. On exhaustion, transition `meta.status = "fire-failed"` (`session_id: null`), append a `"fire-failed"` event to `history.jsonl`, delete the freshly-pushed work / state branches from origin (no session ever ran), exit 5/10/11/12 per §11.2. |
| `GAIA_FAILED:` sentinel received | Poll loop sees failure sentinel | Exit 13. Error body printed. `meta.json.status = "failed"`. Terminal `"failed"` event appended to `history.jsonl`. `gaia recover` offered. |
| `GAIA_ABORTED:` sentinel received (experimental mode) | Poll loop sees abort sentinel | Exit 0. `meta.json.status = "aborted"`. Terminal `"aborted"` event appended. No merge performed; work branch preserved on origin. `gaia recover <id> --merge` can land the changes later. |
| `GAIA_DONE:` sentinel missing expected artifacts | `GAIA_DONE:` commit lacks `<session_id>.md` or `<session_id>.transcript.md` | Driver treats as suspect (potential forgery per §16.1), logs a warning, continues polling the remaining timeout. If nothing valid arrives, falls through to the timeout path (exit 3). |
| No sentinel by timeout | Poll loop deadline | Exit 3. Work + state branches preserved. Session URL printed. `meta.json.status = "timeout"`. Recoverable via `gaia recover`. |
| Merge conflict | `git merge` fails | Abort merge. Exit 4. Work branch preserved; user branch untouched. `meta.json.status = "conflict"`. Recoverable via `gaia recover`. |
| Push rejected after local merge | `git push origin <user-branch>` fails | Exit 15. Local user branch has the merge commit. `meta.json.status = "push-rejected"`. Printed instructions: investigate upstream advance, then `git push` when ready — or run `gaia recover <id>` which retries the push without re-merging (§6.4). |
| User branch diverged non-FF at merge time | `git merge-base --is-ancestor` fails pre-merge | Exit 2 with recovery message. |
| Dirty working tree at start | `git status --porcelain` non-empty | Exit 2. |
| Detached HEAD at start | No symbolic ref | Exit 2. |
| `UserPromptSubmit` hook fails | Hook returns `{"decision": "block"}` or exits 2 | Session blocked by Claude Code. Driver hits no-sentinel timeout; session URL surfaced. |
| `gaia-hook-*` binary missing or version-skewed in gaia session (shim fail-closed) | Global binary not on `$PATH`, or its protocol version disagrees with the committed shim's (§8.0) | `UserPromptSubmit` shim blocks with a clear reason. `PreToolUse` shim denies every tool call with the same reason. Stop/StopFailure shim attempts a best-effort `GAIA_FAILED:` push with `{error: "gaia-hook-binary-missing-or-version-skew"}` then exits 1. Driver exits 13 (fast) or 3 (timeout fallback). |
| `PreToolUse` denial (gaia-protected file or command) | gaia shim returns `{decision: "deny"}` | Tool call rejected; Claude sees the reason and continues. No driver-visible failure; the session terminates normally and finalization runs. |
| Work branch missing hook shim files | Default-branch hooks present but user branch stale (§8.8) | Driver synthesizes `gaia: ensure hook shims on work branch` commit before firing. If the default branch itself lacks the files (gaia init never run), driver exits 2 with a clear error. |
| `Stop` hook fails before sentinel push | No sentinel on state branch | Hook exits 1 (never 2; §8.2). Session terminates; driver times out at 45 min. Any pre-failure pushed commits on the work branch are preserved. `gaia recover` surfaces the session URL. |
| `StopFailure` hook fails | No `GAIA_FAILED:` commit | Side effects best-effort; output and exit code ignored by Claude Code (§8.3). Driver times out (exit 3) instead of exiting 13. Same recovery path. |
| Transcript parse fails (final message) | `last_assistant_message` empty + JSONL shape unknown | Fallback chain: `last_assistant_message` → reverse-scan JSONL → raw last line. Run still succeeds; stdout is whatever the fallback produced. |
| Transcript parse fails (readable transcript) | Same | `.gaia-state/<session_id>.transcript.md` contains the raw JSONL; continuation still works (Claude can read it) with higher token cost. |
| Duplicate sentinel | Two `GAIA_DONE:` commits for the same session_id | Earliest wins. Warning logged. |
| `-c` cannot continue: most recent terminal event on branch is non-success, or no terminal events at all | Per §6.2 — the latest terminal entry in `history.jsonl` for this `user_branch` is anything but `"success"`, or the branch has no terminal entries | Exit 2 naming the offending run (status, run_id, session_id) and suggesting `gaia recover` or `--resume <session_id>` for an older successful run. |
| `--resume <session_id>` unknown | No matching entry | Exit 2. |
| `--resume` of a run whose code did not merge | Prior `meta.status` ∈ {no-merge, conflict, timeout, aborted, push-rejected} and current HEAD does not contain the prior work-branch tip | Exit 2 unless `--resume-seed-from-work` is passed (§6.2). Message points at `gaia recover <run_id>` or the flag as the two supported paths. |
| Continuation transcript missing | `.gaia/transcripts/<session_id>.md` absent | Exit 2 with a message. Can happen if the transcripts directory was deleted manually. |

### 12.1 Cleanup

Default: delete `claude/gaia/$RUN_ID/{work,state}` from origin
immediately after a successful merge. Transcripts have already been
copied locally to `.gaia/transcripts/` at step 8 of §6.1, so the state
branch is disposable at that point.

Exceptions (branches preserved on origin):

- `--keep-branches`.
- `--no-merge` — branches are kept because the caller explicitly
  deferred the merge; `gaia recover` will complete it later.
- `GAIA_ABORTED:` sentinel received (experimental mode) — the user
  aborted and may still want to `gaia recover --merge` later. Exit
  code is 0 but branches are preserved.
- Any non-zero exit (3, 4, 13, 15) — `gaia recover` needs the
  branches to exist.

`gaia gc` sweeps `claude/gaia/*` branches on origin whose **age**
exceeds `--max-age` (default 14d). Branch tip committer-date alone
is unsafe for work branches: a work branch may be seeded from a
months-old commit on the user branch and never receive a synthesis
or session commit before the run dies — its tip date would then be
months in the past despite the run being seconds old.

Age is resolved per-run, in this priority order, and gc operates on
`{work, state}` branch *pairs* (sharing a `run_id`) rather than
individual branches:

1. **`.gaia/runs/<run_id>/meta.json`** — `finished_at` if terminal,
   `started_at` otherwise. Authoritative when present.
2. **State-branch manifest** — if the state branch exists on origin,
   `manifest.issued_at` (§7.4). Authoritative for runs fired from
   another machine.
3. **ULID timestamp** — extracted from the branch name's run id. ULIDs
   embed millisecond precision; this is reliable as long as the
   branch name has the canonical `claude/gaia/<ulid>/{work,state}`
   shape.
4. **Tip committer-date** — last resort. Only safe for state
   branches (whose tip is always a fresh manifest or sentinel
   commit); never used for work-branch decisions.

A pair is deleted only when *both* branches' resolved ages exceed
`--max-age`. If only one side is older (e.g., the state branch was
already gc'd manually), gc deletes the remaining branch only after
warning. `--dry-run` prints the resolution path used per branch
("via meta.json finished_at" / "via manifest issued_at" / "via ULID
timestamp" / "via committer-date") so the user can audit edge cases.

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
- The `Stop`, `StopFailure`, and `PreToolUse` hooks correlate by
  `session_id` via `<runtime-dir>/<session_id>.json` (§8.1 step 4),
  where `<runtime-dir>` is `$XDG_RUNTIME_DIR/gaia/` or
  `/tmp/gaia-$uid/`. The `UserPromptSubmit` hook writes the file
  per-session with `0600` permissions. Different sessions on the
  same VM (unusual but possible) key independently and do not
  collide.

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

`-c` scans `history.jsonl` for the most recent terminal
`status: "success"` event on the current branch. Per §10.3,
history is written only on terminal transitions — in-flight /
pending runs have a `meta.json` but no history entry, so `-c`
simply cannot see them and there is no ambiguity. Failed,
timed-out, aborted, or no-merge runs are visible via
`gaia history` and recoverable via `gaia recover`, but are not
continued by `-c` (which is only for replaying successful
conversations). For determinism across ambiguous cases, use
`--resume <session_id>`.

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
- **`--minimal-artifacts` mode.** For sensitive repos, a flag that
  pushes only the final assistant message to the state branch and
  suppresses the readable-transcript artifact. Reduces what lands
  on origin at the cost of breaking transcript-replay continuation
  for that run (§16.2). Not in v1.
- **Authenticated sentinels.** Either a server-side session-status
  API from Anthropic or hook credentials isolated from the agent's
  Bash tool would close the trust gap described in §16.1. Neither
  is exposed today.
- **Graduating `--experimental-mode` to stable** (§15). Once VM
  idle-lifetime is characterized empirically and the `Stop`-hook
  decision semantics are confirmed stable across Claude Code versions,
  the flag drops the lab-quality warning and/or the mode becomes the
  default transport for `-c` / `--resume`.
- **Composing with pre-existing hooks.** v1 refuses to install if the
  repo already defines `UserPromptSubmit`, `Stop`, or `StopFailure`
  handlers (§8.7). v1.x can lift this by making the gaia shim a
  dispatcher: for non-gaia sessions, exec the pre-existing handler;
  for gaia sessions, run gaia logic. The installed settings layout
  does not change, so the transition is non-breaking. Reserved flag:
  `gaia init --force-merge-hooks`.
- **Shell-only shim install.** Current shims are Node (§8.0) for
  cross-platform teammate support. Repos that want to minimize
  dependencies on the cloud VM or on teammates' machines could opt
  into POSIX `.sh` shims with equivalent behavior. Tracked as a
  v1.x opt-in; not the default.

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

1. Driver fires the routine with a `GAIA_EXPERIMENTAL_MODE: 1` line
   inside the `--- BEGIN GAIA CONTROL ---` block (§7.2). Drops the
   state-branch poll interval to 2s for this run. The
   `UserPromptSubmit` hook records `experimental_mode: "1"` in
   `<runtime-dir>/<session_id>.json` (§8.1 step 4) — that tmp file
   is where the Stop hook reads it. The experimental-mode flag is
   deliberately **not** written to git config (unlike the
   work-branch value in §8.1 step 5) — keeping it out of Claude's
   model-readable surface is consistent with §16.1.
2. VM: Claude processes the initial task, produces a response
   (possibly ending in a question). `Stop` hook fires.
3. Hook reads `experimental_mode` from the session's
   `<runtime-dir>/<session_id>.json` (loaded in §8.2 step 1). If it
   is not `"1"`, fall through to the standard `Stop`-hook path
   (§8.2).
   Otherwise, compute the current turn state by reading the state
   branch: let `P` be the highest turn number across `GAIA_PENDING:`
   commits (0 if none) and `I` the highest across `GAIA_INPUT:`
   commits (0 if none).
   - If `P == I`, this is a fresh turn. The new turn number is
     `T = P + 1`. Proceed to step 4.
   - If `P > I`, a prior pending turn was never answered — the
     previous hook invocation must have been killed or crashed before
     finalizing. Run the fallback-finalize path (§15.3 row 1: write
     the standard `GAIA_DONE:` artifacts with whatever transcript
     exists) and exit 0 — do not push another `GAIA_PENDING`.

   The `stop_hook_active` field on stdin is an informational hint that
   this is not the first `Stop` of the session; it is **not** used as
   the guard. Earlier drafts short-circuited on `stop_hook_active ===
   true`, which ended every multi-turn session after the first block
   (every `Stop` after the first has the field `true` by design). The
   `P` vs `I` comparison sustains arbitrarily many turns while still
   catching genuine re-entry bugs.
4. Instead of the standard `GAIA_DONE:` sentinel, push:
   ```
   GAIA_PENDING: <session_id> #<T>

   A  .gaia-state/<session_id>/pending_<T>.md    (Claude's latest response)
   ```
   The push uses the same **name-matched local-branch
   temporary-worktree** flow as §8.2 step 7
   (`git worktree add -B "$GAIA_STATE_BRANCH" "$tmp"
   "refs/remotes/origin/$GAIA_STATE_BRANCH"` plus an explicit
   `$GAIA_STATE_BRANCH:$GAIA_STATE_BRANCH` refspec on push). Every
   state-branch write in experimental mode — pending, done, failed,
   aborted — uses the identical pattern: fetch → reset local branch
   to latest origin tip → write files → commit → push → retry on
   non-fast-forward (bounded at 3 attempts). Single writer model
   per turn, but the retry loop handles races with any cross-turn
   recovery operation that might touch the same branch.
5. Hook starts polling (2s) `origin/<state_branch>` for a matching
   `GAIA_INPUT: <session_id> #<T>` commit. Blocking-wait up to
   9 minutes (safely under the configured 10-min hook timeout in §8).
6. Driver sees `GAIA_PENDING:`. Reads `pending_<T>.md`. Prints
   Claude's response to stdout. Prompts the user:
   ```
   reply (ctrl-D to finish, :continue to hand back, :abort to cancel without merging):
   ```
7. User provides input on stdin. The vocabulary is **explicit** —
   no more "bare EOF means either finish or continue depending on
   the user's intent," which is impossible to disambiguate at the
   driver. Rules, in order:
   - **Bare `ctrl-D` on empty input** → finish the session
     (`__GAIA_END__`). This is the natural "I'm done" gesture.
   - **Single line `:continue` then EOF** → hand back to Claude
     without a reply (`GAIA_CONTINUE`).
   - **Single line `:abort` then EOF** → abort without merging
     (`__GAIA_ABORT__`).
   - **Any other text (single- or multi-line) then EOF** → send the
     text verbatim as the user's reply.
   Multi-line replies are supported — only the *exact* single-line
   strings `:continue` and `:abort` are interpreted as commands;
   any leading whitespace, trailing text, or different
   capitalization falls through to the reply path. `:abort` is
   printed prominently in the prompt so users who want to exit
   without committing changes have an unambiguous path.
8. Driver writes `.gaia-state/<session_id>/input_<T>.md` with
   the reply or control token and pushes `GAIA_INPUT: <session_id>
   #<T>` to the state branch. The input-file body is one of:
   - `GAIA_CONTINUE` — user typed `:continue`, or driver
     auto-continued (§15.5 heuristic).
   - `__GAIA_END__` — user ended the session with bare ctrl-D.
   - `__GAIA_ABORT__` — user typed `:abort` at the reply prompt.
   - Otherwise: the user's literal reply text.
9. Hook sees the input commit and branches on the body. In every
   case, the hook returns its response via **exit-0 JSON** (the
   documented decision channel). The hook never exits 2 for
   continuation; the `{"decision": "block"}` + exit 0 path is the
   supported control, and reserving exit 2 for unexpected failures
   keeps the semantics parallel to the stable Stop hook (§8.2).
   - **Regular reply:** emit
     ```json
     {
       "decision": "block",
       "reason": "<user's reply, verbatim>"
     }
     ```
     and exit 0. Claude resumes with the reply as a system-reminder
     user turn. Loop to step 2 for turn `T+1`.
   - **`GAIA_CONTINUE`:** emit
     ```json
     {
       "decision": "block",
       "reason": "No user reply was provided. Continue the task autonomously."
     }
     ```
     and exit 0. Loop to step 2 for turn `T+1`.
   - **`__GAIA_END__`:** **the same hook invocation** performs the
     final sweep, writes the standard success artifacts, and pushes
     `GAIA_DONE: <session_id>` — *then* exits 0 without a decision
     object. Driver reads the sentinel, exits 0, and performs the
     normal merge-back.
   - **`__GAIA_ABORT__`:** **the same hook invocation** performs the
     final sweep (so no work is silently lost from the VM), writes
     `.gaia-state/<session_id>.md` containing the last assistant
     message and a line noting the user aborted, writes the
     transcript, and pushes `GAIA_ABORTED: <session_id>` — a
     distinct sentinel (§15.6, §7.3). Driver reads the sentinel,
     sets `meta.status = "aborted"`, appends the terminal event to
     history, preserves the work branch on origin, and exits 0
     **without merging**. `gaia recover <id> --merge` opts into a
     later merge-back if the user changes their mind.

   The `__GAIA_END__`/`__GAIA_ABORT__` paths must finalize *inside*
   this invocation because `decision: "block"` is the only thing
   that keeps the session alive; exiting 0 without it finalizes the
   session immediately, so no separate Stop hook will run later.
   All finalize work must happen here.

Notes on hook-decision semantics (current Claude Code docs):

- `Stop` hook decision-control fields are `decision` and `reason`,
  processed when the hook exits 0. Exit 2 would *prevent* Claude
  from stopping too, with stderr surfaced as feedback — but gaia
  never relies on exit 2 for continuation because the exit-0 JSON
  path is the documented control and mixing the two produces
  inconsistent surfaces.
- `additionalContext` is documented for `UserPromptSubmit`, not
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
| Driver killed (user ctrl-C or SIGTERM) | Driver leaves `meta.json.status = "pending"` (no terminal event written to `history.jsonl`). Hook eventually times out and runs the fallback finalize, writing `GAIA_DONE:` to the state branch. | User runs `gaia recover <run_id>` when ready. Recover reads the sentinel and appends the terminal event to history at promotion time. The final sweep in the hook preserved the work-branch commits; the pending response is visible in the eventual sentinel. |
| Explicit user abort (`:abort` at the reply prompt) | Driver pushes `__GAIA_ABORT__`. Hook finalizes with `GAIA_ABORTED:` in the same invocation — a distinct sentinel from `GAIA_DONE:` (§15.6). | Driver exits 0 with `status: "aborted"`, does **not** merge. Work branch preserved on origin. `gaia recover <id> --merge` opts in to a merge-back later. |

In every fallback case:

- **No work is lost for clean exits.** The Stop hook's unconditional
  push (§8.2 step 3) syncs work-branch state to origin in every clean-
  exit path. Unclean exits preserve the work branch on origin for
  `gaia recover`.
- **Failed runs are first-class in recovery.** Each run writes a
  `status: "pending"` `meta.json` before firing (§6.1 step 3). A
  crashed or timed-out lab-quality run is therefore discoverable
  via `.gaia/runs/<run_id>/meta.json` and actionable by
  `gaia recover`, which promotes the run to a terminal state and
  appends the corresponding event to `history.jsonl` at that point.
  `gaia -c` scans history only, so it can never silently pick up a
  pending or broken run and continue the one before it — it only
  sees terminal `"success"` events.
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
- **Re-entry guard.** §15.2 step 3 compares the highest `GAIA_PENDING`
  and `GAIA_INPUT` turn numbers on the state branch (`P` vs `I`) to
  decide whether the current `Stop` is a fresh turn or a re-entry. A
  fresh turn has `P == I`; re-entry without a matching input
  (`P > I`) triggers fallback-finalize instead of pushing another
  `GAIA_PENDING`. `stop_hook_active` is **not** the guard: every
  `Stop` after the first has it `true` by design, so using it as the
  short-circuit would terminate every multi-turn session after
  turn 1. The driver dying between pending turns is covered by the
  9-minute hook blocking-wait falling back to `GAIA_DONE:`.
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

Default behavior prompts the user on every turn. Auto-continue is
opt-in via `--auto-continue` because a wrong auto-continue can turn
a clarification into unsupervised work — an unacceptable failure
mode for a lab-quality surface.

```
$ gaia --experimental-mode -p "refactor the auth module to use JWT"
[gaia] --experimental-mode is lab-quality. Falls back to -c on timeout or failure.
[gaia] run 01HXX0AB..., session session_01HJK..., polling at 2s

────────────────────────────────────────────────────────────────────
AGENT: I can start, but the current config doesn't specify signing
       method. HS256 (symmetric) or RS256 (asymmetric)?
────────────────────────────────────────────────────────────────────
reply (ctrl-D to finish, :continue to hand back, :abort to cancel without merging):
HS256 with a 512-bit secret, rotated monthly
^D
[gaia] sending input (turn 1)...

────────────────────────────────────────────────────────────────────
AGENT: Got it. Implementing now — committing as I go.
────────────────────────────────────────────────────────────────────
reply (ctrl-D to finish, :continue to hand back, :abort to cancel without merging):
:continue
^D
[gaia] sending input (turn 2, GAIA_CONTINUE)...

────────────────────────────────────────────────────────────────────
AGENT: Done. Refactored middleware.ts, added HS256 signing, wired up
       rotate_secret.ts. 47 tests pass. Anything else?
────────────────────────────────────────────────────────────────────
reply (ctrl-D to finish, :continue to hand back, :abort to cancel without merging):
^D
[gaia] finalizing...
[gaia] merged 4 commits into feature/auth
Refactored the auth middleware to use JWT (HS256, 512-bit secret,
monthly rotation). 47 tests pass.
$
```

Every turn presents the reply prompt with the same three controls
visible: bare ctrl-D ends the run, `:continue` hands back to Claude,
`:abort` cancels without merging. Bare ctrl-D unambiguously means
"finish" because the driver cannot tell whether the user *intends*
to continue or to end on a per-turn basis — the prior draft's "EOF
means continue on intermediate turns and end on final turns" was
not implementable. With this vocabulary, the user always types
exactly the action they want.

**`--auto-continue`** enables a heuristic: assistant responses that
do not end with `?` and do not contain typical question phrasings
trigger an automatic `GAIA_CONTINUE` push after a 2 s idle pass on
stdin (no input received). The user can still interrupt any
auto-continuing turn with ctrl-C, then type `:abort`, ctrl-D, or a
real reply. The heuristic is simple and known to misfire — it exists
for users who have read §15 and accepted the trade-off explicitly,
not as a default. With `--auto-continue` off (the default), the
driver always waits for explicit user input before sending a turn.

**`:abort`** at the reply prompt pushes `__GAIA_ABORT__`. The hook
finalizes the session with `GAIA_ABORTED:` (§15.6), which is
distinct from `GAIA_DONE:`: the driver exits 0 with
`status: "aborted"` and **does not merge**. The work branch is
preserved on origin. If the user later decides they want to keep
the changes, `gaia recover <run_id> --merge` performs the
merge-back and promotes the status to `"success"`.

### 15.6 Protocol additions

Lab-quality mode extends the §7 protocol with three new commit-message
shapes on the state branch. None appear in a non-experimental run.

| Message | Producer | Contents |
|---|---|---|
| `GAIA_PENDING: <session_id> #<n>` | VM `Stop` hook | `.gaia-state/<session_id>/pending_<n>.md` — Claude's response for turn `n`. |
| `GAIA_INPUT: <session_id> #<n>` | Driver | `.gaia-state/<session_id>/input_<n>.md` — user's reply or control sentinel (`GAIA_CONTINUE`, `__GAIA_END__`, `__GAIA_ABORT__`) for turn `n`. |
| `GAIA_ABORTED: <session_id>` | VM `Stop` hook | `.gaia-state/<session_id>.md` (last assistant message + aborted marker) and `.gaia-state/<session_id>.transcript.md` (full readable transcript). Distinct from `GAIA_DONE:`; tells the driver not to merge. |

The standard `GAIA_DONE: <session_id>` sentinel is emitted on a
clean-finalize via EOF / `__GAIA_END__`. A user-initiated
`__GAIA_ABORT__` produces `GAIA_ABORTED:` instead. Exactly one of
the three terminal sentinels (`GAIA_DONE:`, `GAIA_FAILED:`,
`GAIA_ABORTED:`) lands on the state branch per run — the driver's
poll loop (§6.1 step 6) treats any of them as terminal. The final
`transcript.md` artifact is assembled from the full JSONL
transcript — including the system-reminder-injected turns — by the
same reverse-scan + forward-pass logic in §8.2, with one small
extension: system reminders matching gaia's injection pattern are
rendered back as `USER (via --experimental-mode):` in the Markdown
transcript so the continuation path (§6.2) sees them as real user
turns, not as reminders.

### 15.7 Doctor coverage

`gaia doctor` gains four lab-quality-mode checks (only run when
`--experimental-mode` is configured):

1. Fire a session with `GAIA_EXPERIMENTAL_MODE=1` and a known-trivial
   prompt. Confirm a `GAIA_PENDING:` commit appears within 30 s and
   that it was pushed via the local-branch temporary-worktree path
   (verified by the ref trace on the VM — the write path in §15.2
   step 4 is a release gate).
2. Push a `GAIA_INPUT:` reply. Confirm the hook's returned
   `{decision: "block", reason: <reply>}` takes effect and Claude
   produces a follow-up `GAIA_PENDING:` commit within 60 s.
3. End the session with `__GAIA_END__`. Confirm the same hook
   invocation pushes `GAIA_DONE:` within 30 s and the final transcript
   includes the injected turn rendered as `USER (via --experimental-mode):`.
4. In a separate synthetic run, push a `GAIA_INPUT:` with body
   `__GAIA_ABORT__`. Confirm the hook pushes `GAIA_ABORTED:` within
   30 s, that the driver surfaces `status: "aborted"`, that no
   merge commit lands on the user branch, and that the work branch
   remains present on origin so `gaia recover --merge` can follow up.

Failure on any of the four → `--experimental-mode` is not usable on
this Claude Code version; `gaia doctor` reports which step failed and
whether it is likely a hook-timeout, block-decision-semantics, or
VM-lifetime issue.

## 16. Security and data handling

### 16.1 Trust model

gaia is designed for **trusted developer prompts**. The developer
running `gaia -p "..."` is assumed to be acting in good faith on a
repo they have push access to. gaia does not attempt to
cryptographically authenticate the completion sentinel against a
hostile agent, and cannot: any secret available to the hook inside
the cloud VM is also reachable by Claude's own Bash tool, which
runs with the same git credentials and on the same filesystem.

**Not protected against:**

- **Prompt injection from repo content.** A file the session reads
  during the run may contain text that instructs Claude to push a
  forged `GAIA_DONE: <session_id>` to the state branch. If it
  does, the driver will read that commit. The artifact validation
  in §6.1 step 6 catches trivially-empty forgeries but a
  sophisticated injection can include plausible artifact files.
- **Malicious prompts.** A developer who writes a `gaia -p "..."`
  with instructions telling Claude to do arbitrary things on the
  repo — including pushing to non-gaia branches or emitting fake
  sentinels — will get exactly that. The prompt boundary is the
  trust boundary.
- **Compromised toolchain.** gaia trusts the `@your-scope/gaia`
  npm package, Anthropic's `/fire` endpoint, the cloud VM
  environment, and the Claude Code hook surface. A compromise of
  any of these breaks the model.

**Defense-in-depth measures present in v1:**

- The saved routine prompt (§9) explicitly instructs Claude not to
  inspect, checkout, commit to, or push any `claude/gaia/*/state`
  branch, and not to emit commits whose messages begin with any
  `GAIA_*:` sentinel. A well-behaved session never pushes protocol
  commits itself.
- `GAIA_STATE_BRANCH` is not in the model-visible payload (§7.2)
  and not in the `gaia.*` git config surface (§8.1 step 5). The
  hook derives the state-branch name internally from the
  `GAIA_RUN_ID` the model does see. This does not prevent a
  determined reconstruction — the naming convention is knowable —
  but it keeps the name out of the incidental surface.
- The `PreToolUse` hook (§8.5) blocks `Edit` / `Write` /
  `MultiEdit` / `NotebookEdit` / `Bash`-write operations on
  gaia-managed files (`.claude/hooks/gaia-*.js`,
  `.claude/settings.json`, the `.gaia/` ignore rule, `.git/**`)
  and blocks `Bash` invocations of branch-switch / hard-reset / state-branch-push
  commands. This is structural enforcement of the prompt
  instructions above — a session that ignores the prompt or is
  prompt-injected from repo content still cannot mutate the
  protocol shim or push a forged sentinel via `git push`. It does
  not stop file-system writes from outside Claude's tool surface
  (e.g., a process the agent forks into the background that
  bypasses `PreToolUse`); a determined hostile agent can still
  evade this.
- The session's `<runtime-dir>/<session_id>.json` (§8.1 step 4) is
  the **operational** source of truth for the Stop / StopFailure /
  PreToolUse hooks. It is not secret and not tamper-proof against
  a hostile in-VM actor — Claude's Bash tool runs as the same uid
  with the same filesystem access, so the agent can read or
  overwrite the file if it chooses. Directory mode `0700`, file
  mode `0600`, and `O_NOFOLLOW` protect against accidental
  cross-user reads and symlink attacks, not against the agent
  itself. The trust model treats the prompt boundary as the trust
  boundary; the runtime file's role is to keep the protocol
  coherent in the face of routine repo-state mutation, not to
  authenticate the agent.
- Every `GAIA_DONE:` commit is validated by the driver (§6.1 step
  6) to contain both `.gaia-state/<session_id>.md` and
  `.gaia-state/<session_id>.transcript.md`. Missing artifacts are
  treated as suspicious; the driver logs a warning and keeps
  polling through the remaining timeout. This catches trivial
  forgery attempts but is not authentication.
- The `UserPromptSubmit` hook's nonce / manifest check (§7.2, §7.4)
  prevents *external* prompt-spoofing (a prompt from one run
  against another run's branches), but does not help against the
  model pushing to its own run's state branch — which is what
  `PreToolUse` covers structurally.

Stronger guarantees would need server-side help from Anthropic:
either a session-status API that authenticates completion without
the state-branch hop, or hook credentials isolated from the
model's Bash tool. Both are tracked as future work (§14) and would
let v1.x tighten the trust model without changing the caller API.

### 16.2 Data retention

Every gaia run pushes content to `origin`:

- The **work branch** contains committed code changes (same as
  any PR workflow).
- The **state branch** contains the run manifest, the overflow
  context file (if used), the final assistant message, the readable
  conversation transcript, and — on `GAIA_FAILED:` — the error
  body including any `error_details` and `last_assistant_message`
  Claude Code surfaces.

State-branch commits, like all git pushes, enter the remote's
underlying object store. **Successful cleanup deletes refs, not
necessarily the underlying objects.** Depending on the host's GC
policy, forks, audit logs, and mirrors, the objects may persist
for some time after a branch is deleted. Branches are **preserved**
(not deleted) in every recoverable-failure path: `--keep-branches`,
`--no-merge`, `timeout`, `conflict`, `push-rejected`, `aborted`,
and `GAIA_FAILED:`. `gaia gc` sweeps stale branches but also only
deletes refs.

**Implications for callers:**

- **Do not put secrets in prompts.** The prompt is echoed into the
  readable transcript and (on overflow) into
  `.gaia-state/context.md` — both committed to the state branch.
  Any failure path preserves those commits on origin until a
  subsequent `gaia gc` removes the ref.
- **Assume transcripts are remote.** Anything the agent sees or
  says during the run lands on the state branch. Treat the state
  branch as centralized logging, with all the retention
  consequences that implies.
- **Sensitive repos.** For repos where transcript content is
  especially sensitive, §14 tracks a future `--minimal-artifacts`
  mode that would push only the final assistant message to the
  state branch and drop the readable-transcript artifact (at the
  cost of breaking transcript-replay continuation for that run).
  v1 always pushes both artifacts on `GAIA_DONE:` and the error
  body + transcript on `GAIA_FAILED:`.
- **Driver-local data.** `.gaia/` is machine-local and
  `.gitignore`d. Readable transcripts copied to
  `.gaia/transcripts/<session_id>.md` stay on the driver machine;
  they are not automatically pushed. Sharing transcripts across
  machines is a future feature (§14 — cross-machine history
  sync).
