---
title: "INDEX — repo manifest"
engine: none
kind: navigation
topic: manifest
path: INDEX.md
---

# INDEX — Machine-readable manifest

Deterministic listing of every documentation, rule, guardrail, and prompt
file in this repo. AI agents and retrieval pipelines should ingest this
first to build a routing table, then fetch the underlying files on demand.

See [README.md](README.md) for the human overview and [AGENTS.md](AGENTS.md)
for autonomous-agent operating instructions.

## Navigation

| Path | Engine | Kind | Topic | Title |
|------|--------|------|-------|-------|
| [prompts/README.md](prompts/README.md) | none | navigation | prompts-index | Prompts — curated chatbot starters |
| [images/README.md](images/README.md) | none | navigation | images-manifest | Images — reference screenshots manifest |

## Rules (authoritative syntax & semantics)

| Path | Engine | Kind | Topic | Title |
|------|--------|------|-------|-------|
| [rules/jinja2/cc-vs-ansible.md](rules/jinja2/cc-vs-ansible.md) | jinja2 | rules | cc-vs-ansible | Jinja2 — Catalyst Center Jinja2 vs. Ansible Jinja2 |
| [rules/jinja2/constraints.md](rules/jinja2/constraints.md) | jinja2 | rules | constraints | Jinja2 — Syntax & Semantic Constraints |
| [rules/jinja2/variable-types.md](rules/jinja2/variable-types.md) | jinja2 | rules | variable-types | Jinja2 — Variable Types |
| [rules/velocity/constraints.md](rules/velocity/constraints.md) | velocity | rules | constraints | Velocity — Syntax & Semantic Constraints |
| [rules/velocity/variable-types.md](rules/velocity/variable-types.md) | velocity | rules | variable-types | Velocity — Variable Types |

## Guardrails (failure-mode catalogues)

| Path | Engine | Kind | Topic | Title |
|------|--------|------|-------|-------|
| [guardrails/jinja2/common-mistakes.md](guardrails/jinja2/common-mistakes.md) | jinja2 | guardrails | common-mistakes | Jinja2 — Common Mistakes |
| [guardrails/jinja2/invalid-syntax-patterns.md](guardrails/jinja2/invalid-syntax-patterns.md) | jinja2 | guardrails | invalid-syntax-patterns | Jinja2 — Invalid Syntax Patterns |
| [guardrails/jinja2/what-cc-cannot-do.md](guardrails/jinja2/what-cc-cannot-do.md) | jinja2 | guardrails | what-cc-cannot-do | Jinja2 — What Catalyst Center Cannot Do |
| [guardrails/velocity/common-mistakes.md](guardrails/velocity/common-mistakes.md) | velocity | guardrails | common-mistakes | Velocity — Common Mistakes |
| [guardrails/velocity/invalid-syntax-patterns.md](guardrails/velocity/invalid-syntax-patterns.md) | velocity | guardrails | invalid-syntax-patterns | Velocity — Invalid Syntax Patterns |
| [guardrails/velocity/what-cc-cannot-do.md](guardrails/velocity/what-cc-cannot-do.md) | velocity | guardrails | what-cc-cannot-do | Velocity — What Catalyst Center Cannot Do |

## Documentation (narrative tutorials)

| Path | Engine | Kind | Topic | Title |
|------|--------|------|-------|-------|
| [docs/eem.md](docs/eem.md) | shared | docs | eem | Embedded Event Manager (EEM) |
| [docs/jinja2/advanced.md](docs/jinja2/advanced.md) | jinja2 | docs | advanced | Jinja2 — Advanced Patterns |
| [docs/jinja2/basics.md](docs/jinja2/basics.md) | jinja2 | docs | basics | Jinja2 — Scripting Basics |
| [docs/jinja2/dictionaries.md](docs/jinja2/dictionaries.md) | jinja2 | docs | dictionaries | Jinja2 — Dictionaries & Lists-of-Dictionaries |
| [docs/system-variables.md](docs/system-variables.md) | shared | docs | system-variables | System Variables Deep Dive |
| [docs/templates.md](docs/templates.md) | shared | docs | templates | Catalyst Center Template Types |
| [docs/templates/README.md](docs/templates/README.md) | shared | docs | templates-index | Templates — PnP & DayN authoring guide |
| [docs/templates/build-methodology.md](docs/templates/build-methodology.md) | shared | docs | build-methodology | Templates — Build methodology and decision flow |
| [docs/templates/considerations.md](docs/templates/considerations.md) | shared | docs | considerations | Templates — PnP vs DayN considerations |
| [docs/templates/dayn-provisioning.md](docs/templates/dayn-provisioning.md) | shared | docs | dayn-provisioning | Templates — DayN provisioning workflow |
| [docs/templates/dayn-template-design.md](docs/templates/dayn-template-design.md) | shared | docs | dayn-template-design | Templates — DayN template design methodology |
| [docs/templates/pnp-claim-workflow.md](docs/templates/pnp-claim-workflow.md) | shared | docs | pnp-claim | Templates — PnP claim workflow and operationalization |
| [docs/templates/pnp-template-design.md](docs/templates/pnp-template-design.md) | shared | docs | pnp-template-design | Templates — PnP (Day-0) onboarding template design |
| [docs/variables.md](docs/variables.md) | shared | docs | variables | Variables Tutorial |
| [docs/velocity/advanced.md](docs/velocity/advanced.md) | velocity | docs | advanced | Velocity — Advanced Patterns |
| [docs/velocity/basics.md](docs/velocity/basics.md) | velocity | docs | basics | Velocity — Scripting Basics |

## Prompts (curated chatbot starters)

| Path | Engine | Kind | Topic | Title |
|------|--------|------|-------|-------|
| [prompts/authoring/new-composite-template.md](prompts/authoring/new-composite-template.md) | shared | prompt | authoring | Authoring — new Composite (*-Composite.yml) descriptor |
| [prompts/authoring/new-jinja2-template.md](prompts/authoring/new-jinja2-template.md) | jinja2 | prompt | authoring | Authoring — new Jinja2 DayN template |
| [prompts/authoring/new-pnp-onboarding.md](prompts/authoring/new-pnp-onboarding.md) | shared | prompt | authoring | Authoring — new PnP onboarding template |
| [prompts/authoring/new-velocity-template.md](prompts/authoring/new-velocity-template.md) | velocity | prompt | authoring | Authoring — new Velocity DayN template |
| [prompts/debug/ansible-vs-cc-jinja2.md](prompts/debug/ansible-vs-cc-jinja2.md) | jinja2 | prompt | debug | Debug — Jinja2 that works in Ansible but not in CC |
| [prompts/debug/pnp-vs-dayn-binding.md](prompts/debug/pnp-vs-dayn-binding.md) | shared | prompt | debug | Debug — empty __device / __interface in PnP |
| [prompts/debug/why-does-this-fail.md](prompts/debug/why-does-this-fail.md) | shared | prompt | debug | Debug — diagnose a render or push error |
| [prompts/patterns/eem-applet.md](prompts/patterns/eem-applet.md) | shared | prompt | pattern | Pattern — EEM applet from a template |
| [prompts/patterns/interface-macros.md](prompts/patterns/interface-macros.md) | jinja2 | prompt | pattern | Pattern — per-port interface macros (à la Platinum-PortAssign) |
| [prompts/patterns/mltcmd-banner.md](prompts/patterns/mltcmd-banner.md) | shared | prompt | pattern | Pattern — multi-line banner wrapped in <MLTCMD> |
| [prompts/patterns/vlan-list-of-dicts.md](prompts/patterns/vlan-list-of-dicts.md) | jinja2 | prompt | pattern | Pattern — VLAN list-of-dicts (Jinja2) |
| [prompts/review/compliance-block-review.md](prompts/review/compliance-block-review.md) | shared | prompt | review | Review — audit ! @ start-ignore-compliance usage |
| [prompts/review/validate-jinja2.md](prompts/review/validate-jinja2.md) | jinja2 | prompt | review | Review — validate a Jinja2 template |
| [prompts/review/validate-velocity.md](prompts/review/validate-velocity.md) | velocity | prompt | review | Review — validate a Velocity template |

## Example templates

Real-world Catalyst Center templates. Read-only reference — copy patterns,
do not modify in place.

- **examples/jinja2/** — 73 files (`.j2`, `.yml`, `.json`)
- **examples/velocity/** — 14 files (`.vm`)

Notable Jinja2 examples:

- [examples/jinja2/VLAN-Configuration.j2](examples/jinja2/VLAN-Configuration.j2) — dual-uplink dictionaries / list-of-dicts
- [examples/jinja2/AutoNaming-EEM-Scripting.j2](examples/jinja2/AutoNaming-EEM-Scripting.j2) — EEM applet from a template
- [examples/jinja2/Platinum-PortAssign-Template.j2](examples/jinja2/Platinum-PortAssign-Template.j2) — port-assignment macro suite
- [examples/jinja2/Access-DayN-Composite.yml](examples/jinja2/Access-DayN-Composite.yml) — composite-template descriptor

Notable Velocity examples:

- [examples/velocity/Platinum-AAA-Template.vm](examples/velocity/Platinum-AAA-Template.vm)
- [examples/velocity/Platinum-Stacking-Template.vm](examples/velocity/Platinum-Stacking-Template.vm)

