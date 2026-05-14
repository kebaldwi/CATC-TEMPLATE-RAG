---
title: "Velocity — Invalid Syntax Patterns"
engine: velocity
kind: guardrails
topic: invalid-syntax-patterns
path: guardrails/velocity/invalid-syntax-patterns.md
---

# Invalid Syntax Patterns — Velocity Templates in Catalyst Center

Patterns that **fail to parse** or throw at render time. For behavioural
mistakes (parses but wrong output), see
[common-mistakes.md](./common-mistakes.md). For platform limits see
[what-cc-cannot-do.md](./what-cc-cannot-do.md). Authoritative syntax rules
live in [../../rules/velocity/constraints.md](../../rules/velocity/constraints.md).

---

## 1. Mixing Jinja2 syntax into a Velocity template

Velocity uses `$ref` and `#directive`. Jinja2 markers are not recognized.

```vtl
## Bad
{% if hostname %}
  hostname {{ hostname }}
{% endif %}

## Good
#if( $hostname )
  hostname $hostname
#end
```

## 2. Missing `#end`

Every `#if`, `#foreach`, `#macro`, `#define`, `#@bodymacro` requires a matching
`#end`.

```vtl
## Bad
#if( $foo )
  do something
## (missing #end)

## Good
#if( $foo )
  do something
#end
```

## 3. `#else` adjacent to text without brace form

When the next token after `#else` is not whitespace, the parser cannot tell
where the directive ends.

```vtl
## Bad — Velocity sees the directive name as "elseno"
#if($a==1)true#elseno way!#end

## Good
#if($a==1)true#{else}no way!#end
```

## 4. Wrong comment delimiter

```vtl
## Bad — single '#' is a directive prefix, not a comment
# this is not a comment

## Good
## this is a single-line comment
#* this is a
   multi-line comment *#
```

## 5. Using `elif` (Python) instead of `#elseif`

```vtl
## Bad
#if( $a )
  ...
#elif( $b )
  ...
#end

## Good
#if( $a )
  ...
#elseif( $b )
  ...
#end
```

## 6. Identifiers that violate the naming rules

```vtl
## Bad — leading digit
#set( $1stPort = 1 )

## Bad — hyphen is the subtraction operator
#set( $voice-offset = 4000 )

## Bad — punctuation other than '_' is not allowed
#set( $port.count = 48 )    ## ".count" is a property accessor, not part of the name

## Good
#set( $first_port    = 1 )
#set( $voice_offset  = 4000 )
#set( $port_count    = 48 )
```

See [../../rules/velocity/constraints.md#identifiers](../../rules/velocity/constraints.md#identifiers).

## 7. Range or array literal outside `#set` / `#foreach`

```vtl
## Bad — emitted as the literal string "[1..3]"
[1..3]

## Good
#foreach( $i in [1..3] )
  $i
#end
```

## 8. `#set` written with a closing `#end`

`#set` is single-statement; it has no body.

```vtl
## Bad
#set( $foo = 1 )
  ...
#end

## Good
#set( $foo = 1 )
```

## 9. Macro argument count mismatch

```vtl
#macro( banner $name $level )
  banner motd ^ $name / $level ^
#end

## Bad — only one arg provided, no default declared
#banner( "edge-sw" )

## Good — provide all args, OR declare a default in the macro
#macro( banner $name $level = "info" )
  banner motd ^ $name / $level ^
#end
#banner( "edge-sw" )
```

Defaults are right-to-left: if `$arg1` has a default, all subsequent args must
also have defaults.

## 10. Nested `#macro` definitions

```vtl
## Bad
#macro( outer )
  #macro( inner )
    ...
  #end
#end

## Good — define both at top level
#macro( inner )
  ...
#end
#macro( outer )
  #inner()
#end
```

## 11. Using `+` for string concatenation

Velocity has no string-`+` operator; this is a parse error or silent
arithmetic.

```vtl
## Bad
#set( $clock = "Big" + "Ben" )

## Good
#set( $clock = "BigBen" )
#set( $clock = "${a}${b}" )
```

## 12. Map literal with unquoted keys

```vtl
## Bad
#set( $m = { foo : "bar" } )

## Good
#set( $m = { "foo" : "bar" } )
```

## 13. Trailing comma in array or map literal

```vtl
## Bad
#set( $vlans = [ "10", "20", ] )
#set( $m = { "k":"v", } )

## Good
#set( $vlans = [ "10", "20" ] )
#set( $m = { "k":"v" } )
```

## 14. Recursion in `#macro` without termination

Velocimacros that call themselves will recurse indefinitely (no Tail Call
Optimisation). The engine will throw a stack overflow at render time.

```vtl
## Bad
#macro( count_down $n )
  $n
  #count_down( $n )
#end

## Good — include a base case
#macro( count_down $n )
  $n
  #if( $n > 0 )
    #count_down( $n - 1 )
  #end
#end
```

## 15. Method or property on a `null`

In non-strict mode this silently renders nothing; in strict mode this throws.
Either way, code downstream that assumes a value will misbehave.

```vtl
## Risky
$__device.snmpLocation.toUpperCase()

## Safer
#if( $__device.snmpLocation )
  #set( $loc = $__device.snmpLocation.toUpperCase() )
#end
```

## 16. Unterminated `#[[ ... ]]#`

```vtl
## Bad — runs to end of template
#[[
literal block
no closing marker

## Good
#[[
literal block
]]#
```

## 17. Escaping a directive but leaving its `#end` un-escaped

```vtl
## Bad — escapes #if, leaves a dangling #end → parser error
\#if( $foo )
  body
#end

## Good — escape both ends, or escape neither
\#if( $foo )
  body
\#end
```

## 18. `#break` outside a content directive

`#break` is only legal inside `#foreach`, `#parse`, `#evaluate`, `#define`,
`#macro`, or a body macro.

```vtl
## Bad
#if( $stop_now )
  #break
#end

## Good — guard with a wrapper #foreach, or use #stop (with caveats)
#if( $stop_now )
  #stop
#end
```

## 19. Calling `#include` / `#parse` for cross-template composition

Both directives exist in standard Velocity but are not usable in Catalyst
Center (no filesystem access; templates are sandboxed). See
[what-cc-cannot-do.md](./what-cc-cannot-do.md).

```vtl
## Bad — fails or renders literally
#include( "common.vm" )
#parse( "common.vm" )

## Good — attach multiple templates to the Network Profile (DayN)
## or use a Composite template
```

## 20. `==` on different types where a String compare is unintended

Velocity compares unlike types via `toString()`. If you wanted numeric
equality, parse first.

```vtl
## Bad — silently does string compare
#if( $native_bind == 10 )

## Good
#set( $integer = 0 )
#set( $n = $integer.parseInt($native_bind) )
#if( $n == 10 )
```

## 21. Putting parentheses around the macro name on definition

```vtl
## Bad — macro name and arg list must be space-separated
#macro( access_interface() )
  ...
#end

## Good
#macro( access_interface )
  ...
#end
```

## 22. Calling a body macro without `@`

```vtl
## Bad — body is dropped
#macro( wrap )
  before $!bodyContent after
#end
#wrap()hello#end

## Good — '@' prefix activates body-mode
#@wrap()hello#end
```

## 23. Using single-`!` quiet notation expecting logical NOT

`$!foo` is a reference (silent notation), not a logical operation. Logical NOT
requires `!` in front of a parenthesised condition or a bare reference inside
`#if(...)`.

```vtl
## Bad — always renders "$!foo" → always truthy as a string
#if( $!foo )

## Good
#if( !$foo )
#if( !$foo.isEmpty() )
```

## 24. Curly-brace directive syntax with wrong brace placement

The bracket form is `#{directive}`, not `{#directive}`.

```vtl
## Bad
{#if}( $a ){#end}

## Good (rarely needed, but legal)
#{if}( $a )true#{end}
```

## See also

* [../../rules/velocity/constraints.md](../../rules/velocity/constraints.md)
* [../../rules/velocity/variable-types.md](../../rules/velocity/variable-types.md)
* [../../docs/velocity/basics.md](../../docs/velocity/basics.md)
* [../../docs/velocity/advanced.md](../../docs/velocity/advanced.md)
* [./common-mistakes.md](./common-mistakes.md)
* [./what-cc-cannot-do.md](./what-cc-cannot-do.md)
