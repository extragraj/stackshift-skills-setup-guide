# StackShift Setup Guide

Updated installation, configuration, and usage guide for **stackshift-workflow-skills** (`@extragraj/stackshift-skills`, currently 0.8.0) and **ui-forge** (`@extragraj/ui-forge`, currently 1.6.8) in a StackShift project.

> **What changed?** Both skills had major CLI refactors. Stackshift moved to a CLI-owned bootstrap (no more deferred-to-agent mode, no more tier-bundle folders, the skill folder is now `stackshift/` not `stackshift-core/`). UI Forge replaced `npx skills add extragraj/ui-forge` and `scripts/cli.js install` with a first-party CLI: `npx @extragraj/ui-forge init`. The legacy install paths still appear in older docs around the web — ignore them and use the commands below.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install stackshift-workflow-skills](#2-install-stackshift-workflow-skills)
3. [Install ui-forge](#3-install-ui-forge)
4. [Run the Forge Scan](#4-run-the-forge-scan)
5. [Verify the Bootstrap](#5-verify-the-bootstrap)
6. [Day-to-Day Commands](#6-day-to-day-commands)
7. [Fixing Legacy Installations](#7-fixing-legacy-installations)

---

## 1. Prerequisites

- Node.js ≥ 18 (ESM required by both skills' runtime scripts)
- An agentic CLI / IDE: Claude Code, Cursor, Codex, Cline, Gemini, Copilot, or any tool that supports the `.agents/` or `.claude/` skill convention
- Run every command from your **StackShift project root** (the folder that contains `package.json` and where `.stackshift/` should live)

---

## 2. Install stackshift-workflow-skills

The CLI now performs the **full bootstrap end-to-end** in a single pass — protocol materialization, project infrastructure, `design/standards/` seeding, `.forgeignore`, and UI Forge integration. There is no deferred-to-agent mode anymore. The legacy `--no-materialize` flag was removed in 0.3.0 and exits with an error if passed.

### Interactive (recommended)

```bash
npx @extragraj/stackshift-skills init
```

This walks you through:

1. **Protocol tier** — `required`, `recommended` (default), `full`, or a `custom` checkbox selection
2. **Install scope** — `project` (default) or `global`
3. **Platform** — pick one or more of `agents`, `claude`, `copilot`, `gemini`, `cursor`
4. **Seed strategy** — optional; e.g. `initialvalue-seeding` or `none` (default)

When it finishes you'll see a summary listing the materialized protocols, the seeded `design/standards/`, and (if UI Forge was already installed) the paired-mode integration steps.

### Non-interactive

```bash
# Defaults (recommended tier, project scope, agents platform)
npx @extragraj/stackshift-skills init --no-interactive

# Specific tier + multi-platform
npx @extragraj/stackshift-skills init \
  --tier full \
  --scope project \
  --platform agents,claude \
  --no-interactive

# With a seed strategy
npx @extragraj/stackshift-skills init \
  --seed initialvalue-seeding \
  --no-interactive
```

### Available flags

| Flag | Values | Default | Description |
|---|---|---|---|
| `--tier` | `required`, `recommended`, `full` | `recommended` | Protocol tier to install (`custom` requires interactive mode) |
| `--scope` | `project`, `global` | `project` | Install location |
| `--platform` | `agents`, `claude`, `copilot`, `gemini`, `cursor` (csv) | `agents` | Platform(s); comma-separated for multiple |
| `--seed` | seed id or `none` | `none` | Seeding strategy (invalid ids are rejected upfront) |
| `--no-interactive` | flag | off | Skip prompts and use flags + defaults |
| `--help`, `-h` | flag | — | Show help |

> Tiers are cumulative: `recommended` includes `required`, `full` includes everything.

### Platform skill roots

| Platform | Project | Global |
|---|---|---|
| `claude` | `.claude/skills/` | `~/.claude/skills/` |
| `copilot` | `.github/skills/` | `~/.copilot/skills/` |
| `agents` | `.agents/skills/` | `~/.agents/skills/` |
| `gemini` | `.agents/skills/` | `~/.gemini/antigravity/skills/` |
| `cursor` | `.cursor/skills/` | `~/.cursor/skills/` |

### What the install writes

- The skill folder is now named **`stackshift/`** (renamed from `stackshift-core/` in 0.6.0). Only the protocols you selected land in the destination — the in-skill `protocols/_registry.json` is pruned to match. CLI:PROTOCOLS / CLI:SEED / CLI:CROSSCUT marker regions in `SKILL.md` and `workflow/*.md` are rewritten with your installed set, so the agent never reads files for protocols you didn't install.
- `.stackshift/installed.json` — source of truth for tier, protocols, seed, skill version, a11y, and UI Forge integration state
- `.stackshift/protocols/` — materialized copies you can edit; `repair` and `init` never overwrite these
- `.stackshift/references/README.md`
- `.stackshift/injection-map.json` — a per-protocol record of where each installed protocol got injected
- `design/standards/stackshift-section-variants.md`
- `.forgeignore` — created if missing; appended (no data loss) if present

---

## 3. Install ui-forge

UI Forge has a brand-new first-party CLI as of 1.6.0. The pre-1.6.0 paths (`npx skills add extragraj/ui-forge` and `node …/scripts/cli.js install`) are no longer recommended — see [Section 7](#7-fixing-legacy-installations) if you have those already.

Install from your StackShift project root:

```bash
npx @extragraj/ui-forge init
```

This single command:

1. Detects StackShift via `.stackshift/installed.json` and **auto-enables paired mode** (you don't pass `--pair` manually)
2. Copies only the runtime files each selected feature needs into the platform skill dir
3. Writes per-platform slash commands (`/forge-scan`, `/forge`, `/forge-verify`, `/forge-export-design`, `/forge-handoff`) with `# Generated by ui-forge@<version>` provenance headers
4. Patches `permissions.allow` in `.claude/settings.json` (or the platform equivalent) with one entry per script — least-privilege, POSIX paths, `.bak` backup created
5. Detects and patches MCP client configs (Claude Code, Cursor, Codex, Cline) across Windows / macOS / Linux when `mcp-server` is selected
6. Writes a portable `./ui-forge.mjs` shim at the project root (safe to commit; resolves `SKILL_ROOT` at runtime)
7. Writes `.ui-forge/installed.json` — the lockfile that drives `repair`, `update`, `doctor`, and `uninstall`
8. Optionally runs a quick scan to seed `design/design-arch.json` immediately

### Interactive prompts (in order)

The prompt order was finalized in 1.6.2:

1. **Required features** — `scan`, `forge` are always installed (informational note)
2. **Optional features** — multiselect across `verify`, `export-design`, `fetch-handoff`, `mcp-server`, `post-tool-verify-hook`, `project-cli`
3. **Theme** — pick `stackshift` for StackShift projects (or `shadcn`, `mantine`, `plain-tailwind`, `none`)
4. **Quick scan** — run `scan.js --quick` immediately after install (default Yes when a theme is selected)
5. **Platforms** — agentic platforms to wire (renamed from "Target Platforms" in 1.6.2)
6. **Scope** — `project` (default) or `global`

> Re-running `init` later detects the existing lockfile and uses your previous selections as defaults, so adding or removing a feature is one prompt away.

### Non-interactive (CI / recommended StackShift full install)

```bash
npx @extragraj/ui-forge init --yes \
  --scope=project \
  --platforms=claude \
  --features=scan,forge,verify,mcp-server,export-design,fetch-handoff,post-tool-verify-hook,project-cli \
  --theme=stackshift \
  --pair=on \
  --theme-override \
  --quick-scan=on
```

### `init` flags

| Flag | Values | Description |
|---|---|---|
| `-y, --yes` | — | Accept defaults non-interactively |
| `--scope` | `project`, `global` | Install location |
| `--platforms` | csv: `claude`, `cursor`, `agents`, `codex`, `copilot`, `gemini` | Agentic platforms to wire |
| `--features` | csv: `scan`, `forge`, `verify`, `export-design`, `fetch-handoff`, `mcp-server`, `post-tool-verify-hook`, `project-cli` | Feature set (scan + forge always added) |
| `--theme` | `shadcn`, `mantine`, `plain-tailwind`, `stackshift`, `none` | Use `stackshift` for StackShift projects |
| `--pair` | `auto` (default), `on`, `off` | Pairing detection; `auto` reads `.stackshift/installed.json` |
| `--mcp` | `on`, `off` | Enable/disable MCP wiring (prefer `mcp-server` in `--features`) |
| `--mcp-clients` | csv | Subset of detected clients to wire |
| `--quick-scan` | `on`, `off` | Run a quick scan right after install |
| `--theme-override` | flag | StackShift only — rewrite `globals.css` + `tailwind.config.*` with StackShift tokens before scanning |
| `--no-backup` | flag | Skip `.bak` files when `--theme-override` runs |
| `--force-forgeignore` | flag | Overwrite a user-owned `.forgeignore` on re-install |
| `--prune-unknown` | flag | Auto-delete unknown files during legacy sweep |
| `--dry-run` | flag | Print the plan, write nothing |

> **`--theme stackshift` is the correct flag for StackShift projects.** It forces `isStackShift: true` in `design-arch.json`, which makes the StackShift UI conventions inject at generation time even on sparse codebases.

> **Limited mode:** if you run UI Forge without StackShift in the project, the stackshift theme runs in limited mode — `themeOverride` is stripped, paired signals are removed, `_limited: true` is set in the lockfile, and the default `.forgeignore` template is used. Inside a StackShift project, pairing is auto-detected and you get the full template.

---

## 4. Run the Forge Scan

The Forge Scan indexes design tokens, component patterns, and variant structures so UI Forge can generate accurately. For StackShift projects this **must** run with `--theme stackshift`. If you set `--quick-scan=on` (or answered Yes to the prompt) during `ui-forge init`, the scan has already run and you can skip ahead.

### Via slash command (recommended)

```
/forge-scan --theme stackshift
```

Works in Claude Code, Cursor, AntiGravity, and any agentic platform with slash command support. The model executes it, writes `design/design-arch.json`, and registers results with the skill context.

### New StackShift projects (empty globals.css / tailwind.config)

```
/forge-scan --theme stackshift --theme-override
```

`--theme-override` surgically rewrites three sections in your project files **before** the scan reads them:

- The Google Fonts `@import` in `globals.css` → Inter font
- The `@layer base { }` block in `globals.css` → full HSL CSS variable token set (light + dark)
- The `theme.extend { }` section in `tailwind.config.*` → colors, border radius, spacing, font families, sizes, weights

`.bak` files are created by default. Pass `--no-backup` to skip them. The command is idempotent — running it twice produces the same result.

Use `--theme-override` when your project is fresh or unaligned to StackShift conventions. Omit it for established projects with existing tokens.

### Without slash command support

After `ui-forge init` you have a portable shim at the project root:

```bash
node ui-forge.mjs scan --theme stackshift
# new projects:
node ui-forge.mjs scan --theme stackshift --theme-override
```

The shim resolves `SKILL_ROOT` at runtime against your local install, so it's safe to commit to the repo.

### Two-phase scan (AI-agnostic)

As of UI Forge 1.5.0, `/forge-scan` is a two-phase process:

1. **Phase 1** — pure static analysis, always runs, writes `design/design-arch.json`
2. **Phase 2** — the session AI (whichever model is active) reads files listed in `design/.synthesis-request.json` and synthesizes spacing, typography, and color tokens; the model calls `scripts/apply-synthesis.js` to validate and patch the arch file

Pass `--quick` to skip Phase 2 and leave patterns as `'unknown'`. No subprocess, no API key, no `claude` binary required — works in Claude Code, Cursor, Codex, Cline, Gemini, and anywhere else.

---

## 5. Verify the Bootstrap

After both installs complete, your project should have:

| File | Owner | Purpose |
|---|---|---|
| `.stackshift/installed.json` | StackShift | Tier, protocols, seed, version, a11yRequired, uiForgeIntegration |
| `.stackshift/protocols/` | You (materialized) | Edit freely; never overwritten |
| `.stackshift/injection-map.json` | StackShift | Per-protocol injection record |
| `.ui-forge/installed.json` | UI Forge | Lockfile: `writtenByFeature`, `patched`, `pruned`, `summary` |
| `design/design-arch.json` | UI Forge | Scan output: tokens, components, standards |
| `design/standards/stackshift-section-variants.md` | StackShift | Section variant standards |
| `design/standards/stackshift-ui.md` | UI Forge | UI standards bridged from the stackshift theme |
| `.forgeignore` | Mixed | Default + paired-mode patterns |
| `.claude/settings.json` (or platform equiv) | Both | `permissions.allow` entries; PostToolUse hooks when active |
| `./ui-forge.mjs` | UI Forge | Project-root shim (safe to commit) |

Sanity-check the install summaries:

```bash
npx @extragraj/stackshift-skills repair    # idempotent — confirms install integrity
npx @extragraj/ui-forge ls                  # one-screen summary of UI Forge install
npx @extragraj/ui-forge version             # bundled skill version + source location
```

If you have the auto-validate hook (`auto-validate-hook` protocol) and/or `post-tool-verify-hook` feature enabled, open `.claude/settings.json` and confirm `hooks.PostToolUse` contains the corresponding entries with `matcher: "Write|Edit"`.

---

## 6. Day-to-Day Commands

### stackshift-workflow-skills

| Command | Purpose |
|---|---|
| `npx @extragraj/stackshift-skills init` | Install or re-configure (interactive by default) |
| `npx @extragraj/stackshift-skills repair` | Reconcile materialized protocols, purge legacy artifacts, refresh marker injection |
| `npx @extragraj/stackshift-skills validate --file <path>` | Lint a single file for protocol violations |
| `npx @extragraj/stackshift-skills validate --json` | Lint the whole project, JSON output (for CI) |
| `npx @extragraj/stackshift-skills validate --hook` | PostToolUse hook entry point (reads payload from stdin) |

### ui-forge

| Command | Purpose |
|---|---|
| `npx @extragraj/ui-forge init` | Install or modify wiring (re-run to add/remove features) |
| `npx @extragraj/ui-forge repair` | Re-apply wiring from `.ui-forge/installed.json` (non-interactive) |
| `npx @extragraj/ui-forge doctor` | Diagnose the install; `--fix` deletes legacy files and prunes stale wiring |
| `npx @extragraj/ui-forge ls` | One-screen summary of current install |
| `npx @extragraj/ui-forge version` | Print bundled skill version + source location |
| `npx @extragraj/ui-forge uninstall` | Remove everything UI Forge wrote (tracked via lockfile) |
| `npx @extragraj/ui-forge migrate` | One-shot migration from a pre-1.6.0 install |
| `npx @extragraj/ui-forge mcp-config` | Print MCP snippet for manual wiring |

### Slash commands wired by ui-forge

| Command | Description |
|---|---|
| `/forge-scan` | Scan project → `design/design-arch.json` |
| `/forge --task "..." --refs <path> --output <path>` | Prepare generation context; AI generates the component |
| `/forge-verify <component.tsx> <contract.ts>` | Verify a generated component against its contract |
| `/forge-export-design` | Export design system as a Claude Design–ingestible bundle |
| `/forge-handoff <url>` | Fetch a Claude Design handoff URL and materialize refs locally |

### Project-root shim (`./ui-forge.mjs`)

```bash
node ui-forge.mjs --help
node ui-forge.mjs --version
node ui-forge.mjs scan --theme stackshift
node ui-forge.mjs forge --task "Convert hero" --refs ./hero.html --output ./Hero.tsx
node ui-forge.mjs verify ./Hero.tsx ./types.ts
node ui-forge.mjs export
node ui-forge.mjs handoff <url>
node ui-forge.mjs mcp        # run the MCP server (stdio)
```

Exit codes: `0` success, `1` runtime error, `2` usage error.

---

## 7. Fixing Legacy Installations

The biggest sources of confusion right now are installs that predate the recent refactors. Both CLIs have automated cleanup — in most cases you don't need to manually delete anything.

### 7.1 Legacy stackshift-workflow-skills installs

**Symptoms** (any one of these means you have a pre-current install):

- `.agents/skills/stackshift-core/` exists (renamed to `stackshift/` in 0.6.0)
- `.agents/skills/stackshift-protocols-required/`, `…-recommended/`, or `…-full/` folders exist (tier-bundle folders removed in 0.3.0)
- `.agents/skills/stackshift-seed-initialvalue/` or other `stackshift-seed-*` stub folders exist (removed in 0.5.1)
- `.stackshift/installed.json` contains `"bootstrapRequired": true` or `"materializationDone": true` (legacy marker flags, stripped on every install in 0.3.0+)
- Your install scripts pass `--no-materialize` (removed in 0.3.0 — exits with error)
- Your install scripts pass `--platform codex` (removed in 0.1.9B — exits with error; use `--platform agents`)
- `skills-lock.json` has entries starting with `stackshift-protocols-`, `stackshift-seed-`, or `stackshift-core`

**Fix — one command:**

```bash
npx @extragraj/stackshift-skills repair
```

Repair runs four passes and is fully automatic:

1. **Legacy artifact purge** — removes any pre-0.3.0 `stackshift-protocols-*` folders, pre-0.5.1 `stackshift-seed-*` stub folders, and pre-0.6.0 `stackshift-core/` folders across every platform skill root (`.agents/`, `.claude/`, `.cursor/`, `.github/`, `.codex/`). Strips matching entries from every `skills-lock.json`. Strips `bootstrapRequired` and `materializationDone` from `.stackshift/installed.json`. If `stackshift-core/` was present and `stackshift/` was not, the new folder is re-copied from the shipped skill source.
2. **Seed validation** — checks that the `seed` recorded in `installed.json` matches a known strategy in the seed registry.
3. **Materialized protocol reconciliation** — compares `.stackshift/protocols/` against `installed.json`. Removes orphans; restores missing recorded protocols.
4. **Workflow marker injection** — rewrites the CLI:PROTOCOLS / CLI:SEED / CLI:CROSSCUT regions in `SKILL.md` and `workflow/*.md` from the current `installed.json`. Also refreshes `SKILL.md` and every `workflow/*.md` from the shipped skill source so stale pre-0.7.0A content (Required actions / Recommended actions blocks etc.) is cleared.

If you had `--no-materialize` or `--platform codex` baked into a script, just remove the flag (or change `codex` → `agents`) and re-run `init`.

**If repair does not fully resolve it:** re-run `init` explicitly to refresh your selections. The next `init` performs the same legacy purge plus a fresh install.

### 7.2 Legacy ui-forge installs

**Symptoms:**

- UI Forge was installed via `npx skills add extragraj/ui-forge` or `npx skills add extragraj/ui-forge -a claude-code -y -g`
- The skill folder contains `scripts/cli.js`, `examples/`, `tests/`, `change-logs/`, `CLAUDE.md`, or `cli/src/` (dev assets that were shipped accidentally in the old install path)
- No `.ui-forge/installed.json` lockfile at the project root
- No `./ui-forge.mjs` shim at the project root
- Your install command included `node "$d/skills/ui-forge/scripts/cli.js" install` or similar `cli.js install` invocations
- Slash command files (`forge-scan.md`, `forge.md`, etc.) exist in `.claude/commands/` or `.agents/commands/` but no lockfile records them

**Fix — one command:**

```bash
npx @extragraj/ui-forge migrate
```

`migrate` detects pre-existing `commands/forge-*.md` files in the platform commands dir, infers your feature set from them, writes `.ui-forge/installed.json`, and runs the legacy sweep. It's idempotent — re-running on a migrated install is a no-op.

The legacy sweep removes:

- `examples/`, `tests/`, `change-logs/`, `tmp/`, `CLAUDE.md` (shipped by `npx skills add`)
- `scripts/sync-version.mjs`, `scripts/test-mcp.mjs`, `scripts/cli.js`
- `cli/src/`, `cli/tsconfig*` (accidental tarball inclusions)
- `themes/*.json` that don't match your selected theme
- `*.bak`, `*.tmp`, `.DS_Store`, `Thumbs.db`
- Stale slash command files no longer in your selected feature set
- Stale permission entries pointing into the skill dir

After migrate, run:

```bash
npx @extragraj/ui-forge doctor
```

If diagnostics surface anything (a missing file, an out-of-date provenance header, a stale permission entry), run:

```bash
npx @extragraj/ui-forge doctor --fix
```

This runs a full repair after diagnosis.

### 7.3 Pre-1.6.0 → 1.6.1 hook fix

If you originally installed UI Forge on 1.6.0 (where the PostToolUse hook shape was rejected by Claude Code), re-run `init` to migrate:

```bash
npx @extragraj/ui-forge init
```

The legacy 1.6.0 hook entry (object-typed `matcher`, top-level `id`) is migrated in place to the canonical Claude Code schema (`{matcher: "Write|Edit", hooks: [{type: "command", command: "..."}]}`) via a sibling `_uiForgeId` marker. Unrelated user hooks are preserved. The hook command env var is also corrected from `$CLAUDE_FILE_PATH` (which was never exported) to the canonical `$CLAUDE_TOOL_INPUT_file_path`.

### 7.4 Verifying the fix

After running migrate / repair / doctor on either skill:

```bash
npx @extragraj/stackshift-skills repair   # idempotent; should report no changes on a clean install
npx @extragraj/ui-forge ls                 # confirms lockfile + feature set
npx @extragraj/ui-forge doctor             # read-only diagnosis
```

Open `.stackshift/installed.json` and confirm `skillVersion` matches your installed CLI (e.g. `0.8.0`) and there is no `bootstrapRequired` or `materializationDone` field.

Open `.ui-forge/installed.json` and confirm `lockfileVersion: 2`, the `summary` block lists your features, and `writtenByFeature` has the expected groups (`always`, `scan`, `forge`, `verify`, `theme`, `commands`, `project-cli`, `forgeignore`, plus any optional features you chose).

### 7.5 Nuclear option — clean slate

If automated repair/migrate doesn't get you to a clean state (which should be rare):

```bash
# 1. Uninstall ui-forge (lockfile-driven, only removes what it wrote)
npx @extragraj/ui-forge uninstall

# 2. Manually remove stackshift state
rm -rf .stackshift/
rm -rf .agents/skills/stackshift .agents/skills/stackshift-core .agents/skills/stackshift-protocols-*
rm -rf .claude/skills/stackshift .claude/skills/stackshift-core .claude/skills/stackshift-protocols-*
# repeat for any other platform dirs you use: .cursor/, .github/, .gemini/, .codex/

# 3. Optional — remove design state (you'll need to rescan)
rm -f design/design-arch.json design/.synthesis-request.json
rm -rf design/.handoff-cache/ design/claude-design-bundle/

# 4. Reinstall both, stackshift first
npx @extragraj/stackshift-skills init
npx @extragraj/ui-forge init
```

Reinstalling stackshift before ui-forge matters: ui-forge auto-detects pairing via `.stackshift/installed.json` during `init`, so the order ensures paired mode is wired automatically.

---

## Quick reference card

```bash
# Fresh StackShift project — full setup
npx @extragraj/stackshift-skills init --tier recommended --no-interactive
npx @extragraj/ui-forge init --yes \
  --features=scan,forge,verify,mcp-server,project-cli,post-tool-verify-hook \
  --theme=stackshift --pair=on --quick-scan=on

# After any drift, manual edits, or CI restore
npx @extragraj/stackshift-skills repair
npx @extragraj/ui-forge repair

# Coming from a pre-0.6.0 stackshift or pre-1.6.0 ui-forge install
npx @extragraj/stackshift-skills repair
npx @extragraj/ui-forge migrate
npx @extragraj/ui-forge doctor --fix
```
