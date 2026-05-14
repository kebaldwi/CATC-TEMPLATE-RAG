---
title: "Jinja2 — Invalid Syntax Patterns"
engine: jinja2
kind: guardrails
topic: invalid-syntax-patterns
path: guardrails/jinja2/invalid-syntax-patterns.md
---

# Invalid Syntax Patterns — Jinja2 in Catalyst Center

Hard syntax errors that fail at template parse / save time. For behavioural
pitfalls that *parse* but misbehave, see
[common-mistakes.md](./common-mistakes.md). The authoritative rule book is
[../../rules/jinja2/jinja2-constraints.md](../../rules/jinja2/constraints.md).

Each item gives a **bad** form and a **good** form.

---

## 1. Mismatched / missing end tag

Every block-level directive must close.

```j2
{# bad #}
{% if x %}
  hostname FOO

{# good #}
{% if x %}
  hostname FOO
{% endif %}
```

Required closers: `{% endif %}`, `{% endfor %}`, `{% endmacro %}`,
`{% endblock %}`, `{% endset %}` (block form only), `{% endraw %}`,
`{% endcall %}`, `{% endfilter %}`, `{% endwith %}`.

## 2. Wrong closer for the block type

`{% endif %}` does not close a `for`. `{% endfor %}` does not close a macro.

```j2
{# bad #}
{% for v in Vlans %}
  vlan {{ v }}
{% endif %}

{# good #}
{% for v in Vlans %}
  vlan {{ v }}
{% endfor %}
```

## 3. `elseif` instead of `elif`

```j2
{# bad #}
{% if x == 1 %}…{% elseif x == 2 %}…{% endif %}

{# good #}
{% if x == 1 %}…{% elif x == 2 %}…{% endif %}
```

## 4. Velocity directives in a `.j2`

`#if`, `#foreach`, `#set`, `#macro`, `#parse`, `#include` are Velocity. Jinja2
will pass them through as literal text and they will end up in IOS config.

```j2
{# bad #}
#if($Switch == 1)
priority 10
#end

{# good #}
{% if Switch == 1 %}
priority 10
{% endif %}
```

## 5. `$` prefix on references

Jinja2 has no `$`. `$var` is rendered literally.

```j2
{# bad #}
hostname $Hostname

{# good #}
hostname {{ Hostname }}
```

## 6. `{{ }}` inside a `{% %}` block

Directives take bare expressions. Nested print delimiters are a parse error.

```j2
{# bad #}
{% if {{ Switch }} == 1 %}…{% endif %}
{% for i in {{ range(5) }} %}…{% endfor %}

{# good #}
{% if Switch == 1 %}…{% endif %}
{% for i in range(5) %}…{% endfor %}
```

## 7. `{% %}` inside a `{{ }}` block

A print block contains exactly one expression. You cannot embed a statement.

```j2
{# bad #}
{{ {% if x %}1{% else %}0{% endif %} }}

{# good — use a conditional expression #}
{{ 1 if x else 0 }}
```

## 8. Unmatched delimiter

A `{{` without `}}` (or `{%` without `%}`) is a parse error. Also true across
line breaks — there is no continuation character.

```j2
{# bad #}
hostname {{ Hostname

{# good #}
hostname {{ Hostname }}
```

## 9. Hyphens inside identifiers

A hyphen is always the subtraction operator. Identifiers may use only
letters, digits, and `_`.

```j2
{# bad — parsed as voice - vlan = 30 #}
{% set voice-vlan = 30 %}

{# good #}
{% set voice_vlan = 30 %}
```

## 10. Block `set` without `endset`

The block form **requires** a closer.

```j2
{# bad #}
{% set banner %}
  multi
  line

{# good #}
{% set banner %}
  multi
  line
{% endset %}
```

## 11. `{% raw %}` without `{% endraw %}`

```j2
{# bad #}
{% raw %}
banner login ^ {{ literal }} ^

{# good #}
{% raw %}
banner login ^ {{ literal }} ^
{% endraw %}
```

## 12. Macro called as a statement

Macros are expressions, not statements.

```j2
{# bad #}
{% access_interface(20) %}

{# good #}
{{ access_interface(20) }}
```

## 13. Positional argument after keyword argument

When calling a macro, all positional args must come first.

```j2
{# bad #}
{{ make_vlan(name='data', 20) }}

{# good #}
{{ make_vlan(20, name='data') }}
```

## 14. Default values not in trailing position

In a macro definition, parameters with defaults must come **after** parameters
without defaults.

```j2
{# bad #}
{% macro acl(name='X', number) %}…{% endmacro %}

{# good #}
{% macro acl(number, name='X') %}…{% endmacro %}
```

## 15. Filter applied with `.` instead of `|`

Filters use the pipe character. `.` is attribute access.

```j2
{# bad #}
{{ name.upper }}              {# attribute lookup — returns the method object #}
{{ vlans.length }}            {# attribute lookup on a list — undefined #}

{# good #}
{{ name | upper }}
{{ vlans | length }}
```

## 16. Function call form of a filter argument

`| split(",")` is correct; `split(",")` standalone is not a filter call.

```j2
{# bad #}
{{ split(",", ProductID) }}

{# good #}
{{ ProductID | split(",") }}
{# or method form #}
{{ ProductID.split(",") }}
```

## 17. Quoting variables as if they were strings

A bare name inside `{% ... %}` is the variable. Wrapping it in quotes makes
it a literal string.

```j2
{# bad — always true; compares strings #}
{% if "Switch" == "1" %}…{% endif %}

{# good #}
{% if Switch == 1 %}…{% endif %}
```

## 18. Comment syntax errors

Jinja2 comments are `{# ... #}` — not `<!-- -->`, not `//`, not `/* */`,
not `#`.

```j2
{# bad #}
// this is a comment
<!-- this is a comment -->
# this is a comment

{# good #}
{# this is a comment #}
```

## 19. `{# %}` / `{% #}` etc. — mixed delimiters

Each delimiter pair must match.

```j2
{# bad #}
{% set x = 1 #}
{# debug %}

{# good #}
{% set x = 1 %}
{# debug #}
```

## 20. `block` outside the parent/child inheritance model

`{% block name %}` is only meaningful in a template that is `{% extends %}`-ed
by another. A standalone template can declare blocks, but you cannot
*override* a block without `{% extends %}`.

```j2
{# bad — no extends, block override has no effect #}
{% block Vlans %}vlan 10{% endblock %}

{# good #}
{% extends "DayNTemplates/BUILD-MasterBuild.j2" %}
{% block Vlans %}vlan 10{% endblock %}
```

## 21. `extends` not on the first non-comment line

`{% extends %}` must be the **first** template tag (comments above it are
fine).

```j2
{# bad — extends after content is an error #}
hostname FOO
{% extends "Project/Base" %}

{# good #}
{% extends "Project/Base" %}
{% block body %}hostname FOO{% endblock %}
```

## 22. Multiple `extends` in one template

Only one `extends` is permitted.

```j2
{# bad #}
{% extends "A" %}
{% extends "B" %}

{# good — choose one base and chain via the parent #}
{% extends "A" %}
```

## 23. Comments inside an expression

`{# … #}` is a top-level comment. It cannot be embedded inside a `{{ … }}` or
`{% … %}` expression.

```j2
{# bad #}
{{ Hostname {# the device name #} }}

{# good #}
{# the device name #}
{{ Hostname }}
```

## 24. Wrong whitespace-trim placement

The `-` modifier goes **inside** the delimiter, adjacent to the `%` / `}`.

```j2
{# bad #}
{ %- set x = 1 -% }
{ %-set x = 1-% }
{%-set x =1-%}        {# actually fine — spaces are optional —, this one is a trap example only #}

{# good #}
{%- set x = 1 -%}
```

## 25. Treating `{% do %}` as if it returned a value

`{% do %}` is a *statement*; it never emits anything and cannot appear inside
`{{ ... }}`.

```j2
{# bad #}
{{ {% do list.append(1) %} }}

{# good — issue the statement on its own #}
{% do list.append(1) %}
```

## 26. Calling `loop` outside a `for`

`loop.index`, `loop.first`, `loop.cycle(...)` etc. exist only inside a
`{% for %}` body.

```j2
{# bad #}
{% if loop.first %}…{% endif %}        {# no enclosing for #}

{# good #}
{% for v in vlans %}
  {% if loop.first %}…{% endif %}
{% endfor %}
```

## See also

* [common-mistakes.md](./common-mistakes.md)
* [what-cc-cannot-do.md](./what-cc-cannot-do.md)
* [../../rules/jinja2/jinja2-constraints.md](../../rules/jinja2/constraints.md)
* [./invalid-syntax-patterns.md](./invalid-syntax-patterns.md) — Velocity-side equivalent.
