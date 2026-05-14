---
title: "Templates — DayN template design methodology"
engine: shared
kind: docs
topic: dayn-template-design
path: docs/templates/dayn-template-design.md
---

# DayN template design methodology

How to design DayN templates for **ongoing** configuration of a switch
already in Inventory. Read [considerations.md](considerations.md) first —
the design-vs-template overlap rule and the engine recommendation drive
every decision here.

DayN templates can use the full power of Catalyst Center:

- System variables (`__device`, `__interface`, …) — see
  [docs/system-variables.md](../system-variables.md).
- Bind variables (Inventory-sourced) — see
  [docs/variables.md](../variables.md).
- Composite templates that group Regular templates and mix Jinja2 + Velocity.
- Compliance scanning, drift detection, re-push.

## 1. Regular vs Composite

| Template type | Purpose | Used in |
|---------------|---------|---------|
| **Regular** | One focused feature (system config, AAA, interface config, ACL, EEM, …). Authored in **either** Jinja2 or Velocity. Small, single-responsibility, reusable. | PnP and DayN |
| **Composite** | A logical container that groups multiple Regular templates in a defined execution order. Engine-agnostic — Jinja2 and Velocity Regulars can coexist. | **DayN only** |

The DRY (Don't Repeat Yourself) philosophy maps to this split: write each
configuration concern once as a Regular template, then assemble Composites
to deliver the right combination per device role.

## 2. The analyze → discount → modularize workflow

Given a known-good running configuration from a reference device, derive
the template set in three passes.

### Pass A — start with the full reference config

Source of truth for everything the device needs to look like in production.
Example below is a 48-port C9300 access switch:

<details>
<summary>Expand reference configuration</summary>

```text
service tcp-keepalives-in
service tcp-keepalives-out
service timestamps debug datetime msec localtime show-timezone
service timestamps log datetime msec show-timezone year
service password-encryption
service sequence-numbers
service call-home
no platform punt-keepalive disable-kernel-core
!
hostname c9300-1
!
vrf definition Mgmt-vrf
 address-family ipv4
 exit-address-family
 address-family ipv6
 exit-address-family
!
logging buffered 16384 informational
no logging console
aaa new-model
!
aaa group server radius CON-VTY
 server-private 198.18.133.1 key 7 <redacted>
 ip radius source-interface Vlan5
 deadtime 6
!
aaa authentication login default local
aaa authentication login CON-LAB local-case
aaa authorization console
aaa authorization exec default local
aaa authorization exec CON-LAB local
aaa authorization network default group dnac-client-radius-group
aaa accounting network default start-stop group radius
aaa session-id common
!
clock timezone EST -5 0
clock summer-time EDT recurring
switch 1 provision c9300-48u
!
ip routing
!
ip name-server 198.18.133.1
ip domain lookup source-interface Vlan5
ip domain name base2hq.com
!
lldp run
!
vtp domain base2hq.com
vtp mode transparent
vtp version 1
!
port-channel load-balance src-dst-ip
!
spanning-tree mode rapid-pvst
spanning-tree portfast default
spanning-tree portfast bpduguard default
spanning-tree extend system-id
!
enable secret 9 <redacted>
username admin privilege 15 secret 9 <redacted>
!
redundancy
 mode sso
!
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
!
interface Port-channel1
 switchport trunk native vlan 5
 switchport trunk allowed vlan 5,10,20,30,40
 switchport mode trunk
!
interface GigabitEthernet0/0
 vrf forwarding Mgmt-vrf
 no ip address
 shutdown
 negotiation auto
!
interface range GigabitEthernet1/0/1-48
 switchport access vlan 20
 switchport voice vlan 30
 spanning-tree portfast
!
interface TenGigabitEthernet1/1/1, TenGigabitEthernet1/1/2
 switchport trunk allowed vlan 1,5,10,20,30,40
 switchport mode trunk
 channel-group 1 mode active
 channel-protocol lacp
!
interface Vlan5
 description MgmtVlan
 ip address 192.168.5.3 255.255.255.0
 no ip redirects
 no ip proxy-arp
!
ip forward-protocol nd
ip http server
ip http authentication local
ip http secure-server
ip http secure-trustpoint <redacted>
ip http client source-interface Vlan5
ip ssh source-interface Vlan5
ip ssh version 2
!
ip radius source-interface Vlan5
logging origin-id ip
logging source-interface Vlan5
logging host 10.10.0.20
logging host 198.18.133.27 transport udp port 20514
!
snmp-server community private RW
snmp-server community public RO
snmp-server trap-source Vlan5
snmp-server enable traps all
snmp-server host 10.10.0.20 version 2c private
snmp-server host 198.18.133.27 version 2c private
snmp-server host 10.10.0.20 version 2c public
snmp-server host 198.18.133.27 version 2c public
!
radius-server attribute 6 on-for-login-auth
radius-server attribute 6 support-multiple
radius-server attribute 8 include-in-access-req
radius-server attribute 25 access-request include
radius-server attribute 31 mac format ietf upper-case
radius-server attribute 31 send nas-port-detail mac-only
radius-server dead-criteria time 5 tries 3
radius-server retransmit 2
radius-server deadtime 3
!
radius server dnac-radius_198.18.133.27
 address ipv4 198.18.133.27 auth-port 1812 acct-port 1813
 timeout 4
 retransmit 3
 automate-tester username dummy ignore-acct-port probe-on
 pac key 7 <redacted>
!
banner login ^
 ******************************************************************
 * CISCO SYSTEMS EXAMPLE                                          *
 ******************************************************************
^
!
line con 0
 exec-timeout 0 0
 privilege level 15
 authorization exec CON-LAB
 logging synchronous
 login authentication CON-LAB
 stopbits 1
line vty 0 31
 exec-timeout 0 0
 authorization exec CON-LAB
 logging synchronous
 login authentication CON-LAB
 terminal-type mon
 length 0
 transport input ssh
!
ntp source Vlan5
ntp server 198.18.133.1
!
end
```

</details>

### Pass B — discount what Catalyst Center already provides

Two categories of CLI must come out of the template, otherwise it duplicates
or conflicts with what Catalyst Center pushes elsewhere.

**B.1 — already pushed by the PnP auto-config block** (see
[considerations.md §3](considerations.md#3-mandatory-auto-config-deployed-by-catalyst-center-at-claim))
and by the PnP onboarding template (hostname, mgmt VLAN, mgmt SVI, default
gateway, source-interface bindings, netconf-yang). All of this is already
in place after claim and must not appear in DayN.

**B.2 — already pushed by Design Settings.** Anything you've populated in
Network Settings / Telemetry / AAA in the UI:

<details>
<summary>Lines removed because the PnP auto-config or PnP template or Design app already pushes them</summary>

```text
service password-encryption
service sequence-numbers
service call-home
no platform punt-keepalive disable-kernel-core
!
hostname ASW-9300-ACCESS
!
vrf definition Mgmt-vrf
 address-family ipv4
 exit-address-family
 address-family ipv6
 exit-address-family
!
switch 1 provision c9300-48u
!
ip name-server 198.18.133.1
ip domain lookup source-interface Vlan5
ip domain name base2hq.com
!
vtp domain base2hq.com
vtp mode transparent
vtp version 1
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
!
enable secret 9 <redacted>
username admin privilege 15 secret 9 <redacted>
!
vlan 5
 name mgmtvlan
!
interface Port-channel1
 switchport trunk native vlan 5
 switchport trunk allowed vlan 5
 switchport mode trunk
!
interface GigabitEthernet0/0
 vrf forwarding Mgmt-vrf
 no ip address
 negotiation auto
!
interface TenGigabitEthernet1/1/1, TenGigabitEthernet1/1/2
 switchport trunk allowed vlan 1,5
 switchport mode trunk
 channel-group 1 mode active
 channel-protocol lacp
!
interface Vlan5
 description MgmtVlan
 ip address 192.168.5.3 255.255.255.0
 no ip redirects
 no ip proxy-arp
!
ip ssh source-interface Vlan5
ip ssh version 2
!
ip radius source-interface Vlan5
logging source-interface Vlan5
logging host 10.10.0.20
logging host 198.18.133.27 transport udp port 20514
!
snmp-server community private RW
snmp-server community public RO
snmp-server trap-source Vlan5
snmp-server enable traps all
snmp-server host 10.10.0.20 version 2c private
snmp-server host 198.18.133.27 version 2c private
snmp-server host 10.10.0.20 version 2c public
snmp-server host 198.18.133.27 version 2c public
!
ntp source Vlan5
ntp server 198.18.133.1
!
end
```

</details>

### Pass C — the residue is what the template owns

What remains after Passes B.1 and B.2 is the actual scope of the DayN
template set:

<details>
<summary>Residual configuration owned by DayN templates</summary>

```text
service tcp-keepalives-in
service tcp-keepalives-out
service timestamps debug datetime msec localtime show-timezone
service timestamps log datetime msec show-timezone year
!
logging buffered 16384 informational
no logging console
!
clock timezone EST -5 0
clock summer-time EDT recurring
!
ip routing
!
udld enable
lldp run
!
port-channel load-balance src-dst-ip
!
spanning-tree portfast default
spanning-tree portfast bpduguard default
!
redundancy
 mode sso
!
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
!
interface Port-channel1
 switchport trunk allowed vlan 5,10,20,30,40
!
interface GigabitEthernet0/0
 shutdown
!
interface range GigabitEthernet1/0/1-48
 switchport access vlan 20
 switchport voice vlan 30
 spanning-tree portfast
!
interface TenGigabitEthernet1/1/1, TenGigabitEthernet1/1/2
 switchport trunk allowed vlan 1,5,10,20,30,40
!
ip forward-protocol nd
ip http server
ip http authentication local
ip http secure-server
ip http secure-trustpoint <redacted>
ip ssh source-interface Vlan5
ip ssh version 2
!
logging origin-id ip
!
banner login ^ ... ^
!
line con 0
 exec-timeout 0 0
 ...
line vty 0 31
 ...
!
end
```

</details>

### Pass D — group the residue into Regular templates

Three natural groupings:

- **SysConfig** — services, LLDP, port-channel load-balance, spanning-tree
  options, logging defaults, clock.
- **AaaConfig** — AAA new-model, login authentication/authorization, HTTP
  off, SSH v2, banner (wrapped in `<MLTCMD>`), netconf-yang, line con / vty.
- **IntConfig** — VLAN database, port-channel allowed-vlan, mgmt interface
  shutdown, host-port macro applied to non-uplink physical interfaces.

## 3. Regular template — SysConfig (Jinja2)

[//]: # ({% raw %})
```j2
{# System Configuration #}
no service pad
service tcp-keepalives-in
service tcp-keepalives-out
service timestamps debug datetime msec localtime show-timezone
service timestamps log datetime msec show-timezone
service password-encryption
service sequence-numbers
!
lldp run
port-channel load-balance src-dst-ip
!
spanning-tree mode rapid-pvst
spanning-tree portfast default
spanning-tree portfast bpduguard default
!
logging buffered 16384 informational
no logging console
logging origin-id ip
!
clock timezone EST -5 0
clock summer-time EDT recurring
```
[//]: # ({% endraw %})

## 4. Regular template — AaaConfig (Jinja2) — `<MLTCMD>` banner

The login banner is a multi-line CLI block. Wrap it in `<MLTCMD>` /
`</MLTCMD>` so Catalyst Center sends the entire banner as a single CLI
unit instead of one line at a time:

[//]: # ({% raw %})
```j2
{# AAA Configuration #}
aaa new-model
aaa authentication username-prompt "Authorized Username: "
aaa authentication login admin local
aaa authorization console
aaa authorization exec admin local
aaa authentication login admin local-case
aaa authorization exec admin local
!
ip forward-protocol nd
no ip http server
no ip http authentication local
no ip http secure-server
ip ssh version 2
!
<MLTCMD>
banner login ^
 ******************************************************************
 * CISCO SYSTEMS EXAMPLE                                          *
 ******************************************************************
 *                                                                *
 * THIS DEVICE IS PART OF A DEMONSTRATION COMPUTER NETWORK AND IS *
 * PROVIDED FOR OFFICIAL USE BY AUTHORIZED USERS ONLY.            *
 *                                                                *
 ******************************************************************
   Hostname: $(hostname)
 ******************************************************************
 ^
</MLTCMD>
!
netconf-yang
!
line con 0
 exec-timeout 15 0
 login authentication admin
 logging synchronous
 authorization exec admin
 stopbits 1
line vty 0 15
 exec-timeout 15 0
 login authentication admin
 logging synchronous
 authorization exec admin
 transport input ssh
```
[//]: # ({% endraw %})

`<MLTCMD>` is mandatory for any block where line-by-line CLI delivery would
break IOS parsing — banners, certificate chains, key chains. See
[guardrails/jinja2/what-cc-cannot-do.md](../../guardrails/jinja2/what-cc-cannot-do.md)
for the full list of multi-line constructs that need this wrapper.

## 5. Regular template — IntConfig (Jinja2) — dictionaries, macros, `__interface`

The interface template combines three reusable Jinja2 patterns:

1. **VLAN dictionary** — a list of `{vlan, name}` pairs, iterated by macro
   to emit the VLAN database.
2. **Macro for repeating CLI** — `workstation_interface(vlan, voice)`
   factors the per-port stanza so it's written once and applied many times.
3. **`__interface` system variable** — DayN-only loop that walks every
   physical interface on the device and conditionally applies the
   workstation macro to non-uplink, slot-0-excluded, module-1-excluded
   ports.

[//]: # ({% raw %})
```j2
{# Interface Configuration #}
!
{# Vlan Dictionary #}
{% set VlanData = [
    {'vlan':'5','name':'mgmtvlan'},
    {'vlan':'10','name':'apvlan'},
    {'vlan':'20','name':'datavlan'},
    {'vlan':'30','name':'voicevlan'},
    {'vlan':'40','name':'guestvlan'},
    {'vlan':'999','name':'disabledvlan'}
  ] %}
!
{# MACRO: build the VLAN database from the dictionary #}
{% macro configure_vlans(vlanpairs) %}
  {% for vlanpair in vlanpairs %}
    vlan {{ vlanpair['vlan'] }}
     name {{ vlanpair['name'] }}
  {% endfor %}
{% endmacro %}
!
{# MACRO: collect VLAN IDs into an array for later use #}
{% macro search_vlans(vlanpairs) %}
  {% for vlanpair in vlanpairs %}
    {% do vlanArray.append(vlanpair['vlan']) %}
  {% endfor %}
{% endmacro %}
!
{# MACRO: per-port workstation stanza #}
{% macro workstation_interface(vlan_number, voice_number) %}
 description Workstation Interface
 switchport access vlan {{ vlan_number }}
 switchport voice vlan {{ voice_number }}
 switchport mode access
{% endmacro %}
!
{# Render the VLAN database and populate vlanArray #}
{% set vlanArray = [] %}
{{ configure_vlans(VlanData) }}
{{ search_vlans(VlanData) }}
!
{# Update the uplink port-channel allowed VLANs from the collected list #}
interface Port-channel 1
 switchport trunk allowed vlan add {{ vlanArray | join(',') }}
!
{# Management interface shutdown #}
interface GigabitEthernet0/0
 shutdown
!
{# Identify uplinks via __interface so we can exclude them from workstation config #}
{% set portarray = [] %}
{% for interface in __interface %}
  {% if interface.portMode == "trunk" && interface.interfaceType == "Physical" %}
    {% do portarray.append(interface.portName) %}
  {% endif %}
{% endfor %}
!
{# Apply workstation_interface to every physical Gi port that is not an uplink,
   not on slot 0, and not on module 1 (uplink module) #}
{% for interface in __interface %}
  {% if interface.interfaceType == "Physical"
    && interface.portName.replaceAll("(^[a-zA-Z]+).*","$1") == "GigabitEthernet" %}
    {% if interface.portName.replaceAll("(^[a-zA-Z]+.).*", "$1") != "GigabitEthernet0" %}
      {% if interface.portName.replaceAll("^[a-zA-Z]+(\\d+)/(\\d+)/(\\d+)", "$2") != 1 %}
        {% if interface.portName in portarray %}
        {% else %}
          default interface {{ interface.portName }}
          interface {{ interface.portName }}
           {{ workstation_interface(vlanArray[1], vlanArray[2]) }}
        {% endif %}
      {% endif %}
    {% endif %}
  {% endif %}
{% endfor %}
```
[//]: # ({% endraw %})

Notes on this pattern:

- `{% do %}` requires the `do` extension; it's enabled in Catalyst Center
  Jinja2.
- `__interface` is **only** available in DayN — using it in a PnP template
  yields an empty iterator. See
  [prompts/debug/pnp-vs-dayn-binding.md](../../prompts/debug/pnp-vs-dayn-binding.md).
- `vlanArray[1]` is `'10'` (apvlan) and `vlanArray[2]` is `'20'` (datavlan)
  given the dictionary above — ordering of `VlanData` is meaningful.
- `&&` is the Catalyst Center-flavoured Jinja2 `and`; `replaceAll` is the
  Java-style regex method exposed on string variables.

## 6. Composite template assembly

The Composite groups the Regulars into a single deliverable. Steps:

1. Create a new **Project** with **Composite Sequence** as the type and
   IOS-XE / Switches and Hubs / Cisco Catalyst 9300 Series as the scope.
   In the lab walkthrough this project is named `DayN-Templates-J2`.
2. Open the empty Composite template inside that project.
3. **Add Templates** → search for the Regulars by name prefix (`DayN-`),
   click **+** or drag each in.
4. Add any cross-cutting templates not in the prefix (for example
   `AutoNaming-EEM-Scripting`).
5. **Reorder** — drag to the execution order you need. The order matters:
   system services and AAA must precede anything that depends on them.
   Recommended order: SysConfig → AaaConfig → IntConfig → EEM/automation
   layers last.
6. **Save** → **Commit**.

Attaching the Composite to a Network Profile and pushing it through the
Provision workflow is covered in
[dayn-provisioning.md](dayn-provisioning.md).

## 7. Idempotency for re-pushable DayN

DayN templates may be re-pushed when configuration drifts or the template
is updated. Two patterns help ensure re-push is safe:

- **Use absolute statements that overwrite.** `switchport access vlan 20`
  unconditionally sets the value — no `no` is needed even if the previous
  value was different.
- **For removed features, push a `no` line.** Example: if a previous version
  of the template enabled `ip http server`, the new version must include
  `no ip http server` so the second push undoes the prior state.
- **Avoid `default interface` inside conditional branches that aren't always
  taken.** `default interface` wipes the port and is destructive — it
  belongs only where the template is the sole owner of that port's config
  (as in the IntConfig host-port loop above, where the loop's guards
  exclude uplinks).

## 8. Velocity authors

The methodology — analyze → discount → modularize, Regular vs Composite,
`<MLTCMD>` for banners, idempotency rules — applies identically to
Velocity. Velocity differences:

- Variables: `$VlanData`, `#set( $vlanArray = [] )`.
- Loops: `#foreach( $interface in $__interface )` … `#end`.
- Macros: `#macro( workstation_interface $vlan $voice ) ... #end`.
- `#parse` / `#include` are **disabled** in Catalyst Center — Velocity
  cannot pull shared snippets, all logic inlines.

See [rules/velocity/constraints.md](../../rules/velocity/constraints.md),
[docs/velocity/basics.md](../velocity/basics.md), and
[docs/velocity/advanced.md](../velocity/advanced.md).

---

### Source corrections

The original walkthrough (archived at
[docs/source/module4-dayn-template.md](../source/module4-dayn-template.md))
referred to a `PnP-Template-J2` editor screenshot inside the Composite
project-creation step; the project actually being created in that step is
named `DayN-Templates-J2`. The naming has been clarified throughout this
document.
