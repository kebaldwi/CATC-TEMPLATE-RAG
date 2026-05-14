---
title: "Jinja2 — Variable Types"
engine: jinja2
kind: rules
topic: variable-types
path: rules/jinja2/variable-types.md
---

# Variable Types (Jinja2 / Catalyst Center)

This document classifies every kind of variable a Jinja2 template author will
encounter inside Catalyst Center. It is a companion to
[../Variables.md](../../docs/variables.md) and [../SystemVariables.md](../../docs/system-variables.md).
For the hard syntax rules see [jinja2-constraints.md](constraints.md).

Upstream reference: <https://jinja.palletsprojects.com/en/3.0.x/templates/>.

> **Note on prefixing:** Unlike Velocity, Jinja2 references do **not** carry a
> `$` prefix. A reference is just its identifier (`Switch`, `__device`,
> `vlanArray`) — wrapped in `{{ ... }}` for output, used bare inside
> `{% ... %}` directive blocks.

## 1. Categories at a glance

| Category            | Origin                                                            | Available in PnP? | Available in DayN? | Typical reference form               |
|---------------------|-------------------------------------------------------------------|-------------------|--------------------|--------------------------------------|
| User-input          | Declared in template, populated by operator at provisioning time  | Yes               | Yes                | `{{ Hostname }}`, `{{ MgmtVlan }}`   |
| Bind variable       | Bound in Input Form to Inventory / Settings / Profile / Cloud     | **No**            | Yes                | `{{ ProductID }}`, `{{ native_bind }}` |
| System variable     | Built-in Catalyst Center object (`__device`, `__interface`, …)    | **No**            | Yes                | `{{ __device.platformId }}`          |
| Local (`{% set %}`) | Computed inside the template                                      | Yes               | Yes                | `{{ data_vlan_number }}`             |
| Literal             | Inline constant in the template                                   | Yes               | Yes                | `"text"`, `123`, `[1, 2]`, `{'k':'v'}` |
| Attribute reference | Dotted access on an object                                        | Same as parent    | Same as parent     | `{{ interface.portName }}`           |
| Method / filter     | Method call or Jinja2 filter pipeline                              | Same as parent    | Same as parent     | `{{ pid | split(",") }}`             |
| Indexed reference   | List / dict / array element                                       | Same as parent    | Same as parent     | `{{ StackPIDs[0] }}`, `{{ map['k'] }}` |

> **Rule of thumb:** PnP onboarding has no inventory record yet, so anything
> sourced from Catalyst Center's database (`bind` and `__*` system variables)
> is unavailable. Reserve those for DayN templates.

## 2. User-input variables

A user-input variable is any identifier that appears in the template and is
**not** assigned via `{% set %}` and **not** bound to a source. Catalyst Center
surfaces it in the Input Form so the operator (or API caller) supplies a value
at provisioning time.

```j2
hostname {{ Hostname }}
vlan {{ MgmtVlan }}
```

Form configuration (Template Editor → calculator icon):

* **Variable Definition** (the data type)
  * `String`
  * `Integer`
  * `IP Address`
  * `MAC Address`
* **Display Type** (how the operator enters it)
  * `Text` — free text, uses *Default Value* / *Instructional Text*
  * `Single Select` — required for **Bind to Source**
  * `Multi Select` — list of values
* Optional: *Tool Tip*, *Maximum Characters*, *Required*.

## 3. Bind variables

A bind variable is a user-input variable whose value is auto-populated from
Catalyst Center's database at provisioning time. Configure via the Input Form:

1. Set **Display Type** = `Single Select`.
2. Check **Bind to Source**.
3. Choose **Source**:
   * `Network Profile` — e.g., SSID name.
   * `Common Settings` — NTP, DNS, domain.
   * `Cloud Connect` — tunnel info.
   * `Inventory` — per-device attributes (PID, serial, hostname, location…).
4. Choose the **Entity** and **Attribute**.

Bind values arrive as **strings**. Cast with the `int` filter before
arithmetic:

```j2
{% set native_vlan = native_bind | int %}
{% set data_vlan   = native_vlan + 10 %}
```

Common bind-variable patterns from the `.j2` corpus:

```j2
{# Inventory → Platform Id (comma-delimited for a stack) #}
{% set StackPIDs       = ProductID | split(",") %}
{% set StackMemberCount = StackPIDs | length %}
```

## 4. System variables

System variables are objects maintained by Catalyst Center. They are listed
under **Template System Variables** in the Template Editor. The four sources
are the same as for bind variables; the names always begin with `__`.

| System variable     | Source           | Description                                              |
|---------------------|------------------|----------------------------------------------------------|
| `__device`          | Inventory        | Single device object (`hostname`, `platformId`, `snmpLocation`, …) |
| `__interface`       | Inventory        | List of interface objects on the target device           |
| `__networkSettings` | Common Settings  | DNS, NTP, syslog, AAA, etc.                              |
| `__credentials`     | Common Settings  | Device credentials object                                |
| `__networkProfile`  | Network Profile  | Profile-scoped attributes (e.g., SSIDs)                  |
| `__cloudConnect`    | Cloud Connect    | Cloud tunnel/connector data                              |

> The authoritative list is the **Template System Variables** dialog in your
> Catalyst Center version — click **Show More** to see the full set. Names and
> attributes do change between releases.

Iterating a system variable:

```j2
{% for interface in __interface %}
  {% if interface.portMode == "trunk" and interface.interfaceType == "Physical" %}
    interface {{ interface.portName }}
      {{ uplink_interface() }}
  {% endif %}
{% endfor %}
```

Reading a scalar attribute:

```j2
{% if __device.snmpLocation == "" %}
  snmp-server location {{ default_location }}
{% endif %}
```

**Reserved naming:** never declare `{% set __foo = ... %}`. The `__` prefix is
reserved for system variables.

## 5. Local variables (`{% set %}`)

Created and used inside the template. They are typed by what you assign:

```j2
{% set StringVariable  = "text" %}                        {# String   #}
{% set NumericVariable = 10 %}                            {# Integer  #}
{% set Flag            = true %}                          {# Boolean  #}
{% set L2vlans         = ["10", "18"] %}                  {# List     #}
{% set Vlans           = range(1, 6) %}                   {# Range → iterable #}
{% set Naming          = {'banana':'good', 'beef':'bad'} } {# Dict    #}
```

`{% set %}` has two forms:

```j2
{% set count = 5 %}                  {# inline — no endset #}

{% set banner_text %}
  This is a multi-line value
  assigned to a variable.
{% endset %}
```

The block form requires `{% endset %}`. See
[jinja2-constraints.md](constraints.md#set).

## 6. Literals

| Literal kind  | Example                              | Notes                                                                 |
|---------------|--------------------------------------|-----------------------------------------------------------------------|
| String        | `"hello"` / `'hello'`                | Both quote styles are equivalent in Jinja2 (no interpolation inside). |
| Integer       | `123`, `-4`                          | Used for math.                                                        |
| Float         | `1.5`                                | Coerced when mixed with ints.                                         |
| Boolean       | `true`, `false`                      | Lowercase only. (Python's `True/False` also works.)                   |
| None          | `none`                               | Equivalent to Python `None`; tested with `is none`.                   |
| List          | `["a", b, "c"]`                      | Access with `list[i]`. Heterogeneous types allowed.                   |
| Tuple         | `("a", "b")`                         | Immutable list.                                                       |
| Dict          | `{'k': 'v', 'k2': 'v2'}`             | Quote keys. Access with `d['k']` or `d.k`.                            |
| Range         | `range(1, 6)` / `range(n)`           | Use inside `for`; counts up by 1 unless `range(start,stop,step)`.     |

## 7. List-of-dictionaries patterns

The most common composite shape in the Catalyst Center `.j2` corpus is a
**list of dictionaries**: every element of the list is a dictionary with the
same keys. This lets one `{% for %}` loop drive an arbitrary number of
records with uniform shape. The canonical example is
[../../library/templates/jinja2/VLAN-Configuration.j2](../../examples/jinja2/VLAN-Configuration.j2).

### 7.1 Declaration

```j2
{# Minimal — vlan id + name #}
{% set SiteAvlans = [
  {'vlan':'5',   'name':'mgmtvlan'},
  {'vlan':'10',  'name':'apvlan'},
  {'vlan':'20',  'name':'datavlan'},
  {'vlan':'30',  'name':'voicevlan'},
  {'vlan':'40',  'name':'guestvlan'},
  {'vlan':'999', 'name':'disabledvlan'}
] %}

{# Extended — same shape with L3 addressing keys #}
{% set SiteAvlansLibrary = [
  {'vlan':'5',  'name':'mgmtvlan',  'ip':'192.168.5.1',  'mask':'255.255.255.0', 'dhcp':'198.18.133.1'},
  {'vlan':'10', 'name':'apvlan',    'ip':'192.168.10.1', 'mask':'255.255.255.0', 'dhcp':'198.18.133.1'},
  {'vlan':'20', 'name':'datavlan',  'ip':'192.168.20.1', 'mask':'255.255.255.0', 'dhcp':'198.18.133.1'}
] %}
```

Rules:

* Every key must be a quoted string literal. `{vlan:'10'}` (unquoted key)
  is invalid.
* Hyphenated or space-containing keys must use bracket access:
  `entry['roast-beef']`, not `entry.roast-beef` (the latter parses as
  subtraction).
* Every record should have the same key set — heterogeneous rows force
  defensive `'key' in entry` guards.

### 7.2 Iteration

```j2
{% for vlanpair in SiteAvlans %}
vlan {{ vlanpair['vlan'] }}
 name {{ vlanpair['name'] }}
{% endfor %}
```

Branching on a key value while iterating:

```j2
{% for vlanpair in SiteAvlansLibrary %}
  {% if "mgmtvlan" in vlanpair['name'] %}
pnp startup-vlan {{ vlanpair['vlan'] }}
  {% else %}
interface vlan {{ vlanpair['vlan'] }}
 ip address {{ vlanpair['ip'] }} {{ vlanpair['mask'] }}
 ip helper-address {{ vlanpair['dhcp'] }}
 no shutdown
  {% endif %}
{% endfor %}
```

### 7.3 Filtering / projecting

| Goal                                | Idiom                                                                 |
|-------------------------------------|------------------------------------------------------------------------|
| Pull one field from every record    | `{{ SiteAvlans \| map(attribute='vlan') \| join(',') }}`               |
| Sum a numeric field                 | `{{ vlans \| map(attribute='vlan') \| map('int') \| sum }}`            |
| Filter by exact match               | `{% set d = vlans \| selectattr('name','equalto','datavlan') \| list %}` |
| Filter by substring (manual)        | `{% for v in vlans %}{% if 'voice' in v['name'] %}…{% endif %}{% endfor %}` |
| Accumulate matches into a list      | `{% set acc = [] %}` + `{% do acc.append(v['vlan']) %}` inside the loop |

### 7.4 Accumulator scope rule

A `{% set acc = [] %}` declared **inside** a `{% for %}` or `{% macro %}` is
scoped to that block. To carry the accumulated list out, declare it **before**
the loop (or before the macro that owns the loop). This is the pattern used
by `search_vlans` in `VLAN-Configuration.j2`:

```j2
{% set vlanArray = [] %}            {# declared at module scope #}

{% macro search_vlans(vlanpairs) %}
  {% for vlanpair in vlanpairs %}
    {% if 'voice' in vlanpair['name'] %}
      {% do vlanArray.append(vlanpair['vlan']) %}
    {% endif %}
  {% endfor %}
{% endmacro %}

{{ search_vlans(SiteAvlans) }}
{# vlanArray is now populated and visible here #}
```

### 7.5 Worked dictionary reference

Detailed walk-through with multi-site dispatch, macro consumption, and the
trunk-allowed pattern lives in
[../../library/documentation/jinja2/Dictionaries.md](../../docs/jinja2/dictionaries.md).

## 8. Attribute, method, filter, and indexed references

```j2
{{ customer.Address }}            {# Attribute  — same as customer['Address']  #}
{{ ProductID.split(",") }}        {# Method     — direct method call           #}
{{ ProductID | split(",") }}      {# Filter     — Jinja2 pipe form (CC adds split) #}
{{ StackPIDs[Switch] }}           {# Indexed    — list / array element         #}
{{ map['banana'] }}               {# Dict key   — equivalent to map.banana     #}
```

Useful filters / methods exposed in Catalyst Center templates (verified
across the `.j2` corpus and Jinja2 3.x):

| Filter / method            | Purpose                                          |
|----------------------------|--------------------------------------------------|
| `\| length`                | Element count                                    |
| `\| int`                   | Cast string → integer                            |
| `\| split(",")`            | String → list (CC-provided)                      |
| `\| replace(old, new)`     | Substring substitution                           |
| `\| round('ceil')`         | Round-up to integer                              |
| `\| upper` / `\| lower`    | Case change                                      |
| `\| trim`                  | Strip surrounding whitespace                     |
| `\| default('x')`          | Fallback if undefined                            |
| `.append(x)` (with `do`)   | Append to a list — requires `{% do %}`           |
| `.split(",")` (method)     | Same effect as the `split` filter                |
| `range(n)` / `range(a,b)`  | Generate an iterable of integers                 |

## 9. Reference notations

| Notation                     | Example                            | Use                                        |
|------------------------------|------------------------------------|--------------------------------------------|
| Print expression             | `{{ Switch }}`                     | Emit a value into the rendered output      |
| Statement / control          | `{% if Switch %}`                  | Logic and assignment; emits nothing        |
| Comment                      | `{# note #}`                       | Removed at render                          |
| Whitespace-trim left         | `{%- if x %}` / `{{- x }}`         | Strip whitespace before the tag            |
| Whitespace-trim right        | `{% endif -%}` / `{{ x -}}`        | Strip whitespace after the tag             |
| Raw block                    | `{% raw %} ... {% endraw %}`       | Treat content as literal (no parsing)      |

## 10. Identifier rules

* Start with a letter or `_`.
* Then any combination of letters, digits, and `_`.
* **No hyphens.** A hyphen is always the subtraction operator.
* Case sensitive.
* Avoid the `__` prefix — reserved for system variables.
* Avoid Python / Jinja reserved words as identifiers: `and`, `or`, `not`,
  `in`, `is`, `if`, `else`, `elif`, `for`, `do`, `true`, `false`, `none`,
  `loop`.

## 11. Catalyst Center → J2 type cheat sheet

| CC UI "Variable Definition" | CC UI "Display Type"    | Jinja2 data type at render        | Bindable? |
|-----------------------------|-------------------------|-----------------------------------|-----------|
| String                      | Text                    | String                            | No        |
| String                      | Single Select           | String                            | Yes       |
| String                      | Multi Select            | List<String>                      | No        |
| Integer                     | Text                    | String → cast with `\| int`       | No        |
| Integer                     | Single Select           | String → cast                     | Yes       |
| IP Address                  | Text                    | String                            | No        |
| MAC Address                 | Text                    | String                            | No        |

> Bind-to-Source is supported **only** with the `Single Select` display type.

## See also

* [../../library/documentation/variables/Variables.md](../../docs/variables.md) — original tutorial.
* [../../library/documentation/variables/SystemVariables.md](../../docs/system-variables.md) — deep dive on `__*`.
* [../../library/documentation/jinja2/Jinja2.md](../../docs/jinja2/basics.md) — Jinja2 scripting tutorial.
* [../../library/documentation/jinja2/AdvancedJinja2.md](../../docs/jinja2/advanced.md) — advanced patterns.
* [../../library/documentation/jinja2/Dictionaries.md](../../docs/jinja2/dictionaries.md) — VLAN-style list-of-dicts patterns.
* [jinja2-constraints.md](constraints.md) — full syntax rules.
* [../../guardrails/jinja2/common-mistakes.md](../../guardrails/jinja2/common-mistakes.md)
* [../velocity/variable-types.md](../velocity/variable-types.md) — the Velocity-side equivalent.
