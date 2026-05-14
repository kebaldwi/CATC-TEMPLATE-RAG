---
title: "Review — audit ! @ start-ignore-compliance usage"
engine: shared
kind: prompt
topic: review
intent: compliance-audit
path: prompts/review/compliance-block-review.md
---

# Review — audit ! @ start-ignore-compliance usage

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
Audit every `! @ start-ignore-compliance` / `! @ end-ignore-compliance`
block in the following template. For each block, judge whether it is
**justified** (CLI is genuinely outside the CC compliance model) or
**unjustified** (the CLI should be tracked and the block removed).

```{{j2|vtl}}
{{paste template here}}
```
```

## Required reading (agent must load before answering)

- [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md)
- [rules/velocity/constraints.md](../../rules/velocity/constraints.md)
- [guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md)
- [guardrails/velocity/what-cc-cannot-do.md](../../guardrails/velocity/what-cc-cannot-do.md)

## Agent recipe

1. Locate every `start-ignore-compliance` / `end-ignore-compliance` pair.
2. For each pair, classify the wrapped CLI: certs, banners, key-chains, dynamic-generated, or other.
3. Mark `certs`, `banners`, `key-chains`, and any keyed material as **justified**.
4. Mark everything else **unjustified** with a suggested replacement.

## Output contract

Markdown table: block# | start-line | end-line | wrapped-content-summary |
verdict | rationale.
