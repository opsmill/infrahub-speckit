# Hook-Based Extension Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert `infrahub-speckit` from a spec-kit preset (`wrap`-strategy command overrides) into a spec-kit extension that registers `before_specify` / `before_plan` / `before_implement` hooks, so Infrahub artifact routing fires every time the core skills run — regardless of whether the entrypoint is a slash command or another skill (e.g. `opsmill-speckit/auto`) calling into them.

**Architecture:** Replace `preset.yml` with `extension.yml`. Replace the three wrap files (`commands/speckit.{specify,plan,implement}.md`) with three standalone hook command files (`commands/speckit.opsmill.infrahub.route-{specify,plan,implement}.md`). Each hook command runs the existing Infrahub pre-core logic (artifact detection, skill availability check, connectivity gate, skill invocation, template selection) and returns. The template-selection step emits an explicit directive in its output (Workaround H2 from the design discussion) so the core skill picks up the Infrahub template without filesystem mutation. This moves composition from the slash-command layer (which is bypassed by direct skill invocations) to the skill-runtime layer (which honors hooks regardless of caller).

**Tech Stack:** YAML (extension manifest), Markdown (slash command files), spec-kit ≥ v0.8.0 hook contract, `specify extension add` install flow.

---

## Reference Material

**The hook contract (verbatim from any core speckit skill's `## Pre-Execution Checks` block):**

1. Read `.specify/extensions.yml` from project root. Silently skip if missing/unparseable.
2. Look up `hooks.<event>` — list of hook entries (multiple per event, executed in list order).
3. Filter by `enabled` (default true). Skip entries with a non-null `condition`.
4. For each remaining hook, construct the slash command: take the `command` field and replace `.` with `-` (e.g. `speckit.opsmill.infrahub.route-specify` → `/speckit-opsmill-infrahub-route-specify`).
5. If `optional: false` → auto-execute the slash command, wait for result, then proceed to the skill body.
6. If `optional: true` → render an "execute" block; user (or autonomous agent) triggers it.

**Slash-command name → file mapping (dot-to-dash):**

| extension.yml `command:` value | Hook slash command | Source file |
|---|---|---|
| `speckit.opsmill.infrahub.route-specify` | `/speckit-opsmill-infrahub-route-specify` | `commands/speckit.opsmill.infrahub.route-specify.md` |
| `speckit.opsmill.infrahub.route-plan` | `/speckit-opsmill-infrahub-route-plan` | `commands/speckit.opsmill.infrahub.route-plan.md` |
| `speckit.opsmill.infrahub.route-implement` | `/speckit-opsmill-infrahub-route-implement` | `commands/speckit.opsmill.infrahub.route-implement.md` |

**Reference extensions on disk** (read-only; do not modify):
- `/Users/iddo/dev/opsmill/infrahub-sdk-python/.specify/extensions/infrahub/extension.yml` — JPD/Jira branch validator (canonical hook-extension example, `before_specify` hook).
- `/Users/iddo/dev/opsmill/infrahub-sdk-python/.specify/extensions/git/extension.yml` — multi-event hook extension (`before_constitution`, `before_specify`, plus optional `after_*` commit hooks).
- `/Users/iddo/dev/opsmill/infrahub-sdk-python/.specify/extensions/git/commands/speckit.git.feature.md` — example hook command body.

**Template-handoff strategy (Workaround H2 — directive, not file swap):**

The hook can't rewrite the core skill's template reference inline (the hook fires *before* the skill body runs). Instead, the hook command's output must carry an explicit directive that the agent applies when it reaches the template-loading step in the core skill body. The current preset already relies on this pattern (Step 7 of `commands/speckit.specify.md`: "Override for the core workflow below — When the core instructions reference `templates/spec-template.md`, use the Infrahub template selected above instead."). Workaround H2 keeps the same directive, just delivered via hook output rather than via the wrap file's body.

If drift shows up in practice (agent ignores the directive), fall back to H1 (the hook physically copies the Infrahub template over `.specify/templates/spec-template.md` and an `after_specify` hook restores it). H1 is not part of this plan; capture it as a follow-up if needed.

---

## File Structure

**To create:**

- `extension.yml` (repo root) — replaces `preset.yml`. Declares the extension id `opsmill-infrahub`, provides the three route commands, registers the three `before_*` hooks.
- `commands/speckit.opsmill.infrahub.route-specify.md` — hook command, runs Steps 1–8 of the current `commands/speckit.specify.md` (detection, preflight, connectivity gate, classification, skill invocation, template selection, directive emission). Drops the `{CORE_TEMPLATE}` tail.
- `commands/speckit.opsmill.infrahub.route-plan.md` — hook command, runs the "Infrahub Skill Integration (pre-core)" section of the current `commands/speckit.plan.md`. Drops the `{CORE_TEMPLATE}` tail.
- `commands/speckit.opsmill.infrahub.route-implement.md` — hook command, runs the "Infrahub Skill Integration (pre-core)" section of the current `commands/speckit.implement.md`. Drops the `{CORE_TEMPLATE}` tail.
- `docs/superpowers/plans/2026-05-28-hook-extension-migration.md` — this file (already created during planning).

**To delete:**

- `preset.yml`
- `commands/speckit.specify.md`
- `commands/speckit.plan.md`
- `commands/speckit.implement.md`

**To modify:**

- `README.md` — install flow (`specify extension add ...` not `specify preset add ...`), version bump to v3.0.0, hook-model explanation, coexistence note about the standalone `infrahub` (JPD validator) extension, troubleshooting updates.
- `CHANGELOG.md` — add `[3.0.0]` entry documenting the BREAKING migration from preset to extension.

**Out of scope (do NOT touch in this plan):**

- The separate `opsmill/infrahub-skills` skill package (`infrahub-managing-*`) — unchanged.
- The separate `opsmill-speckit` repo (the `auto`/`prep` skill) — unchanged. The whole point of Fix A is that *no change* is needed there.
- Any templates (`spec-schema-template.md` etc.) — they live in the separate optional `infrahub` spec-kit extension, not in this repo. Path references stay the same.

---

## Tasks

### Task 1: Verify pre-flight state

**Goal:** Confirm we're on the right branch with a clean tree and capture a snapshot of the existing wrap-file content for lift-and-shift in later tasks.

**Files:** None modified.

- [ ] **Step 1: Confirm branch and tree state**

Run:
```bash
cd /Users/iddo/dev/opsmill/infrahub-speckit
git status
git branch --show-current
```

Expected:
- Branch: `ic-hook-based-routing`
- Working tree: clean (the plan file under `docs/superpowers/plans/` is acceptable; ignore it).

If branch is not `ic-hook-based-routing`, stop and discuss with the user — the plan was written for this branch.

- [ ] **Step 2: Re-read the three current wrap files to use as source material**

Read all three into context:
```bash
cat commands/speckit.specify.md
cat commands/speckit.plan.md
cat commands/speckit.implement.md
```

These are the source of truth for the new hook command bodies. Every line of pre-core logic (Steps 1–8 of `speckit.specify.md`; the "Infrahub Skill Integration (pre-core)" section of `speckit.plan.md` and `speckit.implement.md`) must be preserved verbatim except for the changes documented in Tasks 3–5. The `{CORE_TEMPLATE}` tail and any text that only makes sense in a wrap context (e.g. "Override for the core workflow below" wording) gets reshaped per the per-task instructions.

- [ ] **Step 3: Confirm reference extensions are reachable**

Run:
```bash
ls /Users/iddo/dev/opsmill/infrahub-sdk-python/.specify/extensions/infrahub/extension.yml
ls /Users/iddo/dev/opsmill/infrahub-sdk-python/.specify/extensions/git/extension.yml
```

Both must exist. They're the on-disk shape that the new `extension.yml` mirrors.

- [ ] **Step 4: No commit yet**

Pre-flight is read-only. Do not commit.

---

### Task 2: Create `extension.yml`

**Goal:** Write the top-level manifest that declares the extension and registers the three `before_*` hooks. This replaces `preset.yml`.

**Files:**
- Create: `extension.yml`

- [ ] **Step 1: Create `extension.yml` with the exact content below**

Write this file verbatim:

```yaml
schema_version: "1.0"

extension:
  id: opsmill-infrahub
  name: "Infrahub Workflow Routing"
  version: "3.0.0"
  description: "Routes speckit specify/plan/implement to Infrahub artifact skills via pre-execution hooks. Hook-based replacement for the v2.x wrap preset."
  author: opsmill
  repository: "https://github.com/opsmill/infrahub-speckit"
  license: Apache-2.0

requires:
  speckit_version: ">=0.8.0"

provides:
  commands:
    - name: speckit.opsmill.infrahub.route-specify
      file: commands/speckit.opsmill.infrahub.route-specify.md
      description: "Detect .infrahub.yml, classify artifact, gate Infrahub skill invocation, select Infrahub spec template."
    - name: speckit.opsmill.infrahub.route-plan
      file: commands/speckit.opsmill.infrahub.route-plan.md
      description: "Gate Infrahub skill invocation before plan."
    - name: speckit.opsmill.infrahub.route-implement
      file: commands/speckit.opsmill.infrahub.route-implement.md
      description: "Gate Infrahub skill invocation before implement."

hooks:
  before_specify:
    command: speckit.opsmill.infrahub.route-specify
    optional: false
    description: "Infrahub artifact routing before specification"
  before_plan:
    command: speckit.opsmill.infrahub.route-plan
    optional: false
    description: "Infrahub artifact routing before planning"
  before_implement:
    command: speckit.opsmill.infrahub.route-implement
    optional: false
    description: "Infrahub artifact routing before implementation"

tags:
  - "infrahub"
  - "infrastructure"
  - "routing"
  - "opsmill"
```

Notes:
- `id: opsmill-infrahub` — chosen to avoid collision with the standalone `infrahub` extension (the JPD/Jira branch validator at `infrahub-sdk-python/.specify/extensions/infrahub/`). Both can be installed in the same project; the extension id only needs to be unique.
- `version: 3.0.0` — the install mechanism changes (`specify preset add` → `specify extension add`), so this is a breaking change for existing consumers.
- `requires.speckit_version: ">=0.8.0"` — same minimum as v2.x. Hooks have existed since earlier versions, but we keep the floor consistent with what consumers already test against.
- `optional: false` on all three — Infrahub routing is mandatory in Infrahub projects (otherwise the wrong template gets used and the wrong skill gets invoked). Not optional.
- No `provides.templates` block — this repo does not ship templates. Templates live in the separate optional `infrahub` spec-kit extension; the route-specify hook references them by path but does not bundle them.

- [ ] **Step 2: Lint the YAML**

Run:
```bash
python -c "import yaml; yaml.safe_load(open('extension.yml'))" && echo "OK"
```

Expected: `OK`. If it errors, the YAML is malformed — fix and re-run.

- [ ] **Step 3: Cross-check against the reference extension shape**

Run:
```bash
diff <(yq 'keys' extension.yml) <(yq 'keys' /Users/iddo/dev/opsmill/infrahub-sdk-python/.specify/extensions/infrahub/extension.yml)
```

(If `yq` is not installed, eyeball the top-level keys instead: both files should have `schema_version`, `extension`, `requires`, `provides`, `hooks`, `tags`.)

Expected: top-level structure matches the reference. Differences in nested values are fine; missing top-level sections are a bug.

- [ ] **Step 4: Commit**

```bash
git add extension.yml
git commit -m "Add extension.yml manifest for hook-based routing

Declares opsmill-infrahub extension v3.0.0, provides three route-*
hook commands, registers before_specify/before_plan/before_implement
hooks. preset.yml and the wrap command files will be removed in a
later commit once the new hook command files are in place."
```

---

### Task 3: Create `commands/speckit.opsmill.infrahub.route-specify.md`

**Goal:** Build the `before_specify` hook command. This is the most substantive of the three — it covers Steps 1–8 of the existing wrap file. The big change vs the wrap version: the hook returns to the calling skill instead of inheriting `{CORE_TEMPLATE}`, so Step 8 ("Proceed to the core workflow") becomes "Emit the routing decision and return", and Step 7's template selection emits a directive in the hook's output rather than substituting inline.

**Files:**
- Create: `commands/speckit.opsmill.infrahub.route-specify.md`
- Reference (do not modify): `commands/speckit.specify.md` (the source for Steps 1–8 content)

- [ ] **Step 1: Write the hook command file**

Write this file verbatim:

```markdown
---
description: "Hook command — runs before /speckit.specify in Infrahub projects. Detects .infrahub.yml, classifies artifact type, gates skill invocation, and emits the template-override directive that the core specify skill consumes."
---

# Infrahub Routing — pre-specify hook

This command is fired as a `before_specify` hook by the `opsmill-infrahub` extension. It runs every time the `speckit-specify` skill is invoked — whether by `/speckit.specify`, by `opsmill-speckit/prep`, or by any other harness. If `.infrahub.yml` is not present in the repository root, this command is a no-op (exit cleanly without emitting any directives).

The core `speckit-specify` skill body runs after this hook returns. Anything you write to the hook's output that the agent treats as part of its context will be applied when the skill reaches the matching step.

## Step 1 — Detect Infrahub project

Check if `.infrahub.yml` exists in the repository root.

- **If it does NOT exist**: Skip the rest of this command. Emit a single line of output:

  ```
  [opsmill-infrahub] No .infrahub.yml detected. No routing applied.
  ```

  Then return. The core specify skill continues unchanged.

- **If it exists**: Continue to Step 2.

## Step 2 — Verify required Infrahub skills are installed

This extension MANDATES invoking the `infrahub-managing-*` skills during specification. Before continuing, confirm these skills appear in your available-skills inventory:

- `infrahub-managing-schemas`
- `infrahub-managing-transforms`
- `infrahub-managing-checks`
- `infrahub-managing-generators`
- `infrahub-managing-menus`
- `infrahub-managing-objects`

**If ANY are missing**, halt and tell the user:

```
The opsmill-infrahub extension requires the opsmill/infrahub Claude Code skills.

Install (recommended):
  npx skills add opsmill/infrahub-skills

Or via the Claude Code plugin marketplace:
  /plugin marketplace add opsmill/claude-marketplace
  /plugin install infrahub@opsmill

Docs: https://docs.infrahub.app/skills/installation-setup

After installing, restart this session and re-run /speckit.specify.
```

Do NOT proceed further until the skills are installed. The "MUST invoke" rules in Step 5 are meaningless without them. Returning from this hook with skills missing means the core specify skill will run without the Infrahub skill context — that's the silent-failure mode v2.0.0 closed and we are not reopening it.

## Step 3 — Verify Infrahub connectivity

Run `infrahubctl info` to verify that the Infrahub instance is reachable.

- **If the command fails or Infrahub is not reachable**, stop and warn the user:
  ```
  Infrahub is not reachable. Please start your Infrahub instance first:
    invoke start
  Then re-run /speckit.specify
  ```
  Do NOT proceed further.
- **If it succeeds**: Continue to Step 4.

## Step 4 — Classify artifact types from the user's input

The user's original input to `/speckit.specify` is available as `$ARGUMENTS` (the same string passed to the core skill). Match it case-insensitively against this table. A prompt can match multiple rows.

| Artifact Type | Keywords / Signals | Extension Template |
|---------------|-------------------|-------------------|
| **Schema**    | schema, data model, nodes, attributes, relationships, hierarchy, generics, namespace, kind, store information, model | `spec-schema-template` |
| **Transform** | transform, render, config, artifact, output, template, jinja, generate config, device config, configuration | `spec-transform-template` |
| **Check**     | check, validate, validation, enforce, rule, constraint, verify | `spec-check-template` |
| **Generator** | generator, auto-create, design-driven, topology, provision, generate objects | `spec-generator-template` |
| **Menu**      | menu, navigation, sidebar, UI, organize | `spec-menu-template` |

If nothing matches, default to `spec-schema-template` (schema-first principle — everything downstream depends on the data model).

## Step 5 — Handle single vs. multiple artifact types

**Single match** — proceed with that template.

**Multiple matches** — present the dependency chain to the user and start with the first:

```markdown
## Infrahub Artifact Routing

Your feature involves multiple Infrahub artifact types:

1. **Schema** — Define the data model (must be done first)
2. **<Second artifact>** — depends on schema

The spec-driven workflow handles one artifact type per cycle:
`/speckit.specify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.implement`

**Starting with: Schema**
After completing this cycle, run `/speckit.specify` again for the next artifact.
```

Dependency order (earlier must complete first):

1. Schema (always first — everything depends on the data model)
2. Check, Generator, Transform, Menu (all depend on schema; independent of each other)

## Step 6 — Invoke the corresponding Infrahub skill

**MANDATORY — do NOT skip, defer, or rationalize around this.** (Availability was already verified in Step 2.)

Invoke the skill using the Skill tool BEFORE returning. The skill provides curated reference material (schema properties, validation rules, naming conventions, API patterns) that the spec MUST be consistent with. Skill content is NOT persisted across commands, so it MUST be re-loaded here even if the same skill was loaded earlier in the session.

| Artifact Type | Skill to Invoke | What It Provides |
|---------------|-----------------|------------------|
| **Schema**    | `infrahub-managing-schemas`    | Schema property reference, node/generic definitions, attribute kinds, relationship types, CoreFileObject, naming conventions, validation rules |
| **Transform** | `infrahub-managing-transforms` | Transform types (Python/Jinja2), query patterns, artifact definitions, content types |
| **Check**     | `infrahub-managing-checks`     | Check definition structure, validation logic patterns, proposed change pipeline |
| **Generator** | `infrahub-managing-generators` | Generator class patterns, target groups, query parameters, idempotent creation |
| **Menu**      | `infrahub-managing-menus`      | Menu structure, node organization, sidebar customization |

For multi-artifact prompts, invoke the skill for the FIRST artifact type in the dependency chain (the one being specified in this cycle).

**Anti-rationalization check** — if any of these thoughts appear, invoke the skill anyway:
- "I already know Infrahub schemas" — the skill has validated reference material that's not in training data
- "I fetched the docs via WebFetch" — web docs can be incomplete or outdated
- "This is a simple change" — simple changes still need correct attribute kinds, cardinalities, and naming
- "I'll invoke it later during planning" — the spec fixes entities and requirements NOW

## Step 7 — Resolve the template path

Resolve the template path for the selected artifact type:

1. Prefer `.specify/extensions/infrahub/templates/<template-name>.md` (e.g. `spec-schema-template.md`). Note: these templates are shipped by the separate optional `infrahub` spec-kit extension (not by `opsmill-infrahub`).
2. If that file does not exist, fall back to `.specify/templates/spec-template.md`. Emit a warning that the Infrahub extension templates are missing.

## Step 8 — Emit the routing directive and return

Emit the following directive verbatim as the final output of this hook. The core `speckit-specify` skill reads the hook output as part of its context before entering its Outline phase, and the agent must honor this directive when reaching the template-loading step in the skill body:

```
[opsmill-infrahub routing — applies to this /speckit.specify run only]

Artifact type: <SELECTED-ARTIFACT-TYPE>
Skill loaded: <SKILL-NAME>
Template override: When the core specify skill references `.specify/templates/spec-template.md` or "spec-template", use the template at <RESOLVED-TEMPLATE-PATH> instead. All other behavior of the core skill (feature directory, branch hook ordering, quality checklist, after_specify hooks) is unchanged.
```

Substitute the bracketed placeholders with the actual values from Steps 4–7. Then return — the core `speckit-specify` skill body runs next.

If multiple artifact types were detected in Step 5, also remind the user at the end of the spec cycle to re-run `/speckit.specify` for the next artifact in the chain. (This reminder lives in the hook output; the core skill will surface it when reporting completion.)
```

- [ ] **Step 2: Confirm content was lifted faithfully from the source**

Run:
```bash
# Verify the new file exists and is non-trivial in size (should be > 6KB)
wc -c commands/speckit.opsmill.infrahub.route-specify.md
```

Expected: byte count similar to or slightly less than `commands/speckit.specify.md` (which is ~7KB). Significantly smaller means content was dropped; significantly larger means content was added beyond what the wrap file had.

- [ ] **Step 3: Spot-check the diff vs. the original wrap file**

Compare the new hook command against the source wrap file:
```bash
diff -u commands/speckit.specify.md commands/speckit.opsmill.infrahub.route-specify.md | head -80
```

Expected differences:
1. Frontmatter `description:` is rewritten to describe a hook command, not a wrapped slash command. `handoffs:` block is dropped (handoffs are part of the wrapped command's flow, not the hook's).
2. New top section ("Infrahub Routing — pre-specify hook") replaces "Infrahub Routing (pre-core)".
3. Step 7 wording changes: removes "Override for the core workflow below" inline-substitution wording; instead just resolves the template path.
4. Step 8 is rewritten: from "Proceed to the core workflow" + `{CORE_TEMPLATE}` to "Emit the routing directive and return" with the directive block.
5. The `{CORE_TEMPLATE}` placeholder at the end of the file is removed.

Steps 1–6 should be near-identical (typo-fix-level differences only). If you see semantic drift in Steps 1–6, the lift went wrong — re-do it.

- [ ] **Step 4: Commit**

```bash
git add commands/speckit.opsmill.infrahub.route-specify.md
git commit -m "Add before_specify hook command (route-specify)

Lifts Steps 1-8 of the existing wrap file into a standalone hook
command. Step 7 resolves the template path; Step 8 emits the
template-override directive instead of inheriting {CORE_TEMPLATE}.
Drops handoffs frontmatter (irrelevant for a hook command)."
```

---

### Task 4: Create `commands/speckit.opsmill.infrahub.route-plan.md`

**Goal:** Build the `before_plan` hook command. Much smaller than route-specify — it lifts the "Infrahub Skill Integration (pre-core)" section of the current `commands/speckit.plan.md` (which is itself a slimmed-down version of the specify pre-core: prereqs → detection → invocation, no connectivity gate, no template selection).

**Files:**
- Create: `commands/speckit.opsmill.infrahub.route-plan.md`
- Reference: `commands/speckit.plan.md`

- [ ] **Step 1: Write the hook command file**

Write this file verbatim:

```markdown
---
description: "Hook command — runs before /speckit.plan in Infrahub projects. Invokes the matching infrahub-managing-* skill so Phase 0 research and data-model design are grounded in authoritative Infrahub reference material."
---

# Infrahub Routing — pre-plan hook

This command is fired as a `before_plan` hook by the `opsmill-infrahub` extension. It runs every time the `speckit-plan` skill is invoked. If `.infrahub.yml` is not present, this command is a no-op.

**MANDATORY — do NOT skip, defer, or rationalize around this.**

Before the core `speckit-plan` skill generates any design artifacts (`research.md`, `data-model.md`, `contracts/`), the matching Infrahub skill must be loaded. The skill loads authoritative reference material that design decisions MUST be consistent with. Skill content is NOT persisted across commands; each command starts fresh, so the skill MUST be re-loaded here even if it was loaded during `/speckit.specify`.

## Step 1 — Detect Infrahub project

If the repository has no `.infrahub.yml`, emit:

```
[opsmill-infrahub] No .infrahub.yml detected. No routing applied.
```

Then return. The core plan skill runs unchanged.

## Step 2 — Verify required Infrahub skills are installed

Confirm these skills appear in your available-skills inventory:

- `infrahub-managing-schemas`
- `infrahub-managing-transforms`
- `infrahub-managing-checks`
- `infrahub-managing-generators`
- `infrahub-managing-menus`
- `infrahub-managing-objects`

**If ANY are missing**, halt and tell the user:

```
The opsmill-infrahub extension requires the opsmill/infrahub Claude Code skills.

Install (recommended):
  npx skills add opsmill/infrahub-skills

Or via the Claude Code plugin marketplace:
  /plugin marketplace add opsmill/claude-marketplace
  /plugin install infrahub@opsmill

Docs: https://docs.infrahub.app/skills/installation-setup

After installing, restart this session and re-run /speckit.plan.
```

Do NOT proceed further until the skills are installed.

## Step 3 — Classify the artifact type from `spec.md`

1. Read `spec.md` from the current feature directory (resolve via `.specify/feature.json` or by inspecting the current working branch, however the core skill resolves it).
2. Classify the Infrahub artifact type from the spec content:

   | Artifact Type | Signals in Spec | Skill to Invoke |
   |---------------|-----------------|-----------------|
   | **Schema**    | schema, data model, nodes, attributes, relationships, generics, namespace, kind, CoreFileObject, inherit_from | `infrahub-managing-schemas` |
   | **Transform** | transform, render, config, artifact, jinja, device config | `infrahub-managing-transforms` |
   | **Check**     | check, validate, validation, enforce, rule, constraint | `infrahub-managing-checks` |
   | **Generator** | generator, auto-create, design-driven, topology, provision | `infrahub-managing-generators` |
   | **Menu**      | menu, navigation, sidebar, UI | `infrahub-managing-menus` |

3. If the spec involves multiple artifact types, invoke the skill for the PRIMARY type (the one being designed in this cycle).

## Step 4 — Invoke the matched skill

Invoke the matched skill using the Skill tool BEFORE returning. Use the skill's reference material when:

- Writing `data-model.md` (entity definitions, attribute kinds, relationship types)
- Making research decisions (SDK patterns, API conventions, naming rules)
- Validating the design against Infrahub capabilities

**Anti-rationalization check** — if you think "I already researched this via WebFetch" or "I know Infrahub well enough" or "the spec already has everything I need", invoke the skill anyway. The skill has curated reference material that general knowledge and web fetches do not provide.

## Step 5 — Emit the routing record and return

Emit a single status line:

```
[opsmill-infrahub routing — applies to this /speckit.plan run only]

Skill loaded: <SKILL-NAME>
```

Then return. The core `speckit-plan` skill runs next.
```

- [ ] **Step 2: Sanity-check the file size**

```bash
wc -c commands/speckit.opsmill.infrahub.route-plan.md
```

Expected: ~3KB (similar to or slightly larger than `commands/speckit.plan.md` which is ~3.3KB; we added explicit steps but dropped `{CORE_TEMPLATE}`).

- [ ] **Step 3: Commit**

```bash
git add commands/speckit.opsmill.infrahub.route-plan.md
git commit -m "Add before_plan hook command (route-plan)

Lifts the Infrahub Skill Integration (pre-core) section from the
existing plan wrap file into a standalone hook command. Steps are
restructured for sequential execution: detect, preflight, classify,
invoke, emit. Drops {CORE_TEMPLATE} tail."
```

---

### Task 5: Create `commands/speckit.opsmill.infrahub.route-implement.md`

**Goal:** Build the `before_implement` hook command. Lifts the "Infrahub Skill Integration (pre-core)" section of the current `commands/speckit.implement.md`, including the per-artifact-type "invoke once per artifact type per session" rule and the timing guidance.

**Files:**
- Create: `commands/speckit.opsmill.infrahub.route-implement.md`
- Reference: `commands/speckit.implement.md`

- [ ] **Step 1: Write the hook command file**

Write this file verbatim:

```markdown
---
description: "Hook command — runs before /speckit.implement in Infrahub projects. Invokes the matching infrahub-managing-* skill for each artifact type that tasks.md touches, so implementation code follows authoritative Infrahub patterns and conventions."
---

# Infrahub Routing — pre-implement hook

This command is fired as a `before_implement` hook by the `opsmill-infrahub` extension. It runs every time the `speckit-implement` skill is invoked. If `.infrahub.yml` is not present, this command is a no-op.

**MANDATORY — do NOT skip, defer, or rationalize around this.**

Before any task that touches Infrahub artifacts is implemented, the corresponding Infrahub skill must be loaded. The skill loads authoritative reference material, validation rules, and implementation patterns that the code MUST follow.

## Step 1 — Detect Infrahub project

If the repository has no `.infrahub.yml`, emit:

```
[opsmill-infrahub] No .infrahub.yml detected. No routing applied.
```

Then return. The core implement skill runs unchanged.

## Step 2 — Verify required Infrahub skills are installed

Confirm these skills appear in your available-skills inventory:

- `infrahub-managing-schemas`
- `infrahub-managing-transforms`
- `infrahub-managing-checks`
- `infrahub-managing-generators`
- `infrahub-managing-menus`
- `infrahub-managing-objects`

**If ANY are missing**, halt and tell the user:

```
The opsmill-infrahub extension requires the opsmill/infrahub Claude Code skills.

Install (recommended):
  npx skills add opsmill/infrahub-skills

Or via the Claude Code plugin marketplace:
  /plugin marketplace add opsmill/claude-marketplace
  /plugin install infrahub@opsmill

Docs: https://docs.infrahub.app/skills/installation-setup

After installing, restart this session and re-run /speckit.implement.
```

Do NOT proceed further until the skills are installed.

## Step 3 — Identify artifact types touched by `tasks.md`

1. Read `tasks.md` and `plan.md` from the current feature directory.
2. Identify every artifact type any task in any phase touches:

   | Task Involves | Skill to Invoke | When to Invoke |
   |---------------|-----------------|----------------|
   | Schema YAML files (`schemas/`)          | `infrahub-managing-schemas`    | Before the first schema task |
   | Python transforms (`transforms/`)       | `infrahub-managing-transforms` | Before writing transform code |
   | Python checks                           | `infrahub-managing-checks`     | Before writing check code |
   | Generator Python files (`generators/`)  | `infrahub-managing-generators` | Before writing generator code |
   | Menu YAML files (`menus/`)              | `infrahub-managing-menus`      | Before writing menu definitions |
   | Object data files (`objects/`)          | `infrahub-managing-objects`    | Before writing seed data |

3. Build the list of skills to load — one per artifact type touched, deduplicated.

## Step 4 — Invoke each matched skill

Invoke each skill on the deduplicated list using the Skill tool, BEFORE returning. Each skill loads once per implementation session — no need to re-invoke per task within the same session.

**Anti-rationalization check** — if you think "I already loaded the skill during /speckit.specify" or "I already loaded it during /speckit.plan" or "the plan has all the details I need", invoke the skill anyway. Skill content is NOT persisted across commands; each command session starts fresh.

## Step 5 — Emit the routing record and return

Emit a single status line listing the skills loaded:

```
[opsmill-infrahub routing — applies to this /speckit.implement run only]

Skills loaded: <comma-separated list>
```

Then return. The core `speckit-implement` skill runs next.
```

- [ ] **Step 2: Sanity-check file size**

```bash
wc -c commands/speckit.opsmill.infrahub.route-implement.md
```

Expected: ~3.5KB (similar to `commands/speckit.implement.md` which is ~3.3KB).

- [ ] **Step 3: Commit**

```bash
git add commands/speckit.opsmill.infrahub.route-implement.md
git commit -m "Add before_implement hook command (route-implement)

Lifts the Infrahub Skill Integration (pre-core) section from the
existing implement wrap file into a standalone hook command.
Preserves the per-artifact-type once-per-session invocation rule."
```

---

### Task 6: Delete the old preset files

**Goal:** Remove the v2.x preset artifacts now that the v3.0 hook extension is in place.

**Files:**
- Delete: `preset.yml`
- Delete: `commands/speckit.specify.md`
- Delete: `commands/speckit.plan.md`
- Delete: `commands/speckit.implement.md`

- [ ] **Step 1: Confirm the new files are in place before deleting old ones**

```bash
ls extension.yml commands/speckit.opsmill.infrahub.route-specify.md commands/speckit.opsmill.infrahub.route-plan.md commands/speckit.opsmill.infrahub.route-implement.md
```

Expected: all four files exist. If any are missing, do not delete the old files — go back and finish Tasks 2–5.

- [ ] **Step 2: Delete the old files**

```bash
git rm preset.yml
git rm commands/speckit.specify.md
git rm commands/speckit.plan.md
git rm commands/speckit.implement.md
```

Verify the working tree:
```bash
ls commands/
```

Expected: only the three new `speckit.opsmill.infrahub.route-*.md` files. No `speckit.specify.md`, `speckit.plan.md`, `speckit.implement.md`.

- [ ] **Step 3: Confirm the repo layout matches the target shape**

```bash
find . -maxdepth 3 -type f \( -name "*.yml" -o -name "*.md" \) -not -path "./.git/*" -not -path "./docs/*" | sort
```

Expected output (order may vary):
```
./CHANGELOG.md
./LICENSE
./README.md
./commands/speckit.opsmill.infrahub.route-implement.md
./commands/speckit.opsmill.infrahub.route-plan.md
./commands/speckit.opsmill.infrahub.route-specify.md
./extension.yml
```

- [ ] **Step 4: Commit**

```bash
git commit -m "Remove preset.yml and v2.x wrap command files

The hook-based extension (extension.yml + route-* commands) is now
in place. The preset mechanism composed at the slash-command layer,
which is bypassed when other skills invoke speckit-specify/plan/
implement directly. The hook mechanism composes at the skill-runtime
layer and fires regardless of caller."
```

---

### Task 7: Update `README.md`

**Goal:** Rewrite README to reflect the hook-extension model: new install command, new file layout, new explanation of where composition happens, coexistence note about the standalone `infrahub` (JPD validator) extension, and an upgrade note for v2.x users.

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace the title and "What it does" sections**

Replace the current top section (lines 1–24, ending just before "## Dependency chain"):

```markdown
# Infrahub Workflow Routing Extension

Hooks the core `/speckit.specify`, `/speckit.plan`, and `/speckit.implement` commands with Infrahub-aware artifact routing and mandatory skill invocation when `.infrahub.yml` is detected in the repository.

## What it does

The `opsmill-infrahub` extension registers three `before_*` hooks against the core speckit skills. When any of those skills is invoked — whether by the slash command, by another skill (e.g. `opsmill-speckit/auto`), or by an autonomous agent — the matching hook fires *before* the skill body runs and:

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
```

- [ ] **Step 2: Update the Installation section**

Find the `## Installation` section and replace its content with:

```markdown
## Installation

Install the spec-kit extension directly from this repository (latest `main`):

```bash
specify extension add --from https://github.com/opsmill/infrahub-speckit/archive/refs/heads/main.zip
```

Pin to a released version for stability:

```bash
specify extension add --from https://github.com/opsmill/infrahub-speckit/archive/refs/tags/v3.0.0.zip
```

Or, once it's published to the public spec-kit catalog:

```bash
specify extension add opsmill-infrahub
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

v2.x installed via `specify preset add infrahub` and overrode `/speckit.specify`, `/speckit.plan`, and `/speckit.implement` via wrap composition. v3.0 installs via `specify extension add opsmill-infrahub` and instead registers `before_*` hooks. Upgrade path:

```bash
specify preset remove infrahub
specify extension add --from https://github.com/opsmill/infrahub-speckit/archive/refs/tags/v3.0.0.zip
```

After upgrading, your `.specify/extensions.yml` will gain three new `before_*` hook entries under `hooks:`. The slash commands themselves are no longer customized — they run the core skill, which then fires the hook.

### Coexistence with the standalone `infrahub` extension

There is a separate, narrowly-scoped `infrahub` extension (id: `infrahub`) that validates Jira/JPD ticket references on feature branch creation. It also hooks `before_specify`. Both extensions can be installed in the same project and will coexist — `specify extension add` appends hook entries in install order. We recommend installing the JPD validator first (it owns branch creation; no point routing artifacts for a feature branch you can't create), then this extension:

```bash
specify extension add infrahub                          # JPD/Jira branch validator
specify extension add --from <opsmill-infrahub URL>     # artifact routing
```

The `before_specify` event will then fire in that order at runtime.
```

- [ ] **Step 3: Update the Usage and What-each-command-does sections**

Find the section starting with `### What each command does (concretely)` and update the `**/speckit.specify <prompt>**` subsection's first line and last line.

The bullet list's content is correct — the hook does the same work. Only update the framing. Change:

```markdown
**`/speckit.specify <prompt>`**

1. Verifies `.infrahub.yml` exists (else runs core specify unchanged).
```

to:

```markdown
**`/speckit.specify <prompt>`**

The core `speckit-specify` skill fires the `before_specify` hook from this extension before its body runs. The hook:

1. Verifies `.infrahub.yml` exists (else returns; core specify runs unchanged).
```

And change the trailing bullet 8 from:

```markdown
8. Runs the core specify workflow with the selected template — writes `spec.md`, creates the feature directory, produces the quality checklist, fires `after_specify` hooks.
```

to:

```markdown
8. Emits a template-override directive and returns. The core `speckit-specify` skill body runs next — it picks up the directive and writes `spec.md` from the Infrahub-specific template, then creates the feature directory, produces the quality checklist, and fires `after_specify` hooks.
```

- [ ] **Step 4: Update the Troubleshooting section**

Replace the `**Spec falls back to the core spec-template.md and warns**` and `**specify preset resolve speckit.specify says "not found"**` and `**Skills exist but my agent didn't invoke them**` entries with hook-flavored equivalents:

```markdown
**"The opsmill-infrahub extension requires the opsmill/infrahub Claude Code skills"** — the preflight check in Step 2 of the hook is telling you the `infrahub-managing-*` skills aren't installed. Run `npx skills add opsmill/infrahub-skills`, restart the session, and retry.

**"Infrahub is not reachable. Please start your Infrahub instance first"** — the `infrahubctl info` connectivity gate in Step 3 of `before_specify` failed. Start your local Infrahub and retry.

**Spec falls back to the core `spec-template.md` and warns** — the separate optional `infrahub` spec-kit extension (the one that ships templates) isn't installed. The fallback produces a valid spec; the only thing you lose is the Infrahub-specific section scaffolding. Install the templates extension if and when available.

**`specify extension list` doesn't show `opsmill-infrahub` after install** — confirm the install command succeeded and that `.specify/extensions/opsmill-infrahub/extension.yml` exists in the target project. If it does, also confirm `.specify/extensions.yml` has three new entries under `hooks.before_specify`, `hooks.before_plan`, and `hooks.before_implement` referencing this extension. If those entries are missing, re-run `specify extension add` — install was incomplete.

**Skills exist but the agent didn't invoke them** — the hook command instructs the agent to invoke the skills as a hard requirement, but agents with weak instruction-following may skip. Look for the "anti-rationalization check" in the route-* command files and paste it verbatim to the agent if it tries to proceed without invoking.

**Two `before_specify` hooks fire and only one is wanted** — the standalone `infrahub` (JPD validator) extension and this `opsmill-infrahub` extension both hook `before_specify`. This is intentional (see the Coexistence section). If you only want one, remove the other with `specify extension remove <id>`.
```

- [ ] **Step 5: Read the final README to confirm coherence**

```bash
wc -l README.md
```

Expected: roughly the same length as before (some sections shrank, some grew). Read the file end-to-end and confirm there are no references to "preset", "wrap composition", or "`{CORE_TEMPLATE}`" left behind. Search:

```bash
grep -n -i 'preset\|{CORE_TEMPLATE}\|wrap composition' README.md
```

Expected: only matches in the "Upgrading from v2.x" section (intentional — that section explains the migration). Any other matches are stale text that needs updating.

- [ ] **Step 6: Commit**

```bash
git add README.md
git commit -m "Rewrite README for hook-extension model

Documents the v3.0 hook-based architecture, the install flow via
specify extension add, the upgrade path from the v2.x preset, and
coexistence with the standalone infrahub (JPD validator) extension.
Updates troubleshooting for the new failure modes."
```

---

### Task 8: Update `CHANGELOG.md`

**Goal:** Document the v3.0.0 release with a BREAKING marker and an upgrade pointer.

**Files:**
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Insert the v3.0.0 entry at the top**

Open `CHANGELOG.md` and insert the following block immediately after the `# Changelog` / format-notes header, BEFORE the existing `## [2.0.0]` entry:

```markdown
## [3.0.0] - 2026-05-28

### Changed

- **BREAKING**: This package is now a spec-kit **extension** (`extension.yml`), not a preset (`preset.yml`). Composition has moved from the slash-command layer (preset/wrap) to the skill-runtime layer (hooks). Install with `specify extension add` instead of `specify preset add`. The extension id is `opsmill-infrahub` (renamed from `infrahub` to avoid collision with the standalone `infrahub` JPD/Jira branch validator extension).
- **BREAKING**: The three customized commands (`speckit.specify`, `speckit.plan`, `speckit.implement`) are no longer wrapped. The Infrahub routing logic now lives in three new hook commands (`speckit.opsmill.infrahub.route-specify`, `route-plan`, `route-implement`) registered against `before_specify`, `before_plan`, and `before_implement` respectively. From a user's perspective, `/speckit.specify` behaves the same — the hook fires before the core skill body runs. From an integrator's perspective, the routing now fires for *any* invocation of the underlying skills, not just the slash command — so callers like `opsmill-speckit/auto` are now covered.

### Why this matters

The v2.x preset wrapped slash command files. When `opsmill-speckit/auto` (or any other skill or agent) invoked the `speckit-specify` skill directly without going through `/speckit.specify`, the wrap was bypassed and Infrahub routing did not run. The v3.0 hook fires inside the core skill's pre-execution checks, regardless of caller.

### Migration

```bash
specify preset remove infrahub
specify extension add --from https://github.com/opsmill/infrahub-speckit/archive/refs/tags/v3.0.0.zip
```

After upgrade, `.specify/extensions.yml` will have three new entries under `hooks.before_specify`, `hooks.before_plan`, and `hooks.before_implement`. No other consumer-side changes required.

### Removed

- `preset.yml` (replaced by `extension.yml`).
- `commands/speckit.specify.md`, `commands/speckit.plan.md`, `commands/speckit.implement.md` (replaced by the three `route-*` hook commands).

```

The existing `## [2.0.0]` and `## [1.0.0]` entries stay in place after the new entry.

- [ ] **Step 2: Verify the CHANGELOG parses as valid markdown**

```bash
head -50 CHANGELOG.md
```

Expected: the new 3.0.0 entry appears first, followed by 2.0.0, followed by 1.0.0. No malformed headers, no broken code blocks.

- [ ] **Step 3: Commit**

```bash
git add CHANGELOG.md
git commit -m "Document v3.0.0 release in CHANGELOG

Adds the BREAKING migration from preset to extension and the
upgrade command. Explains the integration-level motivation:
fixes the gap where opsmill-speckit/auto bypassed the v2.x wrap."
```

---

### Task 9: Smoke test — install in a sandbox project and verify the hook fires

**Goal:** End-to-end validation that the extension installs cleanly, registers its hooks in the target project's `.specify/extensions.yml`, and that the slash command resolver picks up the new `speckit-opsmill-infrahub-route-*` commands.

This is the actual proof Fix A works. Without it, we're shipping based on theory.

**Files:** None (sandbox lives outside the repo).

- [ ] **Step 1: Create a sandbox project with `.infrahub.yml`**

```bash
TMPDIR_SANDBOX=$(mktemp -d /tmp/opsmill-infrahub-sandbox.XXXX)
cd "$TMPDIR_SANDBOX"
specify init --here --integration claude
cat > .infrahub.yml <<'YAML'
---
schemas: []
jinja2_transforms: []
artifact_definitions: []
queries: []
YAML
echo "Sandbox at: $TMPDIR_SANDBOX"
```

Expected: `.specify/` directory created, `.infrahub.yml` written. `specify` CLI must be available in PATH.

- [ ] **Step 2: Install the extension from the local repo**

```bash
specify extension add --from /Users/iddo/dev/opsmill/infrahub-speckit
```

If `specify` requires a zip rather than a directory, build one first:

```bash
cd /Users/iddo/dev/opsmill/infrahub-speckit
zip -r /tmp/opsmill-infrahub.zip . -x '.git/*' 'docs/*'
cd "$TMPDIR_SANDBOX"
specify extension add --from /tmp/opsmill-infrahub.zip
```

Expected: install succeeds without error. If it fails with a schema error, the `extension.yml` is malformed — go back to Task 2.

- [ ] **Step 3: Verify the extension files landed**

```bash
ls -la .specify/extensions/opsmill-infrahub/
ls .specify/extensions/opsmill-infrahub/commands/
```

Expected:
- `.specify/extensions/opsmill-infrahub/extension.yml` exists.
- `.specify/extensions/opsmill-infrahub/commands/` contains all three `speckit.opsmill.infrahub.route-*.md` files.

If any file is missing, install was incomplete — investigate before proceeding.

- [ ] **Step 4: Verify hooks were registered in `.specify/extensions.yml`**

```bash
cat .specify/extensions.yml
```

Expected: the file contains an `installed:` entry for `opsmill-infrahub` and three new entries under `hooks.before_specify`, `hooks.before_plan`, and `hooks.before_implement` referencing the `opsmill-infrahub` extension with `optional: false`.

Example (your output will differ in exact formatting):

```yaml
installed:
- id: opsmill-infrahub
  path: .specify/extensions/opsmill-infrahub
hooks:
  before_specify:
  - extension: opsmill-infrahub
    command: speckit.opsmill.infrahub.route-specify
    enabled: true
    optional: false
    ...
```

If the hook entries are not present, the install logic didn't read the `hooks:` block in `extension.yml` — go back to Task 2 and re-inspect the manifest.

- [ ] **Step 5: Verify the slash command resolver picks up the new commands**

In a fresh Claude Code session opened in the sandbox project, run:
```
/help
```
or whatever the equivalent slash-command discovery mechanism is. Confirm `/speckit-opsmill-infrahub-route-specify` (and the `-plan` / `-implement` variants) appear in the available command list.

If they do not appear, the issue is in how `provides.commands` is published to the resolver — check spec-kit's extension-loading code path. This is a known risk flagged in the design discussion ("Hook command discovery") and is the most likely failure mode for the smoke test.

- [ ] **Step 6: End-to-end test — invoke `/speckit.specify` and confirm the hook fires**

In the sandbox Claude Code session, type:
```
/speckit.specify Create a schema for Devices, Interfaces, and VLANs
```

Expected behavior:
1. The core `speckit-specify` skill enters its pre-execution checks phase.
2. It reads `.specify/extensions.yml`, finds the `before_specify` hook, and auto-executes `/speckit-opsmill-infrahub-route-specify` (since `optional: false`).
3. The hook command runs through Steps 1–8: detects `.infrahub.yml`, verifies skills, gates on connectivity, classifies the prompt as `schema`, invokes `infrahub-managing-schemas`, resolves the template path, emits the directive.
4. The core specify skill body runs, picks up the directive, and writes `spec.md` from the schema template (or warns and falls back if the template isn't found — expected in this sandbox since the optional `infrahub` templates extension isn't installed).

The proof points to record:
- Did the hook fire? (Look for the `[opsmill-infrahub routing — applies to this /speckit.specify run only]` directive in the agent output.)
- Was `infrahub-managing-schemas` actually invoked? (Look for the Skill tool call in the transcript.)
- Did the resulting `spec.md` reflect schema-specific structure?

If any of these proof points is missing, the hook contract is not working as expected for this extension. Capture the failure mode and surface it to the user before declaring Task 9 done.

- [ ] **Step 7: Repeat the test via the `opsmill-speckit/auto` entry point (if installed)**

This is the test that actually validates Fix A — it proves the hook fires when the slash command path is bypassed.

If `opsmill-speckit` is installed in the sandbox session, run:
```
/speckit.opsmill.prep "Create a schema for Devices, Interfaces, and VLANs"
```

(or the equivalent skill-invocation entrypoint). Confirm the same hook fires, the same skill loads, the same directive is emitted. If `opsmill-speckit` is not installed in the sandbox, simulate the skill-direct entry by asking the agent in the session: "Invoke the speckit-specify skill directly with input 'Create a schema for Devices'" and verify the hook still fires.

If the hook fires in this path but not in the slash-command path, something is wrong with hook registration. If the hook fires in the slash-command path but not in this path, Fix A is not actually closing the gap and we need to revisit the design.

- [ ] **Step 8: Cleanup**

```bash
rm -rf "$TMPDIR_SANDBOX"
rm -f /tmp/opsmill-infrahub.zip
```

- [ ] **Step 9: Document smoke-test results**

If everything passed, no commit needed for this task. If you discovered any issues, file them as TODO comments in the README troubleshooting section or open a follow-up issue in the repo before proceeding to Task 10.

---

### Task 10: Open a PR

**Goal:** Push the branch and open a PR against `main` for review.

**Files:** None (PR metadata only).

- [ ] **Step 1: Confirm branch state is clean and all commits are present**

```bash
git status
git log --oneline main..HEAD
```

Expected `git log` output (commits from Tasks 2, 3, 4, 5, 6, 7, 8 — at minimum 7 commits, plus the plan-doc commit if applicable):

- Add extension.yml manifest for hook-based routing
- Add before_specify hook command (route-specify)
- Add before_plan hook command (route-plan)
- Add before_implement hook command (route-implement)
- Remove preset.yml and v2.x wrap command files
- Rewrite README for hook-extension model
- Document v3.0.0 release in CHANGELOG

`git status` must be clean.

- [ ] **Step 2: Push the branch**

```bash
git push -u origin ic-hook-based-routing
```

- [ ] **Step 3: Open the PR**

```bash
gh pr create --title "v3.0: Migrate from preset to hook-based extension" --body "$(cat <<'EOF'
## Summary

- Converts `infrahub-speckit` from a spec-kit preset (`wrap` strategy) to a spec-kit extension that registers `before_specify` / `before_plan` / `before_implement` hooks.
- Fixes the gap where v2.x routing was bypassed when another skill or harness (e.g. `opsmill-speckit/auto`) invoked the `speckit-specify` / `speckit-plan` / `speckit-implement` skills directly without going through the slash command. Hooks fire at the skill-runtime layer and so are honored regardless of caller.
- Extension id is `opsmill-infrahub` (chosen to avoid collision with the standalone `infrahub` JPD/Jira branch validator extension; both can coexist).

## Test plan

- [ ] `python -c "import yaml; yaml.safe_load(open('extension.yml'))"` succeeds
- [ ] `specify extension add --from <this repo>` in a sandbox project installs cleanly
- [ ] After install, `.specify/extensions.yml` in the sandbox shows three new `before_*` hook entries pointing at `opsmill-infrahub`
- [ ] The three `speckit-opsmill-infrahub-route-*` slash commands appear in the slash-command resolver
- [ ] `/speckit.specify "Create a Devices schema"` fires the hook, loads `infrahub-managing-schemas`, and emits the template-override directive
- [ ] Invoking the `speckit-specify` skill directly (bypassing the slash command) fires the same hook — proves Fix A closes the gap

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 4: Return the PR URL to the user**

The `gh pr create` output ends with a URL. Surface it to the user.

---

## Notes on what is NOT in this plan

- **Templates.** This repo does not ship templates (`spec-schema-template.md`, etc.). Those live in the separate optional `infrahub` spec-kit extension. The `route-specify` hook references them by path and falls back to the core `spec-template.md` if missing. If we later decide to bundle templates here, that's a follow-up plan.
- **`opsmill-speckit` (the auto/prep skill).** Fix A's whole point is that `opsmill-speckit` doesn't need to change. PR #3 there continues to invoke `speckit-specify` directly; the hook now fires regardless.
- **Workaround H1 (file swap).** The plan uses Workaround H2 (directive in hook output). If smoke testing surfaces agent-discipline drift around the directive, the fallback is to physically copy the template into `.specify/templates/spec-template.md` from the hook and restore via an `after_specify` hook. That would be a follow-up plan.
- **Catalog publishing.** Once `opsmill-infrahub` is in the public spec-kit catalog, `specify extension add opsmill-infrahub` (no `--from`) will work. README already documents both forms.
