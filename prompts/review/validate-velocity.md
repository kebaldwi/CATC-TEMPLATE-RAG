---
title: "Review — validate a Velocity template"
engine: velocity
kind: prompt
topic: review
intent: validate
path: prompts/review/validate-velocity.md
---

# Review — validate a Velocity template

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
Please review the following Catalyst Center **Velocity** template.

Context:
- Template type: {{Regular | DayN | PnP | Composite member}}
- CC version: {{…}}

```vtl
{{paste template here}}
```
```

## Required reading (agent must load before answering)

- [rules/velocity/constraints.md](../../rules/velocity/constraints.md)
- [guardrails/velocity/common-mistakes.md](../../guardrails/velocity/common-mistakes.md)
- [guardrails/velocity/invalid-syntax-patterns.md](../../guardrails/velocity/invalid-syntax-patterns.md)
- [guardrails/velocity/what-cc-cannot-do.md](../../guardrails/velocity/what-cc-cannot-do.md)

## Agent recipe

1. Pass 1 — Parse: invalid-syntax-patterns.md.
2. Pass 2 — Semantics: common-mistakes.md.
3. Pass 3 — Platform: what-cc-cannot-do.md (special focus: `#parse`, `#include` are disabled).
4. Tabulate findings.

## Output contract

Markdown table of findings then a corrected fenced `vtl` block. Cite
rule sections explicitly.
