---
title: "Templates — PnP vs DayN considerations"
engine: shared
kind: docs
topic: considerations
path: docs/templates/considerations.md
---

# PnP vs DayN — Authoring considerations

Universal rules that apply before you write a single line of template code.
These constraints come directly from how Catalyst Center sequences PnP claim
and provisioning, and they govern every decision below in
[pnp-template-design.md](pnp-template-design.md) and
[dayn-template-design.md](dayn-template-design.md).

## 1. PnP vs DayN — which to use

Cisco recommends **DayN templates for most configurations**, reserving **PnP
templates for the minimum needed to establish a stable management
connection** with Catalyst Center.

| Aspect | PnP (Onboarding) | DayN |
|--------|------------------|------|
| Project | `Onboarding Configuration` (fixed) | Any user project |
| Device in Inventory? | No — claim has not completed | Yes |
| System variables (`__device`, `__interface`, …) | **Unavailable** | Available |
| Bind variables (inventory-sourced) | **Unavailable** | Available |
| Delivery | Single **flat file** over HTTP/HTTPS | Line-by-line CLI |
| Compliance scan | **Not tracked** | Tracked |
| Re-runnable for ongoing change? | No — single-shot at claim | Yes |
| Composite templates allowed? | No | Yes |

Because PnP cannot reach the inventory or system variables, all dynamic
values come from form fields filled in by the operator during the claim
workflow (one row per device, optionally bulk-loaded from CSV).

## 2. Inventory database is unreachable before claim

During PnP the device is not yet a managed entity. Anything that would have
required a lookup against `__device`, `__interface`, site hierarchy, or
network-settings bindings must be supplied through **template variables**
and entered manually (or via CSV) at claim time.

This is the single biggest reason to keep PnP templates small: every line you
add becomes another field the operator has to fill in, and there is no
inventory data to default it from.

## 3. Mandatory auto-config deployed by Catalyst Center at claim

Catalyst Center deploys the following CLI **automatically** during PnP claim
on top of whatever your PnP template renders. It cannot be disabled. Do not
duplicate any of it in your PnP template — overlap creates noise and risks
ordering conflicts.

<details>
<summary>Expand auto-deployed CLI</summary>

```text
archive
 log config
  logging enable
  logging size 500
  hidekeys
  !
 !
!
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
service sequence-numbers
!
! Disable external HTTP(S) access
! Disable external Telnet access
! Enable external SSHv2 access
!
no ip http server
!
no ip http secure-server
!
crypto key generate rsa label dnac-sda modulus 2048
ip ssh version 2
!
ip scp server enable
!
line vty 0 15
 ! maybe redundant
 login xxxxxx
 transport input ssh
 ! maybe redundant
 transport preferred none
!
! Set VTP mode to transparent (no auto VLAN propagation)
! Set STP mode to Rapid PVST+ (prefer for non-Fabric compatibility)
! Enable extended STP system ID
vtp mode transparent
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
no udld enable
!
errdisable recovery cause all
errdisable recovery interval 300
!
snmp-server xxxxxxxx
!
username xxxxxx
enable algorithm-type scrypt secret xxxxxxxx
!
hostname Switch
ip domain name base2hq.com
ip name-server 10.10.0.250
```

</details>

Because the auto-block contains `vtp mode transparent`, `spanning-tree mode
rapid-pvst`, default usernames, and SSH/SCP enablement, your PnP template
should **not** repeat these. It should only carry items the auto-block does
not provide: hostname (the auto-block sets a placeholder `Switch`), mgmt
VLAN/SVI, default gateway, source-interface bindings, system MTU,
netconf-yang.

## 4. Compliance is not tracked for PnP templates

Because the device is not yet in inventory at claim time, **compliance
scanning does not cover anything pushed via a PnP template**. Two
consequences:

1. Use PnP only for the bare minimum needed to bring up management
   connectivity — anything you put here is invisible to drift detection.
2. Put everything else in DayN, where it *will* be compliance-tracked and
   re-pushable.

## 5. Design Settings vs Template configuration — no overlap

Catalyst Center's **Design** app already emits CLI for AAA, NTP, syslog,
SNMP, DNS, banners, and similar settings when configured through the UI.

> If you populate the UI with settings, those parameters should **not** be
> in your templates as they will **conflict**, and the deployment through
> provisioning will **fail**.

Guidance:

- Drive as much configuration as possible from **Design Settings** (Network
  Settings, Device Credentials, AAA/RADIUS, Telemetry).
- Keep templates focused on configuration that **changes frequently** or
  that Design cannot express (custom VLAN layouts, ACLs, EEM applets,
  per-port macros, banners with embedded variables).
- Test with a single device first to confirm exactly which CLI Design pushes
  before finalizing a DayN template.

## 6. Jinja2 vs Velocity — choose Jinja2

Recommendation: **use Jinja2** for new templates.

- Jinja2 is the de-facto template language across Ansible, Salt, OpenStack,
  Flask, etc. Skills transfer.
- Logic constructs follow **Python** semantics — `if`, `for`, dictionaries,
  list comprehensions, macros.
- Composite templates can still mix Jinja2 and Velocity, so legacy Velocity
  modules remain usable without rewrite.

Velocity is supported and remains correct for existing modules. New
authoring should default to Jinja2 unless the team has an existing Velocity
codebase. See [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md)
and [rules/velocity/constraints.md](../../rules/velocity/constraints.md) for
engine-specific syntax rules.

## 7. Greenfield vs Brownfield scope

The methodology in this folder targets **Greenfield** (factory-default
device, brought up via PnP). Brownfield (device already in production,
imported via discovery) reuses the **DayN** parts of this guide unchanged,
but skips the PnP-template and claim steps entirely.
