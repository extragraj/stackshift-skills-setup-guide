# StackShift Skills Setup Guide

Complete installation, configuration, and usage guide for **stackshift-workflow-skills** and **ui-forge** in a StackShift project environment.

---

## Table of Contents

1. [Installing stackshift-workflow-skills](#1-installing-stackshift-workflow-skills)
2. [Pre-Bootstrap & Materialization](#2-pre-bootstrap--materialization)
3. [Installing ui-forge](#3-installing-ui-forge)
4. [Running the Forge Scan](#4-running-the-forge-scan)
5. [Completing the Bootstrap](#5-completing-the-bootstrap)
6. [Use Cases & Examples](#6-use-cases--examples)
7. [Crafting Effective Prompts](#7-crafting-effective-prompts)

---

## 1. Installing stackshift-workflow-skills

> **Recommended:** Use the interactive CLI (Option A). It handles tier selection, platform targeting, materialization, and conflict prevention in a single guided flow. It is the most reliable path and the one this guide emphasizes.

### Option A — Interactive CLI (Recommended)

Run from the root of your StackShift project:

```bash
npx @extragraj/stackshift-skills init
```

This walks you through tier selection, scope, platform, and optional seed strategy in an interactive prompt. Protocol materialization runs automatically by default — no additional flags needed.

#### Common non-interactive variants

```bash
# Defaults: recommended tier, project scope, universal agents platform
npx @extragraj/stackshift-skills init --no-interactive

# Specific tier and platform
npx @extragraj/stackshift-skills init --tier full --platform claude --no-interactive

# With a seeding strategy
npx @extragraj/stackshift-skills init --seed initialvalue-seeding --no-interactive

# Defer all bootstrap steps to the AI agent on first invocation
npx @extragraj/stackshift-skills init --no-materialize --no-interactive
```

#### Available flags

| Flag | Values | Default | Description |
|---|---|---|---|
| `--tier` | `required`, `recommended`, `full` | `recommended` | Protocol tier bundle to install |
| `--scope` | `project`, `global` | `project` | Install location |
| `--platform` | `agents`, `claude`, `copilot`, `gemini`, `cursor` | `agents` | Agentic platform(s) to install to (comma-separated for multiple) |
| `--seed` | seed id or `none` | `none` | Seeding strategy (e.g., `initialvalue-seeding`) |
| `--no-materialize` | *(flag)* | `false` | Skip CLI materialization; defer all steps to the AI agent |
| `--no-interactive` | *(flag)* | `false` | Skip prompts and use flags with defaults |

> **Tier bundles are cumulative.** `recommended` includes all `required` protocols. `full` includes all tiers. Installing multiple tier bundles simultaneously causes conflicts — the CLI prevents this automatically. If you end up in a broken multi-tier state, run `npx @extragraj/stackshift-skills repair`.

---

### Option B — `npx skills add` (Manual)

Use this if you prefer to manage skill packages directly:

```bash
# Universal agents — project scope
npx skills add extragraj/stackshift-workflow-skills -a agents

# Claude Code — project scope
npx skills add extragraj/stackshift-workflow-skills -a claude-code
```

**When using Option B, you must do the following manually:**

1. Always install `stackshift-core` (it is the required base — the workflow, protocols, and references all live there).
2. Install exactly **one** protocol tier bundle (`stackshift-protocols-required`, `stackshift-protocols-recommended`, or `stackshift-protocols-full`).
3. Install at most **one** seeding strategy (optional).
4. If you accidentally install multiple tier bundles or seeds, run `npx @extragraj/stackshift-skills repair`.

Option A handles all of the above automatically. Option B is mainly useful in CI/CD pipelines or environments where the interactive CLI is not practical.

---

### Installation Method Comparison

| Feature | Option A — Interactive CLI | Option B — skills add |
|---|---|---|
| Tier enforcement | Automatic | Manual |
| Core installation | Automatic | Manual |
| Seed activation | Automatic (prompted) | Manual (`npx init` after) |
| Multi-tier prevention | Yes | No (use `repair`) |
| Automation support | Full (`--no-interactive`) | Limited |
| Bootstrap materialization | Default (use `--no-materialize` to defer) | Agent-only |

---

## 2. Pre-Bootstrap & Materialization

After the CLI install completes, a **materialization** step runs automatically (unless you passed `--no-materialize`). This is the pre-bootstrap phase — it does not require the AI agent.

**What the CLI materializes:**

- Copies selected protocol files to `.stackshift/protocols/`
- Creates `.stackshift/protocols/_registry.json` and `_template/`
- Creates `.stackshift/references/README.md`
- Creates `design/standards/stackshift-section-variants.md`
- Writes `.forgeignore` defaults (creates if missing, appends if it already exists — no data loss)
- Writes `.stackshift/installed.json` with `"materializationDone": true`

**Expected terminal output (example):**

```
StackShift Skills Installation

✔ Loading registry...
✔ Installing...

◼ Materialization
  ✅ Materialization complete (CLI phase)
     • Protocols materialized to .stackshift/protocols/
     • Project registries created
     • design/standards/ initialized
     • .forgeignore written

◼ Next steps
  ⏳ Remaining bootstrap steps (require AI agent):
     • UI Forge detection & integration
     • design-arch.json bridging
     • PostToolUse hook installation

  → Run your AI agent to complete bootstrap

Installed 3 skill(s) to .agents/skills/
  ✓ stackshift-core
  ✓ stackshift-protocols-recommended
```

> If you used `--no-materialize`, the marker file will contain `"bootstrapRequired": true` instead of `"materializationDone": true`. The AI agent will detect this on first invocation and run all steps itself, including file materialization.

---

## 3. Installing ui-forge

Once stackshift-workflow-skills is installed, install the **ui-forge** companion skill. The two skills are installed independently — StackShift's bootstrap detects and integrates ui-forge automatically when it is present.

```bash
# Claude Code (project scope)
npx skills add extragraj/ui-forge -a claude-code

# Universal agents (project scope)
npx skills add extragraj/ui-forge -a agents

# Global install (available across all projects)
npx skills add extragraj/ui-forge -y -g -a claude-code
```

**After installing ui-forge, run the setup command once from your project root** to wire slash commands and Bash permissions into your platform directory:

```bash
# Auto-detect platform and install slash commands
for d in .claude .agents .github .cursor .codex .copilot; do
  [ -f "$d/skills/ui-forge/scripts/cli.js" ] && node "$d/skills/ui-forge/scripts/cli.js" install && break
done
```

This writes the `/forge-scan`, `/forge`, `/forge-verify`, and `/forge-export-design` slash commands into your platform directory and adds the required Bash tool permission.

> **Note:** Because this is a StackShift project, ui-forge requires an additional activation step after installation — the Forge Scan described in Section 4. Do not skip it.

---

## 4. Running the Forge Scan

The Forge Scan indexes your project's design tokens, component patterns, and variant structures so ui-forge can generate accurately. For StackShift projects, the scan **must** run with `--theme stackshift`.

### Via an Agentic Model (Slash Command — Recommended)

If your agentic model supports slash commands (Claude Code, Cursor, AntiGravity, etc.), instruct it:

```
/forge-scan --theme stackshift
```

The model executes this, writes `design/design-arch.json`, and registers the scan results with the skill context. This is the preferred invocation path.

---

### For New StackShift Projects

If your project is new and does not yet have an established theme, Tailwind token configuration, or many component files, add the `--theme-override` flag:

```
/forge-scan --theme stackshift --theme-override
```

`--theme-override` surgically replaces three sections in your project files **before** the scan reads them:

- The Google Fonts `@import` in `globals.css` → Inter font
- The `@layer base { }` block in `globals.css` → full HSL CSS variable token set (light + dark)
- The `theme.extend { }` section in `tailwind.config.*` → colors, border radius, spacing, font families, sizes, and weights

This bootstraps a valid StackShift token foundation so the scan produces a meaningful `design-arch.json` even on an empty codebase. `.bak` backup files are created by default. Pass `--no-backup` to skip them. Running the command twice is idempotent.

> **When to use `--theme-override`:** Use it when the project's `globals.css` or `tailwind.config.*` are empty, minimal, or not yet aligned to StackShift conventions. For established projects with existing token configurations, omit it.

---

### If Your Agentic Model Does Not Support Slash Commands

**Option A — Run manually in your terminal:**

```bash
# Resolve the skill root first
for d in .claude .agents .github .cursor .codex .copilot; do
  [ -f "$d/skills/ui-forge/scripts/detect.sh" ] && SKILL_ROOT="$(sh "$d/skills/ui-forge/scripts/detect.sh")" && break
done

# Run the scan
node "$SKILL_ROOT/scripts/scan.js" --theme stackshift

# For new projects
node "$SKILL_ROOT/scripts/scan.js" --theme stackshift --theme-override
```

**Option B — Instruct the model verbally:**

> Please run the ui-forge scan for this StackShift project. Execute `node .agents/skills/ui-forge/scripts/scan.js --theme stackshift` from the project root (adjust the path to `.claude/` if you are using Claude Code) and confirm that `design/design-arch.json` was created before proceeding.

**The key requirement is that the scan runs with `--theme stackshift`.** This forces `isStackShift: true` in `design-arch.json`, ensuring the built-in StackShift UI conventions are always injected at generation time — even on sparse codebases or in `--quick` mode.

---

### Expected Forge Scan Output

```
ui-forge/scan: scanning project...
  applying theme starter: stackshift
  .forgeignore: StackShift exclusions merged (deduplicated).
  48 source files found
  copied built-in standard: stackshift-ui/
  synthesizing patterns...
    using claude CLI for synthesis

Written: design/design-arch.json
Written: design/component-usage.json
```

After this completes, `design/design-arch.json` will contain `"isStackShift": true`, your component library inventory, Tailwind token names, and synthesized pattern data.

---

## 5. Completing the Bootstrap

After the Forge Scan, the remaining bootstrap steps require the AI agent. These steps wire ui-forge's design context into StackShift and set up optional integrations.

### Option A — Explicitly Instruct the Agent

Tell your agentic model:

> Complete the StackShift bootstrap now.

The agent will check `.stackshift/installed.json` for `"materializationDone": true` and run the remaining Phase 2 steps:

- **Step 6a–6b:** Builds the `designStandards` payload and bridges it into `design/design-arch.json`
- **Step 6c:** Detects ui-forge, confirms `design-arch.json` exists (already done)
- **Step 6d:** Records ui-forge integration state in `.stackshift/installed.json`
- **Step 6e:** Version compatibility check between StackShift and ui-forge
- **Step 6h:** Writes the `_paired` mirror block into `design-arch.json`
- **Step 7b:** Installs the PostToolUse auto-verify hook into `.claude/settings.json` *(only if `auto-verify-hook` protocol is installed)*

**Expected bootstrap confirmation:**

```
Bootstrapped StackShift skill (mode: recommended)
  .stackshift/protocols/          ← 11 protocols (4 required, 7 recommended)
  .stackshift/protocols/_registry.json  ← project protocol registry
  .stackshift/references/         ← custom reference lookups (empty)
  design/standards/               ← stackshift-section-variants.md
  .forgeignore                    ← written
  .stackshift/installed.json      ← written

ui-forge integration:             ← detected
  design/design-arch.json:        ← bridged
  _paired mirror block:           ← written
```

---

### Option B — Just Start Your First Task

You can skip the explicit bootstrap instruction and go straight to a workflow task. **StackShift checks bootstrap status at the start of every task.** If `"materializationDone": true` is present without the completed Phase 2 markers, the agent runs the remaining steps automatically before executing the task.

This is the most common real-world path — install everything, scan, then start working.

---

## 6. Use Cases & Examples

The following examples show complete end-to-end workflows demonstrating how both skills operate together.

---

### Use Case 1 — Adding a New Variant to an Existing Section

**Scenario:** The `Features` section exists with `variant_a` through `variant_e`. You want to add `variant_f` — a two-column layout with an icon grid on the right.

**Initial prompt:**

> Add a new `variant_f` to the `Features` section. The variant should be a two-column layout — text content on the left, a 2×3 icon grid on the right. Reference the current `variant_d` for the schema structure and field naming conventions. Use the thumbnail in `images/variant_d.jpg` as a visual reference for spacing patterns.

---

**How the AI processes the protocols (Recommended tier):**

The agent reads `.stackshift/installed.json`, confirms which protocols are active, and enforces them in strict step order.

**Step 1 — Schema Fields**

- `factory-function-pattern` *(required)*: Any new fields must return plain objects with `hidden` + `_hideInVariants`. No `defineField()` wrappers.
- `field-reuse-first` *(recommended)*: Checks local `fields.ts` and the base package for existing factories before creating new ones. Finds `arrayOfImageTitleAndText()` already exists — uses it.
- `hide-if-variant` *(recommended)*: Walks every existing field in `features/schema/index.ts` and adds `"variant_f"` to `hideIfVariantIn([...])` on every field the new variant does not use.
- `preview-conventions` *(recommended)*: Adds `preview` block with `prepare()` to the new icon grid array field.
- `array-layout` *(recommended)*: Sets `options: { layout: "grid" }` on the image array.

**Step 2 — Section Schema**

- `section-directory-layout` *(recommended)*: Creates `initialValue/variant_f.ts` with realistic placeholder copy.
- Appends `{ title: "Variant F", value: "variant_f", description: "Two-column icon grid layout" }` to `variantsList`.

**Step 3 — TypeScript Types**

- Checks `types.ts` for reusable interfaces. Finds `MainImage` and `LabeledRouteWithKey` already defined — uses them.
- Adds new variant fields to the `Variants` interface only if genuinely novel shapes exist.

**Step 4 — Component Variant (ui-forge invocation)**

The agent creates `components/sections/features/variant_f.tsx` as a stub, registers it in `index.tsx`, exports the props interface, then delegates to ui-forge:

```
/forge --task "Generate body for Features_F. Two-column: text left, 2x3 icon grid right. Conform to FeaturesProps from '.'. Do not modify index.tsx." \
       --refs components/sections/features/index.tsx,schemas/custom/.../sections/features/initialValue/variant_f.ts,schemas/custom/.../sections/features/images/variant_d.jpg \
       --output components/sections/features/variant_f.tsx \
       --mode body-only \
       --signal CONVERT_VARIANT
```

**Expected ui-forge stdout (excerpt):**

```
=== UI FORGE ===
SIGNAL: CONVERT_VARIANT
MODE: body-only
PAIRED: stackshift 0.2.5

TASK
Generate body for Features_F...

CONTRACT [components/sections/features/index.tsx]
  interface: FeaturesProps
  version:   1.0.0

DESIGN AUTHORITY: design/design-arch.json
  componentLib: ./components, ./components/ui
  tokens: primary, secondary, background, primary-foreground, ...
  tailwind: tailwind.config.ts
  spacing: Section vertical rhythm: py-20 (hero)...
  usedComponents: Section, Container, Flex, Grid, GridItem, Heading, Text, Button...

DESIGN STANDARDS (load as needed)
// [REF] 02-conditional-link [...]: RULE: Any conditionalLink field must use Button as='link' — READ FULL FILE
// [REF] 04-color-tokens [...]: RULE: Never use raw hex in className — READ FULL FILE
// [REF] 06-spacing [...]: RULE: Section vertical padding always on <Section> — READ FULL FILE
```

**Generated variant file begins with:**

```tsx
// FORGE NOTES
// Signal: CONVERT_VARIANT
// @contract ./components/sections/features/index.tsx
// Anti-slop: verified (no generic gradients, density matches two-column spec)
//
// CONTRACT
//   file: ./components/sections/features/index.tsx
//   interface: FeaturesProps
//   version: 1.0.0
//   props consumed:
//     - title: destructured → rendered as <Heading type="h2">
//     - subtitle: optional → rendered conditionally
//     - items: destructured → mapped to icon grid
//     - primaryButton: optional → rendered as <Button as="link">
//   fallback rule verified: null when !title
//   exports: default + named (PAIRED: stackshift)
```

**Step 5 — GROQ Query**

Adds any non-scalar field projections inside `pages/api/query.ts`. Scalar fields like `title` and `subtitle` are left to the `...` spread. The new `items` array with image sub-fields gets an explicit projection using the existing `mainImageFields` fragment constant.

---

**Validation:**

The agent runs `validate-contract.js` post-generation (or the PostToolUse hook fires if `auto-verify-hook` is installed):

```
UI Forge — Contract Validation
  output:    components/sections/features/variant_f.tsx
  contract:  components/sections/features/index.tsx
  interface: FeaturesProps
  version:   1.0.0
  required:  title, items
  optional:  subtitle, primaryButton
  paired:    stackshift (named export "Features_F" permitted)

CONTRACT CHECK: PASS
```

The workflow checklist then verifies: dynamic import registered in `index.tsx`, no unexpected files written, `index.tsx` bytewise unchanged.

---

### Use Case 2 — Scaffolding a New Section from Scratch

**Scenario:** The project needs a `Testimonials` section with two variants: `variant_a` (3-column grid of cards) and `variant_b` (horizontal scrolling carousel).

**Initial prompt:**

> Scaffold a new `Testimonials` section from scratch. Two variants: `variant_a` (3-column grid) and `variant_b` (horizontal carousel). Reference the `Features` section schema for field naming conventions. Each testimonial card needs: quote (rich text), author name, author role, author avatar, and a star rating (1–5).

---

**Protocol processing sequence:**

**Step 1 — Schema Fields:** `field-reuse-first` checks for existing factories — finds `mainImage()`, `blockContentNormalStyle()`, and `customDropdown()` already exist. Uses them. The star rating becomes a `customDropdown()` with values `["1", "2", "3", "4", "5"]`. `preview-conventions` adds preview blocks with `BsPerson` icon fallback for the testimonial array items.

**Step 2 — Section Schema:** Creates the full directory structure:

```
schemas/custom/.../sections/testimonials/
├── testimonials.ts
├── schema/
│   └── index.ts
├── initialValue/
│   ├── variant_a.ts
│   └── variant_b.ts
└── images/
    └── (thumbnails added manually after components are built)
```

One-time schema setup is already done for this project — skipped.

**Step 3 — Types:** Adds `TestimonialItem` interface to `types.ts`. Reuses `MainImage` for avatar fields.

**Step 4 — ui-forge invoked twice:** Once for `variant_a`, once for `variant_b`. Each invocation uses `--mode body-only` against its respective stub file, with the `initialValue/` directory as a ref for realistic copy.

**Step 5 — GROQ:** The avatar `mainImage` field needs a projection using the existing `mainImageFields` fragment. The `blockContentNormalStyle` quote field is a scalar reference and needs no projection.

---

### Use Case 3 — Tier-Dependent Protocol Behavior

The protocols available at each workflow step depend on what is recorded in `.stackshift/installed.json`. Here is a reference:

| Protocol | Required | Recommended | Full |
|---|---|---|---|
| Factory Function Pattern | ✅ | ✅ | ✅ |
| Sub-Field Visibility | ✅ | ✅ | ✅ |
| Variant Router | ✅ | ✅ | ✅ |
| One-Time Custom Schema Setup | ✅ | ✅ | ✅ |
| Field Reuse First | — | ✅ | ✅ |
| Hide If Variant | — | ✅ | ✅ |
| Preview Conventions | — | ✅ | ✅ |
| Array Layout | — | ✅ | ✅ |
| Section Directory Layout | — | ✅ | ✅ |
| Accessibility (`SIGNAL_A11Y`) | — | ✅ | ✅ |
| Paired-Mode Contract | — | ✅ | ✅ |
| Brand (`SIGNAL_BRAND`) | — | — | ✅ |
| Claude Design Handoff | — | — | ✅ |
| Auto-Verify Hook (PostToolUse) | — | — | ✅ |
| Modal & Sheet | — | — | ✅ |

**Required** protocols block the workflow on violation — they cause build errors, runtime errors, or schema load failures. **Recommended** protocols surface warnings but do not block. **Full-tier optional** protocols activate dedicated systems with their own architecture (modal overlays, Claude Design round-trips, brand signals, auto-verify hooks).

If a task mentions a keyword matching an uninstalled optional protocol, the agent surfaces a notice and applies it contextually:

```
ℹ️ Applying optional protocol "Modal & Sheet" — matched keyword "modal" in your request.
This protocol is not in your installed set. To install it permanently, add it to your
.stackshift/protocols/_registry.json or re-run init with --tier full.
```

---

## 7. Crafting Effective Prompts

Based on real-world usage patterns, the most effective prompts for both skills follow a consistent two-part structure: **context** then **guidelines**.

---

### Recommended Prompt Structure

```
[PART 1 — Context]
Brief description of what needs to be done, where it lives, and what references exist.

[PART 2 — Scheme Guidelines and Design Guidelines (ui-forge)]
Specific instructions about schema conventions, ui-forge flags, design preferences,
or anything the AI might miss due to sparse project context.
```

---

### Part 1 — Context

This is where you answer three questions: **What section or component?** **What are the references?** **What is the general instruction?**

Keep this section focused and specific. The more anchored it is to concrete files and patterns, the less the agent has to infer from sparse codebase context.

**Example:**

> Add a new `variant_c` to the `Stats` section. The variant should be a full-width dark banner with 4 stat items in a horizontal row. Reference `variant_a` for field naming conventions and `images/variant_a.jpg` for the spacing and typography patterns we use on dark backgrounds. General instruction: use the existing `customInteger()` factory for the stat value fields — do not create new factories.

This block tells the agent: which section, which variant key, what layout, which references to load, and one specific field constraint.

---

### Part 2 — Scheme Guidelines and Design Guidelines

This is the most important part for avoiding missed conventions or incorrect assumptions. It splits into two categories:

**Scheme Guidelines** address the schema and data layer — field factories, naming conventions, `hideIfVariantIn` rules, GROQ projection shapes, and anything schema-related the agent might not infer from sparse project files.

**Design Guidelines (ui-forge)** address the component generation layer — explicit ui-forge flags, design token preferences, anti-slop constraints, and visual conventions the agent might miss because the codebase does not yet have enough examples for strong pattern inference.

---

### Full Example Prompt

```
Add a new `variant_d` to the `BlogFeed` section. This variant shows a "featured" post as a
large hero card at the top, with 3 smaller secondary posts in a grid below it. Reference
`variant_b` for the base schema field structure and `variant_b.tsx` for the component layout
patterns. The hero post card should mirror the visual weight of our `Features variant_f`.

---

**Scheme Guidelines**

- The featured post is a `reference` field pointing to the `blog` document type — it is NOT
  a separate array. Use a single reference field pattern, not an array of objects.
- `hideIfVariantIn(["variant_a", "variant_b", "variant_c"])` on all new fields — only
  `variant_d` uses the featured post reference.
- The secondary posts grid reuses the existing `blogPosts` dynamic field — do NOT create a
  new field for this; just expose it for variant_d.
- For the GROQ projection: the featured post reference needs a `->` dereference to pull
  `title`, `slug.current`, `mainImage`, `categories[]->`, and `authors[]-> { name, profileImage }`.
  Use the existing `mainImageFields` fragment for the image projection.

**Design Guidelines (ui-forge)**

- This section lives on a white background page — do NOT infer a dark theme from the hero
  card. The hero card itself has a dark overlay on the image, but the section is `bg-background`.
- The featured post card must use `<Image>` from `@stackshift-ui/image` with the `fill` pattern
  inside a `relative aspect-[16/9]` container — do not use explicit `width` and `height` props.
- For the secondary post cards, use `<Card>` from `@stackshift-ui/card` with its default
  internal padding — do not add extra `p-` utilities on the card element itself.
- The `accessibility` protocol is installed — ensure the section heading has an `id` and
  the `<Section>` has `aria-labelledby` pointing to it.
- Preferred pattern for post "Read more" links: `<Button as="link" link={post.link} variant="link">`.
  Do not use raw `<a>` tags or Next.js `<Link>` directly.
```

---

### Why This Structure Works

**Small, anchored context** (Part 1) prevents the agent from making large structural assumptions about layout or schema shape. Naming the variant key, the references, and the general intent is enough to set scope without over-constraining generation.

**Scheme Guidelines** (Part 2, first half) are most critical when:
- The section being extended has few existing variants, leaving the agent limited schema pattern context to infer from
- You need a specific field type that is easy to get wrong (e.g., a single `reference` vs an array of references)
- You have non-obvious `hideIfVariantIn` requirements that depend on other variants' existence

**Design Guidelines** (Part 2, second half) are most critical when:
- The project is new or sparse, so ui-forge's `design-arch.json` pattern inference may be coarse
- You want to explicitly invoke ui-forge features like `--a11y` or `--handoff` (Claude Design)
- You have a specific token preference that may conflict with what the reference HTML or thumbnail implies (e.g., a white-background section containing a dark card)
- You want to enforce a component usage rule that the agent might miss due to limited variant examples in the codebase

---

### Prompt Patterns for Common Scenarios

**Adding a variant with a known layout reference:**

> Add `variant_e` to `CallToAction`. Use the attached HTML mockup `cta-mockup.html` as the layout reference. Scheme: all new fields use `hideIfVariantIn(["variant_a", "variant_b", "variant_c", "variant_d"])`. Design: the gradient in the mockup maps to `bg-gradient-to-r from-primary to-secondary` — do not use arbitrary hex values.

**Adding a field to an existing variant (no component rebuild):**

> Add a `badge` field to `Features variant_d`. The badge is a short string (e.g. "New", "Popular") displayed above the section heading. Scheme: use `customText("badge", "Badge Label", ...)` factory; add `hideIfVariantIn` for all variants except `variant_d`; only update Steps 1, 2, 3, and 5 — do not touch the component file. Design: no ui-forge invocation needed.

**Rebuilding a variant body without schema changes:**

> Rebuild the component body for `Blog variant_b`. The current implementation uses raw `<a>` tags and `<img>` elements. Fix it to use `<Button as="link">` for all post links and `<Image>` from `@stackshift-ui/image` for thumbnails. Scheme: no schema or GROQ changes. Design: body-only mode against the existing file; all other variants and `index.tsx` must remain untouched.

**New section with Claude Design handoff (Full tier required):**

> Scaffold a new `Showcase` section with a single `variant_a`. The layout comes from the Claude Design handoff at `https://claude.ai/design/h/<id>`. Scheme: the primary data is a curated `projects` array — each item has `title`, `mainImage`, `tag` (string), and `link` (conditionalLink). Design: use `--handoff <url>` for the layout reference; all tokens must map from `design-arch.json` — do not inline any CSS from the handoff.

---

## Appendix — Shared State Files Reference

| File | Owner | Purpose |
|---|---|---|
| `.stackshift/installed.json` | StackShift | Records installed protocols, tier, seed, `a11yRequired`, ui-forge integration state |
| `design/design-arch.json` | ui-forge | Design tokens, component inventory, patterns, `_paired` mirror block |
| `.stackshift/protocols/` | StackShift | Materialized protocol files — project-customizable; take precedence over skill defaults at every lookup |
| `design/standards/stackshift-section-variants.md` | StackShift (CLI) | Variant coding conventions read by ui-forge during generation |
| `design/standards/stackshift-ui/` | ui-forge (`--theme stackshift`) | StackShift UI component conventions (8 focused files) |
| `.claude/settings.json` | Shared | PostToolUse hook entry when `auto-verify-hook` protocol is active |
| `.forgeignore` | Both | Scan exclusions for ui-forge — Sanity Studio, Next.js build output, Claude Design artifacts |
