---
title: "Pattern — multi-line banner wrapped in <MLTCMD>"
engine: shared
kind: prompt
topic: pattern
intent: mltcmd
path: prompts/patterns/mltcmd-banner.md
---

# Pattern — multi-line banner wrapped in <MLTCMD>

## User prompt

Copy, fill in the placeholders, and send to the assistant.

```text
Generate the CLI for a **multi-line banner** (or cert / key-chain) using
the `<MLTCMD>` … `</MLTCMD>` wrapper that CC requires for multi-line
blocks.

- Engine: {{Jinja2 | Velocity}}
- Banner type: {{login | motd | exec | incoming}}
- Body (free text, may contain newlines and special chars): {{paste}}
```

## Required reading (agent must load before answering)

- [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md)
- [rules/velocity/constraints.md](../../rules/velocity/constraints.md)
- [guardrails/jinja2/common-mistakes.md](../../guardrails/jinja2/common-mistakes.md)
- [guardrails/velocity/common-mistakes.md](../../guardrails/velocity/common-mistakes.md)

## Agent recipe

1. Open the `#MODE_ENABLE` / `<MLTCMD>` section in the engine's `constraints.md`.
2. Wrap the banner body in `<MLTCMD>` … `</MLTCMD>`.
3. Pick a delimiter char that does not appear in the body (e.g. `^C`).
4. Add `! @ start-ignore-compliance` around the banner if the body contains tokens that would otherwise trip compliance.

## Output contract

One fenced code block. Highlight which line(s) are the `<MLTCMD>` open /
close and which is the banner delimiter.
