---
title: "Debug — diagnose a render or push error"
engine: shared
kind: prompt
topic: debug
intent: diagnose-error
path: prompts/debug/why-does-this-fail.md
---

# Debug — diagnose a render or push error

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
My Catalyst Center template fails. Help me diagnose it.

- Engine: {{Jinja2 | Velocity}}
- Template type: {{Regular | DayN | PnP | Composite}}
- Error message (verbatim): {{paste}}
- When does it fail? {{render-time / push-time / on a specific device}}

Template:
```{{j2|vtl}}
{{paste}}
```
```

## Required reading (agent must load before answering)

- [guardrails/jinja2/invalid-syntax-patterns.md](../../guardrails/jinja2/invalid-syntax-patterns.md)
- [guardrails/jinja2/common-mistakes.md](../../guardrails/jinja2/common-mistakes.md)
- [guardrails/velocity/invalid-syntax-patterns.md](../../guardrails/velocity/invalid-syntax-patterns.md)
- [guardrails/velocity/common-mistakes.md](../../guardrails/velocity/common-mistakes.md)
- [guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md)
- [guardrails/velocity/what-cc-cannot-do.md](../../guardrails/velocity/what-cc-cannot-do.md)

## Agent recipe

1. Match the error string against the `Error message` columns in the invalid-syntax-patterns file for the declared engine.
2. If no match: scan the template for any `common-mistakes` entry whose `Bad` pattern is present.
3. If still no match: check `what-cc-cannot-do.md` for a platform restriction (PnP binding, `#parse`, etc.).
4. Quote the matching guardrail entry verbatim, then propose the corrected snippet.

## Output contract

Three sections: (1) `## Diagnosis` — one paragraph + cited guardrail row;
(2) `## Fixed snippet` — fenced code block; (3) `## Why this happened` —
one paragraph for the user's mental model.
