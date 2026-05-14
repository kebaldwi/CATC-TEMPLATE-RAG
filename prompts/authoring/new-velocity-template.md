---
title: "Authoring — new Velocity DayN template"
engine: velocity
kind: prompt
topic: authoring
intent: new-template
path: prompts/authoring/new-velocity-template.md
---

# Authoring — new Velocity DayN template

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
I need a new Catalyst Center **Velocity DayN** template that does the following:

- Purpose: {{one-line purpose}}
- Target platforms: {{IOS-XE 17.x / C9300 / …}}
- User inputs:
    - {{name}} ({{type}}, {{default}}, {{description}})
- Bind variables: {{list or none}}
- Use `#macro` for repeated blocks?: {{yes/no}}
- Compliance-ignore block?: {{yes/no}}

Produce a single `.vm` file. Macros (if any) must precede their first call.
```

## Required reading (agent must load before answering)

- [rules/velocity/constraints.md](../../rules/velocity/constraints.md)
- [rules/velocity/variable-types.md](../../rules/velocity/variable-types.md)
- [guardrails/velocity/what-cc-cannot-do.md](../../guardrails/velocity/what-cc-cannot-do.md)
- [docs/velocity/basics.md](../../docs/velocity/basics.md)
- [docs/templates.md](../../docs/templates.md)

## Agent recipe

1. Confirm engine = Velocity, template type = DayN.
2. Load the required-reading files.
3. Reject any request to use `#parse` or `#include` — disabled in CC.
4. Define `#macro` blocks first, then the body.
5. Reference user-input vars as `$Foo`, system vars as `$__device.…`.
6. Verify against `guardrails/velocity/invalid-syntax-patterns.md`.

## Output contract

Return one fenced `vtl` code block, preceded by a short rationale citing
rule sections and followed by a list of assumptions.
