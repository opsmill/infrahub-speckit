# Infrahub Workflow Routing Extension

Hooks the core `/speckit.specify`, `/speckit.plan`, and `/speckit.implement` commands with Infrahub-aware artifact routing and mandatory skill invocation when `.infrahub.yml` is detected in the repository.

## What it does

The `infrahub-speckit` extension registers three `before_*` hooks against the core speckit skills. When any of those skills is invoked — whether by the slash command, by another skill (e.g. `opsmill-speckit/auto`), or by an autonomous agent — the matching hook fires *before* the skill body runs and:

- detects `.infrahub.yml` (no-op if absent),
- verifies the `infrahub-managing-*` Claude Code skills are installed,
- gates on Infrahub connectivity via `infrahubctl info` (for `before_specify` only),
- classifies the requested artifact type (schema / transform / check / generator / menu),
- invokes the matching `infrahub-managing-*` skill,
- (for `before_specify`) emits a template-override directive so the core specify skill writes from the Infrahub spec template.

The hook returns; the core skill runs.

### Why "extension" not "preset" (v3.0 vs v2.x)

v2.x was a `wrap` preset — it composed at the slash-command layer by substituting `{CORE_TEMPLATE}` at install time. That worked when the user typed `/speckit.specify`, but was bypassed when another skill (such as `opsmill-speckit/auto`'s `prep` step) invoked the `speckit-specify` skill directly. The hook model in v3.0 composes at the skill-runtime layer instead: the core skill's `## Pre-Execution Checks` block reads `.specify/extensions.yml` and fires the registered hook regardless of how the skill was entered.

### `before_specify` hook

1. Checks for `.infrahub.yml` — no-op if absent.
2. Verifies the six `infrahub-managing-*` skills are installed.
3. Gates on Infrahub connectivity via `infrahubctl info`.
4. Classifies the user prompt into artifact types.
5. For multi-artifact prompts, presents the dependency chain and starts with Schema.
6. Invokes the matching `infrahub-managing-*` skill.
7. Emits a template-override directive ("use `.specify/extensions/infrahub/templates/spec-schema-template.md` instead of the core template") that the core specify skill applies when reaching its template-loading step.

### `before_plan` hook

Re-invokes the artifact-matched skill before Phase 0 research begins, then returns.

### `before_implement` hook

Identifies every artifact type touched by `tasks.md`, invokes each matching skill once per session, then returns.

## Dependency chain

When a feature spans multiple artifact types:

```
Schema → [Check / Generator / Transform / Menu]
```

Schema is always first — everything else depends on the data model being loaded.

## Requires

- **`spec-kit >= 0.8.0`** — for the hook execution contract (extensions register `before_*` hooks read from `.specify/extensions.yml`)
- **`opsmill/infrahub` Claude Code skills** (REQUIRED) — provides the `infrahub-managing-schemas`, `infrahub-managing-transforms`, `infrahub-managing-checks`, `infrahub-managing-generators`, `infrahub-managing-menus`, and `infrahub-managing-objects` skills. Each hook command halts with install guidance if the skills are not present.
- **`infrahub` spec-kit extension** (OPTIONAL) — provides the Infrahub-specific spec templates (`spec-schema-template`, `spec-transform-template`, `spec-check-template`, `spec-generator-template`, `spec-menu-template`). If absent, the specify command falls back to the core `spec-template.md` and warns.
- **`infrahubctl` CLI** — for the connectivity check against a running Infrahub instance.

## Installation

Install the spec-kit extension directly from this repository (latest `main`):

```bash
specify extension add infrahub-speckit --from https://github.com/opsmill/infrahub-speckit/archive/refs/heads/main.zip
```

Pin to a released version for stability:

```bash
specify extension add infrahub-speckit --from https://github.com/opsmill/infrahub-speckit/archive/refs/tags/v3.0.0.zip
```

Or, once it's published to the public spec-kit catalog:

```bash
specify extension add infrahub-speckit
```

Install the Infrahub skills (required — these provide the `infrahub-managing-*` skills the hooks invoke):

```bash
npx skills add opsmill/infrahub-skills
```

Or via the Claude Code plugin marketplace:

```
/plugin marketplace add opsmill/claude-marketplace
/plugin install infrahub@opsmill
```

Skills documentation: https://docs.infrahub.app/skills/installation-setup

Optionally install the spec-kit `infrahub` extension once it's in the public catalog — this is the package that ships the Infrahub-specific spec templates (`spec-schema-template`, etc.) referenced in the route-specify hook's template-override directive:

```bash
specify extension add infrahub
```

### Upgrading from v2.x

v2.x installed via `specify preset add infrahub` and overrode `/speckit.specify`, `/speckit.plan`, and `/speckit.implement` via wrap composition. v3.0 installs via `specify extension add infrahub-speckit` and instead registers `before_*` hooks. Upgrade path:

```bash
specify preset remove infrahub
specify extension add infrahub-speckit --from https://github.com/opsmill/infrahub-speckit/archive/refs/tags/v3.0.0.zip
```

After upgrading, your `.specify/extensions.yml` will gain three new `before_*` hook entries under `hooks:`. The slash commands themselves are no longer customized — they run the core skill, which then fires the hook.

### Coexistence with the standalone `infrahub` extension

There is a separate, narrowly-scoped `infrahub` extension (id: `infrahub`) that validates Jira/JPD ticket references on feature branch creation. It also hooks `before_specify`. Both extensions can be installed in the same project and will coexist — `specify extension add` appends hook entries in install order. We recommend installing the JPD validator first (it owns branch creation; no point routing artifacts for a feature branch you can't create), then this extension:

```bash
specify extension add infrahub                                        # JPD/Jira branch validator
specify extension add infrahub-speckit --from <infrahub-speckit URL>  # artifact routing
```

The `before_specify` event will then fire in that order at runtime.

## Usage

### Quick start

From a fresh directory:

```bash
# 1. Initialize a spec-kit project with Claude integration
specify init --here --integration claude

# 2. Install this extension (direct from repo — catalog publishing is pending)
specify extension add infrahub-speckit --from https://github.com/opsmill/infrahub-speckit/archive/refs/heads/main.zip

# 3. Install the required Infrahub skills
npx skills add opsmill/infrahub-skills

# 4. Mark the project as an Infrahub project
cat > .infrahub.yml <<'YAML'
---
schemas: []
jinja2_transforms: []
artifact_definitions: []
queries: []
YAML

# 5. Start your Infrahub instance (so the connectivity gate passes)
invoke start   # or however you start your local Infrahub

# 6. Drive the workflow from Claude Code
```

Then from Claude Code:

```
/speckit.specify Create a schema for Devices, Interfaces, and VLANs with a transform that renders interface config
/speckit.plan
/speckit.tasks
/speckit.implement
```

### What each command does (concretely)

**`/speckit.specify <prompt>`**

The core `speckit-specify` skill fires the `before_specify` hook from this extension before its body runs. The hook:

1. Verifies `.infrahub.yml` exists (else returns; core specify runs unchanged).
2. Verifies the six `infrahub-managing-*` skills are installed (else halts with install guidance).
3. Runs `infrahubctl info` (else halts with "start your Infrahub instance").
4. Matches your prompt against artifact-type keywords — `schema`, `transform`, `check`, `generator`, `menu`.
5. If multiple types match, presents the dependency chain and starts with Schema:
   ```
   Your feature involves multiple Infrahub artifact types:
     1. Schema — Define the data model (must be done first)
     2. Transform — depends on schema

   Starting with: Schema
   After completing this cycle, run /speckit.specify again for the next artifact.
   ```
6. Invokes the matching Infrahub skill (e.g. `infrahub-managing-schemas`) — pulls in curated attribute-kind, relationship-kind, cardinality, and naming-convention reference material that the spec must be consistent with.
7. Selects the Infrahub-specific template (`spec-schema-template`, `spec-transform-template`, etc.) from the `infrahub` spec-kit extension if installed, or falls back to the core `spec-template` with a warning.
8. Emits a template-override directive and returns. The core `speckit-specify` skill body runs next — it picks up the directive and writes `spec.md` from the Infrahub-specific template, then creates the feature directory, produces the quality checklist, and fires `after_specify` hooks.

**`/speckit.plan`**

Re-invokes the artifact-matched skill before Phase 0 research begins, then runs the core planning workflow (produces `plan.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`). The skill's reference material feeds directly into the data model and contract decisions.

**`/speckit.implement`**

Re-invokes the relevant skill before the first task of each artifact type in each phase — `infrahub-managing-schemas` before schema YAML tasks, `infrahub-managing-transforms` before transform tasks, and so on. Each command invocation starts fresh so skills must be re-invoked per command, not just per feature.

### Multi-artifact features: the chain

A single feature request that touches multiple artifact types (like the quick-start example: "schema + transform") is split into separate `/speckit.specify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.implement` cycles, one per artifact, in dependency order.

```
Cycle 1 (Schema):    /speckit.specify Schema for Devices, Interfaces, VLANs
                     /speckit.plan
                     /speckit.tasks
                     /speckit.implement
                     → produces schemas/network.yml

Cycle 2 (Transform): /speckit.specify Render interface config from the schema
                     /speckit.plan
                     /speckit.tasks
                     /speckit.implement
                     → produces templates/interface_config.j2 + queries/interface_config.gql
```

Schema is always cycle 1 because checks, generators, transforms, and menus all depend on the data model being loaded first.

### What ends up in the repository

After a full schema + transform cycle:

```
.infrahub.yml                           # referenced schema + transform + query + artifact_def
schemas/
  network.yml                           # DcimDevice, DcimInterface, IpamVlan
queries/
  interface_config.gql                  # GraphQL query for the transform
templates/
  interface_config.j2                   # Jinja2 rendering the device config
specs/
  001-network-schema/
    spec.md, plan.md, research.md, data-model.md, tasks.md,
    checklists/requirements.md, contracts/
  002-interface-config-transform/
    (same set)
```

## Troubleshooting

**"The infrahub-speckit extension requires the opsmill/infrahub Claude Code skills"** — the preflight check in Step 2 of the hook is telling you the `infrahub-managing-*` skills aren't installed. Run `npx skills add opsmill/infrahub-skills`, restart the session, and retry.

**"Infrahub is not reachable. Please start your Infrahub instance first"** — the `infrahubctl info` connectivity gate in Step 3 of `before_specify` failed. Start your local Infrahub and retry.

**Spec falls back to the core `spec-template.md` and warns** — the separate optional `infrahub` spec-kit extension (the one that ships templates) isn't installed. The fallback produces a valid spec; the only thing you lose is the Infrahub-specific section scaffolding. Install the templates extension if and when available.

**`specify extension list` doesn't show `infrahub-speckit` after install** — confirm the install command succeeded and that `.specify/extensions/infrahub-speckit/extension.yml` exists in the target project. If it does, also confirm `.specify/extensions.yml` has three new entries under `hooks.before_specify`, `hooks.before_plan`, and `hooks.before_implement` referencing this extension. Note: in spec-kit 0.8.x the top-level `installed:` list in `extensions.yml` may stay empty even on a healthy install — the install registry moved to `.specify/extensions/.registry`, which is what `specify extension list` reads. The presence of the `hooks.*` entries is the canonical signal. If those entries are missing, re-run `specify extension add` — install was incomplete.

**Skills exist but the agent didn't invoke them** — the hook command instructs the agent to invoke the skills as a hard requirement, but agents with weak instruction-following may skip. Look for the "anti-rationalization check" in the route-* command files and paste it verbatim to the agent if it tries to proceed without invoking.

**Two `before_specify` hooks fire and only one is wanted** — the standalone `infrahub` (JPD validator) extension and this `infrahub-speckit` extension both hook `before_specify`. This is intentional (see the Coexistence section). If you only want one, remove the other with `specify extension remove <id>`.
