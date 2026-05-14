---
title: "Prompts — curated chatbot starters"
engine: none
kind: navigation
topic: prompts-index
path: prompts/README.md
---

# Prompts — curated chatbot starters

This folder contains **ready-made prompts** the chatbot can offer the user
when they ask "what can you help me with?" — and the **agent recipes** the
assistant follows internally once one is selected.

Each prompt file has the same shape:

1. **YAML frontmatter** — `title`, `engine`, `kind: prompt`, `topic`, `intent`.
2. **User prompt** — a copy-pasteable template with `{{placeholders}}`.
3. **Required reading** — the exact files the assistant must load before answering.
4. **Agent recipe** — numbered steps the assistant executes.
5. **Output contract** — what the response must look like.

## Catalogue

### Authoring — write a new template

| Prompt | Engine | Use when |
|--------|--------|----------|
| [authoring/new-jinja2-template.md](authoring/new-jinja2-template.md)     | Jinja2   | User wants a fresh Jinja2 DayN template. |
| [authoring/new-velocity-template.md](authoring/new-velocity-template.md) | Velocity | User wants a fresh Velocity DayN template. |
| [authoring/new-composite-template.md](authoring/new-composite-template.md) | shared | User wants a `*-Composite.yml` descriptor chaining members. |
| [authoring/new-pnp-onboarding.md](authoring/new-pnp-onboarding.md)       | shared   | User wants a PnP onboarding template (no bind / no `__*`). |

### Review — validate existing template

| Prompt | Engine | Use when |
|--------|--------|----------|
| [review/validate-jinja2.md](review/validate-jinja2.md)                     | Jinja2   | User pastes a `.j2` and asks "is this OK?" |
| [review/validate-velocity.md](review/validate-velocity.md)                 | Velocity | User pastes a `.vm` and asks "is this OK?" |
| [review/compliance-block-review.md](review/compliance-block-review.md)     | shared   | User wants the compliance-ignore blocks audited. |

### Debug — diagnose failures

| Prompt | Engine | Use when |
|--------|--------|----------|
| [debug/why-does-this-fail.md](debug/why-does-this-fail.md)                 | shared   | User has an error message and wants triage. |
| [debug/ansible-vs-cc-jinja2.md](debug/ansible-vs-cc-jinja2.md)             | Jinja2   | "It works in Ansible but not in CC." |
| [debug/pnp-vs-dayn-binding.md](debug/pnp-vs-dayn-binding.md)               | shared   | `__device` / bind var is empty. |

### Patterns — common building blocks

| Prompt | Engine | Use when |
|--------|--------|----------|
| [patterns/vlan-list-of-dicts.md](patterns/vlan-list-of-dicts.md)           | Jinja2   | Iterate a list of VLAN dicts. |
| [patterns/eem-applet.md](patterns/eem-applet.md)                           | shared   | Embed an EEM applet from a template. |
| [patterns/mltcmd-banner.md](patterns/mltcmd-banner.md)                     | shared   | Multi-line banner / cert wrapped in `<MLTCMD>`. |
| [patterns/interface-macros.md](patterns/interface-macros.md)               | Jinja2   | Per-port macro suite à la `Platinum-PortAssign`. |

## How a chatbot should use this folder

When the user opens a session and asks an open-ended question, surface a
short menu of these prompt titles grouped by category. Once the user picks
one (or the assistant infers it), load the file, execute the **Agent
recipe**, and respond per the **Output contract**.

See [../AGENTS.md](../AGENTS.md) for repo-wide agent rules.
