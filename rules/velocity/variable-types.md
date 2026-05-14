---
title: "Velocity — Variable Types"
engine: velocity
kind: rules
topic: variable-types
path: rules/velocity/variable-types.md
---

# Variable Types (Velocity / Catalyst Center)

This document classifies every kind of variable a Velocity template author will
encounter inside Catalyst Center. It is a companion to [Variables.md](../../docs/variables.md)
and [SystemVariables.md](../../docs/system-variables.md). For the hard syntax rules see
[velocity-constraints.md](constraints.md).

## 1. Categories at a glance

| Category            | Origin                                                            | Available in PnP? | Available in DayN? | Typical reference form                          |
|---------------------|-------------------------------------------------------------------|-------------------|--------------------|-------------------------------------------------|
| User-input          | Declared in template, populated by operator at provisioning time  | Yes               | Yes                | `$hostname`, `${MgmtVlan}`                      |
| Bind variable       | Bound in Input Form to an Inventory / Settings / Profile / Cloud attribute | **No**   | Yes                | `$ProductID`, `$Serial`, `$native_bind`         |
| System variable     | Built-in Catalyst Center object (Inventory, NetworkProfile, ...)  | **No**            | Yes                | `$__device`, `$__interface`                     |
| Local (`#set`)      | Computed inside the template                                      | Yes               | Yes                | `$data_vlan_number`, `$StackMemberCount`        |
| Literal             | Inline constant in the template                                   | Yes               | Yes                | `"text"`, `123`, `[1..3]`, `{"k":"v"}`          |
| Property reference  | Bean-style accessor on an object                                  | Same as parent    | Same as parent     | `$interface.portName`                           |
| Method reference    | Java method call on an object                                     | Same as parent    | Same as parent     | `$ProductID.split(",")`                         |
| Indexed reference   | List / map / array element                                        | Same as parent    | Same as parent     | `$StackPIDs[$i]`, `$map["banana"]`              |

> **Rule of thumb:** PnP onboarding has no inventory record yet, so anything
> sourced from Catalyst Center's database (`bind` and `__system__` variables)
> is unavailable. Reserve those for DayN templates.

## 2. User-input variables

A user-input variable is any `$name` that appears in the template and is **not**
declared with `#set` and **not** bound to a source. Catalyst Center surfaces it
in the Input Form so the operator (or API caller) supplies a value at
provisioning time.

```vtl
hostname ${Hostname}
vlan ${MgmtVlan}
```

Form configuration (Template Editor → calculator icon):

* **Variable Definition** (the data type)
  * `String`
  * `Integer`
  * `IP Address`
  * `MAC Address`
* **Display Type** (how the operator enters it)
  * `Text` — free text, uses *Default Value* / *Instructional Text*
  * `Single Select` — required for **Bind to Source**
  * `Multi Select` — list of values
* Optional: *Tool Tip*, *Maximum Characters*, *Required*.

## 3. Bind variables

A bind variable is a user-input variable whose value is auto-populated from
Catalyst Center's database at provisioning time. Configure via the Input Form:

1. Set **Display Type** = `Single Select`.
2. Check **Bind to Source**.
3. Choose **Source**:
   * `Network Profile` — e.g., SSID name.
   * `Common Settings` — NTP, DNS, domain.
   * `Cloud Connect` — tunnel info.
   * `Inventory` — per-device attributes (PID, serial, hostname, location…).
4. Choose the **Entity** and **Attribute**.

Bind values arrive as **strings**. Cast before arithmetic:

```vtl
#set( $integer = 0 )
#set( $native_vlan = $integer.parseInt($native_bind) )
#set( $data_vlan  = $native_vlan + 10 )
```

Common bind-variable patterns from the `.vm` corpus:

```vtl
## Bound to Inventory → Platform Id (comma-delimited for a stack)
#set( $StackPIDs       = $ProductID.split(",") )
#set( $StackMemberCount = $StackPIDs.size() )

## Bound to Inventory → Serial Number
#set( $StackCount      = $Serial.split(",") )
```

## 4. System variables

System variables (sometimes called `__system__` variables) are objects
maintained by Catalyst Center. They are listed under **Template System
Variables** in the Template Editor. The four sources are the same as for bind
variables; the names always begin with `$__`.

| System variable       | Source           | Description                                              |
|-----------------------|------------------|----------------------------------------------------------|
| `$__device`           | Inventory        | Single device object (hostname, snmpLocation, …)         |
| `$__interface`        | Inventory        | List of interface objects on the target device           |
| `$__networkSettings`  | Common Settings  | DNS, NTP, syslog, AAA, etc.                              |
| `$__credentials`      | Common Settings  | Device credentials object                                |
| `$__networkProfile`   | Network Profile  | Profile-scoped attributes (e.g., SSIDs)                  |
| `$__cloudConnect`     | Cloud Connect    | Cloud tunnel/connector data                              |

> The authoritative list is the **Template System Variables** dialog in your
> Catalyst Center version — click **Show More** to see the full set. Names and
> attributes do change between releases.

Iterating a system variable:

```vtl
#foreach( $interface in $__interface )
  #if( $interface.portMode == "trunk" && $interface.interfaceType == "Physical" )
    interface $interface.portName
     #uplink_physical()
  #end
#end
```

Reading a scalar attribute:

```vtl
#if( $__device.snmpLocation == "" )
  snmp-server location ${default_location}
#end
```

**Reserved naming:** never declare `#set( $__foo = ... )`. The `$__` prefix is
reserved for system variables.

## 5. Local variables (`#set`)

Created and used inside the template. They are typed by what you assign:

```vtl
#set( $StringVariable  = "text" )      ## String
#set( $NumericVariable = 10 )          ## Integer
#set( $Flag            = true )        ## Boolean
#set( $L2vlans         = [ "10", "18" ] )                ## List
#set( $Vlans           = [1..5] )                        ## Range (becomes a list)
#set( $Naming          = { "banana":"good", "beef":"bad" } ) ## Map
```

`#set` does **not** take an `#end`. The RHS may be a reference, literal,
property, method, or arithmetic expression. See
[velocity-constraints.md](constraints.md#set).

## 6. Literals

| Literal kind  | Example                              | Notes                                                                 |
|---------------|--------------------------------------|-----------------------------------------------------------------------|
| String        | `"hello"` / `'hello'`                | **Double quotes interpolate**, single quotes do not.                  |
| Integer       | `123`, `-4`                          | Used for math; division of two integers truncates.                    |
| Boolean       | `true`, `false`                      | Lowercase only.                                                       |
| List          | `[ "a", $b, "c" ]`                   | Access with `$list[$i]` or `.get($i)`.                                |
| Map           | `{ "k":"v", "k2":"v2" }`             | Keys must be quoted. Access with `$map["k"]` or `$map.k`.             |
| Range         | `[1..5]`                             | Only renders inside `#set` / `#foreach`. Counts down if start > end.  |

## 7. Property, method, and indexed references

```vtl
$customer.Address       ## Property — resolves to getAddress() / get("Address")
$ProductID.split(",")   ## Method  — explicit call with args
$StackPIDs[$Switch]     ## Indexed — equivalent to $StackPIDs.get($Switch)
${purchase.getTotal()}  ## Formal notation with a method call
```

Useful methods exposed in Catalyst Center templates (verified across the `.vm`
corpus and Apache Velocity 2.3):

| Method                    | Purpose                                          |
|---------------------------|--------------------------------------------------|
| `.size()`                 | Element count of a list / map / string           |
| `.add(x)`                 | Append to a list (returns boolean — capture it!) |
| `.get(i)`                 | Element by index / key                           |
| `.split(",")`             | String → list                                    |
| `.replaceAll(re, repl)`   | Regex replace; back-references with `$1`         |
| `.contains("…")`          | Substring / element test                         |
| `.toString()`             | Explicit string coercion                         |
| `.parseInt(s)`            | On a seeded `#set( $integer = 0 )` reference     |

## 8. Reference notations

| Notation                  | Example                            | Use                                        |
|---------------------------|------------------------------------|--------------------------------------------|
| Shorthand                 | `$Switch`                          | Default form                               |
| Formal                    | `${Switch}`                        | Required when adjacent to non-identifier text (`${vice}maniac`) |
| Quiet (silent)            | `$!email`                          | Render empty string if undefined           |
| Formal + quiet            | `$!{email}`                        | Combine adjacency and silencing            |
| Alternate value           | `${name|'John Doe'}`               | Default if null / empty / false / zero     |

## 9. Identifier rules

* Start with a letter or `_`.
* Then any combination of letters, digits, and `_`.
* **No hyphens.** Catalyst Center runs Velocity without
  `parser.allow_hyphen_in_identifiers`, so `$voice-offset` parses as
  `$voice - $offset`.
* Case sensitive.
* Avoid the `$__` prefix — reserved for system variables.

## 10. Catalyst Center → VTL type cheat sheet

| CC UI "Variable Definition" | CC UI "Display Type"    | VTL data type at render | Bindable? |
|-----------------------------|-------------------------|-------------------------|-----------|
| String                      | Text                    | String                  | No        |
| String                      | Single Select           | String                  | Yes       |
| String                      | Multi Select            | List<String>            | No        |
| Integer                     | Text                    | String → cast with `parseInt` | No  |
| Integer                     | Single Select           | String → cast           | Yes       |
| IP Address                  | Text                    | String                  | No        |
| MAC Address                 | Text                    | String                  | No        |

> Bind-to-Source is supported **only** with the `Single Select` display type.

## See also

* [Variables.md](../../docs/variables.md) — original tutorial.
* [SystemVariables.md](../../docs/system-variables.md) — deep dive on `$__*`.
* [velocity-constraints.md](constraints.md) — full syntax rules.
* [Velocity common mistakes](../../guardrails/velocity/common-mistakes.md) — what
  goes wrong with variables in practice.
