---
title: "Templates — PnP & DayN authoring guide"
engine: shared
kind: docs
topic: templates-index
path: docs/templates/README.md
---

# Templates — PnP & DayN authoring guide

Concept docs for **how to design and build** Catalyst Center templates for
Day-0 onboarding (PnP) and Day-N ongoing operations. Distilled from a
five-module lab walkthrough into reusable methodology that an LLM can quote.

For *what kinds of templates Catalyst Center offers* (Regular, Composite,
Onboarding vs DayN, Velocity vs Jinja2) see the broader overview in
[docs/templates.md](../templates.md).

## Files in this folder

| File | Topic | Summary |
|------|-------|---------|
| [considerations.md](considerations.md) | considerations | Universal PnP-vs-DayN rules: inventory unavailable pre-claim, mandatory auto-config, compliance scope, design-vs-template overlap, engine choice. |
| [pnp-template-design.md](pnp-template-design.md) | pnp-template-design | How to design a Day-0 onboarding template: minimum-viable scope, iterative variabilization, form fields, simulation. |
| [pnp-claim-workflow.md](pnp-claim-workflow.md) | pnp-claim | Operationalizing a PnP template: flat-file delivery, Network Profile attachment, claim workflow lifecycle. |
| [dayn-template-design.md](dayn-template-design.md) | dayn-template-design | DayN design methodology: analyze → discount → modularize, Regular vs Composite, `<MLTCMD>`, dictionary/macro patterns, `__interface` loops. |
| [dayn-provisioning.md](dayn-provisioning.md) | dayn-provisioning | DayN provisioning: device TAGs, single-profile-per-site rule, Provision Device workflow. |
| [build-methodology.md](build-methodology.md) | build-methodology | Cross-cutting methodology: DRY, Design-first, iterative build, simulate-before-claim, engine and template-type decision flow. |

## See also

- Authoring prompts:
  [prompts/authoring/new-pnp-onboarding.md](../../prompts/authoring/new-pnp-onboarding.md),
  [prompts/authoring/new-jinja2-template.md](../../prompts/authoring/new-jinja2-template.md),
  [prompts/authoring/new-velocity-template.md](../../prompts/authoring/new-velocity-template.md),
  [prompts/authoring/new-composite-template.md](../../prompts/authoring/new-composite-template.md)
- Debug prompt: [prompts/debug/pnp-vs-dayn-binding.md](../../prompts/debug/pnp-vs-dayn-binding.md)
- Engine syntax: [docs/jinja2/](../jinja2/), [docs/velocity/](../velocity/)
- Rule books: [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md),
  [rules/velocity/constraints.md](../../rules/velocity/constraints.md)
- Guardrails: [guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md),
  [guardrails/velocity/what-cc-cannot-do.md](../../guardrails/velocity/what-cc-cannot-do.md)

## Raw source

The narrative material was distilled from the five lab modules archived under
[docs/source/](../source/). Those originals are kept verbatim as input; the
files in this folder are the cleaned, RAG-ready derivatives.
