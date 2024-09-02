# What is this project?

This project provides a simple parser/interpreter for a LISP-ish language
(without all the cool features you might expect from a LISP, but with polish
notation).
It is intended for the creation of scripts for playing cards.



# How do I use this?

Simply copy `script_interpreter.gd` to your Godot project and either load it
globally, or connect it to a Node.

A card script is a text file that consists of a number of properties, each
containing text or code. The sections can be freely defined via the properties
argument of `ScriptInterpreter.load_script`. This consists of an array of
`ScriptProperty` objects, each defining a property:

- `property_name` - The name of the property. To define the property in a
  script, write the property name in curly brackets, followed by the
  property's value(s).
- `required` - Whether the property is required in a script. If this is false,
  the absence of the property will prevent parsing of the script.
- `multiple` - Whether the property has only or multiple values. Multiple
  values are specified by simply concatenating them, separated by whitespace.
- `pattern` - An optional RegEx against which the values of the property are
  matched. Mismatching values will prevent parsing of the script.
- `is_code` - Whether the property expects strings or
  [script nodes](#script-nodes) as its value(s).

Once you have written a script, load it using `ScriptInterpreter.load_script`
(Note that on failure, an empty `Dictionary` is returned, and a diagnostic
message is printed to the console).

The resulting `Dictionary` can now be queried for the script's properties.

However, to evaluate any of the script's code properties you first need to
create a [`ScriptEnvironment`](#variables) object. This is passed to the
`ScriptNode.evaluate` function.



# Script Nodes

Scripts consist of any number of text and code properties. A code property
consists of one or more `ScriptInterpreter.ScriptNode` properties.

Each such script node is defined as an S-Expression, which is either:

- An atom, i.e. a operator that takes no arguments, a [variable](#variables),
  or a [constant](#constants).
- Any number of S-Expressions, enclosed in parentheses. The first element is
  interpreted as an operator, which is applied to all following elements
  (possibly zero). \
  Note that a completely empty list (`()`) is also allowed as a special case,
  and evaluates to null.

All tokens that are not parentheses, [variables](#variables) or
[constants](#constants) are assumed to be operators. If an operator cannot be
resolved to either a [builtin operator](#builtin-operators) or a
[user-defined operator](#user-defined-operators), the script containing it
cannot be parsed.

Calling `ScriptNode.evaluate` allows evaluating a script node with a list of
arguments and an environment.


## Variables

The script interpreter distinguishes between two types of variables: Global and
local. This distinction is mainly useful because of the intended usage in
scripting playing cards, with global variables tracking e.g. player score, and
local variables tracking the state of the target a card was played onto (e.g.
an enemy's health).

Global variables are accessed by prefixing their name with `$`, while local
variables are accessed with the `%` prefix. 

Variables are looked up in the environment passed to `ScriptNode.evaluate`.
(In `local_vars` or `global_vars` respectively). Additionally, if `has_local`
is set to false, local variable lookup always fails, irrespective of the
actual values stored in `local_vars`.


## Constants

Only a few basic types of constants are supported:

- string - Any text surrounded by double parentheses (`"`)
- integer - Any text that is considered an integer by Godot's
  `String.is_valid_int` function.
- float - Any text that is considered a float by Godot's
  `String.is_valid_float` function and is not an integer.
- bool - The literals `true` and `false`


## Builtin Operators

There are a number of builtin operators which provide support for basic
arithmetic and logic. In the following, names in `<>` indicate
S-Expressions. If they followed by a `:`, the name after it defined the
type

- `(if <condition:bool> <consequent> <alternative>)` - Evaluates to
  `consequent` if `condition` is true; otherwise evaluates to `alternative`.
  The non-taken branch is not evaluated.
- `self` - Evaluates to the suppied `ScriptEnvironment`'s `self_object`.
- `(. <elements>...)` - Takes any number of elements and creates an array out
  of them.
- `(setf <operator:ScriptNode> <variable:Variable> <rhs>)` - equivalent to
  `(set variable (operator variable rhs)`, but only evaluates `variable` once.
- `(set <variable:Variable> <value>)` - Sets the given [`variable`](#variables)
  to `value`.
- `(== <lhs> <rhs>)` - Evaluates to Godot's `==` operator invoked on its
  arguments.
- `(!= <lhs> <rhs>)` - Evaluates to Godot's `!=` operator invoked on its
  arguments.
- `(< <lhs> <rhs>)` - Evaluates to Godot's `<` operator invoked on its
  arguments.
- `(<= <lhs> <rhs>)` - Evaluates to Godot's `<=` operator invoked on its
  arguments.
- `(> <lhs> <rhs>)` - Evaluates to Godot's `>` operator invoked on its
  arguments.
- `(>= <lhs> <rhs>)` - Evaluates to Godot's `>=` operator invoked on its
  arguments.
- `(not <value:bool>)` - Evaluates to Godot's `not` operator invoked on its
  argument.
- `(and <term:bool>...)` - Evaluates to true if and only if all `terms` evaluate
  to true. Note that just `(and)` with no arguments evaluates to true.
- `(or <term:bool>...)` - Evaluates to false if and only if any `terms` evaluate
  to false. Note that just `(or)` with no arguments evaluates to false.
- `(+ <lhs> <rhs>)` - Evaluates to Godot's `+` operator invoked on its
  arguments. If `lhs` and `rhs` are `Array`s, they are concatenated.
- `(- <lhs> <rhs>)` - Evaluates to Godot's `-` operator invoked on its
  arguments.
- `(* <lhs> <rhs>)` - Evaluates to Godot's `*` operator invoked on its
  arguments.


# User-Defined Operators

It is likely that your scripts will want to interact with your game beyond just
reading or modifying Dictionaries with a few basic types.

This is facilitated by allowing the definition custom operators.



While growing a card game, it might become necessary to closely couple the
effects cards may have with different aspects of your game. To bridge the gap
between your game's components and your card scripts, you can define your own
packages. This is done by calling `ScriptInterpreter.define_package`. \
Once defined, functions in these packages will be available to all future
scripts loaded by the interpreter. The `operators` dictionary contains a
mapping from script tokens to classes extending `ScriptInterpreter.ScriptNode`.

Given the ScriptNode class

```
...

var my_integer = 0

func _interpreter_add_to_my_integer(_args: Array[ScriptNode], _env: ScriptEnvironment) -> int
  var value: int = my_integer
  my_integer += 1
  return value

...
```

and the package definition

```
var script_interpreter: ScriptInterpreter = ...

script_interpreter.define_operators([
  ScriptInterpreter.CustomOperator.create("inc", _interpreter_add_to_my_integer, 0)
])
```

the following script node could be used to increment my_integer by 5:

```
(inc (+ 2 3))
```
