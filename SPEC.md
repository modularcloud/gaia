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

## 1.1 Status

**v1 is preview.** The "stable" language used throughout this spec
(e.g., "stable surface" in §2, "v1 stable contract" in §15) refers
to the **CLI shape** — `-p` / `-c` / `--resume` / `--no-merge` /
`--json` — which gaia commits to not break within the v1 series.
The transport underneath is not stable: `/fire`, the `anthropic-
beta` header, the routines model, and the Claude Code hook
surface are all research-preview today, and Anthropic may change
response schemas, header values, or the endpoint itself on short
notice. gaia inherits that preview status:
- Patch releases of gaia will track spawner-side changes (header
  rotations, response-field renames, minor hook-schema shifts)
  without breaking the CLI contract and without bumping the
  manifest's `protocol_version` (§7.4). Because the VM setup
  script uses an **exact pinned version** of `@your-scope/gaia`
  (§5) and Claude Code caches the setup-script output for ~7
  days, patch releases do **not** automatically propagate to
  cloud VMs. After `npm i -g @your-scope/gaia@<new>` on the
  driver, `gaia cloud-upgrade` (§6.4) edits the routine's
  environment setup script to the new pinned version, which
  invalidates the cache and pulls the new binary on the next
  fire. A teammate VM that is still on the older pin
  interoperates for the same `protocol_version` — the
  interoperability guarantee is protocol-version agreement, not
  byte-identical binaries.
- A breaking transport change that cannot be absorbed in a patch
  release will ship as v2.0 with a migration guide, not as a
  silent v1.x bump. v2.0 bumps `protocol_version` and requires
  re-running `gaia init` plus `gaia cloud-upgrade` in every
  environment attached to a gaia routine.
- `gaia doctor` (§8.4) is the canonical check that the current
  transport still works for this repo. Running it before and
  after any gaia / Claude Code upgrade is the supported failure-
  mode discovery path. `gaia doctor --cloud`'s protocol-version
  invariant check (§8.4) catches the "driver upgraded locally,
  VM pin is stale" case and points at `gaia cloud-upgrade`.

The experimental surface — `--experimental-mode` (§15) — is
further-preview: "lab-quality," opt-in, and may be removed or
redesigned without a major-version bump. It is not part of the
CLI-shape stability promise.

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
  per-tool matchers; gaia uses `"*"` (every tool, classified
  inside the shim; §8.5) so every tool call is visible to gaia's
  protections, and foreign `PreToolUse` handlers add to the
  deny surface rather than contending with it (the
  most-restrictive rule means any handler's denial wins).
- **No local Claude invocation.** v1 always goes through a cloud routine.
- **No MCP / connector tools in gaia routines.** Claude Code
  surfaces MCP server tools as regular tools in tool events
  (named `mcp__server__tool`), and a GitHub / filesystem / shell
  MCP could mutate the repo or protocol state outside gaia's
  protected-path set. gaia v1 does not model the write surface
  of arbitrary MCP servers, so the routine attached to a gaia
  run must have no connectors and no write-capable MCP tools
  registered. `gaia init` surfaces this as a prerequisite (§5)
  and `gaia doctor` probes for it end-to-end (§8.4). The
  `PreToolUse` shim denies every `mcp__.*` call in a gaia
  session as a runtime safety net (§8.5). Modeling MCP
  connectors safely is tracked in §14; a future release may
  introduce an allow-list once the common connectors' write
  surfaces are characterized.
- **No project-scoped MCP or plugin configuration in the repo.**
  Cloud sessions clone the repo and load project-level
  `.mcp.json` MCP servers and project-declared plugins (hooks,
  permissions, MCP bindings) from `.claude/settings.json`
  alongside gaia's hooks. Because these can attach additional
  tools, additional hooks on the four gaia events, or new
  write surfaces that gaia's `PreToolUse` shim has not modeled,
  gaia v1 refuses to operate on a repo that contains either:
  - A committed `.mcp.json` at the repo root (or anywhere the
    cloud session will load it).
  - Any entry under `plugins` / `extraKnownMarketplaces` /
    similar plugin-loading keys in the `.claude/settings.json`
    gaia is about to commit.
  `gaia init` refuses to complete setup if either is present
  and points at the remediation (remove, or wait for the v1.x
  allow-list in §14). `gaia doctor --cloud` re-runs both scans
  on the work-branch tree it sees from inside the VM so drift
  after install is detected. A committed `.mcp.json` that only
  declares read-only servers is still refused in v1 — gaia has
  not yet modeled even read-only connectors' interactions with
  the state-branch push surface, and a permissive v1 default
  would foreclose adding stricter policy later without breaking
  repos.

## 3. Architecture

Everything ships as one npm package. The unscoped name `gaia` appears
to be taken on the public npm registry; v1 publishes under a scoped
name that still installs a `gaia` binary:

> **Release blocker — package name unfinalized.** The string
> `@your-scope/gaia` that appears throughout this spec is a
> **placeholder**, not a scope gaia controls. It MUST be replaced
> with the finalized scoped name before any real v1 publish or
> install documentation. Every `@your-scope/gaia` in `gaia init`
> output, `gaia doctor` output, the VM setup script, and the
> error messages cited in §8.0 / §12 has to switch in the same
> release. Until this is resolved, the spec is design-complete
> but not implementable end-to-end; a prototype/alpha can use
> any scope the author controls as long as the same scope is
> used in every place the placeholder appears.
>
> **CI gate.** The release pipeline includes a check that fails
> if the literal string `@your-scope/gaia` appears in any
> emitted file (CLI output snapshots, `gaia init` / `gaia
> doctor` golden files, VM setup-script template, fixtures,
> generated docs, error-message snapshots in tests). This makes
> a forgotten placeholder a hard CI failure rather than
> something that ships and breaks the install line.

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
global binary only when it is. When Node is on `PATH`, a teammate
without the global gaia binary can open Claude Code in the repo
and their non-gaia prompts run unaffected — the shim classifies
the session, exits 0 silently, and Claude Code proceeds as normal.
The *global binary* is only required for *gaia sessions*.

**Node is still a prerequisite for any Claude Code session that
opens this repo.** The shim command string in
`.claude/settings.json` starts with `node` (§8), so if Node is
not on `PATH`, the hook fails to start regardless of whether the
session is gaia — Claude Code reports a hook-start error and the
session is disrupted. The "non-gaia sessions run unaffected"
property above assumes Node is available; it is not an
unconditional promise. §5 documents Node 20+ as a hard
prerequisite for any environment opening Claude Code in a
gaia-initialized repo, and `gaia init` prints a banner so the
repo owner can communicate this to teammates. Future work (§14)
tracks a POSIX `.sh` opt-in for repos that want to drop the
Node prerequisite. See §8 for the full shim / hook layout.

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

**Platform support.** v1 certifies Linux and macOS for both the
driver and teammate environments. Windows is **not certified in
v1** — the shim `command` entries use POSIX shell expansion
(`$CLAUDE_PROJECT_DIR`) that native PowerShell does not handle
without a `"shell": "powershell"` hint, and `gaia doctor` does
not include Windows coverage yet. Windows support and a POSIX
`.sh` opt-in are tracked in §14. A teammate on Windows opening
Claude Code in a gaia-initialized repo is likely to see the hook
command fail to start; repo owners should communicate v1's
platform scope.

**Driver machine.** Linux or macOS, Node.js 20+, git, push access
to the origin repo.

**Teammate machines opening Claude Code locally in a gaia-initialized
repo.** Linux or macOS, Node.js 20+. This is a new, documented
prerequisite for gaia-initialized repos. It is *not* a Claude Code
requirement — Claude Code itself can be installed via native
installer (Homebrew, official native binary) and run with no Node
on `PATH`. The prerequisite comes from gaia: the hook shims
committed to `.claude/hooks/` are Node scripts, and the `command`
entries in `.claude/settings.json` start with `node`. If Node is
not on `PATH` when Claude Code fires the `UserPromptSubmit` /
`PreToolUse` / `Stop` / `StopFailure` hook on a teammate's
machine, the hook command fails to start — disrupting their local
Claude Code session even though they are not running gaia. `gaia
init` prints a banner flagging this expectation and the v1
platform scope so the repo owner can communicate both before the
hook files land on the default branch. See §8.0 for the shim
layout and §14 for tracking Windows support and a future POSIX
`.sh` opt-in.

**Anthropic account.** Pro / Max / Team / Enterprise with Claude Code on
the web enabled. ZDR orgs are blocked.

**Per-project, one-time setup** (walked through by `gaia init`):

1. A Claude Code cloud environment with:
   - Network access including `registry.npmjs.org`.
   - Setup script with an **exact pinned version**:
     ```bash
     #!/bin/bash
     set -euxo pipefail
     npm i -g @your-scope/gaia@<pinned-version>
     ```
     Exact pinning is deliberate: the same driver-local gaia
     release runs in every environment attached to a routine,
     so behavior is reproducible across teammates and CI.
     This script is cached after first run (Claude Code
     environment caching, ~7-day TTL). Patch releases that
     land on the driver machine via `npm i -g` do **not**
     automatically reach the cloud VM; `gaia cloud-upgrade`
     (§6.4) bumps the pinned version in the setup script to
     invalidate the cache and pick up the new binary on the
     next fire.
   - **No MCP servers configured.** gaia v1 bans MCP / connector
     tools in routines (§2). The cloud environment attached to
     the gaia routine must not have any MCP server entries in
     its configuration. `gaia init` prints the exact place to
     check in the Claude Code environment UI; `gaia doctor` runs
     a cloud-session probe (§8.4) that fails if any `mcp__*`
     tool is visible to a gaia session.
2. A Claude Code routine attached to that environment, with the saved
   prompt from §9 and an API trigger, and **no connectors
   attached**. Routines attached to a gaia run must be
   connector-free for the same reason as the environment bullet
   above. The routine must be configured with **exactly one
   repository, matching the driver checkout's `origin` GitHub
   repo** (same `owner/name` pair). `/fire` does not accept a
   repo parameter — the routine's saved configuration picks
   which repos are cloned — so a misconfigured routine can
   `/fire` successfully while the session clones an unrelated
   repo and every gaia push / fetch targets the wrong origin.
   gaia refuses to model multi-repo routines in v1 and `gaia
   doctor --cloud` (§8.4) verifies the binding end-to-end by
   pushing a beacon branch from the driver and confirming the
   VM fetches and echoes the exact branch from the exact repo.
   The binding check fails closed if the routine clones a
   different repo, clones multiple repos, or cannot read the
   beacon.
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
   matcher set (§8.5). Project-level handlers on **other**
   events (`SessionStart`, `PostToolUse`, `SubagentStop`,
   `Notification`, `ConfigChange`, etc.) are not refused, but
   `gaia init` enumerates them and prints a
   `gaia does not control these project hooks` section so the
   repo owner knows what will run alongside gaia. `SessionStart`
   specifically gets a louder warning — it runs before
   `UserPromptSubmit` and can mutate the worktree in ways gaia's
   invariants assume do not happen, so repos that register it
   should verify via `gaia doctor --cloud` that the composition
   still produces clean sentinels. `.mcp.json` in the repo or
   any plugin entry (`plugins`, `extraKnownMarketplaces`, etc.)
   in `.claude/settings.json` is a hard refusal per §2 — even
   if the plugin does not touch the four gaia events, its own
   hooks / MCP bindings / permission entries can inject tools
   into the session that gaia has not modeled.
5. `.gitignore` entries for `.gaia/` (driver-local state) and
   `.gaia-context/` (VM-side overflow-context staging per §7.2).
   `gaia init` appends both, and warns if `.env` is present at the
   repo root but not already matched by the resulting ignore set
   — gaia does not own `.env`'s ignore status (§10.2).
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
commit on the work branch before firing (§6.1 step 4, §8.8).

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

**Driver-side push permissions for merge-back.** Routines only
need to be able to push the `claude/*` namespace from the cloud
VM (§5 step 6 covers that), but the **driver process** (running
on the caller's machine) performs the merge-back push to the
user branch's recorded upstream (§6.1.1 step 6). That push uses
the caller's own git credentials, not the routine's, and is
subject to whatever branch protection rules the upstream has —
required reviewers, required status checks, signed-commit
enforcement, "everyone must use a PR" rules. If the user branch
cannot be fast-forwarded by the caller (e.g., the branch is the
default branch and is protected to require PR reviews), the
merge-back exits 15 (push rejected) and the local user branch
keeps the merge commit; the user can then push via whatever PR
flow the upstream requires. gaia does not try to discover or
work around branch protection — it surfaces the rejection
honestly and lets the user follow the existing flow. Local-only
user branches sidestep this entirely (no upstream push happens).

**Managed policy.** Enterprise Claude Code installs can
neutralize project-level hooks two ways: `disableAllHooks: true`
in any effective settings layer disables every hook regardless
of provenance; `allowManagedHooksOnly: true` in the managed
settings layer restricts hooks to those installed at the managed
layer, blocking user, project, and non-force-enabled plugin
handlers. Either setting makes gaia's project-committed shims
inert on that machine. `gaia doctor --cloud` (§8.4) probes this
on the *cloud VM* by firing a synthetic session against the
configured repo and confirming a beacon-echo hook fires; that
path catches managed policy attached to the Claude Code
environment, which is where routines actually execute. `gaia
doctor --local` probes the *driver machine's* Claude Code
install (if one is present) by registering a scratch test
hook — this covers the case where a teammate opens the repo in
`claude` locally. If either probe fails, gaia is unusable in
that environment until the policy is relaxed. This is a hard
prerequisite, surfaced loudly rather than silently degraded. A
driver machine without `claude` installed is common (§1's
"caller needs no local Claude install" promise), and in that
case `gaia doctor --local` exits 0 with a note that the local
surface was not exercised — `gaia doctor --cloud` is the
mandatory gate.

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
   base64). Read the user branch's upstream configuration once
   and cache it for the run:
   `upstream_remote = git config branch.<user_branch>.remote`,
   `upstream_ref = git config branch.<user_branch>.merge`,
   `upstream_branch = <upstream_ref stripped of refs/heads/>`,
   `local_only_branch = (upstream_remote is empty)`. These four
   values are written into both the §7.4 manifest (before
   firing) and `meta.json` (§10.3), and every push at
   merge-back time (§6.1.1) or during `gaia recover` (§6.4)
   reads them from there rather than re-querying git config,
   so a long-running session cannot drift away from the user's
   intended upstream.
2. **Preflight upstream state.** If `local_only_branch == false`,
   run `git fetch $upstream_remote $upstream_branch` once and
   classify the relationship between local `HEAD` and
   `$upstream_remote/$upstream_branch`:
   - **Upstream ref missing** (branch deleted on remote or
     never pushed) → treat as `local_only_branch: true` for
     this run and record that on the manifest; print a note.
   - **Upstream is ancestor of local** (local has unpushed
     commits) → allowed. Print a note that the unpushed
     commits will land on the gaia work branch on origin
     when the work branch is pushed, so the session sees
     them. The merge-back's upstream push (§6.1.1 step 6)
     will also push these commits since they are already on
     the local user branch.
   - **Local is ancestor of upstream** (remote has advanced
     past local) → `git merge --ff-only` the local user
     branch to the upstream tip before seeding the work
     branch. The work branch therefore contains the upstream
     advance and the session is not operating on a stale
     base. If the fast-forward fails (shouldn't on an
     ancestor relation, but defensive), exit 2 with a
     recovery message.
   - **Divergent** (local and upstream both have unique
     commits) → refuse with exit 2 before `/fire` and a
     recovery message pointing at `git pull --rebase` or the
     user's preferred reconciliation flow. Catching this
     before `/fire` avoids burning a routine run on a
     merge-back that was guaranteed to fail at §6.1.1 step 4.
   Refusing divergent state here is specific to the fresh-run
   path; `gaia recover` tolerates late-discovered divergence
   because the session has already happened.
3. **Write a preparing run record.** Create
   `.gaia/runs/<run_id>/meta.json` with `status: "preparing"` and
   every field populated that is already known (run_id,
   user_branch, work_branch, state_branch, upstream_*, base_sha
   from the current local `HEAD`, started_at, driver_version,
   protocol_version). This record is written **before any remote
   side effect** — if the driver crashes between now and step 5,
   `gaia history` and `gaia recover` can still find the run
   locally and clean up. Acquire
   `.gaia/runs/<run_id>/.lock` as a real interprocess file lock
   (§10.3) around every subsequent rewrite of this meta.json
   through the terminal transition. The `history.jsonl` event
   log is not touched yet (§10.3 — history is written only on
   terminal states).
4. Build and push the run branches:
   - Seed a local `claude/gaia/$RUN_ID/work` from the (possibly
     fast-forwarded) user branch.
   - **Enforce the work-branch hook invariant** (§8.8): reconcile the
     gaia-managed files on the work branch against the default
     branch's copies (`.claude/hooks/gaia-user-prompt-submit.js`,
     `.claude/hooks/gaia-pre-tool-use.js`,
     `.claude/hooks/gaia-stop.js`, `.claude/settings.json` entries
     for the four gaia events plus the gaia-namespace
     `permissions.deny` rules, and the `.gitignore` `.gaia/` and
     `.gaia-context/` lines). If any differ, commit the
     reconciled tree with message `gaia: ensure hook shims on
     work branch`. If nothing differs, skip the commit. This
     guarantees the `Stop` / `StopFailure` shim the session
     checks out matches the version the driver expects,
     regardless of how old the user branch is.
   - **Verify the default-branch shim's `protocol_version`
     matches the driver's expected `protocol_version`** before
     committing. If the default branch's shim advertises a
     different `protocol_version`, abort with exit 2 and
     instruct the user to run `gaia init --verify-default-branch`
     to update the default branch before retrying. This closes
     the stale-default-branch loophole where the synthesis
     step copies an older shim version that disagrees with the
     manifest's `protocol_version`.
   - **Scan the reconciled work-branch settings for foreign
     handlers** (§8.8 step 4). If any foreign handler is
     registered on `UserPromptSubmit`, `Stop`, or `StopFailure`,
     abort with exit 2 before `/fire`. A `PreToolUse` foreign
     handler whose matcher overlaps gaia's is warned but not
     blocked.
   - Update `meta.json.status` to `"pending"` under the lock,
     then push `claude/gaia/$RUN_ID/work` and
     `claude/gaia/$RUN_ID/state` (the latter as an orphan root
     containing `.gaia-state/manifest.json` per §7.4).
5. Compose the payload (§7.2) with `"<prompt>"` as `CONTEXT`.
6. Fire the spawner. The `/fire` endpoint has no idempotency-key
   header documented, so every successful request creates a
   distinct session against whatever payload was accepted. The
   driver therefore never retries the same `GAIA_RUN_ID` across
   any failure path where a session might have been created. One
   `GAIA_RUN_ID` == one `/fire` attempt. The three outcome
   buckets:
   - **Definitely not sent** — DNS failure, TCP connect failure,
     or TLS handshake failure that reached no server socket
     before any request body bytes could have been written. The
     driver retries with the same `GAIA_RUN_ID`, same payload,
     and the same work/state branches per §11.2's network-
     failure row. After retries exhausted, transition
     `meta.status = "fire-failed"` (§10.3), append a
     `"fire-failed"` event to `history.jsonl`, delete the freshly-
     pushed work / state branches from origin (no session ever
     ran, so no recovery is possible), and exit 5.
   - **Side-effect-free HTTP error envelope received** — a
     named error response from the documented side-effect-free
     set (400/401/403/404/429). These responses semantically
     mean "request rejected before work started," so retrying
     is safe for the rows marked retryable (429 only). After
     the documented retry policy exhausts or the row is non-
     retryable, transition `meta.status = "fire-failed"`,
     append the `"fire-failed"` event, delete the work/state
     branches, and exit per the §11.2 mapping (10/11/12).
   - **Ambiguous** — either a 5xx HTTP envelope (`api_error`,
     `overloaded_error`) whose side-effect semantics Anthropic
     does not document, or a network failure after request-body
     bytes may have reached the socket. The driver cannot prove
     a session was not created: a 5xx could have come back
     after the server had already begun session creation, and a
     mid-body TLS error leaves the request in an unknown state.
     The driver does **not** retry the same `GAIA_RUN_ID` in
     this case. Instead it transitions `meta.status =
     "fire-ambiguous"` (§10.3), leaves `session_id` /
     `session_url` as `null`, **preserves** the work and state
     branches on origin (a session may be about to push to
     them), and exits 16 with a message pointing at `gaia
     recover <run_id>` (§6.4). `gaia recover` polls the state
     branch for any valid terminal sentinel — the sentinel
     commit message carries the session_id, so a late arrival
     promotes the run to `success` / `failed` / `aborted`
     without the driver ever needing a `/fire` response. If no
     sentinel arrives, recovery offers to classify the run as
     `fire-failed` (with explicit confirmation) or to retry
     with a fresh `GAIA_RUN_ID` while preserving the ambiguous
     run's branches for audit.
   On a clean 2xx, the spawner returns `claude_code_session_id`
   and `claude_code_session_url`. Update `meta.json` with both and
   proceed to step 7.
7. Poll `claude/gaia/$RUN_ID/state` for a sentinel commit matching the
   `session_id`:
   - `GAIA_DONE: <session_id>` → verify the commit contains both
     `.gaia-state/<session_id>.md` and
     `.gaia-state/<session_id>.transcript.md`. If either is missing,
     log a warning (suspected forgery or truncated hook run) and
     keep polling through the remaining timeout. If the commit is
     well-formed, proceed to step 8.
   - `GAIA_FAILED: <session_id>` → routine / API error (§7.3). Update
     `meta.json.status = "failed"`, append a terminal `"failed"`
     event to `history.jsonl`, print the error body, exit 13.
   - `GAIA_ABORTED: <session_id>` → caller aborted a lab-quality
     run (§15). Update `meta.json.status = "aborted"`, append a
     terminal `"aborted"` event, print the partial transcript hint,
     exit 0. Work branch is preserved for `gaia recover` promotion.
8. Fetch the work and state branches.
9. Copy the transcript artifacts (with YAML front matter
   preserved) to `.gaia/transcripts/<session_id>.md` locally.
   The local store keeps the header for archival; it is
   stripped only at stdout time in step 11.
10. **Merge back transactionally** (§6.1.1).
11. Print `.gaia-state/<session_id>.md`'s body (with the YAML
    front-matter header from §7.3 stripped) to stdout as the
    final assistant message. The header is machine-readable
    bookkeeping the caller should not see, whereas the body
    is what gaia presents as the session's answer. A
    `--json` invocation (§6.3) places the same stripped body
    in the `final_message` field.
12. **Finalize the run.** Update `meta.json` with the terminal status
    (`success`, `merge_sha`, `pushed_upstream`, `finished_at`), then
    append a single corresponding event to `.gaia/history.jsonl`
    (with `summary` populated from the truncated final message) under
    the lock (§10.3). History is written once per terminal transition;
    readers treat the latest history record for a run_id as
    authoritative.
13. Delete `claude/gaia/$RUN_ID/{work,state}` from origin (unless
    `--keep-branches` or any non-zero exit — see §12.1).

#### 6.1.1 Transactional merge-back

After a `GAIA_DONE:` sentinel, the driver performs the merge as a
strict sequence and never force-pushes. Each step's failure surfaces
as a distinct, actionable exit code.

**Local merge lock (v1).** Two gaia processes operating in the
same physical checkout cannot run this sequence concurrently —
they both want to mutate `HEAD` and the index. Before starting
step 1 below, the driver acquires an exclusive advisory lock on
`.gaia/merge.lock` via `flock` (POSIX) or `LockFileEx`
(Windows, when the platform ships). If the lock is held, the
driver waits up to 30 s then fails with exit 2 naming the
run_id that holds the lock. The lock is held through step 7
and released in a `finally` handler so a crashed driver
cannot strand it (POSIX advisory locks release on process
exit). `gaia recover --merge` takes the same lock — a run
recovering into a merge-back serializes against any
concurrent in-flight merge-back in the same checkout.

Concurrent runs in *different* checkouts (the common case
when teammates each have their own clone) do not share
`.gaia/merge.lock`, so they never block each other. The lock
is per-working-tree, which is the level at which `git
checkout` / `git merge` conflict.

1. Verify HEAD is still the caller's original user branch. If the user
   switched branches mid-run, `git checkout` it back and re-verify the
   working tree is clean.
2. If the working tree is dirty, abort with exit 2 — gaia will not
   overwrite manual edits. Run id and work branch are printed to
   stderr.
3. If `meta.local_only_branch == false`, run
   ```
   git fetch $UPSTREAM_REMOTE \
     "+$UPSTREAM_REF:refs/remotes/$UPSTREAM_REMOTE/$UPSTREAM_BRANCH"
   ```
   using `upstream_remote`, `upstream_branch`, and `upstream_ref`
   from `meta.json` (§10.3). This picks up any upstream advance
   regardless of remote name, branch-rename, or fork
   configuration. Local-only user branches skip this step (§5,
   `meta.local_only_branch == true`).
4. If the user branch tracks an upstream, fast-forward local to
   `$UPSTREAM_REMOTE/$UPSTREAM_BRANCH`. If fast-forward is
   impossible (the user committed locally during the run), exit 2
   with a recovery message.
5. Fetch the work branch with an explicit refspec and merge it:
   ```
   git fetch origin \
     "+refs/heads/$GAIA_WORK_BRANCH:refs/remotes/origin/$GAIA_WORK_BRANCH"
   git merge --no-ff "refs/remotes/origin/$GAIA_WORK_BRANCH"
   ```
   Conflict → `git merge --abort`, exit 4; work branch preserved on
   origin. The work branch always lives on `origin` (the Claude
   GitHub proxy pushes there); the upstream for the user branch
   can be a different remote entirely (`upstream_remote` above).
6. If `meta.local_only_branch == false`,
   `git push $UPSTREAM_REMOTE "$USER_BRANCH:$UPSTREAM_BRANCH"`.
   Never force. Rejection → exit 15 (merge local, push upstream) with
   instructions: investigate the upstream advance, then `git push`
   manually when safe **or run `gaia recover <run_id>`** (which
   retries the push without re-merging; see §6.4). The local user
   branch already has the merge commit. Local-only user branches
   skip this step entirely; the run finishes with the merge commit
   on the local branch only and `meta.json` records
   `pushed_upstream: false` for clarity.
7. On full success, delete the remote work / state branches and
   finalize history. Release the merge lock.

Explicit refspecs are used throughout because gaia branches are
slash-heavy (`claude/gaia/<ulid>/work`) and ambiguous fetch / checkout
arguments can resolve to either a tag or a remote-tracking ref
depending on local refs. The `+refs/heads/...:refs/remotes/...` form
fetches exactly the named branch into the corresponding tracking ref
and nothing else.

**Why `meta.upstream_*` and not a fresh `git config` lookup.**
Reading `branch.<name>.remote` / `branch.<name>.merge` at
merge-back time works for the common case but misses two edge
cases: (1) a long-running session where the user deliberately
retargets the branch to a different remote or renames the
upstream; (2) `gaia recover` running hours later after the
user's local git config has drifted. Pinning the upstream at
fire time and replaying it at merge-back time keeps the push
target consistent with what the user meant when they
invoked `gaia`.

#### 6.1.2 `--no-merge`

When the caller passes `--no-merge`, the driver executes §6.1
steps 1–8 (push, fire, poll, fetch, copy transcripts) and
step 10 (print the final assistant message to stdout), **skips
step 9** (merge-back, i.e., the whole of §6.1.1) and **skips
step 12** (remote-branch deletion), and performs a **modified
step 11**: finalize `meta.json` with `status: "no-merge"` (no
`merge_sha`, `pushed_upstream: false`, `finished_at` set) and
append a single terminal `"no-merge"` event to
`history.jsonl` under the lock. Concretely:

- `meta.json.status` is set to `"no-merge"` and a matching terminal
  `"no-merge"` event is appended to `history.jsonl` (§10.3). This
  is a terminal state distinct from `"success"` (merged) and from
  `"conflict"` / `"timeout"` (failures — see §12).
- `claude/gaia/$RUN_ID/{work,state}` are preserved on origin, as
  if `--keep-branches` were also set (§12.1).
- stdout carries the final assistant message, exactly as in a
  merged run.
- stderr carries one line with the work-branch name and a
  `gaia recover <run_id>` hint so the caller can complete the
  merge later.
- Exit code is `0`. `--no-merge` is an intentional caller
  request, not a failure.
- `gaia -c` scoped to the current branch **does not** pick up
  `"no-merge"` runs (it only continues `"success"`). Use
  `--resume <session_id>` to explicitly continue a no-merge
  run's transcript, or `gaia recover <run_id>` to complete the
  merge and promote the run to `"success"` first.

Exit codes:

| Code | Meaning |
|---|---|
| 0 | Session completed; stdout is the final message. |
| 2 | Precondition failed (no `-p`, dirty tree, detached HEAD, missing config, unknown `--resume` id, user branch diverged non-fast-forward at merge). |
| 3 | Session timed out. Work + state branches preserved. `gaia recover <run_id>` offered. |
| 4 | Merge conflicted. Work branch preserved; user branch untouched. Recoverable via `gaia recover`. |
| 5 | Definite-not-sent network failure after retries (DNS / TCP connect / TLS handshake; §11.2). `meta.status = "fire-failed"`. |
| 10 | Spawner returned 401/403 (entitlement / token). `meta.status = "fire-failed"`. |
| 11 | Spawner returned 429 after retries exhausted. `meta.status = "fire-failed"`. |
| 12 | Non-auth, non-rate-limit side-effect-free spawner failure — 400 invalid request, 404 routine not found (§11.2). `meta.status = "fire-failed"`. 5xx responses are ambiguous rather than fire-failed and surface as exit 16. |
| 13 | `GAIA_FAILED:` sentinel received (session aborted by routine / API error; details printed to stderr). |
| 15 | Merge succeeded locally but push to the recorded upstream branch (`meta.upstream_remote/<upstream-branch>`, often but not always `origin/<user-branch>`) was rejected. Local user branch has the merge commit; `git push` when ready, or run `gaia recover <run_id>` to retry the push without re-merging. |
| 16 | Ambiguous `/fire` failure — driver could not prove either outcome (§6.1 step 6, §11.2). `meta.status = "fire-ambiguous"`. Work/state branches preserved for `gaia recover <run_id>` to poll for a late sentinel. |

### 6.2 Continuation

```
gaia -c -p "<prompt>"                              # continue most recent successful run on current branch
gaia --resume <session_id> -p "<prompt>"           # resume a specific session (any branch)
```

**`-c` / `--continue`** resolves the **most recent run on the
current branch** by scanning `.gaia/runs/*/meta.json` (the
canonical source of truth per §10.3) — not only
`.gaia/history.jsonl` — and filtering to runs whose
`user_branch` equals the current branch. "Most recent" is
ordered by the ULID embedded in `run_id`, which is
time-ordered at millisecond precision, so parallel runs
fired on the same branch sort unambiguously.
  
- **If the newest run is non-terminal** (`status: "preparing"` or
  `"pending"`), `-c` refuses with exit 2 and prints
  `run <run_id> is still in flight — wait for it to finish, run
  "gaia recover <run_id>", or --resume <older_session_id> to skip
  this run`. Silently continuing the one before it would hide an
  in-flight run whose outcome may still land on this branch.
- **If the newest run is `fire-ambiguous`**, `-c` similarly
  refuses with exit 2 and points at `gaia recover <run_id>` (which
  polls for a late sentinel and resolves the ambiguity).
- **If the newest run is any other non-success terminal state**
  (`failed`, `timeout`, `conflict`, `push-rejected`, `no-merge`,
  `aborted`, `fire-failed`), `-c` refuses with exit 2 naming the
  offending run and pointing at `gaia recover` for the
  recoverable cases. `--resume <session_id>` is the deliberate
  path when the caller actually wants an older successful run.
- **If the newest run is `"success"`**, `-c` proceeds with its
  transcript.
- **If there is no run at all on the branch**, `-c` exits 2 with
  `no prior gaia run on <branch> — run gaia -p "..." first`.

This rule is intentionally strict: `-c` refuses to skip past a
newer non-success or pending run to continue an older successful
one, because silently doing so would hide a broken / unresolved
most-recent attempt. Scoping to the current branch avoids
surprising cross-branch continuation. An older successful run
can still be resumed deliberately with `--resume <session_id>`.

Reading from `meta.json` rather than `history.jsonl` is the
load-bearing fix for the pending-race case: history is only
written on terminal transitions (§10.3), so a run that was fired
two minutes ago and is still processing has a meta.json but no
history entry. Reading history alone would make `-c` invisibly
pick up an older successful run and start a second, racing
session while the first is still in flight. Reading meta.json
first closes that gap.

**`--resume <session_id>`** targets a specific prior session by its
Claude Code session id (the same id `--resume` takes in `claude`
itself). Works across branches; the caller's current branch becomes
the new user branch for the continuation run. History records the
session id alongside the run id so lookup is direct.

**Resuming a run whose code did not merge.** When the resolved prior
run's `meta.status` is `"no-merge"`, `"conflict"`, `"timeout"`,
`"aborted"`, `"push-rejected"`, or `"failed"`, the prior run's
code changes do not necessarily live on the caller's current
branch. A naive replay would hand Claude a transcript describing
file edits that are not actually in the new session's working
tree — a guaranteed source of "you said you changed X but X is
not here" confusion. (`"failed"` runs are included because a
`StopFailure` can fire after the agent committed work locally
but before the work-branch push completed; the work branch on
origin may have only a subset of what the transcript describes.)
For `"fire-failed"` runs there is no transcript to replay (the
session never existed) and `--resume` exits 2 unconditionally.
For `"fire-ambiguous"` runs `--resume` similarly exits 2 — the
caller should run `gaia recover <run_id>` first to resolve the
ambiguity before continuing.

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

**Cross-branch guard on `--resume-seed-from-work`.** When
`--resume-seed-from-work` is passed and the prior run's
`meta.user_branch` does **not** match the caller's current branch,
the driver refuses with exit 2 unless `--cross-branch` is also
passed. A prior run that fired on `feature/auth` and is now being
seeded into a new run on `feature/payments` rewrites repository
history in a way that is easy to request accidentally (`gaia
-c -p "..."` on the wrong branch, a continuation flag on a branch
renamed underneath) and hard to reason about after the fact.
The `--cross-branch` opt-in is deliberate: it names the
non-obvious mutation so the caller cannot request it silently.
The guard also verifies `git merge-base <prior.base_sha> HEAD`
produces a sane common ancestor (i.e., the two branches share
history); if they do not, the driver refuses unconditionally
because the resulting merge-back is almost certainly a
divergence the user did not intend. Both the prior-run's
`user_branch` and the current branch are recorded in the new
run's `meta.json` under `user_branch` (current) and a new
`seeded_from_branch` field (prior), so `gaia history` and
`gaia recover` can surface the cross-branch origin when
auditing.

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
| `--resume-seed-from-work` | off | When `--resume` / `-c` targets a prior run whose code did not merge (`no-merge`, `conflict`, `timeout`, `aborted`, `push-rejected`, `failed`), seed the new run's work branch from the prior run's work-branch tip instead of the current user branch. See §6.2. |
| `--cross-branch` | off | Opt-in acknowledgement that `--resume-seed-from-work` (or `--resume`) is carrying a prior run whose `meta.user_branch` differs from the current branch. Required when cross-branch seeding is actually happening; ignored otherwise. Prevents accidental cross-branch history rewrites. See §6.2. |
| `--spawner <name>` | from config | Override the spawner. |
| `--timeout <duration>` | `45m` | Per-session timeout. |
| `--json` | off | Print a single-line JSON result instead of bare text. Schema: `{run_id, session_id, status, final_message, merge_sha?, continued_from?, transcript_path?, error?}`. `final_message` carries the same content the bare-text mode prints to stdout. On failure (any non-zero exit), `--json` writes the JSON object to **stdout** with `final_message: null` and `error: {kind, message}`; human-readable diagnostics still go to stderr. |
| `--experimental-mode` | off | **Lab-quality** (§15). Keeps the session alive across Q/A turns via Stop-hook block-and-inject. Prints a warning on start. When possible, finalizes via the same `GAIA_DONE` / `GAIA_FAILED` / `GAIA_ABORTED` sentinels as stable runs (§15.3); on hook crashes and VM wall-clock timeouts recovery goes through `gaia recover`. Not part of the v1 stable contract. |
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
gaia doctor [--cloud|--local|--full]           # validate transport. Default --cloud fires a synthetic session and checks spawner, hooks, transcript schema, branch permissions, MCP-free routine, VM managed policy, routine/repo binding, project .mcp.json / plugin absence. --local requires a local `claude` and checks driver-machine hook honoring and the Node shim. --full runs both.
gaia cloud-upgrade                             # bump the routine's environment setup-script pin to the currently-installed @your-scope/gaia version. Prints the edited script and, when possible, edits it in place via the routines configuration API; otherwise prints the exact value to paste. Invalidates the Claude Code setup-script cache so the next fire installs the new binary.
gaia version
```

`gaia init`:
1. Prompts for routine URL + token; stores them securely.
2. Writes `gaia.config.json` to the repo root.
3. Writes `.claude/settings.json` and `.claude/hooks/*.js` to the repo
   root (§8). If `.claude/settings.json` exists **but does not already
   define project-level `UserPromptSubmit`, `Stop`, or `StopFailure`
   handlers**, merges the gaia-managed subset (§8.8) — the four hook
   entries plus the `permissions.deny` entries — non-destructively,
   preserving foreign entries on the same top-level keys. If it
   does define any of those three exclusive events, `gaia init`
   aborts with the error documented in §8.7 (hook coexistence)
   and offers migration guidance. Notes that managed-policy or
   plugin-scope handlers for the same events are outside `gaia
   init`'s visibility (§8.7) and, if present, will still run
   alongside gaia's.
   Also enumerates every project hook registered on every event
   outside gaia's four (`SessionStart`, `PostToolUse`,
   `SubagentStop`, `Notification`, `ConfigChange`, `WorktreeCreate`,
   etc.) and prints a `gaia does not control these project hooks`
   block listing each event, matcher, and command. Hooks on these
   events are **not** refused — they coexist freely with gaia —
   but the list makes their presence explicit so the operator
   knows what will run alongside gaia's protocol. `SessionStart`
   gets a louder warning because it runs before `UserPromptSubmit`
   and can mutate the worktree or env in ways gaia's invariants
   assume do not happen; the banner recommends running
   `gaia doctor --cloud` to confirm the composition still
   produces clean sentinels.
   Refuses outright if `.mcp.json` is present at any load path
   the cloud session will see, or if `.claude/settings.json`
   defines plugin entries (§2, §5). These are hard blockers,
   not warnings.
4. Appends `.gaia/` and `.gaia-context/` to `.gitignore` (each
   only if not already present). Also verifies that if `.env`
   exists at the repo root, a matching ignore rule is already in
   effect — warns and points at remediation if not (§10.4).
5. Prints the saved routine prompt (§9) for the user to paste into the
   Claude Code routines UI.
6. Prints the one-line VM setup-script snippet.
7. Prints the Node-prerequisite banner (§5) so the repo owner can
   communicate to teammates that opening Claude Code in this repo
   requires Node 20+ on `PATH`.
8. Prints the **MCP-free prerequisite banner** (§2, §5): the
   Claude Code environment attached to the gaia routine must
   not have any MCP servers configured, and the routine must
   not have connectors attached. gaia cannot check this via
   the public API, so the banner names the exact UI surface to
   inspect. `gaia doctor --cloud` verifies the prerequisite
   end-to-end (§8.4).
9. Verifies the hook files exist on the default branch. There are
   three exit paths from this step:
   - **Hook files already on the default branch.** Setup is fully
     complete. `gaia init` proceeds to step 10 and exits 0 with
     `status: complete`.
   - **Hook files missing from the default branch and the current
     branch is the default branch.** `gaia init` commits the files
     directly on the current branch (no PR needed) and exits 0
     with `status: complete` (or surfaces a push failure
     verbatim).
   - **Hook files missing from the default branch and the current
     branch is *not* the default branch.** `gaia init` cannot
     declare setup complete on its own — the hook files have to
     reach the default branch before any gaia run can use them
     (§5). Two sub-paths:
     - With `gh` available, `gaia init` offers to open a PR
       targeting the default branch. If the user accepts and the
       PR is opened, `gaia init` exits 0 with
       `status: pending-default-branch-merge` and prints the PR
       URL and the next-step command:
       `gaia init --verify-default-branch` to re-run only step 9
       once the PR has merged. **The `pending-default-branch-merge`
       status is not "success" — it is "PR opened, waiting for
       merge."** Skip step 10 (`gaia doctor` would fail without
       hooks on the default branch).
     - Without `gh`, or if the user declines the PR offer, `gaia
       init` prints the git commands to push the branch and open
       a PR manually, then exits non-zero (exit 2) with
       `status: pending-default-branch-merge`. Same re-run
       instructions.
   `gaia init --verify-default-branch` is a no-arg admin command
   that re-checks step 9, runs step 10 if step 9 passes, and
   updates the status to `complete`. Idempotent; safe to re-run.
10. Offers to run `gaia doctor --cloud` before exit (the
    required tier; §8.4). Skipped on the
    `pending-default-branch-merge` exit path. `--full` can be
    requested at the prompt to add local-tier checks when a local
    `claude` is available.

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
- **Same-clone recovery is the v1 supported path.** The recover
  command needs `meta.json` to know the recorded `user_branch`,
  `upstream_*`, and `work_branch` so it can land the change on
  the right branch via the right push target. If the caller is
  in a *different clone* (e.g., the original run fired from a CI
  job and the user is now on their laptop) and the local
  `.gaia/runs/<run_id>/meta.json` does not exist, recover exits
  2 with a message naming the missing file and pointing at the
  cross-machine recovery path: fetch the run's state branch
  (`git fetch origin "refs/heads/claude/gaia/<run_id>/state"`),
  read the manifest (`git show
  refs/remotes/origin/claude/gaia/<run_id>/state:.gaia-state/manifest.json`),
  and reconstruct the meta locally — or wait until cross-machine
  history sync ships (§14). The `claude/gaia/<run_id>/state`
  branch on origin carries enough metadata
  (`user_branch`, `upstream_*`, `work_branch`, `base_sha`) to
  bootstrap a local `meta.json` for an alternate-clone recover,
  but v1 does not implement the bootstrap automatically — the
  message just describes the manual sequence so a determined
  user can complete the merge-back from a cold checkout.
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
    true, it fetches
    `$UPSTREAM_REMOTE/$UPSTREAM_BRANCH` using the
    `upstream_remote` / `upstream_branch` recorded in
    `meta.json` (not a hardcoded `origin`) and retries
    `git push $UPSTREAM_REMOTE "$USER_BRANCH:$UPSTREAM_BRANCH"`
    without re-merging. This preserves upstream configurations
    where the user branch tracks a fork, a non-`origin` remote,
    or a renamed upstream branch. If the upstream has advanced
    such that fast-forward is no longer possible, `gaia recover`
    stops and instructs the user to rebase/merge manually,
    printing both `merge_sha` and the current upstream tip. If
    the local branch no longer contains `merge_sha` (e.g., the
    user reset locally), `gaia recover` refuses and prints the
    work-branch name so the caller can decide whether to
    re-merge from scratch or abandon. Never force-pushes.
    Acquires `.gaia/merge.lock` for the retry (§6.1.1) so a
    recover retry does not race an in-flight run in the same
    checkout. Local-only runs (`meta.local_only_branch: true`)
    have no push to retry; `gaia recover` reports success
    immediately since the merge already landed locally at run
    end.
  - **`"no-merge"`**, **`"timeout"`**, **`"conflict"`**: fetches
    the preserved `claude/gaia/$RUN_ID/{work,state}` branches. If
    a `GAIA_DONE:` sentinel exists and validates (§6.1 step 7),
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
    any session existed (§6.1 step 6). There is no `session_id`,
    no work / state branches on origin (deleted at fire-failure
    time), and no recovery action available — `gaia recover` simply
    prints the recorded error body and exits 0. Listed for
    completeness so `gaia history` entries with this status are not
    silently ignored.
  - **`"fire-ambiguous"`** (§6.1 step 6): the driver could not
    tell whether `/fire` created a session. `session_id` is
    `null`, the work and state branches are preserved on origin.
    `gaia recover` fetches the state branch with an explicit
    refspec and polls it for any `GAIA_DONE: <session_id>` /
    `GAIA_FAILED: <session_id>` / `GAIA_ABORTED: <session_id>`
    commit (bounded, default ~5 min — override with `--wait
    <duration>`). If a sentinel arrives, `meta.session_id` is
    extracted from the commit message and the run promotes to
    the corresponding terminal state (`"success"` + merge-back,
    `"failed"`, or `"aborted"`) exactly as if the sentinel had
    arrived during the original run's poll loop.

    `meta.session_url` **stays `null`** on fire-ambiguous
    promotion: the sentinel artifacts
    (`.gaia-state/<session_id>.md`,
    `.gaia-state/<session_id>.transcript.md`,
    `.gaia-state/<session_id>.error.json`) do not carry a
    `session_url`, and gaia does not synthesize URLs from the
    session_id because the endpoint's URL shape is opaque
    (§10.3). A future release could have the Stop /
    StopFailure hooks also push a
    `.gaia-state/<session_id>.meta.json` artifact carrying
    `{session_id, session_url, issued_at}` so recovery can
    set `session_url` too; that artifact does not exist in v1
    and `session_url: null` is the correct terminal state for
    a fire-ambiguous run that resolves via late sentinel. The
    URL is informational (for debugging in the Claude Code UI),
    not load-bearing on recovery.

    If no sentinel arrives within the wait window, `gaia
    recover` offers to mark the run as `"fire-failed"` and
    delete the preserved branches (explicit user confirmation,
    or `--assume-failed` for scripts); declining leaves the
    run in `"fire-ambiguous"` for a later retry. Never auto-
    promotes ambiguous runs to `"fire-failed"` without
    confirmation, since a slow session may still push a late
    sentinel. The caller can alternatively start a fresh run
    (new `GAIA_RUN_ID`) while leaving the ambiguous branches
    preserved for later audit.
  - **`"preparing"`** (driver crashed between §6.1 step 3 and
    step 4's push — the run has no work / state branches on
    origin and no session yet): `gaia recover` confirms via
    `git ls-remote origin "refs/heads/claude/gaia/$run_id/*"`
    that no gaia branches exist, transitions the status to
    `"fire-failed"` with a synthetic `error: "driver-crashed-
    before-fire"`, appends the terminal history event, and
    cleans up the local `.gaia/runs/<run_id>/` directory after
    confirmation.
  - **`"pending"`** (driver crashed after §6.1 step 4's push but
    before a terminal sentinel landed): prints the
    `session_url` (if known — `null` for the crashed-before-fire
    case above, which is caught by `"preparing"` handling),
    polls the state branch once for a late sentinel, and treats
    a subsequent `GAIA_DONE` / `GAIA_FAILED` / `GAIA_ABORTED`
    arrival like the corresponding terminal-state path above.

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
(`claude/gaia/<ulid>/state`) is *not* in the model-visible
payload. The hook derives it internally from `GAIA_RUN_ID`, which
is shown. **This is accidental-surface reduction, not access
control.** A motivated agent can still derive the state-branch
name from `GAIA_RUN_ID` plus the documented `claude/gaia/<id>/
state` convention (the convention appears throughout this spec
and the saved routine prompt explicitly mentions `claude/gaia/*/
state` so the agent knows to *avoid* it). The agent can also
freely run `git fetch origin "refs/heads/claude/gaia/<id>/state"`
or `git show refs/remotes/origin/claude/gaia/<id>/state:.gaia-state/
manifest.json` to read it — gaia does not block reads on
`claude/gaia/*/state` (only writes via §8.5's protected-commands
set). The benefit of keeping the name out of the payload is
narrow: it removes one trivial accident path where an off-task
agent that scans its own prompt might bump into the state branch
without being told it exists. The §9 prompt instructs avoidance,
the §8.5 PreToolUse set blocks writes, and the §16.1 trust model
treats the prompt boundary as the trust boundary; the read
surface is acknowledged.

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
.gaia-context/<run_id>/context.md
--- END CONTEXT FILE ---
```

The `UserPromptSubmit` hook (§8.1) detects this marker, fetches the
state branch, and copies `.gaia-state/context.md` into the **work-tree**
at `.gaia-context/<run_id>/context.md` (gitignored per §10.4, so the
Stop hook's `git add -A` will not commit it). The hook then returns
`hookSpecificOutput.additionalContext` pointing Claude at the file;
Claude Code injects that string as a system reminder the model sees
alongside the original prompt.

**Why `.gaia-context/` and not `.gaia/runs/<run_id>/`.** An earlier
draft placed the overflow file under `.gaia/runs/<run_id>/context.md`,
which collides with the `Read(/.gaia/**)` deny rule in §8: the
driver-local `.gaia/` tree holds secrets and transcripts and is
fully read-denied as defense in depth, so a file that Claude must
read cannot live under it. `.gaia-context/` is a distinct,
VM-only namespace — gitignored (§10.4), write-denied (§8's
`permissions.deny` + §8.5's protected-path set), and read-allowed.
The shim writes to it (hooks are not tool calls, so permission
rules do not apply); only tool-driven writes are blocked.

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

**Artifact header format.** Every `.gaia-state/*.md`, `.json`, and
`.error.json` artifact gaia writes (including the manifest, the
final-message file, the readable transcript, and the experimental-
mode `pending_<n>.md` / `input_<n>.md` files) **begins with a
YAML front-matter block** delimited by `---` on its own line:

```
---
created_by: gaia
schema_version: 1
artifact_type: final_message
session_id: session_01HJK...
run_id: 01HXX...
---

<body — the final assistant message, the transcript, etc.>
```

`artifact_type` is one of: `manifest`, `final_message`,
`transcript`, `error`, `overflow_context`, `pending`, `input`,
`rescue_patch`, `rescue_bundle`, `rescue_head`. Producers
always emit the front matter; readers tolerate a missing
header (the v1 release is the first one writing it) and fall
back to treating the whole file as the body.

**Driver-side stripping.** The driver strips the front matter
before:
- Printing `.gaia-state/<session_id>.md` (the final assistant
  message) to stdout — the caller sees only the body (§6.1
  step 11).
- Injecting a prior transcript into a `PREVIOUS CONVERSATION:`
  block for `-c` / `--resume` (§6.2, §7.2) — the model sees the
  Markdown body, not the gaia bookkeeping header.

The full file (front matter + body) is preserved in the local
transcript store (`.gaia/transcripts/<session_id>.md`, §10.3)
for archival and for future migrations that may want to
discriminate schema versions.

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

**Work-branch-push-rescue artifacts.** When the `Stop` (or
`StopFailure`) hook cannot push the work branch at §8.2 step 7
(authentication glitch, proxy rejection, partial network
failure), the hook still writes a sentinel — but without the
work-branch push, the session's final commits may not be on
origin. To keep the "no work silently lost" invariant honest,
the hook falls back to embedding the work-tree state in the
state-branch commit itself:

```
commit <sha>
    GAIA_DONE: <session_id>         (or GAIA_FAILED:/GAIA_ABORTED:)

A  .gaia-state/<session_id>.md                 (final assistant message)
A  .gaia-state/<session_id>.transcript.md      (readable conversation transcript)
A  .gaia-state/<session_id>.rescue.head.txt    (work-branch HEAD sha at Stop time)
A  .gaia-state/<session_id>.rescue.patch       (git diff origin/work..HEAD)
A  .gaia-state/<session_id>.rescue.log.txt     (git log --oneline origin/work..HEAD)
A  .gaia-state/<session_id>.rescue.bundle      (git bundle create - <base>..HEAD)
```

The `.patch`, `.bundle`, `.log.txt`, and `.head.txt` files carry
enough metadata to reconstruct the missing commits in a local
clone. The `.bundle` is authoritative: `git bundle verify` and
`git fetch <bundle> <ref>` restore the full commit graph
(including merges) when the original work-branch push never
landed. The `.patch` is a human-readable fallback for cases
where the bundle is unusable. The hook writes these files only
when the work-branch push fails; a successful push elides them.

The hook attempts the rescue artifacts **via the same state-
branch push** that carries the sentinel, so the failure is
surfaced atomically alongside the final message. If even the
state-branch push fails, the hook exits 1 and the driver
times out (exit 3); `gaia recover` surfaces the session URL so
the user can inspect the cloud VM in the Claude Code UI before
it recycles. Losing both pushes is rare — the two branches
share a remote (`origin`) and a proxy — but the spec
acknowledges the residual risk rather than claiming absolute
durability.

**`gaia recover` restoring from rescue artifacts.** When a
sentinel includes rescue artifacts, `gaia recover` (§6.4):
1. Verifies `git rev-parse origin/$GAIA_WORK_BRANCH` — if the
   work branch on origin already matches `rescue.head.txt`,
   the work-branch push eventually succeeded (e.g., a late
   retry from a subsequent hook invocation) and the rescue
   artifacts are redundant; recover falls through to the
   normal merge-back path.
2. Otherwise runs `git bundle verify` on the bundle; if valid,
   fetches the bundle's refs into the local repository and
   recreates `origin/$GAIA_WORK_BRANCH` locally with those
   commits, then continues merge-back as in §6.1.1.
3. If the bundle is invalid or unusable, falls back to
   applying `rescue.patch` on top of the recorded `base_sha`
   and alerts the user that the reconstruction is diff-based,
   not commit-graph-based (fewer guarantees about merge
   history).

All artifacts are plain text (Markdown, JSON, or standard git
formats). The driver reads `<session_id>.md` on success or
abort, `<session_id>.error.json` on failure, and
`<session_id>.transcript.md` for future continuation. Binary
artifacts (`<session_id>.rescue.bundle`) are stored as git blobs
and fetched through the normal git plumbing.

### 7.4 Manifest

Before firing, the driver commits `.gaia-state/manifest.json` to the
state branch:

```json
{
  "created_by": "gaia",
  "schema_version": 1,
  "run_id": "01HXX...",
  "nonce": "<base64-24>",
  "work_branch": "claude/gaia/01HXX.../work",
  "state_branch": "claude/gaia/01HXX.../state",
  "base_sha": "<sha of user branch at fire time>",
  "user_branch": "feature/auth",
  "upstream_remote": "origin",
  "upstream_branch": "feature/auth",
  "upstream_ref": "refs/heads/feature/auth",
  "local_only_branch": false,
  "driver_version": "1.0.0",
  "protocol_version": 1,
  "issued_at": "2026-04-21T10:30:00Z",
  "managed_file_hashes": {
    ".claude/hooks/gaia-user-prompt-submit.js": "<sha256>",
    ".claude/hooks/gaia-pre-tool-use.js": "<sha256>",
    ".claude/hooks/gaia-stop.js": "<sha256>",
    ".claude/settings.json": "<sha256>"
  },
  "managed_gitignore_hash": "<sha256 of normalized gaia ignore subset>"
}
```

`created_by` and `schema_version` lead every state artifact gaia
writes so future migrations can detect and translate older
artifact shapes. The concrete serialization depends on the
file's content type:
- **JSON artifacts** (`manifest.json`, `<id>.error.json`) carry
  the header as top-level JSON keys, as shown above for the
  manifest.
- **Markdown artifacts** (`<id>.md`, `<id>.transcript.md`, and
  the experimental-mode `pending_<n>.md` / `input_<n>.md`)
  carry the header as **YAML front matter** delimited by `---`
  on its own line at the start of the file, followed by a blank
  line and then the Markdown body (§7.3 specifies the full
  format and the driver's stripping behavior before the body
  is printed to stdout or fed into transcript replay).
- **Rescue artifacts** (`<id>.rescue.*` — §7.3) follow the
  convention that matches their content type: the text files
  (`.head.txt`, `.log.txt`, `.patch`) carry an empty YAML front
  matter block `---\nartifact_type: rescue_*\n...\n---\n` at
  the top; the binary `.rescue.bundle` is a raw git bundle
  with no header (recovery inspects it with `git bundle verify`).

Readers tolerate a missing header on any artifact type (the v1
release is the first one writing it) and fall back to treating
the whole file as body / value.

The `managed_file_hashes` map records the sha256 of each
gaia-managed file as it appears on the work-branch tip
*after* §8.8 synthesis (so it reflects the known-good state
the session starts from). The Stop hook's integrity spot-
check (§8.2 step 6) compares these hashes against the
worktree at finalization time and refuses to push the work
branch if any mismatch — protocol mutations cannot flow into
the user branch through the merge-back that way. Hashes are
a cheap defense against `PreToolUse` being bypassed; they
are not cryptographic authentication of the session itself
(§16.1 trust model still applies).

`.gitignore` is **not** in `managed_file_hashes` because gaia
does not own the file as a whole — Claude may legitimately add
non-gaia ignore rules during the session (e.g., a new build-
output directory). Instead, `managed_gitignore_hash` records a
sha256 of the **gaia-owned ignore subset**: the
exact lines `.gaia/` and `.gaia-context/` (one per line, in
that fixed order, no surrounding whitespace, with a trailing
`\n`). The Stop hook's integrity check verifies that those
two lines are still present in the work-branch `.gitignore`
and computes the same normalized hash; foreign ignore lines
are ignored by the comparison. This catches the case where the
session deletes or alters the gaia ignore lines while leaving
unrelated edits intact.

The four upstream fields are captured from the caller's git
configuration at fire time and reused by the merge-back
(§6.1.1) and by `gaia recover` (§6.4) so every push targets
the actual upstream the user branch tracks, not a hardcoded
`origin`. `upstream_remote` and `upstream_branch` are pulled
from `git config branch.<name>.remote` and
`git config branch.<name>.merge` (stripping the
`refs/heads/` prefix from the latter for display);
`upstream_ref` is the full refspec as git recorded it. A
user branch with no upstream (`local_only_branch: true`)
sets all three upstream fields to the empty string and
skips the push step at merge time (§5, §6.1.1). Recording
them once on the manifest prevents drift across a long-
running session where the user might change the branch's
upstream mid-run.

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

**`driver_version` vs `protocol_version`.** The manifest carries
both. `protocol_version` is the **load-bearing compatibility
field** — the `UserPromptSubmit` shim and the global
`gaia-hook-*` binary each pin a `protocol_version` they support,
and the shim refuses to proceed if `manifest.protocol_version`
does not match its pinned value (§8.0). `protocol_version` is a
small integer that increments only when the wire-level shape of
the protocol changes (manifest schema, sentinel commit format,
shim ↔ binary interface, control-block fields). `driver_version`
is the full semver of the driver that fired this run and is
**informational** — recorded for diagnostics and surfaced in
`gaia history` / error messages. Patch and minor releases of
gaia that share a `protocol_version` are interoperable at the
shim/binary boundary, so a teammate VM with an older `npm i -g`
than the local driver still works as long as the protocol
versions agree. Only a `protocol_version` bump (a major-release
event) requires re-running `gaia init` and re-deploying the
global binary in the cloud environment.

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
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "deny": [
      "Read(/.gaia/**)",
      "Write(/.gaia/**)",
      "Edit(/.gaia/**)",
      "MultiEdit(/.gaia/**)",
      "Write(/.gaia-state/**)",
      "Edit(/.gaia-state/**)",
      "MultiEdit(/.gaia-state/**)",
      "NotebookEdit(/.gaia-state/**)",
      "Write(/.gaia-context/**)",
      "Edit(/.gaia-context/**)",
      "MultiEdit(/.gaia-context/**)"
    ]
  },
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
        "matcher": "*",
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

**Why a narrow `permissions.deny` plus a session-gated
`PreToolUse` shim.** Claude Code's `permissions.deny` rules
deny a tool call before the agent even sees the opportunity —
they are cheap, static, and enforced by Claude Code directly.
But project-level `permissions.deny` rules apply to *every*
Claude Code session that opens this repo, gaia or not, and the
`UserPromptSubmit` / `PreToolUse` / `Stop` shims classify the
session before doing any gaia-specific work. The `.gaia/`,
`.gaia-state/`, and `.gaia-context/` namespaces are gaia-owned
surfaces that have no legitimate use outside gaia, so blocking
writes there for every session is a free correctness win
(teammate Claude Code sessions never have a reason to write to
gaia-internal namespaces).

Everything else gaia needs to enforce — `.claude/hooks/gaia-*.js`
and `.claude/settings.json` writes, `.git/**` writes, branch-
switch / hard-reset / force-push commands, state-branch pushes,
and MCP-tool denials — is **session-gated** in the shim and
applies only when the shim has classified the session as gaia.
That keeps the "non-gaia sessions run unaffected (assuming Node
on `PATH`)" promise (§3) honest: a teammate running ordinary
Claude Code in a gaia-initialized repo can still
`git checkout other-branch`, edit `.claude/settings.json` to
adjust their own permissions, or rebase against `main` without
hitting a gaia-installed deny.

The shim still does this work because:
- `permissions.deny` cannot distinguish gaia sessions from
  non-gaia sessions; gaia's session-identity-dependent rules
  have to be hook-gated to keep teammates' local sessions
  unrestricted.
- `permissions.deny` Bash patterns are coarser than the shim's
  parser — they do not decode `bash -c`, subshells, or
  redirects. The shim's structural decoder (§8.5 step 4)
  catches those forms.
- State-branch push denial (`claude/gaia/*/state`) is not a
  clean `permissions.deny` pattern because gaia itself (the
  Stop hook) pushes to state branches from inside the same
  session. Only the shim can make the "writes from tool calls
  but not from the hook's own git push" distinction.
- MCP tool calls (`mcp__*`) are banned in gaia v1 (§5)
  because gaia has not yet modeled their write semantics; the
  shim denies every `mcp__*` tool call in gaia sessions as a
  runtime safety net.

**Path anchoring.** Every path in the deny list above uses a
leading `/` to anchor it to the project root
(`$CLAUDE_PROJECT_DIR`). Claude Code's permission syntax
distinguishes project-root-relative paths (`/path`) from
cwd-relative paths (`path` or `./path`); without anchoring,
the rule binds to whatever directory Claude has `cd`'d into,
which can diverge from the repo root mid-session. Anchored
paths are stable under `cd` and match both the shim's
`$CLAUDE_PROJECT_DIR`-rooted checks and the doctor's
end-to-end denial probes.

For the gaia-namespace paths covered by both layers, denial
from either is honored. The `PreToolUse` shim re-checks them
during a gaia session anyway, so a teammate who removes the
static deny from project settings (e.g., to debug something)
still gets the protection inside gaia runs. Drift is contained
by `gaia doctor --cloud`'s `PreToolUse` denial probe (§8.4)
exercising at least one operation from each protected category
end-to-end.

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
the gaia codebase. The Node *language* is cross-platform — the
shim script itself runs unchanged on Linux, macOS, and Windows —
but the `command` entry `node "$CLAUDE_PROJECT_DIR/..."` assumes
a POSIX-style shell for `$`-expansion and quoting; on native
Windows PowerShell sessions, Claude Code requires `"shell":
"powershell"` (or equivalent) for reliable expansion. **v1
supports Linux and macOS driver and teammate environments**, and
uses the POSIX command string above. Windows driver support and
Windows-teammate support are tracked in §14 as follow-up work —
`gaia doctor` on a Windows machine should report the platform is
uncertified rather than claim the setup works. A future release
may add a Windows-specific `command` variant (or the `"shell":
"powershell"` hint) and extend doctor coverage accordingly. The
trade-off even on supported platforms is that Node becomes a
prerequisite on every machine that opens Claude Code in a
gaia-initialized repo (§5) — Claude Code itself does *not*
require Node (the native installer / Homebrew binary works fine
without it), so this prereq is gaia-imposed. Future work (§14)
may add a POSIX `.sh` opt-in for repos that want to drop the Node
prereq for Linux/macOS teammates.

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
  matchers (and `Bash` subcommand `if` rules); gaia uses the
  wildcard matcher `"*"` and classifies inside the shim. Scoping
  narrowly with a tool-name regex would create future-tool
  blind spots — new write-capable tools (built-in GitHub
  tools that post PR comments, `WebFetch`, `WebSearch`, any
  successor of the `Agent` subagent tool, and MCP tools a
  misconfigured routine managed to surface) would skip the
  shim entirely. With `"*"`, every tool call from a gaia
  session goes through the shim's classification (§8.5); the
  overhead on non-gaia teammate sessions is a single file stat
  plus an exit-0 (§8.0 step 2-3) and is bounded by Node startup
  per call, which `gaia doctor` (§8.4) measures so regressions
  are visible.
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
   the `protocol_version` it understands (a small integer baked in
   at `gaia init` time). Each global `gaia-hook-*` binary advertises
   the same `protocol_version` it implements. If the shim's pinned
   `protocol_version` differs from the global binary's
   `protocol_version` (the routine VM may have an older
   environment-cached install per §5; teammates may have an older
   global install), the shim fails closed with the same path as
   "binary missing" below — the driver gets a fast, explicit
   failure rather than a silent protocol mismatch. The shim does
   **not** compare driver/binary semvers — only `protocol_version`.
   Patch and minor releases that ship under the same
   `protocol_version` are interoperable, so a freshly-published
   gaia version reaching the VM 7 days late (Claude Code's setup-
   script cache window per §5) does not fail-close as long as the
   protocol field still matches.
5. Otherwise locate the global `gaia-hook-*` binary on `$PATH` and
   spawn it with the original stdin, piping stdout/stderr through.
   Exit with the child's exit code.
6. **Fail-closed when the global binary is missing or version-skewed
   for a gaia session.** A gaia session that cannot reach a
   compatible finalizer must not silently succeed — that would push
   no sentinel and the driver would time out after 45 minutes with
   no useful error.
   - For `UserPromptSubmit`: the shim is in a good position to
     surface the failure as a `GAIA_FAILED:` sentinel instead of
     forcing a 45-minute timeout. It has already parsed the control
     block (so it knows `GAIA_RUN_ID`) and validated the manifest
     (so it has `session_id` from Claude Code stdin, plus the state-
     branch name). The shim attempts a best-effort sentinel push
     before blocking the prompt:
     1. Write `.gaia-state/<session_id>.error.json` with
        `{created_by: "gaia", schema_version: 1, error:
        "gaia-hook-binary-missing-or-version-skew", expected_protocol_version: <n>}`
        to the state branch via the §8.2 step 10 name-matched
        local-branch flow (same commands, committed with
        `GAIA_FAILED: <session_id>`). If the push succeeds, the
        driver reads the sentinel and exits 13 in seconds instead
        of timing out at 45 minutes.
     2. Then emit a `decision: "block"` JSON output with a clear
        reason ("gaia hook binary not installed in this
        environment, or protocol-version mismatch — install
        `@your-scope/gaia@<expected>` globally") and exit 0, which
        blocks the prompt per Claude Code's `UserPromptSubmit`
        decision-control semantics.
     The sentinel push is best-effort: if git is unreachable from
     the shim environment, the fence ran but the manifest fetch
     failed, or the state-branch push is rejected, the shim falls
     through to the block-and-timeout path above (driver exits 3
     after 45 minutes, `gaia recover` surfaces the session URL for
     diagnosis). This path only fires when the shim has already
     *fully validated* the manifest (fence parsed, nonce matched,
     work-branch / run-id agreed); a pre-validation failure uses
     the plain block-path instead, because at that point the shim
     cannot trust the `session_id` / state-branch it would be
     writing to.
   - For `PreToolUse`: emit the documented deny envelope —
     ```json
     {
       "hookSpecificOutput": {
         "hookEventName": "PreToolUse",
         "permissionDecision": "deny",
         "permissionDecisionReason": "gaia hook binary not installed in this environment, or version mismatch — install `@your-scope/gaia@<expected>` globally"
       }
     }
     ```
     and exit 0 on every tool call from a gaia session. The shim
     uses the `hookSpecificOutput.permissionDecision` shape
     current Claude Code documents for `PreToolUse`; the older
     top-level `decision: "deny"` shape is deprecated and maps
     "block" (not "deny") to a denial on the legacy path, so
     emitting top-level `"deny"` on a current Claude Code version
     silently does nothing. The session cannot make progress and
     terminates; the driver times out the same way. This is
     preferable to fail-open — a gaia session running without
     protection is exactly what `PreToolUse` exists to prevent.
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

This means a teammate who opens Claude Code on the repo without
the global gaia binary installed sees no disruption for their
own (non-gaia) prompts **as long as Node is on `PATH`**: the
shim runs, classifies the session as non-gaia, exits 0. If Node
is *not* on `PATH`, the `command` entry fails to start and
Claude Code reports a hook error regardless of whether the
prompt was gaia — §5 documents Node 20+ as a hard prerequisite
for any environment opening Claude Code in a gaia-initialized
repo for exactly this reason. The "non-gaia sessions run
unaffected" property is conditional on Node being available; it
is not unconditional. A gaia session in the same environment
(with Node but without the global gaia binary) fails explicitly
rather than silently — which is what the driver wants.

**Protocol-version invariant.** The committed shim's
`protocol_version`, the manifest's `protocol_version` (§7.4),
and the global `gaia-hook-*` binary's `protocol_version` must
all agree per run. The driver records both `protocol_version`
(load-bearing) and `driver_version` (informational) into the
manifest at fire time; the shim reads the manifest after
fetching the state branch and refuses to proceed if
`manifest.protocol_version` differs from the shim's pinned
value. `driver_version` is recorded for diagnostics and is not
compared. The `UserPromptSubmit` hook is the authoritative
gate — `PreToolUse` and `Stop` rely on the session-tmp file
having been created earlier in the session, which implies the
gate already passed.

**Node is a prerequisite for the shim, not for Claude Code.** Claude
Code itself can run with no Node on `PATH` (native installer / Homebrew
ship a binary that does not invoke Node). The Node prereq is specific
to gaia-initialized repos because the committed shims are Node
scripts and the `command` entries start with `node`. If Node is not
on `PATH`, the hook command fails to start — disrupting the session
regardless of whether the prompt was gaia or not. §5 documents this
as a prerequisite for any environment opening Claude Code in a
gaia-initialized repo. `gaia init` prints a banner so repo owners
can communicate this expectation. v1 certifies Linux and macOS —
driver and teammate. Windows is not certified in v1; the POSIX
shell expansion used in the `command` string (e.g.,
`$CLAUDE_PROJECT_DIR`) is not guaranteed under native PowerShell
without a `"shell": "powershell"` hint, and `gaia doctor` does not
yet run on Windows. A Windows teammate opening Claude Code in a
gaia-initialized repo is likely to see the hook command fail to
start. Windows support is tracked in §14 as v1.x work, alongside a
POSIX `.sh` opt-in for repos that want to drop the Node prereq on
Linux/macOS.

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
3. Fetch the derived state branch with an explicit refspec, then
   read `.gaia-state/manifest.json` directly out of the fetched
   ref via `git show` (no checkout, no temporary worktree):
   ```
   git -C "$CLAUDE_PROJECT_DIR" fetch origin \
     "+refs/heads/$GAIA_STATE_BRANCH:refs/remotes/origin/$GAIA_STATE_BRANCH"
   git -C "$CLAUDE_PROJECT_DIR" show \
     "refs/remotes/origin/$GAIA_STATE_BRANCH:.gaia-state/manifest.json"
   ```
   Refuse if the manifest is missing, if `manifest.nonce` does not
   match `GAIA_NONCE`, if `manifest.work_branch` does not match the
   prompt's `GAIA_WORK_BRANCH`, if `manifest.run_id` does not match
   `GAIA_RUN_ID`, or if `manifest.protocol_version` differs from
   this shim's pinned `protocol_version` (§8.0).
   `manifest.driver_version` is recorded for diagnostics and is
   not compared. Refusal uses the same `{"decision": "block",
   "reason": ...}` path as step 2.
4. Write `<runtime-dir>/<session_id>.json` with
   `{run_id, work_branch, state_branch, experimental_mode,
   driver_version, managed_file_hashes}`, where `<runtime-dir>`
   is `$XDG_RUNTIME_DIR/gaia/` when set and `/tmp/gaia-$uid/`
   otherwise. `managed_file_hashes` is copied from
   `manifest.managed_file_hashes` (§7.4) and consumed by §8.2
   step 6's integrity spot-check. The directory is created with
   mode `0700`; the file is written with mode `0600`,
   `O_NOFOLLOW`, and `session_id` is rejected if it does not
   match `^session_[0-9A-Za-z_-]+$` (defense against path
   traversal). The tmp file is the **operational source of
   truth** for the `Stop`, `StopFailure`, and `PreToolUse`
   hooks during the session — they key by `session_id` and
   read branch names and managed-file hashes from this file
   rather than from any git config the agent could overwrite.
   The file is **not secret** and is not tamper-proof against
   a hostile in-VM actor with shell access (§16.1); the
   directory permissions and `O_NOFOLLOW` are about preventing
   accidental cross-user reads and symlink attacks, not about
   authenticating the agent.
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
   GitHub proxy's empirically-observed "push to current working
   branch" check (§8.4 cloud check 3 verifies this end-to-end on
   the live transport) is satisfied for any subsequent push from
   this branch. After
   checkout, `.claude/hooks/gaia-*.js` and `.claude/settings.json`
   on the worktree are guaranteed by the driver's synthesis step
   (§6.1 step 4, §8.8).
7. If the payload contains `--- CONTEXT FILE ---`, fetch the state
   branch, copy `.gaia-state/context.md` into the worktree at
   `.gaia-context/<run_id>/context.md` (the `.gaia-context/` rule
   in `.gitignore` keeps it out of any commit; `.gaia-context/` is
   a read-allowed, write-denied namespace per §8 so Claude can
   read the file but tool calls cannot mutate it), and emit
   `additionalContext` pointing Claude at the file:
   ```json
   {
     "hookSpecificOutput": {
       "hookEventName": "UserPromptSubmit",
       "additionalContext": "The user prompt references .gaia-context/<run_id>/context.md. Read that file before acting — it holds the task content that did not fit inline."
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
3. **Verify the current branch is the work branch.** Run
   `git -C "$cd" symbolic-ref --short HEAD` (and fall back to
   `git rev-parse HEAD` if HEAD is detached). If the current
   branch is not `$GAIA_WORK_BRANCH`, the hook does **not**
   reset the local work branch to the remote tip — doing so
   would destroy any commits the session made locally before
   ending up on a different branch. Instead:
   1. Capture the current HEAD commit sha.
   2. Push that sha to a rescue ref:
      ```
      git -C "$cd" push origin \
        "$CURRENT_HEAD:refs/heads/claude/gaia/$GAIA_RUN_ID/rescue/$session_id"
      ```
      The `claude/gaia/*` namespace is in the proxy's default
      push allowlist, so the rescue push is authorized.
   3. Skip the defensive-restore and final sweep (steps 4–5 below)
      and the work-branch push (step 7 below).
   4. Fall through to step 10 to push a `GAIA_FAILED:
      <session_id>` sentinel carrying
      `{error: "work-branch-not-current",
      current_branch: "<whatever HEAD was>",
      rescue_ref: "claude/gaia/$GAIA_RUN_ID/rescue/$session_id"}`,
      and exit 1.
   5. `gaia recover` surfaces the rescue ref in its output so
      the user can inspect or merge it manually. The driver
      does not auto-merge rescue refs — the reason for the
      wrong-branch state is unknown, and a blind merge could
      land unintended commits.

   The `PreToolUse` hook (§8.5) blocks `git checkout` /
   `git switch` / `git reset --hard` for gaia sessions, so this
   fallback is defense-in-depth rather than a routine path; the
   rescue-ref approach is conservative for the rare cases where
   something upstream of the hook (a shell escape the parser
   missed, a hostile prompt injection that evaded `PreToolUse`,
   a Claude Code bug) leaves the session's HEAD somewhere
   unexpected.
4. **Defensive restore of gaia-managed files.** Before the sweep,
   restore the gaia-managed files from HEAD — this reverts any
   modifications the session made to them (which `PreToolUse`
   should have blocked, but defense in depth). These files are
   known-good on the work-branch HEAD because §8.8 synthesis
   ran against the default-branch copies before `/fire`:
   ```
   git -C "$cd" checkout HEAD -- \
     .claude/hooks/gaia-user-prompt-submit.js \
     .claude/hooks/gaia-pre-tool-use.js \
     .claude/hooks/gaia-stop.js \
     .claude/settings.json 2>/dev/null || true
   ```
   The `|| true` handles the case where the agent deleted one
   of these files (subsequent sweep will catch the delete and
   restore via checkout-from-HEAD is already done).

   **`.gitignore` is reconciled at the line level, not restored
   wholesale.** The agent may have legitimately added ignore
   rules during the session (e.g., a new build-output directory).
   Wholesale `git checkout HEAD -- .gitignore` would discard
   those edits. Instead, the hook re-reads `.gitignore`, ensures
   the gaia-owned ignore lines (`.gaia/` and `.gaia-context/`)
   are present and exact, leaves all other lines alone, and
   stages the file only if a gaia-owned line was missing or
   altered. Order is preserved; gaia inserts its lines at the
   bottom if absent. The integrity check (step 6) verifies the
   gaia-owned ignore subset, not the whole `.gitignore` file
   (§7.4 records a normalized hash of just the gaia ignore
   entries for this reason).
5. **Final commit sweep (conditional, no push yet).**
   Root every command at `$CLAUDE_PROJECT_DIR` and exclude
   `.gaia/`, `.gaia-state/`, and `.gaia-context/` from staging
   via explicit pathspec exclusions — the agent may have
   modified `.gitignore` during the session, and belt-and-
   suspenders pathspec exclusions keep gaia-owned paths out of
   the commit regardless.
   ```bash
   cd="$CLAUDE_PROJECT_DIR"
   excludes=( ':(exclude).gaia/**' ':(exclude).gaia-state/**' ':(exclude).gaia-context/**' )
   if [ -n "$(git -C "$cd" status --porcelain -- . "${excludes[@]}")" ]; then
     git -C "$cd" add -A -- . "${excludes[@]}"
     git -C "$cd" commit -m "gaia: final sweep"
   fi
   ```
   The sweep does not push here — the work branch push happens
   only after the integrity check in step 6 passes. Earlier
   drafts pushed first and then verified, which made the
   "do not push the work branch on integrity failure" check in
   step 6 impossible to honor (the push had already happened).
6. **Integrity spot-check on gaia-managed files.** After the
   sweep commit (or after confirming no sweep was needed),
   compute the sha256 of each gaia-managed file in the tree
   and compare to the expected hashes recorded in
   `manifest.managed_file_hashes` (§7.4, populated at fire time
   from the default-branch copies). For `.gitignore`, compare a
   normalized hash of the gaia-owned ignore entries
   (`manifest.managed_gitignore_hash`) rather than the whole
   file, so legitimate non-gaia edits do not trigger a
   mismatch. If any hash mismatches, push a `GAIA_FAILED:`
   sentinel with
   `{error: "managed-files-tampered", mismatches: [...]}` via
   the §8.2 step 10 state-branch push path and exit 1 — **do
   not push the work branch** in this path. This catches the
   case where step 4's restore missed a path (e.g., a newly-
   added hook file the driver's version does not know about)
   or a `PreToolUse` denial was bypassed. Protocol mutations
   cannot flow back to the user branch via the merge-back
   that way.
7. **Push the work branch (unconditional, only on integrity
   pass).**
   ```bash
   git -C "$cd" push origin "$GAIA_WORK_BRANCH:$GAIA_WORK_BRANCH"
   ```
   The push is **unconditional** even if the sweep added no new
   commits. Earlier drafts only pushed when the working tree was
   dirty, which dropped clean local commits Claude had created
   but not pushed — those are now always synced to origin. The
   push uses an explicit `<src>:<dst>` refspec so the Claude
   Code GitHub proxy's empirically-observed "push to current
   working branch" rule (§8.4) sees a name-matched refspec from
   the same-named local branch (verified in step 3).

   **Rescue-artifact fallback on push failure.** If the work-
   branch push is rejected (auth glitch, proxy permissions
   failure, partial network timeout), the hook does **not**
   abort the sentinel write — the driver would then time out at
   45 min and the session's commits would only live on the
   cloud VM's disk, which is recycled on session end. Instead
   the hook captures the missing commits into rescue artifacts
   that ride on the state-branch sentinel commit:
   ```bash
   head_sha=$(git -C "$cd" rev-parse HEAD)
   base=$(git -C "$cd" merge-base HEAD "refs/remotes/origin/$GAIA_WORK_BRANCH" 2>/dev/null \
          || git -C "$cd" rev-parse "$MANIFEST_BASE_SHA")
   git -C "$cd" diff "$base..HEAD" > "$tmp/.gaia-state/$session_id.rescue.patch"
   git -C "$cd" log --oneline "$base..HEAD" > "$tmp/.gaia-state/$session_id.rescue.log.txt"
   printf '%s\n' "$head_sha" > "$tmp/.gaia-state/$session_id.rescue.head.txt"
   git -C "$cd" bundle create "$tmp/.gaia-state/$session_id.rescue.bundle" \
                              "$base..HEAD" \
                              --branches="$GAIA_WORK_BRANCH" 2>/dev/null || true
   ```
   Each rescue file carries the §7.3 header format and the
   files are added to the same temporary worktree the hook
   uses for the state-branch write (step 10 below). The
   sentinel commit message is unchanged (`GAIA_DONE:
   <session_id>` on clean exit; `GAIA_FAILED:` or
   `GAIA_ABORTED:` on the respective terminal paths); the
   `.rescue.*` files appear alongside the normal final-message
   and transcript artifacts. `gaia recover` uses the bundle
   (preferred) or the patch to reconstruct the missing
   commits on the driver side (§6.4 / §7.3). If the bundle
   create itself fails (e.g., due to missing objects in the
   VM's repo), the `|| true` lets the rest of the rescue
   bundle proceed; the patch + log + head trio is already
   enough for a best-effort human-driven reconstruction.
   The `manifest.base_sha` (§7.4) is the authoritative
   fallback for `$base` when no origin ref exists yet
   (first-push scenario, though this case is vanishingly
   rare by the time Stop fires).
8. **Extract the final assistant message** from the stdin
   `last_assistant_message` field. If missing or empty, fall back to a
   reverse-scan of the JSONL transcript (looking for the last line
   where `message.role === "assistant"` or `type === "assistant"`,
   concatenating text blocks). Last-resort fallback: the raw last line
   of the JSONL — still human-readable and the driver will print it.
9. **Build the readable transcript** by one forward pass through the
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
10. **Write both artifacts** via a temporary worktree seeded at the
    state-branch tip as a **real local branch with the same name as
    the remote target**, then push with an explicit name-matched
    refspec. The Claude Code GitHub proxy is empirically observed
    to restrict git push operations to a refspec whose remote target
    matches the session's current working branch (§8.4 verifies
    this on the live transport); using a different local name
    (e.g., `gaia-state-$GAIA_RUN_ID`) and pushing to a differently-
    named remote ref is exactly the case the proxy is observed to
    reject. The `-B "$GAIA_STATE_BRANCH"` on `git worktree add`
    plus an explicit `<src>:<dst>` refspec where `src == dst`
    removes the ambiguity:
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
    Each `.gaia-state/*.md` and `*.error.json` artifact begins
    with a small header (`created_by: gaia`, `schema_version: 1`)
    so future migrations can detect and translate older artifact
    shapes; readers tolerate a missing header (the v1 release
    is the first one writing it).
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
11. **Cleanup the runtime session file** with `unlink
    <runtime-dir>/<session_id>.json` (best-effort; ignore errors).
    The file is written by `UserPromptSubmit` (§8.1 step 4) and
    has no purpose after the terminal sentinel lands. Leaving it
    behind is not a security issue (the directory is mode `0700`
    and the VM is single-tenant), but it accumulates per-session
    files on long-lived VMs and confuses post-mortem inspection.
12. Exit 0.

The sentinel push is the only way the driver learns the session is
done; the hook must not exit before completing step 10.

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
2. Best-effort attempt at the §8.2 step 3 current-branch check,
   the §8.2 steps 4–5 defensive-restore + final-sweep, and the
   §8.2 step 7 work-branch push. The API error may have damaged
   the VM state; failures here are logged but do not block the
   failure sentinel. The integrity spot-check (§8.2 step 6) is
   skipped on StopFailure — landing a sentinel is higher priority
   than blocking the final-message push on a hash mismatch when
   the session is already failing.
3. Writes `.gaia-state/<session_id>.error.json` containing
   `{created_by: "gaia", schema_version: 1, error,
   error_details, last_assistant_message}` and
   `.gaia-state/<session_id>.transcript.md` (whatever transcript
   content exists) to the state branch via the same name-matched
   local-branch temporary-worktree flow as §8.2 step 10. The
   worktree's local branch is `$GAIA_STATE_BRANCH` (matching the
   remote name) and the push uses an explicit
   `$GAIA_STATE_BRANCH:$GAIA_STATE_BRANCH` refspec. Retry on
   non-fast-forward, bounded at 3 attempts.
4. Commits with message `GAIA_FAILED: <session_id>` and pushes.
5. Cleanup `<runtime-dir>/<session_id>.json` (best-effort `unlink`,
   ignore errors) — same rationale as §8.2 step 11.
6. Exits 0 on success, exits 1 on any error. The exit code does not
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

`gaia doctor` runs in tiers, because the driver machine and the
cloud VM are separate environments with overlapping-but-not-
identical surfaces. The driver machine has no guarantee of a
local `claude` binary (§1 promises teammates can call `gaia`
without one), so any check that depends on exercising Claude
Code locally must be opt-in.

**Tiers:**

- `gaia doctor --cloud` — the **required** tier. Fires a
  synthetic cloud session via the configured routine and
  exercises everything that can be verified end-to-end in the
  VM: spawner contract, work/state branch push permissions,
  hook schema, `PreToolUse` enforcement, MCP prohibition,
  managed-policy honoring *on the VM*, `StopFailure`
  registration, beta-header freshness. This is what `gaia
  init` offers to run at the end of setup and what users
  should run before relying on the pipeline. Requires no
  local `claude` install.
- `gaia doctor --local` — optional. Requires `claude`
  installed on the driver machine. Exercises the local
  teammate surface: that project-level hooks fire in a local
  `claude` session (i.e., managed policy on *this* machine
  does not disable them), that the Node shim classifies
  non-gaia sessions correctly, and that the Node prerequisite
  is satisfied on `PATH`. If `claude` is not installed, this
  tier prints a short banner naming the checks it would run
  and exits 0 — the surface it covers is teammate-local, so
  a driver machine without `claude` is legitimately unable
  to verify it.
- `gaia doctor --full` — runs both tiers. Fails if `--local`
  would fail.
- `gaia doctor` with no tier flag defaults to `--cloud`. The
  `gaia init` completion hook runs `--cloud` unless the user
  passes `--full` on the init command line.

The tiers share a single report format; a check performed in
the cloud tier does not repeat in the local tier.

**Cloud tier checks (`gaia doctor --cloud`).** Every check below
runs inside a cloud session that has cloned the configured
repo at its default branch. The managed-policy probe therefore
reflects what `/fire` will experience in practice — not a temp
directory with a scratch repo.

Each check is tagged with one of three verification levels so
operators understand what doctor can and cannot prove:

- **[verified]** — doctor exercises the behavior end-to-end
  and observes the result directly. A pass here is a strong
  guarantee the check held at probe time.
- **[inferred]** — doctor exercises a synthetic surface and
  infers the result from its behavior. A pass means the
  inference held in the synthetic case; the production case
  may still differ at the margins.
- **[manual]** — doctor cannot programmatically verify the
  prerequisite from inside the cloud VM. The probe falls back
  to a structural check (e.g., "did the synthetic session
  see this surface?") and the report flags the limitation in
  the failure / "unexercised" state.

The tags appear in the doctor report alongside each row so the
operator knows what evidence backs a green check.

1. **[verified]** **Hook-honored check on the VM.** Doctor's
   synthetic `/fire` includes a `GAIA_DOCTOR_BEACON: <nonce>`
   line inside the `--- BEGIN GAIA CONTROL ---` block that the
   `UserPromptSubmit` shim echoes into a beacon commit on the
   doctor run's state branch if it fires. If the beacon does
   not appear within the usual poll window, hooks did not fire
   on the VM and doctor fails loudly. Running the probe against
   the *configured* repo and default branch (rather than a
   scratch directory) catches the "policy attached to this
   environment specifically" case that a temp-directory probe
   would miss.

   **[manual]** **Identifying *why* hooks did not fire is
   best-effort.** When `disableAllHooks: true` or
   `allowManagedHooksOnly: true` in a managed-policy layer
   blocks gaia's project hooks from firing, gaia's project hook
   is precisely the surface that cannot report what blocked it.
   Doctor therefore reports "hooks did not fire on the VM —
   most likely a managed-policy layer blocking project hooks
   (`disableAllHooks` or `allowManagedHooksOnly`)" without
   naming the specific setting / layer. The remediation guide
   points at the Claude Code managed-policy docs and the
   environment's settings UI.
2. **[verified]** Spawner config is valid: 200 on `/fire`,
   correct response field names
   (`claude_code_session_id`, `claude_code_session_url`).
3. **[verified]** Cloud repository access is configured and the
   session can push to both `claude/gaia/<doctor_run_id>/work`
   **and** `claude/gaia/<doctor_run_id>/state` from inside the
   VM. This is the release gate for §8.2 step 10's name-matched
   local-branch state push — if the proxy rejects the state-
   branch write, doctor fails with the exact remediation and
   gaia is unusable on this repo until it is fixed.
4. **[verified]** **`UserPromptSubmit.prompt` content shape.**
   The doctor's `/fire` includes a unique fingerprint inside
   the `--- BEGIN GAIA CONTROL ---` block. The doctor hook
   records the *full* `prompt` field it receives and pushes a
   sample to a doctor-only state ref so the driver can verify
   whether Claude Code prepended the saved routine prompt,
   appended it, wrapped it, or passed `text` through verbatim.
5. **[verified]** The work branch the session sees after
   `UserPromptSubmit` checkout contains `.claude/hooks/
   gaia-stop.js`, `.claude/hooks/gaia-pre-tool-use.js`,
   `.claude/hooks/gaia-user-prompt-submit.js`, and
   `.claude/settings.json` — including the gaia-namespace
   entries in the `permissions.deny` block (§8) — at the current
   driver's expected version. The doctor hook computes sha256
   of each gaia-managed file and compares to the expected
   values passed in via the doctor's `--- BEGIN GAIA CONTROL
   ---` block.
6. **[inferred]** **No MCP tools surfaced to the gaia session.**
   The doctor hook records every `tool_name` it sees on
   `PreToolUse` invocations during the synthetic run and
   pushes the list to a doctor-only state ref. The synthetic
   prompt is engineered to invite tool use across the typical
   surfaces (Read, Write, Bash) and to attempt an MCP call by
   name; if any `mcp__*` `tool_name` was observed, doctor
   fails with the specific MCP tool name. **This does not
   enumerate the full tool catalog** — Claude Code's hook
   docs surface the current `tool_name` on each call, not a
   global catalog, so the inference holds only for the tools
   the synthetic run *exercised*. The PreToolUse shim's
   runtime ban (§8.5) catches `mcp__*` calls that the
   synthetic run missed; doctor's check is a high-confidence
   inference, not an exhaustive enumeration. The `gaia init`
   banner (§5, §6.4) remains the authoritative check that the
   environment / routine has no MCP servers configured at all.
7. **[verified]** **Protocol-version invariant.** The committed
   shim and the global binary on the VM advertise the same
   `protocol_version`. Catches the cloud environment-cache
   skew problem when the shim has bumped to a new
   `protocol_version` since the cached install: setup scripts
   cache for ~7 days, so a freshly published gaia version with
   a `protocol_version` bump may not reach the VM until the
   cache rolls over. Patch / minor releases that share
   `protocol_version` are not flagged here.
8. **[verified]** **Shim fails closed when the manifest's
   `protocol_version` does not match.** The doctor's
   `UserPromptSubmit` payload commits a manifest with a
   deliberately-mismatched `protocol_version` value, and
   doctor verifies the shim's `{"decision": "block"}` output
   fires when the manifest `protocol_version` disagrees with
   the shim's pinned value. The probe does not temporarily
   shadow `$PATH` (the local driver cannot reach into the
   cloud VM's environment to do that); shim-without-binary
   coverage on teammate machines is the local tier's job.
9. **[verified]** The `UserPromptSubmit` hook receives the
   expected stdin fields on the current Claude Code version.
10. **[verified]** `last_assistant_message` is present on the
    `Stop` input on this Claude Code version.
11. **[verified]** The JSONL transcript on this version parses
    into the expected forward-pass output.
12. **`StopFailure` is wired up.** Triggering a real `StopFailure`
    on demand requires producing a Claude Code-side API error
    (rate-limit, overloaded, auth) *inside* an established
    session, which is not something doctor can reliably engineer:
    both the error rate and the ability to force one depend on
    Anthropic-side quota / backend state that varies day-to-day.
    Doctor therefore only promises a static probe here, with a
    live probe available as an opt-in best-effort smoke test:
    - **[manual]** **Static probe (always-on, this is the v1
      release gate):** doctor confirms the project-settings
      `.claude/settings.json` block on the *work-branch tip*
      registers gaia's handler for `StopFailure` and that the
      spawner contract step (check 2 above) succeeded against a
      Claude Code version known to support `StopFailure`. Doctor
      cannot enumerate the local Claude Code build's full hook-
      event catalog from the routine surface; absence of an
      error on the documented hook registration syntax is
      treated as inferential evidence of support, not direct
      proof. The live `StopFailure` sentinel path is recorded
      as "unexercised" in this mode. The implementation-level
      confidence that the `StopFailure` handler behaves correctly
      comes from the gaia test suite's mocked-stdin integration
      tests (§8.3 input shape + §7.3 artifact format), not from
      the live probe.
    - **[optional, best-effort]** **Live probe (`gaia doctor
      --live-stop-failure`):** when Anthropic is currently
      returning overloaded errors in this account (typical during
      peak hours for some plans), doctor fires a session whose
      payload is engineered to invite a `StopFailure` mid-turn
      and checks whether `GAIA_FAILED:` lands. This path is
      **not** a release gate — a passing run is extra evidence,
      a failing run (e.g., because no overloaded_error was
      actually returned) is not a bug. Doctor reports which it
      was and points at the static probe result as the
      authoritative answer. Consumes real quota; gated behind
      an explicit flag. A payload rejected by `/fire` itself
      (e.g., one that exceeds the documented 65,536-character
      cap) is rejected before any session is created and does
      **not** produce a `StopFailure` — the probe must trigger
      the failure inside an established session, which is what
      makes it unreliable.
13. **[verified]** **`PreToolUse` denials work end-to-end.**
    Doctor fires a session whose prompt instructs Claude to
    attempt one operation from each gaia-protected category
    (edit `.claude/hooks/gaia-stop.js`, edit `.gaia-state/x`,
    edit `.gaia-context/x`, `git checkout main`, `git push
    origin claude/gaia/<id>/state`, and — if any MCP tool
    surfaces — an `mcp__*` call) and confirms the `PreToolUse`
    shim's `hookSpecificOutput.permissionDecision: "deny"`
    response is honored. If any denial is bypassed, the
    protection in §8.5 is not load-bearing on this Claude
    Code version — this is a release gate because the
    deprecated top-level `{"decision": "deny"}` envelope
    silently no-ops on the modern path.
14. **[verified]** No foreign project-level handler has appeared
    on `UserPromptSubmit`, `Stop`, or `StopFailure` since
    install (§8.7). For `PreToolUse`, doctor checks for
    matcher overlap with gaia's protected matchers and warns
    rather than fails. **[manual]** Managed-policy and
    plugin-scope handlers cannot be enumerated from inside
    the repo; doctor explicitly notes this gap in the report
    so the operator knows project-layer cleanliness does not
    rule out higher-priority handlers attached out-of-band.
15. **[verified]** **Beta header staleness.** The configured
    `spawner_config.beta_header` (§10.1) is still accepted by
    `/fire`. Doctor fires a minimal synthetic request with the
    configured header; if the response or error envelope
    indicates the header is unknown, deprecated, or downgraded,
    doctor reports a stale-beta-header condition. Warning the
    first time, fail on the second so stale-header conditions
    do not accumulate silently.
16. **[verified]** **`.gaia/merge.lock` works on this filesystem.**
    Acquires and releases the lock in-process, confirms
    blocking behavior with a second process (subprocess), and
    falls back with a loud warning if the platform lacks
    working `flock` / `LockFileEx`. A filesystem without
    working advisory locks (rare — typically network shares
    or filesystems mounted `-o nolock`) means same-checkout
    concurrency guards degrade to best-effort. Not a release
    gate.
17. **[verified]** **Routine / repo binding.** The driver pushes a
    beacon branch `claude/gaia/doctor-<nonce>/binding` to its
    local origin and includes the expected repo `owner/name` in
    the `--- BEGIN GAIA CONTROL ---` block. The `UserPromptSubmit`
    hook on the VM reads the cloned repo's `origin/url` (via
    `git -C "$CLAUDE_PROJECT_DIR" config remote.origin.url`),
    normalizes it, compares to the expected `owner/name`, and
    fetches the beacon branch to confirm it is present. If the
    VM's clone is of a different repo or lacks the beacon,
    doctor fails with the specific mismatch (expected vs.
    observed `owner/name`, beacon-fetch result) and points at
    the routine's configuration as the likely culprit. This is
    the end-to-end check that the routine is attached to
    exactly the same GitHub repo as the driver checkout's origin
    (§5 prerequisite). Multi-repo routines fail this check even
    if one of the cloned repos matches, because a single gaia
    session may end up cloning the wrong one on any given fire.
18. **[verified]** **No project `.mcp.json` visible to the VM.**
    The doctor hook lists the cloned repo's tree and fails if
    a `.mcp.json` exists at any path the cloud session loads
    MCP configuration from. Covers the §2 / §5 ban on project-
    scoped MCP. A warning-not-fail alternative was considered
    and rejected: a project `.mcp.json` that the driver has not
    characterized can inject tools that slip past the shim's
    `mcp__.*` deny precisely because the MCP binding happens
    above the matcher.
19. **[verified]** **No project plugin entries in
    `.claude/settings.json` on the work branch.** Doctor reads
    the settings block the VM loaded and fails if any key under
    the documented plugin-loading surface (`plugins`,
    `extraKnownMarketplaces`, or equivalent successor names)
    has a non-empty value. Same reasoning as `.mcp.json`:
    plugins can register hooks and MCP servers outside gaia's
    visibility.
20. **[manual]** **Enumerate foreign project hooks.** Doctor
    lists every project-level hook registered on every event
    outside gaia's four (`SessionStart`, `PostToolUse`,
    `SubagentStop`, `Notification`, `ConfigChange`,
    `WorktreeCreate`, etc.) and prints them in the report so
    the operator sees exactly what runs alongside gaia. This
    is not a pass/fail check — any of these can legitimately
    coexist — but it is mandatory output so surprises do not
    hide in the managed-subset reconcile. `SessionStart`
    specifically is called out as higher-risk because it runs
    before `UserPromptSubmit` and can mutate worktree state
    gaia assumes is stable.
21. **[verified]** **Subagent tool-call coverage.** Doctor's
    synthetic payload prompts the session to launch a subagent
    (e.g., via the `Agent` tool) that attempts one operation
    from each gaia-protected category. The `PreToolUse` shim
    keys by `session_id` from hook stdin; subagent tool calls
    carry the parent session's `session_id` in current Claude
    Code, so gaia's protections should apply. Doctor verifies
    this end-to-end by observing the expected `permissionDecision:
    "deny"` for each attempted write. If the protections do not
    apply inside subagents on this Claude Code version, doctor
    fails with a pointer at gaia's future-work tracking for
    subagent gating (§14). Not exercised if the synthetic
    session refuses to launch a subagent (rare; doctor retries
    with an explicit instruction before falling through to
    "unexercised").
22. **[verified]** (If `--experimental-mode` is configured)
    the experimental-mode flow from §15.7 completes.

**Local tier checks (`gaia doctor --local`).** These all require
`claude` installed on the driver machine:

L1. **Managed-policy / hook-honored check on *this* machine.**
    The local analogue of cloud step 1. Registers a trivial
    test hook in a temp directory and confirms it fires via
    `claude -p`. If the local Claude Code has `disableAllHooks:
    true` or `allowManagedHooksOnly: true` such that project
    hooks are inert, the driver machine cannot open a gaia-
    initialized repo locally without disrupting the session;
    teammates on this machine will see hook-start failures.
    Fails the local tier; the cloud tier is unaffected.
L2. **Node on `PATH`.** The committed shims start with `node`
    (§8.0). If Node 20+ is not on the local shell's `PATH`,
    any `claude` session that opens this repo will fail at
    hook-start regardless of whether the session is gaia.
    Fails the local tier.
L3. **Shim classifies non-gaia sessions correctly.** Opens a
    local `claude` session with a plain prompt (no `--- BEGIN
    GAIA CONTROL ---` fence) and confirms the shim exits 0
    silently — no `decision: "block"`, no `PreToolUse` denies,
    no `Stop`-hook sentinel push. This is the "teammate opens
    Claude Code in the repo without gaia installed" path; it
    must not disrupt their session.
L4. **Shim fails closed on the driver when the global binary
    is missing.** Temporarily shadows `$PATH` so the
    `gaia-hook-*` binary is not resolvable, opens a local
    `claude` session with a gaia-authored prompt (via
    doctor's internal fire-helper that bypasses the routine),
    and confirms the shim blocks with the documented
    fail-closed output (§8.0 step 6). Local-only because the
    `$PATH` shadow is a local driver operation — the cloud
    tier covers the version-skew case via the protocol
    version check (cloud step 8).

`gaia init` offers to run `gaia doctor --cloud` (the required
tier) automatically before declaring setup complete; it
prints a hint to run `gaia doctor --local` on any machine
where teammates plan to open the repo in `claude`.

### 8.5 `gaia-hook-pre-tool-use` (PreToolUse event)

Fires before **every** tool call from a session (the settings
matcher is `"*"` per §8). For non-gaia sessions the shim exits 0
immediately and the call proceeds normally. For gaia sessions the
shim **denies** any tool call that would mutate gaia-managed
protocol state or any non-read-only surface outside gaia's known
allow-list, and **allows** every known read-only tool and every
known safe operation.

**Why this exists.** The `Stop` and `StopFailure` shims execute
`$CLAUDE_PROJECT_DIR/.claude/hooks/gaia-stop.js` from the *work
branch's* working tree (§8.8). Without `PreToolUse` enforcement,
Claude could `Edit` or `Write` over `gaia-stop.js`, replace the
finalizer with a no-op, and the protocol would silently break — the
driver would time out at 45 min with no useful sentinel. Similarly,
nothing structural prevents Claude from `git checkout main`-ing
out of the work branch mid-session, or pushing a forged sentinel
to `claude/gaia/<id>/state`, or using a built-in GitHub tool to
comment on a PR or push a commit through a non-git surface. The
saved routine prompt (§9) tells a well-behaved session not to do
these things; `PreToolUse` enforces it for sessions that drift,
ignore the prompt, or are prompt-injected from repo content
(§16.1).

**Why `*` and not a narrow matcher.** A narrow matcher (e.g.,
`Edit|Write|MultiEdit|Bash|NotebookEdit|mcp__.*`) leaves
future-tool blind spots: any new write-capable tool Claude
Code adds (built-in GitHub tools, successor of `WebFetch`,
successor of `Agent`/`Task`, etc.) skips the shim entirely
because its name is not in the matcher. A gaia session that
happens to call such a tool escapes the protections. With
`*`, every tool call surfaces to the shim, and the shim's
classification (below) is the single place that decides what
to allow — so adding a new write-capable tool becomes a
known-unknown that doctor reports, not an unmonitored
bypass.

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
2. Read `tool_name` and `tool_input` from stdin. Classify
   `tool_name` into one of four buckets:
   - **Known read-only tools** — `Read`, `Glob`, `Grep`,
     `LS`, `NotebookRead`, `TodoRead`, `TodoWrite` (read
     semantics from the protocol's perspective — it mutates a
     per-session todo list that is not protected state).
     Allowed without further inspection; exit 0.
   - **Known write-capable tools gaia understands** — `Edit`,
     `Write`, `MultiEdit`, `NotebookEdit`, `Bash`. Apply the
     protected-paths and protected-commands rules below
     (steps 3, 4). Bash is structurally decoded per step 4.
   - **Known write-capable tools in the ban / constrained
     set** — any `mcp__*` tool (banned per §2, denied in
     step 5), `WebFetch` and `WebSearch` (network read; denied
     during gaia runs because an agent that needs to reach
     the web should be doing so from the driver side, not
     the cloud session — keeps the conversation's external
     surface auditable), and built-in GitHub tools whose
     `tool_name` starts with `Github` or `GitHubPullRequest`
     or similar (denied because the saved routine prompt
     forbids opening PRs or commenting on GitHub and
     these tools would let the session do exactly that).
   - **Unknown `tool_name`** — a tool name gaia has not yet
     classified. v1 denies it with
     `permissionDecisionReason: "gaia has not yet classified
     tool_name=<name>; denying conservatively. Run
     gaia doctor --cloud to refresh, or file a gaia issue."`
     and logs the tool name so doctor's enumeration surfaces
     it for future classification. Fail-closed on unknown
     tools is the conservative choice — a new GitHub write
     tool that appeared since the current gaia version was
     published must not get a free pass through a gaia run.
     Resolve user-supplied paths against `$CLAUDE_PROJECT_DIR`
     and reject any path that escapes the project root or
     resolves through a symlink to a protected location.
3. **Protected paths.** Any tool call (`Edit`, `Write`, `MultiEdit`,
   `NotebookEdit`, or `Bash` with a write-like subcommand —
   `rm`, `mv`, `cp`, `tee`, `truncate`, `chmod`, redirect-to,
   `dd of=`, `git rm`, etc.) that targets one of the following
   paths is denied. All paths are resolved against
   `$CLAUDE_PROJECT_DIR` and rejected if a symlink resolves
   through a protected location:
   - `.claude/hooks/gaia-*.js`
   - `.claude/settings.json` — the file as a whole; the rule is
     paranoid because partial edits could disable individual
     hooks (or strip gaia's `permissions.deny` entries the
     work-branch invariant in §8.8 reconciles)
   - `.gitignore` — only edits that remove or alter the `.gaia/`
     or `.gaia-context/` lines; other `.gitignore` edits are
     allowed (Claude may legitimately add ignore rules during
     development)
   - `.gaia/**` (driver-local state; should not exist in the VM
     but is locked down for safety)
   - `.gaia-state/**` (the state-branch namespace; writes are
     the Stop hook's exclusive surface, not Claude's. The work
     branch may contain a `.gaia-state/` directory if the agent
     accidentally materializes one in the worktree; this rule
     blocks that. Stop-hook writes go through a separate
     worktree seeded on the state branch (§8.2 step 10) and are
     unaffected)
   - `.gaia-context/**` (overflow-context namespace; read-only
     from Claude's perspective. The `UserPromptSubmit` hook
     materializes files here; tool calls must not mutate them)
   - `.git/**`
4. **Protected commands.** `Bash` calls whose decoded subcommand
   matches one of these patterns are denied. The shim tokenizes
   `tool_input.command` into a shell argv using a dedicated
   parser (`shell-quote` or equivalent); the patterns match the
   first **command word** after an arbitrary prefix of
   `NAME=value` environment assignments, `env`/`env -i`
   prefixes, optional `sudo`/`doas`, and `nice`/`nohup`/`time`
   launchers. Deny patterns:
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

   In addition, the parser recursively decodes these structural
   forms and applies the same deny patterns to every nested
   command it finds:
   - `&&`, `||`, `;`, `|` command lists — every segment is
     checked independently.
   - `(...)` subshell and `{ ...; }` group — the inner list is
     checked.
   - `bash -c "..."`, `bash -lc "..."`, `sh -c "..."`,
     `zsh -c "..."`: the `-c` string is re-parsed as a nested
     command list.
   - Line continuations (`\<newline>`) — collapsed before
     tokenization so `git \<NL>push ...` is parsed as
     `git push ...`.
   - Redirection targets — `> path`, `>> path`, `tee path`,
     `dd of=path`, `cat <<EOF > path` etc. — the target `path`
     is run through the protected-paths matcher in step 3 as if
     it were a `Write` tool call. This is what blocks
     `printf '...' > .claude/settings.json` and
     `cat <<EOF > .gitignore`.
   - Command substitution (`$(...)`, backticks) is decoded and
     the inner command list is checked.

   **Explicit disclaimer.** This parser is best-effort
   enforcement, not a shell sandbox. It cannot reliably defeat
   adversarial evasion — a hostile agent that sequences
   base64-encoded command strings through `eval`, writes a
   helper script to `/tmp/x.sh` and executes it, or invokes
   Python / Node / Perl as a shell alternative can bypass these
   patterns. gaia's trust model (§16.1) treats the prompt
   boundary as the trust boundary; the `PreToolUse` rules exist
   to catch the overwhelming majority of accidents, prompt-
   injected misfires, and confused agents that approach a
   protected operation head-on, not to contain a motivated
   attacker who controls the prompt.

   **Test corpus.** The gaia test suite exercises the parser
   against this minimum corpus; each row is either "deny" or
   "warn in doctor/trust-the-model" depending on whether gaia
   can classify it cleanly:
   | Input | Expected |
   |---|---|
   | `git checkout main` | deny |
   | `git -C . checkout main` | deny |
   | `git -C /tmp/other checkout main` | deny (we over-block rather than under-block on non-repo-root git -C) |
   | `FOO=bar git push origin claude/gaia/X/state` | deny |
   | `env FOO=bar git push origin claude/gaia/X/state` | deny |
   | `sudo git push origin claude/gaia/X/state` | deny |
   | `bash -c 'git push origin claude/gaia/X/state'` | deny (nested parse) |
   | `sh -c "git checkout main"` | deny (nested parse) |
   | `(git push origin claude/gaia/X/state)` | deny (subshell decoded) |
   | `echo hi && git push -f origin main` | deny (second segment matches) |
   | `git \<NL>push origin HEAD:refs/heads/claude/gaia/X/state` | deny (continuation collapsed) |
   | `printf '...' > .claude/settings.json` | deny (redirect target matches protected path) |
   | `cat <<EOF > .gitignore<NL>.gaia<NL>EOF` | warn-in-doctor (heredoc-redirect target checked; content-level `.gaia/` detection is best-effort) |
   | `echo x > /tmp/ok.txt` | allow (redirect target outside protected set) |
   | `python3 -c "import os; os.remove('.claude/settings.json')"` | allow (out of parser scope; trusted to §16.1 trust model) |
   | `eval "$(printf 'git checkout main')"` | allow (eval contents not statically decodable; §16.1) |
   | `git status` / `git log` / `git diff` | allow |
   | `git add -A && git commit -m "..."` | allow on a non-protected path change |
5. **MCP tool calls are denied in gaia v1.** Any tool call whose
   `tool_name` matches `^mcp__` is denied in a gaia session with
   reason `"gaia v1 does not support MCP / connector tools;
   routines attached to gaia runs must be MCP-free (§5)"`. gaia
   does not model the write semantics of arbitrary MCP servers
   (a GitHub connector can mutate repo state, a filesystem
   connector can write anywhere, etc.), so the shim's bucket
   classifier (step 2) sends every `mcp__*` call through this
   deny path regardless of tool_input. This is belt-and-
   suspenders — `gaia init` / `gaia doctor` are expected to
   verify the routine has no connectors at setup time (§5,
   §8.4) — but a misconfigured routine that exposes an MCP
   tool to Claude will still have its calls rejected at
   runtime. A future release may introduce an MCP allow-list in
   gaia config once the write-surface of common connectors is
   better understood (§14).
6. **Allow-list overrides.** Claude must still be able to commit on
   the work branch, push the work branch (the Stop hook does this,
   but agents may also push intermediate progress), read any of the
   protected files, and run `git status` / `git log` / `git diff`
   freely. The shim only blocks *write* and *branch-switch*
   operations — read paths are not in the protected set.
7. **Decision output.** When denying, emit the documented
   `PreToolUse` deny envelope:
   ```json
   {
     "hookSpecificOutput": {
       "hookEventName": "PreToolUse",
       "permissionDecision": "deny",
       "permissionDecisionReason": "gaia: <path or command> is gaia-managed; the saved routine prompt explains why this branch / file is off-limits"
     }
   }
   ```
   and exit 0. Claude Code processes `PreToolUse` JSON on exit 0
   via the `hookSpecificOutput.permissionDecision` field; the tool
   call is rejected and Claude sees `permissionDecisionReason` as
   feedback. The older top-level `{"decision": "deny"}` shape is
   deprecated on current Claude Code — on the legacy envelope,
   `"block"` (not `"deny"`) maps to a denial, so a top-level
   `"deny"` silently no-ops. gaia deliberately uses the current
   shape and `gaia doctor --cloud`'s `PreToolUse` denial probe
   (§8.4) confirms an end-to-end denial is honored before any
   real run depends on it. When allowing, exit 0 with no JSON
   output.

**Coexistence with other `PreToolUse` handlers.** `PreToolUse`
accepts per-tool matchers, so multiple project-level handlers can
coexist without races as long as their matchers are disjoint.
gaia's matcher is `"*"` (every tool), so gaia runs alongside any
foreign `PreToolUse` handler regardless of matcher. The
most-restrictive rule means a denial from either handler is
honored, which is generally the desired composition for security
hooks — a foreign handler that denies a tool call gaia would have
allowed is still denied; a foreign handler that allows a tool call
gaia denied is still denied. A sibling handler that *allows*
gaia-protected writes cannot override gaia's deny in parallel
execution. `gaia init` prints foreign `PreToolUse` handlers
verbatim in the project-hooks enumeration (§5, §6.4) so the repo
owner confirms the composition is intentional; `gaia doctor`
warns if a foreign handler appears after install but does not
fail.

**Failure modes.** Claude Code's hook semantics require the shim
to exit 0 with the right JSON envelope to be considered a
blocking decision; exit 1 is non-blocking (the tool call
proceeds), exit 2 is a generic blocking error, and an unhandled
exception that crashes the Node process or prints invalid JSON
produces neither a parseable decision nor a clean deny. Given
these defaults, gaia's layered fail-closed strategy for gaia
sessions is:

1. **Binary missing or version-skewed.** §8.0 step 6 emits a
   well-formed `hookSpecificOutput.permissionDecision: "deny"`
   on exit 0. This fails closed reliably — the cited behavior is
   what the documented envelope does.
2. **Shim exception after the session has already been
   classified as gaia.** The shim wraps the decision logic in a
   try/catch; on any uncaught exception after
   `<runtime-dir>/<session_id>.json` has been located, the catch
   branch emits the same deny envelope with
   `permissionDecisionReason: "gaia: protection hook crashed;
   failing closed for safety"` and exits 0 — unless the incoming
   `tool_name` is in a narrow read-only allow-list (`Read`,
   `Glob`, `Grep`, `Agent`), in which case it exits 0 silently so
   a bug in the protection logic does not brick the session's
   reads. This gives the gaia-protected write surface a
   deny-on-crash, not just deny-on-classified-denial.
3. **Shim timeout (30 s per §8).** Claude Code's default is
   fail-open. gaia does not try to convert this to fail-closed
   — a stuck process cannot emit any JSON — and `gaia doctor`
   (§8.4) verifies the shim stays well under the timeout
   on realistic inputs so a production timeout indicates a
   genuine bug to chase.
4. **Non-gaia session.** Every branch above is conditional on
   the session being classified as gaia (§8.0 step 2). Non-gaia
   sessions exit 0 silently with no decision, regardless of what
   the shim thinks of the tool call, so a teammate running
   Claude Code locally in the repo is never constrained by
   gaia's rules.

This means the protected write surface is **deny-closed on
classified-denial and deny-on-crash**, but not literally
deny-closed on every conceivable shim failure — a timeout
specifically fails open, by Claude Code's default, and gaia
treats that as a `gaia doctor` concern rather than a runtime
guarantee. Non-protected calls (reads, unmatched tool names)
fail open by Claude Code defaults, which is intentional.

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
gaia's matcher is `"*"` (§8.5), which overlaps with any foreign
`PreToolUse` handler by construction. The most-restrictive rule
means a denial from any handler wins, which is the intended
composition for security hooks: a foreign handler that also denies
a tool call gaia would have allowed leaves the call denied
(intentional strictness); a foreign handler that *allows* a call
gaia denies does not override gaia's deny. Foreign `PreToolUse`
handlers therefore install freely alongside gaia's and add to
the protection rather than contending with it.

**v1 policy: gaia owns the three exclusive events at the
project-settings layer, per repo, shares `PreToolUse` with every
foreign handler, and enumerates all other project-hook events
without refusing them.** This is narrower than "gaia owns the
project hook surface globally." Claude Code composes hooks from
multiple scopes — user settings (`~/.claude/settings.json`),
project settings (`.claude/settings.json`), project-local
settings (not loaded in routines), managed policy settings,
plugin and skill scopes, and agent scopes. `gaia init` and
`gaia doctor` can only see and control the *project-settings*
layer. Managed policy or plugin-level handlers on any of the
four gaia events may still run alongside gaia's and interfere
with it; gaia cannot prevent that from inside the repo.

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
  matcher entry alongside any pre-existing matcher entries. Any
  pre-existing `PreToolUse` handler overlaps with gaia's matcher
  (`"*"`) by construction and is enumerated in the `gaia does
  not control these project hooks` banner rather than blocked —
  the most-restrictive rule means a denial from either handler
  is honored, which is the desired security composition.
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

**What this does not block (but does enumerate).**
`PostToolUse`, `SubagentStop`, `Notification`, `SessionStart`,
`ConfigChange`, `WorktreeCreate`, and any other event outside the
four gaia owns are permitted to coexist freely. `PreToolUse` is
shared, not exclusive (per the bullets above). `gaia init` and
`gaia doctor` enumerate all foreign project hooks — event,
matcher, and command — under a `gaia does not control these
project hooks` banner so the operator sees exactly what runs
alongside gaia. `SessionStart` gets a louder advisory because it
runs before `UserPromptSubmit` and can mutate worktree or
environment state that gaia's invariants assume is stable; the
advisory is informational, not a refusal.

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

**Invariant enforced by the driver** (§6.1 step 4): before `/fire`,
the work branch must contain the gaia-managed files at exactly the
versions the current driver expects. The driver enforces this by
synthesizing a setup commit on the work branch:

1. `claude/gaia/$RUN_ID/work` has been created locally by §6.1
   step 4's first sub-step (seeded from the — possibly fast-
   forwarded — user branch). On a fresh run, there is **no**
   `refs/remotes/origin/$GAIA_WORK_BRANCH` yet because the work
   branch has not been pushed. The reconcile step therefore adds
   a worktree against the **local** work branch, not an origin
   ref:
   ```
   git worktree add "$tmp" "$GAIA_WORK_BRANCH"
   ```
   This creates the worktree at the current tip of the local
   `$GAIA_WORK_BRANCH` (already pointing at the user-branch
   HEAD from the seed step). Earlier drafts of this spec used
   `-B "$GAIA_WORK_BRANCH" "refs/remotes/origin/$GAIA_WORK_BRANCH"`
   which is a chicken-and-egg on first push — the remote ref
   does not exist, so the worktree add would fail before the
   driver ever reached `/fire`. (The name-matched
   local/remote invariant from §8.2 step 10 is preserved: the
   local branch name equals the prospective remote target;
   the push in §6.1 step 4's last sub-step uses the matching
   refspec.)
   Inside the worktree, read each gaia-managed file from
   `refs/remotes/origin/<default-branch>` (fetched with an
   explicit refspec, which is a ref the driver has independent
   reason to trust exists — §5 and §6.4 / §8.4 all depend on
   the default branch having run `gaia init`):
   - `.claude/hooks/gaia-user-prompt-submit.js`
   - `.claude/hooks/gaia-pre-tool-use.js`
   - `.claude/hooks/gaia-stop.js`
   - `.claude/settings.json` — reconciled as the **gaia-managed
     subset** (see below): the four gaia-hook entries under
     `hooks.*` plus the `permissions.deny` entries gaia installs.
     Other keys (other event handlers, custom permissions, model
     config, etc.) are preserved non-destructively.
   - `.gitignore` entries for `.gaia/` and `.gaia-context/`
     (each appended if missing; existing matching lines left
     alone)
2. **Default-branch `protocol_version` compatibility check.**
   Compute the sha256 of each default-branch gaia-managed file
   and compare to the driver's own expected hashes (shipped with
   the installed `@your-scope/gaia` package). If the default-
   branch shim's advertised `protocol_version` differs from the
   driver's pinned `protocol_version`, the default branch is
   stale relative to the installed driver and a synthesis based
   on it would commit shims that disagree with the manifest
   the driver is about to push. Abort with exit 2 and a message
   pointing at `gaia init --verify-default-branch` (which
   re-installs the current driver's shims on the default
   branch, typically via a PR). Patch/minor-version differences
   between the default-branch shim and the driver's bundled
   copies are tolerated as long as `protocol_version` matches —
   that is §1.1's patch-release promise.
3. If any file differs from the work-branch copy, write the
   default-branch version into the work-branch tree and stage it.
4. If anything was staged, commit with message
   `gaia: ensure hook shims on work branch` and push. If the tree
   was already clean against the default branch, skip the commit.
5. **Foreign-handler scan on the work branch.** After the
   reconcile step (whether or not anything was committed), parse
   the work branch's `.claude/settings.json` and enumerate every
   handler registered under `UserPromptSubmit`, `Stop`, and
   `StopFailure`. If any handler's `command` does **not** match
   the gaia-owned shim paths (`.claude/hooks/gaia-user-prompt-
   submit.js`, `.claude/hooks/gaia-stop.js` — with or without a
   `$CLAUDE_PROJECT_DIR` prefix), abort before `/fire` with exit
   2 and a message naming the offending event and the first
   foreign handler's `command`. This closes the loophole that the
   reconcile step only *ensures* gaia's entries are present — it
   is not a rewrite. A feature branch that carried a foreign
   `Stop` handler across the gaia install on the default branch
   would otherwise fire parallel to gaia's finalizer in the
   cloud session, violating the §8.7 "gaia owns exclusive
   events" policy at runtime. For `PreToolUse`, perform the same
   scan and emit a warning (not an abort) for **any** foreign
   handler, since gaia's matcher is now `"*"` and every foreign
   handler by definition overlaps. Denial wins in parallel
   execution, which is the intended composition for security
   hooks (§8.7). The remediation for the abort case is either
   `gaia init --force-merge-hooks` (v1.x, not implemented in
   v1) or merging the default branch into the feature branch
   before running gaia so the default-branch settings version
   lands.

**The gaia-managed settings subset.** Step 1 reconciles a
defined subset of `.claude/settings.json`, not the whole file:

- `hooks.UserPromptSubmit` — gaia's entry (matched by `command`
  referencing `gaia-user-prompt-submit.js`).
- `hooks.PreToolUse` — gaia's matcher entry (matched by
  `command` referencing `gaia-pre-tool-use.js` AND the gaia
  matcher string `"*"`).
- `hooks.Stop` — gaia's entry (matched by `command` referencing
  `gaia-stop.js`).
- `hooks.StopFailure` — gaia's entry (matched by `command`
  referencing `gaia-stop.js`).
- `permissions.deny` — only the gaia-namespace deny rules
  (`/.gaia/**`, `/.gaia-state/**`, `/.gaia-context/**` per §8).
  Foreign `permissions.deny` entries are preserved on the work
  branch; gaia only reconciles the subset it owns, identified by
  exact-string match against the gaia-installed rule set.
  Branch-switch / force-push / settings-edit / `.git/**` /
  `.claude/hooks/gaia-*.js` enforcement is **not** in the
  static deny list (that would block teammates' ordinary
  Claude Code work in the repo; see §8) and is enforced
  session-gated by the `PreToolUse` shim instead. The shim's
  rules live in the global `gaia-hook-pre-tool-use` binary, so
  they apply uniformly to every gaia session regardless of
  work-branch settings state.

Reconciling the gaia-namespace `permissions.deny` rules alongside
the hook entries matters because a stale feature branch that
predates a gaia security patch (which, e.g., added
`.gaia-context/**` to the deny list) would otherwise run without
the static backstop on the work branch. The session-gated rules
in the shim continue to apply regardless of work-branch settings
state, so this is purely a defense-in-depth measure for the
gaia-namespace paths.

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

If the user message tells you to read a file under .gaia-context/,
read it first — it holds the real task content that did not fit
inline.

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
driver validates sentinel artifacts (§6.1 step 7) as the
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

The `beta_header` value is **volatile**. The `/fire` endpoint is
research preview and Anthropic may rotate this header on short
notice; `gaia doctor --cloud`'s beta-header probe (§8.4) detects
the "configured
header no longer accepted" condition and warns on the first
failed run, escalating to a fail on the second so stale headers
do not accumulate silently. Patch releases of gaia will ship
updated default header values as Anthropic rotates them; repos
that pin this value in `gaia.config.json` should expect to update
it alongside gaia upgrades.

### 10.2 Secrets

Provided via (in precedence order):

1. `gaia config set <KEY> <VALUE>` — OS keychain where available,
   `.gaia/secrets.json` (0600, gitignored) fallback.
2. Ambient environment variable.
3. `.env` at repo root (parsed by gaia on startup when present).
   gaia does not manage `.env`'s ignore status — `gaia init`
   warns if `.env` exists but is not covered by an effective
   ignore rule (§10.4), rather than appending `.env` to
   `.gitignore` itself, since project conventions around `.env*`
   vary.

v1 secrets: `GAIA_ROUTINE_URL`, `GAIA_ROUTINE_TOKEN`. (The URL is not
strictly secret but stored alongside the token for simplicity.)

**Reading secrets back.** `gaia config get <KEY>` redacts values
for any key matching `*_TOKEN` (case-insensitive) by default —
output is `<redacted: N chars>` rather than the raw token.
`gaia config get --show-secret <KEY>` prints the raw value
after a loud stderr warning. This protects the "secret-aware
storage" story from an incidental `gaia config get` in a shell
history, a terminal recording, or a pasted diagnostic. Non-
token keys print verbatim. Scripts that need to pipe a token
(e.g., a CI harness) must pass `--show-secret` explicitly, which
makes the exfiltration path auditable rather than accidental.

### 10.3 Local state

```
.gaia/                                         (gitignored; driver-local)
  history.jsonl                                append-only terminal-event log for -c / --resume
  history.lock                                 advisory lock for appends
  merge.lock                                   per-working-tree advisory lock for §6.1.1 merge-back and gaia recover --merge
  transcripts/
    <session_id>.md                            readable conversation transcripts
  secrets.json                                 fallback store (0600)
  runs/
    <run_id>/
      meta.json                                full per-run metadata (canonical)
      fire.json                                raw spawner response
      .lock                                    per-run advisory file lock (flock/LockFileEx) for meta.json rewrites across processes
.gaia-context/                                 (gitignored; VM-only staging surface)
  <run_id>/
    context.md                                 overflow context materialized by UserPromptSubmit (only if needed; never exists in driver-local checkouts under normal use)
```

The two top-level directories serve different purposes:
`.gaia/` is **driver-local** state (history, transcripts,
secrets fallback, per-run metadata, merge lock) and is
read-denied to the agent's tool calls (§8). It should never
contain anything the agent needs to read. `.gaia-context/` is a
**VM-only** staging surface for content the agent must read
but the driver owns the lifecycle of (overflow context,
future variants). Both are gitignored (§10.4); the split is
what lets `permissions.deny` keep `.gaia/` read-denied without
blocking the overflow path.

**Source of truth.** `.gaia/runs/<run_id>/meta.json` is canonical.
`history.jsonl` is a derived append-only event log — one record per
**terminal** state transition per run. If the two disagree,
`meta.json` wins; gaia offers to rebuild `history.jsonl` from the
runs directory.

`meta.json` schema:

```json
{
  "run_id": "01HXX...",
  "session_id": "<opaque session id string returned by /fire>" | null,
  "session_url": "<opaque session URL string returned by /fire>" | null,
  "status": "preparing" | "pending" | "success" | "no-merge" | "failed" | "fire-failed" | "fire-ambiguous" | "timeout" | "push-rejected" | "conflict" | "aborted",
  "user_branch": "feature/auth",
  "work_branch": "claude/gaia/01HXX.../work",
  "state_branch": "claude/gaia/01HXX.../state",
  "upstream_remote": "origin",
  "upstream_branch": "feature/auth",
  "upstream_ref": "refs/heads/feature/auth",
  "local_only_branch": false,
  "pushed_upstream": true,
  "base_sha": "abc1234",
  "merge_sha": "def5678",
  "continued_from": "<prior session_id>",
  "seeded_from_branch": "<prior run's user_branch, when --resume-seed-from-work --cross-branch was used>" | null,
  "driver_version": "1.0.0",
  "protocol_version": 1,
  "started_at": "2026-04-21T10:30:00Z",
  "finished_at": "2026-04-21T10:35:12Z",
  "prompt_first_line": "refactor the auth module...",
  "error": null
}
```

The `upstream_*` / `local_only_branch` fields mirror the §7.4
manifest (the manifest is the authoritative cloud-side copy;
`meta.json` carries the same values driver-side for §6.1.1 and
`gaia recover`). Recording them at fire time prevents a push
rejection or a recover call minutes/hours later from
re-reading a branch's upstream config, which the user might
have changed in the interim.

The `session_id` and `session_url` fields are written verbatim
from the `/fire` response (`claude_code_session_id` and
`claude_code_session_url` respectively). gaia does not parse,
reformat, or reconstruct either string — if the endpoint's URL
shape changes (e.g., `/sessions/session_...` vs `/session_...`),
gaia still records whatever the server returned. Code that
displays `session_url` treats it as an opaque string and does
not embed a hardcoded prefix.

`meta.json` is rewritten under an **exclusive interprocess
advisory lock** on `.gaia/runs/<run_id>/.lock` (via `flock` on
POSIX or `LockFileEx` on Windows when that platform lands)
during a run — a fire path transitions it through `preparing`
→ `pending` → terminal. Interprocess (not merely in-process)
matters: a crashed driver and a subsequent `gaia recover`
invocation are two separate processes racing to mutate the
same meta.json file, and an in-process lock gives no guarantee
against the cross-process race. The fire-path driver and
`gaia recover <run_id>` both acquire this per-run lock before
any meta.json rewrite; appends to `history.jsonl` use the
separate `.gaia/history.lock` (§10.3 — the per-run lock and the
history lock are orthogonal). The lock releases on process
exit (POSIX advisory lock semantics), so a crashed holder
cannot strand it. The `state_branch` field is internal
bookkeeping only and is not consumed by any code path that
runs in the cloud VM (§7.2, §16.1).

**Statuses.** `preparing` and `pending` are non-terminal;
everything else is terminal. `preparing` is the transient state
set in §6.1 step 3 *before any remote side effect* — if the
driver crashes between step 3 and step 4's push, the run is
discoverable locally but has no work/state branches on origin
yet, so `gaia recover` simply transitions it to `fire-failed`
and cleans up the local record. `pending` is set once the work
and state branches have been pushed (§6.1 step 4's final
sub-step); from that point, a crash requires the full recovery
path because a session may exist on the spawner side.
`fire-failed` represents the case where `/fire`
rejected the request with a parseable error envelope before any
session was created (§6.1 step 6): `session_id` and `session_url`
are `null`, the work and state branches were deleted from origin
(no recovery is possible without a session), and `gaia history`
lists the run with the recorded error so it does not silently
disappear. `fire-ambiguous` is the distinct case where the
driver could not tell whether a session was created — request
body may have been sent, no response was parsed (§6.1 step 6):
`session_id` / `session_url` are `null` at the point of the
transition, the work and state branches are **preserved** on
origin so a session that does exist can still push its sentinel,
and `gaia recover` polls the state branch for a late sentinel
before deleting anything. `pushed_upstream` is `false` for
local-only user branches that skipped the upstream push step
(§6.1.1 step 6).

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
event are: `success`, `no-merge`, `failed`, `fire-failed`,
`fire-ambiguous`, `timeout`, `conflict`, `push-rejected`,
`aborted`. A stuck `preparing` or `pending` run therefore has no
history entry until recovery finalizes it — that is deliberate so
that crashed/abandoned runs never look like successful
continuation targets to `-c`, and `gaia -c` reads meta.json (not
just history) to detect in-flight runs explicitly (§6.2).

If a run transitions again (e.g., `gaia recover` promotes a
`push-rejected` run whose push eventually succeeded, lands an
`aborted` run via `--merge`, or resolves a `fire-ambiguous` run
via a late sentinel), the driver appends a **new** event with the
new terminal status. Readers treat the **last record for each
run_id** as authoritative. `gaia history` surfaces all events
and `gaia -c` resolves continuations by reading the latest
terminal record per user_branch and requiring its status to be
`"success"` (§6.2); an older successful run whose branch has
been stepped on by a newer non-success run is not a valid `-c`
target.

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

`gaia init` appends two rules, each added only if not already
present (no duplicate lines on re-run):

```
.gaia/
.gaia-context/
```

`.gaia/` covers the driver-local state layout (§10.3). `.gaia-
context/` covers the VM-side overflow staging directory
`gaia-hook-user-prompt-submit` materializes the overflow context
into (§7.2, §8.1). Both rules are kept distinct because the
driver-local `.gaia/` is read-denied at the permission layer
(§8) — a path Claude must read during the run cannot live there
— so the overflow file needs its own namespace, and that
namespace needs its own ignore rule so the Stop hook's
`git add -A` never commits it to the work branch.

`.gaia-state/` lives on orphan state branches exclusively and
never touches reviewed branches. It is also covered by the
§8.5 protected-paths set on the work branch as belt-and-
suspenders: the work branch should never contain a
`.gaia-state/` directory, but a confused agent that materializes
one in its worktree cannot write through it and the Stop hook's
sweep explicitly excludes it from staging (§8.2 step 5).

**`.env` handling.** gaia does not append `.env` to `.gitignore`
automatically — `.env` conventions vary by project (some repos
intentionally commit an `.env.example`, some ignore `.env*`
wildcards, some use dotenv-free configuration). `gaia init`
instead *verifies* that if `.env` is present at the repo root,
a matching ignore rule is already in effect (via project
`.gitignore`, `~/.gitignore`, or `.git/info/exclude`); if not,
it warns and points the user at the remediation. §10.2's
precedence list reads `.env` when present — treat that as
"gaia picks up `.env` if you have one" rather than "gaia
guarantees it is gitignored."

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
- HTTP error mapping. HTTP-error-envelope cases fall into two
  buckets. Responses that the server contract makes
  side-effect-free — 400/401/403/404/429, all of which
  semantically mean "request rejected before work started" —
  surface as `meta.status = "fire-failed"` per §10.3. 5xx
  responses are treated as **ambiguous** by default: Anthropic
  does not document that `/fire` is side-effect-free for 5xx,
  and a 500/503 can plausibly be returned after the server has
  already begun creating a session. Since `/fire` has no
  idempotency key (confirmed in Anthropic's current docs),
  retrying the same payload with the same `GAIA_RUN_ID` against
  a possibly-live session would create two sessions racing on
  the same work/state branches. Ambiguous network failures that
  could not be classified surface the same way. Definite-not-
  sent network failures are safe to retry and surface as
  `"fire-failed"` only after retries exhaust:
  | Code | `error.type` | gaia behavior | `meta.status` on exhaustion |
  |---|---|---|---|
  | 400 | `invalid_request_error` (text too large, paused, malformed) | Non-retryable. Exit 12. | `fire-failed` |
  | 401 | `authentication_error` | Non-retryable. Exit 10. | `fire-failed` |
  | 403 | `permission_error` (entitlement) | Non-retryable. Exit 10. | `fire-failed` |
  | 404 | `not_found_error` (routine id wrong / deleted) | Non-retryable. Exit 12. | `fire-failed` |
  | 429 | `rate_limit_error` | Honor `Retry-After`. Safe to retry with same `GAIA_RUN_ID` — a rate-limited request means the server rejected it before admission. After retries exhausted, exit 11. | `fire-failed` |
  | 500 | `api_error` | **Ambiguous. No retry with the same `GAIA_RUN_ID`.** Exit 16. Work/state branches preserved for `gaia recover` to poll for a late sentinel. If no sentinel arrives, recovery offers to classify the run as `fire-failed` or retry with a fresh `GAIA_RUN_ID`. | `fire-ambiguous` |
  | 503 | `overloaded_error` | Same as 500. Exit 16. | `fire-ambiguous` |
  | Definite-not-sent network failure (DNS failure, TCP connect refusal, TLS handshake failure before any request bytes are written) | n/a | Exponential backoff up to 3 attempts. Then exit 5. | `fire-failed` |
  | Ambiguous network failure (TLS error after request body bytes may have been sent, connection reset mid-body or mid-response, read-response timeout, 2xx body that fails to parse `claude_code_session_id`) | n/a | **No retry with the same `GAIA_RUN_ID`.** Exit 16. Work/state branches preserved for `gaia recover`. | `fire-ambiguous` |
- The classifier for "definite-not-sent" vs "ambiguous" lives in
  the HTTP client: any error raised before the first byte of the
  request body enters the socket write buffer is definite-not-
  sent (DNS, TCP, TLS handshake); errors after that point are
  ambiguous unless the client received a complete, parseable
  4xx status envelope from the documented side-effect-free set
  (400/401/403/404/429). 5xx envelopes are ambiguous even when
  parseable, because the response body does not prove no
  session was created. gaia errs on the side of ambiguous — a
  failure that cannot be proven definite-not-sent or a
  documented side-effect-free rejection is treated as
  ambiguous, because the cost of a false "fire-failed"
  classification is losing a real running session.
- **If Anthropic confirms 5xx is side-effect-free.** The 5xx
  classification above is conservative. If Anthropic publishes
  a written guarantee that `/fire` 500/503 responses are
  side-effect-free (no session created) — or, equivalently,
  adds an idempotency-key header — a patch release can move
  500/503 back into the retryable `fire-failed` bucket without
  changing the CLI contract. gaia tracks the conservative
  behavior as the v1 default and will relax it when the
  server-side invariant is documented, not assumed.
- `maxPayloadChars: 65536`, `defaultTimeout: "45m"`.
- The endpoint returns immediately after session creation — it does
  not stream or wait. gaia's sentinel / polling design is the intended
  completion transport.

## 12. Failure modes

| Failure | Detection | Policy |
|---|---|---|
| `-p` missing | Arg parsing | Exit 2 with usage. |
| Spawner returns parseable side-effect-free HTTP error | `fire()` rejects with a typed 400/401/403/404/429 envelope | Retry per §11.2 where applicable (429 only). On exhaustion, transition `meta.status = "fire-failed"` (`session_id: null`), append a `"fire-failed"` event to `history.jsonl`, delete the freshly-pushed work / state branches from origin (the documented server contract for these codes is no side effects), exit 10/11/12 per §11.2. |
| Spawner returns 5xx envelope (500 `api_error` / 503 `overloaded_error`) | `fire()` rejects with a typed 5xx envelope | **Treated as ambiguous** per §11.2. No retry with the same `GAIA_RUN_ID`. Transition `meta.status = "fire-ambiguous"`, preserve work / state branches on origin, exit 16. `gaia recover` polls for a late sentinel; see the ambiguous row below for the resolution flow. |
| Definite-not-sent network failure | DNS / TCP / TLS error before any request-body bytes written | Retry with same `GAIA_RUN_ID` per §11.2. On exhaustion, `meta.status = "fire-failed"`, branches deleted, exit 5. |
| Ambiguous `/fire` failure — session may or may not have been created | Any failure after request-body bytes may have reached the socket; 2xx body that fails to parse `claude_code_session_id` | **Never retry with the same `GAIA_RUN_ID`.** Transition `meta.status = "fire-ambiguous"` (`session_id: null`), append a `"fire-ambiguous"` event, **preserve** work/state branches on origin, exit 16. `gaia recover <run_id>` polls the state branch for a late sentinel (§6.4); if one arrives, the run promotes to `"success"` / `"failed"` / `"aborted"` as if the driver had received the sentinel inline. |
| `GAIA_FAILED:` sentinel received | Poll loop sees failure sentinel | Exit 13. Error body printed. `meta.json.status = "failed"`. Terminal `"failed"` event appended to `history.jsonl`. `gaia recover` offered. |
| `GAIA_ABORTED:` sentinel received (experimental mode) | Poll loop sees abort sentinel | Exit 0. `meta.json.status = "aborted"`. Terminal `"aborted"` event appended. No merge performed; work branch preserved on origin. `gaia recover <id> --merge` can land the changes later. |
| `GAIA_DONE:` sentinel missing expected artifacts | `GAIA_DONE:` commit lacks `<session_id>.md` or `<session_id>.transcript.md` | Driver treats as suspect (potential forgery per §16.1), logs a warning, continues polling the remaining timeout. If nothing valid arrives, falls through to the timeout path (exit 3). |
| No sentinel by timeout | Poll loop deadline | Exit 3. Work + state branches preserved. Session URL printed. `meta.json.status = "timeout"`. Recoverable via `gaia recover`. |
| Merge conflict | `git merge` fails | Abort merge. Exit 4. Work branch preserved; user branch untouched. `meta.json.status = "conflict"`. Recoverable via `gaia recover`. |
| Push rejected after local merge | `git push $UPSTREAM_REMOTE $USER_BRANCH:$UPSTREAM_BRANCH` fails | Exit 15 (push to the recorded upstream branch — not necessarily `origin` — was rejected). Local user branch has the merge commit. `meta.json.status = "push-rejected"`. Printed instructions: investigate upstream advance, then `git push` when ready — or run `gaia recover <id>` which retries the push (using the recorded `meta.upstream_*` fields, not a hardcoded `origin`) without re-merging (§6.4). |
| User branch diverged non-FF at merge time | `git merge-base --is-ancestor` fails pre-merge | Exit 2 with recovery message. |
| User branch diverged from upstream at fire time | §6.1 step 2 classifies upstream as divergent (both sides have unique commits) | Exit 2 before `/fire` with a message pointing at `git pull --rebase` (or the user's preferred reconciliation). Catches divergence before burning a routine run on a merge-back that was guaranteed to fail. |
| Stop-hook work-branch push failed but sentinel landed | `.gaia-state/<id>.rescue.bundle` (or `.rescue.patch`) present in the sentinel commit | `gaia recover <run_id>` verifies the origin work branch, then reconstructs missing commits from the bundle (preferred) or patch; merge-back continues per §6.1.1. If both pushes (work-branch and state-branch sentinel) failed, driver exits 3 and recovery surfaces the session URL for manual inspection. |
| Dirty working tree at start | `git status --porcelain` non-empty | Exit 2. |
| Detached HEAD at start | No symbolic ref | Exit 2. |
| `UserPromptSubmit` hook fails | Hook returns `{"decision": "block"}` or exits 2 | Session blocked by Claude Code. Driver hits no-sentinel timeout; session URL surfaced. |
| `gaia-hook-*` binary missing or version-skewed in gaia session (shim fail-closed) | Global binary not on `$PATH`, or its protocol version disagrees with the committed shim's (§8.0) | `UserPromptSubmit` shim blocks with a clear reason. `PreToolUse` shim denies every tool call with the same reason. Stop/StopFailure shim attempts a best-effort `GAIA_FAILED:` push with `{error: "gaia-hook-binary-missing-or-version-skew"}` then exits 1. Driver exits 13 (fast) or 3 (timeout fallback). |
| `PreToolUse` denial (gaia-protected file or command) | gaia shim returns `hookSpecificOutput.permissionDecision: "deny"` + exit 0 | Tool call rejected; Claude sees `permissionDecisionReason` as feedback and continues. No driver-visible failure; the session terminates normally and finalization runs. |
| Work branch missing hook shim files | Default-branch hooks present but user branch stale (§8.8) | Driver synthesizes `gaia: ensure hook shims on work branch` commit before firing. If the default branch itself lacks the files (gaia init never run), driver exits 2 with a clear error. |
| `Stop` hook fails before sentinel push | No sentinel on state branch | Hook exits 1 (never 2; §8.2). Session terminates; driver times out at 45 min. Any pre-failure pushed commits on the work branch are preserved. `gaia recover` surfaces the session URL. |
| `StopFailure` hook fails | No `GAIA_FAILED:` commit | Side effects best-effort; output and exit code ignored by Claude Code (§8.3). Driver times out (exit 3) instead of exiting 13. Same recovery path. |
| Transcript parse fails (final message) | `last_assistant_message` empty + JSONL shape unknown | Fallback chain: `last_assistant_message` → reverse-scan JSONL → raw last line. Run still succeeds; stdout is whatever the fallback produced. |
| Transcript parse fails (readable transcript) | Same | `.gaia-state/<session_id>.transcript.md` contains the raw JSONL; continuation still works (Claude can read it) with higher token cost. |
| Duplicate sentinel | Two `GAIA_DONE:` commits for the same session_id | Earliest wins. Warning logged. |
| `-c` cannot continue: most recent terminal event on branch is non-success, or no terminal events at all | Per §6.2 — the latest terminal entry in `history.jsonl` for this `user_branch` is `failed` / `timeout` / `conflict` / `push-rejected` / `no-merge` / `aborted` / `fire-failed` / `fire-ambiguous`, or the branch has no terminal entries | Exit 2 naming the offending run (status, run_id, session_id) and suggesting `gaia recover` or `--resume <session_id>` for an older successful run. |
| `--resume <session_id>` unknown | No matching entry | Exit 2. |
| `--resume` of a run whose code did not merge | Prior `meta.status` ∈ {no-merge, conflict, timeout, aborted, push-rejected, failed} and current HEAD does not contain the prior work-branch tip | Exit 2 unless `--resume-seed-from-work` is passed (§6.2). Message points at `gaia recover <run_id>` or the flag as the two supported paths. For `fire-failed` / `fire-ambiguous`, `--resume` exits 2 unconditionally — fire-failed has no transcript, fire-ambiguous needs `gaia recover` first. |
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
- Any non-zero exit in {3, 4, 13, 15, 16} — `gaia recover` needs
  the branches to exist. Exit 16 (`fire-ambiguous`) specifically
  requires the preserved state branch so recovery can poll for a
  late sentinel whose session_id was never known to the driver.

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

- **Merge-back in the same physical checkout.** Two parallel
  runs whose drivers share a working tree cannot perform
  §6.1.1 concurrently — both processes want to mutate HEAD
  and the index. The driver serializes this with an advisory
  lock on `.gaia/merge.lock` (§6.1.1), held from step 1
  through step 7 (and released in a `finally`). `gaia recover
  --merge` acquires the same lock. Concurrent runs in
  *different* clones do not share this lock and never block
  each other — the lock is per-working-tree, which is the
  level at which git operations conflict. If the lock cannot
  be acquired within 30 s, the second process exits 2 naming
  the holder's run_id.
- **Merge-back conflicts at run-end.** Two parallel runs that
  both modify the same files on the user branch will conflict
  when the second merges — even with the local lock, the
  merge itself can fail. Handled per §12 exit 4: abort the
  merge, preserve the work branch, surface the run id, let
  the user resolve. This is the same behavior as any
  concurrent git work and is not specific to gaia.
- **Spawner quota.** The Claude Code routines daily cap is
  per-account. N parallel invocations draw N times the budget. 429
  from the spawner is terminal for the current window per §11.2.
- **Same-terminal `--experimental-mode`.** Two `--experimental-mode`
  invocations from the same terminal conflict on stdin — only one
  process can read input. Run them in separate terminals, or
  serialize. Stable (non-experimental) runs have no stdin dependency
  and can share a terminal via backgrounding without issue.

**What happens when `-c` is invoked during in-flight runs:**

`-c` resolves the latest run on the current `user_branch` by
scanning `.gaia/runs/*/meta.json` directly (not just
`history.jsonl`) and ordering by `run_id` ULID timestamp. If
the newest run is `preparing` or `pending`, `-c` refuses with
exit 2 and points at `gaia recover <run_id>` or `--resume
<older_session_id>` (§6.2). Reading meta.json is the load-
bearing change over older designs that read history only:
history is written only on terminal transitions, so an
in-flight run is invisible to a history-only scan, which would
let `-c` silently pick up an older successful run and start a
second racing session while the first is still processing.
Failed, timed-out, aborted, no-merge, fire-failed,
or fire-ambiguous runs are visible via `gaia history` and
recoverable via `gaia recover`, but block `-c`: if one of those
is the latest terminal record on the branch, `-c` exits 2
(§12) naming the offending run. Deliberately `-c` does **not**
skip past a newer non-success run to pick up an older success
— an older successful run can still be resumed via
`--resume <session_id>`, but silently continuing "the last
*succeeding* thing" when a newer run failed or is unresolved
is a bug hiding in convenience.

Same-checkout merge-back is serialized via `.gaia/merge.lock`
(above, §6.1.1). An origin-side advisory lock on the user
branch — preventing concurrent merges at the repo level rather
than per-checkout — remains future work (§14) for callers who
want belt-and-suspenders serialization across unrelated clones.

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
- **Origin-side branch lock.** v1 serializes same-checkout
  merges via `.gaia/merge.lock` (§6.1.1, §13). A future mode
  could add an origin-side advisory lock on the user branch
  for callers who want to serialize merges across unrelated
  clones (e.g., CI jobs on separate machines). Deferred
  because the use case is rare in practice and cross-machine
  locking adds a server-side dependency.
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
- **Session metadata artifact.** Currently the Stop hook pushes
  only a final-message and transcript artifact alongside each
  sentinel. Adding a `.gaia-state/<session_id>.meta.json`
  artifact carrying `{session_id, session_url, issued_at}`
  would let `gaia recover` on a `fire-ambiguous` run populate
  `meta.session_url` from a late sentinel, which today stays
  `null` on that path (§6.4). Cheap but not needed for
  correctness — the URL is informational for debugging, not
  load-bearing on recovery.
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
  Linux/macOS teammate support. Repos that want to minimize
  dependencies on the cloud VM or on teammates' machines could opt
  into POSIX `.sh` shims with equivalent behavior. Tracked as a
  v1.x opt-in; not the default.
- **Windows support.** v1 does not certify Windows (§5, §8.0).
  The shim command string uses POSIX shell expansion that native
  PowerShell does not handle without a `"shell": "powershell"`
  hint, and `gaia doctor` has no Windows coverage. v1.x work:
  add a Windows-variant `command` (or the shell hint) to the
  settings block that `gaia init` emits, extend `gaia doctor` to
  exercise end-to-end on Windows, and update the prerequisite
  banner accordingly.
- **Cloud-upgrade UX smoothing.** `gaia cloud-upgrade` (§6.4)
  bumps the routine's setup-script pin in v1 and relies on
  Claude Code's ~7-day setup-script cache to roll over on its
  own. A future release could probe the cache freshness from
  the driver side (e.g., by firing a no-op doctor session and
  inspecting the returned binary version) and surface a
  `your VM is still on v1.2.3, the script has been bumped to
  v1.2.4, the cache will roll over by <date>` banner. Purely
  informational; v1's "exact pin, explicit upgrade, wait for
  cache roll-over" model already works.
- **Ambiguous-fire auto-recover.** v1's `gaia recover
  <run_id>` polls the state branch for a late sentinel when the
  original `/fire` attempt returned an ambiguous outcome (§6.1
  step 5, §6.4). A future `gaia` could promote this polling into
  the main driver loop — the original process waits a bounded
  interval after an ambiguous fire before exiting 16, so many
  common network blips collapse into a normal run instead of a
  recover-required pause. Not in v1 because the UX tradeoffs
  (how long to wait before giving up, whether to exit 0 on late
  success, whether the caller expects their shell prompt back
  quickly) need empirical tuning.
- **Protection for non-Bash evasion paths.** The `PreToolUse`
  Bash parser is best-effort (§8.5); adversarial shell
  constructs (`eval`, base64 decoding, Python/Node shell-outs)
  bypass it. A stricter sandbox (allow-list Bash instead of
  deny-list, or a non-Bash tool-invocation policy) would close
  the gap at the cost of breaking legitimate agent flexibility.
  Deferred until the observed failure-mode rate justifies the
  ergonomic cost.
- **`ConfigChange` hook for tamper detection.** Current Claude
  Code documents a `ConfigChange` hook event that can block
  configuration changes from taking effect. Since
  `.claude/settings.json` tampering is one of gaia's central
  risks (the §8.5 PreToolUse rule blocks writes from inside a
  gaia session, but a session that escapes the parser via
  `eval` / Python could still mutate it), gaia v1.x may adopt
  `ConfigChange` to reject changes to gaia-owned settings during
  a gaia session. Not in v1 because it expands the hook-
  coexistence footprint (§8.7) and requires another tier of
  doctor verification.
- **OS-level sandboxing for gaia-owned paths.** Claude Code
  documents an OS-level sandbox surface for Bash subprocesses
  that complements `permissions.deny`. Because gaia's
  `PreToolUse` Bash parser is best-effort and bypassable by
  `eval` / language interpreters / background processes (§8.5,
  §16.1), a future hardening mode could ask `gaia init` to opt
  the repo into a sandbox config that denies writes to
  `./.git`, `./.claude/hooks`, `./.gaia`, `./.gaia-state`, and
  `./.gaia-context` at the OS layer. Conceptually:
  ```json
  {
    "sandbox": {
      "enabled": true,
      "failIfUnavailable": true,
      "filesystem": {
        "denyWrite": [
          "./.git",
          "./.claude/hooks",
          "./.gaia",
          "./.gaia-state",
          "./.gaia-context"
        ]
      }
    }
  }
  ```
  The exact syntax has to be validated against the Claude Code
  sandbox API at implementation time; the v1.x goal is to take
  `PreToolUse` from "catches the overwhelming majority of
  accidents" to "catches accidents and most adversarial
  evasion." Until that lands, §16.1's trust model applies.
- **MCP / connector allow-list.** v1 bans all `mcp__*` tool
  calls in gaia sessions (§2, §5, §8.5). Supporting them in
  v1.x requires modeling each connector's write surface —
  which paths / repos / APIs it can mutate — and composing
  that model with gaia's protected-path set. The first
  connectors to graduate are likely the read-only ones
  (Linear, Sentry, etc.); write-capable ones (GitHub,
  filesystem) require more careful policy design. A gaia
  config entry (`spawner_config.allowed_mcp: ["linear",
  "sentry"]`) is the planned surface.
- **Subagent gating and dedicated tooling.** v1 relies on the
  observation that subagents (launched via `Agent` / `Task`)
  carry the parent session's `session_id` in their hook
  stdin, so `PreToolUse` protections apply uniformly.
  `gaia doctor --cloud` check 21 verifies this on the live
  transport. If a future Claude Code version changes that
  correlation (e.g., subagents get their own session_id and
  need their own tmp-file), gaia will need a dedicated
  `SubagentStop` / subagent-scoped hook path to preserve the
  same guarantees. Likely a v1.x adjustment rather than a
  breaking change — the mechanism extends the existing
  hook-classification pattern.

## 15. Lab-quality mode: `--experimental-mode`

**Status: lab-quality. Not part of the v1 stable contract.** Ships in
v1.x behind `--experimental-mode`. Prints a warning on start. Depends
on current `Stop`-hook block-decision semantics, which are sparsely
documented and may change between Claude Code versions. When possible,
finalizes using the same terminal sentinels as the stable
fire-and-forget path (§15.3 covers each case); on hook crashes and
VM wall-clock timeouts the run leaves no sentinel and recovery
goes through the standard `gaia recover <run_id>` flow rather than
finalizing automatically. Intended to graduate once the constraints
in §15.4 are characterized and `gaia doctor` can certify them.

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
   temporary-worktree** flow as §8.2 step 10
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
   On timeout (no `GAIA_INPUT` for 9 minutes), the hook treats the
   absence of input as an *abort*, not a continuation: it runs the
   final sweep so no work is silently lost from the VM, writes a
   final-message artifact noting "user did not reply within 9
   minutes," writes the transcript, and pushes `GAIA_ABORTED:
   <session_id>` (§15.6). The driver exits 0 with `status:
   "aborted"` and **does not merge** — see §15.3 row 1. Auto-merge
   on input timeout would silently land partial work after a
   clarification went unanswered, which is the wrong default for
   an interactive surface.
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
| Hook timeout (9 min without `GAIA_INPUT`) | Hook runs the final sweep so no work is silently lost from the VM, then writes `.gaia-state/<session_id>.md` (last assistant message + a "user did not reply within 9 minutes" marker) and `.gaia-state/<session_id>.transcript.md`, and pushes `GAIA_ABORTED: <session_id>`. | Driver reads the abort sentinel and exits 0 with `status: "aborted"`. **Does not merge.** Work branch preserved on origin. The user can complete the session two ways: `gaia recover <run_id> --merge` to land the partial work, or `gaia -c -p "..."` (transcript replay) to continue from a fresh session. The previous draft auto-merged on input timeout, which silently landed half-done work after a clarification went unanswered — never the right default for an interactive surface. |
| Hook exits non-zero (git error, JSON schema drift) | Session terminates abnormally. No sentinel pushed. | Driver times out. Exits 3. Recovery via `gaia recover <run_id>`. |
| `StopFailure` fires instead of `Stop` (API error) | `GAIA_FAILED:` sentinel pushed; no pending turn. | Driver exits 13. `gaia recover` surfaces the run; `gaia -c` replay possible with the transcript captured pre-failure. |
| VM wall-clock timeout (undocumented Anthropic cap) | Session dies server-side. | Driver detects state-branch inactivity, exits 3. Recovery via `gaia recover`. |
| Driver killed (user ctrl-C or SIGTERM) | Driver leaves `meta.json.status = "pending"` (no terminal event written to `history.jsonl`). The hook eventually hits its 9-minute input timeout with no `GAIA_INPUT`, runs the final sweep so no work is silently lost, and pushes `GAIA_ABORTED:` (same path as the hook-timeout row above — the driver's absence and an explicit `:abort` both look identical to the hook: no input arrived within the window). | User runs `gaia recover <run_id>` when ready. Recover reads the `GAIA_ABORTED:` sentinel, sets `meta.status = "aborted"`, and appends the terminal event to history at promotion time. `gaia recover <run_id> --merge` opts in to a merge-back if the user decides to keep the partial work; otherwise recover leaves the run aborted and the work branch preserved on origin. |
| Explicit user abort (`:abort` at the reply prompt) | Driver pushes `__GAIA_ABORT__`. Hook finalizes with `GAIA_ABORTED:` in the same invocation — a distinct sentinel from `GAIA_DONE:` (§15.6). | Driver exits 0 with `status: "aborted"`, does **not** merge. Work branch preserved on origin. `gaia recover <id> --merge` opts in to a merge-back later. |

In every fallback case:

- **No work is lost for clean exits.** The Stop hook's unconditional
  push (§8.2 step 7) syncs work-branch state to origin in every clean-
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
- **No protocol divergence.** Every fallback ends with one of
  the three terminal sentinels — `GAIA_DONE:` (clean exit),
  `GAIA_FAILED:` (Claude Code / Anthropic API error), or
  `GAIA_ABORTED:` (user `:abort`, driver-death-after-pending, or
  9-minute input timeout) — plus the standard artifacts.
  Downstream tooling (merge, history append, stdout, recovery)
  branches on which sentinel landed: `GAIA_DONE` merges by
  default, `GAIA_FAILED` prints the error body, `GAIA_ABORTED`
  exits 0 without merging and preserves the work branch for
  `gaia recover <id> --merge`.

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
  9-minute hook blocking-wait falling back to `GAIA_ABORTED:`
  (§15.2 step 5, §15.3 row 1) — the user later runs
  `gaia recover <run_id> --merge` to land the partial work or
  treats the run as abandoned.
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
- **Subagents unchanged.** The subagent-launching tool (the
  `Agent` tool, previously `Task`) fires `SubagentStop`, not
  `Stop`, so subagent execution is unaffected by
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
poll loop (§6.1 step 7) treats any of them as terminal. The final
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
  in §6.1 step 7 catches trivially-empty forgeries but a
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
  and not in the `gaia.*` git config surface (§8.1 step 5). This
  is **accidental-surface reduction, not access control**: the
  state-branch name is fully derivable from `GAIA_RUN_ID` plus
  the documented `claude/gaia/<id>/state` convention, and the
  agent can freely `git fetch` and `git show` it (only writes are
  blocked by §8.5). The benefit is narrow: it removes one trivial
  path where an off-task agent scanning its own prompt might
  stumble onto the state branch.
- The `PreToolUse` hook (§8.5) runs on **every** tool call
  (matcher `"*"`) and blocks `Edit` / `Write` / `MultiEdit` /
  `NotebookEdit` / `Bash`-write operations on gaia-managed files
  (`.claude/hooks/gaia-*.js`, `.claude/settings.json`, the
  `.gaia/` ignore rule, `.git/**`), blocks `Bash` invocations of
  branch-switch / hard-reset / state-branch-push commands, denies
  every `mcp__*` call, denies `WebFetch` / `WebSearch`, denies
  built-in GitHub tools that could open PRs or post comments
  (matching the saved routine prompt's "do not open PRs" rule),
  and denies any unknown tool name conservatively. This is
  structural enforcement of the prompt instructions above — a
  session that ignores the prompt or is prompt-injected from
  repo content still cannot mutate the protocol shim, push a
  forged sentinel via `git push`, comment on a PR via a built-
  in GitHub tool, or reach the web via `WebFetch`. It does not
  stop file-system writes from outside Claude's tool surface
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

A complementary local hardening — opting the repo into Claude
Code's OS-level sandbox to deny writes against gaia-owned paths
at the OS layer rather than only at the `PreToolUse` parser
layer — is also tracked in §14. That would close the
"adversarial bypass via `eval` / Python / background process"
gap that §8.5's parser explicitly does not handle, taking the
defense-in-depth story from "catches accidents" to "catches
accidents and most adversarial evasion." Not in v1 because the
exact sandbox syntax requires validation against the current
Claude Code build, but it is the obvious next hardening step.

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
