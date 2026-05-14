---
title: "Jinja2 — What Catalyst Center Cannot Do"
engine: jinja2
kind: guardrails
topic: what-cc-cannot-do
path: guardrails/jinja2/what-cc-cannot-do.md
---

# What Catalyst Center Cannot Do (Jinja2)

Hard platform limits when authoring Jinja2 templates in Catalyst Center.
These are not bugs you can work around with cleverer template code — they
are constraints of the Template Editor / provisioning engine itself.

For Jinja2 syntax-level rules see
[../../rules/jinja2/jinja2-constraints.md](../../rules/jinja2/constraints.md).

---

## 1. PnP templates cannot use bind variables

At PnP onboarding the device has not yet been added to Inventory. Anything
sourced from Inventory / Network Settings / Network Profile / Cloud Connect
is unavailable.

* Affected: every `{% set ... = … bind … %}` form, plus all `__device`,
  `__interface`, `__networkSettings`, `__networkProfile`, `__credentials`,
  `__cloudConnect` references.
* Workaround: collect the values as **user-input** form variables on PnP,
  bind them on the DayN template.

## 2. PnP templates cannot use system variables (`__*`)

Same root cause as §1. `__device.hostname`, `__interface[*].portName`, etc.
all return undefined at PnP time.

## 3. Compliance is not tracked for PnP templates

Catalyst Center's compliance engine starts evaluating against the **DayN**
template. Mandatory PnP config (interface/IP/credentials needed for the
device to phone home) is *not* compliance-checked.

## 4. Mandatory PnP configuration cannot be disabled

The CLI bits Catalyst Center injects to make PnP work (Day-0 hostname / IP /
controller binding) are non-negotiable. You cannot template them away or
override them from your `.j2`.

## 5. Composite templates are DayN-only

The `*-Composite.yml` orchestrator form (which sequences regular templates
under conditions) is **not** valid in a PnP workflow. Use a single regular
`.j2` for PnP.

## 6. No filesystem access from a template

Jinja2 in Catalyst Center has no `open()`, no Python imports, no
`fileSystem`/`url` global. You cannot:

* read a file from disk,
* fetch an HTTP resource,
* shell out.

The only data sources are user-input, bind, and system variables.

## 7. No interactive prompts

A template renders once, server-side, in a single pass. There is no
`prompt()`, no read-from-user, no "ask the operator now".

## 8. No way to read the device's current running-config

The template renders before being pushed; it cannot inspect what is already
on the device. Use **EEM** (embedded in the rendered config) when you need
runtime introspection.

## 9. No template-to-template state

Two templates rendering against the same device share no variables. State
must be encoded into form variables, bind variables, or the running config
itself.

## 10. Macros are not shared globally across templates

A macro defined in template A is not automatically callable from template B.
Use one of:

* `{% import "Project/Lib" as m %}` and call `m.macro()`,
* `{% include "Project/Lib" %}` (renders the included template's output —
  macros defined there become available **for the remainder of this
  template** in CC's Jinja2 environment but not for sibling Composite
  members),
* Composite templates assemble multiple `.j2` files server-side.

There is no notion of a Jinja2 *package* or persistent global library.

## 11. No `break` / `continue` in loops

Catalyst Center does not enable Jinja2's `loopcontrols` extension. Restructure
loops with `{% if %}` guards.

## 12. No safe template abort

Jinja2 has no `#stop` equivalent. Wrapping the entire template body in a
single `{% if precondition %}…{% endif %}` is the only safe abort pattern;
the device still receives whatever fragments were emitted *before* the
guard.

## 13. Strict-undefined is not enforced

Catalyst Center runs Jinja2 in non-strict mode: a typo'd variable name
silently renders as empty. The Template Editor will not flag it. Add
`is defined` guards during authoring.

## 14. Bind-to-Source requires `Single Select` display type

A bind variable that is hidden as `Text` or `Multi Select` will not bind.
Set the display type to `Single Select` (visible to the operator) to make
binding work.

## 15. Bind values are always strings

Even when the **Variable Definition** is `Integer`, the value arrives at
the template as a string. Cast with `| int` before arithmetic / comparison.

## 16. No access to credentials in plaintext from the template

`__credentials` exposes the *binding* but not the raw password material. Use
the credential reference in the IOS command (`aaa group`, `radius server …
key 7 …`) rather than trying to read and print the secret.

## 17. No looping over an unknown number of form variables

Form variables are statically declared. You cannot generate, e.g., 50
optional bind variables on the fly. Use a single Multi Select / structured
data variable instead and iterate that.

## 18. Templates cannot create or modify Catalyst Center objects

A `.j2` renders CLI. It cannot create a site, add a credential, register a
device, schedule a workflow, or call a Catalyst Center API. For
orchestration use **Ansible** or the Catalyst Center API directly.

## 19. No template-level versioning beyond the editor's save history

You can commit one template at a time. The editor's "Versions" tab is the
only built-in history. Use git / external storage for serious version
management.

## 20. No locale-aware or timezone-aware functions

There is no equivalent of `strftime`, no `now()`, no timezone object. If you
need a timestamp in the rendered config, render it on the device with EEM at
push time.

## 21. No regex-based replace / match in templates

Standard Jinja2 has no built-in regex. CC does not add one. For regex
matching, use:

* multiple `replace`/`split`/`startswith`/`in` operations, or
* an EEM action at run time on the device.

## 22. `<MLTCMD>` is a CC marker, not a Jinja2 tag

`<MLTCMD>...</MLTCMD>` is emitted into the rendered text and interpreted by
the **provisioner**, not by Jinja2. You cannot, e.g., put `{% if %}` logic
*around* the angle brackets and expect them to disappear at render time
without emitting their content.

## 23. Compliance ignore markers are literal text only

`! @ start-ignore-compliance` and `! @ end-ignore-compliance` work only when
emitted verbatim into the rendered config. They are not Jinja2 directives
and cannot be parameterised away.

## 24. Differences from Velocity worth knowing

These limitations exist in Velocity but **not** in Jinja2 on Catalyst Center:

| Capability                                | Velocity | Jinja2 |
|-------------------------------------------|----------|--------|
| Cross-template include / inheritance      | No (`#include` / `#parse` blocked) | **Yes** — `{% include %}`, `{% extends %}`, `{% import %}` |
| In-place list mutation                    | Limited  | **Yes** — `{% do list.append(...) %}` |
| Filter pipelines (`| upper`, `| int`, …)  | No       | **Yes** |
| `range()` generator                       | Limited  | **Yes** |

If you are porting from Velocity, prefer `{% import %}` over copy-paste
to consolidate macros.

## See also

* [common-mistakes.md](./common-mistakes.md)
* [invalid-syntax-patterns.md](./invalid-syntax-patterns.md)
* [../../rules/jinja2/jinja2-constraints.md](../../rules/jinja2/constraints.md)
* [../../rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md)
* [./what-cc-cannot-do.md](./what-cc-cannot-do.md) — Velocity-side equivalent.
