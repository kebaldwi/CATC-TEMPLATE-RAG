---
title: "Authoring — new PnP onboarding template"
engine: shared
kind: prompt
topic: authoring
intent: new-pnp
path: prompts/authoring/new-pnp-onboarding.md
---

# Authoring — new PnP onboarding template

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
I need a new Catalyst Center **PnP onboarding** template.

- Engine: {{Jinja2 | Velocity}}
- Target platform: {{C9300 / C9200 / IR1101 / …}}
- Day-0 minimum required: {{hostname, mgmt IP, default route, AAA stub, …}}
- User inputs (filled in via the PnP claim form): {{list}}

Reminder for the assistant: PnP templates **cannot** use bind variables
or `__*` system variables — the device is not yet in Inventory.
```

## Required reading (agent must load before answering)

- [guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md)
- [guardrails/velocity/what-cc-cannot-do.md](../../guardrails/velocity/what-cc-cannot-do.md)
- [docs/templates.md](../../docs/templates.md)
- [rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md)
- [rules/velocity/variable-types.md](../../rules/velocity/variable-types.md)

## Agent recipe

1. Confirm PnP context. If user mentions `__device`, `__interface`, etc., reject and explain.
2. Confirm there are no bind-variable inputs; if there are, reject and explain.
3. Generate the template using only user-input vars and literals.
4. Include only commands that are valid in Day-0 ROMmon-bootstrapped state.

## Output contract

Return a single fenced code block (`j2` or `vtl`). Above it, a 2-sentence
warning about PnP variable restrictions if the user's requirements brushed
against them.
