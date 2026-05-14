---
title: "Jinja2 — Syntax & Semantic Constraints"
engine: jinja2
kind: rules
topic: constraints
path: rules/jinja2/constraints.md
---

# Jinja2 Constraints (Catalyst Center)

Hard syntactic and semantic constraints for authoring Jinja2 templates inside
Catalyst Center. Use this file as the authoritative rule book; narrative
material lives in [../Jinja2.md](../../docs/jinja2/basics.md) and
[../AdvancedJinja2.md](../../docs/jinja2/advanced.md), and variable taxonomy lives in
[variable-types.md](./variable-types.md).

Upstream reference: <https://jinja.palletsprojects.com/en/3.0.x/templates/>.
Catalyst Center deviations from upstream are flagged inline and summarised in
§13.

---

## 1. Delimiters

| Delimiter         | Purpose                                         | Renders output? |
|-------------------|-------------------------------------------------|-----------------|
| `{{ ... }}`       | Print an expression                             | Yes             |
| `{% ... %}`       | Statement / control directive                   | No              |
| `{# ... #}`       | Comment                                         | No              |
| `{%- ... -%}`     | Statement + whitespace trim on either side      | No              |
| `{{- ... -}}`     | Print + whitespace trim on either side          | Yes             |
| `{% raw %}…{% endraw %}` | Literal block (no parsing)               | Yes (literal)   |

```j2
{# Comments removed at render time #}
{% set hostname = "EDGE-01" %}
hostname {{ hostname }}
```

`!` is **not** a Jinja2 comment — it is forwarded verbatim to IOS.

## 2. References

### 2.1 Notation forms

Unlike Velocity, Jinja2 has no `$` prefix. The same identifier is referenced
two ways depending on context:

```j2
{{ Switch }}             {# emit value into output #}
{% if Switch == 1 %}     {# bare name inside a directive — no braces #}
```

### 2.2 Identifier rules <a id="identifiers"></a>

* First character: letter (a–z, A–Z) or `_`.
* Remaining characters: letters, digits, `_`.
* **No hyphens.** `voice-offset` is parsed as `voice - offset`.
* Case sensitive (`Foo` ≠ `foo`).
* Do not create your own `__*` names — reserved for Catalyst Center system
  variables.
* Avoid reserved tokens: `and`, `or`, `not`, `in`, `is`, `if`, `else`, `elif`,
  `for`, `do`, `true`, `false`, `none`, `loop`.

### 2.3 Attribute lookup

`obj.attr` tries (in order):

1. `obj.attr`        (Python attribute)
2. `obj['attr']`     (Python `__getitem__`)
3. Returns *undefined* (renders as empty string in non-strict mode).

### 2.4 Indexed access

```j2
{{ list[i] }}                {# same as list.__getitem__(i) #}
{{ map['banana'] }}          {# same as map.__getitem__('banana') #}
{{ map.banana }}             {# attribute-style dict access — works in Jinja2 #}
```

## 3. Quoting and interpolation

Both quote styles are **equivalent** in Jinja2. There is **no string
interpolation** like Velocity's `"$ref"` — use concatenation or a format
filter.

```j2
{% set name = "Ben" %}

{# Concatenation with ~ #}
{% set a = "Hello " ~ name %}                {# "Hello Ben" #}

{# String formatting #}
{% set b = "Hello %s" | format(name) %}      {# "Hello Ben" #}

{# Inside print blocks — just emit expressions #}
description "{{ site_code }} - {{ closet }}"
```

## 4. String concatenation

Jinja2 has a dedicated concat operator `~`. Do **not** use `+` on strings:
that's addition and may TypeError when mixed with numbers.

```j2
{# Good #}
{% set fqdn = host ~ "." ~ domain %}

{# Bad — '+' on strings is arithmetic in Jinja2 #}
{% set fqdn = host + "." + domain %}
```

## 5. Arithmetic

```j2
{% set value = foo + 1 %}     {# addition #}
{% set value = foo - 1 %}     {# subtraction #}
{% set value = foo * bar %}   {# multiplication #}
{% set value = foo / bar %}   {# true division — returns float #}
{% set value = foo // bar %}  {# floor division — returns integer #}
{% set value = foo % bar %}   {# modulus #}
{% set value = foo ** bar %}  {# exponent #}
```

Bind variables arrive as **strings**; cast first with `| int`:

```j2
{% set native_vlan = native_bind | int %}
```

## 6. Truthiness (what `if` treats as false)

`{% if x %}` is false when `x` is any of:

* `undefined` (missing) — note this **never** raises in Catalyst Center
* `none`
* the boolean `false`
* the empty string `""`
* an empty list / dict / tuple
* the number `0`

Everything else is true. Catalyst Center runs Jinja2 in **non-strict mode**;
an undefined reference renders as an empty string, not an exception. Guard
with `is defined`, `is none`, or the `default()` filter.

## 7. Operators

| Category   | Symbolic                | Catalyst Center extras | Notes                                       |
|------------|-------------------------|------------------------|---------------------------------------------|
| Equal      | `==`                    |                        |                                             |
| Not equal  | `!=`                    |                        |                                             |
| Greater    | `>`                     |                        |                                             |
| Less       | `<`                     |                        |                                             |
| ≥          | `>=`                    |                        |                                             |
| ≤          | `<=`                    |                        |                                             |
| AND        | `and`                   | `&&`                   | CC accepts JS-style; standard Jinja2 does not. |
| OR         | `or`                    | `\|\|`                 | CC accepts JS-style.                        |
| NOT        | `not`                   | `!`                    | Prefer the word form for portability.       |
| Membership | `in` / `not in`         |                        | `x in list`, `'a' in 'abc'`                 |
| Identity   | `is` / `is not`         |                        | `x is none`, `x is defined`                 |
| Concat     | `~`                     |                        | Strings only.                               |
| Pipe       | `\|`                    |                        | Filter application.                         |

> Recommendation: use the word forms (`and`, `or`, `not`) so the same template
> renders identically in upstream Jinja2.

## 8. Directives (tags)

### 8.1 `{% set %}` <a id="set"></a>

Two forms:

```j2
{% set count = 5 %}          {# inline — no endset #}

{% set banner %}             {# block form — must close with endset #}
  multi
  line
{% endset %}
```

Scope rules:

* A `{% set %}` inside a `{% for %}` does not leak out of the loop (Python's
  scoping rules apply).
* For cross-iteration accumulation, use a list with `{% do list.append(...) %}`
  or namespace objects (`{% set ns = namespace(total=0) %}` then `ns.total = …`).

### 8.2 `{% if %}` / `{% elif %}` / `{% else %}` / `{% endif %}`

```j2
{% if hostname.contains("edge") %}
  ! edge branch
{% elif hostname.contains("border") %}
  ! border branch
{% else %}
  ! default branch
{% endif %}
```

Every `if` block must close with `{% endif %}`. `elif` (not `elseif`).

### 8.3 `{% for %}` / `{% endfor %}`

```j2
{% for vlan in Vlans %}
  interface vlan {{ vlan }}
{% else %}
  ! no vlans defined
{% endfor %}
```

`else` runs only when the sequence is empty. Inside the loop:

| Token                | Meaning                              |
|----------------------|--------------------------------------|
| `loop.index`         | 1-based index                        |
| `loop.index0`        | 0-based index                        |
| `loop.revindex`      | 1-based countdown from end           |
| `loop.revindex0`     | 0-based countdown from end           |
| `loop.first`         | true on first iteration              |
| `loop.last`          | true on last iteration               |
| `loop.length`        | total elements                       |
| `loop.previtem`      | previous item (undefined first iter) |
| `loop.nextitem`      | next item (undefined last iter)      |
| `loop.cycle('a','b')` | cycle through values per iteration  |
| `loop.depth` / `loop.depth0` | recursion depth in recursive loops |

Recursive loops need the `recursive` keyword:

```j2
{% for item in tree recursive %}
  {{ item.name }}
  {% if item.children %}{{ loop(item.children) }}{% endif %}
{% endfor %}
```

### 8.4 `{% macro %}` / `{% endmacro %}`

```j2
{% macro access_interface(vlan_number) %}
  switchport mode access
  switchport access vlan {{ vlan_number }}
  spanning-tree portfast
{% endmacro %}

interface GigabitEthernet1/0/1
  {{ access_interface(20) }}
```

Constraints:

* Define before use **within the same template** (or import / include from a
  template that defines them — see §8.5).
* Arguments are positional or keyword; defaults are right-to-left.
* Macros must `endmacro`.
* Inside a macro, `caller()` invokes a body passed via `{% call %}`.
* Use `{{ macro_name(args) }}` to render (note the **double braces** — macros
  are *expressions*, not statements).

Body-passing pattern:

```j2
{% macro wrap() %}
  before
  {{ caller() }}
  after
{% endmacro %}

{% call wrap() %}
  inner CLI
{% endcall %}
```

### 8.5 `{% include %}` and `{% extends %}`  <a id="include-extends"></a>

Catalyst Center **supports** these (unlike Velocity's `#include` / `#parse`).

* `{% include "Project/Template" %}` splices another template's rendered
  output at this point. Project/Template path is relative to the Template
  Editor project tree.
* `{% extends "Project/Base" %}` declares inheritance. Define `{% block name %}`
  in the parent and override with the same name in the child.
* `{% import "Project/Macros" as m %}` imports macros without rendering — call
  as `{{ m.macro_name(...) }}`.

```j2
{# Include — embeds another template's output #}
{% include "DayNTemplates/DEFN-VlanInfo.j2" %}

{# Extend — inherit blocks from a parent #}
{% extends "DayNTemplates/BUILD-MasterBuild.j2" %}
{% block SiteVlans %}
  vlan 10
   name override
{% endblock %}
```

### 8.6 `{% do %}`

CC enables the Jinja2 `do` extension. Use it for in-place mutation that does
not render output:

```j2
{% set ports = [] %}
{% for pid in StackPIDs %}
  {% do ports.append(pid) %}
{% endfor %}
```

Without `do`, calling a mutating method inside `{{ ... }}` will render the
method's return value (often `None`) into the configuration.

### 8.7 `{% raw %}` / `{% endraw %}`

Treats the body as literal text. Use it when CLI / banner content contains
`{{`, `}}`, `{%`, or `%}` that must not be parsed.

### 8.8 Other directives

| Directive       | Purpose                                       | CC support       |
|-----------------|-----------------------------------------------|------------------|
| `{% block %}`   | Named region overridable by child template    | Yes              |
| `{% import %}`  | Import macros from another template           | Yes              |
| `{% from … import … %}` | Selective import                       | Yes              |
| `{% filter %}`  | Apply a filter to a block                     | Yes              |
| `{% with %}`    | Local scope for variables                     | Yes              |
| `{% break %}`   | Exit the innermost loop                       | Requires loopcontrols extension — assume **not** available in CC. |
| `{% continue %}`| Skip to next iteration                        | Same caveat as `break`. |

## 9. Filters

Filter syntax: `{{ value | filter(arg) | other_filter }}`.

Commonly used in the CC `.j2` corpus:

```j2
{{ ProductID | split(",") | length }}
{{ native_bind | int + 10 }}
{{ name | upper }}
{{ snmpLocation | default('UNKNOWN') }}
{{ vlans | join(",") }}
{{ count | round('ceil') }}
{{ value | replace("old", "new") }}
{{ "Name: %s" | format(host) }}
```

`split` is a Catalyst Center-provided filter (not in stock Jinja2). For
portability you can also call `.split(",")` as a method.

## 10. Catalyst Center–specific syntax

These markers are **carried over from the Velocity layer** and work identically
in CC Jinja2 templates. They are **not** part of upstream Jinja2.

### 10.1 `#MODE_ENABLE` / `#MODE_END_ENABLE`

Wraps blocks that require privileged-EXEC execution:

```j2
#MODE_ENABLE
#MODE_END_ENABLE
#MODE_ENABLE
{% for s in range(0, StackMemberCount) %}
  {% if loop.index == 1 %}
    switch {{ loop.index }} priority 10
  {% elif loop.index == 2 %}
    switch {{ loop.index }} priority 9
  {% else %}
    switch {{ loop.index }} priority 8
  {% endif %}
{% endfor %}
#MODE_END_ENABLE
```

Note: these markers begin with `#` and are emitted literally — they are not
Jinja2 tags.

### 10.2 `<MLTCMD>...</MLTCMD>`

Multi-line CLI bundle (banners, key chains, certs):

```j2
<MLTCMD>banner login ^
  Session On {{ hostname }} Is Monitored!!!
^</MLTCMD>
```

### 10.3 Compliance ignore markers

```j2
interface GigabitEthernet1/0/1
 ! @ start-ignore-compliance
 no switchport
 ! @ end-ignore-compliance
 switchport mode trunk
```

## 11. Whitespace control

Jinja2 keeps newlines around tags by default, which often leaves blank lines
in the rendered CLI. Use the `-` modifier to strip them:

```j2
{%- if MgmtVlan > 1 -%}
vlan {{ MgmtVlan }}
{%- endif -%}
```

| Form          | Effect                          |
|---------------|---------------------------------|
| `{%- ... %}`  | Strip whitespace before the tag |
| `{% ... -%}`  | Strip whitespace after the tag  |
| `{%- ... -%}` | Both                            |
| `{{- ... }}`  | Strip before a print            |
| `{{ ... -}}`  | Strip after a print             |

Catalyst Center documents that Jinja2 whitespace artefacts in the simulation
output do **not** affect provisioning, but they hurt readability.

## 12. Escaping

| Source            | Renders as        | Notes                                            |
|-------------------|-------------------|--------------------------------------------------|
| `{{ '{{' }}`      | `{{`              | Print the literal delimiter.                     |
| `{% raw %}…{% endraw %}` | literal    | Preferred for large blocks containing delimiters.|
| `{{ value | e }}` | HTML-escaped      | Rarely useful for IOS CLI but available.         |

## 13. Catalyst Center vs upstream Jinja2 — quick deltas

| Feature                      | Upstream Jinja2 3.x         | Catalyst Center                          |
|------------------------------|-----------------------------|------------------------------------------|
| `&&` / `\|\|` / `!` operators | Not supported              | Accepted (alongside `and`/`or`/`not`).   |
| `split` filter               | Not a builtin               | Provided.                                |
| `.contains(...)` method      | Not standard on strings     | Accepted (alongside `in`).               |
| `.replaceAll(...)` method    | Not standard                | Accepted (use `replace` filter for portability). |
| `do` extension               | Optional, off by default    | Enabled.                                 |
| `loopcontrols` extension     | Optional                    | Assume **not** enabled (`break`/`continue` unavailable). |
| `include` / `extends`        | Available                   | Available; paths resolve into the Template Editor project tree. |
| Strict undefined mode        | Configurable                | Non-strict — undefined renders empty.    |
| `__device`, `__interface`    | Not standard                | Provided system objects (DayN only).     |
| `#MODE_ENABLE`, `<MLTCMD>`, `! @ start-ignore-compliance` | N/A | CC-specific markers. |

## 14. Conventions (Catalyst Center `.j2` corpus)

These idioms are present across multiple templates in
[examples/jinja2/](../../examples/jinja2/) and should be
considered canonical for this repo:

1. **Stack discovery from a system / bind variable**
   ```j2
   {% set StackPIDs        = __device.platformId | split(",") %}
   {% set StackMemberCount = StackPIDs | length %}
   ```
2. **Per-switch port count via `do`**
   ```j2
   {% set PortTotal = [] %}
   {% for Switch in range(StackMemberCount) %}
     {% set Model = StackPIDs[Switch] %}
     {% if '48' in Model %}
       {% do PortTotal.append(48) %}
     {% elif '24' in Model %}
       {% do PortTotal.append(24) %}
     {% endif %}
   {% endfor %}
   ```
3. **0-based array index, 1-based switch number**
   ```j2
   {% for Switch in range(StackMemberCount) %}
     {% set SwiNum = Switch + 1 %}
     interface range gi{{ SwiNum }}/0/1 - {{ PortTotal[Switch] }}
       {{ access_interface() }}
   {% endfor %}
   ```
4. **Composable Day-N master template using `{% include %}`**
   ```j2
   {% include "DayNTemplates/DEFN-VlanInfo.j2" %}
   {% include "DayNTemplates/BUILD-InterfaceMacros.j2" %}
   ```

## See also

* [../Jinja2.md](../../docs/jinja2/basics.md)
* [../AdvancedJinja2.md](../../docs/jinja2/advanced.md)
* [variable-types.md](./variable-types.md)
* [../velocity-constraints.md](../velocity/constraints.md) — Velocity-side equivalent.
* [../../guardrails/jinja2/common-mistakes.md](../../guardrails/jinja2/common-mistakes.md)
* [../../guardrails/jinja2/invalid-syntax-patterns.md](../../guardrails/jinja2/invalid-syntax-patterns.md)
* [../../guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md)
