---
title: "Velocity — Common Mistakes"
engine: velocity
kind: guardrails
topic: common-mistakes
path: guardrails/velocity/common-mistakes.md
---

# Common Mistakes — Velocity Templates in Catalyst Center

Behavioural pitfalls — patterns that **parse cleanly but produce wrong output**.
For pure syntax errors see [invalid-syntax-patterns.md](./invalid-syntax-patterns.md);
for Catalyst Center platform limits see [what-cc-cannot-do.md](./what-cc-cannot-do.md).

Cross-references throughout point at [../../rules/velocity/constraints.md](../../rules/velocity/constraints.md)
and [../../rules/velocity/variable-types.md](../../rules/velocity/variable-types.md).

---

## 1. Using `!` as a Velocity comment

`!` is **not** a comment to Velocity — it is forwarded verbatim to IOS (where
it is a CLI separator).

```vtl
## Bad — leaks an extra blank/comment line to the device every render
! This is a developer note

## Good
## This is a developer note
```

## 2. Doing math on a bind variable without `parseInt`

Every bind / system variable arrives as a **string**, even when it represents
an integer. `"2" + 10` does not equal `12` in Velocity.

```vtl
## Bad
#set( $data_vlan = $native_bind + 10 )

## Good
#set( $integer    = 0 )
#set( $native_vlan = $integer.parseInt($native_bind) )
#set( $data_vlan   = $native_vlan + 10 )
```

## 3. Missing formal notation when adjacent to text

`$vicemaniac` is one identifier. Use `${...}` when a reference is immediately
followed by letters, digits, or `_`.

```vtl
## Bad — Velocity looks for $vicemaniac (undefined → renders literally)
Jack is a $vicemaniac.

## Good
Jack is a ${vice}maniac.
```

Real-world example (interface names):

```vtl
## Bad
interface gi $Switch/0/1

## Good
interface gi ${Switch}/0/1
```

## 4. Single quotes vs double quotes in `#set`

Single quotes are literal — they do **not** interpolate.

```vtl
#set( $name = "Ben" )
#set( $a = "Hello $name" )    ## "Hello Ben"
#set( $b = 'Hello $name' )    ## "Hello $name"  ← bug if you wanted interpolation
```

## 5. Calling a macro before it is defined

Velocity resolves macros at parse time within the template. Forward references
fail silently (the call renders as literal text) or throw, depending on engine
config.

```vtl
## Bad
#access_interface()   ## macro not yet defined → renders literally
#macro( access_interface )
  switchport mode access
#end

## Good — define first, then call
#macro( access_interface )
  switchport mode access
#end
#access_interface()
```

## 6. Defining macros inside `#if` or `#foreach`

Macros must be defined at the top level. Defining one inside a block leads to
inconsistent visibility and is rejected by stricter parsers.

```vtl
## Bad
#foreach( $i in [1..3] )
  #macro( pickle ) ... #end   ## don't do this
#end

## Good — define once, outside; parameterise instead
#macro( pickle $idx )
  ...
#end
#foreach( $i in [1..3] )
  #pickle( $i )
#end
```

## 7. Forgetting `#MODE_ENABLE` for privileged commands

Stack priority and similar operational commands must be issued from
privileged-EXEC, not from config mode. Wrap them.

```vtl
## Bad — config-mode push, ignored by IOS
#foreach( $Switch in [1..$StackMemberCount] )
  switch $Switch priority 10
#end

## Good
#MODE_ENABLE
#MODE_END_ENABLE
#MODE_ENABLE
#foreach( $Switch in [1..$StackMemberCount] )
  switch $Switch priority 10
#end
#MODE_END_ENABLE
```

## 8. Not capturing the return value of mutating methods

`.add()` returns a boolean, `.set()` returns the previous value. If you don't
swallow the return value via `#set`, Velocity renders it as a literal token in
the CLI output.

```vtl
## Bad — emits "true true true" into the configuration
#foreach( $Switch in [1..$Switches] )
  $PortArray.add($PortsCount)
#end

## Good — capture and discard
#foreach( $Switch in [1..$Switches] )
  #set( $foo = $PortArray.add($PortsCount) )
#end
```

## 9. Off-by-one between array index and switch number

Catalyst 9k stacks number switches starting at 1; lists are 0-based. Keep both
indices explicit.

```vtl
#set( $offset = $StackMemberCount - 1 )
#foreach( $Switch in [0..$offset] )      ## 0-based for array access
  #set( $SwiNum = $Switch + 1 )           ## 1-based for IOS
  interface range gi ${SwiNum}/0/1 - $PortTotal[$Switch]
    #access_interface()
#end
```

## 10. Treating a string bind variable as iterable

```vtl
## Bad — $Count is the string "8", not the number 8
#foreach( $i in [1..$Count] )

## Good
#set( $integer = 0 )
#set( $count   = $integer.parseInt($Count) )
#foreach( $i in [1..$count] )
```

## 11. Reusing the `$__` namespace

Catalyst Center reserves `$__*` for system variables. Declaring your own breaks
introspection and is a subtle source of bugs.

```vtl
## Bad
#set( $__device = "myswitch" )

## Good
#set( $device_name = "myswitch" )
```

## 12. Hyphens in identifiers

Catalyst Center does **not** enable `parser.allow_hyphen_in_identifiers`, so
`-` is always subtraction or a literal CLI character — never part of an
identifier.

```vtl
## Bad — parsed as $voice - $offset, both undefined
#set( $voice-offset = 4000 )

## Good
#set( $voice_offset = 4000 )
```

## 13. Forgetting `<MLTCMD>` around multi-line CLI

Banners, key chains, and certificate blobs must be delivered as a single CLI
unit. Without the wrapper, Catalyst Center sends each line separately and IOS
breaks out of the multi-line context.

```vtl
<MLTCMD>banner login ^
  Authorized access only.
^</MLTCMD>
```

## 14. Forgetting `! @ start-ignore-compliance` for intentional drift

If you knowingly deviate from Design settings, mark the block so compliance
does not flag it on every run.

```vtl
interface GigabitEthernet1/0/1
 ! @ start-ignore-compliance
 no switchport
 ! @ end-ignore-compliance
 switchport mode trunk
```

## 15. Comparing different types with `==`

When the two sides are different classes, Velocity falls back to comparing
`toString()` results. This silently hides type bugs.

```vtl
## Looks like a number compare, is actually a string compare
#if( $native_bind == 10 )   ## "10" == 10 → toString compare, may pass by accident
```

Parse to a known type first (see item 2).

## 16. Whitespace gobbling joining CLI lines

Velocity removes the newline after a directive-only line, which can fuse lines
in the rendered config. Confirm with the Simulation feature.

```vtl
## Looks fine in source
interface gi1/0/1
  #access_interface()
  description core uplink

## May render as
interface gi1/0/1
  switchport mode access  description core uplink
```

Insert an explicit newline inside the macro body or after the call as needed.

## 17. Putting Velocity directives inside a single-quoted string

Directives are not parsed inside string literals. Single quotes additionally
suppress reference interpolation.

```vtl
## Bad — '#if' is never evaluated
#set( $banner = '#if( $secure ) Secure #end' )
```

## 18. Mixing PnP-onboarding logic with bind / system variables

During PnP the device is not yet in the Inventory database. Bind variables and
`$__*` system variables are **empty**. Plan onboarding templates around the
operator's input form only.

```vtl
## Bad — in an onboarding template
hostname $__device.hostname

## Good — use a user-input variable in onboarding, switch to bind in DayN
hostname ${Hostname}
```

## 19. Overlapping CLI between template and Design settings

If Catalyst Center's Design app already pushes a setting, do not push it again
from a template. Two sources of truth produce compliance flapping.

* Hostname, domain name, NTP, AAA, SNMP: prefer Design.
* Use templates only for things Design does not cover, or for site-specific
  overrides.

## 20. Composite templates in PnP

Composite templates are **DayN only**. Attaching one to a PnP workflow yields
no effect.

## 21. Forgetting that `#stop` still emits accumulated output

`#stop` halts further rendering but Catalyst Center sends whatever was
generated up to that point. It is not a safe abort. Use `#if` to guard a whole
template body instead.

```vtl
#if( $do_provision )
  ! ...full template here...
#end
```

## 22. Mistaking the quiet reference for logical NOT

`$!foo` is the **silent** notation; `!$foo` is logical NOT.

```vtl
#if( !$foo )   ## logical NOT
#if( $!foo )   ## ALWAYS true (a string is truthy); not what you wanted
```

## 23. Trying to concatenate strings with `+`

There is no `+` for strings in Velocity. Use adjacency.

```vtl
## Bad
#set( $clock = $size + $name )

## Good
#set( $clock = "${size}${name}" )
```

## See also

* [../../rules/velocity/constraints.md](../../rules/velocity/constraints.md)
* [../../rules/velocity/variable-types.md](../../rules/velocity/variable-types.md)
* [../../docs/velocity/basics.md](../../docs/velocity/basics.md)
* [../../docs/velocity/advanced.md](../../docs/velocity/advanced.md)
* [../../docs/templates.md](../../docs/templates.md)
* [./invalid-syntax-patterns.md](./invalid-syntax-patterns.md)
* [./what-cc-cannot-do.md](./what-cc-cannot-do.md)
