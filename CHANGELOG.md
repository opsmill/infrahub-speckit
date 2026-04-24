# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.0.0] - 2026-04-24

### Changed

- **BREAKING**: Commands now use the `wrap` composition strategy introduced in spec-kit v0.8.0 instead of `replace`. The core command body (including `before_specify` / `after_plan` / `after_implement` hooks, `SPECIFY_FEATURE_DIRECTORY` resolution, `.specify/feature.json` persistence, marker-based agent context updates, and language-specific `.gitignore` patterns) is inherited automatically from upstream.
- Minimum required `speckit_version` bumped from `>=0.4.1` to `>=0.8.0`.
- `speckit.specify.md`, `speckit.plan.md`, and `speckit.implement.md` now contain only the Infrahub-specific pre-logic (routing, connectivity gating, skill invocation, template selection) followed by the `{CORE_TEMPLATE}` placeholder.

### Removed

- `requires.extensions` entry in `preset.yml` — not part of the official spec-kit preset schema. The `infrahub` extension dependency is now documented in README instead.
- `replaces:` fields on template entries in `preset.yml` — superseded by `strategy: "wrap"`, which identifies the wrapped command via the `name` field.
- Duplicated `before_specify` / `after_specify` / `before_plan` / `after_plan` / `before_implement` / `after_implement` hook-handling blocks — now inherited from core.
- Duplicated Outline, Phases, and Quick Guidelines sections — now inherited from core.

## [1.0.0] - 2026-04-01

### Added

- Initial release of the Infrahub Workflow Routing preset
- `speckit.specify` command override with Infrahub artifact routing
- `speckit.plan` command override with mandatory Infrahub skill invocation
- `speckit.implement` command override with task-level artifact detection
- Multi-artifact dependency chain ordering (Schema first)
- Connectivity gating via `infrahubctl info`
- Artifact type detection for schema, transform, check, generator, and menu
