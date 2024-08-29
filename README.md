# What is this project?

This project provides a simple parser/interpreter for a LISP-ish language
(without all the cool features you might expect from a LISP, but with polish
notation).
It is intended for the creation of scripts for playing cards.

# How do I run this?

Simply copy `script_interpreter.gd` to the Godot project of your choice and
connect it to a Node.

A card script is a text file that consists of a number of section, each
containing text or code:

- `NAME` - The name of the script, intended for internal referencing
- `DISPLAYNAME` - The display name of the card defined by the script
- `DESCRIPTION` - The card's description
- `IMAGE-PATH` - Path to the card's image
- `TAGS` - Optional list of string tags the card is associated with
- `TARGET` - Either `global` or `local`; Indicates whether the card is
  intended to target a specific entitiy (local) or has a global effect
  (global). This is taken into account by ScriptBlock.is_applicable before
  evaluating the card's `CONDITION`, meaning that false is returned directly if
  the environment's has_target value does not match the card's TARGET value.
- `CONDITION` - A single expression that is evaluated when calling
  `ScriptBlock.is_applicable`, to check whether the card is playable given the
  target and environment
- `EFFECT` - Any number of expressions that are evaluated when
  `ScriptBlock.apply` is called.

Once you have written a script, load it using `ScriptInterpreter.parse_script`.

The resulting `ScriptBlock` object can now be queried for the script's
properties.

However, to evaluate the script you first need to create a `ScriptEnvironment`
object. This is passed to the `is_applicable` and `apply` functions.

An environment consists of

- A dictionary of global variables (`global_vars`)
- An optional dictionary of variables local to the current target
  (`target_vars`). This is only valid if `has_target` is true.
- A dictionary of package data. This is for use by
  [user-defined functions](#user-defined-packages). Use
  `ScriptInterpreter.package_data` to initialize this member.

# User-Defined Packages

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

class AdditionScriptNode extends ScriptInterpreter.ScriptNode:
    static func create(token_: String, filename_: String, line_number_: int, line_offset_: int) -> ScriptNode:
        AdditionScriptNode.new().init_helper(1, token_, filename_, line_number_, line_offset_)

    func evaluate(args: Array[ScriptNode], env: ScriptEnvironment) -> int:
        var context = env.package_data("my-script-pkg")
        context.my_integer += args[0].evaluate([], env)
        return context.my_integer

...
```

and the package definition

```
var script_interpreter: ScriptInterpreter = ...

script_interpreter.define_package("my-script-pkg", { "inc" = AdditionScriptNode }, self)
```

the following script could be used to increment my_integer by 5:

```
...
EFFECT
    ...
    (inc (+ 2 3))
    ...
```
