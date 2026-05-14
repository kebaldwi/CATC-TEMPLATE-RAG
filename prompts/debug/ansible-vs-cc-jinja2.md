---
title: "Debug — Jinja2 that works in Ansible but not in CC"
engine: jinja2
kind: prompt
topic: debug
intent: ansible-vs-cc
path: prompts/debug/ansible-vs-cc-jinja2.md
---

# Debug — Jinja2 that works in Ansible but not in CC

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
This Jinja2 snippet renders correctly in Ansible but breaks (or renders
differently) inside Catalyst Center. Why?

```j2
{{paste snippet}}
```

Observed CC behaviour: {{empty output / parse error / wrong concat / …}}
```

## Required reading (agent must load before answering)

- [rules/jinja2/cc-vs-ansible.md](../../rules/jinja2/cc-vs-ansible.md)
- [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md)
- [guardrails/jinja2/common-mistakes.md](../../guardrails/jinja2/common-mistakes.md)

## Agent recipe

1. Walk `rules/jinja2/cc-vs-ansible.md` row by row and check each delta against the snippet (operators, `~` vs `+`, strict-undefined, `loopcontrols`, `split`).
2. Identify the specific delta that explains the divergence.
3. Show the corrected CC-portable form.

## Output contract

Two paragraphs + one corrected fenced `j2` block. Cite the exact section
in `rules/jinja2/cc-vs-ansible.md`.
