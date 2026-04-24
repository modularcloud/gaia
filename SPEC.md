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

**R0 release gate — package name unfinalized.** The spec is
**design-complete but not build-complete**: the string
`@your-scope/gaia` that appears throughout this document is a
placeholder, not a scope gaia controls. No publish, install
documentation, VM setup script, fixture, or snapshot that embeds
that literal is permitted to ship under a v1 tag. Resolving the
package name is the **R0 release gate** — v1 preview framing does
not apply until a real scope is chosen and every placeholder is
switched in the same release. See §3 for the full gate description
and the CI check that enforces it; no "v1 preview" wording applies
to any build that still contains the placeholder.

**v1 is preview (once R0 closes).** The "stable" language used
throughout this spec (e.g., "stable surface" in §2, "v1 stable
contract" in §15) refers to the **CLI shape** — `-p` / `-c` /
`--resume` / `--no-merge` /
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
  driver, `gaia cloud-upgrade` (§6.4) emits the updated
  setup-script text with the new pinned version for the user
  to paste into the Claude Code environment UI; that paste
  invalidates the cache and pulls the new binary on the next
  fire. In-place editing of the setup script would require a
  public routine/environment configuration API, which is not
  documented today — v1 therefore ships `gaia cloud-upgrade`
  as a paste-assist, not an automated editor. A teammate VM that is still on the older pin
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
  not accept matchers at all; `StopFailure` matches on the stdin
  `error` field (the documented matcher field — earlier spec drafts
  called this `error_type`, which is an erratum), and carries no
  per-session or per-run identity matcher gaia could use to scope
  itself. In all three cases, every registered handler fires in parallel
  on the events gaia cares about, and when multiple handlers return
  decision objects the most restrictive wins. gaia cannot safely
  share any of those events with another hook that might race its
  checkout, block its run, or mutate the conversation. `gaia init`
  refuses to install into a repo that already defines any of the
  three (§5, §6.4, §8.7). Composing with pre-existing hooks is
  deferred to v1.x (§14) and is a non-breaking addition when it
  lands. The fourth event gaia owns — `PreToolUse` — *does* accept
  per-tool matchers, but gaia uses `"*"` (every tool, classified
  inside the shim; §8.5), so every foreign `PreToolUse` handler
  overlaps gaia's matcher by construction. `gaia init` permits
  foreign `PreToolUse` handlers anyway: the most-restrictive rule
  means a denial from any handler wins in parallel execution, so
  foreign handlers add to the deny surface rather than contending
  with it. This is the desired composition for security hooks
  (§8.7), and it is why the spec describes `PreToolUse` as
  *shared* rather than *exclusive* even though gaia's matcher
  overlaps with every foreign matcher.
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
- **No project-scoped MCP, plugin, skill, command, or
  subagent configuration in the repo.** Cloud sessions clone
  the repo and load project-level `.mcp.json` MCP servers and
  project-declared plugins (hooks, permissions, MCP bindings)
  from `.claude/settings.json` alongside gaia's hooks. **In
  addition, Claude Code loads project-scoped skills from
  `.claude/skills/` (including auto-discovered nested
  directories under `.claude/skills/**`), project-scoped
  slash-commands from `.claude/commands/**` (current Claude
  Code docs describe command files as behaving like skills
  with the same YAML frontmatter surface), and project-scoped
  subagents from `.claude/agents/`**. A committed skill,
  command, or subagent file is protocol-relevant even when its
  frontmatter does not declare hooks on gaia's events: the
  same frontmatter surface grants `allowed-tools` (which can
  add Bash, Write, or future write-capable tools to the
  session's permission set without gaia seeing them), sets
  `context: fork` (which spawns a forked conversation under
  policies gaia has not modeled), selects a subagent via
  `agent:` (which re-enters the §8.5 Agent-denial path through
  a surface the committed file declares, not a runtime tool
  call), picks an alternate `shell`, defines hooks on non-
  gaia events (which still run alongside gaia's handlers),
  and runs inline shell commands via leading-`!` lines or
  ``!...`` fenced blocks that execute **before** Claude sees
  the rendered skill body. That last path is the one gaia
  cannot catch at runtime at all — `PreToolUse` fires on
  `Bash` tool calls the model issues, not on skill
  preprocessor side-effects that run before the model's turn
  begins. A scan that filters only on `hooks:` frontmatter
  leaves every other load-bearing field open, so v1 takes
  the conservative cut:
  - **A committed `.mcp.json`** at the repo root (or anywhere
    the cloud session will load it).
  - **Any committed `.claude/skills/**/*`,
    `.claude/commands/**/*`, or `.claude/agents/**/*` file**
    — the recursive `**` is deliberate: current Claude Code
    auto-discovers nested subdirectories under each of the
    three roots, and a content-bearing file one level deeper
    than `.claude/skills/<name>/SKILL.md` (e.g.,
    `.claude/skills/<group>/<name>/SKILL.md`) is loaded the
    same way. v1 refuses on presence alone, regardless of
    whether the file declares hooks, tools, context, agent,
    shell, or nothing at all. This is intentionally broader
    than a structural scan: the surface is still under active
    Claude Code evolution and a deny-on-frontmatter-field
    scanner has to be maintained against every new field
    Claude Code introduces. A presence check fails safely on
    any new field name because every such file is blocked
    regardless of contents. gaia v1 also sets
    `disableSkillShellExecution: true` in the committed
    `.claude/settings.json` (§8, §8.8) as a defense-in-depth
    backstop against user- or plugin-scoped skills that
    would otherwise run inline shell preprocessing under
    gaia's settings scope. Composing skill-, command-, and
    agent-level content with gaia — under a structural
    allow-list that enumerates the safe subset (no
    `allowed-tools`, no `hooks:`, no `agent:`, no
    `context: fork`, no alternate `shell`, no leading-`!`
    body lines, no `` `!` `` preprocessor blocks) — is
    tracked in §14 alongside the v1.x `--force-merge-hooks`
    work; the `disableSkillShellExecution` backstop stays in
    effect regardless of the allow-list. Landing the
    allow-list is non-breaking from the repo layout's
    perspective. `gaia init` prints each offending path under
    the three roots and points at the remediation: remove the
    files entirely, move them outside `.claude/`, or wait for
    the v1.x allow-list.
  - A non-empty value under `enabledPlugins` or
    `extraKnownMarketplaces` (the current Claude Code
    project-settings keys that load plugins and plugin
    marketplaces) in the `.claude/settings.json` gaia is
    about to commit. `strictKnownMarketplaces` is managed-
    settings-only per current Claude Code docs, so a value
    under it in *project* settings is not a load-bearing
    plugin-loader; gaia does not block on it unless a future
    Claude Code version changes the scope. Any plugin-loading
    key future Claude Code versions add under a new name
    that gaia has not classified is treated as a warning
    until classified, rather than a silent allow — the
    reconcile step (§8.8) flags unknown top-level keys in
    the settings subset for operator review.
  `gaia init` refuses to complete setup if any of the above
  is present and points at the remediation (remove, or wait
  for the v1.x allow-list in §14). `gaia doctor --cloud`
  re-runs every scan on the work-branch tree it sees from
  inside the VM so drift after install is detected. A
  committed `.mcp.json` that only declares read-only servers
  is still refused in v1 — gaia has not yet modeled even
  read-only connectors' interactions with the state-branch
  push surface, and a permissive v1 default would foreclose
  adding stricter policy later without breaking repos.

## 3. Architecture

Everything ships as one npm package. The unscoped name `gaia` appears
to be taken on the public npm registry; v1 publishes under a scoped
name that still installs a `gaia` binary:

> **R0 release blocker — package name unfinalized.** The string
> `@your-scope/gaia` that appears throughout this spec is a
> **placeholder**, not a scope gaia controls. It MUST be replaced
> with the finalized scoped name before any real v1 publish or
> install documentation. Every `@your-scope/gaia` in `gaia init`
> output, `gaia doctor` output, the VM setup script, and the
> error messages cited in §8.0 / §12 has to switch in the same
> release. Until this is resolved, the spec is **design-complete
> but not build-complete**: no v1 tag, no "v1 preview" messaging,
> no public install line can ship while the placeholder remains.
> A prototype/alpha can use any scope the author controls as long
> as the same scope is used in every place the placeholder appears,
> but such a build is labeled pre-R0 and not v1.
>
> **CI gate (release-blocking).** The release pipeline includes a
> check that fails a v1 tag or publish if the literal string
> `@your-scope/gaia` appears in any emitted file — source, CLI
> output snapshots, `gaia init` / `gaia doctor` golden files, VM
> setup-script template, fixtures, generated docs, and error-
> message snapshots in tests. This makes a forgotten placeholder a
> hard CI failure rather than something that ships and breaks the
> install line. Alpha / pre-R0 tags are permitted to contain the
> placeholder; the gate keys off the v1 release label, not on
> every build.

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
   - **`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` set on the
     environment.** This is the current Claude Code toggle
     that strips Anthropic / cloud credentials from subprocess
     environments and isolates Bash subprocesses on Linux. It
     is not a full sandbox and does not replace the
     `PreToolUse` shim, but it reduces the blast radius of
     prompt injection through `Bash`: a session that talks a
     subprocess into exfiltrating environment variables cannot
     reach Anthropic API keys or the cloud platform's own
     credentials if they have been scrubbed before the
     subprocess started. `gaia init` prints the exact place
     to add this env var in the Claude Code environment UI,
     and `gaia doctor --cloud` probes for it by running a
     synthetic `env | grep -i ANTHROPIC` Bash call via the
     VM-side doctor hook and confirming the scrubbed keys are
     absent. The probe is `[inferred]` (§8.4) rather than
     verified because the scrub surface is under active
     Claude Code evolution; a newly-added credential env var
     that gaia has not named would slip past the grep.
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
   are tolerated regardless of matcher — gaia's matcher is `"*"`
   (§8.5), so every foreign handler overlaps by construction, but
   the most-restrictive rule means denials compose correctly
   (§8.7). Project-level handlers on most **other**
   events (`SessionStart`, `PostToolUse`, `SubagentStop`,
   `Notification`, `ConfigChange`, etc.) are not refused, but
   `gaia init` enumerates them and prints a `gaia does not
   control these project hooks` section so the repo owner knows
   what will run alongside gaia. `SessionStart` specifically
   gets a louder warning — it runs before `UserPromptSubmit`
   and can mutate the worktree in ways gaia's invariants assume
   do not happen, so repos that register it should verify via
   `gaia doctor --cloud` that the composition still produces
   clean sentinels. **`TaskCreated` and `TaskCompleted` are the
   exception to the toleration rule**: they can spawn
   agent-shaped work outside the `PreToolUse` shim's visibility,
   so `gaia init` refuses setup when a project-scoped handler
   is registered on either event (§6.4, §8.4 check 23, §8.7).
   `.mcp.json` in the repo, any committed file under
   `.claude/skills/**`, `.claude/commands/**`, or
   `.claude/agents/**`, and any non-empty `enabledPlugins` or
   `extraKnownMarketplaces` entry in `.claude/settings.json`
   are all hard refusals per §2 — even if none of these declare
   hooks on gaia's events, their frontmatter can grant
   write-capable tools, fork contexts, select subagents, or
   run inline shell preprocessing that bypasses the
   `PreToolUse` shim entirely.
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
user branch's recorded upstream (§6.1.1 step 7). That push uses
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
   fetch the upstream using an explicit refspec (matching §6.1.1
   step 4 — `git fetch remote branch` does not reliably update
   `refs/remotes/...` for every configuration, especially when
   the user branch tracks a non-`origin` remote or a renamed
   upstream):
   ```
   git fetch $upstream_remote \
     "+$upstream_ref:refs/remotes/$upstream_remote/$upstream_branch"
   ```
   Then classify the relationship between local `HEAD` and
   `$upstream_remote/$upstream_branch`:
   - **Upstream ref missing** (branch deleted on remote or
     never pushed) → treat as `local_only_branch: true` for
     this run and record that on the manifest; print a note.
   - **Upstream is ancestor of local** (local has unpushed
     commits) → allowed. Print a note that the unpushed
     commits will land on the gaia work branch on origin
     when the work branch is pushed, so the session sees
     them. The merge-back's upstream push (§6.1.1 step 7)
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
     merge-back that was guaranteed to fail at §6.1.1 step 3 /
     step 5.
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
4. Build the run branches locally (no remote push yet — the state
   branch needs to carry the manifest and, if overflow triggers
   at step 5, the overflow context.md in a single push, so the
   push is deferred to step 5a below). The branch-creation
   sequence MUST NOT leave the caller's primary worktree
   checked out on `claude/gaia/$RUN_ID/work` or
   `claude/gaia/$RUN_ID/state` — from the caller's perspective,
   `HEAD` stays on the user branch for the entire fire. The
   implementation uses `git worktree add` (preferred) or the
   `git branch` + `git commit-tree` + `git update-ref` plumbing
   triad, both of which create / advance refs without moving
   the caller's `HEAD`. Do not use `git checkout -B` / `git
   switch -c` on the caller's worktree; those would leave the
   caller on a gaia branch if the driver crashed before the
   intended checkout-back step.
   - Seed a local `claude/gaia/$RUN_ID/work` from the (possibly
     fast-forwarded) user branch (via `git branch
     claude/gaia/$RUN_ID/work $USER_BRANCH` — no checkout
     against the caller's worktree).
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
   - **Prepare the local state branch's tree — no commit yet.**
     Create an orphan working tree for the state branch
     (temporary worktree or plumbing-only `git mktree` / `git
     commit-tree` flow). `refs/heads/$GAIA_STATE_BRANCH` is
     not advanced yet; no commit exists on the branch at this
     point. Earlier drafts said "manifest is the state
     branch's first commit" here, which was ambiguous — it
     implied the manifest was already committed before step 5
     could add overflow `context.md`, even though the spec's
     intent is that both files land on **exactly one** root
     commit. Step 5 below writes both files and creates the
     single commit; step 4 only stages the tree.
5. Compose the payload (§7.2) with `"<prompt>"` as `CONTEXT`,
   then build the state branch's single root commit. The
   ordering is normative:
   1. Serialize `.gaia-state/manifest.json` into the state
      branch's working tree (§7.4).
   2. Measure the composed payload's character count.
   3. If the payload exceeds `max_payload_chars` (default
      65,536 for `claude-code-routine`; §11.2), serialize the
      overflow body into `.gaia-state/context.md` in the same
      working tree and replace the overflowed content in the
      payload with the `--- CONTEXT FILE ---` marker per §7.2.
      If not, omit `context.md`.
   4. Create **exactly one** root commit on
      `refs/heads/$GAIA_STATE_BRANCH` whose tree contains the
      manifest (and `context.md` if step 3 wrote it). No prior
      commit exists on the branch, and no subsequent commit is
      made before the push in step 5a; origin sees one
      state-branch push with one commit regardless of whether
      overflow triggered.
   Folding overflow and manifest into the same root commit
   keeps the state branch monotonic: one commit lands the
   manifest plus any overflow context, then `/fire` runs. An
   earlier draft committed the manifest in step 4 and appended
   a second commit with `context.md` after the state branch had
   already been pushed, which required a second origin push
   after `meta.status = "pending"` had advanced.
5a. Push `claude/gaia/$RUN_ID/work` and
    `claude/gaia/$RUN_ID/state` to origin **in that order** (the
    work branch first, then the state branch carrying the
    manifest plus any overflow `context.md` produced in step 5).
    **Only after both pushes succeed**, update `meta.json`
    under the lock with `status: "pending"` and
    `branches_pushed: true` (see §10.3's substates —
    `fire_attempted` and `request_body_started` stay `false` at
    this point). Writing the status *after* the push keeps the
    invariant in §10.3 honest: `pending` implies the branches are
    on origin, and `preparing` implies they are not. An earlier
    draft flipped these sub-steps (set `pending`, then push) and
    opened a crash window where `meta.status` claimed branches
    existed before they did, which confused recovery.

    **Partial-push and crash-window handling.** Three outcomes
    can land a run in a recoverable-from-`preparing` state with
    `meta.status: "preparing"` still on disk:

    (a) **Work-push fails outright.** No remote ref was
    created (git push is an atomic operation on a single ref).
    The driver's live process transitions
    `meta.status = "fire-failed"` with
    `error: "initial-branch-push-failed"`, appends the
    terminal `"fire-failed"` event, leaves no origin refs to
    delete, and exits 5. Nothing to recover.

    (b) **Work-push succeeds, state-push fails** (or the
    driver crashes in between). On the live process this is
    the same ordering: the driver retries the state-branch
    push within §11.2's bounded window; if retries exhaust,
    it deletes the just-created `claude/gaia/$RUN_ID/work`
    ref from origin under the per-run lock, transitions
    `meta.status = "fire-failed"` with
    `error: "initial-branch-push-failed"`, appends the
    terminal event, and exits 5. If the driver crashed
    between the two pushes, the state `{preparing, only work
    branch on origin}` is visible to `gaia recover`; the
    `preparing` recovery table row (§6.1.2) handles it by
    deleting the orphan work branch and transitioning to
    `fire-failed` with
    `error: "driver-crashed-after-work-push-before-state-push"`.
    No session could have been created (the manifest never
    reached origin for the hook to fetch), so there is no
    sentinel to poll for.

    (c) **Both pushes succeed, driver crashes before the
    lock-guarded status flip to `pending`.** The state is
    `{preparing, both branches on origin, manifest on state
    branch}`. Because `/fire` has not run (the flip is
    sequenced *before* the HTTP call in step 6), no session
    can exist. The `preparing + gaia branches observed on
    origin` recovery row (§6.1.2) handles it: promote
    `meta.status = "pending"` with `branches_pushed: true,
    fire_attempted: false` under the per-run lock, then apply
    the matching `pending` recovery row (delete both origin
    branches, transition to `fire-failed` with
    `error: "driver-crashed-after-push-before-fire"`, append
    terminal event). The recovery-time promotion is what
    makes the ls-remote probe in the `preparing` row
    load-bearing; without it, a crash in this window would
    leak branches on origin indefinitely.

    All three cases result in `fire-failed` with a distinct
    `error` field that names the observed failure so
    operators can tell push-failure from driver-crash from
    state-push-partial at audit time.
6. Fire the spawner. Before the HTTP call, flip the runtime
   substates under the lock: `fire_attempted: true` (so a crash
   between this write and the first send-attempt is classifiable
   as "fire attempted but not sent").

   **HTTP client contract (normative).** gaia uses a low-level
   HTTP client that exposes the socket lifecycle explicitly —
   v1 ships Node's built-in `http`/`https` module directly
   (not `fetch`, not the high-level `undici` request helper).
   The concrete discipline:
   - **Disable connection reuse for `/fire`.** Pass
     `agent: false` on the request options, or equivalently a
     fresh `new https.Agent({keepAlive: false, maxSockets: 1})`
     per request. A reused keep-alive TLS socket can already
     be connected by the time the caller registers the
     `'secureConnect'` listener — the event fires once per
     socket, so on a reused socket the listener never runs and
     the substate flip below never happens. A fresh socket per
     `/fire` call guarantees the connect event is observable
     before any body byte is written.
   - Construct the request with `http.request(options)`. Do
     **not** pass a body argument or call `req.end()` yet.
   - Register a listener for the `socket`'s `'connect'` event
     (for plain HTTP) or, with HTTPS, the TLS socket's
     `'secureConnect'` event. DNS resolution and TCP / TLS
     handshake complete before either fires. The
     non-keep-alive agent above ensures the listener is
     attached *before* the socket transitions to connected.
   - On `'connect'` / `'secureConnect'`, flip
     `request_body_started: true` under the per-run lock via
     `atomicMetaWrite` (§6.1's write protocol: write temp →
     fsync temp → rename → fsync parent dir) before calling
     `req.write(body)` / `req.end()`. The fsync-then-rename
     pattern is mandatory; a naive `fs.writeFileSync` of the
     flipped substate can lose the write on a power failure
     even after the fsync returns, because the directory
     entry is not durable until the containing directory is
     fsynced.
   - Only then attempt to write the body.

   This sequence makes the substate flip load-bearing: DNS
   failures, TCP connect failures, and TLS handshake failures
   all surface as events that fire **before** the connect
   listener runs, so `request_body_started` stays `false` and
   the driver can safely retry with the same `GAIA_RUN_ID`
   (the session is provably not created — no bytes left
   userspace). A failure **after** the connect event fires is
   classified as ambiguous because the driver cannot prove
   whether request-body bytes reached the server before the
   connection broke. High-level clients that hide the socket
   lifecycle (`fetch`, `undici.request` with a body argument)
   can dispatch writes before the handshake completes and
   would misclassify pre-handshake failures as ambiguous,
   burning `GAIA_RUN_ID`s on failures that were safely
   retryable; the low-level `http` module avoids that by
   letting the driver gate the flip on the socket event.

   Corporate HTTP proxies that terminate TLS at the proxy can
   also reuse sockets transparently. Disabling keep-alive at
   the agent layer is the load-bearing protection against
   proxy-side socket reuse — gaia cannot observe the proxy's
   socket pool, but a fresh `https.Agent` per request
   guarantees the driver's Node process starts a new outbound
   connection regardless of what the proxy does downstream.

   If a future gaia version switches the client (e.g., to
   `undici.Client.dispatch()` using its `onConnect` callback),
   the same contract applies: the client MUST expose a hook
   that fires after the socket is ready and before any body
   byte is written, the client MUST NOT reuse sockets across
   `/fire` calls, and gaia MUST flip the substate via
   `atomicMetaWrite` in that hook. Bypassing any part of this
   discipline is a protocol bug.

   Both substates are fsynced before the write attempt to
   guarantee the substate reaches disk before any byte that
   could plausibly create a remote session. The `/fire` endpoint has no
   idempotency-key header documented, so every successful request
   creates a distinct session against whatever payload was accepted.
   The driver therefore never retries the same `GAIA_RUN_ID` across
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
   - **Admission-error HTTP envelope received** — a named
     error response from the documented admission-error set
     (400/401/403/404/429). gaia treats these as side-effect-
     free based on their documented semantics ("request
     rejected before work started"); this is not an explicit
     server-side idempotency guarantee, but the /fire
     contract does not document a path where a 4xx admission
     error leaves a session live on the other side, and the
     docs do not ship an idempotency-key header that would
     make the guarantee explicit. Retrying is therefore safe
     for the rows marked retryable (429 only). After the
     documented retry policy exhausts or the row is non-
     retryable, transition `meta.status = "fire-failed"`,
     append the `"fire-failed"` event, delete the work/state
     branches, and exit per the §11.2 mapping (10/11/12). If
     Anthropic later documents one of these codes as
     *possibly* side-effect-bearing, the corresponding row
     moves to the "ambiguous" bucket below without a CLI
     contract change.
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
   `session_id`. Any of the three terminal sentinel kinds —
   `GAIA_DONE: <session_id>`, `GAIA_FAILED: <session_id>`, or
   `GAIA_ABORTED: <session_id>` — flows through the shared
   `handleTerminalSentinel(kind)` routine below (steps 8–13).
   Each kind requires different artifacts and drives a different
   merge / exit path, but the shared routine guarantees every
   terminal path records `work_tip_sha`, copies the transcript
   locally when one exists, handles rescue artifacts uniformly,
   and finalizes `meta.status` + `history.jsonl` exactly once
   (§6.1 terminal-finalization invariant below). The earlier
   design, where `GAIA_FAILED:` and `GAIA_ABORTED:` exited
   immediately without fetching artifacts or the work-branch
   tip, left later sections (`gaia recover` on failed /
   aborted runs, `-c` transcript replay after `gaia recover
   --merge`, post-mortem audit) assuming fields that were
   never populated; unifying the path here closes that gap.

   Per-kind artifact expectations (verified by the shared
   routine before advancing). §7.3's "sentinel categories"
   paragraph splits terminal sentinels into two categories —
   **well-formed v1 sentinels** (the contract below) and
   **minimal emergency sentinels** produced by shim fail-closed
   paths; the driver requires the well-formed shape by default
   and relaxes the requirement only for the documented emergency
   categories, recording `error_details.minimal_sentinel: true`
   in `meta.json` on those paths. Concretely:
   - **Well-formed `GAIA_DONE:`** — commit MUST contain
     `.gaia-state/<session_id>.md`,
     `.gaia-state/<session_id>.transcript.md`, **and**
     `.gaia-state/<session_id>.meta.json` (the session-meta
     artifact from §7.3). If any of the three is missing, log a
     warning (suspected forgery, truncated hook run, or a
     pre-v1 Stop hook that did not emit `meta.json`) and keep
     polling through the remaining timeout before treating the
     commit as a minimal emergency sentinel.
   - **Well-formed `GAIA_FAILED:`** — commit MUST contain
     `.gaia-state/<session_id>.error.json` and SHOULD contain
     `.gaia-state/<session_id>.transcript.md` (may be empty if
     `StopFailure` fired before the first assistant message)
     and `.gaia-state/<session_id>.meta.json`. A missing
     transcript is recorded as `null` in `meta.json` and does
     not block finalization. A missing meta artifact is
     tolerated (the driver has `session_id` from the sentinel
     commit; `meta.session_url` stays at whatever the /fire
     response already populated, which may be non-null on the
     normal path).
   - **Well-formed `GAIA_ABORTED:`** — commit MUST contain
     `.gaia-state/<session_id>.md`,
     `.gaia-state/<session_id>.transcript.md`, and
     `.gaia-state/<session_id>.meta.json` (lab-quality abort
     always has all three — §15).
   - **Minimal emergency sentinel (any kind).** Produced only
     by documented fail-closed paths: the `UserPromptSubmit`
     shim fail-closed sentinel (§8.0 step 6 — binary missing,
     protocol-version mismatch), an early-failure `Stop` /
     `StopFailure` path that could not assemble a transcript,
     and the step-3 wrong-branch `GAIA_FAILED:` fallback (§8.2
     step 3) when the hook cannot reach the work tree. Required
     artifacts: the terminal commit message itself plus the
     sentinel-appropriate error-body JSON (for `GAIA_FAILED:`)
     or `.md` (for `GAIA_DONE:` / `GAIA_ABORTED:`). Transcript
     and `meta.json` artifacts MAY be omitted. The driver
     recognizes the sentinel by commit-message prefix, sets
     `meta.error_details.minimal_sentinel = true`, populates
     whichever fields the artifact supports, and finalizes per
     the kind's normal exit code. This is the only category
     permitted to ship without a transcript on `GAIA_ABORTED:`
     (lab-quality `:abort` always produces a well-formed
     sentinel; only hook-crash or binary-missing paths emit a
     minimal-emergency abort).
   Any sentinel may additionally carry `<session_id>.rescue.*`
   artifacts if the Stop / StopFailure hook could not push
   the work branch (§7.3); the shared routine handles those
   uniformly in step 8's rescue-handling paragraph.
8. **`handleTerminalSentinel(kind)` — fetch and record
   work-branch tip.** Fetch the work and state branches. After
   the fetch, record
   `meta.work_tip_sha = git rev-parse refs/remotes/origin/$GAIA_WORK_BRANCH`
   under the per-run lock (§10.3) for **every** terminal
   kind, not only `GAIA_DONE:`. This is the load-bearing
   point where the driver captures the post-session work-
   branch tip before §6.1.1's merge-back (on `GAIA_DONE:`)
   consumes it and before §6.1 step 12's branch-cleanup substep's delete may remove
   the remote ref. The `gaia recover` ancestry check on
   `--resume <session_id>` continuations (§6.2), a later
   `gaia recover --merge` on `aborted` / `failed` runs
   (§6.4), and post-`gc` audit all depend on this field being
   populated by every terminal path that reached step 8.

   If the work-branch push never landed (no `origin/$GAIA_WORK_BRANCH`
   ref or the ref exists but predates the run), the driver
   records `meta.work_tip_sha = null` and proceeds to the
   rescue-handling paragraph below — a rescue-restored tip,
   when reconstruction succeeds, overwrites the `null`.

   **Rescue-artifact handling (active driver).** If the
   sentinel commit contains any `<session_id>.rescue.*` file
   (§7.3 — the Stop / StopFailure hook could not push the
   work branch), the driver does **not** merge or accept the
   stale `origin/$GAIA_WORK_BRANCH` as-is regardless of the
   sentinel kind. It runs the same reconstruction path
   `gaia recover` uses (§6.4 — verify `rescue.head.txt`
   against the origin work-branch tip; prefer `rescue.bundle`
   over `rescue.patch`; apply the bundle into the local repo
   or the patch on top of the `rescue_base` recorded in
   `<session_id>.rescue.base.txt` (§7.3) — falling back to
   `manifest.base_sha` only when the rescue-base artifact is
   missing from a pre-v1 sentinel) to restore the missing
   commits into a **local private rescue ref**,
   `refs/gaia/recovered/<run_id>/work`, and records the ref
   name and the restored tip sha in `meta.rescue_ref` and
   `meta.work_tip_sha`. The rescue ref is deliberately **not**
   written under `refs/remotes/origin/$GAIA_WORK_BRANCH`:
   §6.1.1's merge-back begins with a force-fetch of the origin
   work branch into that exact tracking ref, which would
   silently overwrite the reconstructed tip with whatever
   stale (or missing) bytes the origin ref currently holds.
   Recording the rescue ref separately and passing it as an
   override into `mergeBack(run_id, {workRefOverride})`
   (§6.1.1) preserves the reconstruction across the
   merge-back; the driver skips the origin fetch in §6.1.1
   step 6 when `meta.rescue_ref` is set and merges the
   private ref directly. The rescue-restored tip is the value
   recorded as `meta.work_tip_sha`.

   **Rescue reconstruction failure — behavior depends on
   sentinel kind.**
   - `GAIA_DONE:` with rescue reconstruction failure →
     `meta.status = "timeout"` with
     `error: "rescue-reconstruction-failed"`, preserve the
     work / state branches on origin, exit 3, and point at
     `gaia recover <run_id>`. The agent *completed* but the
     work-branch bytes are unreachable, so the run is not a
     clean success.
   - `GAIA_FAILED:` with rescue reconstruction failure →
     `meta.status = "failed"` with
     `error_details.rescue_reconstruction_failed = true`,
     still print the error body, still exit 13. The agent
     failed *and* the partial work cannot be reconstructed,
     but the outcome for the user is the same: the session
     errored out and the transcript is what it is.
   - `GAIA_ABORTED:` with rescue reconstruction failure →
     `meta.status = "aborted"` with the same
     `error_details` flag, exit 0. User asked to abort; the
     inability to reconstruct unmerged work is surfaced in
     `gaia history show` but does not change the exit.
9. **`handleTerminalSentinel(kind)` — copy artifacts.** Copy
   the transcript artifact (when present, per the per-kind
   table in step 7) to `.gaia/transcripts/<session_id>.md`
   locally with YAML front matter preserved — the local
   store keeps the header for archival; it is stripped only
   at stdout time in step 12's final-print substep. If the
   sentinel is `GAIA_FAILED:`, also copy
   `.gaia-state/<session_id>.error.json` to
   `.gaia/runs/<run_id>/error.json` locally so `gaia
   history show` and `gaia recover` can surface the recorded
   error body after the state branch is pruned. If rescue
   artifacts were produced, copy the full set
   (`.patch`, `.bundle`, `.log.txt`, `.head.txt`, `.base.txt`)
   to `.gaia/runs/<run_id>/rescue/` for operator audit.
10. **`handleTerminalSentinel(kind)` — per-kind terminal path.**
    - `GAIA_DONE:` with `--no-merge` → skip merge-back; finalize
      with `status = "no-merge"` and preserve branches (§6.1.2).
    - `GAIA_DONE:` without `--no-merge` → **merge back
      transactionally** (§6.1.1).
    - `GAIA_FAILED:` → skip merge-back entirely (the session
      errored out; its work is not automatically landed).
      Finalize with `status = "failed"`, preserve work / state
      branches, print the error body to stderr, exit 13.
      `gaia recover <run_id> --merge` is the explicit opt-in
      path if the operator wants to land whatever the agent
      did commit before the failure.
    - `GAIA_ABORTED:` → skip merge-back entirely (user
      deliberately aborted — see §15). Finalize with
      `status = "aborted"`, preserve work / state branches,
      print the partial-transcript hint to stderr, exit 0.
      `gaia recover <run_id> --merge` is the explicit opt-in
      path if the user changes their mind.
11. **`handleTerminalSentinel(kind)` — finalize the run (before
    the user-facing print).** Update `meta.json` under the
    per-run lock (§10.3) using the `atomicMetaWrite` protocol
    (write temp → fsync temp → rename → fsync parent dir)
    with the terminal status selected in step 10 (`success`,
    `no-merge`, `failed`, or `aborted`), `work_tip_sha` (from
    step 8 — already recorded for every kind), `finished_at`,
    and on `GAIA_DONE:` without `--no-merge` also `merge_sha`
    and `pushed_upstream`. Then append a single corresponding
    event to `.gaia/history.jsonl` (with `summary` populated
    from the truncated final message on `success` /
    `no-merge` / `aborted`, or from the recorded error on
    `failed`) under the history lock. History is written once
    per terminal transition; readers treat the latest history
    record for a run_id as authoritative. **Finalize precedes
    the final stdout/stderr print** so a broken stdout (EPIPE
    / SIGPIPE from a caller that closed the pipe early — e.g.
    `gaia -p "..." | head -n1`) cannot skip the meta/history
    writes. Without this ordering a `pending` run that
    actually completed could linger in `meta.json` because
    the driver died writing to stdout before finalize ran.
12. **`handleTerminalSentinel(kind)` — branch cleanup and
    user-facing print.** The two happen in this order, still
    before the driver returns:
    - **Branch cleanup.** On `success`, delete
      `claude/gaia/$RUN_ID/{work,state}` from origin unless
      `--keep-branches` was set. This is the **canonical
      branch-deletion step**; §6.1.1 step 8 only releases the
      merge lock and does not delete. Additionally, on
      `success` when `meta.rescue_ref` is non-null (the
      merge-back ran with `workRefOverride` — §6.1 step 8),
      best-effort `git update-ref -d refs/gaia/recovered/<run_id>/work`
      to release the local rescue ref; the reconstructed
      commits are already held alive by the merge commit's
      second parent. On every other terminal kind
      (`no-merge`, `failed`, `aborted`, and any §6.1.1 exit
      that finalized with `conflict` / `push-rejected` /
      `timeout`), preserve both branches on origin for `gaia
      recover`'s explicit paths, and preserve the rescue ref
      (if any) for a later `gaia recover --merge`. If the
      `success`-path delete push fails (auth glitch, branch
      protection, network blip), the run **stays `"success"`**
      — the user's code is already merged and pushed at step
      10's merge-back; leftover branches on origin are
      cosmetic clutter, not a correctness problem. `gaia`
      prints a warning naming the branches that could not be
      deleted and points at `gaia gc` (§6.4) to sweep them on
      the next invocation; `meta.json.status` is **not**
      downgraded from `"success"` to `"push-rejected"` or
      `"conflict"` in this case, because those statuses imply
      the user branch did not get the run's commits, which is
      false. The worst case is a week of extra refs on origin
      until `gaia gc` runs — benign.
    - **Print final message.** On `GAIA_DONE:`, print
      `.gaia-state/<session_id>.md`'s body (with the YAML
      front-matter header from §7.3 stripped) to stdout as the
      final assistant message. The header is machine-readable
      bookkeeping the caller should not see, whereas the body
      is what gaia presents as the session's answer. A
      `--json` invocation (§6.3) places the same stripped body
      in the `final_message` field. On `GAIA_FAILED:`, print
      the error body from `.gaia-state/<session_id>.error.json`
      to stderr (no stdout); a `--json` invocation emits
      `{final_message: null, error: {...}}`. On
      `GAIA_ABORTED:`, print the `.gaia-state/<session_id>.md`
      body plus the partial-transcript hint to stderr; a
      `--json` invocation emits `{final_message: <body>,
      error: null, status: "aborted"}`. The print is wrapped
      in a try/catch that swallows `EPIPE` (the run is already
      finalized; a caller who closed the pipe does not need
      the message). Implementations MUST additionally install a
      SIGPIPE handler that ignores the signal so the Node
      process does not die before returning its exit code.

**Terminal-finalization invariant.** Steps 8 – 13 above — the
shared `handleTerminalSentinel(kind)` routine — are the body of
a single driver-side wrapper whose contract is: **every exit
path, for every terminal sentinel kind and regardless of
whether §6.1.1 succeeded, must write exactly one terminal
`meta.status` under the per-run lock and append exactly one
terminal event to `history.jsonl` under the history lock
before the driver returns.** §6.1.1 exits (2, 4, 15) surface
distinct outcomes but they must **not** short-circuit the
finalize step; likewise, `GAIA_FAILED:` and `GAIA_ABORTED:`
must **not** short-circuit step 8's `work_tip_sha` record /
transcript copy. An implementation that codes step 10 as
"call merge-back, propagate non-zero exit on failure" and
omits the `meta.status` / `history.jsonl` write leaves the
run visible to subsequent `gaia -c` / `gaia recover`
invocations as a stale `pending` — which is exactly the bug
this invariant exists to prevent.

Reference pseudocode for the wrapper, one row per observed
outcome:

```
handleTerminalSentinel(run_id, session_id, sentinel_kind):
  # Lock-acquisition order is merge.lock → run.lock → history.lock
  # (§10.3). run.lock is held only across short meta-writes, never
  # across mergeBack() or stdout prints. Finalize ALWAYS precedes
  # the final stdout/stderr print so a broken stdout (EPIPE /
  # SIGPIPE from a caller that closed the pipe early) cannot leave
  # meta.status as pending while the merge already landed.

  # Shared fetch / record / artifact-copy path (steps 8–9)
  fetchWorkAndStateBranches(run_id)
  validatePerKindArtifacts(sentinel_kind, session_id)   # step 7
  work_tip_sha = readOriginWorkTip(run_id)              # may be null
  rescue_ref = null
  if sentinelHasRescueArtifacts(sentinel_kind, session_id):
    try:
      (work_tip_sha, rescue_ref) =
        reconstructWorkTipFromRescue(run_id, session_id)
    except RescueReconstructionFailed:
      if sentinel_kind == "GAIA_DONE":
        finalize(run_id, status="timeout",
                 error="rescue-reconstruction-failed",
                 work_tip_sha=null, exit_code=3,
                 preserve_branches=true,
                 final_print=null)
        return
      if sentinel_kind == "GAIA_FAILED":
        finalize(run_id, status="failed",
                 error_details={rescue_reconstruction_failed: true},
                 work_tip_sha=null, exit_code=13,
                 preserve_branches=true,
                 final_print=errorBody(session_id))
        return
      if sentinel_kind == "GAIA_ABORTED":
        finalize(run_id, status="aborted",
                 error_details={rescue_reconstruction_failed: true},
                 work_tip_sha=null, exit_code=0,
                 preserve_branches=true,
                 final_print=abortedMessage(session_id))
        return

  # Short run.lock hold to record work_tip_sha / rescue_ref
  with_lock(".gaia/runs/<run_id>/.lock"):
    recordWorkTipSha(run_id, work_tip_sha, rescue_ref)

  copyTranscriptIfPresent(run_id, session_id)
  copyErrorJsonIfFailed(run_id, session_id, sentinel_kind)
  copyRescueArtifactsIfPresent(run_id, session_id)

  # Per-kind terminal path (step 10) — finalize before print.
  if sentinel_kind == "GAIA_ABORTED":
    finalize(run_id, status="aborted", exit_code=0,
             preserve_branches=true,
             final_print=abortedMessage(session_id))
    return

  if sentinel_kind == "GAIA_FAILED":
    finalize(run_id, status="failed", exit_code=13,
             preserve_branches=true,
             final_print=errorBody(session_id))
    return

  # GAIA_DONE path
  if --no-merge passed:
    finalize(run_id, status="no-merge", exit_code=0,
             preserve_branches=true,
             final_print=finalAssistantMessage(session_id))
    return

  # mergeBack() acquires merge.lock internally; no run.lock held
  # here so the merge-back does not deadlock against a concurrent
  # `gaia recover --merge` that also wants merge.lock. Pass
  # rescue_ref so §6.1.1 step 6 uses the private rescue ref
  # instead of fetching origin (when rescue reconstruction ran).
  try:
    mergeBack(run_id, workRefOverride=rescue_ref)   # §6.1.1
  except LocalDirtyOrDivergent as e:         # §6.1.1 steps 1–3
    finalize(run_id, status=e.recoverable_status,
             error=e.detail, exit_code=2,
             preserve_branches=true,
             final_print=null)
    return
  except MergeConflict:                      # §6.1.1 step 6
    finalize(run_id, status="conflict", exit_code=4,
             preserve_branches=true,
             final_print=null)
    return
  except UpstreamPushRejected:               # §6.1.1 step 7
    finalize(run_id, status="push-rejected", exit_code=15,
             preserve_branches=true,
             final_print=null)
    return

  # Merge-back succeeded
  finalize(run_id, status="success", exit_code=0,
           preserve_branches=(--keep-branches ? true : false),
           final_print=finalAssistantMessage(session_id))
  # Branch-delete is inside finalize() on success paths, but
  # its failure does not downgrade status — see step 12's
  # branch-cleanup substep.

finalize(run_id, status, ..., final_print):
  # 1. Under run.lock: write the terminal meta.status and fields.
  # 2. Under history.lock (nested inside run.lock): append
  #    exactly one history.jsonl event. Release both locks.
  # 3. Best-effort remote branch cleanup (failure does not
  #    downgrade status; see §6.1 step 12's branch-cleanup substep).
  # 4. Print the final_print payload to stdout/stderr.
  #
  # The print is LAST so a stdout EPIPE (pipe closed by the
  # caller — e.g., `gaia -p "..." | head -n1`) cannot skip
  # the meta/history/cleanup work. EPIPE / SIGPIPE on step 4
  # is caught and swallowed: the run has already been
  # finalized, and the caller who closed the pipe is not
  # waiting for the message anyway. Implementations MUST
  # install a SIGPIPE handler that ignores the signal (Node:
  # `process.stdout.on('error', e => e.code === 'EPIPE' ?
  # null : throw e)`) and MUST write the final message inside
  # a try/catch that treats EPIPE as success.
  with_lock(".gaia/runs/<run_id>/.lock"):
    atomicMetaWrite(run_id, {status, ...terminal_fields})
    with_lock(".gaia/history.lock"):
      append exactly one history.jsonl event
  if not preserve_branches:
    best-effort delete origin branches (failure → log only;
    status stays "success" per step 12 above)
  # Rescue-ref cleanup on success: the reconstructed commits
  # are now held alive by the merge commit's second parent,
  # so the private ref can be dropped. Deletion only runs on
  # clean "success"; every non-success status preserves the
  # rescue ref for a later gaia recover --merge.
  if status == "success" and meta.rescue_ref is not null:
    best-effort `git update-ref -d` the rescue ref
  try:
    if final_print is not null:
      writeFinalPrint(final_print)          # stdout or stderr
  except EPIPE:
    pass   # meta/history/cleanup are already durable
```

`atomicMetaWrite(run_id, fields)` is the normative write
protocol for every `meta.json` mutation in §10.3. It MUST:
1. write the updated JSON body to a temp file
   (`meta.json.tmp.<pid>`) in the same directory;
2. `fsync` the temp file descriptor;
3. `rename` the temp file over `meta.json` (atomic on POSIX);
4. `fsync` the parent directory so the rename hits disk.
Without step 4 the rename can be lost on a power failure even
after the file contents were synced. Every `.gaia/runs/<run_id>/meta.json`
update — substate flips in §6.1 steps 3/5a/6, terminal writes
via `finalize()`, and `gaia recover`'s per-status mutations —
goes through this protocol.

The **exactly-one** guarantee is what each `finalize()` call
encodes: `history.jsonl` is append-only and §10.3 requires one
event per terminal transition per run, so the wrapper must not
call `finalize()` more than once on a single invocation. A
subsequent `gaia recover` that promotes a non-terminal-to-
terminal transition (e.g., `fire-ambiguous → success`) is a
separate driver invocation and appends its own event; the
latest record per `run_id` wins at read time (§10.3).

The same wrapper shape applies to `gaia recover` paths that
finalize a stuck run: every `gaia recover` exit — merged,
re-conflicted, push-retry-succeeded, push-retry-rejected,
late sentinel, `--assume-failed` — flows through a
`finalize()`-shaped write so the post-recovery run never
stays in a non-terminal state. The recover-side wrappers
live in §6.4 per-status branches; they inherit the same
exactly-one contract described here.

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
run_id that holds the lock. The lock is held through step 8
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
3. **Detect local commits on the user branch during the run.**
   Compare `git rev-parse HEAD` to `meta.base_sha` (§10.3 — the
   local `HEAD` sha recorded at §6.1 step 3 before any remote side
   effect). If they differ, the user committed to the user branch
   locally during the run. Exit 2 with a recovery message pointing
   at `--no-merge` (complete the session, defer the merge), manual
   reconciliation, or `gaia recover <run_id>` after the user has
   integrated their local commits into a compatible base. gaia
   refuses to merge gaia's work-branch commits on top of
   out-of-band local commits silently because it cannot tell
   whether the two sets of changes are compatible; a naive
   fast-forward or no-ff merge could overwrite reasoning the user
   had not committed when the run started. Relying on
   `git merge --ff-only` at step 5 to catch this is insufficient:
   if upstream did not move during the run, the local branch's new
   commits still make `upstream` an ancestor of `HEAD`, so the
   `--ff-only` at step 5 would report "Already up to date" and the
   work-branch merge below would proceed on top of the user's
   unrelated local commits.
4. If `meta.local_only_branch == false`, run
   ```
   git fetch $UPSTREAM_REMOTE \
     "+$UPSTREAM_REF:refs/remotes/$UPSTREAM_REMOTE/$UPSTREAM_BRANCH"
   ```
   using `upstream_remote`, `upstream_branch`, and `upstream_ref`
   from `meta.json` (§10.3). This picks up any upstream advance
   regardless of remote name, branch-rename, or fork
   configuration. Local-only user branches skip this step (§5,
   `meta.local_only_branch == true`).
5. If the user branch tracks an upstream and the local branch is
   still at `meta.base_sha` (per step 3), fast-forward local to
   `$UPSTREAM_REMOTE/$UPSTREAM_BRANCH`. This picks up any
   upstream advance that landed during the run; the step-3 check
   has already guaranteed the local branch did not advance
   out-of-band. If the fast-forward is impossible here (should
   not happen given the step-3 and step-4 invariants, but
   defensive), exit 2 with a recovery message.
6. Fetch the work branch with an explicit refspec and merge it.
   The merge source depends on whether the driver invoked
   `mergeBack(run_id)` with a `{workRefOverride}` argument
   (set when rescue reconstruction produced a private ref at
   `refs/gaia/recovered/<run_id>/work`; see §6.1 step 8's
   rescue-artifact handling and §7.3's restore sequence):
   - **No override (normal path).** Force-fetch the origin
     work branch into its tracking ref and merge:
     ```
     git fetch origin \
       "+refs/heads/$GAIA_WORK_BRANCH:refs/remotes/origin/$GAIA_WORK_BRANCH"
     GIT_MERGE_AUTOEDIT=no \
       git merge --no-ff --no-edit "refs/remotes/origin/$GAIA_WORK_BRANCH"
     ```
   - **Override set (rescue path).** Skip the origin fetch —
     a force-fetch here would overwrite any `refs/remotes/`
     tip with stale origin bytes, and the private rescue ref
     already carries the reconstructed tip. Merge the override
     ref directly:
     ```
     GIT_MERGE_AUTOEDIT=no \
       git merge --no-ff --no-edit "$workRefOverride"
     ```
     with `$workRefOverride == refs/gaia/recovered/<run_id>/work`.
   Both `GIT_MERGE_AUTOEDIT=no` and `--no-edit` are mandatory
   (§7.4 commit identity table) — gaia runs non-interactively
   and must not open `$EDITOR` for the merge commit message.
   The merge commit uses the **caller's** configured git
   identity and signing policy (unlike the gaia-authored
   work-branch commits above); merges are the caller's
   "accept this session's work" action and should flow through
   the caller's normal attribution / signing flow.
   Conflict → `git merge --abort`, exit 4; work branch (or
   rescue ref) preserved. The work branch always lives on
   `origin` (the Claude GitHub proxy pushes there); the
   upstream for the user branch can be a different remote
   entirely (`upstream_remote` above). The private rescue ref
   namespace (`refs/gaia/recovered/*`) is driver-local — it
   is not pushed anywhere and is cleaned up by `gaia recover`
   once the run is promoted to `success` (§6.4).
7. If `meta.local_only_branch == false`, push with **explicit
   full refs on both sides of the colon** so slash-heavy branch
   names cannot resolve ambiguously to a tag or partial-name
   match:
   ```
   git push $UPSTREAM_REMOTE \
     "refs/heads/$USER_BRANCH:refs/heads/$UPSTREAM_BRANCH"
   ```
   (`refs/heads/...` on the left-hand side is the local branch
   ref; `refs/heads/...` on the right-hand side targets the
   upstream branch by its full ref, matching `meta.upstream_ref`
   recorded at fire time — §7.4.) Never force. Rejection →
   exit 15 (merge local, push upstream) with instructions:
   investigate the upstream advance, then `git push` manually
   when safe **or run `gaia recover <run_id>`** (which retries
   the push without re-merging; see §6.4). The local user
   branch already has the merge commit. Local-only user
   branches skip this step entirely; the run finishes with the
   merge commit on the local branch only and `meta.json`
   records `pushed_upstream: false` for clarity.
8. Release the merge lock. Remote-branch deletion and history
   finalization are §6.1's responsibility (steps 12–13), not
   §6.1.1's — earlier drafts duplicated the delete here, which
   diverged from the `--no-merge` skip-step-13 contract in §6.1.2
   and from the per-status branch-retention rules in the §6.1.2
   state table. §6.1.1 owns the merge-back transaction; cleanup
   and history append happen in the caller's wrapper sequence.

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

When the caller passes `--no-merge`, the driver performs the
same §6.1 flow up through the transcript copy, prints the final
assistant message, finalizes the run with a dedicated status,
and preserves both run branches on origin so `gaia recover` can
complete the merge later. Concretely, relative to §6.1's
numbered steps:

- Steps 1 through 9 (including step 5a's push) run unchanged:
  ID generation, upstream preflight, preparing-record write,
  build the run branches locally, compose payload (with overflow
  `context.md` written in step 5 if needed), push work and state
  branches in step 5a, fire, poll for sentinel, fetch work/state
  branches, and copy the transcript artifacts.
- **Step 10 (transactional merge-back) is skipped entirely** —
  the whole of §6.1.1 does not run.
- **Step 11 (finalize the run) runs with modified content:**
  `meta.json.status` is set to `"no-merge"` instead of
  `"success"`, `merge_sha` stays `null`, `pushed_upstream` is
  `false`, `work_tip_sha` is populated from §6.1 step 8's
  fetch (the field is recorded for every status that reached
  step 8, including `no-merge`, so a later `gaia recover` can
  read it), and a single terminal `"no-merge"` event is
  appended to `history.jsonl` under the lock. Finalize
  precedes the user-facing print, identical to the stable
  path.
- **Step 12 (cleanup + print) runs with the cleanup substep
  skipped:** `claude/gaia/$RUN_ID/{work,state}` are preserved
  on origin as if `--keep-branches` were also set (§12.1),
  and the final-print substep runs unchanged — stdout
  receives the session's final assistant message with the
  YAML front matter stripped, the same as a merged run.

Observable semantics (on top of the step-list above):

- `"no-merge"` is a terminal state distinct from `"success"`
  (merged) and from `"conflict"` / `"timeout"` (failures — see
  §12).
- stderr carries one line with the work-branch name and a
  `gaia recover <run_id>` hint so the caller can complete the
  merge later.
- Exit code is `0`. `--no-merge` is an intentional caller
  request, not a failure.
- `gaia -c` scoped to the current branch **does not** pick up
  `"no-merge"` runs (it only continues `"success"`). Neither
  does `gaia --resume <session_id>` — per §6.2, only runs whose
  `meta.status == "success"` are resumable. The supported path
  is `gaia recover <run_id>` to complete the merge and promote
  the run to `"success"`; subsequent `gaia -c` / `gaia --resume
  <session_id>` then proceeds normally. Continuation directly
  from a `no-merge` run is deliberately excluded from v1 — a
  `--resume-unmerged`-style flag is tracked in §14 as possible
  future work, not a v1 default.

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
| 17 | `gaia recover <id> --manual-merge` left a half-merged working tree for the user to resolve manually (§6.4 conflict flow). `meta.status` stays `"conflict"` until the user completes the merge and re-runs `gaia recover <id>`. Only emitted by `gaia recover --manual-merge`; the fresh-run merge-back never produces this code. |

**Where the normative status table lives.** The rest of this
section, §6.4, §12, §12.1, and §10.3 all describe individual
rows of the same state machine. The single normative source for
status transitions, branch retention, session existence,
`session_id` knowability, recover action, and terminality is
the **status transition table at the end of §6.1.2** below.
Narrative elsewhere is derived; if it disagrees with the table
the table wins (see the table's "normative" note for the full
contract).

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
ordered by the composite key `(started_at, run_id)` — the
`meta.json.started_at` ISO-8601 timestamp first, breaking ties
on the ULID embedded in `run_id`. Two gaia processes that fire
within the same millisecond would otherwise produce ULIDs whose
random tail does not encode wall-clock order; the
`started_at` prefix disambiguates explicitly, and v1 does not
rely on the monotonic-ULID generator variant (which guarantees
lexicographic order across same-millisecond generations at the
cost of occasional forward-skew). Clocks on the driver can run
slightly non-monotonically (NTP adjustment, VM host clock step,
daylight-saving transition on a laptop that sleeps mid-run);
the two-key ordering tolerates that because the run_id is an
ULID and still provides a stable per-millisecond tiebreaker.
  
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
  offending run. The error message points at `gaia recover
  <run_id>` for the recoverable cases (`no-merge`,
  `push-rejected`, `timeout`, `conflict`, and the opt-in merge
  path for `aborted` / `failed`); once recover promotes the run
  to `success`, `gaia -c` works normally. `fire-failed` has no
  recovery action (no session existed); `--resume <session_id>`
  is the deliberate path when the caller actually wants an
  older successful run on the same branch.
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

**Only `success` runs are resumable.** `gaia -c` and
`gaia --resume <session_id>` require the prior run's
`meta.status` to be `"success"`. Every other terminal state
(`"no-merge"`, `"conflict"`, `"timeout"`, `"aborted"`,
`"push-rejected"`, `"failed"`, `"fire-failed"`,
`"fire-ambiguous"`) exits 2 with a message naming the run and,
where applicable, pointing at `gaia recover <run_id>` as the
explicit path that promotes the run to `"success"` before
continuation can proceed. This is the "just run recover first"
contract: gaia does not auto-seed a new run's work branch from
a prior run's uncommitted work, because the surface that would
support it (tracking a divergent seed on the manifest, carrying
rescue artifacts across bases, reasoning about cross-branch
history rewrites) is the complexity the v1 scope deliberately
omits. Users who want to carry non-merged work forward run
`gaia recover <run_id>` — which completes the merge-back on
the current state machine's recoverable statuses, or (with
`--merge`) opts `aborted` / `failed` into a merge-back — and
then rerun `gaia -c`. `fire-failed` has no recovery action
(the session never existed); `fire-ambiguous` requires the
recover-side late-sentinel poll first.

**Containment guard for `--resume <session_id>` across
branches.** Even when the prior run is `"success"`, the
caller's current branch may not contain the prior run's
merge. A naive replay would hand Claude a transcript
describing file edits that are not actually in the new
session's working tree — a guaranteed source of "you said you
changed X but X is not here" confusion.

Guard: before firing the continuation, the driver reads
`meta.work_branch` for the prior run and requires the
ancestry check
`git merge-base --is-ancestor <work_branch_tip> HEAD` to
succeed on the caller's current branch. If the check fails,
the driver exits 2 with a message naming the recorded
`user_branch` so the caller can switch to it (or manually
merge the prior work into the current branch) before
retrying. This is the deliberately-narrow v1 posture: no
cross-branch opt-in flag, no auto-seed from a prior work
tip; either the continuation lands on a branch that already
contains the prior work, or it fails and the caller does the
cross-branch work by hand. Cross-branch replay through a
dedicated opt-in is tracked in §14 as possible v1.x work if
the ergonomics warrant it.

Both forms:

1. Locate the prior run in `.gaia/runs/*/meta.json` or
   `.gaia/history.jsonl`.
2. Read `.gaia/transcripts/<session_id>.md` (the readable transcript
   captured by the `Stop` hook; §8.2).
3. **Sanitize the transcript body** before injecting it. Strip the
   YAML front matter (§7.3 header) and, as defense in depth,
   strip any `--- BEGIN GAIA CONTROL ---` … `--- END GAIA CONTROL ---`
   fences and surrounding `--- BEGIN CONTEXT ---` / `--- END
   CONTEXT ---` / `PREVIOUS CONVERSATION:` / `END PREVIOUS
   CONVERSATION` / `--- CONTEXT FILE ---` / `--- END CONTEXT FILE ---`
   wrapper lines that may have been captured by an older gaia
   version's `Stop` hook (§8.2 step 9 now strips at capture
   time, but transcripts produced by pre-strip versions of the
   hook may still be on disk). The driver performs this strip
   at replay time rather than trusting the on-disk transcript
   to be clean — otherwise a fence that rode along in a legacy
   transcript would resurface in the current fire's prompt,
   violate §7.2's "last fence before `--- BEGIN CONTEXT ---`"
   selection rule, and steer the new run toward the old run's
   state branch.
4. Compose the payload with the sanitized transcript injected as a
   `PREVIOUS CONVERSATION:` block *above* the new prompt:
   ```
   --- BEGIN CONTEXT ---
   PREVIOUS CONVERSATION:
   <sanitized transcript>
   END PREVIOUS CONVERSATION

   <new prompt>
   --- END CONTEXT ---
   ```
5. Fire a fresh session (new `session_id`, new run) and proceed exactly
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
| `-c` / `--continue` | off | Prepend the most recent successful session's transcript. Scoped to current branch; refuses if the newest run is not `success` and points at `gaia recover`. See §6.2. |
| `--resume <session_id>` | off | Prepend a specific session's transcript. Mutually exclusive with `-c`. Only resumes `success` runs; non-success targets must be promoted via `gaia recover <run_id>` first (§6.2). |
| `--no-merge` | off | After the `GAIA_DONE:` sentinel arrives, skip the merge-back in §6.1.1 entirely. See §6.1.2 for exact semantics (history status, branch retention, stdout behavior, exit code). |
| `--keep-branches` | off | Do not delete `claude/gaia/$RUN_ID/*` on origin after merge. |
| `--spawner <name>` | from config | Override the spawner. |
| `--timeout <duration>` | `45m` | Per-session timeout. |
| `--json` | off | Print a single-line JSON result instead of bare text. Schema: `{run_id, session_id, status, final_message, merge_sha?, continued_from?, transcript_path?, error?}`. `final_message` carries the same content the bare-text mode prints to stdout. On failure (any non-zero exit), `--json` writes the JSON object to **stdout** with `final_message: null` and `error: {kind, message}`; human-readable diagnostics still go to stderr. |
| `--experimental-mode` | off | **Lab-quality** (§15). Keeps the session alive across Q/A turns via Stop-hook block-and-inject. Prints a warning on start. When possible, finalizes via the same `GAIA_DONE` / `GAIA_FAILED` / `GAIA_ABORTED` sentinels as stable runs (§15.3); on hook crashes and VM wall-clock timeouts recovery goes through `gaia recover`. Not part of the v1 stable contract. |
| `--auto-continue` | off | Experimental-mode only. Auto-reply with `GAIA_CONTINUE` when the assistant response does not appear to ask a question (heuristic in §15.5). Off by default so every turn prompts the user — a wrong auto-continue can turn a clarification into unsupervised work. |

### 6.4 Admin commands

```
gaia init [--hardened | --no-hardened]         # interactive project wizard. --hardened is the v1 **recommended posture** (§16.1) that enrolls the repo in Claude Code's OS-level sandbox: writes a `sandbox.filesystem.denyWrite` block to .claude/settings.json that denies writes to .git/, .claude/hooks/, .gaia/, .gaia-state/, and .gaia-context/ at the OS layer for every Bash subprocess. Closes the §8.5 PreToolUse Bash-parser bypass gap (eval, Python, background processes) for gaia-owned paths. Adds an OS dependency on the sandbox surface; gaia init prints which OSes the current Claude Code version supports it on. Off by default in v1 because teammate machines may lack the sandbox runtime, but `gaia init` without `--hardened` prompts the operator to confirm they want the softer posture (step 0 in the detail below); pass `--no-hardened` to skip the prompt non-interactively. v1.x will tighten the default once the sandbox surface is broadly available.
gaia init --verify-default-branch               # re-run gaia init step 9's default-branch verification + step 10's doctor offer after a prior init exited pending-default-branch-merge or -shim-update-merge; also reconciles committed shim drift against the driver's bundled files (§6.4 step 9 detail). Idempotent.
gaia init --upgrade-hooks                       # alias for --verify-default-branch when the default branch already has hook files but the shim bytes drift from the driver's bundled copies (same effect; different affordance name for repos that only need the drift reconcile).
gaia config set <key> <value>                  # secret-aware: *_TOKEN → keychain
gaia config get <key>
gaia config unset <key>
gaia history                                   # list recent runs: run_id, session_id, status, prompt. Scans .gaia/runs/*/meta.json (canonical) and folds in history.jsonl terminal events. Pending runs show as "pending (in-flight or crashed)". By default, doctor runs (`meta.run_type == "doctor"`) are filtered out; pass `--include-doctor` to surface them.
gaia history show <session_id>                 # print the transcript for a prior session
gaia recover <run_id|session_id> [--merge] [--manual-merge] [--wait <duration>] [--assume-failed]  # finish a stuck run: fetch preserved state/work, surface the sentinel (if any), offer merge or manual review. --merge opts aborted runs into a merge-back. --manual-merge leaves a re-conflicted merge in place for manual resolution (exit 17) instead of aborting. --wait sets the fire-ambiguous sentinel poll window. --assume-failed auto-answers the "classify this ambiguous run as fire-failed" prompt for scripts.
gaia gc [--max-age 14d] [--dry-run]            # sweep stale claude/gaia/* branches on origin (work/state pairs, rescue refs, doctor protocol runs identified by `manifest.run_type == "doctor"`). See §12.1 for per-ref-type policy.
gaia gc --local-runs [--max-age 30d] [--dry-run] # optional sweep of terminal .gaia/runs/<run_id>/ directories older than --max-age. Requires explicit confirmation; never touches non-terminal runs.
gaia gc --local-refs [--max-age 30d] [--dry-run] # sweep local claude/gaia/* branch refs in the driver's clone that survived non-success terminal paths; mirrors --local-runs for git refs.
gaia doctor [--cloud|--local|--full] [--live-stop-failure] [--verify-subagents]  # validate transport. Default --cloud fires a synthetic session and checks spawner, hooks, transcript schema, branch permissions, MCP-free routine, VM managed policy, routine/repo binding, project .mcp.json / plugin absence. --local requires a local `claude` and checks driver-machine hook honoring and the Node shim. --full runs both. --live-stop-failure opts the cloud tier into an end-to-end StopFailure probe against a live rate-limit / overload condition (§8.4); off by default because it depends on the upstream being in an error state. --verify-subagents runs the release-gate subagent-correlation probe for opting into `spawner_config.allow_agent` (§8.5) — fires a manifest with `run_type: "doctor"` + `verify_subagents: true` so the PreToolUse shim allows the parent Agent launch (doctor-only bootstrap bypass, §8.5), then observes that the subagent's first tool call lands on PreToolUse with the parent session_id. Must pass before the Agent opt-in takes effect.
gaia cloud-upgrade                             # bump the routine's environment setup-script pin to the currently-installed @your-scope/gaia version. In v1, prints the exact new setup-script text for the user to paste into the Claude Code environment UI (no documented public API exists for editing environment setup scripts, so automated in-place edits are out of scope for v1). The paste invalidates the Claude Code setup-script cache so the next fire installs the new binary. If Anthropic later exposes a routine/environment configuration API, a minor release may add in-place editing under a confirmation prompt.
gaia version
```

`gaia init`:
0. **Prominent `--hardened` banner (when the flag is absent).**
   Before prompting for routine URL + token, `gaia init`
   prints a recommendation banner when `--hardened` was not
   passed:
   ```
   NOTICE: gaia init is running without --hardened.

   --hardened enrolls this repo in Claude Code's OS-level
   sandbox (§16.1), denying writes to .git/, .claude/hooks/,
   .gaia/, .gaia-state/, and .gaia-context/ from every Bash
   subprocess the cloud session spawns. This closes the
   PreToolUse Bash-parser bypass gap (eval, Python,
   background processes) that the parser explicitly cannot
   contain (§8.5, §16.1).

   The recommended posture for production repos — and any
   repo CI / automation fires gaia against — is
   `gaia init --hardened`. The flag is opt-in in v1 only
   because the OS-level sandbox runtime is not uniformly
   available across every Claude Code build yet; teammate
   machines running older Claude Code may lack sandbox
   support and would see a fail-closed hook-start error. Run
   --hardened unless you have teammates on a Claude Code
   build that does not support it.

   Continue without --hardened? [y/N]
   ```
   A `y` / default-`no` prompt gives the operator a chance to
   rerun with `--hardened` rather than fall through silently
   into the softer posture. `--hardened`-not-passed + `n` /
   empty answer exits 0 with a pointer to `gaia init
   --hardened`; `y` continues to step 1 below. Scripted
   invocations that want the softer posture explicitly can
   pass `--no-hardened` to skip the prompt (v1 adds this flag
   only as an escape hatch for CI that does not have a tty;
   the documentation still recommends `--hardened` for every
   real repo).
1. Prompts for routine URL + token; stores them securely.
2. Writes `gaia.config.json` to the repo root.
3. Writes `.claude/settings.json` and `.claude/hooks/*.js` to the repo
   root (§8). If `.claude/settings.json` exists **but does not already
   define project-level `UserPromptSubmit`, `Stop`, or `StopFailure`
   handlers**, merges the gaia-managed subset (§8.8) — the four hook
   entries plus the `permissions.deny` entries — non-destructively,
   preserving foreign entries on the same top-level keys.
   **If `--hardened` was passed on the `gaia init` command line**,
   the `sandbox` block from §16.1 is emitted into the same
   `.claude/settings.json` at this step (denying writes to
   `./.git`, `./.claude/hooks`, `./.gaia`, `./.gaia-state`, and
   `./.gaia-context` for every Bash subprocess). The hardened
   settings MUST land in the committed file before default-branch
   verification in step 9 — otherwise step 9 would verify a tree
   that does not include the hardened block, and the repo would
   reach `status: complete` while its committed
   `.claude/settings.json` omits the sandbox. `gaia init` also
   prints a banner naming the OS / Claude Code version
   requirements so the operator knows teammates without sandbox
   support will see hook-start failures; this matches the §16.1
   posture (opt-in in v1, opt-out tracked for v1.x).
   If `.claude/settings.json` does define any of the three
   exclusive events, `gaia init` aborts with the error documented
   in §8.7 (hook coexistence) and offers migration guidance.
   Notes that managed-policy or plugin-scope handlers for the
   same events are outside `gaia init`'s visibility (§8.7) and,
   if present, will still run alongside gaia's.
   Also enumerates every project hook registered on every event
   outside gaia's four (`SessionStart`, `PostToolUse`,
   `SubagentStop`, `Notification`, `ConfigChange`, `WorktreeCreate`,
   etc.) and prints a `gaia does not control these project hooks`
   block listing each event, matcher, and command. Hooks on
   those events are **not** refused — they coexist freely with
   gaia — but the list makes their presence explicit so the
   operator knows what will run alongside gaia's protocol.
   `SessionStart` gets a louder warning because it runs before
   `UserPromptSubmit` and can mutate the worktree or env in
   ways gaia's invariants assume do not happen. The banner
   recommends running `gaia doctor --cloud` to confirm the
   composition still produces clean sentinels.

   **`TaskCreated` / `TaskCompleted` are refused, not warned.**
   These events can spawn agent-shaped work outside the
   `PreToolUse` shim's visibility (§8.4 check 23), which is the
   same class of protocol-bypass risk that §2 treats as a hard
   refusal for `.claude/skills/**`, `.claude/commands/**`, and
   `.claude/agents/**`. `gaia init` therefore refuses setup when
   the effective `.claude/settings.json` registers a project-
   scoped `TaskCreated` or `TaskCompleted` handler — the init
   path and the doctor release gate are consistent on this
   surface. Operators who need those events for a non-gaia
   workflow should move the handler out of project scope
   (managed-policy / plugin / user settings) or remove it.
   `gaia doctor --cloud` check 23 stays a release gate for the
   cloud-side composition (managed-policy or plugin handlers
   gaia init cannot see); failing either the init refusal or
   the doctor check means gaia will not run safely in the
   repo until the handler is gone.
   Refuses outright if `.mcp.json` is present at any load
   path the cloud session will see, if **any file exists
   under `.claude/skills/**`, `.claude/commands/**`, or
   `.claude/agents/**`** (§2 — v1 refuses on presence; the
   v1.x structural allow-list in §14 is the path forward for
   repos that want these surfaces), or if
   `.claude/settings.json` defines plugin entries (§2, §5).
   These are hard blockers, not warnings.
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
   It additionally **reconciles committed shim / settings
   drift against the driver's bundled files** regardless of
   whether `protocol_version` changed: gaia patch and minor
   releases under the same `protocol_version` may still ship
   shim fixes (bug fixes in the classifier, Bash-decoder
   updates, doctor-check additions), and `gaia cloud-upgrade`
   (§6.4) only refreshes the VM's global binary — it does
   **not** touch the committed `.claude/hooks/gaia-*.js` /
   `.claude/settings.json` / `.gitignore` entries. Without a
   reconcile path, a repo running gaia v1.0.0 shims against a
   v1.0.3 global binary would carry any shim bug the patch
   releases fixed. `gaia init --verify-default-branch`
   therefore:
   - **For `.claude/hooks/gaia-*.js`:** computes the sha256 of
     each gaia-owned hook file in the driver's bundled copy and
     on the default branch's tree. Whole-file hashing is correct
     here — gaia owns these files outright, and foreign edits to
     them are not a supported composition.
   - **For `.claude/settings.json`:** computes a **normalized
     hash of only the gaia-managed subset** of the settings
     file, not the whole file. The managed subset is defined in
     §8.8 and covers:
     - `hooks.UserPromptSubmit` gaia entry;
     - `hooks.PreToolUse` gaia matcher entry;
     - `hooks.Stop` gaia entry;
     - `hooks.StopFailure` gaia entry;
     - the gaia-namespace `permissions.deny` rules
       (`/.gaia/**`, `/.gaia-state/**`, `/.gaia-context/**`);
     - `disableSkillShellExecution: true`;
     - the `sandbox` block (only when the repo is committed
       `--hardened` per §16.1).
     Foreign keys (other handlers, custom permissions, model
     config, IDE preferences, etc.) are excluded from the
     normalized hash by construction, so legitimate foreign
     edits never false-positive against the bundled template.
     Comparing the whole file would misfire on every repo that
     has any non-gaia settings entry, which is the overwhelming
     majority of production repos and the exact case the v1
     composition policy is designed to support (§8.7, §8.8).
   - **For `.gitignore`:** computes the normalized hash of the
     gaia-owned ignore subset (the fixed `.gaia/` and
     `.gaia-context/` lines, one per line, in that order, with
     a trailing newline — the same shape `manifest.managed_gitignore_hash`
     uses in §7.4 and §8.2 step 6).
   - If every hash matches, exits 0 with `status: complete`.
   - If any hash differs, on the current branch path it
     proposes an update commit `gaia: update managed hook
     shims to driver vX.Y.Z` (listing the changed files) and
     opens a PR against the default branch (with `gh` if
     available, or prints the git commands otherwise). The
     reconcile commit for `.claude/settings.json` edits only
     the managed subset — foreign keys in the committed
     settings file are preserved non-destructively, mirroring
     the init-time merge from step 3. The command exits with
     `status: pending-shim-update-merge` in that case and
     prints the PR URL and the next-step re-run command,
     mirroring the `pending-default-branch-merge` UX from
     step 9.
   This closes the same-protocol shim-drift loophole without
   forcing a `protocol_version` bump for every shim fix and
   without false-positives on foreign settings. `gaia doctor`
   (§8.4 / §8.8) applies the same normalized hashing and prints
   a `managed file drift` advisory when the gaia subset
   differs, pointing at `gaia init --verify-default-branch` as
   the remediation.

   Note on the Stop-hook integrity check (§8.2 step 6): that
   check uses the **whole-file** hash recorded in
   `manifest.managed_file_hashes` for `.claude/settings.json`,
   because at Stop time the comparison is against the
   work-branch tree the manifest was derived from — both sides
   have identical foreign-settings content by construction, so
   whole-file equality is the correct gate there. The
   normalized-subset hash is specific to `gaia init
   --verify-default-branch`'s cross-default-branch comparison
   against the driver's bundled template.
10. Offers to run `gaia doctor --cloud` before exit (the
    required tier; §8.4). Skipped on the
    `pending-default-branch-merge` exit path. `--full` can be
    requested at the prompt to add local-tier checks when a local
    `claude` is available.
11. Prints a one-line summary: `status` (`complete` /
    `pending-default-branch-merge` / `pending-shim-update-merge`),
    whether `--hardened` was applied, and the next step (usually
    the paste instructions or the follow-up `gaia init
    --verify-default-branch` invocation). The hardened settings
    themselves were already written in step 3; this step only
    reports that the flag took effect.

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
invocation's process to still be alive. The per-status branch
retention, session-existence claims, and recover-action selection
below are **derived from the §6.1.2 normative status transition
table**; if a row here and the table disagree, the table wins.
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
  - **`"push-rejected"`** (§6.1.1 step 7 failed after the local
    merge landed): the local user branch should already contain the
    recorded `merge_sha`. `gaia recover` verifies this with
    `git merge-base --is-ancestor <merge_sha> <user_branch>`. If
    true, it fetches
    `$UPSTREAM_REMOTE/$UPSTREAM_BRANCH` using the
    `upstream_remote` / `upstream_branch` recorded in
    `meta.json` (not a hardcoded `origin`) via an explicit
    `+refs/heads/$UPSTREAM_BRANCH:refs/remotes/$UPSTREAM_REMOTE/$UPSTREAM_BRANCH`
    refspec, then inspects upstream's current tip:
    - **Upstream already contains `merge_sha`** — the caller
      (or someone else) pushed manually between the failed
      run and the recover call. `gaia recover` skips the
      retry, promotes `meta.status = "success"` with
      `pushed_upstream: true`, appends the terminal
      `"success"` history event, and exits 0 without
      force-pushing or re-merging. Without this short-circuit,
      the retry would look like a no-op push (or fail as
      non-fast-forward against a superset upstream) and leave
      the run stuck in `push-rejected` forever despite the
      work being live.
    - **Upstream is ancestor of the local user branch (fast-
      forward path still open)** — retry
      `git push $UPSTREAM_REMOTE "refs/heads/$USER_BRANCH:refs/heads/$UPSTREAM_BRANCH"`
      using explicit full refs on both sides of the colon so
      slash-heavy branch names (`feature/auth`,
      `claude/gaia/*`) cannot resolve ambiguously to a tag.
    - **Upstream has diverged** — stop and instruct the user
      to rebase/merge manually, printing both `merge_sha`
      and the current upstream tip.
    If the local branch no longer contains `merge_sha` (e.g.,
    the user reset locally), `gaia recover` refuses and prints
    the work-branch name so the caller can decide whether to
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

    **`conflict` recovery UX.** A blind retry of the same
    `git merge --no-ff` after a `"conflict"` status is almost
    guaranteed to re-conflict on the same paths. `gaia recover
    <id>` on a conflict status therefore supports two flows:
    - **Default (no flag).** Recover attempts the transactional
      merge from §6.1.1. If it conflicts again, abort the merge,
      leave the user branch untouched, and print the work branch
      name plus a pointer at `--manual-merge` for the caller to
      opt into a hands-on flow. This preserves the existing
      behavior for callers who expect recover to refuse to
      clobber state.
    - **`gaia recover <id> --manual-merge`.** Fetches the work
      branch into the local repo, acquires `.gaia/merge.lock`
      (§6.1.1), checks out the user branch, runs
      `git merge --no-ff refs/remotes/origin/$GAIA_WORK_BRANCH`,
      and — on conflict — **leaves the merge conflict in place**
      for the user to resolve manually (does not abort).
      Releases the merge lock and exits with a distinct code
      (exit 17, `"manual-merge-in-progress"`) so scripts can
      distinguish "recover did nothing because it conflicted
      again" from "recover left you in a half-merged state."
      `meta.status` stays `"conflict"` until the user completes
      the merge and re-runs `gaia recover <id>` (which then sees
      the merge commit already on HEAD and promotes the status
      to `"success"`).
  - **`"aborted"`** (§15, user `:abort`): prints the work-branch
    name and the partial transcript. Merging is **not offered by
    default** — abort means "do not land this." The user can pass
    `gaia recover <id> --merge` to opt in to a merge-back if they
    change their mind; otherwise `gaia recover` exits leaving the
    status as `"aborted"` and the work branch preserved.
  - **`"failed"`**: prints the `session_url`, the error body from
    `.gaia-state/<session_id>.error.json`, and the work-branch
    name for manual review. Offers **`gaia recover <run_id>
    --merge`** as the explicit opt-in that lands whatever the
    agent did commit before the failure and promotes the run to
    `"success"`. After promotion, `gaia -c` / `gaia --resume
    <session_id>` resumes normally. Direct `-c` against a
    `failed` run is deliberately refused (§6.2's success-only
    rule); without the `--merge` promotion the transcript
    captured pre-failure is preserved locally and can still be
    read via `gaia history show <session_id>`, but it is not a
    valid continuation source in v1.
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

    **`meta.session_id` is populated from
    `.gaia-state/<session_id>.meta.json`** on fire-ambiguous
    promotion. The Stop / StopFailure hook writes that
    artifact alongside every terminal sentinel (§7.3, §8.2
    step 10, §8.3), carrying
    `{session_id, session_url: null, issued_at,
    finalized_at}`. The hook reads `session_id` from the
    documented `CLAUDE_CODE_REMOTE_SESSION_ID` environment
    variable (present in Claude Code's current cloud-session
    builds) and writes `session_url: null` unconditionally
    — gaia does not synthesize a URL from `session_id`
    because no current Claude Code env var exposes the URL
    directly, and the driver's documented way of building one
    (`https://claude.ai/code/<session_id>`) is a convention
    that may drift. `meta.session_url` is therefore populated
    **only on the normal fire path**, where the `/fire`
    response carries `claude_code_session_url` verbatim (§10.3).
    On the fire-ambiguous recovery path, `meta.session_url`
    stays `null` even after a late sentinel arrives — the
    URL is informational (for debugging in the Claude Code UI),
    not load-bearing on recovery, so the loss is acceptable.
    If `CLAUDE_CODE_REMOTE_SESSION_ID` itself is absent on a
    given Claude Code build, the hook falls back to the
    stdin `session_id` field Claude Code passes to every hook,
    which is the authoritative value for the protocol
    regardless. If the sentinel commit predates v1 and lacks
    the meta artifact entirely, recovery falls back to both
    `session_id` extracted from the sentinel commit message
    and `session_url: null`.

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
    the lock-guarded flip to `pending` that follows step 5a's
    push): `gaia recover` **probes origin first** via
    `git ls-remote origin "refs/heads/claude/gaia/$run_id/*"`
    because the `preparing` status alone does not disambiguate
    pre-push crashes, partial pushes, and post-push-pre-status-
    flip crashes. Three observed ref-sets map to three
    recovery actions:
    - **No gaia branches on origin** → transition `meta.status =
      "fire-failed"` with `error: "driver-crashed-before-push"`,
      append the terminal history event. **Clean up local
      refs:** delete any `refs/heads/claude/gaia/<run_id>/work`,
      `refs/heads/claude/gaia/<run_id>/state`, or
      `refs/gaia/recovered/<run_id>/*` refs that the crashed
      driver may have created in the local clone (§6.1 step 4
      creates the local work branch before pushing; a crash
      between that step and the push leaves the local ref
      behind). `best-effort` (swallow errors); the origin
      cleanup is implicit because no origin refs exist.
    - **Only `claude/gaia/<run_id>/work` on origin (state
      branch missing)** → the work push succeeded but the
      state push did not (or the driver crashed in between).
      No manifest reached origin, so no session could have
      been created. Delete the orphan origin work ref, delete
      any matching local refs under
      `refs/heads/claude/gaia/<run_id>/*` and
      `refs/gaia/recovered/<run_id>/*`, transition to
      `"fire-failed"` with
      `error: "driver-crashed-after-work-push-before-state-push"`,
      append the terminal history event.
    - **Both `claude/gaia/<run_id>/{work,state}` on origin** →
      the driver crashed between §6.1 step 5a's pushes and the
      lock-guarded flip to `pending`. No session could have
      been created (the flip is sequenced before step 6's
      HTTP call). Promote `meta.status = "pending"` with
      `branches_pushed: true, fire_attempted: false` under
      the per-run lock, then follow the matching `pending`
      recovery action below (delete both origin branches,
      delete matching local refs, transition to
      `"fire-failed"` with
      `error: "driver-crashed-after-push-before-fire"`, append
      terminal event). The three `preparing` rows in the
      §6.1.2 status transition table spell each branch out
      normatively.
    **The local `.gaia/runs/<run_id>/` directory is
    preserved** — §10.3 makes `meta.json` canonical and allows
    rebuilding `history.jsonl` from the runs directory, so
    deleting the directory on recovery would break that
    invariant and leave the terminal history event referencing
    a run whose metadata has vanished. Trimming old terminal
    run directories is the job of a separate `gaia gc --local-
    runs [--max-age <d>]` sweep that operates on `.gaia/runs/*`
    after explicit confirmation; that sweep is v1 admin-command
    scope (§6.4) and unrelated to recovery finalization.
  - **`"pending"`** branches on the substate trio
    (`branches_pushed`, `fire_attempted`, `request_body_started`)
    from §10.3:
    - `branches_pushed: true, fire_attempted: false` **or**
      `fire_attempted: true, request_body_started: false` — the
      driver crashed after pushing the branches but before any
      request-body byte reached the socket, so no session could
      exist. `gaia recover` deletes the preserved branches from
      origin, transitions `meta.status = "fire-failed"` with
      `error: "driver-crashed-after-push-before-fire"` (or
      `"...-before-body"` respectively), and appends the
      terminal history event. No state-branch polling — the
      sentinel will never land.
    - `request_body_started: true` with `session_id: null` — the
      HTTP request had begun but the driver never parsed a
      response. Treated as `fire-ambiguous` for recovery
      purposes: poll the state branch for a late sentinel
      (bounded, default ~5 min; `--wait <duration>` overrides),
      promoting to the corresponding terminal state on
      arrival. Absent a sentinel, offers `--assume-failed` or a
      fresh retry with the ambiguous branches preserved.
    - `session_id` populated — the normal mid-poll crash case.
      Prints the `session_url`, polls the state branch once for
      a late sentinel, and treats a subsequent `GAIA_DONE` /
      `GAIA_FAILED` / `GAIA_ABORTED` arrival like the
      corresponding terminal-state path above.

**Status transition table — normative.** The table below is the
**single normative source of truth** for status transitions,
work / state branch retention on origin, session existence,
`session_id` knowability, recover actions, and terminality.
Every other section that describes state behavior — §6.1's
fire flow, §6.1.1's merge-back, §6.1.2's `--no-merge`, §6.4's
`gaia recover` rows, §10.3's substate descriptions, §12's
failure-mode table, §12.1's cleanup exceptions — is **derived
narrative** that elaborates the rationale for individual
rows. If a narrative paragraph and this table disagree, the
**table wins**, and the disagreement is an erratum against the
narrative. Implementations MUST code against this table (and
the row-level test corpus referenced below) rather than against
the prose elsewhere; in particular, branch-retention decisions,
`gaia recover` action selection, and the `--no-merge` /
`fire-failed` / `fire-ambiguous` distinctions are all read from
this table.

The accompanying **row-level test corpus** (one test per row,
exercising the substate transitions, the recover action, and
the branch-retention assertion) is the executable expression
of this table. Implementers MUST keep the test corpus and the
table in lockstep — adding a new status or substate requires
adding both a row and a test in the same change. Drift between
rows is the documented failure mode the table-as-source-of-
truth pattern is intended to prevent; drift between the table
and the test corpus is the equivalent failure mode at the
implementation layer.

| Status (+ substates) | Work branch on origin | State branch on origin | Session may exist? | `session_id` known? | `gaia recover` action | Terminal? |
|---|---|---|---|---|---|---|
| `preparing` + no gaia branches on origin | no | no | no | no | preserve the local `.gaia/runs/<run_id>/` directory (§10.3 keeps `meta.json` canonical so `history.jsonl` can be rebuilt from it); transition to `fire-failed` with `error: "driver-crashed-before-push"`; append the terminal history event. Trimming the local run dir later is `gaia gc --local-runs`'s job, not recovery's. | no |
| `preparing` + only work branch on origin (partial push: work-push succeeded, state-push did not land) | yes (orphan) | no | no (manifest never reached origin, so the `UserPromptSubmit` hook would have blocked any fire that did happen) | no | delete the orphan `claude/gaia/<run_id>/work` ref from origin; transition to `fire-failed` with `error: "driver-crashed-after-work-push-before-state-push"`; append the terminal history event. | no |
| `preparing` + both gaia branches observed on origin (crash between §6.1 step 5a's pushes and the lock-guarded status flip to `pending`) | yes | yes (manifest only, ± overflow `context.md`) | no (status flip was the fsync that would have preceded `/fire`; crash before the flip means `fire_attempted` also remained `false`, and no session could have been created) | no | probe `git ls-remote origin "refs/heads/claude/gaia/$run_id/*"` first (this is the normative detection step — `recover(preparing)` always probes origin before applying any branch of this row); with both refs present, promote the local `meta.status` to `pending` with `branches_pushed: true, fire_attempted: false` under the per-run lock, then apply the matching `pending` recovery row below. A partial-push observation dispatches to the partial-push row above instead, and a no-refs observation dispatches to the first `preparing` row. | no |
| `pending` + `branches_pushed: true`, `fire_attempted: false` | yes | yes (manifest only, ± overflow `context.md`) | no | no | delete origin branches; transition to `fire-failed`. | no |
| `pending` + `fire_attempted: true`, `request_body_started: false` | yes | yes | no (no `request.write()` / `request.end()` call had been attempted; nothing could have been sent) | no | delete origin branches; transition to `fire-failed`. | no |
| `pending` + `request_body_started: true`, `session_id: null` | yes | yes | unknown (ambiguous) | no | poll state branch for late sentinel; promote on arrival; otherwise offer `--assume-failed` or fresh retry. | no |
| `pending` + `session_id` populated | yes | yes | yes | yes | poll state branch once; promote on arrival (same as the inline poll loop). | no |
| `success` | deleted unless `--keep-branches` (delete-failure policy in §6.1 step 12's branch-cleanup substep) | deleted unless `--keep-branches` | n/a | yes | none; prints recorded `merge_sha`. | yes |
| `no-merge` | yes (preserved by `--no-merge`) | yes | yes (completed) | yes | fetch work branch; complete merge-back; promote to `success`. | yes |
| `timeout` | yes (preserved) | yes (may lack sentinel) | yes (may or may not have finalized) | usually yes | poll state branch; promote if a late sentinel lands. | yes |
| `conflict` | yes (preserved) | yes (has `GAIA_DONE` sentinel) | yes (completed) | yes | retry merge-back; on re-conflict, default aborts, `--manual-merge` leaves the conflict in place (exit 17). | yes |
| `push-rejected` | yes (preserved — §6.1.1 step 8 only releases the lock, and §6.1 step 12's branch-cleanup substep deletes only on `success`) | yes (preserved) | yes (completed) | yes | retry `git push $UPSTREAM_REMOTE` using recorded `meta.upstream_*`; no re-merge. | yes |
| `failed` | yes (preserved) | yes (has `GAIA_FAILED:`) | yes (but errored out) | yes | prints error body + session URL; offers `-c` via transcript replay. | yes |
| `aborted` | yes (preserved) | yes (has `GAIA_ABORTED:`) | yes (experimental abort) | yes | prints partial transcript; `--merge` opts into a merge-back that promotes to `success`. | yes |
| `fire-failed` | deleted (or never existed) | deleted (or never existed) | no | no | prints recorded error body; no recoverable action. | yes |
| `fire-ambiguous` | yes (preserved) | yes (manifest-only; may receive a late sentinel) | unknown | no | poll state branch for late sentinel; on arrival, promote to the corresponding terminal state (`success` / `failed` / `aborted`) and populate `session_id` from the sentinel's commit message; absent a sentinel, offer `--assume-failed` or fresh retry. | yes |

**Transition rules.**
- `preparing → pending` fires only after both work and state branches
  are on origin (§6.1 step 5a). The status write is under the
  per-run lock and happens *after* the pushes; a crash between the
  pushes and the status write leaves `status: "preparing"` with the
  branches on origin — this is exactly the second `preparing` row in
  the status table above. `recover(preparing)` therefore **always
  probes origin first** via `git ls-remote origin
  "refs/heads/claude/gaia/$run_id/*"`; if gaia branches are
  observed, recovery promotes `meta.status` to `pending` with
  `branches_pushed: true, fire_attempted: false` under the per-run
  lock and then applies the matching `pending` recovery row. If no
  branches are observed, recovery follows the no-gaia-branches
  `preparing` row (`fire-failed` with `error: "driver-crashed-
  before-push"`). The promotion is the recovery action, not a new
  fire-path transition — the fire-path driver never writes
  `pending` without performing the flip itself; only recovery is
  allowed to reconcile a stale `preparing` against observed origin
  state. Earlier prose described `preparing → pending` as a
  generic "crash window" without naming which code path performs
  the reconciliation; that prose is superseded by this table row
  and this rule.
- `pending → success | no-merge | failed | timeout | conflict |
  push-rejected | aborted | fire-failed | fire-ambiguous` is
  always monotonic-forward in the fire path. A status never
  reverts to `preparing` or `pending` once it has reached any
  terminal value.
- **"Terminal" means terminal for the original driver
  invocation, not immutable.** A run's `meta.status` can still
  change after the original driver exits — `gaia recover` is
  the **only** actor allowed to drive these terminal-to-terminal
  promotions, and each is named explicitly below. Implementers
  treating "terminal" as "`meta.status` is now frozen" will
  miss these legitimate transitions and leave runs stuck in
  recoverable states.
  - `no-merge → success` when `gaia recover <run_id>` completes
    the deferred merge-back.
  - `conflict → success` when `gaia recover <run_id>` (default
    or `--manual-merge`) resolves the merge and reruns the
    transactional merge-back.
  - `push-rejected → success` when `gaia recover <run_id>`
    retries the upstream push after the caller integrated the
    upstream advance.
  - `timeout → success | failed | aborted` when `gaia recover
    <run_id>` observes a late terminal sentinel and promotes
    per the sentinel kind.
  - `aborted → success` when `gaia recover <run_id> --merge`
    opts the aborted run into a merge-back.
  - `failed → success` when `gaia recover <run_id> --merge`
    opts the failed run into a merge-back (the explicit
    opt-in exists because `failed` means the agent errored,
    and landing partial work on the user branch is not safe
    by default).
  - `fire-ambiguous → success | failed | aborted` when
    `gaia recover <run_id>` observes a late sentinel on the
    preserved state branch.
  - `fire-ambiguous → fire-failed` when `gaia recover
    <run_id> --assume-failed` (or an interactive confirmation)
    classifies an ambiguous run as unrecoverable after the
    `--wait` window expires.
  `fire-failed` and `success` are truly immutable — no
  recovery action promotes out of either.
- `history.jsonl` append happens at each terminal transition —
  once per run per transition (§10.3). A run that moves from
  `conflict` to `success` via `gaia recover` appends a new
  terminal event with status `success`; the older `conflict`
  event is retained, and readers treat the latest record for a
  `run_id` as authoritative.
- Every row preserves work/state branches on origin for **every
  non-`success` terminal state**, because recovery can still act
  on them. Cleanup happens only on clean `success` (or via `gaia
  gc`).

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

origin/claude/gaia/$RUN_ID/state    ○──○                            (orphan, never merges)
                                    │  └── GAIA_DONE:   <session_id>    (or GAIA_FAILED: or GAIA_ABORTED:)
                                    │         adds:
                                    │           .gaia-state/<session_id>.md
                                    │           .gaia-state/<session_id>.transcript.md
                                    │           .gaia-state/<session_id>.meta.json
                                    └── orphan root commit (manifest commit — exactly one)
                                          adds:
                                            .gaia-state/manifest.json
                                            .gaia-state/context.md   (only on overflow; §6.1 step 5)
```

The **orphan root commit and the manifest commit are the same
commit** — the root of the state branch carries the manifest
(plus `context.md` when the payload overflows) and no earlier
commit exists. The terminal-sentinel commit is the root's only
child. An experimental-mode session adds `GAIA_PENDING:` /
`GAIA_INPUT:` commits between the root and the terminal
sentinel (§15.2), but the stable-mode topology is always a
2-commit line.

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

**Deterministic selection when multiple control blocks appear.**
A `-c` / `--resume` payload embeds a prior conversation transcript
in a `--- BEGIN CONTEXT --- / PREVIOUS CONVERSATION: …` block
(see §6.2). Prior user turns in that transcript can themselves
contain old `--- BEGIN GAIA CONTROL ---` … `--- END GAIA CONTROL ---`
fences — a teammate's `gaia -c` from two runs ago, a quoted
prompt in the conversation, a model output that echoed the
fence, or (on a subsequent continuation) the previous run's own
control block. If the `UserPromptSubmit` hook picked any of those
old fences it would fetch the wrong run's state branch and block
the prompt incorrectly (or, on rare `--keep-branches` /
`--no-merge` preservation paths, validate against a stale
manifest).

The hook MUST use the **current run's** control block
deterministically, regardless of how many stale fences the
prompt contains. The selection rule is structural: the hook
parses the submitted prompt for the **last** well-formed
`--- BEGIN GAIA CONTROL ---` … `--- END GAIA CONTROL ---` fence
that appears **before** the first `--- BEGIN CONTEXT ---` line.
The gaia payload format (§7.2) always places exactly one
control block immediately above the `--- BEGIN CONTEXT ---`
block, so the last-before-context fence is unambiguously the
current run's; any older fences embedded in the transcript live
*inside* the `--- BEGIN CONTEXT ---` block and are skipped by
construction. If no `--- BEGIN CONTEXT ---` marker is present
(unusual — gaia always wraps the task in the context block),
the hook falls back to selecting the last well-formed control
block in the prompt; a gaia-issued payload always has at least
one. The hook also rejects (blocks with a clear reason) any
prompt where the control block immediately before
`--- BEGIN CONTEXT ---` fails to parse, rather than silently
searching further back; this guarantees a broken current-run
header does not hide behind an old intact header.

Belt-and-suspenders: the driver also **strips** any
`--- BEGIN GAIA CONTROL ---` … `--- END GAIA CONTROL ---` fences
from the transcript body before saving it to
`.gaia/transcripts/<session_id>.md` (§8.2 step 9) and before
replaying it in a `PREVIOUS CONVERSATION:` block (§6.2 step 3),
so a fresh gaia run in the wild never has to rely on the
selection rule above to tolerate stale embedded fences. The
selection rule remains the load-bearing correctness property
because an older gaia version that did not strip may have
already written transcripts into `.gaia/transcripts/` that
contain embedded fences; stripping at replay time catches
those legacy transcripts as well.

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
(65,536 for the Claude routines spawner), the driver writes
`.gaia-state/context.md` into the **same local state-branch commit
that carries the manifest** (§6.1 step 5) — not a separate commit
after the state branch was already pushed. That single commit is
then pushed to origin in §6.1 step 5a, so overflow runs and
non-overflow runs both produce exactly one state-branch push before
`/fire`. The `context.md` artifact is a `overflow_context`-typed
Markdown file (see the §7.3 schema table) with `session_id: null`
because the session id is not known until after `/fire` returns.
The payload's overflowed content is replaced with:

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

**Artifact header format.** Every `.gaia-state/*` artifact gaia
writes (including the manifest, the final-message file, the
readable transcript, the error body, the session metadata, the
rescue artifacts, and the experimental-mode `pending_<n>.md` /
`input_<n>.md` files) carries the same bookkeeping header. The
serialization depends on the artifact's content type:

- **Markdown artifacts** (`<id>.md`, `<id>.transcript.md`, the
  experimental-mode `pending_<n>.md` / `input_<n>.md`, and the
  text-mode rescue files `<id>.rescue.head.txt`,
  `<id>.rescue.log.txt`, `<id>.rescue.patch`) begin with a
  YAML front-matter block delimited by `---` on its own line
  at the start of the file, followed by a blank line and then
  the body:

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

- **JSON artifacts** (`manifest.json`, `<id>.error.json`, and
  the v1 `<id>.meta.json` session-metadata artifact — §7.4)
  carry the same metadata as **top-level JSON keys**, not as
  YAML front matter. A JSON file with a leading `---\n` YAML
  block would no longer be valid JSON, so producers write
  `created_by`, `schema_version`, `artifact_type`, `session_id`,
  and `run_id` as top-level keys alongside the artifact's own
  payload keys. The manifest example in §7.4 shows this
  concretely.
- **Binary artifacts** (`<id>.rescue.bundle` — a raw git bundle
  with a binary signature) carry **no header**. Recovery
  identifies them structurally via `git bundle verify`. The
  sibling `.rescue.*` text files carry enough metadata
  (artifact_type, session_id, run_id) for the bundle to be
  associated with its run.

`artifact_type` is one of: `manifest`, `final_message`,
`transcript`, `error`, `session_meta`, `overflow_context`,
`pending`, `input`, `rescue_patch`, `rescue_bundle`,
`rescue_head`, `rescue_log`, `rescue_base`. Producers always
emit the header (in the serialization appropriate to the
content type); readers tolerate a missing header (the v1
release is the first one writing it) and fall back to
treating the whole file as the body / value.

**Canonical per-type schema table (v1).** The table below names
every artifact gaia writes, the path it lands at, the
serialization family (JSON / Markdown / binary), and the keys
or front-matter fields that MUST appear. Examples elsewhere in
this spec conform to this table; any apparent drift between an
example and this table is an errata against the example.

| Artifact | Path | Serialization | Required keys / front matter |
|---|---|---|---|
| `manifest` | `.gaia-state/manifest.json` (state branch) | JSON top-level keys | `created_by`, `schema_version`, `artifact_type: "manifest"`, `run_id`, `run_type` (`"user"` for normal runs, `"doctor"` for `gaia doctor` synthetic runs — used by `gaia gc` to apply the §12.1 aggressive doctor-age policy and by `gaia history` to filter doctor runs out of the default listing), `nonce`, `work_branch`, `state_branch`, `base_sha`, `user_branch`, `upstream_remote`, `upstream_branch`, `upstream_ref`, `local_only_branch`, `driver_version`, `protocol_version`, `issued_at`, `managed_file_hashes`, `managed_gitignore_hash` |
| `session_meta` | `.gaia-state/<session_id>.meta.json` (state branch) | JSON top-level keys | `created_by`, `schema_version`, `artifact_type: "session_meta"`, `run_id`, `session_id`, `session_url` (string or null), `issued_at`, `finalized_at` |
| `error` | `.gaia-state/<session_id>.error.json` (state branch) | JSON top-level keys | `created_by`, `schema_version`, `artifact_type: "error"`, `run_id`, `session_id`, `error`, `error_details` (optional object), `last_assistant_message` (optional string) |
| `final_message` | `.gaia-state/<session_id>.md` (state branch) | Markdown w/ YAML front matter | `created_by`, `schema_version`, `artifact_type: "final_message"`, `run_id`, `session_id` |
| `transcript` | `.gaia-state/<session_id>.transcript.md` (state branch) | Markdown w/ YAML front matter | `created_by`, `schema_version`, `artifact_type: "transcript"`, `run_id`, `session_id` |
| `overflow_context` | `.gaia-state/context.md` (state branch) | Markdown w/ YAML front matter | `created_by`, `schema_version`, `artifact_type: "overflow_context"`, `run_id`. **`session_id` is `null`** because this artifact is written by the driver before `/fire` returns a `session_id` (§7.2). Readers that need a `session_id` use `null` rather than erroring. |
| `pending` (experimental) | `.gaia-state/<session_id>/pending_<n>.md` (state branch) | Markdown w/ YAML front matter | `created_by`, `schema_version`, `artifact_type: "pending"`, `run_id`, `session_id`, `turn` (number) |
| `input` (experimental) | `.gaia-state/<session_id>/input_<n>.md` (state branch) | Markdown w/ YAML front matter | `created_by`, `schema_version`, `artifact_type: "input"`, `run_id`, `session_id`, `turn` (number) |
| `rescue_head` | `.gaia-state/<session_id>.rescue.head.txt` (state branch) | Text w/ YAML front matter above a single-line sha body | `created_by`, `schema_version`, `artifact_type: "rescue_head"`, `run_id`, `session_id`. Body is the 40-char work-branch HEAD sha at Stop time, followed by a newline. Consumers strip the YAML block before `git rev-parse` reads the sha. |
| `rescue_base` | `.gaia-state/<session_id>.rescue.base.txt` (state branch) | Text w/ YAML front matter above a single-line sha body | `created_by`, `schema_version`, `artifact_type: "rescue_base"`, `run_id`, `session_id`. Body is the 40-char commit sha the patch / bundle was generated against (the `$base` variable in §8.2's rescue block), followed by a newline. This is authoritative for recovery: the patch must be applied on top of this sha, not on `manifest.base_sha` — the two can differ when the work branch had already been advanced by prior in-session pushes. |
| `rescue_patch` | `.gaia-state/<session_id>.rescue.patch` (state branch) | Text w/ YAML front matter above a unified-diff patch body (`git diff --binary $base..HEAD`) | `created_by`, `schema_version`, `artifact_type: "rescue_patch"`, `run_id`, `session_id`. Consumers strip the YAML block before piping the patch body into `git apply`; a leading `---\n` YAML start would otherwise confuse the patch parser. |
| `rescue_log` | `.gaia-state/<session_id>.rescue.log.txt` (state branch) | Text w/ YAML front matter above `git log --oneline` body | `created_by`, `schema_version`, `artifact_type: "rescue_log"`, `run_id`, `session_id` |
| `rescue_bundle` | `.gaia-state/<session_id>.rescue.bundle` (state branch) | Raw git bundle (binary; no header) | none — identified structurally via `git bundle verify`; associated with a run via the sibling `rescue_head` / `rescue_base` artifacts |

**Note on `*.error.json`.** Earlier drafts described `*.error.json`
as "begins with a small header" — that phrasing is an erratum
against the JSON-top-level-keys rule. The error artifact carries
`created_by`, `schema_version`, `artifact_type: "error"`, `run_id`,
and `session_id` as top-level keys alongside `error` /
`error_details` / `last_assistant_message`. The Stop / StopFailure
hook examples in §8.2 / §8.3 now conform to this shape.

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

**Sentinel categories (normative).** Terminal sentinel commits
fall into two explicit categories; the driver's per-kind
artifact expectations in §6.1 step 7 dispatch on which category
produced the sentinel.

1. **Well-formed v1 terminal sentinel** (the default). Produced
   by the §8.2 / §8.3 happy path. Required components:
   - a terminal commit message (`GAIA_DONE: <session_id>`,
     `GAIA_FAILED: <session_id>`, or `GAIA_ABORTED:
     <session_id>`);
   - the kind-specific primary artifact — `<session_id>.md` on
     done/aborted, `<session_id>.error.json` on failed;
   - `.gaia-state/<session_id>.meta.json` carrying
     `{created_by, schema_version, artifact_type, run_id,
     session_id, session_url, issued_at, finalized_at}`;
   - `.gaia-state/<session_id>.transcript.md` unless explicitly
     documented as impossible for the kind.
2. **Minimal emergency sentinel** (exceptional). Produced only
   by the documented fail-closed paths: the `UserPromptSubmit`
   shim fail-closed sentinel (§8.0 step 6 — binary missing,
   protocol-version mismatch), an early-failure `Stop` /
   `StopFailure` path that could not assemble a transcript or
   meta artifact, and §8.2 step 3's wrong-branch
   `GAIA_FAILED:` fallback when the hook cannot reach the work
   tree. Required components:
   - a terminal commit message;
   - the kind-specific error-body JSON (for `GAIA_FAILED:`) or
     a minimal `<session_id>.md` placeholder (for `GAIA_DONE:`
     / `GAIA_ABORTED:`) naming the failure so `gaia history
     show` has something to display.
   Transcript and `meta.json` MAY be omitted. The driver
   identifies the category by the well-formed components'
   absence, records `meta.error_details.minimal_sentinel =
   true`, and finalizes per the kind's normal exit code. A
   missing transcript on a minimal sentinel leaves
   `meta.transcript_path: null`; a missing meta artifact leaves
   `meta.session_url` at whatever /fire already provided
   (non-null on the normal path; `null` on fire-ambiguous).

The categories are stable v1 surface: gaia implementations MUST
produce well-formed sentinels on every non-emergency path and
MUST continue to recognize minimal emergency sentinels on
read. Producers that downgrade a well-formed sentinel to a
minimal one outside the documented emergency categories are
violating the contract, not exercising a v1 flexibility.

**Success.** The `Stop` hook emits one commit on the state branch:

```
commit <sha>
    GAIA_DONE: <session_id>

A  .gaia-state/<session_id>.md                 (final assistant message)
A  .gaia-state/<session_id>.transcript.md      (readable conversation transcript)
A  .gaia-state/<session_id>.meta.json          (session identifier / timestamps; session_url always null from the hook)
```

The `<session_id>.meta.json` artifact carries
`{created_by: "gaia", schema_version: 1, artifact_type:
"session_meta", run_id, session_id, session_url, issued_at,
finalized_at}` as top-level JSON keys (the JSON header shape
from §7.3). The hook reads `session_id` from the documented
`CLAUDE_CODE_REMOTE_SESSION_ID` environment variable when it
is present, or falls back to the `session_id` field in the
hook's stdin JSON (the authoritative value Claude Code passes
to every hook). `session_url` is written as `null`
**unconditionally** — no current Claude Code environment
variable exposes the cloud-session URL directly, and the
driver's documented way of constructing one
(`https://claude.ai/code/<session_id>`) is a convention that
may drift, so the hook does not synthesize it. The URL is
populated in `meta.json` (§10.3) only from the `/fire`
response on the normal path; on fire-ambiguous recovery
`meta.session_url` stays `null` (§6.4). `issued_at` is
copied from the manifest (§7.4); `finalized_at` is the
hook's own wall clock at the point it pushes the sentinel.
This artifact is what lets `gaia recover` populate
`meta.session_id` when the driver never saw the `/fire`
response (fire-ambiguous — §6.4).

**Failure.** The `StopFailure` hook (§8.3) emits one commit on the
state branch when Claude Code reports an API error (rate-limit, auth,
overloaded, prompt-too-long, etc.):

```
commit <sha>
    GAIA_FAILED: <session_id>

A  .gaia-state/<session_id>.error.json         ({created_by, schema_version, artifact_type: "error", run_id, session_id, error, error_details, last_assistant_message} — §7.3 schema)
A  .gaia-state/<session_id>.transcript.md      (whatever transcript exists at failure time)
A  .gaia-state/<session_id>.meta.json          (session identifier / URL / timestamps — same shape as success)
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
A  .gaia-state/<session_id>.meta.json          (session identifier / URL / timestamps — same shape as success)
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
A  .gaia-state/<session_id>.rescue.head.txt    (work-branch HEAD sha at Stop time, YAML + body)
A  .gaia-state/<session_id>.rescue.base.txt    (base sha the patch was generated against, YAML + body)
A  .gaia-state/<session_id>.rescue.patch       (YAML + `git diff --binary $base..HEAD`)
A  .gaia-state/<session_id>.rescue.log.txt     (YAML + `git log --oneline $base..HEAD`)
A  .gaia-state/<session_id>.rescue.bundle      (git bundle create - $base..HEAD; no header)
```

The `.patch`, `.bundle`, `.log.txt`, `.base.txt`, and
`.head.txt` files carry enough metadata to reconstruct the
missing commits in a local clone. The `.bundle` is
authoritative: `git bundle verify` and `git fetch <bundle>
<ref>` restore the full commit graph (including merges) when
the original work-branch push never landed. The `.patch` is a
human-readable fallback for cases where the bundle is unusable;
its body is the raw `git diff --binary $base..HEAD` output
(no YAML front matter in the body itself — the artifact-level
front matter is stripped before `git apply` consumes the
patch, see recovery paragraph below). `.base.txt` records the
`$base` sha the patch was generated against so recovery can
apply the patch on the correct commit without having to guess
between `manifest.base_sha` and the origin work tip. The hook
writes these files only when the work-branch push fails; a
successful push elides them.

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
sentinel includes rescue artifacts, `gaia recover` (§6.4) and
the in-fire rescue handler (§6.1 step 8) apply the same
reconstruction sequence. For every step below, the YAML
front-matter block is stripped from each text rescue artifact
before the body is consumed (the block begins with a `---\n`
line at the start of the file, ends at the matching `---\n`
line, and is followed by a blank line; consumers read bytes
after that blank line). The stripping is mandatory: a leading
YAML block would make `git rev-parse` read "---" instead of a
sha, and `git apply` would fail to parse the patch header.

1. Read `<session_id>.rescue.head.txt` body; verify it is a
   40-char hex sha. Read `<session_id>.rescue.base.txt` body
   the same way. If either is malformed, fall back to the
   `manifest.base_sha` as `$base` and treat the HEAD sha as
   unknown (reconstruction still possible from the bundle
   alone since the bundle is self-contained).
2. Compare `rescue.head` against `git rev-parse
   refs/remotes/origin/$GAIA_WORK_BRANCH`. If they match, the
   work-branch push eventually succeeded (e.g., a late retry
   from a subsequent hook invocation) and the rescue
   artifacts are redundant; recover falls through to the
   normal merge-back path without setting `meta.rescue_ref`.
3. Otherwise run `git bundle verify` on the bundle. If valid,
   fetch the bundle's refs into the local repository under
   the **private rescue ref** `refs/gaia/recovered/<run_id>/work`.
   The bundle was created by §8.2 step 7 as
   `git bundle create <path> "$GAIA_WORK_BRANCH" "^$base"`,
   which packages the ref `$GAIA_WORK_BRANCH` (pointing at
   the work-branch tip) plus every reachable object not
   already present at `$base`. The matching restore is a
   `git fetch` from the bundle path, with an explicit
   refspec that rewrites the remote ref name into the
   private namespace:
   ```bash
   git -C "$repo" fetch \
     "$session_id.rescue.bundle" \
     "+refs/heads/$GAIA_WORK_BRANCH:refs/gaia/recovered/$run_id/work"
   ```
   (The bundle's refs carry `refs/heads/$GAIA_WORK_BRANCH`
   because that is the form `git bundle create` records when
   the source ref was a local branch; `+` on the left-hand
   side forces the update even if the private rescue ref
   already exists from a prior attempt.) Use the
   `refs/gaia/...` namespace rather than
   `refs/remotes/origin/` so the merge-back's origin fetch
   cannot clobber it. Do **not** write to
   `refs/remotes/origin/$GAIA_WORK_BRANCH`; that ref is the
   merge-back's force-fetch target (§6.1.1 step 6) and a
   concurrent or subsequent fetch would silently overwrite
   the reconstructed tip with stale or missing origin bytes.
   Record the rescue ref name and the restored tip sha in
   `meta.rescue_ref` / `meta.work_tip_sha`, then continue
   merge-back as in §6.1.1 — passing the rescue ref through
   as `{workRefOverride}` so step 6 merges the private ref
   and skips the origin fetch entirely.
4. If the bundle is invalid or unusable, strip the YAML
   front matter from `rescue.patch`, check out a temporary
   worktree at `rescue.base` (from `rescue.base.txt`, not
   `manifest.base_sha`), run `git apply --index <patch-body>`
   against it, and commit the result into the same private
   rescue ref `refs/gaia/recovered/<run_id>/work` via
   `git update-ref`. The private ref is again the merge-back's
   `{workRefOverride}` source. Alert the user that the
   reconstruction is diff-based, not commit-graph-based
   (fewer guarantees about merge history).

All artifacts are plain text (Markdown, JSON, or standard git
formats). The driver reads `<session_id>.md` on success or
abort, `<session_id>.error.json` on failure, and
`<session_id>.transcript.md` for future continuation. Binary
artifacts (`<session_id>.rescue.bundle`) are stored as git blobs
and fetched through the normal git plumbing.

### 7.4 Manifest

Before firing, the driver commits `.gaia-state/manifest.json` to
the state branch (locally, then pushes the state branch in §6.1
step 5a — the commit may also carry `.gaia-state/context.md` if
step 5 triggered overflow):

```json
{
  "created_by": "gaia",
  "schema_version": 1,
  "artifact_type": "manifest",
  "run_id": "01HXX...",
  "run_type": "user",
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

`run_type` is **required** on every manifest emitted by v1
and the driver always writes a concrete value: `"user"` for
the normal `gaia -p` / `-c` / `--resume` fire path and
`"doctor"` for synthetic runs the `gaia doctor` tier creates
(§8.4). Readers that encounter a manifest with `run_type`
missing or set to a value outside the enumerated set treat
it as `"user"` (a safe default matching v1 pre-release
manifests that may predate the field); this is the sole
back-compat exception, not a license for producers to omit
it. The `UserPromptSubmit` hook does not branch on
`run_type` for ordinary validation — manifest validation is
identical regardless — but the field is load-bearing for
`gaia gc` (§12.1's aggressive 24-hour policy applies only to
`run_type: "doctor"` runs), for `gaia history` filtering,
and for the `verify_subagents` cross-check below. Earlier
drafts spelled doctor refs as `claude/gaia/doctor-<nonce>/...`,
which collided with the `UserPromptSubmit` regex (§7.2
requires the ULID + fixed-suffix shape). Using a normal ULID
with a `run_type` discriminator keeps the parser uniform and
lets the same gc / history tooling cover both run kinds.

**Doctor-only optional field: `verify_subagents`.** Manifests
produced by `gaia doctor --cloud --verify-subagents` carry an
additional top-level boolean key `verify_subagents: true`.
The field is the sole gate that unlocks the §8.5 doctor-only
bypass allowing the parent `Agent` / `Task` launch through
the `PreToolUse` shim; without it, even a `run_type: "doctor"`
manifest keeps the default `Agent` denial. `UserPromptSubmit`
(§8.1) refuses any manifest that carries
`verify_subagents: true` unless `run_type == "doctor"` is
also set — this pairing closes the path where a forged
`verify_subagents` flag on a normal `run_type: "user"`
manifest would let a `user` run launch subagents outside the
opt-in. The field is absent on every `run_type: "user"`
manifest; readers treat the missing key as `false`.

`created_by` and `schema_version` lead every state artifact gaia
writes so future migrations can detect and translate older
artifact shapes. The concrete serialization per content type
(top-level JSON keys for `.json`, YAML front matter for `.md` and
rescue text files, no header for the binary `.rescue.bundle`) is
defined in §7.3. The manifest shown above is a JSON artifact and
therefore carries `created_by` / `schema_version` / `run_id` /
`nonce` / etc. as top-level keys, not as a YAML block.

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

**Commit identity and signing policy.** gaia writes several
classes of commits during a run, each under a normative
identity + signing posture. This table is the single source of
truth; any concrete command example elsewhere in the spec that
omits a flag is an erratum against this table.

| Commit class | Where | Identity | Signing | Editor |
|---|---|---|---|---|
| State-branch writes (`GAIA_DONE:` / `GAIA_FAILED:` / `GAIA_ABORTED:` / `GAIA_PENDING:` / `GAIA_INPUT:` / doctor beacons / rescue-artifact commits) | VM / hook | `user.name=gaia`, `user.email=gaia@noreply.invalid` | `commit.gpgsign=false` | n/a |
| Work-branch synthesis commit (`gaia: ensure hook shims on work branch`) | Driver (§6.1 step 4, §8.8) | `user.name=gaia`, `user.email=gaia@noreply.invalid` | `commit.gpgsign=false` | n/a |
| Stop-hook final sweep commit (`gaia: final sweep`) | VM / hook (§8.2 step 5) | `user.name=gaia`, `user.email=gaia@noreply.invalid` | `commit.gpgsign=false` | n/a |
| Wrong-branch rescue commit (`gaia rescue: uncommitted changes at wrong-branch Stop`) | VM / hook (§8.2 step 3) | `user.name=gaia`, `user.email=gaia@noreply.invalid` | `commit.gpgsign=false` | n/a |
| Patch-based rescue reconstruction commit (rescue reconstruction when bundle is unusable) | Driver (§7.3 rescue restore) | `user.name=gaia`, `user.email=gaia@noreply.invalid` | `commit.gpgsign=false` | n/a |
| Merge-back merge commit (`git merge --no-ff` in §6.1.1 step 6) | Driver | **Caller's configured identity** (no override) | **Caller's configured signing policy** (not overridden — honor `commit.gpgsign=true` if set) | `GIT_MERGE_AUTOEDIT=no` + `--no-edit` (never open an editor) |

Rationale:
- **VM/hook-authored commits** — gaia fixes the identity to
  `gaia@noreply.invalid` so commit attribution is unambiguous
  (the cloud VM's git config is typically empty) and disables
  signing because the VM has no signing key and a repo- or
  user-global `commit.gpgsign=true` would otherwise make the
  `git commit` hang or fail.
- **Driver synthesis / rescue-reconstruction commits** — also
  gaia identity, signing disabled, for the same reason: these
  are protocol-bookkeeping commits gaia authored, not the
  caller's intent, and attributing them to the caller would
  confuse `git blame` / `git log` on the work branch.
- **Merge-back merge commit** — the **caller's identity** and
  the **caller's signing policy**, because the merge commit is
  the caller's "accept this session's work on my branch"
  moment. Users who sign commits expect their merges to be
  signed; users who have not set up git identity will see a
  git error at merge time and can fix their config (gaia does
  not paper over that). `GIT_MERGE_AUTOEDIT=no` and `--no-edit`
  are mandatory to suppress the editor — gaia runs
  non-interactively and a hung editor looks like a hang to the
  caller.

A command example that appears elsewhere in the spec without
the `-c commit.gpgsign=false` / `-c user.name=...` flags (or
without `GIT_MERGE_AUTOEDIT=no` / `--no-edit` on the merge
commit) is understood to inherit the flags from this table.
Implementations MUST set them on the concrete `git` call
regardless.

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
  "disableSkillShellExecution": true,
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
*file-tool* writes there for every session is a free correctness
win (teammate Claude Code sessions never have a reason to write
to gaia-internal namespaces).

**Scope of the static block — precise claim.** The deny list
above names the Claude Code *file tools* (`Read`, `Write`,
`Edit`, `MultiEdit`, `NotebookEdit`) against gaia-namespace
paths. It does **not** block a `Bash` tool call from writing
into those paths via a shell redirect or subprocess, because
Claude Code's `permissions.deny` patterns on file tools do not
decode Bash argv. A non-gaia session can still run
`echo x > .gaia-state/x` or `rm -rf .gaia/` from `Bash`
unimpeded. Inside a gaia session, the session-gated
`PreToolUse` shim's Bash decoder (§8.5 step 4) picks up those
redirect targets and denies them. Outside a gaia session, the
shim classifies as non-gaia and no-ops, so the block is absent.
The narrowness is deliberate — a teammate running ordinary
Claude Code in a gaia-initialized repo should not be surprised
by a denial on their own Bash call — but the spec does not
claim a stronger-than-this "all sessions, all tools" guarantee.
Callers who want all-session Bash protection can opt in
alongside Claude Code's OS-level sandbox (`sandbox.filesystem.
denyWrite`; tracked for v1.x in §14); v1 does not ship that by
default because sandbox config adds OS dependencies that
teammates may not have.

Everything else gaia needs to enforce — `.claude/hooks/gaia-*.js`
and `.claude/settings.json` writes, `.git/**` writes, branch-
switch / hard-reset / force-push commands, state-branch pushes,
MCP-tool denials, and Bash redirects targeting gaia namespaces
— is **session-gated** in the shim and applies only when the
shim has classified the session as gaia. That keeps the
"non-gaia sessions run unaffected (assuming Node on `PATH`)"
promise (§3) honest: a teammate running ordinary Claude Code in
a gaia-initialized repo can still `git checkout other-branch`,
edit `.claude/settings.json` to adjust their own permissions,
or rebase against `main` without hitting a gaia-installed deny.

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

**Per-file-tool rule coverage.** The deny list names `Read`,
`Write`, `Edit`, `MultiEdit`, and `NotebookEdit` separately
against gaia-namespace paths because Claude Code's permission
syntax treats each file-tool as a distinct surface: an `Edit`
rule does not implicitly cover `Write` or `NotebookEdit`.
Current Claude Code docs show per-tool rules for all five
tools (and `Bash` matches on command shape, not on path),
so the explicit enumeration above is the portable spelling.
If a future Claude Code version consolidates these (e.g.,
applying `Edit` rules to all built-in file-writes), the
extra entries are redundant, not incorrect. `gaia doctor
--cloud`'s `PreToolUse` denial probe (§8.4 check 13)
exercises one deny from each tool category end-to-end, so
a proxy or settings-schema change that silently stops
honoring one of the five would fail doctor before a real
run depended on it. The session-gated `PreToolUse` shim
re-enforces the same protected-path set for gaia sessions
regardless of whether the static rule fires, so a
consolidation / drift in the static layer does not expose
gaia-managed state to writes during a gaia run.

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
  matches on the stdin `error` field (the documented matcher field;
  earlier spec drafts called it `error_type` — corrected here), but
  carries no per-session or per-run matcher — gaia therefore
  registers it without a matcher and classifies inside the shim. `PreToolUse` does accept tool-name
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
`.claude/hooks/gaia-stop.js` are Node scripts committed to the
repo. They are small in the *non-gaia* path (the common case for
teammates) and fail-closed on the *gaia-session-but-binary-
unreachable* path. The shim/global-binary split is intentionally
not symmetric:

**Shim-layer responsibilities (committed to every repo):**

- **All three shims**: read stdin, classify the session (§8.0
  step 2 below), no-op on exit 0 if non-gaia, and — for gaia
  sessions — locate the global `gaia-hook-*` binary and check
  protocol-version compatibility before delegating.
- **The `UserPromptSubmit` shim additionally** implements
  enough of the protocol to surface a `GAIA_FAILED:` sentinel
  when the global binary is unreachable: the `--- BEGIN GAIA
  CONTROL ---` fence parser (to extract `GAIA_RUN_ID` /
  `GAIA_WORK_BRANCH` / `GAIA_NONCE`), the state-branch fetch
  and manifest parse (to obtain `session_id` + validate the
  nonce + read `protocol_version`), the ability to write a
  minimal `<session_id>.error.json` to the state branch via
  the §8.2 step 10 name-matched local-branch flow, and the
  `{"decision": "block"}` output used to block the prompt.
  This is a substantial subset of the global binary — the
  shim cannot be trivially small for `UserPromptSubmit`
  specifically, because delegation might be impossible and
  the driver must be told about it without a 45-minute
  timeout. The commit in `gaia init` output notes this
  explicitly so implementers do not strip the shim back to
  "check existence + exec" for that event.
- **The `PreToolUse` shim** stays minimal: on the gaia-but-
  binary-missing path, it emits the documented deny envelope
  and exits 0 on every tool call. No git, no manifest.
- **The `Stop` / `StopFailure` shim** makes a single
  best-effort last-resort push of a minimal `GAIA_FAILED:`
  sentinel if git is reachable from the shim environment
  (§8.0 step 6 below), and otherwise exits 1 and lets the
  driver time out. The shim does **not** re-implement
  integrity checks, transcript parsing, final sweeps, or
  work-branch pushes — those live in the global binary and
  a missing global binary simply means the session cannot
  finalize cleanly.

**Global-binary responsibilities (installed via `npm i -g`):**

- The full `UserPromptSubmit` flow once validation has
  passed: work-branch checkout, overflow materialization,
  `additionalContext` emission, per-session tmp-file writes.
- The complete `PreToolUse` policy (all classifier buckets,
  path / command decoders, MCP denials, crash fallback).
- The complete `Stop` / `StopFailure` flow: defensive
  restore, final-sweep commit, integrity spot-check, work-
  branch push (with rescue artifacts on failure), transcript
  parsing, and the state-branch write.
- Every `git` flow that requires retry / name-matched
  worktrees / bounded backoff.

**Common shape.** Each shim:

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
     1. Write `.gaia-state/<session_id>.error.json` with the
        §7.3 error-artifact shape:
        `{created_by: "gaia", schema_version: 1, artifact_type:
        "error", run_id: <GAIA_RUN_ID>, session_id,
        error: "gaia-hook-binary-missing-or-version-skew",
        error_details: {expected_protocol_version: <n>}}`
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

1. Scan the entire `prompt` for `--- BEGIN GAIA CONTROL ---` …
   `--- END GAIA CONTROL ---` fences and select the **current
   run's** fence using the deterministic rule from §7.2: the last
   well-formed fence that appears before the first
   `--- BEGIN CONTEXT ---` line (falling back to the last fence
   in the prompt if no `--- BEGIN CONTEXT ---` is present). This
   tolerates prior-run fences embedded in a replayed transcript
   without picking the wrong run's branches. Parse the **known
   required keys** — `GAIA_RUN_ID`, `GAIA_WORK_BRANCH`,
   `GAIA_NONCE` — and the **known optional keys** the v1
   control-block format reserves (`GAIA_EXPERIMENTAL_MODE` for
   §15 lab-quality mode, `GAIA_DOCTOR_BEACON` for §8.4's doctor
   synthetic fire, plus any future key this spec defines).
   **Unknown `KEY: value` lines inside the fence are ignored**,
   not rejected — the parser skips any line whose key is not in
   the recognized set and continues reading. This forward-
   compatibility rule lets a newer gaia driver add a control-
   block field without a `protocol_version` bump for every
   minor protocol addition, and lets an older shim tolerate a
   field it has not yet learned about. The well-formedness
   requirement for the current-run fence (step below) only
   looks at whether the three required keys parse cleanly;
   unknown keys neither pass nor fail on their own. If no
   fence is present at all, exit 0 (non-gaia session — hook
   no-ops). If the fence immediately before
   `--- BEGIN CONTEXT ---` is present but **the required keys
   fail to parse** (missing, duplicate, malformed value), the
   hook blocks with a clear reason rather than silently
   searching further back — a malformed current-run header
   must not be hidden by an older intact one. The state
   branch is derived internally as
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
   `GAIA_RUN_ID`, if `manifest.protocol_version` differs from
   this shim's pinned `protocol_version` (§8.0), **or if
   `manifest.verify_subagents` is `true` while
   `manifest.run_type != "doctor"`** (forged doctor-bypass flag
   on a normal run; §7.4, §8.5 doctor-only bypass).
   `manifest.driver_version` is recorded for diagnostics and is
   not compared. Refusal uses the same `{"decision": "block",
   "reason": ...}` path as step 2.
4. Write `<runtime-dir>/<session_id>.json` with the **full
   validated manifest** plus `{session_id, experimental_mode}`:
   ```json
   {
     "session_id": "session_01HJK...",
     "experimental_mode": "0" | "1",
     "manifest": {
       "created_by": "gaia",
       "schema_version": 1,
       "artifact_type": "manifest",
       "run_id": "01HXX...",
       "run_type": "user",
       "nonce": "<base64-24>",
       "work_branch": "claude/gaia/01HXX.../work",
       "state_branch": "claude/gaia/01HXX.../state",
       "base_sha": "<sha>",
       "user_branch": "feature/auth",
       "upstream_remote": "origin",
       "upstream_branch": "feature/auth",
       "upstream_ref": "refs/heads/feature/auth",
       "local_only_branch": false,
       "driver_version": "1.0.0",
       "protocol_version": 1,
       "issued_at": "2026-04-21T10:30:00Z",
       "managed_file_hashes": { "...": "..." },
       "managed_gitignore_hash": "<sha256>",
       "verify_subagents": false
     }
   }
   ```
   The embedded manifest includes every key `UserPromptSubmit`
   validated (including `run_type` and the doctor-only
   `verify_subagents` flag), so the `PreToolUse` / `Stop` /
   `StopFailure` shims reading this file can distinguish a
   `--verify-subagents` doctor fire from a normal run without
   re-fetching the state branch. Unknown top-level keys on the
   embedded manifest are preserved verbatim for forward
   compatibility (see §7.2's control-block parsing rule).
   `<runtime-dir>` is `$XDG_RUNTIME_DIR/gaia/` when set and
   `/tmp/gaia-$uid/` otherwise. Storing the whole manifest is
   deliberate: later hooks (§8.2 step 6's integrity spot-check
   keys off `managed_file_hashes` and `managed_gitignore_hash`;
   §8.2 step 7's rescue-artifact fallback keys off `base_sha`;
   §8.2 step 10's `session_meta` artifact copies `issued_at`;
   §8.0's protocol-version invariant keys off `protocol_version`)
   need those fields, and rereading the manifest from the state
   branch on every invocation would double every hook's git
   round-trips and introduce a race with the state branch being
   advanced by a concurrent operation. Storing the full manifest
   here makes the runtime file the actual operational source of
   truth the spec promises it is.

   The directory is created with mode `0700`; the file is written
   with mode `0600`, `O_NOFOLLOW`, and `session_id` is rejected if
   it does not match `^session_[0-9A-Za-z_-]+$` (defense against
   path traversal). The tmp file is the operational source of
   truth for the `Stop`, `StopFailure`, and `PreToolUse`
   hooks during the session — they key by `session_id` and
   read every manifest field from this file rather than from any
   git config the agent could overwrite or from a fresh state-
   branch fetch. The file is **not secret** and is not tamper-
   proof against a hostile in-VM actor with shell access (§16.1);
   the directory permissions and `O_NOFOLLOW` are about preventing
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
2. Read the full validated manifest and the session-scoped
   fields directly from the JSON loaded in step 1. At minimum
   later steps consume: `manifest.work_branch`,
   `manifest.state_branch`, `manifest.run_id`, `experimental_mode`,
   `manifest.managed_file_hashes`, `manifest.managed_gitignore_hash`,
   `manifest.base_sha`, `manifest.issued_at`, and
   `manifest.protocol_version`. The session-scoped tmp file is the
   **operational source of truth** for these values — it was
   written by the `UserPromptSubmit` hook in the same session
   with the full manifest embedded (§8.1 step 4) and is the only
   path that survives the agent overwriting `git config` or
   mutating repo state. Rereading the manifest from the state
   branch here would add a round-trip and open a race against any
   concurrent state-branch write; the runtime file is the
   authoritative local copy. It is not secret or tamper-proof
   against a hostile in-VM actor (§16.1); the trust model treats
   the prompt boundary as the trust boundary, and the
   `PreToolUse` hook (§8.5) blocks edits to the shim files and
   to `.gaia-state/`.
3. **Verify the current branch is the work branch.** Run
   `git -C "$cd" symbolic-ref --short HEAD` (and fall back to
   `git rev-parse HEAD` if HEAD is detached). If the current
   branch is not `$GAIA_WORK_BRANCH`, the hook does **not**
   reset the local work branch to the remote tip — doing so
   would destroy any commits the session made locally before
   ending up on a different branch. Two independent work
   products may need to be preserved:
   - **Local `$GAIA_WORK_BRANCH` commits the session already
     committed before drifting off-branch.** A session that
     committed several times on the work branch, then ran `git
     checkout main` and continued working, has useful commits
     *on the local work branch ref* that are ahead of
     `refs/remotes/origin/$GAIA_WORK_BRANCH`. Those commits
     must reach origin via the normal work-branch push — not
     only the rescue ref — so the driver's merge-back picks
     them up through the documented path.
   - **The divergent HEAD and dirty worktree at Stop time.**
     Whatever the agent's HEAD currently points at (plus any
     unstaged edits) gets captured into the rescue ref below.
   The hook handles them in order:
   1. **Attempt to push `$GAIA_WORK_BRANCH` if it is ahead.**
      Before touching rescue branches, check whether the local
      `$GAIA_WORK_BRANCH` ref exists and is ahead of
      `refs/remotes/origin/$GAIA_WORK_BRANCH`
      (`git merge-base --is-ancestor "refs/remotes/origin/$GAIA_WORK_BRANCH" "$GAIA_WORK_BRANCH"`
      evaluating true AND the two refs differing). If so, push
      via the canonical name-matched flow — use a temporary
      worktree checked out at the local `$GAIA_WORK_BRANCH`
      tip (same pattern as §8.2 step 10, for the same
      GitHub-proxy reason) and push `"$GAIA_WORK_BRANCH:$GAIA_WORK_BRANCH"`.
      Log success/failure into the rescue-log artifact so the
      driver-side audit can reconstruct what happened.
   2. **Ancestry guard (normal path — not triggered here).**
      Before the normal-path `git push origin
      "$GAIA_WORK_BRANCH:$GAIA_WORK_BRANCH"` at step 7 runs,
      the hook verifies that `manifest.base_sha` is an
      ancestor of the local work-branch tip
      (`git merge-base --is-ancestor "$manifest_base_sha"
      "$GAIA_WORK_BRANCH"`). If not — i.e., something rewrote
      the work-branch history out from under the session
      (`git reset --hard <older>`, `git checkout -B
      $GAIA_WORK_BRANCH <unrelated-sha>`, etc.) — the hook
      **does not push** the work branch. Instead it falls
      through to the rescue-ref path below as if the branch
      check had failed, with a distinct rescue-log entry
      `{error: "work-branch-history-rewritten"}`. This closes
      the gap where a bypass of the `PreToolUse` `git reset`
      denial could land a forged history on origin via the
      Stop hook's unconditional push.
   3. Capture the current HEAD commit sha for the divergent
      rescue.
   4. Create a fresh worktree on a rescue branch whose local
      name equals the prospective remote target, seeded at the
      divergent HEAD:
      ```bash
      rescue_branch="claude/gaia/$GAIA_RUN_ID/rescue/$session_id"
      tmp_rescue=$(mktemp -d)
      git -C "$cd" worktree add -B "$rescue_branch" "$tmp_rescue" "$CURRENT_HEAD"
      ```
   5. **Sweep any uncommitted changes into the rescue branch.**
      Pushing `CURRENT_HEAD` alone would lose any dirty
      working-tree state the session produced before wandering
      off the work branch — exactly the content the rescue ref
      is supposed to preserve. The earlier draft used a `tar | tar`
      mirror, which copies *additions* but never removes files
      that were deleted in `$cd`'s worktree relative to
      `CURRENT_HEAD`; a session that ran `rm` on a tracked file
      and then drifted off-branch would land that delete only on
      the agent's HEAD, not in the rescue commit. v1 captures
      the worktree state via `git diff --binary` against
      `CURRENT_HEAD` — the binary-patch format records deletes
      natively (`deleted file mode` headers + an empty target),
      so the resulting commit is a faithful snapshot of the
      drifted worktree, including removals. The capture is run
      in `$cd` **without touching the agent's index file**:
      gaia writes to a scratch index pointed at by
      `GIT_INDEX_FILE` rather than copying `$cd/.git/index`
      byte-for-byte (the `$repo/.git/index` path is not always
      where git looks — worktrees and multi-worktree checkouts
      put per-worktree index files elsewhere, which
      `git rev-parse --git-path index` resolves correctly).
      Using `GIT_INDEX_FILE` is the supported way to run `git
      add` / `git diff --cached` against a sandbox index while
      the primary index stays untouched:
      ```bash
      # Redirect every `git` invocation below to a scratch
      # index so the agent's actual index (wherever git keeps
      # it for this worktree) is never perturbed.
      scratch_index=$(mktemp)
      # Seed the scratch index from the agent's current index
      # so the diff captures unstaged as well as staged changes.
      GIT_INDEX_FILE="$scratch_index" \
        git -C "$cd" read-tree "$CURRENT_HEAD"

      # Stage every change (including deletes) under the
      # scratch index, restricted to non-gaia paths. The
      # explicit pathspec exclusions match §8.2 step 5's sweep.
      GIT_INDEX_FILE="$scratch_index" \
        git -C "$cd" add -A -- . \
          ':(exclude).gaia/**' \
          ':(exclude).gaia-state/**' \
          ':(exclude).gaia-context/**'

      # Build a binary patch worktree-vs-HEAD so deletes survive.
      worktree_patch=$(mktemp)
      GIT_INDEX_FILE="$scratch_index" \
        git -C "$cd" diff --cached --binary "$CURRENT_HEAD" -- . \
          ':(exclude).gaia/**' \
          ':(exclude).gaia-state/**' \
          ':(exclude).gaia-context/**' \
          > "$worktree_patch"

      # Capture is read-only: the scratch index is discarded, the
      # agent's real index is untouched throughout.
      rm -f "$scratch_index"

      # Apply the patch into the rescue worktree (which is sitting
      # on $CURRENT_HEAD via the worktree add above) and commit.
      if [ -s "$worktree_patch" ]; then
        git -C "$tmp_rescue" apply --binary "$worktree_patch"
        git -C "$tmp_rescue" add -A
        git -C "$tmp_rescue" \
          -c commit.gpgsign=false \
          -c user.name="gaia" \
          -c user.email="gaia@noreply.invalid" \
          commit -m "gaia rescue: uncommitted changes at wrong-branch Stop"
      fi
      rm -f "$worktree_patch"
      ```
      The result: the rescue branch's tip equals
      `CURRENT_HEAD` plus a single commit whose tree exactly
      matches the agent's worktree at Stop time, deletions
      included. If `git apply --binary` fails (rare —
      indicates an inconsistent diff against the rescue
      worktree's reset state), the hook proceeds with just
      `CURRENT_HEAD`'s contents on the rescue branch and
      records the apply failure in the
      `.gaia-state/<session_id>.rescue.log.txt` so the user
      sees that uncommitted changes did not make it.
   6. Push the rescue branch with a name-matched refspec:
      ```bash
      git -C "$tmp_rescue" push origin "$rescue_branch:$rescue_branch"
      git -C "$cd" worktree remove "$tmp_rescue"
      ```
      The `claude/gaia/*` namespace is in the proxy's default
      push allowlist, so the rescue push is authorized. If the
      push fails, the hook continues to step 10 with a rescue
      patch and bundle captured instead (the fallback below).
   7. Skip the defensive-restore and final sweep (steps 4–5 below)
      and the work-branch push (step 7 below). The
      step-3.1 work-branch push attempt above already preserved
      any already-committed work; the rescue ref below preserves
      the divergent HEAD + dirty worktree separately.
   8. Fall through to step 10 to push a `GAIA_FAILED:
      <session_id>` sentinel carrying
      `{error: "work-branch-not-current",
      current_branch: "<whatever HEAD was>",
      work_branch_pushed: <true|false from step 3.1>,
      rescue_ref: "claude/gaia/$GAIA_RUN_ID/rescue/$session_id",
      rescue_push_ok: <true|false from step 3.6>}`, and exit 1.
      If the rescue-branch push failed in step 3.6, the state-
      branch sentinel also carries the full rescue artifact set
      (`.rescue.patch`, `.rescue.log.txt`, `.rescue.head.txt`,
      and a `.rescue.bundle` if one can be built) computed
      against the divergent HEAD so `gaia recover` can
      reconstruct the commits even without the rescue branch.
   9. `gaia recover` surfaces the rescue ref in its output so
      the user can inspect or merge it manually. The driver
      does not auto-merge rescue refs — the reason for the
      wrong-branch state is unknown, and a blind merge could
      land unintended commits. If step 3.1's work-branch push
      succeeded, the normal merge-back path still lands that
      portion of the work after `gaia recover <run_id> --merge`;
      the divergent HEAD in the rescue ref is handed to the
      operator to inspect manually.

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
     git -C "$cd" \
       -c commit.gpgsign=false \
       -c user.name="gaia" \
       -c user.email="gaia@noreply.invalid" \
       commit -m "gaia: final sweep"
   fi
   ```
   Per §7.4's commit identity table, gaia-authored work-branch
   commits fix the identity to gaia and disable signing so a
   repo- or user-global `commit.gpgsign=true` does not make
   the cloud VM ask for a signing key it does not have.
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
7. **Push the work branch (on integrity pass, only if the
   local ref's history is descendant of the manifest base).**
   Before pushing, re-verify the ancestry guard from step 3's
   wrong-branch fallback:
   ```bash
   if ! git -C "$cd" merge-base --is-ancestor \
        "$manifest_base_sha" "$GAIA_WORK_BRANCH"; then
     # The local work branch's history no longer descends from
     # manifest.base_sha — a bypass of the PreToolUse reset /
     # history-mutation denials rewrote the branch. Do NOT push
     # a rewritten history to origin. Fall back to the rescue-
     # ref path (step 3.4+): capture the current tree into a
     # rescue branch, write a GAIA_FAILED sentinel with
     # error: "work-branch-history-rewritten", and exit 1.
     ...
   fi
   git -C "$cd" push origin "$GAIA_WORK_BRANCH:$GAIA_WORK_BRANCH"
   ```
   The push is **unconditional within the ancestry-guarded
   path** — it runs even if the sweep added no new commits, so
   any clean local commits Claude made before the final Stop
   are synced to origin. Earlier drafts only pushed when the
   working tree was dirty, which dropped committed-but-unpushed
   work; the current rule always pushes on the integrity-pass
   path but gates on the ancestry check first to prevent a
   bypass of the `PreToolUse` reset / rebase denials from
   landing rewritten history on origin. The push uses an
   explicit `<src>:<dst>` refspec so the Claude Code GitHub
   proxy's empirically-observed "push to current working
   branch" rule (§8.4) sees a name-matched refspec from the
   same-named local branch (verified in step 3).

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
   # --binary is mandatory: plain `git diff` drops binary-file
   # changes, which would make the .rescue.patch unable to
   # reconstruct e.g. a new PNG / zip / executable the session
   # committed. The .rescue.bundle is still preferred for
   # reconstruction (commit-graph fidelity); the .rescue.patch is
   # the bundle-invalid fallback and must not be lossy.
   # Record the $base sha explicitly in rescue.base.txt so
   # recovery does not have to guess whether to apply the
   # patch on top of manifest.base_sha or the origin work
   # tip — either of which may differ from the base the
   # patch was actually generated against.
   git -C "$cd" diff --binary "$base..HEAD" > "$tmp/.gaia-state/$session_id.rescue.patch.body"
   git -C "$cd" log --oneline "$base..HEAD" > "$tmp/.gaia-state/$session_id.rescue.log.body"
   printf '%s\n' "$head_sha" > "$tmp/.gaia-state/$session_id.rescue.head.body"
   printf '%s\n' "$base"     > "$tmp/.gaia-state/$session_id.rescue.base.body"
   # Prepend the §7.3 YAML front matter to each body, then
   # rename into the final artifact path. Producers MUST emit
   # the header; recovery MUST strip it before `git apply` /
   # `git rev-parse` consume the body (§7.3 drivers strip).
   emit_with_header "rescue_patch" "$session_id" "$tmp/.gaia-state/$session_id.rescue.patch.body" \
                    > "$tmp/.gaia-state/$session_id.rescue.patch"
   emit_with_header "rescue_log"   "$session_id" "$tmp/.gaia-state/$session_id.rescue.log.body" \
                    > "$tmp/.gaia-state/$session_id.rescue.log.txt"
   emit_with_header "rescue_head"  "$session_id" "$tmp/.gaia-state/$session_id.rescue.head.body" \
                    > "$tmp/.gaia-state/$session_id.rescue.head.txt"
   emit_with_header "rescue_base"  "$session_id" "$tmp/.gaia-state/$session_id.rescue.base.body" \
                    > "$tmp/.gaia-state/$session_id.rescue.base.txt"
   # Bundle creation: include every commit reachable from the
   # work-branch tip but not from $base, and carry the
   # $GAIA_WORK_BRANCH ref pointer so the receiver can fetch
   # by name. The revision syntax is `<ref> ^<not-ref>`, not
   # `--branches=...` — mixing `--branches` with a
   # rev-range arg produces a bundle whose rev-list is the
   # union of both selectors, which includes commits
   # reachable from the work branch regardless of $base and
   # therefore ships more objects than needed.
   git -C "$cd" bundle create \
     "$tmp/.gaia-state/$session_id.rescue.bundle" \
     "$GAIA_WORK_BRANCH" "^$base" \
     2>/dev/null || true
   ```
   Each text-format rescue file carries the §7.3 YAML front-
   matter header and a single-line body (head sha, base sha,
   unified diff, log lines). `.rescue.bundle` is binary and
   carries no header. All files are added to the same temporary
   worktree the hook uses for the state-branch write (step 10
   below). The sentinel commit message is unchanged
   (`GAIA_DONE: <session_id>` on clean exit; `GAIA_FAILED:` or
   `GAIA_ABORTED:` on the respective terminal paths); the
   `.rescue.*` files appear alongside the normal final-message
   and transcript artifacts. `gaia recover` and the in-fire
   rescue handler in §6.1 step 8 both use the bundle
   (preferred) or the patch to reconstruct the missing commits
   on the driver side (§6.4 / §7.3) — and both strip the YAML
   front matter from `.rescue.patch` / `.rescue.head.txt` /
   `.rescue.base.txt` before `git apply` / `git rev-parse`
   consume the body. If the bundle create itself fails (e.g.,
   due to missing objects in the VM's repo), the `|| true` lets
   the rest of the rescue bundle proceed; the patch + log +
   head + base quartet is already enough for a best-effort
   human-driven reconstruction.
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

   **Strip gaia protocol metadata from user turns, and inline
   overflow context.** The very first user turn of a gaia
   session is the `/fire` payload, which carries the
   `--- BEGIN GAIA CONTROL ---` … `--- END GAIA CONTROL ---`
   fence and may carry `--- BEGIN CONTEXT ---` / `PREVIOUS
   CONVERSATION:` / `--- END CONTEXT ---` / `--- CONTEXT FILE ---`
   wrappers (§7.2). Subsequent user turns in `--experimental-mode`
   (§15) do not carry gaia fences directly, but a later session
   resumed via `-c` / `--resume` will replay this transcript back
   inside a new `PREVIOUS CONVERSATION:` block, and letting the
   fences and wrappers ride along would stack protocol metadata
   inside the next run's user-visible surface and pollute future
   `UserPromptSubmit` scans that (per §7.2) must pick the current
   fence. Before emitting each user turn's text to the readable
   transcript, the hook strips:
   - Any `--- BEGIN GAIA CONTROL ---` … `--- END GAIA CONTROL ---`
     fence and its contents.
   - The outer `--- BEGIN CONTEXT ---` / `--- END CONTEXT ---`
     wrapper, when it brackets the turn's task content.
   - `PREVIOUS CONVERSATION:` / `END PREVIOUS CONVERSATION`
     markers (the replay wrapper; the inner transcript body is
     kept because it is the actual conversation).

   **Overflow-context inlining.** A `--- CONTEXT FILE --- /
   --- END CONTEXT FILE ---` pointer in the first user turn
   means the real task body lives in `.gaia-state/context.md`
   on the state branch (§7.2). Stripping the marker without
   replacing it would leave the readable transcript describing
   a task the model appears never to have received — a future
   `-c` / `--resume` that replayed this transcript would hand
   the next session a conversation where the agent did work
   in response to, effectively, nothing. Instead, when the
   hook sees the pointer, it reads `.gaia-state/context.md`
   from the state branch (via the same `git show
   refs/remotes/origin/$GAIA_STATE_BRANCH:.gaia-state/context.md`
   flow §8.1 uses, without a checkout), strips the `.md`
   artifact's YAML front matter, and substitutes the body
   **inline** into the first user turn of the readable
   transcript — between the `## USER` header and any
   trailing prompt text that survived the marker strip. The
   substituted body is fenced with clear delimiters
   (`<!-- overflow context (inlined by gaia Stop hook) -->`
   … `<!-- end overflow context -->`) so a human reading the
   transcript can see what was inlined without confusing it
   for agent output. If `context.md` is missing on the state
   branch (shouldn't happen — the driver wrote it before
   firing — but defensive), the hook emits a
   `(overflow context unavailable)` marker in place of the
   body so the transcript stays readable and later replay
   surfaces the missing-content condition explicitly.

   The readable transcript captures the user's real task content,
   not the gaia transport envelope. If a turn's content is empty
   after stripping **and** no overflow pointer was present, the
   hook still emits a `## USER (<ts>)` header with a
   `(protocol metadata only)` body marker so the forward-pass
   structure stays aligned with turn numbering. Thinking and
   tool-use stripping is unchanged.
10. **Write both artifacts** via the canonical state-branch write
    routine: a temporary worktree seeded at the state-branch tip
    as a **real local branch with the same name as the remote
    target**, then push with an explicit name-matched refspec.

    **Canonical routine (shared across every state-branch write).**
    The same write flow appears in multiple places — the
    `UserPromptSubmit` fail-closed sentinel (§8.0 step 6), this
    Stop finalize step, `StopFailure` (§8.3), the experimental-
    mode `GAIA_PENDING:` / `GAIA_INPUT:` / `GAIA_DONE:` /
    `GAIA_ABORTED:` pushes (§15.2 step 4, step 9), `gaia doctor`'s
    beacon and report pushes (§8.4), and the rescue-artifact
    writes (§8.2 step 7, §8.3). Implementations MUST implement
    this as a single shared routine (e.g.
    `writeStateCommit({repo, stateBranch, message, files,
    maxRetries: 3})`) with a shared test corpus, rather than
    open-coding the sequence in each call site. Drift across
    copies was the single biggest source of bugs in earlier
    drafts. The routine's responsibilities:
    - Fetch the current state branch with an explicit refspec
      (`+refs/heads/$GAIA_STATE_BRANCH:refs/remotes/origin/$GAIA_STATE_BRANCH`).
    - **Handle stale / crashed worktrees that already check out
      `$GAIA_STATE_BRANCH`.** A prior `writeStateCommit` that
      crashed before `worktree remove` can leave the state
      branch checked out in a dangling worktree; the next
      `worktree add -B "$GAIA_STATE_BRANCH"` would refuse
      because git does not let a branch be checked out in two
      worktrees. The routine runs `git worktree list --porcelain`,
      looks for any existing worktree whose HEAD points at
      `$GAIA_STATE_BRANCH`, and removes it with
      `git worktree remove --force <path>`; if the path no
      longer exists on disk (crash left orphan metadata),
      `git worktree prune` cleans up the registration. Only
      then does the new `worktree add -B` run. This is safe
      because the state branch is protocol transport — no user
      edits live there — and because any in-progress writer
      that genuinely still held the worktree would be sharing
      a process ancestry that already died.
    - `git worktree add -B "$GAIA_STATE_BRANCH" "$tmp"
      "refs/remotes/origin/$GAIA_STATE_BRANCH"` so the local
      branch name equals the remote target.
    - Serialize artifact payloads according to §7.3 (JSON with
      top-level metadata keys for JSON artifacts; YAML front
      matter for Markdown artifacts; no header for binary
      `rescue_bundle`).
    - `git add .gaia-state`, commit with
      `git -c commit.gpgsign=false -c user.name=gaia
      -c user.email=gaia@noreply.invalid commit -m <message>`
      (repo or global `commit.gpgsign=true` would otherwise
      cause the cloud VM to ask for a signing key it does not
      have), push with an explicit `<src>:<dst>` refspec
      where `src == dst`.
    - On non-fast-forward rejection: refetch, reset the local
      branch to the new tip, re-apply the writes, recommit,
      retry (bounded at 3 attempts).
    - **Cleanup in `finally`.** Every path out of the routine
      (normal completion, push-rejection-after-retries, any
      thrown exception) runs the following cleanup sequence:
      ```bash
      git -C "$cd" worktree remove --force "$tmp" 2>/dev/null || true
      git -C "$cd" branch -D "$GAIA_STATE_BRANCH" 2>/dev/null || true
      git -C "$cd" worktree prune 2>/dev/null || true
      ```
      All three commands are required, in order: `worktree
      remove` drops the checkout, `branch -D` drops the local
      branch ref (which `worktree remove` does **not** do — it
      only detaches the worktree), and `worktree prune` clears
      any registration orphan left by a prior crashed writer.
      Each command tolerates a missing target (exit code
      swallowed) so cleanup is idempotent across the
      canonical-routine's retry and crash paths. The origin ref
      is the only one that matters for protocol correctness and
      is handled by the fire-path `git push` above plus §6.1
      step 12's branch-cleanup substep. Leaving the local
      branch ref behind
      is not a correctness issue on its own, but it confuses
      `writeStateCommit`'s next invocation (a subsequent
      `worktree add -B` would refuse because the branch is
      already checked out somewhere) and pollutes
      `git branch --list claude/gaia/*` displays. This policy
      supersedes the earlier draft's "leaving the branch ref
      behind is harmless" language, which conflicted with the
      `branch -D` line shown in the code example and left it
      ambiguous whether the delete was required or decorative.
    Centralizing this removes a class of bugs where one call
    site updates the retry loop, the refspec, the JSON/MD
    header, or the worktree cleanup, and the others lag behind.

    The Claude Code GitHub proxy is empirically observed to
    restrict git push operations to a refspec whose remote
    target matches the session's current working branch
    (§8.4 verifies this on the live transport); using a
    different local name (e.g., `gaia-state-$GAIA_RUN_ID`) and
    pushing to a differently-named remote ref is exactly the
    case the proxy is observed to reject. The `-B
    "$GAIA_STATE_BRANCH"` on `git worktree add` plus an
    explicit `<src>:<dst>` refspec where `src == dst` removes
    the ambiguity:
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
    # Session-meta artifact (top-level JSON keys per §7.3's JSON
    # header shape). session_id is read from
    # CLAUDE_CODE_REMOTE_SESSION_ID (the documented cloud-session
    # env var); if that env var is unset on a given Claude Code
    # build the hook falls back to the stdin session_id field,
    # which Claude Code passes to every hook invocation.
    # session_url is emitted as JSON null unconditionally — no
    # current Claude Code env var exposes the URL, and gaia does
    # not synthesize one from the session_id (§7.3, §10.3). The
    # JSON is produced via a proper encoder, not shell
    # concatenation, so the "null vs quoted string" distinction
    # is preserved.
    write_session_meta "$tmp/.gaia-state/$session_id.meta.json" \
      run_id="$GAIA_RUN_ID" \
      session_id="$session_id" \
      session_url=null \
      issued_at="$manifest_issued_at" \
      finalized_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
    git -C "$tmp" add .gaia-state
    # Gaia-authored commits use a fixed gaia identity and disable
    # signing so repo- or user-global commit.gpgsign=true does not
    # make the cloud VM ask for a signing key it does not have.
    # The identity is recorded in the commit metadata and flows
    # through the merge-back, so a merge commit is attributable
    # to gaia rather than to whatever identity was configured in
    # the VM's git config (often empty).
    git -C "$tmp" \
      -c commit.gpgsign=false \
      -c user.name="gaia" \
      -c user.email="gaia@noreply.invalid" \
      commit -m "GAIA_DONE: $session_id"
    git -C "$tmp" push origin "$GAIA_STATE_BRANCH:$GAIA_STATE_BRANCH"
    # Worktree cleanup: remove the worktree and delete the local
    # gaia-namespace branch ref. Leaving the branch behind is
    # merely cosmetic clutter, but the fire path's §6.1 step 12's branch-cleanup substep
    # deletes the remote ref, and leaving only the local ref
    # behind confuses `gaia history` displays that read
    # `git branch --list claude/gaia/*`.
    git -C "$cd" worktree remove --force "$tmp" 2>/dev/null || true
    git -C "$cd" branch -D "$GAIA_STATE_BRANCH" 2>/dev/null || true
    git -C "$cd" worktree prune 2>/dev/null || true
    ```
    Each `.gaia-state/*.md` artifact begins with a YAML front-
    matter header (`created_by`, `schema_version`, `artifact_type`,
    `run_id`, `session_id`); JSON artifacts (`*.meta.json`,
    `*.error.json`) carry the same metadata as **top-level JSON
    keys** rather than a front-matter block (a YAML block at the
    top of a `.json` file would fail JSON parsing). The per-type
    canonical shape is defined by the schema table in §7.3 and
    producers conform to it; readers tolerate a missing header
    (the v1 release is the first one writing it). The local
    branch name *is* the remote branch name; `git worktree
    remove` on its own leaves the local ref behind (it only
    detaches the worktree), so the `branch -D` line is required
    — a subsequent `writeStateCommit` invocation running
    `worktree add -B "$GAIA_STATE_BRANCH"` against a still-live
    ref would fail. `worktree prune` clears registration orphans
    left by prior crashed writers; together the three commands
    make cleanup idempotent.

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

   **Rescue-artifact fallback mirrors §8.2 step 7.** If the work-
   branch push fails (the same way it can on Stop), the
   StopFailure path also captures the full rescue-artifact set
   (`<session_id>.rescue.head.txt`, `<session_id>.rescue.base.txt`,
   `<session_id>.rescue.patch`, `<session_id>.rescue.log.txt`,
   and — best-effort — `<session_id>.rescue.bundle`) via the
   same commands so the driver / `gaia recover` can reconstruct
   the missing commits from the `GAIA_FAILED:` sentinel. Without
   this, a StopFailure that lands before a successful work-
   branch push would leave partial commits only on the VM's
   disk — which is recycled on session end — and the `"no work
   silently lost"` invariant would only hold for clean exits.
3. Writes `.gaia-state/<session_id>.error.json` as a JSON artifact
   with the top-level keys mandated by the §7.3 schema table:
   `{created_by: "gaia", schema_version: 1, artifact_type:
   "error", run_id, session_id, error, error_details,
   last_assistant_message}`,
   `.gaia-state/<session_id>.transcript.md` (whatever transcript
   content exists), and `.gaia-state/<session_id>.meta.json`
   (same session-metadata shape as §8.2 step 10) to the state
   branch via the same name-matched local-branch temporary-
   worktree flow as §8.2 step 10. The worktree's local branch
   is `$GAIA_STATE_BRANCH` (matching the remote name) and the
   push uses an explicit `$GAIA_STATE_BRANCH:$GAIA_STATE_BRANCH`
   refspec. Retry on non-fast-forward, bounded at 3 attempts.
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

**Doctor is environment verification, not release-gate
correctness.** gaia's correctness depends on several moving
Claude Code surfaces (hook event names, `StopFailure`
semantics, `last_assistant_message` presence,
`permissionDecision` envelope shape, `disableSkillShellExecution`,
sandbox settings, task/subagent events, skills / commands /
agents loader semantics, routine `/fire` beta header, branch-
push restrictions). Two layers protect against surface drift
and they are deliberately separated:

1. **Deterministic mocked tests in CI.** The gaia test suite
   ships **release-gate fixtures and mocks** for every one of
   those surfaces — stdin JSONL samples for each hook event,
   `permissionDecision` envelope shapes, `/fire` response
   bodies, sentinel commit formats, rescue-bundle create /
   fetch pairs, the §6.1.2 status transition table rows
   (one test per row), the `PreToolUse` Bash-parser deny
   corpus from §8.5, HTTP `/fire` failure modes from §6.1
   step 6, and the `atomicMetaWrite` temp→fsync→rename→fsync
   protocol from §6.1. These run on every PR without a cloud
   session and are what actually gates a release. A
   regression caught here is caught before any user sees it.
2. **Live `gaia doctor` checks.** The checks below are
   **environment verification** — they confirm the current
   cloud session, repo configuration, routine, and Claude
   Code build are wired up correctly for *this* operator. A
   passing doctor run says "gaia works against your current
   environment right now," not "gaia's logic is correct." A
   failing doctor check indicates an environment-side
   problem (stale beta header, missing Node on the VM,
   managed policy blocking hooks, etc.), not a gaia bug.

This split is deliberate. Some cloud surfaces are inherently
probabilistic — whether Claude launches a subagent in response
to a prompt (check 21), whether Anthropic is currently
returning overloaded errors (check 12), whether the model
attempts a protected operation at all (check 13's live probe)
— and treating a probabilistic live probe as the sole proof
of a protocol-level property would make "gaia is correct"
depend on model cooperation in the doctor run. The mocked
CI tests stand in for the protocol-level correctness claim;
the live doctor probes stand in for environment verification.
Doctor reports which layer backed each check via the
`[verified]` / `[inferred]` / `[manual]` tags above, so an
operator reading the doctor output can tell a deterministic
pass from a probabilistic one without having to read the
gaia source to understand the difference.

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
   entries in the `permissions.deny` block (§8). The doctor
   hook verifies **presence** of each gaia-managed file and
   parses the shim's advertised `protocol_version` (per §8.0 —
   each committed shim carries it inline), failing the check
   only if a file is missing or its `protocol_version` differs
   from the driver's pinned value. sha256 of each file is
   recorded in the doctor report for audit and surfaced as an
   **informational drift signal** (§8.8): patch-version
   differences between the driver's bundled shims and the
   default-branch/work-branch copies are tolerated as long as
   `protocol_version` matches, so byte-level equality is the
   wrong gate here. Hash drift that doctor surfaces alongside
   a passing `protocol_version` is expected during a normal
   patch-release rollout and is not a doctor failure.
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
13. **`PreToolUse` denials work end-to-end.** This check has
    two layers because model cooperation is not deterministic
    — a live probe can fail because Claude declined to attempt
    a protected operation, not because the hook failed.
    - **[verified]** **Shim self-test (release gate).** The
      local `gaia doctor` command cannot reach into the cloud
      VM to `exec()` the hook binary directly — the driver has
      no command-execution channel into a running routine
      session. The self-test is therefore performed **inside
      the VM, by the doctor-mode `UserPromptSubmit` hook**
      (which does have `child_process.spawn` access to the
      VM's filesystem): after parsing the `--- BEGIN GAIA
      CONTROL ---` fence and confirming `manifest.run_type ==
      "doctor"`, **and after writing the per-session runtime
      file `<runtime-dir>/<session_id>.json` with the full
      validated manifest (§8.1 step 4)** — but before
      performing the rest of the UserPromptSubmit work (work-
      branch checkout, overflow materialization,
      `additionalContext` emission) — the hook spawns the
      global `gaia-hook-pre-tool-use` binary once per gaia-
      protected category with synthetic stdin (one spawn per:
      edit `.claude/hooks/gaia-stop.js`, edit
      `.gaia-state/x`, edit `.gaia-context/x`, `Bash` call
      for `git checkout main`, `Bash` call for `git push
      origin claude/gaia/<id>/state`, an `mcp__*` tool call,
      an `AskUserQuestion` call, an `ExitPlanMode` call, a
      `Task` / `Agent` call, and an unknown-tool call).

      The ordering is load-bearing: the `PreToolUse` shim
      classifies a session as gaia iff the runtime file
      exists (§8.0 step 2, §8.5 step 1). If the self-test
      spawned the binary before the runtime file was
      written, the binary would classify every synthetic
      call as non-gaia and exit 0 silently, and the doctor
      check would pass vacuously — exactly the release-gate
      regression the self-test exists to catch. Moving the
      runtime-file write ahead of the self-test, rather than
      plumbing a test-mode env var through the global binary,
      keeps the self-test exercising the real classification
      path.

      For each spawn it captures stdout, stderr, and exit
      code, and asserts the binary emits the documented
      `hookSpecificOutput.permissionDecision: "deny"`
      envelope and exits 0. The hook then writes a
      `.gaia-state/<doctor_session_id>.doctor-report.json`
      artifact (§7.3-style JSON with top-level metadata,
      plus a `checks` array carrying per-category result,
      envelope, exit code, and captured stderr) into the
      state branch on the same commit as the rest of the
      doctor-run state. The driver-side `gaia doctor`
      consumes that artifact from origin after the doctor
      run finishes and renders it into the doctor report.
      This exercises the shim's decision logic without
      depending on the model's cooperation and is the
      load-bearing release gate: if the shim's output shape
      ever regresses to the deprecated top-level
      `{"decision": "deny"}` envelope, doctor fails before
      any real run depends on it. Doctor-mode hook execution
      is gated by `manifest.run_type == "doctor"`; non-
      doctor runs skip the self-test entirely so it cannot
      slow down real-user sessions.
    - **[inferred]** **Live end-to-end probe.** Doctor fires
      a session whose prompt instructs Claude to attempt one
      operation from each category and confirms Claude Code
      honored the shim's denial on at least one attempted
      operation in the session. A pass is strong evidence
      that the documented envelope is still what Claude Code
      processes on this version; a fail is ambiguous between
      "envelope broken" and "model never tried the
      operation," and doctor reports which case was
      observed. Because the shim self-test above covers the
      deterministic half, the live probe is marked
      `[inferred]` rather than `[verified]` and is not a
      release gate on its own.
14. **[verified]** No foreign project-level handler has appeared
    on `UserPromptSubmit`, `Stop`, or `StopFailure` in
    `.claude/settings.json` since install. For `PreToolUse`,
    doctor checks for matcher overlap with gaia's protected
    matchers and warns rather than fails.
    **[verified]** **No committed files under
    `.claude/skills/**`, `.claude/commands/**`, or
    `.claude/agents/**`** (§2, §8.7). Doctor enumerates
    every path under those three roots in the work-branch
    tree the VM sees and fails on presence — the check is a
    recursive file count, not a frontmatter scan, because v1
    refuses on presence regardless of contents. Includes
    `.md`, `.json`, and any other file extension (Claude
    Code's skill/command/agent loaders may recognize
    additional formats in future versions; the presence
    check is stable against those extensions).
    **[manual]** Managed-policy and plugin-scope handlers cannot
    be enumerated from inside the repo; doctor explicitly notes
    this gap in the report so the operator knows project-layer
    cleanliness does not rule out higher-priority handlers
    attached out-of-band.
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
17. **[verified]** **Routine / repo binding.** The driver runs a
    doctor synthesis (a normal ULID-shaped run with `run_type:
    "doctor"` recorded on the manifest — see "Doctor protocol
    runs" below) and pushes a `binding` beacon ref alongside the
    work / state branches under that doctor run's
    `claude/gaia/<doctor_run_ulid>/binding`. The
    `--- BEGIN GAIA CONTROL ---` block carries the expected repo
    `owner/name`. The `UserPromptSubmit` hook on the VM reads
    the cloned repo's `origin/url` (via `git -C
    "$CLAUDE_PROJECT_DIR" config remote.origin.url`), normalizes
    it, compares to the expected `owner/name`, and fetches the
    beacon branch to confirm it is present. If the VM's clone
    is of a different repo or lacks the beacon, doctor fails
    with the specific mismatch (expected vs. observed
    `owner/name`, beacon-fetch result) and points at the
    routine's configuration as the likely culprit. This is the
    end-to-end check that the routine is attached to exactly
    the same GitHub repo as the driver checkout's origin (§5
    prerequisite). Multi-repo routines fail this check even if
    one of the cloned repos matches, because a single gaia
    session may end up cloning the wrong one on any given fire.

**Doctor protocol runs.** Doctor runs use the **normal
`claude/gaia/<ulid>/{work,state}` branch shape** the
`UserPromptSubmit` regex enforces (§7.2: `^claude/gaia/[0-9A-
HJKMNP-TV-Z]{26}/work$`); earlier drafts spelled doctor refs as
`claude/gaia/doctor-<nonce>/*`, which the hook would have
rejected before ever validating the manifest. The doctor uses a
fresh ULID per probe, distinguished from a real run by:
- `manifest.run_type: "doctor"` (a required manifest field —
  see the §7.4 schema table) — real runs write `"user"`
  explicitly, doctor runs write `"doctor"`. The
  `UserPromptSubmit` hook does not change behavior based on
  `run_type` (the manifest is still validated the same way),
  but the field is load-bearing for branch lifecycle: `gaia gc`
  uses it to apply the aggressive 24-hour age policy to doctor
  refs (§12.1) and `gaia history` filters doctor runs out of
  the default listing.
- `meta.json.run_type: "doctor"` recorded driver-side under the
  same per-run lock (§10.3).
Beacon refs (binding, doctor-only state samples, etc.) live
under the same doctor run id — e.g., `claude/gaia/<doctor_
run_ulid>/binding` — so they are swept by the same gc rule and
reach the proxy through the same `claude/gaia/*` allowlist.
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
    the settings block the VM loaded and fails if either
    `enabledPlugins` or `extraKnownMarketplaces` (the current
    Claude Code project-settings plugin-loading keys) has a
    non-empty value. Same reasoning as `.mcp.json`: plugins
    can register hooks and MCP servers outside gaia's
    visibility. Doctor also warns (does not fail) if the
    settings block contains any plugin-loading-like key gaia
    has not classified — a successor name Claude Code may
    introduce — so a new plugin-loader does not slip past the
    policy silently. `strictKnownMarketplaces` is managed-
    settings-only and is not flagged in project settings.
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
21. **Subagent tool-call coverage.** `Agent` is **denied by
    default in v1** for normal `run_type: "user"` runs
    (§8.5); this check exists to support the opt-in
    (`gaia.config.json` →
    `spawner_config.allow_agent: true`) by positively
    observing the session-id correlation that would make
    `Agent` safe to allow. The default `gaia doctor --cloud`
    run reports the subagent state and pass/fail, but a fail
    here does **not** by itself fail the doctor tier — `Agent`
    is denied by default regardless. The only path that
    treats this check as a gating release-gate is `gaia
    doctor --cloud --verify-subagents`, which is what the
    operator runs to enable the opt-in.

    **Bootstrap bypass (doctor-only).** The probe needs a
    subagent to actually launch to observe correlation, but
    the default shim denies `Agent`. Resolution: `gaia doctor
    --cloud --verify-subagents` fires a manifest with
    `run_type: "doctor"` **and** `verify_subagents: true`.
    The `PreToolUse` shim allows the parent `Agent` / `Task`
    launch only when both flags are set on the runtime
    manifest (§8.5 doctor-only bypass); the subagent's own
    tool calls still flow through the default classifier, so
    this exception does not widen the bypass surface for
    anything other than the single parent launch. A real
    (`run_type: "user"`) manifest that carries
    `verify_subagents: true` fails `UserPromptSubmit`
    manifest validation before any tool call can fire, so
    the flag cannot be abused from a normal run.

    - **[inferred] Default subagent probe.** Doctor's
      synthetic payload prompts the session to launch a
      subagent that attempts one operation from each
      protected category, then observes whether the
      `permissionDecision: "deny"` envelope surfaces for
      each. Without the `--verify-subagents` flag the
      manifest has `verify_subagents: false`, the shim
      denies the parent launch, and the probe reports
      "unexercised — run with --verify-subagents to
      bootstrap." The default tier **does not fail the
      tier** on an unexercised probe — `Agent` denial is
      gaia's safe default, and an unexercised probe just
      means the operator has no evidence yet to flip the
      opt-in.
    - **[verified] Session-id correlation static probe.** When
      `--verify-subagents` has allowed the subagent to
      launch, the subagent's first tool call lands on the
      `PreToolUse` shim and the shim captures the
      `session_id` field from stdin. Doctor verifies that
      the captured `session_id` equals the parent session's
      `session_id` and logs the match. A mismatch here
      means the `Agent` bucket in §8.5 is not safe to allow
      under any opt-in, and doctor refuses to record a pass
      even if the operator ran `--verify-subagents`. Drives
      the v1.x §14 work to ship a subagent-scoped hook
      path.
    - **`--verify-subagents` mode (release-gate for opt-in).**
      Same probe as above, with the bootstrap bypass active.
      Doctor retries the subagent-launch prompt with
      progressively more-explicit instructions (up to a
      small bounded retry count) until either a subagent
      launches and the static probe passes (verified —
      opt-in eligible) or the retry budget exhausts with no
      subagent launch (unexercised — opt-in not eligible,
      doctor fails this check). The driver records a
      successful `--verify-subagents` pass with a timestamp
      in `.gaia/runs/<doctor_run_id>/meta.json` (where
      `run_type: "doctor"`) and a separate
      `.gaia/verify-subagents.json` summary (timestamp,
      Claude Code version observed, parent / child
      `session_id` pair); the next normal `gaia` invocation
      reads that file and respects the opt-in only if it is
      present and recent (default: 14d freshness window;
      stale pass auto-disables the opt-in until a fresh
      `--verify-subagents` runs).
22. **[inferred]** **Subprocess env scrub.** The cloud
    environment has `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` set
    (§5). The VM-side doctor hook spawns a synthetic `Bash`
    invocation of `env` and greps for `ANTHROPIC_*` and
    known cloud-platform credential env var names, pushing
    the (scrubbed) list to the doctor report artifact for
    the driver to render. A match fails the check with the
    env var name that leaked. `[inferred]` rather than
    `[verified]` because the scrub surface is under active
    Claude Code evolution; newly-added credential env vars
    that gaia has not named would slip past the grep. The
    check is a defense-in-depth signal, not a release gate
    — the §8.5 `PreToolUse` shim remains the primary
    protection surface.
23. **[verified]** **No `TaskCreate` / `Task` surface
    exposed to gaia sessions.** Claude Code's current docs
    surface `Task` / `TaskCreated` / `TaskCompleted` as a
    teammate-task workflow that can spawn agent-like work
    parallel to the session. The `PreToolUse` shim (§8.5)
    already denies `Task` in the Agent bucket at tool-call
    time, but `TaskCreated` is a **separate hook event**
    that fires when a task is created outside the normal
    tool-call surface (direct slash-command expansion is a
    documented path). Doctor verifies two things: (a) the
    VM-side doctor hook's synthetic-fire record does not
    list `Task` in the enumerated `tool_name`s its
    `PreToolUse` classifier saw (check 6 does the
    enumeration; this check asserts `Task` is absent on
    that list or was denied if present); (b) no project
    settings layer registers a `TaskCreated` or
    `TaskCompleted` handler that would fire under gaia's
    session scope. Failing either is a release gate — a
    surface that spawns agent-shaped work outside gaia's
    `PreToolUse` visibility is a protocol bypass.
24. **[verified]** (If `--experimental-mode` is configured)
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
   `tool_name` into one of six buckets:
   - **Known read-only tools** — `Read`, `Glob`, `Grep`,
     `LS`, `NotebookRead`, `TodoRead`, `TodoWrite` (read
     semantics from the protocol's perspective — it mutates a
     per-session todo list that is not protected state).
     Allowed without further inspection; exit 0. `LS`,
     `NotebookRead`, `TodoRead`, and `MultiEdit` appear in
     this and neighboring buckets for compatibility: current
     Claude Code hook-reference docs may no longer emit some
     of these names in production hook stdin, but the classifier
     keeps them listed as aliases so a future Claude Code
     build that surfaces a legacy name does not land in the
     unknown-tool bucket. `gaia doctor`'s enumeration (§8.4)
     surfaces the actual live set for periodic reconciliation.
   - **Known subagent launcher** — `Agent`, `Task`, and any
     future successor Claude Code adds for the same purpose
     (the Claude Code docs currently describe both names for
     subagent-launching surfaces; gaia's classifier treats
     them as one bucket so a rename in either direction does
     not silently widen the bypass). **Denied by default in
     v1**, with the same `permissionDecision: "deny"`
     envelope as the unknown-tool bucket and reason `"gaia v1
     denies Agent / Task by default — subagent session-id
     correlation is not yet verified end-to-end. Re-enable
     after gaia doctor --cloud --verify-subagents observes a
     live subagent tool-call under the parent session_id;
     opt-in via gaia.config.json
     spawner_config.allow_agent: true."` The concern is real:
     subagent tool calls *should* carry the parent's
     `session_id` and fire `PreToolUse` under the same
     classification — and §8.4 check 21 attempts to verify
     this — but a passive doctor check that reports
     "unexercised" when the model declines to launch a
     subagent does not prove the correlation holds, only that
     we did not see evidence either way. Treating "unexercised"
     as a green release gate would let a future Claude Code
     version (or a model that did not launch a subagent during
     the doctor run) silently widen the shim's bypass surface.
     v1's safer default: deny `Agent` until the operator
     explicitly enables it after doctor *positively observes* a
     subagent's tool-call hitting the shim under the parent
     session_id as expected.

     **Doctor-only bootstrap bypass.** Without a narrow
     exception the opt-in is self-contradicting: `--verify-
     subagents` needs a subagent to actually launch to observe
     the correlation, but the shim denies `Agent` because
     `allow_agent` has not yet been verified. The shim therefore
     **allows the parent `Agent` / `Task` launch exclusively
     when both** `<runtime-dir>/<session_id>.json`'s embedded
     `manifest.run_type == "doctor"` **and** the manifest
     carries `verify_subagents: true` (a doctor-fire-only
     manifest flag — absent on every `run_type: "user"`
     manifest). The bypass covers **only** the parent launch;
     tool calls made *inside* the spawned subagent still flow
     through the normal classifier, so protected paths,
     protected commands, `mcp__*`, `WebFetch`/`WebSearch`, and
     every other deny bucket continue to apply unchanged. The
     shim also records the observed subagent `session_id`
     against the parent in the runtime artifact the doctor
     probe consumes (§8.4 check 21) so a rogue launcher that
     correlates under a different id still registers as a
     failed correlation, not a silent pass. Any real (`run_type:
     "user"`) fire that carries `verify_subagents: true` is a
     manifest-forgery attempt — the `UserPromptSubmit` hook
     rejects it on manifest validation (§8.1), so the shim
     never sees a non-doctor manifest with the flag set.

     **Opt-in path (`gaia.config.json` →
     `spawner_config.allow_agent: true`).** Setting this key
     allows `Agent` for normal `run_type: "user"` runs. The
     opt-in is gated on a prior successful run of `gaia doctor
     --cloud --verify-subagents` (a check 21 variant that fires
     a manifest with `run_type: "doctor"` +
     `verify_subagents: true`, then retries the subagent-launch
     prompt with progressively-more-explicit instructions until
     either (a) a subagent tool call lands on the shim with
     `stdin.session_id == parent.session_id`, in which case the
     check passes [verified], or (b) the retry budget exhausts
     with no subagent launch, in which case it fails
     [unexercised → red]). Without a prior verified pass, the
     driver refuses to enable the opt-in even if the config key
     is set, and prints the doctor command to run.

     A future Claude Code version that breaks the correlation
     fails check 21 on the next doctor run; the driver
     auto-disables the opt-in until a fresh
     `--verify-subagents` pass restores it. A subagent-scoped
     hook path that lets gaia model subagents independently
     ships in §14 and would let v1.x lift this default.
   - **Known user-input tools** — `AskUserQuestion` (the
     documented tool Claude Code exposes for in-session user
     questioning; not to be confused with the stable
     `--experimental-mode` Stop-hook block-and-inject path in
     §15). Denied in gaia stable mode with reason
     `"gaia stable mode has no in-session user input surface
     — put the question in the final assistant message or use
     --experimental-mode."` In `--experimental-mode`, the
     Stop-hook block-and-inject path (§15.2) is how user input
     reaches the session, not `AskUserQuestion`, so the tool
     is denied in that mode too with the same reason — gaia
     controls user input through the state-branch reply
     channel, not through a parallel in-tool surface.

     **Why deny rather than allow + defer.** Current Claude
     Code's `PreToolUse` documents a `defer` /
     `updatedInput` flow for `AskUserQuestion` in non-
     interactive mode: the tool call is paused and the
     model waits for an out-of-band resume that supplies
     the user's answer. That works in interactive
     (foreground) Claude Code where the human sees the
     question, but routines have **no documented public
     resume / send-message API** (§14): `/fire` is fire-
     and-forget, the response carries only the session id,
     and there is no endpoint that injects a "deferred
     answer" into a live session. gaia's `-c` /
     `--resume` is *replay* continuation through a fresh
     `/fire` (§6.2), not in-session resume — so a
     deferred `AskUserQuestion` could never be answered;
     the session would either hit its hook timeout or
     burn the wall-clock VM ceiling waiting for a resume
     that the transport cannot deliver. Allowing the tool
     and trying to defer would silently strand sessions;
     denying it surfaces the constraint immediately and
     points the model at a path that actually closes the
     loop (final assistant message → `gaia -c` re-
     invocation). If Anthropic later exposes a
     send-message-to-session API for routines, this
     bucket can flip to allow + defer in a v1.x release.
   - **Known plan-mode tool** — `ExitPlanMode`. Denied with
     reason `"gaia routines do not run in plan mode; the
     saved routine prompt (§9) does not enable plan mode."`
     If a future Claude Code version puts a gaia session into
     plan mode unexpectedly (or an injected prompt convinces
     the model to request plan mode via some other surface),
     denying `ExitPlanMode` keeps the session from flipping
     modes silently. This is belt-and-suspenders — gaia
     routines never have a documented path into plan mode —
     but it is a fail-closed default that costs nothing.
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
   launchers. The v1 policy reflects §9's design intent that
   the agent mutates only the gaia work branch and the driver
   owns every merge-back / cross-branch operation; deny
   patterns are therefore broader than "just state-branch
   writes and force pushes":

   **Branch / worktree switching (denied — agent stays on the
   work branch).**
   - `git checkout *` (all forms, including file-checkouts —
     tracking is handled by the agent-facing `git restore`
     subset below)
   - `git switch *`
   - `git worktree add *` / `git worktree remove *` / `git
     worktree move *`

   **History mutation (denied — the driver, not the agent,
   owns history surgery).**
   - `git reset --hard *`, `git reset --merge *`,
     `git reset --keep *`, `git reset --soft *`,
     `git reset <ref>` (argument-forms that rewind HEAD
     regardless of mode; the bare-ref form is the default
     mixed mode)
   - `git rebase *` (every subcommand: start, continue,
     abort, skip, edit-todo)
   - `git merge *` (driver handles merge-back; an agent-
     initiated merge onto the work branch collides with the
     driver's merge topology)
   - `git pull *` (pull implies fetch + merge; the fetch alone
     is allowed via `git fetch`)
   - `git cherry-pick *`
   - `git revert *` (produces a commit; allowed-looking but
     often paired with rebase in practice and easier to
     deny uniformly — agents can still `git revert` via
     manual edit + commit if the task requires it)
   - `git commit --amend *`
   - `git filter-branch *`, `git filter-repo *`,
     `git replace *`
   - `git branch -D *`, `git branch -d *`, `git branch -m *`,
     `git branch -M *` against any `claude/gaia/*` ref or
     against the recorded `meta.user_branch`. Deletion /
     rename of unrelated local branches is allowed (agents
     may legitimately tidy branches they created during the
     task).
   - `git restore --source=* *` when the source is a ref
     other than `HEAD`.

   **Pushes (denied unless target is the current run's work
   branch).**
   - `git push` to any ref other than the current run's
     `work_branch` (the value from `<runtime-dir>/<session_id>.json`'s
     `manifest.work_branch`) — this closes the gap where an
     agent could push `main` or any `claude/gaia/*/state`
     branch via a non-force push. The rule is deny-by-default:
     only a push whose target refspec resolves to the
     session's own work branch is allowed.
   - `git push -f` / `git push --force*` / `git push
     --force-with-lease*` (all force forms, regardless of
     target — gaia never force-pushes; cf. §6.1.1)
   - `git push * claude/gaia/*/state*` (redundant with the
     work-branch-only rule but kept as a named deny so a
     bug that lets the work-branch check through does not
     re-expose state-branch writes)

   **Ref-manipulation plumbing.**
   - Any command containing `git update-ref` or `git
     symbolic-ref` against a gaia ref or the recorded user
     branch.
   - `git fetch *` to any remote other than `origin` (gaia's
     state transport is `origin`-bound; an agent fetching
     from an arbitrary remote introduces untracked
     dependencies into the session).

   **gaia-owned config.**
   - `git config --unset gaia.*` / `git config --remove-section gaia`.

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
   can classify it cleanly. Assume `manifest.work_branch ==
   "claude/gaia/X/work"` for the pushes.
   | Input | Expected |
   |---|---|
   | `git checkout main` | deny |
   | `git -C . checkout main` | deny |
   | `git -C /tmp/other checkout main` | deny (we over-block rather than under-block on non-repo-root git -C) |
   | `git switch feature/other` | deny |
   | `git reset --hard HEAD~1` | deny |
   | `git reset --soft HEAD~1` | deny (all reset modes denied) |
   | `git reset HEAD~1` | deny (default mixed mode) |
   | `git rebase main` | deny |
   | `git rebase --continue` | deny |
   | `git merge feature/other` | deny |
   | `git pull origin main` | deny |
   | `git cherry-pick <sha>` | deny |
   | `git commit --amend` | deny |
   | `git branch -d claude/gaia/X/work` | deny (gaia-namespace rename/delete) |
   | `git push origin HEAD:claude/gaia/X/work` | allow (target is the current run's work branch) |
   | `git push origin HEAD:main` | deny (target is not the current run's work branch) |
   | `FOO=bar git push origin claude/gaia/X/state` | deny |
   | `env FOO=bar git push origin claude/gaia/X/state` | deny |
   | `sudo git push origin claude/gaia/X/state` | deny |
   | `bash -c 'git push origin claude/gaia/X/state'` | deny (nested parse) |
   | `sh -c "git checkout main"` | deny (nested parse) |
   | `(git push origin claude/gaia/X/state)` | deny (subshell decoded) |
   | `echo hi && git push -f origin main` | deny (second segment matches force-push) |
   | `git push --force-with-lease origin HEAD:claude/gaia/X/work` | deny (all force forms denied) |
   | `git \<NL>push origin HEAD:refs/heads/claude/gaia/X/state` | deny (continuation collapsed) |
   | `git fetch origin claude/gaia/X/work` | allow |
   | `git fetch upstream main` | deny (fetch from non-origin remote denied) |
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
   `Glob`, `Grep`), in which case it exits 0 silently so a bug
   in the protection logic does not brick the session's reads.
   **`Agent` (and any future subagent-launcher successor such as
   `Task`) is deliberately excluded from this allow-list**: a
   crashed protection hook must not let a gaia session launch a
   subagent, because a subagent runs tool calls gaia has not
   classified under this session's protections and the shim
   cannot verify that the subagent's `session_id` correlation
   (§8.4 check 21) still holds when the classifier crashed. On
   any exception-path request to launch a subagent, the shim
   emits the deny envelope the same as for any other write-
   capable tool. This gives the gaia-protected write surface a
   deny-on-crash, not just deny-on-classified-denial, and keeps
   the deny-on-crash posture coherent with the default-denial
   posture for `Agent` in step 2 of the main classifier above.
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
- `StopFailure` accepts an `error`-field matcher but no per-session
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

- `gaia init` reads the existing `.claude/settings.json` (if any)
  and scans for pre-existing handlers on `UserPromptSubmit`,
  `Stop`, or `StopFailure`. If it finds any handler it did not
  write itself (see detection below), it aborts before
  modifying any file. The printed error:
  - Names the conflicting event(s) and the first handler's `command`.
  - Offers three remediations: (1) remove the pre-existing handler;
    (2) inline whatever it does into the gaia shim yourself and rerun
    `gaia init --force-merge-hooks` (flag reserved for v1.x, not
    implemented in v1 — the message spells that out); (3) do not
    install gaia in this repo.
  - Exits non-zero.
- **Committed files under `.claude/skills/**`,
  `.claude/commands/**`, or `.claude/agents/**` are a separate
  hard refusal** (§2): `gaia init` enumerates every file under
  those roots and aborts on *presence*, not on frontmatter
  contents. Those load surfaces can inject tools, fork contexts,
  select subagents, and run inline shell preprocessing that
  bypasses the `PreToolUse` shim, so v1's presence check is
  strictly broader than a per-field scan. The v1.x structural
  allow-list tracked in §14 is the path that will let
  "pure-content" skills/commands/agents land in a gaia repo
  without relaxing the protocol's invariants.
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
- `gaia doctor` re-runs both scans on every invocation — the
  settings-layer scan for foreign handlers on `UserPromptSubmit`,
  `Stop`, or `StopFailure`, and the presence scan for files
  under `.claude/skills/**`, `.claude/commands/**`, or
  `.claude/agents/**` — and **fails** if a foreign handler
  appears post-install or a committed file shows up under any
  of those three roots. For `PreToolUse`, doctor warns only on
  matcher overlap.
  Doctor **cannot reliably detect** managed-policy or plugin-
  level handlers from inside the repo and is explicit about that
  limitation — if an enterprise configuration or plugin attaches
  a `Stop` hook to every session, gaia's protocol may race or be
  overridden without visible warning.

**What this does not block (but does enumerate).**
`PostToolUse`, `SubagentStop`, `Notification`, `SessionStart`,
`ConfigChange`, `WorktreeCreate`, and any other event outside
the four gaia owns are permitted to coexist freely.
`PreToolUse` is shared, not exclusive (per the bullets above).
`gaia init` and `gaia doctor` enumerate all foreign project
hooks — event, matcher, and command — under a `gaia does not
control these project hooks` banner so the operator sees
exactly what runs alongside gaia. `SessionStart` gets a louder
advisory because it runs before `UserPromptSubmit` and can
mutate worktree or environment state that gaia's invariants
assume is stable.

**What this now blocks (in addition to the four exclusive
events).** `gaia init` also refuses a project-scoped
`TaskCreated` or `TaskCompleted` handler, for the same
protocol-bypass reasoning that drives the doctor release gate
(§8.4 check 23): those events can spawn agent-shaped work
parallel to the session, outside the `PreToolUse` shim's
visibility. This is a policy alignment with §2's refusal of
committed skills / commands / agents; the resulting list of
"cannot coexist in project scope" events is: `UserPromptSubmit`,
`Stop`, `StopFailure`, `TaskCreated`, `TaskCompleted`. `gaia
doctor --cloud` records whether those events fire during a synthetic
run so the operator can reason about the composition. The
advisories are informational, not refusals.

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
   the push in §6.1 step 5a uses the matching refspec.)
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
   Read the default-branch shim's advertised `protocol_version`
   (a small integer baked into each shim file at `gaia init`
   time, surfaced via a top-of-file comment or a JSON sibling
   manifest the reconcile step parses). If it differs from the
   driver's pinned `protocol_version`, the default branch is
   stale relative to the installed driver and a synthesis
   based on it would commit shims that disagree with the
   manifest the driver is about to push. Abort with exit 2 and
   a message pointing at `gaia init --verify-default-branch`
   (which re-installs the current driver's shims on the
   default branch, typically via a PR). **Patch/minor-version
   differences between the default-branch shim and the
   driver's bundled copies are tolerated as long as
   `protocol_version` matches** — that is §1.1's patch-release
   promise, and `protocol_version` is the sole hard-fail gate
   here. The driver does **not** compare sha256 hashes of
   default-branch shim files against its bundled copies: that
   would contradict the patch-release promise (identical
   `protocol_version` + non-identical bytes is the exact shape
   of a patch release), turn every v1.0.0 → v1.0.1 into a
   work-branch synthesis abort, and defeat the "same protocol,
   drop-in-compatible" invariant the shim layer relies on.
   Hash-level drift between the driver's bundled shims and the
   default-branch copies is a `gaia doctor` informational
   signal (§8.4), not a fire-path gate.
3. If any file differs from the work-branch copy, write the
   default-branch version into the work-branch tree and stage it.
4. If anything was staged, commit with message
   `gaia: ensure hook shims on work branch`. The synthesis
   commit uses the **gaia identity with signing disabled** per
   §7.4's commit identity table:
   ```
   git -C "$tmp" \
     -c commit.gpgsign=false \
     -c user.name="gaia" \
     -c user.email="gaia@noreply.invalid" \
     commit -m "gaia: ensure hook shims on work branch"
   ```
   **Do not push here** — pushing the work branch is §6.1 step
   5a's responsibility, and that step deliberately defers the
   push until the state branch has its manifest commit ready so
   origin sees one work-branch push and one state-branch push
   per run. Earlier drafts said "commit and push" here, which
   conflicted with §6.1 step 5a's single-push contract and could
   double-push the work branch on runs that triggered synthesis.
   If the tree was already clean against the default branch,
   skip the commit (§6.1 step 5a still pushes the work branch
   unconditionally).
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
- `disableSkillShellExecution: true` — the current Claude Code
  project-settings key that disables inline shell execution
  for skills and custom commands from project, user, plugin,
  and additional-directory load surfaces. v1 already hard-bans
  committed files under `.claude/skills/**`,
  `.claude/commands/**`, and `.claude/agents/**` (§2), so this
  entry is a **defense-in-depth backstop** for two scenarios
  the hard-ban does not cover: (a) a teammate who has their
  own user-layer skills and pulls a gaia repo in the same
  Claude Code install — user-scoped skills would otherwise
  run inline shell preprocessing under gaia's settings scope;
  (b) a future Claude Code version that adds a new skill load
  path gaia has not classified yet. Setting the key to `true`
  kills the shell-preprocessing surface at the engine layer
  rather than at the presence-check layer, so even if the
  hard-ban misses a new path, the `!` / `` `!…` `` bypass
  path does not fire. Foreign settings that already set this
  key to `true` are left alone; if a foreign settings writes
  `false` (explicitly opting into inline shell execution),
  `gaia init` aborts and refuses to compose with that setting
  — the conflict cannot be silently reconciled.

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

A protection hook (PreToolUse) will deny attempts to mutate
gaia-managed state or to take history-rewriting actions on
any branch. Specifically it denies edits to
.claude/hooks/gaia-*.js, .claude/settings.json, the .gitignore
".gaia/" / ".gaia-context/" rules, .git/**; and it denies
`git checkout`, `git switch`, `git reset` (all modes),
`git rebase`, `git merge`, `git pull`, `git cherry-pick`,
`git commit --amend`, force pushes, pushes to any branch other
than the current work branch, and pushes to claude/gaia/*/state.
If you see a denial from that hook, do not try to work around
it — it means you are touching gaia-managed state that is
supposed to be off-limits for this session. Read the denial
reason and pick a different approach; if you need work to
land on another branch, the driver handles that at merge-back
time and you should put the request in your final assistant
message instead.

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
  "run_type": "user",
  "session_id": "<opaque session id string returned by /fire>" | null,
  "session_url": "<opaque session URL string returned by /fire>" | null,
  "status": "preparing" | "pending" | "success" | "no-merge" | "failed" | "fire-failed" | "fire-ambiguous" | "timeout" | "push-rejected" | "conflict" | "aborted",
  "branches_pushed": false,
  "fire_attempted": false,
  "request_body_started": false,
  "user_branch": "feature/auth",
  "work_branch": "claude/gaia/01HXX.../work",
  "work_tip_sha": "9a8b7c6",
  "state_branch": "claude/gaia/01HXX.../state",
  "upstream_remote": "origin",
  "upstream_branch": "feature/auth",
  "upstream_ref": "refs/heads/feature/auth",
  "local_only_branch": false,
  "pushed_upstream": true,
  "base_sha": "abc1234",
  "merge_sha": "def5678",
  "rescue_ref": "refs/gaia/recovered/01HXX.../work" | null,
  "continued_from": "<prior session_id>",
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

**`work_tip_sha`** is the work branch's HEAD sha after the
session pushed its final commits, recorded by the driver
after §6.1 step 8's fetch and **before** §6.1 step 12's branch-cleanup substep's
optional remote deletion. It is recorded for audit purposes
(surfaced by `gaia history show` so operators can still
inspect a prior run's work-branch tip after the origin ref
has been swept by §6.1 step 12's branch-cleanup substep or `gaia gc`) and for the
`gaia recover` ancestry check on `--resume <session_id>`
continuations (§6.2 — the containment guard re-reads
`meta.work_branch` + `work_tip_sha` to verify the caller's
current branch transitively contains the prior run's
commits). Record it under the per-run lock alongside the
terminal-status write so a crash between merge-back and
remote delete does not leave the field `null` while the
remote ref is gone.

If the local repository's object store no longer contains
`work_tip_sha` (e.g., `git gc --prune=now` ran on the local
clone, or the clone was made fresh after the gaia run
completed and the remote work branch was already deleted),
the audit display falls back to "tip unavailable in local
object store." A `success` run's merge commit holds the
work-tip object alive in practice via the second parent
(`git rev-parse <merge_sha>^2` on the typical no-ff merge);
loss of the object is the rare-clone-without-the-merge-commit
case and does not block any v1 flow because continuation
requires the prior run already be promoted to `success`
before it can proceed (§6.2).

**`rescue_ref`** is populated only when the driver (or `gaia
recover`) reconstructed the missing work-branch commits from
rescue artifacts (§6.1 step 8, §7.3). On those paths the
reconstruction lands in a local private ref under
`refs/gaia/recovered/<run_id>/work` — deliberately **not**
`refs/remotes/origin/$GAIA_WORK_BRANCH`, because §6.1.1's
merge-back begins with a force-fetch of the origin work
branch into that tracking ref and a concurrent or subsequent
fetch would silently overwrite the reconstructed tip with
stale or missing origin bytes. Recording the ref name in
`meta.rescue_ref` lets `mergeBack(run_id, {workRefOverride})`
merge the rescue ref directly and skip the origin fetch.
`null` on every non-rescue path. The `refs/gaia/recovered/*`
namespace is driver-local — never pushed — and `gaia
recover` deletes the ref once the run promotes to `success`
(`git update-ref -d`).

The `session_id` and `session_url` fields are written verbatim
from the `/fire` response (`claude_code_session_id` and
`claude_code_session_url` respectively) on the normal path. On
the `fire-ambiguous` path — where the driver never received a
response to parse — `gaia recover` populates `session_id` from
the `.gaia-state/<session_id>.meta.json` artifact written by
the Stop / StopFailure hook alongside the terminal sentinel
(§7.3, §6.4), and **`session_url` stays `null`**: the hook has
no reliable Claude Code environment variable to read the URL
from, so it writes `session_url: null` into the meta artifact
unconditionally. gaia does not synthesize a URL from
`session_id` at recovery time — the URL shape (e.g.,
`https://claude.ai/code/<session_id>`) is a convention
documented for the Claude Code UI rather than a field the
`/fire` endpoint promises verbatim, and gaia treats it as
opaque. Losing the URL on fire-ambiguous recovery is
acceptable because the URL is informational only (for
debugging in the Claude Code UI); the `session_id` is
sufficient for every recovery action and for constructing
the URL manually when needed. Code that displays
`session_url` on the normal path treats it as an opaque
string and does not embed a hardcoded prefix.

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

**Global lock-acquisition order (deadlock avoidance).** gaia
uses three advisory locks: `.gaia/merge.lock` (per-working-
tree, held across §6.1.1's merge-back sequence),
`.gaia/runs/<run_id>/.lock` (per-run, held across each
`meta.json` rewrite), and `.gaia/history.lock` (per-repo,
held across each `history.jsonl` append). A fire path that
acquires the per-run lock and then calls `mergeBack()` inside
it would request `.gaia/merge.lock` with the per-run lock
already held; a concurrent `gaia recover --merge` that
acquired `merge.lock` first and then tried to rewrite
`meta.json` under the per-run lock would deadlock against it.
The normative resolution is:
- **Acquisition order.** When multiple locks must be held
  simultaneously, implementations MUST acquire them in the
  order `merge.lock → run.lock → history.lock`. Any path
  that needs `merge.lock` and `run.lock` together must
  acquire `merge.lock` first.
- **Short-lived `run.lock` holds during merge-back.** The
  §6.1 `handleTerminalSentinel()` wrapper does **not** hold
  `run.lock` across the `mergeBack()` call. Instead, it
  acquires `run.lock` only for short, bounded writes
  (`work_tip_sha` after fetch; each `finalize()` call's
  `meta.status` + history append) and releases it between
  those writes. The merge-back itself runs under
  `merge.lock` alone, then the finalize step re-acquires
  `run.lock`, writes the terminal state, acquires
  `history.lock` while still holding `run.lock`, appends
  the event, releases `history.lock`, then releases
  `run.lock`. `history.lock` is therefore always acquired
  last and released first, which is safe with the
  acquisition order above.
- **No lock spans user I/O.** Stdout prints (the final
  assistant message in §6.1 step 12) run **after** the
  finalize step's meta/history writes have already committed
  and after the best-effort branch cleanup, not while a lock
  is held. This ordering is load-bearing: a stdout EPIPE or
  SIGPIPE from a caller that closed the pipe early (e.g.,
  `gaia -p "..." | head -n1`) would otherwise kill the driver
  before `meta.status` + `history.jsonl` had been written, and
  the run would linger as `pending` despite the merge-back
  having already landed. Implementations catch EPIPE around
  the print and swallow it because the run is already
  finalized; they also install a SIGPIPE handler that ignores
  the signal so the process survives to return its exit code.
- **Crash-safe release.** POSIX advisory locks release on
  process exit, so a crashed driver cannot permanently
  strand any of the three; Windows `LockFileEx` has the
  same semantics via the process handle closure.
The `gaia doctor` shim-release probe (§8.4) exercises the
ordering end-to-end on a synthetic run so a regression that
reintroduces the deadlock fails loudly instead of wedging
production.

**Statuses.** `preparing` and `pending` are non-terminal;
everything else is terminal. Three boolean substates —
`branches_pushed`, `fire_attempted`, `request_body_started` —
further classify what a crashed driver has already done, so
recovery can tell "driver crashed before any push" from "driver
crashed after /fire started writing bytes." The driver sets each
substate under the `.gaia/runs/<run_id>/.lock` immediately after
the corresponding side effect and fsyncs the file before the
side effect proceeds, so a mid-write crash leaves the substate
already flipped to `true` — the conservative bias. `gaia
recover` reads the substates to decide whether to poll the state
branch (branches pushed, session may exist) or short-circuit to
`fire-failed` (no branches on origin, no session could exist).
The per-substate recover actions and the resulting terminal
transitions are **defined normatively in the §6.1.2 status
transition table**; the bullet list below is the
substate-by-substate rationale narrative.

- **`preparing`** — transient, set in §6.1 step 3 *before any
  remote side effect*. `branches_pushed: false`,
  `fire_attempted: false`, `request_body_started: false`. The
  meta write happens before any push; the §6.1 step 5a pushes
  run next, then the lock-guarded flip to `pending` with
  `branches_pushed: true`. Two crash windows result, both covered
  by dedicated rows in the §6.1.2 status transition table:
  (a) crash before the step-5a pushes (no gaia branches on
  origin) — recovery transitions to `fire-failed` and cleans up
  the local record; (b) crash between the step-5a pushes and the
  lock-guarded status flip (branches on origin, meta still
  `preparing`) — recovery probes origin first and, on any gaia
  branch observed, promotes `meta.status` to `pending` with
  `branches_pushed: true, fire_attempted: false` before applying
  `pending`'s recovery actions. The `gaia recover` code path
  implements the probe-then-apply sequence directly rather than
  inferring intent from the `status` field alone, so the same
  recovery run handles both crash windows without bespoke
  per-window logic.
- **`pending` with `branches_pushed: true`, `fire_attempted: false`** —
  branches pushed in §6.1 step 5a, but the
  driver crashed before the `/fire` HTTP call began. No session
  was created; `gaia recover` deletes the origin branches and
  transitions to `fire-failed` (with a synthetic
  `error: "driver-crashed-after-push-before-fire"`). The state
  branch carries only the manifest (no sentinel will ever
  arrive), so polling is unnecessary.
- **`pending` with `fire_attempted: true`, `request_body_started: false`** —
  the driver flipped the substate under the lock right before
  the HTTP client opened the socket, but crashed before any
  `request.write()` / `request.end()` call was attempted (so no
  body byte could have left the userspace buffer). DNS / TCP /
  TLS handshake never produced a session; `gaia recover` treats
  this the same as the row above: no session could exist,
  delete the branches, transition to `fire-failed`.
- **`pending` with `request_body_started: true`, `session_id: null`** —
  the HTTP client's first `request.write()` / `request.end()`
  call was attempted (whether or not bytes actually escaped the
  userspace buffer is not observable). A session may or may not
  have been created (§6.1's ambiguous bucket). `gaia recover`
  polls the state branch for a late sentinel and promotes the
  run to the corresponding terminal state; absent a sentinel
  within the wait window, offers `--assume-failed` or a fresh
  retry with the ambiguous branches preserved for audit. This
  is the row `fire-ambiguous` would cover if the driver had
  completed its transition — treating the crash-before-
  transition case as `pending + request_body_started` is a
  safer default than letting a crash hide a live session.
- **`pending` with a populated `session_id`** — the driver
  received a clean `/fire` response and has started polling.
  `branches_pushed: true`, `fire_attempted: true`,
  `request_body_started: true`. Crash here is the original
  "crashed mid-poll" case: recover polls for a sentinel and
  promotes when one arrives.
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
(§6.1.1 step 7).

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
  "work_tip_sha": "9a8b7c6",
  "continued_from": "session_01HAA..."
}
```

`work_tip_sha` is included in the history record for every
terminal status that has it (every status where the work branch
was fetched, i.e., everything except `preparing`, `pending`
crashes that never reached step 8, `fire-failed`, and
`fire-ambiguous` runs that never resolved). Recording it in
history makes it discoverable from `gaia history` even when the
per-run `meta.json` directory has been swept by `gaia gc
--local-runs`.

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
  buckets. Responses that gaia treats as admission errors —
  400/401/403/404/429, whose documented semantics indicate
  "request rejected before work started" — surface as
  `meta.status = "fire-failed"` per §10.3. The "side-effect-
  free" classification is gaia's inference from admission-error
  semantics, not an explicit server-side guarantee; there is no
  documented idempotency-key header and no written statement
  that these codes never create a session. 5xx
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
  | Definite-not-sent network failure (DNS failure, TCP connect refusal, TLS handshake failure surfaced before any `request.write()` / `request.end()` call was attempted) | n/a | Exponential backoff up to 3 attempts. Then exit 5. | `fire-failed` |
  | Ambiguous network failure (TLS / connection error at or after the first send attempt, connection reset mid-body or mid-response, read-response timeout, 2xx body that fails to parse `claude_code_session_id`) | n/a | **No retry with the same `GAIA_RUN_ID`.** Exit 16. Work/state branches preserved for `gaia recover`. | `fire-ambiguous` |
- The classifier for "definite-not-sent" vs "ambiguous" lives in
  the HTTP client. The observable boundary is **before any
  `request.write()` / `request.end()` call** — any error raised
  before the first such call is definite-not-sent (DNS
  resolution failure, TCP connect refused, TLS handshake
  failure that surfaces before send), so retry with the same
  `GAIA_RUN_ID` is safe. Any error raised at or after the first
  send attempt is ambiguous (the request may be on the wire),
  unless the client received a complete, parseable 4xx status
  envelope from the documented side-effect-free set
  (400/401/403/404/429). The "first send-attempt" boundary is
  what Node's `http` / `undici` stack actually surfaces
  reliably; the prior draft phrased this as "first byte entered
  the socket write buffer," which has no observable callback in
  the client. Some send attempts that never escape the
  userspace buffer will be classified as ambiguous under this
  rule — that is the deliberate conservative bias (over-
  classifying as ambiguous keeps a possibly-live session
  recoverable; under-classifying would lose it). 5xx envelopes
  are ambiguous even when parseable, because the response body
  does not prove no session was created. **Treating 5xx as
  ambiguous is a conservative *product choice*, not a server-
  side guarantee.** Anthropic's docs note `503 overloaded_error`
  as retryable, but `/fire` ships no idempotency-key header and
  no written statement that 5xx leaves no session behind, so
  gaia opts not to retry the same `GAIA_RUN_ID` in this case
  and instead surfaces ambiguity for `gaia recover` to
  resolve. If Anthropic later documents 5xx as side-effect-
  free for `/fire`, a patch release moves these rows back into
  the retryable bucket without a CLI change. gaia errs on the
  side of ambiguous — a failure that cannot be proven
  definite-not-sent or a documented side-effect-free rejection
  is treated as ambiguous, because the cost of a false
  "fire-failed" classification is losing a real running session.
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

The Policy column below names the resulting `meta.status` and
exit code. Branch retention on origin and the per-status
recover action are the **§6.1.2 normative status transition
table**'s responsibility — refer to it as the authoritative
source. This table elaborates the *detection* and *exit-code
mapping* for failures; it is not itself the source of truth
for state transitions.

| Failure | Detection | Policy |
|---|---|---|
| `-p` missing | Arg parsing | Exit 2 with usage. |
| Spawner returns parseable side-effect-free HTTP error | `fire()` rejects with a typed 400/401/403/404/429 envelope | Retry per §11.2 where applicable (429 only). On exhaustion, transition `meta.status = "fire-failed"` (`session_id: null`), append a `"fire-failed"` event to `history.jsonl`, delete the freshly-pushed work / state branches from origin (the documented server contract for these codes is no side effects), exit 10/11/12 per §11.2. |
| Spawner returns 5xx envelope (500 `api_error` / 503 `overloaded_error`) | `fire()` rejects with a typed 5xx envelope | **Treated as ambiguous** per §11.2. No retry with the same `GAIA_RUN_ID`. Transition `meta.status = "fire-ambiguous"`, preserve work / state branches on origin, exit 16. `gaia recover` polls for a late sentinel; see the ambiguous row below for the resolution flow. |
| Definite-not-sent network failure | DNS / TCP / TLS error before any `request.write()` / `request.end()` call was attempted | Retry with same `GAIA_RUN_ID` per §11.2. On exhaustion, `meta.status = "fire-failed"`, branches deleted, exit 5. |
| Ambiguous `/fire` failure — session may or may not have been created | Any failure at or after the first send attempt; 2xx body that fails to parse `claude_code_session_id` | **Never retry with the same `GAIA_RUN_ID`.** Transition `meta.status = "fire-ambiguous"` (`session_id: null`), append a `"fire-ambiguous"` event, **preserve** work/state branches on origin, exit 16. `gaia recover <run_id>` polls the state branch for a late sentinel (§6.4); if one arrives, the run promotes to `"success"` / `"failed"` / `"aborted"` as if the driver had received the sentinel inline. |
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
| `--resume` / `-c` of a non-success run | Prior `meta.status` is anything other than `success` | Exit 2 naming the run. For `{no-merge, conflict, timeout, push-rejected}` the message points at `gaia recover <run_id>` (which promotes to `success` on completion). For `{aborted, failed}` it points at `gaia recover <run_id> --merge` (explicit opt-in to land the work). For `fire-failed` and `fire-ambiguous` there is no recovery path that produces a resumable success: `fire-failed` has no session, and `fire-ambiguous` requires `gaia recover` to poll for a late sentinel first. |
| `--resume <session_id>` on a success run whose merge is not in current HEAD | Ancestry check `git merge-base --is-ancestor <work_tip_sha> HEAD` fails (§6.2 cross-branch guard) | Exit 2 naming the recorded `meta.user_branch`. Caller switches to that branch (or merges it into the current branch manually) before retrying. v1 has no opt-in to bypass this; cross-branch replay is §14 work. |
| Continuation transcript missing | `.gaia/transcripts/<session_id>.md` absent | Exit 2 with a message. Can happen if the transcripts directory was deleted manually. |

### 12.1 Cleanup

The per-status branch-retention rules are normatively defined
by the **§6.1.2 status transition table**'s "Work branch on
origin" and "State branch on origin" columns; this section
expands the rationale and lists the operational sweep paths
(`gaia gc`, `gaia gc --local-runs`). If a sentence here
conflicts with the table, the table wins.

Default: delete `claude/gaia/$RUN_ID/{work,state}` from origin
immediately after a successful merge. Transcripts have already been
copied locally to `.gaia/transcripts/` at step 9 of §6.1, so the state
branch is disposable at that point. If the delete push fails after
a successful merge (auth glitch, branch protection, network blip),
the run stays `"success"` per §6.1 step 12's branch-cleanup substep — leftover refs are
swept by `gaia gc` and never downgrade the status. Only failures
that affect whether the code merged (`conflict`, `push-rejected`)
or whether the session finalized (`timeout`, `failed`, `aborted`)
produce non-`success` statuses.

**Local ref cleanup.** §6.1 step 12's branch-cleanup substep deletes the *origin* refs
on `success` runs; the corresponding **local** branches
(`claude/gaia/<run_id>/work` and `claude/gaia/<run_id>/state`
in the driver's clone, and the local state-branch refs the
hooks create via `git worktree add -B`) are also pruned by
the same finalize path: `git branch -D` on the local refs
runs after the origin delete push on `success`, and after
the `fire-failed` transition on a driver-side cleanup. On
`fire-failed` both local branches are deleted (whether or
not the origin pushes landed) because no session could have
produced recoverable state. On every other terminal status
(`no-merge` / `conflict` / `push-rejected` / `timeout` /
`failed` / `aborted` / `fire-ambiguous`) the local refs are
**preserved** so `gaia recover` can find them without a
fresh fetch. `gaia gc --local-refs [--max-age <d>]` sweeps
local refs older than `<max-age>` independent of origin
state, mirroring `gaia gc --local-runs` for the
`.gaia/runs/*` directories; the remote-ref sweep is
`gaia gc` itself.

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

**Rescue branches (`claude/gaia/<run_id>/rescue/<session_id>`).**
These refs are pushed by the §8.2 wrong-branch Stop fallback when
a session ended on an unexpected branch, and they may carry the
only copy of the session's commits. `gaia gc` deletes a rescue ref
only when:

- The associated run's `meta.json` is present locally and its
  status is terminal (any of `success`, `failed`, `aborted`,
  `push-rejected`, `conflict`, `timeout`, `no-merge`, `fire-
  failed`, `fire-ambiguous`), **and**
- The age resolution for the rescue ref (same priority order as
  the work/state pair: meta.json → manifest → ULID → committer-
  date on the rescue ref itself) exceeds `--max-age`.

A terminal-status requirement is deliberate — rescue refs exist
precisely for the case where the driver / recovery tool has not
yet inspected them, and deleting one whose run has not been
classified as terminal throws away work the user may still want.
If no local `meta.json` is found for the run id (e.g., rescue ref
pushed from a peer's clone), gc leaves the ref in place and lists
it under `--dry-run` as `unclaimed rescue` so the operator sees
it.

**Doctor protocol runs (`claude/gaia/<ulid>/*` with
`manifest.run_type: "doctor"`).** Doctor runs share the
`claude/gaia/<ulid>/{work,state,binding,...}` branch shape with
real runs (the `UserPromptSubmit` regex requires it; §7.2,
§8.4). gaia gc identifies them by reading `manifest.run_type`
from the state branch's `.gaia-state/manifest.json` (or
`meta.run_type` from a local `.gaia/runs/<run_id>/meta.json`
when present) and applies an **aggressive 24-hour age policy**
regardless of `--max-age` — doctor completes within seconds and
leaves no pending work on its refs. A short window is fine
because the run cannot enter a recoverable terminal state that
needs the branches preserved (any failure surfaces as a doctor
report immediately and the run is discarded). Stretching doctor
refs out for the full `--max-age` would accumulate unused refs
that clutter `git ls-remote`. If the manifest is unreachable
(state branch already gc'd, partial push), gc falls back to
treating the orphan branch under the standard work/state-pair
policy at the top of this section — the doctor-specific 24-hour
shortcut requires positive identification via `run_type`.

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
- **Structural allow-list for skills / commands / subagents.**
  v1 refuses on presence any file under `.claude/skills/**`,
  `.claude/commands/**`, or `.claude/agents/**` (§2, §5, §8.7).
  v1.x lifts this by shipping a structural scan that accepts
  files only when their frontmatter has **no** `allowed-tools`,
  **no** `hooks:`, **no** `agent:`, **no** `context: fork`,
  **no** alternate `shell`, and the body contains **no**
  leading-`!` lines or `` `!…` `` preprocessor blocks. The
  `disableSkillShellExecution: true` project-settings entry
  that gaia v1 already writes (§8, §8.8) stays in place and
  continues to disable inline shell preprocessing across
  project, user, plugin, and additional-directory load
  surfaces — the allow-list gates the content shape, the
  settings key gates the runtime behavior. The scan is paired
  with a `gaia doctor --cloud` probe that re-runs the same
  check against the work-branch tree the VM sees so
  newly-committed skill files cannot slip past `gaia init`.
  Adding these surfaces to a gaia repo remains an opt-in
  per-repo decision even after the v1.x allow-list lands; the
  check lives behind a config key (`allow_pure_skill_content:
  true`) so repos that never want to expose the surface stay
  on the v1 hard-ban posture. Not in v1 because the allow-list surface area is
  wide enough (six frontmatter fields, two body-content
  patterns, one managed-settings key) to deserve its own soak
  time, and the hard-ban is a safe default to ship on top of.
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
- **Cloud-upgrade automation.** `gaia cloud-upgrade` (§6.4)
  emits paste-ready setup-script text in v1 because no public
  routine/environment configuration API is documented. If
  Anthropic ships such an API, a future release can edit the
  setup script in place under a confirmation prompt — the
  command's argument shape stays the same. A further release
  could probe the cache freshness from the driver side (e.g.,
  by firing a no-op doctor session and inspecting the returned
  binary version) and surface a `your VM is still on v1.2.3,
  the script has been bumped to v1.2.4, the cache will roll
  over by <date>` banner. Purely informational; v1's "exact
  pin, paste to upgrade, wait for cache roll-over" model
  already works.
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
- **OS-level sandbox default-flip.** `gaia init --hardened`
  ships in v1 as an opt-in; the flag and the emitted
  `sandbox.*` settings block are documented normatively in
  §16.1 as part of the v1 security posture. The v1.x entry
  tracked here is the **default-flip**: once the sandbox
  surface is uniformly available across supported Claude Code
  builds, the default becomes "hardened unless the operator
  opts out," and `gaia init` without a flag emits the sandbox
  block. The v1.x release is expected to take `PreToolUse`
  from "catches the overwhelming majority of accidents" to
  "catches accidents and most adversarial evasion" by default.
  Until the default flips, §16.1's trust model applies to any
  repo that did not run `gaia init --hardened`.
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

**Status: lab-quality. Not part of the v1 stable contract.**
**Ships in v1 behind `--experimental-mode`**, off by default, and
prints a warning on start. The flag is present in the v1 CLI
(§6.3) and has v1 doctor coverage (§15.7), but the feature itself
is preview-quality: it may be removed, redesigned, or disabled
without a major-version bump, and it is not part of the CLI-shape
stability promise (§1.1). Depends on current `Stop`-hook block-
decision semantics, which are sparsely documented and may change
between Claude Code versions. When possible, finalizes using the
same terminal sentinels as the stable fire-and-forget path
(§15.3 covers each case); on hook crashes and VM wall-clock
timeouts the run leaves no sentinel and recovery goes through
the standard `gaia recover <run_id>` flow rather than finalizing
automatically. Intended to graduate to stable in a later v1.x
release once the constraints in §15.4 are characterized and
`gaia doctor` can certify them.

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
| Hook timeout (9 min without `GAIA_INPUT`) | Hook runs the final sweep so no work is silently lost from the VM, then writes `.gaia-state/<session_id>.md` (last assistant message + a "user did not reply within 9 minutes" marker) and `.gaia-state/<session_id>.transcript.md`, and pushes `GAIA_ABORTED: <session_id>`. | Driver reads the abort sentinel and exits 0 with `status: "aborted"`. **Does not merge.** Work branch preserved on origin. The supported completion path is `gaia recover <run_id> --merge` to land the partial work and promote the run to `success`; after promotion, `gaia -c` / `gaia --resume <session_id>` resumes normally per §6.2's success-only rule. Direct `gaia -c` against an `aborted` run is refused. The previous draft auto-merged on input timeout, which silently landed half-done work after a clarification went unanswered — never the right default for an interactive surface. |
| Hook exits non-zero (git error, JSON schema drift) | Session terminates abnormally. No sentinel pushed. | Driver times out. Exits 3. Recovery via `gaia recover <run_id>`. |
| `StopFailure` fires instead of `Stop` (API error) | `GAIA_FAILED:` sentinel pushed; no pending turn. | Driver exits 13. `gaia recover` surfaces the run; the supported continuation path is `gaia recover <run_id> --merge` to promote to `success`, after which `gaia -c` / `gaia --resume <session_id>` resumes normally (§6.2). Direct `-c` against the `failed` run is refused. |
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

**OS-level sandbox hardening — `gaia init --hardened` (v1
opt-in).** gaia ships a first-class opt-in that extends the
`PreToolUse` parser's best-effort deny surface with Claude
Code's OS-level sandbox. Passing `--hardened` to `gaia init`
(§6.4 admin commands) emits an additional `sandbox` block into
the committed `.claude/settings.json`:

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

Claude Code's sandbox restrictions apply to **subprocess
commands**, not only to Claude's file tools — so this closes
the `eval` / Python / background-process bypass that §8.5's
Bash parser explicitly cannot handle. A hostile agent that
sequences base64-encoded command strings through `eval`, or
writes a helper script to `/tmp/x.sh` and executes it, or
invokes Python / Node / Perl as a shell alternative, cannot
mutate gaia-owned paths because the OS-level sandbox denies
the syscall regardless of how the subprocess was launched.

`failIfUnavailable: true` is deliberate: if the Claude Code
build in the cloud VM (or on a teammate machine) lacks the
sandbox runtime, the session fails fast instead of silently
running without OS-level protection. This is the safer
composition — a hardened repo's operator should know when the
hardening is not in effect.

The flag is **opt-in in v1** because the OS sandbox surface is
not uniformly available on every driver / teammate / VM
combination, and a default-on hardened profile would break
teammate environments whose Claude Code install lacks sandbox
support. v1.x will tighten the default to opt-out once the
surface is broadly available; the current `--hardened` flag is
the migration path. Until the default flips, §16.1's trust
model applies to any repo that did not run `gaia init
--hardened`. The v1.x tracking entry is in §14; the flag and
the emitted settings block are part of the v1 surface.

Recommended posture: production repos with CI / automation
that fire gaia should run `gaia init --hardened` and set
`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` on the cloud environment
(§5). Together those close both the "agent escalates from
Bash to OS-level writes against gaia paths" gap and the
"prompt-injected Bash exfiltrates environment credentials"
gap. Neither is a full sandbox, but the combination is
materially stronger than the `PreToolUse` parser alone.

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
