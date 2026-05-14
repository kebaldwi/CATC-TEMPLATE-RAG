---
title: "Templates — Build methodology and decision flow"
engine: shared
kind: docs
topic: build-methodology
path: docs/templates/build-methodology.md
---

# Build methodology and decision flow

Cross-cutting authoring principles that apply across PnP and DayN. The
detailed mechanics live in [pnp-template-design.md](pnp-template-design.md)
and [dayn-template-design.md](dayn-template-design.md); this page
summarizes the philosophy and gives a decision flow for engine and
template-type choices.

## 1. Five principles

### Design first, template only the rest

Catalyst Center's **Design** app is authoritative for inventory-wide
settings (Network Settings, AAA/RADIUS, NTP, syslog, SNMP, telemetry,
banners, credentials). Use Design wherever possible. Templates should
carry only:

- Configuration the Design app cannot express (custom ACLs, EEM applets,
  per-port macros, complex VLAN layouts, vendor-specific quirks).
- Configuration that **changes frequently** at a cadence different from
  Design-Settings change cycles.
- Per-device or per-role customizations (gated by TAGs — see
  [dayn-provisioning.md §2](dayn-provisioning.md#2-differentiating-template-variants-by-device-role-with-tags)).

If you find yourself templating something Design already pushes, the
deployment **will fail** at provisioning time. See
[considerations.md §5](considerations.md#5-design-settings-vs-template-configuration--no-overlap).

### Don't Repeat Yourself (DRY)

Same goal as in software: write each construct once, parameterize, reuse.

- Variables and conditionals replace literals.
- Macros replace repeated CLI blocks.
- Dictionaries / lists drive looped CLI emission.
- Regular templates encapsulate one feature.
- Composite templates assemble Regulars into device-role bundles.

See [dayn-template-design.md §5](dayn-template-design.md#5-regular-template--intconfig-jinja2--dictionaries-macros-__interface)
for the canonical dictionary + macro + `__interface` pattern.

### Iterative build with simulation between steps

Don't try to write the final template in one pass. The recommended loop:

1. **Static CLI** — paste a known-good config exactly as it appears on a
   reference device. Save and commit.
2. **Variabilize** — replace literals with `{{Variable}}` placeholders.
   Save, commit, simulate.
3. **Add conditional logic** — `{% if %}` / `{% for %}` around the
   variable parts that should change behaviour. Simulate after each
   conditional.
4. **Add edge-case guards** — the `!{{Var}}` first-use workaround,
   default-value clauses, VLAN-1 exception, single-uplink vs port-channel
   branches. Simulate combinatorially (see
   [pnp-template-design.md §5](pnp-template-design.md#5-simulation-matrix)).
5. **Form fields** — set Field Name, Type, Required, Default on every
   variable so the claim/provision UI is self-documenting.
6. **Review Form** + **Simulation** one more time before committing.

### Simulate before Claim

The **Simulation** tab renders the template against a chosen device model
without pushing anything. Use it to:

- Catch unresolved variables (Catalyst Center Jinja2 renders undefined
  references as empty strings — silently — see
  [rules/jinja2/cc-vs-ansible.md](../../rules/jinja2/cc-vs-ansible.md)).
- Catch broken conditional logic.
- See exactly what CLI the operator will be pushing at claim time.
- Verify the `!{{Var}}` workaround did not leave a stray `!{{...}}` in
  the rendered output.

Treat **simulation green** as a precondition for attaching the template
to a Network Profile.

### Idempotency

DayN templates must be safe to re-push. See
[dayn-template-design.md §7](dayn-template-design.md#7-idempotency-for-re-pushable-dayn)
for the rules. PnP templates are single-shot at claim and do not need
to be idempotent — but they should still avoid pushing the same line
multiple times within a single render.

## 2. Decision flow

### Q1 — Is this Day 0 (Greenfield) or Day N?

```
Device is factory-default and about to be claimed?
  └── YES → PnP Onboarding template
  └── NO  → DayN template
              └── Device imported via discovery (Brownfield)?
                    └── Skip PnP entirely; design from existing config
                        using the same analyze → discount → modularize
                        method as Greenfield DayN.
```

### Q2 — Regular or Composite?

```
Single, focused feature (one configuration concern)?
  └── YES → Regular template
            └── Mandatory for PnP (Composites are DayN-only).
Multiple Regulars need to ship together on one device role?
  └── YES → Composite template (DayN only)
            └── Mix Jinja2 + Velocity Regulars freely.
            └── Order matters — system/AAA before features that depend on them.
```

### Q3 — Jinja2 or Velocity?

```
Greenfield / new template?
  └── Default: Jinja2.
      Reasons: Python-flavoured logic, industry-standard syntax,
      __interface / __device / dictionaries / macros / list-of-dicts
      patterns all idiomatic.

Existing Velocity codebase or strong team preference?
  └── Velocity is fine; both engines have full Catalyst Center support.
      Constraints to know:
        - #parse and #include are disabled (no shared-snippet pulls).
        - Logic is slightly less expressive than Jinja2 for
          list-of-dicts patterns.

Mixed environment?
  └── Composite templates can carry both. New work goes in Jinja2,
      legacy Velocity Regulars stay as-is and are reused via Composite.
```

See [rules/jinja2/constraints.md](../../rules/jinja2/constraints.md) and
[rules/velocity/constraints.md](../../rules/velocity/constraints.md) for
engine-specific syntax rules, and the per-engine basics docs at
[docs/jinja2/basics.md](../jinja2/basics.md) /
[docs/velocity/basics.md](../velocity/basics.md).

### Q4 — Where does this CLI belong: Design, PnP, or DayN?

```
Inventory-wide setting expressible in the Design app (NTP, syslog,
SNMP, AAA, banners-without-variables, credentials, telemetry)?
  └── YES → Design Settings. Do NOT also put it in any template.

Required to establish stable management connectivity at claim?
  └── YES → PnP Onboarding template. Minimum scope only:
            hostname, mgmt VLAN/SVI, default gateway, source-interface
            bindings, system MTU (if non-default), netconf-yang.
            Compliance does NOT cover this.

Everything else?
  └── DayN. Compliance-tracked, re-pushable, full system+bind variables
      available, Composite-able.
```

## 3. What to put in PnP (cheat-sheet)

Include:

- Hostname
- VTP domain/mode
- Mgmt VLAN database entry + VLAN-1 shutdown (if mgmt VLAN != 1)
- Mgmt SVI: IP, mask, `no ip redirects`, `no ip proxy-arp`
- Default gateway (L2 case) or routing-protocol bootstrap (L3 case)
- System MTU (only if non-default)
- Uplink trunk + optional LACP port-channel
- Source-interface bindings for: `ip http client`, `ip ssh`, `ip ftp`,
  `ip tftp`, `ip radius`, `ip domain lookup`, `logging`,
  `snmp-server trap-source`, `ntp source`
- `netconf-yang`

Exclude:

- Anything in the auto-deployed PnP block
  ([considerations.md §3](considerations.md#3-mandatory-auto-config-deployed-by-catalyst-center-at-claim))
- Anything in Design Settings
- Anything that needs `__device` / `__interface` / bind variables
  (those are empty here)
- AAA, SNMP, NTP, syslog, banner detail — all DayN/Design territory

## 4. What to put in DayN (cheat-sheet)

Include:

- The residue from
  [dayn-template-design.md §2 Pass C](dayn-template-design.md#pass-c--the-residue-is-what-the-template-owns).
- Per-role variants gated by Device TAGs.
- Anything that benefits from `__interface` iteration
  (per-port macros, dynamic uplink detection).
- Anything that benefits from list-of-dicts iteration
  (VLAN database, ACL entries, route-map entries).
- Banners and other multi-line CLI wrapped in `<MLTCMD>`.

Exclude:

- Anything in Design Settings.
- Anything PnP already set on this device.
- Hard-coded device-specific values that should come from inventory
  bindings instead.

## 5. Further reading

- Concept overviews: [docs/templates.md](../templates.md),
  [docs/variables.md](../variables.md),
  [docs/system-variables.md](../system-variables.md),
  [docs/eem.md](../eem.md)
- Authoring prompts:
  [prompts/authoring/](../../prompts/authoring/)
- Review prompts:
  [prompts/review/validate-jinja2.md](../../prompts/review/validate-jinja2.md),
  [prompts/review/validate-velocity.md](../../prompts/review/validate-velocity.md)
- Debug prompts:
  [prompts/debug/](../../prompts/debug/)
- Pattern recipes:
  [prompts/patterns/](../../prompts/patterns/)
- Working examples:
  [examples/jinja2/](../../examples/jinja2/),
  [examples/velocity/](../../examples/velocity/)
