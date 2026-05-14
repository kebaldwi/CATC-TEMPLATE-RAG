---
title: "Authoring — new Composite (*-Composite.yml) descriptor"
engine: shared
kind: prompt
topic: authoring
intent: new-composite
path: prompts/authoring/new-composite-template.md
---

# Authoring — new Composite (*-Composite.yml) descriptor

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
I need a new Catalyst Center **Composite template** (`*-Composite.yml`) that
chains the following member templates in order:

1. {{template name + project path}}
2. {{…}}

Constraints:
- Composite scope: **DayN only** (PnP composites are not supported).
- Tags / device-family filters: {{list}}
- Common variables shared across members: {{list or none}}

Produce a single `.yml` file in the Composite descriptor format.
```

## Required reading (agent must load before answering)

- [docs/templates.md](../../docs/templates.md)
- [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md)
- [rules/velocity/constraints.md](../../rules/velocity/constraints.md)
- [examples/jinja2/Access-DayN-Composite.yml](../../examples/jinja2/Access-DayN-Composite.yml)
- [examples/jinja2/BGP-EVPN-BUILD.yml](../../examples/jinja2/BGP-EVPN-BUILD.yml)

## Agent recipe

1. Refuse if the user wants a PnP composite — explain composites are DayN-only.
2. Open both example `*-Composite.yml` files and copy the structural skeleton.
3. Enumerate members in execution order with each member's project path.
4. Surface any cross-member variables as top-level inputs.

## Output contract

Return one fenced `yaml` code block. Cite the example descriptor you
modelled the structure after.
