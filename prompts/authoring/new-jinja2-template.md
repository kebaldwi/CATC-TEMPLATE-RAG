---
title: "Authoring — new Jinja2 DayN template"
engine: jinja2
kind: prompt
topic: authoring
intent: new-template
path: prompts/authoring/new-jinja2-template.md
---

# Authoring — new Jinja2 DayN template

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
I need a new Catalyst Center **Jinja2 DayN** template that does the following:

- Purpose: {{one-line purpose, e.g. configure NTP + timezone}}
- Target platforms: {{IOS-XE 17.x / C9300 / …}}
- User inputs:
    - {{name}} ({{type}}, {{default}}, {{description}})
- Bind variables (Inventory / Settings / Profile / Cloud): {{list or none}}
- Must include `#MODE_ENABLE` block?: {{yes/no}}
- Compliance-ignore block?: {{yes/no, which lines}}
- Other constraints: {{e.g. must be idempotent, must roll back cleanly}}

Produce a single `.j2` file. Show me both the Input Form variable
declarations (as a comment block at the top) and the body.
```

## Required reading (agent must load before answering)

- [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md)
- [rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md)
- [guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md)
- [docs/jinja2/basics.md](../../docs/jinja2/basics.md)
- [docs/templates.md](../../docs/templates.md)

## Agent recipe

1. Confirm engine = Jinja2, template type = DayN (not PnP).
2. Load the four required-reading files above.
3. If `bind variables` are present, restate the binding category (Inventory / Settings / NetworkProfile / CloudConnect).
4. Draft the template: variable comment header → `{% if … %}` guards on optional inputs → CLI body.
5. Wrap privileged-EXEC commands in `#MODE_ENABLE` / `#MODE_END_ENABLE` if `yes`.
6. Wrap any unmanaged CLI in `! @ start-ignore-compliance` / `! @ end-ignore-compliance` if `yes`.
7. Verify against `guardrails/jinja2/invalid-syntax-patterns.md` before returning.

## Output contract

Return exactly one fenced `j2` code block. Above the code block, a 2–4
sentence rationale citing rule sections (e.g. `rules/jinja2/constraints.md §8`).
Below the code block, list every assumption you made about the placeholders.
