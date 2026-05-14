---
title: "Jinja2 — Catalyst Center Jinja2 vs. Ansible Jinja2"
engine: jinja2
kind: rules
topic: cc-vs-ansible
path: rules/jinja2/cc-vs-ansible.md
---

# Catalyst Center Jinja2 vs. Ansible Jinja2

A side-by-side reference for engineers who move templates between Catalyst
Center's Template Editor and Ansible playbooks / roles. Both engines descend
from **Jinja2 3.x** (Pallets), but each host environment layers different
extensions, filters, globals, and execution semantics on top.

Sources:

* Catalyst Center Template Editor docs (Apache Velocity & Jinja2 use cases):
  <https://explore.cisco.com/dnac-use-cases/apache-velocity>
* Ansible templating reference:
  <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html>
* Jinja2 upstream: <https://jinja.palletsprojects.com/en/3.0.x/templates/>

This file lives alongside [Jinja2 basics](../../docs/jinja2/basics.md) and
[AdvancedJinja2](../../docs/jinja2/advanced.md); for the CC-specific rule book see
[Jinja2 constraints](./constraints.md).

---

## 1. TL;DR

| Concern                       | Catalyst Center Jinja2                                          | Ansible Jinja2                                                  |
|-------------------------------|-----------------------------------------------------------------|-----------------------------------------------------------------|
| Engine version                | Jinja2 3.x, embedded in Catalyst Center provisioner             | Jinja2 3.x, embedded in Ansible (Python process on controller)  |
| Render time                   | At deploy / preview inside Catalyst Center                      | At task time on the Ansible controller                          |
| Render output                 | Device CLI (or YAML for Composite descriptors)                  | Any text — files, configs, inline strings                       |
| Variable sources              | User-input, Bind, System (`__*`)                                | Inventory, vars, group_vars/host_vars, facts, registered output |
| Cross-template composition    | `include` / `extends` / `import` against the Editor project tree | `include` / `import` against the role / playbook search path    |
| Strict-undefined              | **Disabled** — undefined renders empty                          | **Enabled by default** (`StrictUndefined`) — fails the task     |
| Extra filters                 | `split`, plus the Pallets builtins                              | Hundreds (`to_yaml`, `ipaddr`, `regex_replace`, `combine`, …)   |
| `do` extension                | Enabled                                                         | Enabled                                                         |
| `loopcontrols` (`break`/`continue`) | **Disabled**                                              | Disabled by default; some collections enable it                 |
| CC-only markers (`#MODE_ENABLE`, `<MLTCMD>`, `! @ start-ignore-compliance`) | **Yes — required for some CLI**         | **N/A** — pure text output, no compliance/EXEC concept          |
| JS-style operators (`&&`, `||`, `!`) | **Accepted** alongside `and`/`or`/`not`                  | **Not** accepted — Python word-form only                        |
| `$var` references             | **No** — Velocity-only                                          | **No** — same                                                   |
| Comment syntax                | `{# ... #}`                                                     | `{# ... #}`                                                     |

The high-order takeaway: **a `.j2` you wrote for Catalyst Center will not
render cleanly under Ansible if you used `&&`, `||`, `!`, `split` as a
filter, `__device`/`__interface`, `#MODE_ENABLE`, or `<MLTCMD>`. A `.j2`
written for Ansible will not render cleanly inside Catalyst Center if you
used `to_yaml`, `ipaddr`, `regex_replace`, `hostvars`, `inventory_hostname`,
or relied on facts.**

---

## 2. Engine identity and version

Both products embed Pallets' Jinja2 3.x. The shared core (`{{ }}`, `{% %}`,
`{# #}`, `if`/`for`/`macro`/`set`/`include`/`extends`/`block`/`raw`/`do`,
the standard filter set such as `length`, `upper`, `lower`, `trim`,
`replace`, `default`, `join`, `int`, `round`, `format`, `tojson`,
`indent`, and the `loop` object) is **identical**.

Differences arise from:

1. **Which extensions are enabled** (do, loopcontrols, autoescape).
2. **Which filters / tests / globals are registered** at environment
   construction time.
3. **What variables get injected** into the rendering namespace.
4. **What strict-undefined mode is set to**.
5. **What post-render processing happens** to the resulting text.

---

## 3. Variable namespace

### 3.1 Catalyst Center

| Source             | Where it comes from                                                                                                | Example                                  |
|--------------------|--------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| **User-input**     | Declared in the template body, populated from the Input Form at provisioning                                       | `{{ Hostname }}`                         |
| **Bind variable**  | User-input variable bound to Inventory / Common Settings / Network Profile / Cloud Connect; arrives as a *string*  | `{{ native_bind }}`                      |
| **System variable**| Built-in CC objects, names always start with `__`                                                                  | `{{ __device.platformId }}`              |
| **Local**          | Computed inside the template via `{% set %}`                                                                       | `{{ data_vlan_number }}`                 |

Bind / system variables are **not** available in PnP templates (the device
is not yet in Inventory). See
[jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md).

### 3.2 Ansible

| Source              | Where it comes from                                                                | Example                                            |
|---------------------|------------------------------------------------------------------------------------|----------------------------------------------------|
| `vars:` / `set_fact`| Declared in the playbook, role, or task                                            | `{{ ntp_server }}`                                 |
| `group_vars` / `host_vars` | YAML files alongside the inventory                                          | `{{ vlan_data }}`                                  |
| Inventory variables | Inventory file / plugin                                                            | `{{ ansible_host }}`                               |
| **Facts**           | Gathered from the target host (`ansible_facts.*`, or top-level `ansible_*`)        | `{{ ansible_facts['default_ipv4']['address'] }}`   |
| Registered output   | `register:` on a previous task                                                     | `{{ show_version.stdout }}`                        |
| Magic variables     | `inventory_hostname`, `hostvars`, `groups`, `play_hosts`, `ansible_play_batch`, …  | `{{ hostvars['leaf-1'].mgmt_ip }}`                 |
| Lookup plugins      | `lookup('file', '…')`, `lookup('env', 'PATH')`, `lookup('vault', '…')`, …          | `{{ lookup('file', 'banner.txt') }}`               |

> Catalyst Center has **no** equivalent of `hostvars`, `groups`,
> `inventory_hostname`, facts, lookups, or registered output. Cross-device
> information flow is not a template-time concern in CC; it is handled by
> the provisioner itself.

### 3.3 Reserved names

* CC reserves the `__` prefix for system variables — never `{% set __foo … %}`.
* Ansible reserves any name starting with `ansible_` — colliding with a fact
  causes confusion.
* Both reserve `loop` inside `for` bodies.

---

## 4. Operators — word form vs. symbolic

| Operator   | Upstream Jinja2 / Ansible | Catalyst Center        |
|------------|---------------------------|------------------------|
| AND        | `and`                     | `and`, **also `&&`**   |
| OR         | `or`                      | `or`,  **also `||`**   |
| NOT        | `not x`                   | `not x`, **also `!x`** |
| Membership | `in`, `not in`            | `in`, `not in`         |
| Identity   | `is`, `is not`            | `is`, `is not`         |
| Concat     | `~`                       | `~`                    |
| Filter pipe| `|`                       | `|`                    |

Examples from the CC `.j2` corpus that **will not render under Ansible**:

```j2
{# CC-only — accepted #}
{% if MgmtVlan > 1 || MgmtVlan == 0 %}

{# CC-only — accepted #}
{% if interface.portMode == "trunk" && interface.interfaceType == "Physical" %}
```

Portable equivalents that work in both:

```j2
{% if MgmtVlan > 1 or MgmtVlan == 0 %}
{% if interface.portMode == "trunk" and interface.interfaceType == "Physical" %}
```

> **Recommendation:** always author with the word-form (`and`/`or`/`not`) so
> the same template body renders identically under both engines.

---

## 5. Filters

### 5.1 Available in **both** (Pallets builtins)

`abs`, `attr`, `batch`, `capitalize`, `center`, `default`, `dictsort`,
`escape` / `e`, `filesizeformat`, `first`, `float`, `forceescape`, `format`,
`groupby`, `indent`, `int`, `join`, `last`, `length`, `list`, `lower`,
`map`, `max`, `min`, `pprint`, `random`, `reject`, `rejectattr`, `replace`,
`reverse`, `round`, `safe`, `select`, `selectattr`, `slice`, `sort`,
`string`, `striptags`, `sum`, `title`, `tojson`, `trim`, `truncate`,
`unique`, `upper`, `urlencode`, `urlize`, `wordcount`, `wordwrap`, `xmlattr`.

### 5.2 Catalyst Center adds

| Filter      | Purpose                                  | Notes                                   |
|-------------|------------------------------------------|-----------------------------------------|
| `split(sep)`| String → list                            | Same effect as Python's `str.split`; also exposed as the `.split()` *method*. **Not** a stock Jinja2 filter. |

Other "filter-like" calls in the CC corpus (`.contains(...)`,
`.replaceAll(...)`) are **method invocations**, not filters; they work
because CC exposes Python/JVM string methods. The portable equivalents are
`in` and the `replace` filter, respectively.

### 5.3 Ansible adds (representative — hundreds total)

Categorised highlights — none of these are available in Catalyst Center:

* **YAML / JSON:** `to_yaml`, `to_nice_yaml`, `from_yaml`, `to_json`,
  `to_nice_json`, `from_json`.
* **Regex:** `regex_search`, `regex_findall`, `regex_replace`,
  `regex_escape`.
* **Strings / paths:** `quote`, `b64encode`, `b64decode`, `hash`,
  `password_hash`, `basename`, `dirname`, `splitext`, `realpath`,
  `relpath`, `expanduser`.
* **Dict / list ops:** `combine`, `dict2items`, `items2dict`, `zip`,
  `zip_longest`, `subelements`, `flatten`, `union`, `intersect`,
  `difference`, `symmetric_difference`.
* **Networking (`ansible.utils` / `ansible.netcommon`):** `ipaddr`,
  `ipv4`, `ipv6`, `ipsubnet`, `ipmath`, `cidr_merge`, `next_nth_usable`,
  `network_in_network`, `hwaddr`, `macaddr`.
* **Random / fileglob:** `random`, `shuffle`, `fileglob`, `urlsplit`.
* **Failure / safety:** `mandatory` (raises if undefined),
  `default(omit)` (Ansible-specific sentinel).

```jinja2
{# Pure Ansible — none of these work in Catalyst Center #}
{{ ansible_facts.ansible_default_ipv4.address | ipaddr('network') }}
{{ vlans | to_nice_yaml }}
{{ raw_config | regex_replace('^\s+', '', multiline=True) }}
{{ groups['leaves'] | map(attribute='ansible_host') | list }}
```

---

## 6. Tests

The Pallets test set (`is defined`, `is none`, `is number`, `is string`,
`is mapping`, `is sequence`, `is iterable`, `is sameas`, `is in`,
`is divisibleby`, `is odd`, `is even`, `is upper`, `is lower`, …) is
present in both.

Ansible adds tests via collections:

* `is search`, `is match`, `is regex` (regex tests)
* `is version`, `is version_compare` (semver-aware comparisons)
* `is subset`, `is superset`
* `is contains`
* `is truthy`, `is falsy` (Ansible's interpretation of truthy/falsy)

Catalyst Center does **not** add custom tests.

---

## 7. Globals and functions

| Global         | Upstream | Catalyst Center | Ansible                                       |
|----------------|----------|-----------------|-----------------------------------------------|
| `range`        | Yes      | Yes             | Yes                                           |
| `dict`         | Yes      | Yes             | Yes                                           |
| `cycler`, `joiner`, `namespace` | Yes | Yes        | Yes                                           |
| `lipsum`       | Yes      | Yes             | Yes                                           |
| `lookup(plugin, …)` | No  | **No**          | **Yes**                                       |
| `query(plugin, …)` | No   | **No**          | **Yes** (wraps `lookup` but always returns a list) |
| `now()`        | No       | **No**          | **Yes** (returns a `datetime.datetime`)       |
| `undef()`      | No       | **No**          | **Yes** (forces strict undefined)             |
| `q(plugin, …)` | No       | **No**          | **Yes** (alias for `query`)                   |

> A common Ansible idiom — `{{ lookup('file', 'banner.txt') }}` — has no
> equivalent in Catalyst Center. Catalyst Center templates cannot read
> from the filesystem (see
> [jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md#6-no-filesystem-access-from-a-template)).

---

## 8. Strict-undefined behaviour

| Behaviour                                | Catalyst Center | Ansible (default)          |
|------------------------------------------|-----------------|----------------------------|
| Reference an undeclared variable         | Renders empty   | **Task fails** with `'X' is undefined` |
| Use `is defined` / `is not defined`      | Works           | Works                      |
| Use `default('X')`                       | Works           | Works                      |
| Force a failure                          | Not available   | `{{ var | mandatory }}` or `{{ undef() }}` |

This single difference is the most frequent porting trap. A template that
silently produces empty CLI in Catalyst Center will fail loudly in
Ansible — which is usually what you want. To make a CC template robust
**before** porting it to Ansible, wrap suspicious references:

```j2
{% if Hostname is defined and Hostname %}
hostname {{ Hostname }}
{% endif %}
```

---

## 9. Cross-template composition

Both engines support `include`, `extends`, `import`, and
`from … import …` with identical syntax. What differs is the **resolver**.

### 9.1 Catalyst Center

* Path is **relative to the Template Editor project tree**:
  ```j2
  {% include "DayNTemplates/DEFN-VlanInfo.j2" %}
  ```
* Project names with spaces work but should be URL-safe.
* Composite templates (`*-Composite.yml`) provide an alternative,
  declarative composition mechanism that is **DayN only**.

### 9.2 Ansible

* Path is relative to the **role's `templates/` directory** or any path
  listed in `template_path`:
  ```jinja2
  {% include 'partials/vlan_block.j2' %}
  ```
* `ansible.builtin.template` is the module that renders; the `template`
  filter does the same in-string.

> Direct copy/paste of `{% include "DayNTemplates/…" %}` from CC into an
> Ansible role will fail until the path is relocated under
> `roles/<role>/templates/`.

---

## 10. Whitespace, trimming, and output post-processing

| Knob                          | Catalyst Center                                      | Ansible                                              |
|-------------------------------|------------------------------------------------------|------------------------------------------------------|
| `trim_blocks`                 | Defaults observed in CC: trim_blocks **off**         | Off by default; enable per task with `lstrip_blocks: yes` (template module) |
| `lstrip_blocks`               | Off by default                                       | Off by default; can be set via module args / config  |
| `keep_trailing_newline`       | On (output preserved)                                | Off by default (final newline stripped)              |
| `{%- ... -%}` trim modifiers  | Same semantics                                       | Same semantics                                       |
| Post-render text passes       | None on Jinja2 output (CC then parses CLI markers like `#MODE_ENABLE` and `<MLTCMD>`) | None (raw text is written or sent verbatim)    |

Stylistic implication: a CC template that renders pretty CLI may render an
extra blank line at the bottom when used with Ansible's
`ansible.builtin.template` module, and vice versa. Use
`{%- ... -%}` consistently to neutralise this.

---

## 11. CC-only CLI markers

These markers exist purely because the Catalyst Center provisioner
post-processes the rendered text before pushing it to a device. They are
**not** Jinja2 tags and have **no meaning in Ansible**.

| Marker                                          | What CC does with it                              | Ansible behaviour            |
|-------------------------------------------------|---------------------------------------------------|------------------------------|
| `#MODE_ENABLE` … `#MODE_END_ENABLE`             | Runs the enclosed CLI in privileged EXEC          | Emitted verbatim as comments |
| `<MLTCMD>` … `</MLTCMD>`                        | Bundles a multi-line CLI command (banners, certs) | Emitted verbatim as text     |
| `! @ start-ignore-compliance` / `! @ end-ignore-compliance` | Excludes the block from compliance scans | Emitted verbatim as comments |

When porting CC → Ansible (e.g., to push via `ios_config`), strip these
markers and reorganise around the target module's idioms (e.g.,
`cisco.ios.ios_banner`, `cisco.ios.ios_command` for EXEC, ignoring
compliance entirely).

---

## 12. Loop and control behaviour

Identical core: `loop.index`, `loop.index0`, `loop.first`, `loop.last`,
`loop.length`, `loop.previtem`, `loop.nextitem`, `loop.cycle(...)`,
`loop.depth`, `loop.changed(...)`, recursive `loop(...)`.

Difference: **`{% break %}` and `{% continue %}`**

* Catalyst Center: `loopcontrols` extension **not enabled** — these tags
  raise a template error.
* Ansible: `loopcontrols` is **also not enabled by default** in the
  template module; it is reachable only through custom Jinja2 environments
  or specific collections. Most Ansible authors restructure with
  `{% if %}` guards instead, just as in CC.

> Effective parity: in both environments, you should write loops as if
> `break`/`continue` did not exist.

---

## 13. The `do` extension

Enabled in **both** environments. `{% do list.append(x) %}` works
identically. This is the standard pattern in Catalyst Center for building
per-stack lists, and in Ansible for mutating dicts before serialising
them to YAML/JSON.

```j2
{% set PortTotal = [] %}
{% for pid in StackPIDs %}
  {% if '48' in pid %}
    {% do PortTotal.append(48) %}
  {% endif %}
{% endfor %}
```

---

## 14. Macros

Same syntax in both. Differences are minor but worth knowing:

* **Imports across files**: CC resolves against the project tree; Ansible
  against the role `templates/` directory.
* **`{% macro %}` with default args**: identical rules
  (defaults trail required args).
* **`caller()` / `{% call %}`**: works in both.
* **Recursive macros**: work in both.

There is **no** "shared global macro library" in either environment. Both
require an explicit `{% import %}` (or `{% from … import … %}`) in every
template that uses the macros.

---

## 15. Error handling and debugging

### 15.1 Catalyst Center

* Errors surface in the Template Editor's **Simulator** pane and on the
  **Provision** action.
* Common silent failures: undefined variable rendered empty, missing
  `endif`/`endfor` reported with a line number, mismatched delimiters.
* There is no `pdb`; the only inspection mechanism is
  *render-and-read-the-output*.

### 15.2 Ansible

* Errors surface as a **task failure** with traceback. With
  `ANSIBLE_DEBUG=1` and `-vvvv`, you get the rendered template inline.
* `{{ var | mandatory }}` forces an immediate failure with a clear message.
* `--check --diff` previews template output against a target file.
* `ansible.builtin.debug: msg: "{{ … }}"` is the standard inspection idiom.

---

## 16. Performance & rendering model

| Aspect                  | Catalyst Center                                          | Ansible                                                    |
|-------------------------|----------------------------------------------------------|------------------------------------------------------------|
| Render scope            | One template per device (or one composite chain)         | One template per task; same play renders many in sequence  |
| Caching                 | Re-rendered on every preview / push                      | Re-rendered on every task; facts cached separately         |
| Concurrency             | Server-side, single-render per provisioning              | Controller-side, concurrent across `forks` workers         |
| Network calls inside template | **None** possible                                   | Allowed via lookups (`url`, `pipe`, `community.*`)         |

---

## 17. Porting checklist

When moving a `.j2` between Catalyst Center and Ansible, walk through:

1. **Operators** — replace `&&` / `||` / `!` with `and` / `or` / `not`.
2. **Filters** —
   * CC → Ansible: `split(",")` filter usage is fine (Ansible has it too as
     `.split()` method; also exposed via `split` filter in many
     environments — verify), but `__device.*` references must be replaced
     with `ansible_facts.*` or `hostvars[…]…`.
   * Ansible → CC: `to_yaml`, `to_json`, `ipaddr`, `regex_replace`, and
     `combine` have no CC equivalent. Pre-compute the values in the form
     or in a Network Profile.
3. **Globals** — strip `lookup(...)`, `query(...)`, `now()`, `hostvars`,
   `groups`, `inventory_hostname` when porting Ansible → CC.
4. **System variables** — replace `__device` / `__interface` /
   `__networkSettings` with Ansible inventory / facts when porting CC →
   Ansible.
5. **CLI markers** — strip `#MODE_ENABLE` / `<MLTCMD>` / compliance markers
   when porting CC → Ansible; reintroduce them when porting back.
6. **`include` / `extends`** — relocate the referenced file into the role's
   `templates/` directory (Ansible) or under the same project tree (CC).
7. **Undefined handling** — wrap suspicious references in `is defined`
   guards before targeting Ansible's strict mode.
8. **PnP vs DayN constraints** — if the source is a CC PnP template,
   confirm no bind / `__*` references slipped in.

---

## 18. Side-by-side: same intent, two engines

### 18.1 "Hostname from inventory, fallback to a user value"

**Catalyst Center**

```j2
{% if __device.hostname %}
hostname {{ __device.hostname }}
{% else %}
hostname {{ Hostname }}
{% endif %}
```

**Ansible**

```jinja2
hostname {{ ansible_facts.ansible_hostname | default(hostname_override) }}
```

### 18.2 "Loop the device's trunk uplinks"

**Catalyst Center**

```j2
{% for iface in __interface %}
  {% if iface.portMode == "trunk" and iface.interfaceType == "Physical" %}
interface {{ iface.portName }}
  description UPLINK
  {% endif %}
{% endfor %}
```

**Ansible** (assuming `ios_facts` populated `ansible_facts.net_interfaces`)

```jinja2
{% for name, iface in ansible_facts.net_interfaces.items() %}
{% if iface.mode == 'trunk' and iface.type == 'physical' %}
interface {{ name }}
  description UPLINK
{% endif %}
{% endfor %}
```

### 18.3 "Stack of 9300s — assign priorities"

**Catalyst Center**

```j2
{% set StackPIDs        = __device.platformId | split(",") %}
{% set StackMemberCount = StackPIDs | length %}
#MODE_ENABLE
{% for s in range(StackMemberCount) %}
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

**Ansible**

```jinja2
{% set pids = stack_platform_ids %}
{% for pid in pids %}
{% if loop.index == 1 %}
switch {{ loop.index }} priority 10
{% elif loop.index == 2 %}
switch {{ loop.index }} priority 9
{% else %}
switch {{ loop.index }} priority 8
{% endif %}
{% endfor %}
```

Push via `cisco.ios.ios_command` (EXEC) — Ansible has no concept of
`#MODE_ENABLE`; the *module* selects the mode.

---

## 19. Quick reference card

| If you see this in a `.j2`                | It's…              | Port to the other side by…                                          |
|-------------------------------------------|--------------------|---------------------------------------------------------------------|
| `__device.platformId`                     | CC-only            | Replace with `ansible_facts.*` / inventory variable                 |
| `&&`, `\|\|`, `!`                          | CC-only (accepted) | Replace with `and`, `or`, `not`                                     |
| `\| split(",")`                            | CC-extra filter    | Use `.split(",")` method (works in both)                            |
| `#MODE_ENABLE` / `<MLTCMD>` / `! @ …-compliance` | CC-only      | Strip when porting to Ansible; choose the right module instead      |
| `\| to_yaml`, `\| ipaddr(...)`, `\| regex_replace(...)` | Ansible-only | Pre-compute in form / bind / Network Profile; strip in CC      |
| `hostvars`, `groups`, `inventory_hostname`, `lookup(...)`, `now()` | Ansible-only | No CC equivalent; rework data flow upstream             |
| `{% include "DayNTemplates/…" %}`         | CC project path    | Move file under `roles/<role>/templates/` and adjust path           |
| Undefined variable renders empty silently | CC behaviour       | Add `is defined` guards or `\| mandatory` before targeting Ansible  |

---

## See also

* [Jinja2 basics](../../docs/jinja2/basics.md) — CC Jinja2 tutorial.
* [AdvancedJinja2](../../docs/jinja2/advanced.md) — advanced CC patterns.
* [Jinja2 variable types](./variable-types.md)
* [Jinja2 constraints](./constraints.md)
* [guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md)
* Ansible docs: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html>
* Pallets Jinja2 docs: <https://jinja.palletsprojects.com/en/3.0.x/templates/>
