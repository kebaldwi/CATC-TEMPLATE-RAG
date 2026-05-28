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

## How to use this repo with an AI assistant/Chat bot

This repo is designed to be a knowledge source for an AI assistant that answers questions about Catalyst Center template authoring. The assistant should use the routing table above to determine which files to load based on the user's question, and then cite specific sections of those files when providing an answer. When relevant, the assistant should quote "bad / good" pairs from the `guardrails/` files to illustrate common mistakes and their corrections. Additionally, when the user asks for examples, the assistant should search the `examples/` directory for relevant patterns and quote them to provide concrete illustrations of how to use certain features or avoid certain pitfalls in Catalyst Center templates.

When answering questions, the assistant should always check for platform-specific limitations by consulting the `what-cc-cannot-do.md` files in the `guardrails/` directory before proposing a solution. This ensures that the advice given is not only syntactically correct but also compatible with the capabilities of Catalyst Center.

Using Monica, AutoGPT, LangSmith, or a similar agent framework, you can set up an agent with access to this repository as its knowledge base. The agent can be programmed to follow the routing logic outlined above, allowing it to provide accurate and contextually relevant answers to user questions about Catalyst Center template authoring.

Generally: 

1. Point the assistant at this repo as a knowledge source.
2. When the user asks a question, classify it according to the routing table
   above, load the relevant files, and answer citing specific sections.
3. When relevant, quote "bad / good" pairs from the `guardrails/` files.
4. When the user asks for examples, search the `examples/` directory for
   relevant patterns and quote them.

Use the [`AGENTS.md`](.AGENTS.md) file for specific agent recipes and prompt templates for common question types.

Create a new issue if you want help writing prompts or agent code to work with this repo!

Use the [`INDEX.md`](.INDEX.md) file as a manifest of all content in the repo, with links to each section.

Prompts in the `prompts/` directory are curated starters for common question types — feel free to use or adapt them when building your assistant.

A typical workflow for a user question might look like some of these examples:

#### How do I write a Velocity template that configures VLANs using a list of dictionaries?

1. User: "How do I write a Velocity template that configures VLANs using a list of dictionaries?"
1. Assistant:
   - Detects "Velocity" and "how do I write" → loads `docs/velocity/basics.md` and `docs/velocity/advanced.md`.
   - Finds relevant sections on loops, variables, and dictionaries.
   - Cites specific sections (e.g., "See `docs/velocity/advanced.md §3.2` for loops and `§4.1` for variable usage in Velocity templates").
   - Optionally, searches `examples/velocity/` for VLAN-related templates and quotes relevant snippets.

#### Is this template valid? (provides a snippet)

1. User: "Is this template valid?" (provides a snippet)
1. Assistant:
   - Detects "Velocity" and "is this valid" → loads `guardrails/velocity/invalid-syntax-patterns.md` and `guardrails/velocity/common-mistakes.md`.
   - Matches the user's snippet against "Bad" patterns and checks for any matches.
   - If a match is found, quotes the corresponding "Good" pattern and explains the issue.

#### What can I NOT do in a Velocity template for Catalyst Center?

1. User: "What can I NOT do in a Velocity template for Catalyst Center?"
1. Assistant:
   - Loads `guardrails/velocity/what-cc-cannot-do.md`.
   - Summarizes key limitations (e.g., "Velocity templates in Catalyst Center cannot use `#parse` or `#include` for cross-template composition, and they do not support macros defined after their first call. Additionally, `!` is not a comment and will be forwarded into IOS config, so you should use `##` or `#*` for comments instead.").

#### Show me an example of a Jinja2 template that uses system variables

1. User: "Show me an example of a Jinja2 template that uses system variables."
1. Assistant:
   - Detects "Jinja2" and "example" → searches `examples/jinja2/` for templates that reference `__*` system variables.
   - Finds and quotes relevant snippets (e.g., "Here's an example from `examples/jinja2/Device-Configuration.j2` that uses `{{ __device.platformId }}` to access the device's platform ID: `{% set platform = __device.platformId %} ...`").

#### Can I use Ansible-style Jinja2 syntax in Catalyst Center templates?

1. User: "Can I use Ansible-style Jinja2 syntax in Catalyst Center templates?"
1. Assistant:
    - Loads `rules/jinja2/cc-vs-ansible.md`.
    - Explains differences (e.g., "Catalyst Center's Jinja2 engine supports JS-style operators like `&&`, `||`, and `!` in addition to the standard `and`, `or`, `not`. However, it does not support the `loopcontrols` extension, so `{% break %}` and `{% continue %}` are not available. Also, string concatenation uses `~` instead of `+`, and `+` on strings is treated as arithmetic. Undefined references render as empty strings, which is different from Ansible's default strict mode where they would raise an error.").

#### What are the variable types available in Jinja2 templates for Catalyst Center?

1. User: "What are the variable types available in Jinja2 templates for Catalyst Center?"
1. Assistant:
    - Loads `rules/jinja2/variable-types.md`.
    - Explains the taxonomy of variables (e.g., "In Jinja2 templates for Catalyst Center, there are several categories of variables: User-input variables that are declared in the template and filled in via the Input Form (e.g., `{{ Hostname }}`), Bind variables that are user-input variables bound to Inventory, Settings, Profile, or Cloud (e.g., `{{ ProductID }}`), System variables that are built-in CC objects (e.g., `{{ __device.platformId }}`), and Local variables that are computed within the template (e.g., `{% set x = ... %}`). Each category has specific use cases and limitations, especially in PnP onboarding templates where bind and system variables are not available.").

#### Can I use `{% include %}` in a Jinja2 template for Catalyst Center?

1. User: "Can I use `{% include %}` in a Jinja2 template for Catalyst Center?"
1. Assistant:
    - Loads `rules/jinja2/constraints.md`.
    - Confirms that `{% include %}` is supported and explains how it resolves against the Template Editor project tree, allowing for modular template design.

#### What are some common mistakes to avoid in Jinja2 templates for Catalyst Center?

1. User: "What are some common mistakes to avoid in Velocity templates for Catalyst Center?"
1. Assistant:
    - Loads `guardrails/velocity/common-mistakes.md`.
    - Summarizes common pitfalls (e.g., "A common mistake in Velocity templates is using `#set` to define a variable after its first use, which will lead to an error since Velocity requires macros to be defined before they are called. Another mistake is treating `!` as a comment, which will actually be included in the output and can cause issues in the generated IOS config. Instead, use `##` or `#*` for comments. Additionally, remember that strings only interpolate references inside double quotes, so using single quotes will not work for variable interpolation.").

#### Can I use loops in Velocity templates for Catalyst Center?

1. User: "Can I use loops in Velocity templates for Catalyst Center?"
1. Assistant:
    - Loads `docs/velocity/advanced.md` and `rules/velocity/constraints.md`.
    - Explains that loops are supported using `#foreach`, but there are specific constraints to be aware of (e.g., "You can use `#foreach` to iterate over lists, but keep in mind that Velocity does not support the `loopcontrols` extension, so you cannot use `{% break %}` or `{% continue %}`. Also, be cautious with variable scoping within loops, as Velocity has specific rules about variable visibility. For more details, see `docs/velocity/advanced.md §3.2` and `rules/velocity/constraints.md §5.1`.").

#### What are the limitations of using `{% set %}` in Jinja2 templates for Catalyst Center?

1. User: "What are the limitations of using `{% set %}` in Jinja2 templates for Catalyst Center?"
1. Assistant:
    - Loads `rules/jinja2/constraints.md`.
    - Explains that while `{% set %}` is supported for defining local variables, it cannot be used to define variables that persist across different scopes in the way that bind or system variables do. Additionally, in PnP onboarding templates, you cannot use `{% set %}` to define variables that would be expected to interact with Inventory or Settings, since those are not available in that context. For more details, see `rules/jinja2/constraints.md §4.3` and `guardrails/jinja2/what-cc-cannot-do.md §2.1`.".

#### Can I use macros in Velocity templates for Catalyst Center?

1. User: "Can I use macros in Velocity templates for Catalyst Center?"
1. Assistant:
    - Loads `rules/velocity/constraints.md`.
    - Explains that macros are supported in Velocity templates, but they must be defined before their first call. This means you cannot have a macro that is called before it is defined in the template. For example, if you have `#macro myMacro() ... #end`, you must ensure that this definition appears in the template before any invocation of `#myMacro()`. This is a common source of errors, so be sure to structure your template accordingly. For more details, see `rules/velocity/constraints.md §6.2`.".

#### What are some examples of using system variables in Jinja2 templates for Catalyst Center?

1. User: "What are some examples of using system variables in Jinja2 templates for Catalyst Center?"
1. Assistant:
    - Loads `examples/jinja2/Device-Configuration.j2` and `docs/system-variables.md`.
    - Provides examples of using system variables (e.g., "In Jinja2 templates for Catalyst Center, you can access system variables that provide information about the device, interfaces, network settings, and credentials. For instance, you can use `{{ __device.platformId }}` to get the platform ID of the device, or `{{ __interface.name }}` to get the name of the current interface in a loop. Here's a snippet from `examples/jinja2/Device-Configuration.j2` that demonstrates this: `{% set platform = __device.platformId %} ...` where we set a local variable `platform` to the value of the system variable `__  device.platformId`. For a full list of system variables, see `docs/system-variables.md`.".

#### Can I use `{% break %}` in a Jinja2 template for Catalyst Center?

1. User: "Can I use `{% break %}` in a Jinja2 template for Catalyst Center?"
1. Assistant:
    - Loads `rules/jinja2/constraints.md`.
    - Explains that the `loopcontrols` extension, which provides `{% break %}` and `{% continue %}`, is not enabled in Catalyst Center's Jinja2 engine. This means you cannot use these directives to control loop execution. Instead, you will need to structure your loops and conditionals in a way that does not rely on breaking or continuing. For example, you can use conditional statements to skip certain iterations instead of using `{% continue %}`. For more details, see `rules/jinja2/constraints.md §7.4`.".

#### Can I use string concatenation with `+` in Jinja2 templates for Catalyst Center?

1. User: "Can I use string concatenation with `+` in Jinja2 templates for Catalyst Center?"
1. Assistant:
    - Loads `rules/jinja2/cc-vs-ansible.md`.
    - Explains that in Catalyst Center's Jinja2 engine, string concatenation uses the `~` operator instead of `+`. The `+` operator is treated as arithmetic, so if you try to concatenate strings with `+`, you may end up with unexpected results or errors. For example, instead of writing `{{ "Hello, " + name }}`, you should write `{{ "Hello, " ~ name }}` to concatenate the string "Hello, " with the variable `name`. For more details, see `rules/jinja2/cc-vs-ansible.md §3.2`.".

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
