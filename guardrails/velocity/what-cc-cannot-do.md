---
title: "Velocity — What Catalyst Center Cannot Do"
engine: velocity
kind: guardrails
topic: what-cc-cannot-do
path: guardrails/velocity/what-cc-cannot-do.md
---

# What Catalyst Center Cannot Do — Template Platform Limits

Hard limits and platform constraints. Even when a Velocity (or Jinja2) idiom is
valid in the upstream engine, Catalyst Center may sandbox, ignore, or
not-yet-implement it. Author templates against these limits up front.

Companion files:

* [common-mistakes.md](./common-mistakes.md) — wrong behaviour with valid syntax.
* [invalid-syntax-patterns.md](./invalid-syntax-patterns.md) — pure syntax errors.
* [../../docs/templates.md](../../docs/templates.md) — Catalyst Center template lifecycle.

---

## 1. No filesystem or network access from a template

Templates render in a sandbox. There is no I/O.

| Standard Velocity directive | In Catalyst Center      |
|-----------------------------|-------------------------|
| `#include( "file.txt" )`    | Not usable.             |
| `#parse( "tmpl.vm" )`       | Not usable.             |
| `#evaluate( ... )`          | Not exposed.            |
| `#define( ... )`            | Not exposed.            |

For composition, use:

* Multiple **Regular Templates** attached to the Network Profile, or
* A **Composite Template** (DayN only — see §5).

## 2. No custom Java classes

Velocity normally allows arbitrary Java method calls. Catalyst Center exposes
only the introspection surface of its template context — primarily the
system-variable objects (`$__device`, `$__interface`, …) and the standard
methods on Java strings, lists, and maps. You cannot import classes, register
tools (`EscapeTool`, `MathTool`, `DateTool`), or instantiate objects.

## 3. Bind and system variables are unavailable during PnP onboarding

The Inventory database is not populated until **after** the claim process. In
an Onboarding template:

* `$__device`, `$__interface`, `$__networkSettings`, `$__credentials`, … all
  resolve to empty / null.
* Variables marked **Bind to Source** are not populated.

Use **user-input** variables only in PnP. Reserve bind / system variables for
DayN templates. From [../../docs/templates.md](../../docs/templates.md):

> "Inventory Database Limitations: The inventory database is inaccessible
> before the claim process… Using onboarding templates post-PnP is impractical
> due to the absence of system and bind variables."

## 4. No compliance tracking on PnP templates

Compliance only kicks in after a device is in inventory. Anything in a PnP
template is fire-and-forget — keep it minimal (just enough to bring the device
up) and put the rest in DayN.

## 5. Composite templates are DayN-only

You cannot attach a composite template to a PnP workflow. Composites group
Regular Templates and provide ordering; if you need that during onboarding,
attach the Regular Templates individually.

## 6. The claim-time mandatory config cannot be disabled

Catalyst Center pushes a fixed baseline during PnP claim
(see [../../docs/templates.md](../../docs/templates.md) for the verbatim list). It
includes, among other things:

* `archive log config` block
* `service timestamps`, `service password-encryption`, `service sequence-numbers`
* `no ip http server`, `no ip http secure-server`
* `crypto key generate rsa label dnac-sda modulus 2048`, `ip ssh version 2`
* `vtp mode transparent`
* `spanning-tree mode rapid-pvst`, `spanning-tree extend system-id`
* `no udld enable`
* `errdisable recovery cause all` / `interval 300`
* `snmp-server` + `username` + `enable secret` + `hostname Switch`
* `ip domain name` / `ip name-server`

If you need different values for any of those, push the *correction* in a DayN
template (e.g., `udld enable`, `errdisable recovery interval 30`).

## 7. No interactive prompting at render time

A template cannot pause for input. All operator data is collected up-front
through the Input Form (or API payload). Plan defaults and `Single Select`
choices accordingly.

## 8. No read access to a device's running configuration

You cannot read the live running-config inside a template. Only inventory
attributes that Catalyst Center has already collected and exposed as system
variables are visible.

## 9. No persistent state between template runs

Each render is stateless. Counters, run-IDs, "did this template execute"
flags must come from outside (Inventory tags, custom Common Settings, etc.).

## 10. No global Velocimacro library

In standard Velocity, `velocimacro.library` lets you share macros across
templates. Catalyst Center does **not** expose this. A `#macro` is visible only
within the template that defines it.

## 11. Strict mode is engine-controlled

You cannot toggle `runtime.strict_mode.enable` from a template. Catalyst Center
runs non-strict, so undefined references render literally (e.g., `$missing` →
`$missing`). Guard with `$!ref` or `#if( $ref )` rather than relying on a
thrown exception.

## 12. `parser.allow_hyphen_in_identifiers` is **off**

`-` is always an arithmetic / CLI character — never part of an identifier. See
[../../rules/velocity/constraints.md#identifiers](../../rules/velocity/constraints.md#identifiers).

## 13. Template Editor device picker hides PIDs

The picker shows device series and model description, not Product IDs. Look up
the PID on cisco.com, then map it back to series/model for selection.

## 14. Tag-based filtering requires matching device tags

If you filter templates by tag, every target device must also carry the tag,
or you get:

> Cannot select the device. Not compatible with template.

## 15. `#stop` is not a safe abort

`#stop` halts rendering, but the partial output produced so far is **still
sent to the device**. Use a wrapping `#if( $do_provision ) … #end` instead.

## 16. Bind-to-Source requires Single Select

In the Input Form, only the `Single Select` display type supports **Bind to
Source**. `Text` and `Multi Select` do not — the bind silently fails to
populate.

## 17. Software-Type mismatch silently skips the template

When provisioning, Catalyst Center checks the target device against the
template's *Software Type* / *Software Version*. A mismatch causes the
template to be **skipped** for that device with no rendering error. Configure
the type correctly when you create the template.

## 18. No `#evaluate` ⇒ no dynamic VTL strings

Patterns that work in upstream Velocity to "compile a string at render time"
will not work:

```vtl
## Not supported in Catalyst Center
#set( $src = "switchport mode $mode" )
#evaluate( $src )
```

Use a normal `#set` with double-quoted interpolation instead.

## 19. No `#include` for static text

There is no way to splice in a pre-canned file at render time. If you need
common banners or ACL boilerplate, paste them into the template or share them
via a Composite Template.

## 20. Catalyst Center–specific markers are non-standard VTL

These are CC extensions, not Apache Velocity directives. They will not render
correctly in any other Velocity engine, and should not be expected to behave
identically in other tools:

* `#MODE_ENABLE` / `#MODE_END_ENABLE` — privileged-EXEC block.
* `<MLTCMD>…</MLTCMD>` — multi-line CLI bundle.
* `! @ start-ignore-compliance` / `! @ end-ignore-compliance` — compliance
  exclusion markers.

See [../../rules/velocity/constraints.md](../../rules/velocity/constraints.md#9-catalyst-center-specific-syntax).

## 21. Variables cannot be re-typed in the UI after a bind is created

Switching Display Type from `Single Select` (bound) back to `Text` will usually
require deleting and recreating the variable. Plan binding intent early.

## 22. Only committed templates are deployable

A *saved* template is not enough — Catalyst Center requires **Commit** before
the template can be attached to a Network Profile. Uncommitted edits never
reach a device.

## 23. No way to call macros across templates

Macros are per-template. If a Regular Template wants to use a macro from
another template, copy the macro or refactor into a Composite Template with
shared sections.

## See also

* [../../docs/templates.md](../../docs/templates.md)
* [../../docs/velocity/basics.md](../../docs/velocity/basics.md)
* [../../docs/velocity/advanced.md](../../docs/velocity/advanced.md)
* [../../docs/system-variables.md](../../docs/system-variables.md)
* [../../rules/velocity/variable-types.md](../../rules/velocity/variable-types.md)
* [../../rules/velocity/constraints.md](../../rules/velocity/constraints.md)
* [./common-mistakes.md](./common-mistakes.md)
* [./invalid-syntax-patterns.md](./invalid-syntax-patterns.md)
