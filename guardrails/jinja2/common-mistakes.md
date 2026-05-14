---
title: "Jinja2 — Common Mistakes"
engine: jinja2
kind: guardrails
topic: common-mistakes
path: guardrails/jinja2/common-mistakes.md
---

# Common Mistakes — Jinja2 in Catalyst Center

Behavioural pitfalls that pass syntax check but produce broken or surprising
configuration at render time. Each item gives a **bad** form and a **good**
form. For pure syntax errors see [invalid-syntax-patterns.md](./invalid-syntax-patterns.md);
for platform limits see [what-cc-cannot-do.md](./what-cc-cannot-do.md).

---

## 1. Using `!` as a Jinja2 comment

`!` is forwarded verbatim into IOS. It is **not** a Jinja2 comment.

```j2
{# bad — your "comment" becomes IOS config #}
! this is a header

{# good — Jinja2 strips this at render time #}
{# this is a header #}
```

## 2. Forgetting to cast a bind variable

Bind variables arrive as **strings**. Arithmetic on strings silently
concatenates or errors at run time.

```j2
{# bad — "10" + 5 → TypeError or "105" depending on order #}
{% set offset = native_bind + 5 %}

{# good — cast first #}
{% set offset = (native_bind | int) + 5 %}
```

## 3. Mixing Velocity and Jinja2 syntax

CC accepts both engines but never in the same template. If you see `#if`,
`#foreach`, `$var`, `#set`, or `#macro` in a `.j2`, the file is wrong.

```j2
{# bad #}
#if($Switch == 1) priority 10 #end

{# good #}
{% if Switch == 1 %}priority 10{% endif %}
```

## 4. `elseif` instead of `elif`

Jinja2 spells it `elif`. `elseif` raises a template error.

```j2
{# bad #}
{% if x == 1 %}…{% elseif x == 2 %}…{% endif %}

{# good #}
{% if x == 1 %}…{% elif x == 2 %}…{% endif %}
```

## 5. String concatenation with `+`

`+` is arithmetic. On strings it works only as long as both operands are
strings; mix with an integer and you get a TypeError. Use `~`.

```j2
{# bad #}
{% set fqdn = host + "." + domain %}
{% set label = "switch-" + index %}      {# breaks: index is int #}

{# good #}
{% set fqdn  = host ~ "." ~ domain %}
{% set label = "switch-" ~ index %}
```

## 6. Off-by-one with `loop.index`

`loop.index` is **1-based**; `loop.index0` is **0-based**. List indices are
0-based. Pick the right one.

```j2
{# bad — StackPIDs[loop.index] skips index 0 and overruns the list #}
{% for s in range(StackMemberCount) %}
  switch {{ loop.index }} platform {{ StackPIDs[loop.index] }}
{% endfor %}

{# good #}
{% for s in range(StackMemberCount) %}
  switch {{ loop.index }} platform {{ StackPIDs[loop.index0] }}
{% endfor %}
```

## 7. `range(stop)` is exclusive of `stop`

`range(1, 5)` yields `1, 2, 3, 4` — not `5`. Common cause of missing the last
VLAN / switch / port.

```j2
{# bad — only covers 1..3 #}
{% for v in range(1, 4) %}vlan {{ v }}{% endfor %}

{# good #}
{% for v in range(1, 5) %}vlan {{ v }}{% endfor %}
```

## 8. `.append()` (or any mutator) without `{% do %}`

`{{ list.append(x) }}` mutates the list **and** emits `None` into the output.
Always wrap mutators in `{% do %}`.

```j2
{# bad — adds "None" into the config #}
{{ PortTotal.append(48) }}

{# good #}
{% do PortTotal.append(48) %}
```

## 9. Assuming `{% set %}` inside a loop leaks out

Variables set inside a `for` block are scoped to that iteration. Use a list
plus `{% do append %}`, or a `namespace`, to carry state across iterations.

```j2
{# bad — total is always 0 after the loop #}
{% set total = 0 %}
{% for p in PortTotal %}{% set total = total + p %}{% endfor %}
total = {{ total }}

{# good #}
{% set ns = namespace(total=0) %}
{% for p in PortTotal %}{% set ns.total = ns.total + p %}{% endfor %}
total = {{ ns.total }}
```

## 10. Calling a macro before it is defined

Macros are resolved top-down. Define before use, or `{% import %}` them from
another template.

```j2
{# bad #}
{{ access_interface(20) }}
{% macro access_interface(v) %}…{% endmacro %}

{# good #}
{% macro access_interface(v) %}…{% endmacro %}
{{ access_interface(20) }}
```

## 11. Rendering a macro as a statement

Macros are *expressions* — invoke them with `{{ ... }}`, not `{% ... %}`.

```j2
{# bad #}
{% access_interface(20) %}

{# good #}
{{ access_interface(20) }}
```

## 12. Iterating a string by accident

Strings are iterable in Jinja2 — `for c in name` yields characters. If you
expected a list, you got a single comma-delimited string.

```j2
{# bad — iterates characters #}
{% for pid in __device.platformId %}…{% endfor %}

{# good — split first #}
{% for pid in __device.platformId | split(",") %}…{% endfor %}
```

## 13. Comparing types instead of values

`"10"` (string from a bind/user-input variable) ≠ `10` (integer literal).

```j2
{# bad — never matches when MgmtVlan came from the form as a string #}
{% if MgmtVlan == 1 %}…{% endif %}

{# good #}
{% if (MgmtVlan | int) == 1 %}…{% endif %}
```

## 14. Forgetting whitespace control around `{% %}` blocks

Each Jinja2 tag on its own line leaves a blank line in the output. The
rendered config grows ugly fast.

```j2
{# bad #}
{% if MgmtVlan > 1 %}
vlan {{ MgmtVlan }}
{% endif %}

{# good — strips the surrounding newlines #}
{%- if MgmtVlan > 1 -%}
vlan {{ MgmtVlan }}
{%- endif -%}
```

## 15. Using `{{ }}` inside a `{% %}` block

`{% if {{ x }} %}` is a syntax error. Inside a directive, just write the
expression bare.

```j2
{# bad #}
{% if {{ Switch }} == 1 %}…{% endif %}

{# good #}
{% if Switch == 1 %}…{% endif %}
```

## 16. Literal braces inside banners / certs

Banner text containing `{{` or `}}` will be parsed by Jinja2. Wrap in
`{% raw %}` or use `<MLTCMD>` with `{{ '{{' }}` escape.

```j2
{# bad — Jinja2 tries to parse {{ EXAMPLE }} #}
banner login ^
Use {{ EXAMPLE }} format
^

{# good #}
{% raw %}
banner login ^
Use {{ EXAMPLE }} format
^
{% endraw %}
```

## 17. Forgetting `#MODE_ENABLE` for privileged-EXEC commands

Commands like `switch X priority Y`, `archive download-sw`, and `write erase`
must run from EXEC mode, not config mode.

```j2
{# bad #}
{% for s in range(StackMemberCount) %}
switch {{ loop.index }} priority 10
{% endfor %}

{# good #}
#MODE_ENABLE
{% for s in range(StackMemberCount) %}
switch {{ loop.index }} priority 10
{% endfor %}
#MODE_END_ENABLE
```

## 18. Using `__device` / `__interface` in a PnP template

PnP runs **before** the device exists in Inventory. System variables and
bind variables are unavailable. The template renders empty / undefined values.

```j2
{# bad — PnP template referencing inventory #}
hostname {{ __device.hostname }}

{# good — accept the hostname as user input on PnP, bind it on DayN #}
hostname {{ Hostname }}
```

## 19. Using `break` / `continue`

Catalyst Center does not enable the `loopcontrols` extension. Restructure the
loop with `{% if %}` guards instead.

```j2
{# bad #}
{% for v in vlans %}
  {% if v == 0 %}{% break %}{% endif %}
  vlan {{ v }}
{% endfor %}

{# good #}
{% for v in vlans %}
  {% if v != 0 %}
    vlan {{ v }}
  {% endif %}
{% endfor %}
```

## 20. Relying on `default()` to mask a real binding error

`{{ var | default('X') }}` hides typos in variable names (CC runs Jinja2 in
non-strict mode, so undefined silently renders to empty — and `default` only
fires for explicit `none`/undefined). Prefer naming discipline plus
`is defined` guards during template development.

```j2
{# risky — masks a missing form variable #}
hostname {{ HostNme | default('EDGE-01') }}

{# explicit and discoverable #}
{% if Hostname is defined and Hostname %}
hostname {{ Hostname }}
{% else %}
{# … render a guard / abort path … #}
{% endif %}
```

## 21. Calling `int` on a value that may be empty

`'' | int` returns `0`, which can pass an `if` check unintentionally and
generate `vlan 0` style errors.

```j2
{# bad — empty bind value silently becomes 0 #}
vlan {{ native_bind | int }}

{# good #}
{% if native_bind %}
vlan {{ native_bind | int }}
{% endif %}
```

## 22. Hyphens inside identifiers

`{% set voice-vlan = 30 %}` is parsed as `voice - vlan = 30`. Use
underscores.

```j2
{# bad #}
{% set voice-vlan = 30 %}

{# good #}
{% set voice_vlan = 30 %}
```

## 23. Forgetting `{% endset %}` on a block `set`

The inline form `{% set x = 1 %}` has no closer. The block form does.

```j2
{# bad #}
{% set banner %}
  multi
  line
{# missing endset #}

{# good #}
{% set banner %}
  multi
  line
{% endset %}
```

## 24. Treating `include` as a code import

`{% include %}` **renders** the included template at that point and splices
its output. It does **not** import macros for later use. Use `{% import %}`
(or `{% from … import … %}`) when you only want macros.

```j2
{# bad — included template's macros are NOT in scope #}
{% include "DayNTemplates/BUILD-InterfaceMacros.j2" %}
{{ access_interface(20) }}    {# may be undefined depending on template order #}

{# good — explicit macro import #}
{% import "DayNTemplates/BUILD-InterfaceMacros.j2" as m %}
{{ m.access_interface(20) }}
```

## 25. Composite templates in PnP

Composite templates (the `Access-DayN-Composite.yml` style) are **DayN
only**. Selecting one in a PnP workflow fails silently or returns an empty
config.

```yaml
# bad — composite assigned to a PnP onboarding flow
PnP-Onboarding: Access-DayN-Composite.yml

# good — assign a single regular template to PnP, composite to DayN
PnP-Onboarding: Titanium-L2-PnP-Jinja-Template.j2
DayN:           Access-DayN-Composite.yml
```

## See also

* [invalid-syntax-patterns.md](./invalid-syntax-patterns.md)
* [what-cc-cannot-do.md](./what-cc-cannot-do.md)
* [../../rules/jinja2/jinja2-constraints.md](../../rules/jinja2/constraints.md)
* [../../rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md)
* [./common-mistakes.md](./common-mistakes.md) — Velocity-side equivalent.
