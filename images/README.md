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
└── eem/               ← consumed by docs/eem.md
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

## How an AI should use this folder

1. If a user asks about a UI element shown in one of the consuming docs,
   cite the corresponding image path as evidence (e.g. 
   `images/templates/CreateTemplate.png`).
2. If the image file is missing on disk, fall back to the surrounding
   prose in the consuming doc and tell the user the screenshot is not
   yet checked in.
3. Do not invent image filenames not listed above.

