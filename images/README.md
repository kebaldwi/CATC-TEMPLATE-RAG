---
title: "Images — reference screenshots manifest"
engine: none
kind: navigation
topic: images-manifest
path: images/README.md
---

# Images — reference screenshots

Catalyst Center Template Editor screenshots referenced from the docs.
Organised by the consuming document so an AI agent can resolve
"what does the screenshot for X look like?" by topic, not by filename.

## Layout

```text
images/
├── templates/         ← consumed by docs/templates.md
├── variables/         ← consumed by docs/variables.md
├── system-variables/  ← consumed by docs/system-variables.md
├── eem/               ← consumed by docs/eem.md
├── pnp-claim/         ← consumed by docs/templates/pnp-claim-workflow.md
├── dayn-provisioning/ ← consumed by docs/templates/dayn-provisioning.md
└── day0dayn/          ← extra Day-0/Day-N screenshots (unreferenced reserve)
```

## Expected files

Each subfolder lists every filename the corresponding doc references.
Drop the matching `.png` (or `.jpg`) into the folder and links resolve
automatically.

### `images/templates/` — referenced by [docs/templates.md](../docs/templates.md)

- `AddNewTemplate.png`
- `AddTemplate.png`
- `CreateTemplate.png`
- `EditTemplateWindow.png`
- `GoToTemplateEditor.png`
- `SelectDeviceTemplate.png`
- `SelectOSTemplate.png`
- `SelectTemplateProject.png`

### `images/variables/` — referenced by [docs/variables.md](../docs/variables.md)

- `Input-Form.png`
- `TemplateEditor.png`
- `variable-basic.png`
- `variable-bind-device.png`
- `variable-bind-inventory.png`
- `variable-bind-platformid.png`
- `variable-binding.png`
- `variable-data-entry.png`
- `variable-definition.png`
- `variable-instructionaltext.png`
- `variable-selections.png`
- `variable-singlechoice.png`

### `images/system-variables/` — referenced by [docs/system-variables.md](../docs/system-variables.md)

- `Input_Form_SystemVariables.png`
- `button.png`
- `calculator_icon.png`
- `create_simulation.png`
- `interfaces_options.png`
- `play_icon.png`
- `set_snmp_location.png`
- `set_vlan.png`
- `set_vlan_2.png`
- `simulation_input.png`
- `simulation_output.png`
- `template_system_interfaces_1.png`
- `template_system_interfaces_2.png`
- `template_system_interfaces_3.png`
- `template_system_interfaces_4.png`
- `template_system_main.png`

### `images/eem/` — referenced by [docs/eem.md](../docs/eem.md)

- `EEMDetectors.png`
- `EEMOperation.png`

### `images/pnp-claim/` — referenced by [docs/templates/pnp-claim-workflow.md](../docs/templates/pnp-claim-workflow.md)

Sequential screenshots of the PnP claim workflow on a Catalyst 9300.

- `c9300-1-claim-1.png` — Plug and Play list, Actions → Claim
- `c9300-1-claim-2.png` — Section 1: Assign hierarchy link
- `c9300-1-claim-3.png` — Section 1: Site selection dialog
- `c9300-1-claim-4.png` — Section 1: Site assigned
- `c9300-1-claim-5.png` — Section 2: Template & image review
- `c9300-1-claim-6.png` — Section 3: Variable entry form
- `c9300-1-claim-7.png` — Section 4: Review + Claim
- `c9300-1-claim-8.png` — Onboarding / Provisioned transition
- `c9300-1-claim-9.png` — Device in Inventory after claim

### `images/dayn-provisioning/` — referenced by [docs/templates/dayn-provisioning.md](../docs/templates/dayn-provisioning.md)

Sequential screenshots of the DayN Provision Device workflow on a
Catalyst 9300 already in Inventory.

- `c9300-1-provision-1.png` — Tag the device (ACCESS / DISTRO)
- `c9300-1-provision-2.png` — Inventory after tagging
- `c9300-1-provision-3.png` — Actions → Provision → Provision Device
- `c9300-1-provision-4.png` — Section 1: site already assigned
- `c9300-1-provision-5.png` — Section 2: device selection, checkboxes
- `c9300-1-provision-6.png` — Section 3: rendered CLI review
- `c9300-1-provision-7.png` — Section 4: Now / schedule + Apply
- `c9300-1-provision-8.png` — Inventory with Provisioning focus

### `images/day0dayn/` — additional Day-0 / Day-N screenshots (unreferenced reserve)

Extra DNAC inventory, Network Profile, and provisioning screenshots not
currently cited by any doc. Held here for future references when the
concept docs under `docs/templates/` are expanded.

- `CATC-Inventory-TAG-9300-1-ACCESS.png`
- `CATC-Inventory-TAG-9300-2-DISTRO.png`
- `CATC-Inventory-TAG-9300s-RESULT.png`
- `DNAC-InventoryProvision.png`
- `DNAC-NavigateInventory.png`
- `DNAC-NavigateProfile.png`
- `DNAC-ProfileCloseAddTemplate.png`
- `DNAC-ProfileDayN-ACCESS.png`
- `DNAC-ProfileDayN-DISTRO.png`
- `DNAC-ProfileDayNAdd.png`
- `DNAC-ProfileEdit.png`
- `DNAC-ProfileSuccess.png`
- `DNAC-ProfileSuccess-1.png`
- `DNAC-ProfileSuccess-2.png`
- `DNAC-ProvisionAdvConfig-1.png`
- `DNAC-ProvisionAdvConfig-2.png`
- `DNAC-ProvisionApply.png`
- `DNAC-ProvisionBegin.png`
- `DNAC-ProvisionDeploy.png`
- `DNAC-ProvisionScheduled.png`
- `DNAC-ProvisionScheduled-2.png`
- `DNAC-ProvisionSite.png`
- `DNAC-ProvisionTask.png`
- `DNAC-ProvisionTasking.png`

## How an AI should use this folder

1. If a user asks about a UI element shown in one of the consuming docs,
   cite the corresponding image path as evidence (e.g. 
   `images/templates/CreateTemplate.png`).
2. If the image file is missing on disk, fall back to the surrounding
   prose in the consuming doc and tell the user the screenshot is not
   yet checked in.
3. Do not invent image filenames not listed above.

