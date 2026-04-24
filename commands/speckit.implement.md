---
description: Execute the implementation plan by processing and executing all tasks defined in tasks.md. Infrahub-aware — invokes the matching Infrahub skill before tasks that touch schemas, transforms, checks, generators, menus, or seed objects.
---

## Infrahub Skill Integration (pre-core)

**MANDATORY — do NOT skip, defer, or rationalize around this.**

Before implementing any task that touches Infrahub artifacts, invoke the corresponding Infrahub skill. The skill loads authoritative reference material, validation rules, and implementation patterns that the code MUST follow.

### Detection

1. Read `tasks.md` and `plan.md` from the current feature directory.
2. Before starting each implementation phase, check whether any task in that phase touches an Infrahub artifact:

   | Task Involves | Skill to Invoke | When to Invoke |
   |---------------|-----------------|----------------|
   | Schema YAML files (`schemas/`)          | `infrahub:schema-creator`    | Before the first schema task |
   | Python transforms (`transforms/`)       | `infrahub:transform-creator` | Before writing transform code |
   | Python checks                           | `infrahub:check-creator`     | Before writing check code |
   | Generator Python files (`generators/`)  | `infrahub:generator-creator` | Before writing generator code |
   | Menu YAML files (`menus/`)              | `infrahub:menu-creator`      | Before writing menu definitions |
   | Object data files (`objects/`)          | `infrahub:object-creator`    | Before writing seed data |

3. Invoke each skill ONCE per artifact type per implementation session — you do not need to re-invoke for each task of the same type.
4. If the repository has no `.infrahub.yml`, skip this section entirely and proceed to the core workflow.

### Invocation Timing

- **Phase with schema tasks**: Invoke `infrahub:schema-creator` before the FIRST schema task.
- **Phase with generator tasks**: Invoke `infrahub:generator-creator` before the FIRST generator task.
- **Phase with transform tasks**: Invoke `infrahub:transform-creator` before the FIRST transform task.
- **Multiple artifact types in one phase**: Invoke each relevant skill before its first task.

**Anti-rationalization check** — if you think "I already loaded the skill during /speckit.specify" or "the plan has all the details I need", invoke the skill anyway. Skill content is NOT persisted across commands; each command session starts fresh.

---

{CORE_TEMPLATE}
