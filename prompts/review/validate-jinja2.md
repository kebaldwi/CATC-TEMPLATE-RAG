---
title: "Review — validate a Jinja2 template"
engine: jinja2
kind: prompt
topic: review
intent: validate
path: prompts/review/validate-jinja2.md
---

# Review — validate a Jinja2 template

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
Please review the following Catalyst Center **Jinja2** template for
syntactic legality, semantic correctness, and platform-policy compliance.

Context:
- Template type: {{Regular | DayN | PnP | Composite member}}
- CC version: {{e.g. 2.3.7 / 3.1.x}}

```j2
{{paste template here}}
```
```

## Required reading (agent must load before answering)

- [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md)
- [guardrails/jinja2/common-mistakes.md](../../guardrails/jinja2/common-mistakes.md)
- [guardrails/jinja2/invalid-syntax-patterns.md](../../guardrails/jinja2/invalid-syntax-patterns.md)
- [guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md)
- [rules/jinja2/cc-vs-ansible.md](../../rules/jinja2/cc-vs-ansible.md)

## Agent recipe

1. Pass 1 — Parse: flag every construct that matches a row in `guardrails/jinja2/invalid-syntax-patterns.md`.
2. Pass 2 — Semantics: flag every construct that matches `guardrails/jinja2/common-mistakes.md`.
3. Pass 3 — Platform: flag every construct that matches `guardrails/jinja2/what-cc-cannot-do.md`.
4. Pass 4 — Ansible-isms: if the template uses `+` for concat, `is defined` without a guard, etc., reference `rules/jinja2/cc-vs-ansible.md`.
5. Tabulate findings: line | severity (error/warn/info) | rule cite | suggested fix.

## Output contract

Return a markdown table of findings, then a corrected fenced `j2` block.
If the template is clean, say so explicitly and cite the rule sections
that were checked.
