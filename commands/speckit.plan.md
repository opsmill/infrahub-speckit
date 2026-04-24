---
description: Execute the implementation planning workflow using the plan template to generate design artifacts. Infrahub-aware — invokes the matching Infrahub skill before Phase 0 research.
handoffs:
  - label: Create Tasks
    agent: speckit.tasks
    prompt: Break the plan into tasks
    send: true
  - label: Create Checklist
    agent: speckit.checklist
    prompt: Create a checklist for the following domain...
---

## Infrahub Skill Integration (pre-core)

**MANDATORY — do NOT skip, defer, or rationalize around this.**

Before generating any design artifacts (research.md, data-model.md, contracts/), invoke the Infrahub skill that matches the feature's artifact type. The skill loads authoritative reference material that design decisions MUST be consistent with.

### Detection

1. Read the feature spec (`spec.md`) from the current feature directory.
2. Classify the Infrahub artifact type from the spec content:

   | Artifact Type | Signals in Spec | Skill to Invoke |
   |---------------|-----------------|-----------------|
   | **Schema**    | schema, data model, nodes, attributes, relationships, generics, namespace, kind, CoreFileObject, inherit_from | `infrahub:schema-creator` |
   | **Transform** | transform, render, config, artifact, jinja, device config | `infrahub:transform-creator` |
   | **Check**     | check, validate, validation, enforce, rule, constraint | `infrahub:check-creator` |
   | **Generator** | generator, auto-create, design-driven, topology, provision | `infrahub:generator-creator` |
   | **Menu**      | menu, navigation, sidebar, UI | `infrahub:menu-creator` |

3. If the spec involves multiple artifact types, invoke the skill for the PRIMARY type (the one being designed in this cycle).
4. If the repository has no `.infrahub.yml`, skip this section entirely and proceed to the core workflow.

### Invocation

Invoke the matched skill using the Skill tool BEFORE Phase 0 research begins. Use the skill's reference material when:

- Writing `data-model.md` (entity definitions, attribute kinds, relationship types)
- Making research decisions (SDK patterns, API conventions, naming rules)
- Validating the design against Infrahub capabilities

**Anti-rationalization check** — if you think "I already researched this via WebFetch" or "I know Infrahub well enough", invoke the skill anyway. The skill has curated reference material that general knowledge and web fetches do not provide.

---

{CORE_TEMPLATE}
