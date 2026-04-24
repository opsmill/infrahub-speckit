# Infrahub Workflow Routing Preset

Wraps the core speckit workflow commands with Infrahub-specific artifact routing and mandatory skill invocation when `.infrahub.yml` is detected in the repository.

## What it does

Uses spec-kit's v0.8.0 `wrap` composition strategy to layer Infrahub-aware behavior *on top of* the core `/speckit.specify`, `/speckit.plan`, and `/speckit.implement` commands. The core command body is inherited automatically (via `{CORE_TEMPLATE}`), so upstream improvements to extension hooks, feature-directory handling, quality checklists, and agent-context updates flow through without manual porting.

### `/speckit.specify`

1. Checks for `.infrahub.yml` in the repo root — if absent, runs the core specify command unchanged.
2. Gates on Infrahub connectivity via `infrahubctl info`.
3. Classifies the user prompt into artifact types (schema, transform, check, generator, menu).
4. For multi-artifact prompts, presents the dependency chain and starts with Schema.
5. Invokes the matching Infrahub skill (`infrahub:schema-creator`, `infrahub:transform-creator`, …) before any spec is written.
6. Selects the Infrahub-specific spec template from the `infrahub` extension and runs the core workflow with that template.

### `/speckit.plan`

Invokes the artifact-matched Infrahub skill before Phase 0 research begins, then runs the core planning workflow.

### `/speckit.implement`

Invokes the artifact-matched Infrahub skill before each implementation phase that touches schema/transform/check/generator/menu/object files, then runs the core implementation workflow.

## Dependency chain

When a feature spans multiple artifact types:

```
Schema → [Check / Generator / Transform / Menu]
```

Schema is always first — everything else depends on the data model being loaded.

## Requires

- **`spec-kit >= 0.8.0`** — for `wrap` composition strategy
- **`opsmill/infrahub` Claude Code skills** (REQUIRED) — provides the `infrahub:schema-creator`, `infrahub:transform-creator`, `infrahub:check-creator`, `infrahub:generator-creator`, `infrahub:menu-creator`, and `infrahub:object-creator` skills. Each wrapped command halts with install guidance if the skills are not present.
- **`infrahub` spec-kit extension** (OPTIONAL) — provides the Infrahub-specific spec templates (`spec-schema-template`, `spec-transform-template`, `spec-check-template`, `spec-generator-template`, `spec-menu-template`). If absent, the specify command falls back to the core `spec-template.md` and warns.
- **`infrahubctl` CLI** — for the connectivity check against a running Infrahub instance.

## Installation

Install the spec-kit preset:

```bash
specify preset add infrahub
```

Install the Infrahub skills (recommended — npx):

```bash
npx skills add opsmill/infrahub-skills
```

Or via the Claude Code plugin marketplace:

```
/plugin marketplace add opsmill/claude-marketplace
/plugin install infrahub@opsmill
```

Skills documentation: https://docs.infrahub.app/skills/installation-setup

Optionally install the spec-kit `infrahub` extension once it's in the public catalog:

```bash
specify extension add infrahub
```
