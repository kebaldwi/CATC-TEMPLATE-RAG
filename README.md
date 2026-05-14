---
title: "Readme"
engine: shared
kind: meta
topic: README
path: README.md
---

# catc-template-rag

Reference / RAG (Retrieval-Augmented Generation) corpus for **Cisco Catalyst
Center** template authoring. The repository captures everything an AI
assistant (or a human engineer) needs to write, review, and debug Catalyst
Center templates in either of the two supported engines — **Apache Velocity**
(`.vm`) and **Jinja2** (`.j2`).

The goal is that a chat bot pointed at this repo can answer "how do I do X in
a Catalyst Center template?" without inventing syntax that the platform does
not actually support.

---

## Quick map for AI assistants

When a user asks a question, decide which engine and which concern they are
asking about, then load the matching files.

```text
                    ┌──────────────────────────────────────────────────┐
                    │              Catalyst Center templates           │
                    └──────────────────────────────────────────────────┘
                                          │
                ┌─────────────────────────┴─────────────────────────┐
                │                                                   │
        ┌───────▼────────┐                                  ┌───────▼────────┐
        │   Velocity     │                                  │     Jinja2     │
        │     (.vm)      │                                  │      (.j2)     │
        └───────┬────────┘                                  └───────┬────────┘
                │                                                   │
   rules/velocity/  guardrails/velocity/             rules/jinja2/  guardrails/jinja2/
   docs/velocity/                                    docs/jinja2/
   examples/velocity/                                examples/jinja2/

                       shared cross-engine concerns
                       ─────────────────────────────
                       docs/eem.md
                       docs/templates.md
                       docs/variables.md
                       docs/system-variables.md
```

### Routing table — "If the user asks about …, read these"

| User intent / question shape                                     | Load first                                                                                       | Then load if relevant                                                                          |
|------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| "How do I write a Velocity template?"                            | [docs/velocity/basics.md](docs/velocity/basics.md)         | [docs/velocity/advanced.md](docs/velocity/advanced.md), [rules/velocity/constraints.md](rules/velocity/constraints.md) |
| "How do I write a Jinja2 template?"                              | [docs/jinja2/basics.md](docs/jinja2/basics.md)                 | [docs/jinja2/advanced.md](docs/jinja2/advanced.md), [rules/jinja2/constraints.md](rules/jinja2/constraints.md) |
| "Is this template valid in CC?" / "Why does this fail?"          | [guardrails/jinja2/invalid-syntax-patterns.md](guardrails/jinja2/invalid-syntax-patterns.md) **or** [guardrails/velocity/invalid-syntax-patterns.md](guardrails/velocity/invalid-syntax-patterns.md) | [guardrails/jinja2/common-mistakes.md](guardrails/jinja2/common-mistakes.md) **or** [guardrails/velocity/common-mistakes.md](guardrails/velocity/common-mistakes.md) |
| "What can CC NOT do in a template?"                              | [guardrails/jinja2/what-cc-cannot-do.md](guardrails/jinja2/what-cc-cannot-do.md) **or** [guardrails/velocity/what-cc-cannot-do.md](guardrails/velocity/what-cc-cannot-do.md) |                                                                                                |
| Variables: user-input vs bind vs system vs local                 | [rules/jinja2/variable-types.md](rules/jinja2/variable-types.md) **or** [rules/velocity/variable-types.md](rules/velocity/variable-types.md) | [docs/variables.md](docs/variables.md), [docs/system-variables.md](docs/system-variables.md) |
| Dictionaries / list-of-dicts (VLAN-style data)                   | [docs/jinja2/dictionaries.md](docs/jinja2/dictionaries.md)     | [examples/jinja2/VLAN-Configuration.j2](examples/jinja2/VLAN-Configuration.j2) |
| `__device`, `__interface`, `__networkSettings`, `__credentials`  | [docs/system-variables.md](docs/system-variables.md) | engine-specific `variable-types.md`                                                            |
| `#MODE_ENABLE`, `<MLTCMD>`, `! @ start-ignore-compliance`        | [rules/jinja2/constraints.md](rules/jinja2/constraints.md) **or** [rules/velocity/constraints.md](rules/velocity/constraints.md) |                                                                                                |
| Composite / DayN templates                                       | [docs/templates.md](docs/templates.md)     |                                                                                                |
| EEM (Embedded Event Manager) scripting from a template           | [docs/eem.md](docs/eem.md)                             | [examples/jinja2/AutoNaming-EEM-Scripting.j2](examples/jinja2/AutoNaming-EEM-Scripting.j2) |
| PnP vs DayN binding / system variable availability               | [guardrails/jinja2/what-cc-cannot-do.md](guardrails/jinja2/what-cc-cannot-do.md)                 | [docs/templates.md](docs/templates.md)   |
| "Will this template render the same way in Ansible?"             | [rules/jinja2/cc-vs-ansible.md](rules/jinja2/cc-vs-ansible.md)                   |                                                                                                |
| Worked end-to-end examples (real config)                         | `examples/jinja2/*.j2` and `examples/velocity/*.vm`                            |                                                                                                |

---

## Repository layout

```text
catc-template-rag/
├── README.md                    ← human + AI entry point (this file)
├── INDEX.md                     ← machine-readable manifest of every doc
├── AGENTS.md                    ← instructions for autonomous coding agents
├── LICENSE
│
├── prompts/                     ← curated chatbot starters + agent recipes
│   ├── authoring/               ← "write me a new …"
│   ├── review/                  ← "is this template OK?"
│   ├── debug/                   ← "why does this fail?"
│   └── patterns/                ← common building blocks (VLAN dicts, EEM, …)
│
├── images/                      ← Template Editor reference screenshots
│   ├── templates/               ← consumed by docs/templates.md
│   ├── variables/               ← consumed by docs/variables.md
│   ├── system-variables/        ← consumed by docs/system-variables.md
│   └── eem/                     ← consumed by docs/eem.md
│
├── rules/                       ← AUTHORITATIVE syntax/semantic rule book
│   ├── jinja2/
│   │   ├── constraints.md               ← every legal directive + CC deviations
│   │   ├── variable-types.md            ← variable taxonomy for Jinja2
│   │   └── cc-vs-ansible.md             ← CC Jinja2 vs Ansible Jinja2 deltas
│   └── velocity/
│       ├── constraints.md               ← every legal directive + CC deviations
│       └── variable-types.md            ← variable taxonomy for Velocity
│
├── guardrails/                  ← WHAT CAN GO WRONG — bad/good pattern pairs
│   ├── jinja2/
│   │   ├── common-mistakes.md           ← parses fine, behaves wrong
│   │   ├── invalid-syntax-patterns.md   ← parse errors
│   │   └── what-cc-cannot-do.md         ← platform-level limits
│   └── velocity/
│       ├── common-mistakes.md
│       ├── invalid-syntax-patterns.md
│       └── what-cc-cannot-do.md
│
├── docs/                        ← NARRATIVE tutorials / explainers
│   ├── eem.md                           ← Embedded Event Manager from a template
│   ├── templates.md                     ← PnP, DayN, Composite, Regular
│   ├── variables.md                     ← user-input / bind / local
│   ├── system-variables.md              ← __device, __interface, …
│   ├── jinja2/
│   │   ├── basics.md                    ← Jinja2 basics in CC
│   │   ├── advanced.md                  ← advanced patterns
│   │   └── dictionaries.md              ← list-of-dicts (VLAN-style)
│   └── velocity/
│       ├── basics.md                    ← Velocity basics in CC
│       └── advanced.md                  ← advanced patterns
│
└── examples/                    ← REAL-WORLD CC templates (canonical examples)
    ├── jinja2/                  ← 85+ .j2 templates
    └── velocity/                ← 14  .vm templates
```

### Folder-by-folder semantics

* **`rules/`** — Authoritative, normative reference. Source of truth for
  "is X legal?" / "what does Y mean?". Cite specific section numbers when
  answering.
* **`guardrails/`** — Catalogue of failure modes. Use when validating
  user-supplied code, or when the user reports an error. Each item is a
  bad/good pair that can be quoted verbatim.
* **`docs/`** — Narrative, tutorial-style explainers.
  Use for "how do I…" or conceptual orientation.
* **`examples/`** — Real, in-production templates from the
  reference design. Use to ground concrete examples, copy idioms, and
  validate that a proposed pattern actually appears in the wild.

---

## How to answer questions about Catalyst Center templates

A reliable recipe for an AI assistant working with this repo:

1. **Detect the engine** from filenames or syntax cues:
   * `.vm` files / `#set` / `$var` / `#foreach` → **Velocity**.
   * `.j2` files / `{% %}` / `{{ }}` / `{# #}` / `{% set %}` → **Jinja2**.
   * If unclear, ask.
2. **Classify the question**:
   * *Reference / "is this valid?"* → load `rules/<engine>/`.
   * *Debugging / "why does this fail?"* → load `guardrails/<engine>/`.
   * *"How do I…"* → load `docs/<engine>/` (or top-level `docs/*.md` for shared concerns).
   * *"Show me an example"* → search `examples/<engine>/`.
3. **Always check platform limits** before proposing a solution
   ([guardrails/jinja2/what-cc-cannot-do.md](guardrails/jinja2/what-cc-cannot-do.md) /
   [guardrails/velocity/what-cc-cannot-do.md](guardrails/velocity/what-cc-cannot-do.md)).
4. **Be explicit about PnP vs DayN**:
   * PnP onboarding templates **cannot** use bind variables or `__*` system
     variables (the device is not yet in Inventory).
   * Composite templates (`*-Composite.yml`) are DayN-only.
5. **Quote source files when citing rules.** Prefer specific section
   anchors (e.g., `rules/jinja2/constraints.md §8.5`) over vague
   references.

---

## Key facts the assistant should keep in working memory

These are the deltas most likely to bite a user writing a template by analogy
with Python, Ansible, or generic Velocity.

### Both engines (Catalyst Center–specific)

* `!` is **not** a comment to the template engine — it is forwarded into IOS
  config. Use `{# … #}` (Jinja2) or `## …` / `#* … *#` (Velocity).
* `#MODE_ENABLE` … `#MODE_END_ENABLE` wraps privileged-EXEC commands and is
  processed by the Catalyst Center provisioner, not the template engine.
* `<MLTCMD> … </MLTCMD>` bundles a multi-line CLI block (banners, certs,
  key-chains).
* `! @ start-ignore-compliance` / `! @ end-ignore-compliance` excludes
  enclosed CLI from compliance scans.
* PnP onboarding templates have **no** access to bind variables or `__*`
  system variables.
* Composite templates are valid only in DayN workflows.

### Velocity specifics (in CC)

* `#parse` and `#include` are **disabled** — there is no cross-template
  composition.
* `#stop` aborts but emits any output produced *before* it ran.
* Macros are defined with `#macro` and must precede their first call.
* Strings interpolate references inside double quotes (`"$ref"`) but not
  inside single quotes.

### Jinja2 specifics (in CC)

* `{% include %}`, `{% extends %}`, `{% import %}` **are** supported, and
  resolve against the Template Editor project tree.
* `{% do list.append(x) %}` works — the `do` extension is enabled.
* CC accepts JS-style operators `&&`, `||`, `!` **in addition to**
  `and`, `or`, `not`. Prefer the word form for portability.
* The `split` filter (`| split(",")`) is CC-provided; standard Jinja2 only
  has the `.split()` method.
* `loopcontrols` is **not** enabled — `{% break %}` and `{% continue %}`
  are unavailable.
* Undefined references render as empty strings (non-strict mode), unlike
  Ansible's default strict mode.
* String concatenation uses `~`, not `+`. `+` on strings is arithmetic.

### Variable taxonomy (both engines)

| Category    | Origin                                                      | Velocity reference          | Jinja2 reference                  |
|-------------|-------------------------------------------------------------|-----------------------------|-----------------------------------|
| User-input  | Declared in template, filled in via Input Form              | `$Hostname`                 | `{{ Hostname }}`                  |
| Bind        | User-input bound to Inventory / Settings / Profile / Cloud  | `$ProductID` (single-select)| `{{ ProductID }}` (single-select) |
| System      | Built-in CC objects                                         | `$__device.platformId`      | `{{ __device.platformId }}`       |
| Local       | Computed in template                                        | `#set( $x = … )`            | `{% set x = … %}`                 |

---

## Conventions used in this repo

* Markdown files are the human + bot interface; templates (`.vm`, `.j2`)
  are the executable artefacts.
* Code blocks use the language hint `vtl` for Velocity, `j2` for Jinja2.
* Cross-references are markdown links so an assistant can follow them
  programmatically.
* "Bad / Good" pairs appear in `guardrails/` files — when validating user
  code, match the user's snippet against the "Bad" column and quote the
  "Good" form.
* The `examples/` directory is **read-only reference**; do not
  modify those files when answering questions — copy their patterns into
  new content instead.

---

## Source materials

This corpus distills and reorganises content from:

* Cisco Catalyst Center template editor documentation:
  <https://explore.cisco.com/dnac-use-cases/apache-velocity>
* Apache Velocity 2.3 user guide:
  <https://velocity.apache.org/engine/2.3/user-guide.html>
* Jinja2 (Pallets) 3.0.x template designer documentation:
  <https://jinja.palletsprojects.com/en/3.0.x/templates/>
* DevNet `kebaldwi/DNAC-TEMPLATES` reference design and its in-production
  `.j2` / `.vm` template library (mirrored under `examples/`).
