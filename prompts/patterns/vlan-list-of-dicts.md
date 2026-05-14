---
title: "Pattern — VLAN list-of-dicts (Jinja2)"
engine: jinja2
kind: prompt
topic: pattern
intent: list-of-dicts
path: prompts/patterns/vlan-list-of-dicts.md
---

# Pattern — VLAN list-of-dicts (Jinja2)

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
Generate a Jinja2 block that iterates a **list of VLAN dictionaries** and
renders the IOS VLAN configuration. Each dict has keys:

- `id` (int)
- `name` (str)
- `gateway` (str, optional)
- `dhcp_helper` (list of str, optional)
- `stp_priority` (int, optional)

Source variable name: `{{var name, e.g. VLANS}}`.
```

## Required reading (agent must load before answering)

- [docs/jinja2/dictionaries.md](../../docs/jinja2/dictionaries.md)
- [rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md)
- [examples/jinja2/VLAN-Configuration.j2](../../examples/jinja2/VLAN-Configuration.j2)

## Agent recipe

1. Open `examples/jinja2/VLAN-Configuration.j2` and lift the canonical loop shape.
2. Use `{% for v in VLANS %}` and `{% if v.gateway is defined %}` guards (non-strict undefined).
3. Use `~` not `+` for string concatenation if any is needed.
4. Wrap any multi-line interface stanza output in real CLI form (no `<MLTCMD>` needed for plain config).

## Output contract

One fenced `j2` block. Below it, cite the source example file and the
rule section justifying the `is defined` guards.
