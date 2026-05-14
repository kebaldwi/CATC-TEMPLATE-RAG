---
title: "Velocity — Syntax & Semantic Constraints"
engine: velocity
kind: rules
topic: constraints
path: rules/velocity/constraints.md
---

# Velocity Constraints (Catalyst Center)

Hard syntactic and semantic constraints for authoring Apache Velocity (VTL)
templates inside Catalyst Center. Use this file as the authoritative rule book;
narrative material lives in [Velocity.md](../../docs/velocity/basics.md) and
[AdvancedVelocity.md](../../docs/velocity/advanced.md), and variable taxonomy lives in
[variable-types.md](./variable-types.md).

Velocity engine reference: <https://velocity.apache.org/engine/2.3/vtl-reference.html>.
Cisco DNAC reference: <https://explore.cisco.com/dnac-use-cases/apache-velocity>.

---

## 1. Comments

| Form                | Example                          | Notes                                   |
|---------------------|----------------------------------|-----------------------------------------|
| Single line         | `## comment`                     | Until end of line. **Use this.**        |
| Multi line          | `#* ... *#`                      | Any span.                               |
| Doc block           | `#** ... *#`                     | Velocity doc-style.                     |
| **Not a comment**   | `! comment`                      | Renders as a real IOS CLI line.         |

```vtl
## This is removed by Velocity at render time
! This is sent verbatim to the device (IOS treats it as a comment)
```

## 2. References

### 2.1 Notation forms

```vtl
$Switch              ## shorthand
${Switch}            ## formal — use when adjacent to text or other refs
$!Switch             ## quiet — empty string when undefined
$!{Switch}           ## quiet + formal
${Switch|'default'}  ## alternate value if null/empty/false/zero
```

### 2.2 Identifier rules <a id="identifiers"></a>

* First character: letter (a–z, A–Z) or `_`.
* Remaining characters: letters, digits, `_`.
* **No hyphens.** Catalyst Center does not enable
  `parser.allow_hyphen_in_identifiers`; `$voice-offset` is parsed as
  `$voice - $offset`.
* Case sensitive (`$Foo` ≠ `$foo`).
* Do not create your own `$__*` names — reserved for Catalyst Center system
  variables.

### 2.3 Property lookup

`$obj.attr` tries (in order):

1. `getattr()`
2. `getAttr()`
3. `get("attr")`
4. `isAttr()`

If the property starts uppercase (`$obj.Attr`), order 1 and 2 swap.

### 2.4 Indexed access

```vtl
$list[$i]            ## same as $list.get($i)
$map["banana"]       ## same as $map.get("banana")
$map.banana          ## same as $map.get("banana")
#set( $list[0] = 1 ) ## same as $list.set(0, 1)
```

## 3. Quoting and interpolation

| Form                 | Interpolates `$ref`? | Use                                         |
|----------------------|----------------------|---------------------------------------------|
| `"text $ref"`        | Yes                  | Build dynamic strings.                      |
| `'text $ref'`        | No                   | Treat content as literal.                   |
| `#[[ ... ]]#`        | No (raw block)       | Big blocks that contain `$`/`#` you want preserved. |

```vtl
#set( $name = "Ben" )
#set( $a = "Hello $name" )   ## $a = "Hello Ben"
#set( $b = 'Hello $name' )   ## $b = "Hello $name"
```

## 4. String concatenation

There is **no `+` operator for strings.** Use adjacency or formal notation.

```vtl
## Good
#set( $clock = "${size}Tall${name}" )
description "${site_code} - ${closet}"

## Bad — '+' on strings does arithmetic or fails
#set( $clock = $size + $name )
```

## 5. Arithmetic

```vtl
#set( $value = $foo + 1 )     ## addition
#set( $value = $foo - 1 )     ## subtraction
#set( $value = $foo * $bar )  ## multiplication
#set( $value = $foo / $bar )  ## integer division — truncates
#set( $value = $foo % $bar )  ## modulus
```

Bind variables arrive as **strings**; cast first:

```vtl
#set( $integer    = 0 )
#set( $native_vlan = $integer.parseInt($native_bind) )
```

## 6. Truthiness (what `#if` treats as false)

`#if( $x )` is false when `$x` is any of:

* `null` (undefined / missing)
* the boolean `false`
* the empty string `""`
* an empty collection / map / array
* the number `0`

Everything else is true. Catalyst Center runs in **non-strict mode**, so an
undefined reference renders literally (e.g., outputs `$x`) instead of throwing.
Guard with `$!x` or `#if( $x )`.

## 7. Operators

| Category   | Symbolic | Word   | Notes                                                   |
|------------|----------|--------|---------------------------------------------------------|
| Equal      | `==`     | `eq`   | Cross-type: falls back to `toString()` compare.         |
| Not equal  | `!=`     | `ne`   |                                                         |
| Greater    | `>`      | `gt`   |                                                         |
| Less       | `<`      | `lt`   |                                                         |
| ≥          | `>=`     | `ge`   |                                                         |
| ≤          | `<=`     | `le`   |                                                         |
| AND        | `&&`     | `and`  | Short-circuit.                                          |
| OR         | `\|\|`   | `or`   | Short-circuit.                                          |
| NOT        | `!`      | `not`  | Don't confuse with `$!ref` quiet notation.              |

## 8. Directives

### 8.1 `#set` <a id="set"></a>

* Single statement, **no `#end`**.
* LHS must be a variable or property reference.
* RHS may be: reference, string literal, property, method, number, boolean,
  list, map, range, or arithmetic expression.
* If RHS evaluates to `null`, the LHS is **not** assigned.

### 8.2 `#if` / `#elseif` / `#else`

```vtl
#if( $hostname.contains("C9300-48") )
  ! 48-port branch
#elseif( $hostname.contains("C9300-24") )
  ! 24-port branch
#else
  ! default branch
#end
```

When `#else` is immediately followed by non-whitespace text, use brace form:

```vtl
#if( $a == 1 )true#{else}false#end
```

### 8.3 `#foreach`

```vtl
#foreach( $vlan in $Vlans )
  interface vlan $vlan
#else
  ! no vlans defined
#end
```

`arg` may be: a reference to a list / array / map / collection, an array
literal `["a",$b,"c"]`, or a range `[1..n]`.

Inside the loop:

| Token                | Meaning                              |
|----------------------|--------------------------------------|
| `$foreach.count`     | 1-based index                        |
| `$foreach.index`     | 0-based index                        |
| `$foreach.first`     | true on first iteration              |
| `$foreach.last`      | true on last iteration               |
| `$foreach.hasNext`   | false on last iteration              |
| `$foreach.stop()`    | exit loop (same as `#break`)         |
| `$foreach.parent`    | outer loop's scope when nested       |
| `$foreach.topmost`   | outermost loop's scope               |

Engine-wide max iteration cap is `directive.foreach.max_loops` (default `-1` =
unlimited).

### 8.4 `#macro` (Velocimacro)

```vtl
#macro( access_interface )
  switchport mode access
  switchport access vlan ${data_vlan_number}
  spanning-tree portfast
#end

interface gi1/0/1
  #access_interface()
```

Constraints:

* **Define before use.** Velocity resolves macros at parse time within a
  template.
* Argument count at invocation must match the definition (unless trailing
  defaults are supplied).
* Defaults are right-to-left: if any arg has a default, all later args must
  also have defaults.
* Do **not** nest `#macro` inside `#macro`, `#if`, or `#foreach`. Define at the
  top level of the template.
* Catalyst Center has **no global macro library**. A macro is visible only
  within the template that defines it.
* Body macros: call with `#@name(...) body #end` and reference the body inside
  the macro as `$!bodyContent`.

### 8.5 `#break` and `#stop`

* `#break` exits the innermost content directive (e.g., a `#foreach`).
  Optionally pass a scope reference: `#break($foreach.parent)`.
* `#stop` halts rendering of the entire template. Output already produced is
  still emitted to the device — **not a safe abort**.

### 8.6 Directives effectively unavailable in Catalyst Center

| Directive    | Status in CC                                                  |
|--------------|---------------------------------------------------------------|
| `#include`   | Not usable — no filesystem access.                            |
| `#parse`     | Not usable — cross-template inclusion is not supported.       |
| `#evaluate`  | Not exposed.                                                  |
| `#define`    | Not exposed.                                                  |

For composition, use composite templates (DayN) or multiple template
attachments on the Network Profile.

## 9. Catalyst Center–specific syntax

### 9.1 `#MODE_ENABLE` / `#MODE_END_ENABLE`

Wraps blocks that require privileged-EXEC (not config-mode) execution, e.g.,
stack priority assignment:

```vtl
#MODE_ENABLE
#MODE_END_ENABLE
#MODE_ENABLE
#foreach( $Switch in [1..$StackMemberCount] )
  #if( $Switch == 1 )
    switch $Switch priority 10
  #elseif( $Switch == 2 )
    switch $Switch priority 9
  #else
    switch $Switch priority 8
  #end
#end
#MODE_END_ENABLE
```

### 9.2 `<MLTCMD>...</MLTCMD>`

Used for banners and other multi-line CLI that must be delivered as a single
command:

```vtl
<MLTCMD>banner login ^
  Session On $hostname Is Monitored!!!
  ... legal notice ...
^</MLTCMD>
```

### 9.3 Compliance ignore markers

```vtl
interface GigabitEthernet1/0/1
 ! @ start-ignore-compliance
 no switchport
 ! @ end-ignore-compliance
 switchport mode trunk
```

Lines between the markers are not evaluated for compliance drift.

## 10. Escaping

| Source            | Renders as                | Notes                                            |
|-------------------|---------------------------|--------------------------------------------------|
| `\$foo` (defined) | `$foo`                    | Backslash escape only fires when ref is defined. |
| `\$bar` (undef.)  | `\$bar`                   | Backslash preserved verbatim.                    |
| `#[[ raw ]]#`     | `raw` (no parsing)        | Preferred for big literal blocks.                |
| `\#if( ... )`     | `#if( ... )` (literal)    | Escapes a directive — beware unbalanced `#end`.  |

## 11. Whitespace gobbling

Velocity removes the newline that immediately follows a directive on its own
line. This can collapse CLI tokens:

```vtl
interface gi1/0/1
  #access_interface()
  description …
```

`#access_interface()` is on its own line, so its trailing newline is swallowed.
The macro body must therefore start with a newline (or explicit indentation) if
the next CLI line must not join its last line.

## 12. Velocity vs IOS character interactions

| Character | Velocity meaning              | IOS meaning              | Conflict mitigation                                       |
|-----------|-------------------------------|--------------------------|-----------------------------------------------------------|
| `#`       | Directive prefix              | Privileged prompt        | OK in CLI body; only meaningful at line start in VTL.     |
| `$`       | Reference prefix              | Banner / regex anchor    | Escape with `\$` when defined; use `#[[ ]]#` for blocks.  |
| `!`       | None — passed through         | Comment / separator      | Use `##` for Velocity comments.                           |
| `{ }`     | Formal reference / map        | None                     | Use `${var}` only when needed.                            |

## 13. Conventions (Catalyst Center `.vm` corpus)

These idioms are present across multiple templates in
[examples/velocity/](../../examples/velocity/) and should be considered
canonical for this repo:

1. **Stack discovery from a bind variable**
   ```vtl
   #set( $StackPIDs        = $ProductID.split(",") )
   #set( $StackMemberCount = $StackPIDs.size() )
   ```
2. **Per-switch port count**
   ```vtl
   #set( $PortTotal = [] )
   #set( $offset    = $StackMemberCount - 1 )
   #foreach( $Switch in [0..$offset] )
     #set( $Model     = $StackPIDs[$Switch] )
     #set( $PortCount = $Model.replaceAll("C9300L?-([2|4][4|8]).*","$1") )
     #set( $foo       = $PortTotal.add($PortCount) )
   #end
   ```
3. **0-based array index, 1-based switch number**
   ```vtl
   #foreach( $Switch in [0..$offset] )
     #set( $SwiNum = $Switch + 1 )
     interface range gi ${SwiNum}/0/1 - $PortTotal[$Switch]
       #access_interface()
   #end
   ```
4. **Swallow mutating method return values**
   ```vtl
   #set( $foo = $PortTotal.add($PortCount) )   ## $foo discarded
   ```

## See also

* [Velocity.md](../../docs/velocity/basics.md)
* [AdvancedVelocity.md](../../docs/velocity/advanced.md)
* [variable-types.md](./variable-types.md)
* [Velocity common mistakes](../../guardrails/velocity/common-mistakes.md)
* [Velocity invalid syntax](../../guardrails/velocity/invalid-syntax-patterns.md)
* [What CC cannot do](../../guardrails/velocity/what-cc-cannot-do.md)
