---
title: "Pattern — per-port interface macros (à la Platinum-PortAssign)"
engine: jinja2
kind: prompt
topic: pattern
intent: interface-macros
path: prompts/patterns/interface-macros.md
---

# Pattern — per-port interface macros (à la Platinum-PortAssign)

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
Generate a **macro-based per-port assignment** module in Jinja2.

- Port roles to support: {{e.g. access-data, access-voice, uplink-trunk, ap-trunk}}
- Per-role parameters: {{list of dict keys for each role}}
- Source variable name (list of port dicts): {{e.g. PORTS}}
```

## Required reading (agent must load before answering)

- [docs/jinja2/advanced.md](../../docs/jinja2/advanced.md)
- [docs/jinja2/dictionaries.md](../../docs/jinja2/dictionaries.md)
- [examples/jinja2/Platinum-PortAssign-Template.j2](../../examples/jinja2/Platinum-PortAssign-Template.j2)
- [examples/jinja2/Interfaces-Macros.j2](../../examples/jinja2/Interfaces-Macros.j2)
- [rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md)

## Agent recipe

1. Open `examples/jinja2/Platinum-PortAssign-Template.j2` and `examples/jinja2/Interfaces-Macros.j2` to model the macro signatures.
2. Declare one `{% macro role_xxx(port) %}` per role.
3. Iterate `{% for p in PORTS %}` and dispatch by `p.role` with `{% if %}` / `{% elif %}`.
4. Add `is defined` guards for every optional key.

## Output contract

One fenced `j2` block. Cite the example file you modelled it after and
the rule section justifying the `is defined` guards.
