---
title: "Debug — empty __device / __interface in PnP"
engine: shared
kind: prompt
topic: debug
intent: pnp-binding
path: prompts/debug/pnp-vs-dayn-binding.md
---

# Debug — empty __device / __interface in PnP

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
My template uses `__device` / `__interface` / `__networkSettings` (or a
bind variable) and the value renders as empty / fails. Why?

- Template type: {{PnP | DayN | Regular}}
- Variable in question: {{name}}

```{{j2|vtl}}
{{relevant excerpt}}
```
```

## Required reading (agent must load before answering)

- [guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md)
- [guardrails/velocity/what-cc-cannot-do.md](../../guardrails/velocity/what-cc-cannot-do.md)
- [docs/system-variables.md](../../docs/system-variables.md)
- [docs/templates.md](../../docs/templates.md)
- [rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md)
- [rules/velocity/variable-types.md](../../rules/velocity/variable-types.md)

## Agent recipe

1. If template type = PnP: explain that bind vars and `__*` system vars are not available pre-Inventory — that is the answer. Stop.
2. If DayN/Regular: check that the variable name matches the published system-variable schema (`docs/system-variables.md`).
3. Check that the device is actually in the expected site / profile so the bind resolves.
4. If a bind variable: confirm the binding category in the Input Form matches the data source.

## Output contract

Single answer paragraph naming the root cause + a fix (move to DayN /
switch to user-input / re-bind / correct the variable name). Cite the
relevant guardrail row.
