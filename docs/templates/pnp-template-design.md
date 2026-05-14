---
title: "Templates — PnP (Day-0) onboarding template design"
engine: shared
kind: docs
topic: pnp-template-design
path: docs/templates/pnp-template-design.md
---

# PnP (Day-0) onboarding template design

How to design a Catalyst Center **Onboarding Configuration** template (Day-0,
Greenfield). Read [considerations.md](considerations.md) first — the rules
about no inventory, no system variables, no bind variables, no compliance
tracking, and the mandatory auto-config block all govern what is sensible to
put here.

## 1. Minimum-viable scope

Include in PnP **only what is required to establish stable management
connectivity** between the device and Catalyst Center. Everything else
belongs in DayN, where compliance applies and inventory bindings are
available.

| Category | Include |
|----------|---------|
| System | Hostname, system MTU (only if non-default), VTP mode/domain, enable netconf-yang |
| Management interface | Mgmt VLAN, SVI IP / mask, `no ip redirects`, `no ip proxy-arp` |
| Next hop | Default gateway (L2 switch) or routing-protocol bootstrap (L3 switch) |
| Source traffic | `ip http client`, `ip ssh`, `ip ftp`, `ip tftp`, `ip radius`, `ip domain lookup`, `logging`, `snmp-server trap-source`, `ntp source` — all bound to the mgmt SVI |
| Uplink | Trunk member ports, optional LACP port-channel |

The `ip http client source-interface` line is non-optional: it tells the
HTTP client which interface to source from, which is how Catalyst Center
detects post-PnP IP address changes and updates Inventory automatically.

## 2. Build iteratively — basic → variables → logic → guards

The recommended authoring loop is to start with a working static config,
then variabilize, then add conditional logic, then add edge-case guards.
Test in the **Simulation** tab between every iteration.

### Iteration 1 — static CLI

Paste in a known-good configuration as the starting point:

```text
hostname c9300-1
!
vlan 5
 name mgmtvlan
!
interface vlan 5
 ip address 192.168.5.3 255.255.255.0
 no shutdown
 exit
!
interface vlan 1
 shutdown
!
ip default-gateway 192.168.5.1
!
interface range gigabitethernet1/0/10-11
 switchport mode trunk
 switchport trunk allowed vlan 5
 channel-protocol lacp
 channel-group 1 mode active
 no shut
!
interface port-channel 1
 switchport trunk native vlan 5
 switchport trunk allowed vlan 5
 switchport mode trunk
 no port-channel standalone-disable
!
```

### Iteration 2 — variables (DRY)

Replace literals with `{{Variable}}` placeholders so one template fits many
devices. Canonical variable set for a Layer-2 mgmt-VLAN onboarding:

| Literal | Variable |
|---------|----------|
| `c9300-1`            | `{{Hostname}}` |
| `5`                  | `{{MgmtVlan}}` |
| `Gi1/0/10, Gi1/0/11` | `{{Interfaces}}` |
| `192.168.5.3`        | `{{SwitchIP}}` |
| `255.255.255.0`      | `{{SubnetMask}}` |
| `192.168.5.1`        | `{{Gateway}}` |
| `1`                  | `{{Portchannel}}` |
| `1500`               | `{{SystemMTU}}` |

### Iteration 3 — conditional logic on the interface list

`interface range` accepts three forms: single (`gi1/0/1`), comma-delimited
(`gi1/0/1, gi1/0/2`), or dash range (`gi1/0/1-2`). Use this to make
port-channel optional — a single-uplink small-branch device needs no
LACP, a dual-uplink branch does.

[//]: # ({% raw %})
```j2
interface range {{Interfaces}}
 shut
 switchport mode trunk
 switchport trunk allowed vlan {{MgmtVlan}}
 {% if "," in Interfaces || "-" in Interfaces %}
    channel-protocol lacp
    channel-group {{Portchannel}} mode active
 {% endif %}
 no shut
!
{% if "," in Interfaces || "-" in Interfaces %}
  interface Port-channel {{Portchannel}}
   switchport trunk native vlan {{MgmtVlan}}
   switchport trunk allowed vlan {{MgmtVlan}}
   switchport mode trunk
   no port-channel standalone-disable
{% endif %}
```
[//]: # ({% endraw %})

Note: Catalyst Center Jinja2 accepts `&&` / `||` / `!` as logical operators
in addition to `and` / `or` / `not`. See
[rules/jinja2/cc-vs-ansible.md](../../rules/jinja2/cc-vs-ansible.md).

### Iteration 4 — VTP, VLAN naming, and the VLAN-1 guard

Set VTP to transparent mode and derive the domain from the hostname. Only
name and shutdown VLAN 1 when the mgmt VLAN is non-default:

[//]: # ({% raw %})
```j2
{% set VtpDomain = Hostname %}
vtp domain {{VtpDomain}}
vtp mode transparent
!
vlan {{MgmtVlan}}
!
{% if MgmtVlan > 1 %}
  name MgmtVlan
  interface Vlan 1
   shutdown
{% endif %}
```
[//]: # ({% endraw %})

### Iteration 5 — system MTU guard, source-interface block, SVI hardening, the `Portchannel` workaround

Only set `system mtu` when it differs from default:

[//]: # ({% raw %})
```j2
{% if SystemMTU != 1500 %}
   system mtu {{SystemMTU}}
{% endif %}
```
[//]: # ({% endraw %})

Bind every management protocol to the mgmt SVI and enable NETCONF:

[//]: # ({% raw %})
```j2
ip domain lookup source-interface Vlan {{MgmtVlan}}
ip http client source-interface Vlan {{MgmtVlan}}
ip ftp source-interface Vlan {{MgmtVlan}}
ip tftp source-interface Vlan {{MgmtVlan}}
ip ssh source-interface Vlan {{MgmtVlan}}
ip radius source-interface Vlan {{MgmtVlan}}
logging source-interface Vlan {{MgmtVlan}}
snmp-server trap-source Vlan {{MgmtVlan}}
ntp source Vlan {{MgmtVlan}}
!
netconf-yang
```
[//]: # ({% endraw %})

Harden the SVI:

[//]: # ({% raw %})
```j2
interface Vlan {{ MgmtVlan }}
 description MgmtVlan
 ip address {{ SwitchIP }} {{ SubnetMask }}
 no ip redirects
 no ip proxy-arp
 no shut
```
[//]: # ({% endraw %})

#### The `!{{Portchannel}}` first-use workaround

Catalyst Center has a known issue where a variable referenced **only inside
a conditional** can fail to register. Workaround: reference the variable
once, prefixed with `!`, *before* the conditional so the parser sees it
unconditionally. The `!` makes IOS treat the line as a comment, so it has
no functional effect:

[//]: # ({% raw %})
```j2
!{{Portchannel}}
interface range {{Interfaces}}
 ...
```
[//]: # ({% endraw %})

## 3. Reference template — full Jinja2

Canonical Day-0 Jinja2 template combining all five iterations:

[//]: # ({% raw %})
```j2
{# <------Onboarding-Template-------> #}
{# To be used for onboarding when using Day N Templates #}
!
{# Set MTU if required #}
{% if SystemMTU != 1500 %}
   system mtu {{ SystemMTU }}
{% endif %}
!
{# Set hostname #}
hostname {{ Hostname }}
!
{% set VtpDomain = Hostname %}
!
{# Set VTP and VLAN for onboarding #}
vtp domain {{ VtpDomain }}
vtp mode transparent
!
{# Set Management VLAN #}
vlan {{ MgmtVlan }}
!
{% if MgmtVlan > 1 %}
  name MgmtVlan
  {# Disable Vlan 1 (optional) #}
  interface Vlan 1
   shutdown
{% endif %}
!
{# Set Interfaces and Build Port Channel #}
!{{ Portchannel }}
interface range {{ Interfaces }}
 shut
 switchport mode trunk
 switchport trunk allowed vlan {{ MgmtVlan }}
 {% if "," in Interfaces || "-" in Interfaces %}
    channel-protocol lacp
    channel-group {{ Portchannel }} mode active
 {% endif %}
 no shut
!
{% if "," in Interfaces || "-" in Interfaces %}
  interface Port-channel {{ Portchannel }}
   switchport trunk native vlan {{ MgmtVlan }}
   switchport trunk allowed vlan {{ MgmtVlan }}
   switchport mode trunk
   no port-channel standalone-disable
{% endif %}
!
{# Set Up Management Vlan #}
interface Vlan {{ MgmtVlan }}
 description MgmtVlan
 ip address {{ SwitchIP }} {{ SubnetMask }}
 no ip redirects
 no ip proxy-arp
 no shut
!
ip default-gateway {{ Gateway }}
!
{# Source of Management Traffic #}
ip domain lookup source-interface Vlan {{ MgmtVlan }}
ip http client source-interface Vlan {{ MgmtVlan }}
ip ftp source-interface Vlan {{ MgmtVlan }}
ip tftp source-interface Vlan {{ MgmtVlan }}
ip ssh source-interface Vlan {{ MgmtVlan }}
ip radius source-interface Vlan {{ MgmtVlan }}
logging source-interface Vlan {{ MgmtVlan }}
snmp-server trap-source Vlan {{ MgmtVlan }}
ntp source Vlan {{ MgmtVlan }}
!
netconf-yang
!
```
[//]: # ({% endraw %})

## 4. Variables (Form tab) — field-name / type / default mapping

Each variable parsed from the template appears in the **Variables** tab,
where you set its display label, type, required flag, and default. Defaults
apply when the operator does not override during claim.

| Variable     | Field Name             | Type       | Required | Default              |
|--------------|------------------------|------------|----------|----------------------|
| `SystemMTU`  | System MTU             | Integer    | No       | `1500`               |
| `Hostname`   | Hostname               | String     | Yes      | `c9300-1`            |
| `MgmtVlan`   | Management Vlan        | Integer    | Yes      | `5`                  |
| `SwitchIP`   | Management IP Address  | IP Address | Yes      | `192.168.5.3`        |
| `SubnetMask` | Management Subnet Mask | IP Address | Yes      | `255.255.255.0`      |
| `Gateway`    | Gateway Address        | IP Address | Yes      | `192.168.5.1`        |
| `Portchannel`| Port-Channel Number    | Integer    | No       | `1`                  |
| `Interfaces` | Uplink Interfaces      | String     | Yes      | `Gi1/0/10, Gi1/0/11` |

Use **Review Form** (bottom right of the Variables tab) to preview the form
exactly as the claim operator will see it. Bulk-import via CSV uses the
field labels (Field Name), not the raw variable names.

## 5. Simulation matrix

Before saving and committing, run the **Simulation** tab against a model
that matches the deployment target. A minimum suite to exercise the logic:

| Run | `SystemMTU` | `MgmtVlan` | `Interfaces`        | `Portchannel` |
|-----|-------------|------------|---------------------|---------------|
| 1   | `1500`      | `5`        | `Gi1/0/10, Gi1/0/11`| `1`           |
| 2   | `1500`      | `1`        | `Gi1/0/10, Gi1/0/11`| `1`           |
| 3   | `1500`      | `5`        | `Gi1/0/10-11`       | `110`         |
| 4   | `1450`      | `5`        | `Gi1/0/10`          | `1`           |

Validate per run:

- Run 2 omits the `name MgmtVlan` / `interface Vlan 1 shutdown` block.
- Run 3 still renders LACP and Port-channel (dash counts as a multi-port marker).
- Run 4 omits LACP and Port-channel (single uplink, no delimiter).
- Run 4 emits `system mtu 1450`.
- Every run emits exactly one `!{{Portchannel}}` placeholder line.

## 6. Velocity authors

The methodology — minimum scope, iterative variabilization, conditional
guards, form fields, simulation — applies identically to Velocity. The
syntax differences are the only delta: `#set( $Hostname = "..." )`,
`#if( $Interfaces.contains(",") )`, `$MgmtVlan`. See
[rules/velocity/constraints.md](../../rules/velocity/constraints.md) and
[docs/velocity/basics.md](../velocity/basics.md). Note that Velocity
`#parse` / `#include` are disabled in Catalyst Center, so Velocity PnP
templates cannot pull shared snippets — all logic must be inlined.

---

### Source corrections

The original walkthrough (archived at
[docs/source/module2-day0-template.md](../source/module2-day0-template.md))
contained three typographical errors in its consolidated code listing that
would prevent the template from rendering. They have been corrected in this
document:

1. `hostname (Hostname}}` → `hostname {{Hostname}}` — opening `(` should
   have been `{{`.
2. `interface Port-channel ((Portchannel}}` → `interface Port-channel
   {{Portchannel}}` — double `((` should have been `{{`.
3. `{% if "," in Interfaces || in Interfaces %}` →
   `{% if "," in Interfaces || "-" in Interfaces %}` — the second
   delimiter (`"-"`) was omitted.

The step-by-step snippets earlier in the source module were correct; only
the final "expand to copy" consolidated listing was affected.
