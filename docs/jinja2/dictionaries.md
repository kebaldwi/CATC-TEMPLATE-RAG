---
title: "Jinja2 — Dictionaries & Lists-of-Dictionaries"
engine: jinja2
kind: docs
topic: dictionaries
path: docs/jinja2/dictionaries.md
---

# Jinja2 Dictionaries and Lists-of-Dictionaries [![published](https://static.production.devnetcloud.com/codeexchange/assets/images/devnet-published.svg)](https://developer.cisco.com/codeexchange/github/repo/kebaldwi/DNAC-TEMPLATES)

This document is a worked example of one of the most useful Jinja2 patterns in
Catalyst Center: building a **list of dictionaries** ("list-of-dicts") to
describe per-site / per-VLAN data, then driving macros and CLI generation from
it. All examples are drawn from
[../../templates/jinja2/VLAN-Configuration.j2](../../examples/jinja2/VLAN-Configuration.j2)
and the related `DEFN-VlanInfo.j2` / `BUILD-VlanConfiguration.j2` modules.

For the general variable taxonomy, see
[../variables/Variables.md](../variables.md). For the rule book see
[../../../rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md)
and [../../../rules/jinja2/jinja2-constraints.md](../../rules/jinja2/constraints.md).

---

## 1. Why use list-of-dicts?

A single dictionary captures one record:

```j2
{% set mgmt_vlan = {'vlan':'5', 'name':'mgmtvlan'} %}
vlan {{ mgmt_vlan['vlan'] }}
 name {{ mgmt_vlan['name'] }}
```

A **list of dictionaries** captures many records of the same shape, so you can
iterate them with a single `{% for %}` loop:

```j2
{% set SiteAvlans = [
  {'vlan':'5',   'name':'mgmtvlan'},
  {'vlan':'10',  'name':'apvlan'},
  {'vlan':'20',  'name':'datavlan'},
  {'vlan':'30',  'name':'voicevlan'},
  {'vlan':'40',  'name':'guestvlan'},
  {'vlan':'999', 'name':'disabledvlan'}
] %}

{% for vlanpair in SiteAvlans %}
vlan {{ vlanpair['vlan'] }}
 name {{ vlanpair['name'] }}
{% endfor %}
```

Renders:

```text
vlan 5
 name mgmtvlan
vlan 10
 name apvlan
vlan 20
 name datavlan
vlan 30
 name voicevlan
vlan 40
 name guestvlan
vlan 999
 name disabledvlan
```

## 2. Dictionary syntax basics

```j2
{# Empty dict #}
{% set empty = {} %}

{# Single-record dict #}
{% set vlan = {'vlan':'10', 'name':'data'} %}

{# Access — both forms work #}
{{ vlan['vlan'] }}    {# bracket form  — preferred when key has spaces / hyphens #}
{{ vlan.vlan }}       {# attribute form — only valid for Python identifier keys #}

{# Add / overwrite a key — needs the do extension #}
{% do vlan.update({'ip':'10.10.10.1'}) %}

{# Test for a key #}
{% if 'ip' in vlan %} … {% endif %}

{# Iterate keys / values / items #}
{% for k, v in vlan.items() %}{{ k }}={{ v }}{% endfor %}
{% for k in vlan.keys() %} … {% endfor %}
{% for v in vlan.values() %} … {% endfor %}
```

Rules:

* Keys must be quoted string literals (or numeric) — `{vlan:'10'}` (no quotes
  on `vlan`) is **invalid**.
* Hyphenated keys (`'roast-beef':'bad'`) require bracket access:
  `map['roast-beef']`.
* Single quotes and double quotes are equivalent — pick one and be consistent.

## 3. Three list-of-dicts patterns from `VLAN-Configuration.j2`

### 3.1 Minimal VLAN database (vlan + name)

```j2
{% set SiteAvlans = [
  {'vlan':'5',   'name':'mgmtvlan'},
  {'vlan':'10',  'name':'apvlan'},
  {'vlan':'20',  'name':'datavlan'},
  {'vlan':'30',  'name':'voicevlan'},
  {'vlan':'40',  'name':'guestvlan'},
  {'vlan':'999', 'name':'disabledvlan'}
] %}
```

Consumed by:

```j2
{% macro configure_vlans(vlanpairs) %}
  {% for vlanpair in vlanpairs %}
    vlan {{ vlanpair['vlan'] }}
     name {{ vlanpair['name'] }}
  {% endfor %}
{% endmacro %}

{{ configure_vlans(SiteAvlans) }}
```

### 3.2 Extended VLAN database with L3 addressing

```j2
{% set SiteAvlansLibrary = [
  {'vlan':'5',   'name':'mgmtvlan',     'ip':'192.168.5.1',  'mask':'255.255.255.0', 'dhcp':'198.18.133.1'},
  {'vlan':'10',  'name':'apvlan',       'ip':'192.168.10.1', 'mask':'255.255.255.0', 'dhcp':'198.18.133.1'},
  {'vlan':'20',  'name':'datavlan',     'ip':'192.168.20.1', 'mask':'255.255.255.0', 'dhcp':'198.18.133.1'},
  {'vlan':'30',  'name':'voicevlan',    'ip':'192.168.30.1', 'mask':'255.255.255.0', 'dhcp':'198.18.133.1'},
  {'vlan':'40',  'name':'guestvlan',    'ip':'192.168.40.1', 'mask':'255.255.255.0', 'dhcp':'198.18.133.1'},
  {'vlan':'999', 'name':'disabledvlan', 'ip':'192.168.99.1', 'mask':'255.255.255.0', 'dhcp':'198.18.133.1'}
] %}
```

Consumed with branching on a key — note how `mgmtvlan` is treated specially:

```j2
{% macro configure_vlans_svi(vlanpairs) %}
  {% for vlanpair in vlanpairs %}
    vlan {{ vlanpair['vlan'] }}
     name {{ vlanpair['name'] }}
    {% if "mgmtvlan" in vlanpair['name'] %}
      pnp startup-vlan {{ vlanpair['vlan'] }}
    {% else %}
      interface vlan {{ vlanpair['vlan'] }}
       description {{ vlanpair['name'] }}
       ip address {{ vlanpair['ip'] }} {{ vlanpair['mask'] }}
       ip helper-address {{ vlanpair['dhcp'] }}
       ip ospf 1 area 0
       no shutdown
    {% endif %}
  {% endfor %}
{% endmacro %}

{{ configure_vlans_svi(SiteAvlansLibrary) }}
```

### 3.3 Per-port "deployment code" list (different shape, same idiom)

```j2
{% set Deployment_Codes = [
  {'port':'1/0/1', 'code':'S028'},
  {'port':'1/0/2', 'code':'S020'},
  {'port':'1/0/3', 'code':'S012'}
] %}

{% for entry in Deployment_Codes %}
interface GigabitEthernet{{ entry['port'] }}
 description {{ entry['code'] }}
{% endfor %}
```

## 4. Filtering / searching a list-of-dicts

Catalyst Center's Jinja2 does **not** ship a SQL-like filter, but a plain
`{% for %}` with `{% do …append %}` is the canonical pattern. This is taken
verbatim from `VLAN-Configuration.j2`:

```j2
{% set vlanArray = [] %}

{% macro search_vlans(vlanpairs) %}
  {% for vlanpair in vlanpairs %}
    {% if vlanpair['name'] == "apvlan" %}
      {% do vlanArray.append(vlanpair['vlan']) %}
    {% elif vlanpair['name'] == "datavlan" %}
      {% do vlanArray.append(vlanpair['vlan']) %}
    {% elif "voice" in vlanpair['name'] %}
      {% do vlanArray.append(vlanpair['vlan']) %}
    {% elif "guest" in vlanpair['name'] %}
      {% do vlanArray.append(vlanpair['vlan']) %}
    {% elif "disabled" in vlanpair['name'] %}
      {% do vlanArray.append(vlanpair['vlan']) %}
    {% endif %}
  {% endfor %}
{% endmacro %}

{{ search_vlans(SiteAvlans) }}

! vlanArray after search:
{{ vlanArray | join(',') }}
```

Two things to notice:

1. **`vlanArray` is declared at module scope** so the append survives outside
   the macro. A `{% set %}` *inside* a macro would not be visible outside
   it. See §4 of
   [../../../rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md).
2. The accumulator pattern (`{% set ns = [] %}` + `{% do ns.append(...) %}`)
   is the **only** way to build a list across iterations in Jinja2; you
   cannot reassign over a `{% set %}` inside the loop and have the value
   survive.

## 5. Using a list-of-dicts to build a `switchport trunk allowed vlan` line

```j2
{% set vlans = [
  {'vlan':'10',  'name':'apvlan'},
  {'vlan':'20',  'name':'datavlan'},
  {'vlan':'30',  'name':'voicevlan'}
] %}

interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan {{ vlans | map(attribute='vlan') | join(',') }}
```

The `map(attribute='vlan')` filter pulls one field from every record, and
`join(',')` flattens the result.

## 6. Composing per-site data

A common organisational trick: keep one list-of-dicts per site, then dispatch
at render time on a bind variable:

```j2
{% set SiteAvlans = [
  {'vlan':'5','name':'mgmtvlan'},  {'vlan':'10','name':'apvlan'},
  {'vlan':'20','name':'datavlan'}, {'vlan':'30','name':'voicevlan'},
  {'vlan':'40','name':'guestvlan'},{'vlan':'999','name':'disabledvlan'}
] %}

{% set SiteBvlans = [
  {'vlan':'15','name':'mgmtvlan'}, {'vlan':'110','name':'apvlan'},
  {'vlan':'120','name':'datavlan'},{'vlan':'130','name':'voicevlan'},
  {'vlan':'140','name':'guestvlan'},{'vlan':'999','name':'disabledvlan'}
] %}

{% if SiteCode == "A" %}
  {{ configure_vlans(SiteAvlans) }}
{% elif SiteCode == "B" %}
  {{ configure_vlans(SiteBvlans) }}
{% endif %}
```

## 7. Common mistakes specific to dictionaries

| Mistake                                       | Bad                                                  | Good                                                |
|-----------------------------------------------|------------------------------------------------------|-----------------------------------------------------|
| Unquoted key                                  | `{vlan:'10', name:'data'}`                           | `{'vlan':'10', 'name':'data'}`                      |
| Mixing JSON-style booleans                    | `{'active': True}` (Python `True` works but mixed-case fails) | `{'active': true}`                            |
| Accessing a missing key as attribute          | `{{ vlan.ip }}` returns empty silently               | `{{ vlan.ip if 'ip' in vlan else '0.0.0.0' }}` or `{{ vlan['ip'] | default('0.0.0.0') }}` |
| Mutating without `{% do %}`                   | `{{ vlan.update({'ip':'X'}) }}` (emits `None`)       | `{% do vlan.update({'ip':'X'}) %}`                  |
| Hyphenated key via attribute access           | `{{ entry.roast-beef }}` (parsed as subtraction)     | `{{ entry['roast-beef'] }}`                         |
| Forgetting accumulator scope                  | `{% set acc = [] %}` *inside* a `for`                | declare `{% set acc = [] %}` *before* the loop      |
| Treating dict iteration as ordered            | Assuming `keys()` returns in literal order (works on Py 3.7+ but don't rely on it for CLI ordering) | iterate the source list-of-dicts directly  |

## 8. Reference patterns at a glance

```j2
{# Print a single field #}
{{ entry['vlan'] }}

{# Sum a numeric field across a list-of-dicts #}
{{ vlans | map(attribute='vlan') | map('int') | sum }}

{# Filter to records where 'name' == 'datavlan' #}
{% set data = vlans | selectattr('name', 'equalto', 'datavlan') | list %}

{# Filter to records whose 'name' contains 'voice' #}
{% set voice = vlans | selectattr('name', 'search', 'voice') | list %}
{# (the 'search' test requires Ansible — in CC use the manual {% do append %} loop) #}
```

## See also

* [../variables/Variables.md](../variables.md) — variable tutorial.
* [../variables/SystemVariables.md](../system-variables.md) — `__device`, `__interface`.
* [Jinja2 basics](./basics.md) — Jinja2 fundamentals.
* [AdvancedJinja2](./advanced.md) — advanced patterns.
* [../../templates/jinja2/VLAN-Configuration.j2](../../examples/jinja2/VLAN-Configuration.j2) — the canonical example.
* [../../templates/jinja2/DEFN-VlanInfo.j2](../../examples/jinja2/DEFN-VlanInfo.j2) — definition-only form.
* [../../templates/jinja2/BUILD-VlanConfiguration.j2](../../examples/jinja2/BUILD-VlanConfiguration.j2) — macro-only form.
* [../../../rules/jinja2/variable-types.md](../../rules/jinja2/variable-types.md) — variable taxonomy and rules.
* [../../../rules/jinja2/jinja2-constraints.md](../../rules/jinja2/constraints.md) — syntax rule book.
