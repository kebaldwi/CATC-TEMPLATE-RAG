---
title: "Pattern — EEM applet from a template"
engine: shared
kind: prompt
topic: pattern
intent: eem
path: prompts/patterns/eem-applet.md
---

# Pattern — EEM applet from a template

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
Generate an **EEM applet** to embed in a CC template.

- Engine: {{Jinja2 | Velocity}}
- Trigger: {{cron / syslog pattern / counter / interface state / …}}
- Action: {{cli commands / mail / snmp-trap / …}}
- Should the action template-render at push time, or run literally on the device?
  {{render-at-push / device-side}}
```

## Required reading (agent must load before answering)

- [docs/eem.md](../../docs/eem.md)
- [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md)
- [rules/velocity/constraints.md](../../rules/velocity/constraints.md)
- [examples/jinja2/AutoNaming-EEM-Scripting.j2](../../examples/jinja2/AutoNaming-EEM-Scripting.j2)

## Agent recipe

1. Open `examples/jinja2/AutoNaming-EEM-Scripting.j2` for the canonical wrapping.
2. If the EEM body must include literal `$` (Velocity) or `{{ }}` (Jinja2) characters, escape them per the engine's quoting rules in `rules/<engine>/constraints.md`.
3. Wrap the entire applet body in `<MLTCMD>` … `</MLTCMD>` so CC pushes it as one block.
4. If actions are privileged-EXEC, wrap in `#MODE_ENABLE` / `#MODE_END_ENABLE`.

## Output contract

One fenced code block in the requested engine. Show the resulting CLI in a
second fenced `text` block so the user sees what the device receives.
