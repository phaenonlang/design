# Phaenon — PHP versions: targets, and following PHP

Status: **draft** · 2026-11-02 · milestone *PHP 8.6* (`roadmap.md` §3) · companions:
`language-overview.md` §2, `functions.md`, `generics.md`

## 1. The rule: Phaenon follows PHP

Every PHP file is a Phaenon file (`language-overview.md` §2), so a PHP release's syntax, semantics
and library become Phaenon's when the release ships. Phaenon tracks PHP releases: the lexer and
parser port the release branch's `zend_language_scanner.l` and `zend_language_parser.y`, the
library is read from its stubs, and that release's `php` is the oracle. Each PHP release is a
milestone named after it ("PHP 8.6"), a track beside the feature milestones.

What a release adds is weighed like any Phaenon feature, against "can Tequila still run on it?":
nothing PHP ships is held back, and nothing Phaenon adds may collide with what PHP is reserving
(§8).

## 2. The target

A project compiles for a **target**: the oldest PHP version (major.minor) the emitted code must run
on.

| Source | Example | Target |
|---|---|---|
| `phaenon.json` `"php"` | `{"roots": {…}, "php": "8.6"}` | 8.6 |
| otherwise composer.json `require.php`, its lowest admitted version | `>=8.5`, `^8.5`, `~8.5.0`, `8.5.*`, `^8.5 \|\| ^9.0` | 8.5 |
| neither | | the floor (§2.2), with a note |

- **`phaenon.json` wins** because migrating to Phaenon and to a newer PHP is often one move:
  `phn migrate --php 8.6 src` writes `"php": "8.6"` into `phaenon.json` (creating the file if it
  does not exist) and lifts the code for that target.
- **A target above composer.json's floor is reported**, as a warning on `phaenon.json`:
  composer.json admits PHP 8.5 while the code is compiled for 8.6, so an install on 8.5 would load
  code it cannot run. Raising `require.php` is the fix, and is the developer's change:
  `phn migrate` does not edit composer.json. A target below the floor is merely conservative and
  passes silently.
- **A target outside what `phn` knows** (below the floor, above the newest release it supports)
  is an error on the project file.

### 2.1 What the target decides

- **Syntax and semantics.** A construct newer than the target is lowered to the target's PHP when
  the lowered code is exact and idiomatic, and reported otherwise (§3).
- **The library.** A function, class, method or constant newer than the target is reported where
  it is used: "`clamp()` needs PHP 8.6 (target 8.5)", rather than "unknown function" (§6).
- **Deprecations** are reported whatever the target. Code compiled for 8.5 will run on 8.6, and a
  deprecation is a warning about that future (`--no-deprecations` hides them, as today).

### 2.2 The floor

**PHP 8.5 stays the oldest target.** The cost is bounded:
- the target machinery is needed by the first PHP release after the newest one anyway;
- the 8.6 lowerings are few (§4);
- the rest of 8.6 is refused on an 8.5 target, which costs a diagnostic each.

What it buys:
- Tequila keeps building on 8.5 while its images, CI and FrankenPHP move to 8.6 after the
  release;
- a project can adopt Phaenon without upgrading PHP first.

The floor is revisited when PHP 8.5 leaves active support (2027-12-31), and in any case before the
repositories are published.

## 3. Newer than the target: lower or refuse

A feature newer than the target is **lowered** when the lowered PHP is exact (same behaviour,
reflection aside where §4 says so) and idiomatic (what a person writing for the older version
would write). Otherwise it is **refused** with one diagnostic, "… needs PHP 8.6 (target 8.5)",
naming the feature and the target. That is one code in the syntax family for constructs and one in
the interop family for library symbols.

Lowering never adds an identifier a reader could not trace (roadmap D9): where a lowered form needs
a parameter, it takes the name of the parameter it stands for in the called function's
declaration. For example, `str_replace(' ', '-', ?)` lowers to
`fn (array|string $subject) => str_replace(' ', '-', $subject)`.

## 4. PHP 8.6

| 8.6 change | Target 8.6 | Target 8.5 |
|---|---|---|
| **Partial function application** (RFC `partial_function_application_v2`): `f(?, 1)`, `f(1, ...)`, `f(b: ?)`, `$o->m(?)`, `C::m(?)`, in constant expressions too | parsed, typed (§5), emitted as written | lowered (§5.3) |
| **readonly property defaults** (RFC `readonly_property_defaults`) | allowed; a later assignment, including in the constructor, is an error, since the property is initialised | refused: moving the value into the constructor changes `ReflectionProperty::hasDefaultValue()`, `newInstanceWithoutConstructor()`, `unserialize()` and every constructor-less path |
| **`#[\Override]` on class constants and enum cases** (RFC `override_constants`) | the checker verifies that a parent declares the constant, as PHP does | the checker verifies it; the emitter drops the attribute, which 8.5 rejects at compile time. The check is all the attribute does, so dropping it is exact |
| **`__debugInfo()` on enums** (RFC `debugable-enums`) | allowed | refused |
| **Writes through a constant's object**, `CONFIG->prop = $v` (RFC `const_object_property_write`) | parsed and typed as a property write | refused |
| Reference assignment into a `WeakMap` without the key | nothing to do | nothing to do |
| `??` / `empty()` on a magic property no longer calling `__get()` after `__isset()` wrote it | nothing to do (runtime semantics) | — |
| **New library**: `clamp()`, `Time\Duration`, `SortDirection`, `Io\Poll\*`, the stream-error API, URI builders, `ReflectionProperty::isReadable()` / `isWritable()`, `ReflectionParameter::getDocComment()`, … | available | "needs PHP 8.6" (§6) |
| **Deprecations** (core) | reported | reported |

The deprecations, each a checker diagnostic in the deprecation group:
- `return` inside a `finally` block;
- a value returned from `__construct()` or `__destruct()`, and either one being a generator;
- `is` or `let` as the name of a class, interface, trait, enum, function or constant, and as an
  alias in `use` or `class_alias()`;
- `_` as a constant name or an import alias;
- a function named `readonly`;
- `namespace` as a class constant name;
- `define()`'s third argument;
- `is_a()` / `is_subclass_of()` with a string and `allow_string: false`;
- `?array` as `__debugInfo()`'s return type.

Library deprecations (`is_double()`, `spl_object_hash()`, the CSV methods of `SplFileObject`, …)
come from the stubs' `#[\Deprecated]`.

## 5. Partial function application

### 5.1 Syntax

As PHP 8.6 has it:
- `?` stands for one argument, and `...` for all the remaining ones;
- `name: ?` is a named placeholder;
- positional placeholders come before named ones, `...` comes last, and placeholders do not mix
  with unpacking.

The syntax works in calls to functions, methods, static methods and invocable values, and in
constant expressions (attribute arguments, parameter defaults, constants), beside the closures
`functions.md` already admits there. PHP's compile errors are Phaenon's syntax errors.

### 5.2 Types

A partial application is a `Closure(…): R`:
- **Parameters:** the parameters its placeholders stand for, in PHP's order (positional
  placeholders first, then named ones in the order written, then what `...` leaves), with their
  declared types.
- **Optionality:** a parameter is required when a `?` stands for it, and inherits its
  optionality when `...` stands for it.
- **Return:** the called function's return type, with the type parameters the bound arguments
  determine substituted. A type parameter that only a placeholder determines widens to its bound.
  Phaenon closure types are not generic, so `array_map(?, $xs)` is
  `Closure(Closure(T): mixed): list<mixed>` with `T` fixed by `$xs`.

**In a pipe, a partial application is typed as the call it becomes.** `A |> f(B, ?, C)` is checked
exactly as `f(B, A, C)`, so a generic `f` infers from `A` too, as the first-class callable
`A |> f(...)` already does. A closure written in a bound argument is typed from its parameter's
declared type, so `$users |> array_filter(?, fn ($u) => $u->active)` types `$u` as the element of
`$users`.

Errors PHP raises at run time are compile-time diagnostics:
- a placeholder count that does not match the parameters, PHP's `ArgumentCountError`
  ("Partial application of foo() expects exactly 3 arguments, 1 given");
- a bound argument whose type does not fit its parameter, PHP's `TypeError` at creation.

A placeholder may stand for a by-reference parameter: `sort(?)` is a closure taking its array by
reference. In a pipe it may not, because the piped value cannot be passed by reference (§5.4).

### 5.3 Lowering for an 8.5 target

Bound arguments are evaluated when the partial application is created, not when it is called
(php-src `rfc_examples_eval_order`). In a pipe, the left side is evaluated first, then the bound
arguments (`pipe_optimization_009`).

| Form | Lowered to | Exact because |
|---|---|---|
| `A \|> f(B, ?, C)`, with `B` and `C` literals or constants | `f(B, A, C)` | nothing is evaluated out of order |
| `A \|> f(B, ?, C)`, otherwise | `A \|> (fn (T $p) => f(B, $p, C))` | the closure is called at once, so `B` and `C` run after `A`, as in 8.6 |
| `f(B, ?)` outside a pipe, with every bound argument a literal, a constant or a variable | `fn (T $p) => f(B, $p)` | an arrow function captures variables by value when it is created |
| `f(B, ?)` outside a pipe, with another bound argument (a call, a property, an operator) | refused: "assign it to a variable first, or target PHP 8.6" | lowering would call it at each invocation, or need a name the programmer did not write |
| `f(B, ...)` | `fn (T2 $b, T3 $c = …) => f(B, $b, $c)`, the remaining parameters copied from the declaration; `fn (mixed ...$args) => f(B, ...$args)` for a variadic or internal function whose defaults are not expressible | same rules for the bound arguments |
| `f(b: ?)` | `fn (T $b) => f(b: $b)` | |

`$p` stands for the called function's parameter name (§3), and `T` for its declared type, written
so that the closure rejects what the partial application would reject; a by-reference parameter
keeps its `&`.

### 5.4 Pipes

The pipe operator is PHP 8.5's, and Phaenon keeps its meaning: the right side is a callable that
receives the left side. Partial application is what makes most pipelines need no arrow function:

```php
$ns
    |> sprintf('✓ namespace %s', ?)
    |> Style::line(...)
    |> $this->output->writeln(...);
```

The checker:
- **types a pipeline stage by stage**, including first-class callables of methods and partial
  applications;
- **reports at compile time what PHP reports at run time:**
  - a stage whose piped parameter is by reference: `$xs |> sort(...)` and `$xs |> sort(?)` are
    an `Error` on 8.5 and 8.6, "could not be passed by reference";
  - a stage that needs more than one argument;
  - a right side that is not callable;
- **warns** when a `void` stage feeds another stage.

Not adopted:
- a topic variable for the piped value (Hack's `$$`);
- first-argument insertion (Elixir's `|> f(1)` meaning `f($x, 1)`).

Either would make Phaenon's `|>` mean something other than PHP's, and partial application covers
the uses of both. For lists and strings, the method syntax (`$users->filter(…)->map(…)`,
`collections.md`) remains the left-to-right form; pipes serve free functions and static helpers.

## 6. The library across versions

The embedded stubs come from the newest supported release branch (`PHP-8.6`). A generated table
records the version each symbol appeared in, built by diffing the stub sets of the supported
release branches (`PHP-8.5`, `PHP-8.6`): functions, classes, methods, constants, and parameters
added to existing functions. A use of a symbol newer than the target is reported. Phaenon's
overlay adds types the stubs cannot express: `clamp<T>(T $value, T $min, T $max): T`, and `<` /
`>` between two `Time\Duration`, which PHP compares by value.

Extension stubs keep coming from the reflection of the local `php` (`internal/program`). A target
newer than that `php` is reported once, because the extensions' 8.6 signatures are then unknown.

## 7. Tests

- **The 8.6 oracle:** the conformance suites run the `PHP-8.6` corpus under PHP 8.6 (a variable
  next to `PHAENON_PHP_SRC`, in the vault's `testing` note).
- **Emitter goldens** are compiled for both targets where they differ, and each output must pass
  `php -l` and PHPStan level max under the version it targets.
- **Dogfood:** the Tequila trees stay on target 8.5 and exercise the lowering. tq-cli is also
  built with `"php": "8.6"`, and its suite runs on PHP 8.6.

## 8. What PHP is reserving: `is`

PHP 8.6 deprecates `is` and `let` as names, reserving `is` for a future use with `match`: the
pattern-matching proposal, `$x is <pattern>`. Phaenon's `is` (`generics.md`) is a type test with
the same spelling. It stays one, and it should remain **a compatible superset**: for the type
patterns PHP adopts (`int`, `Foo`, `int|string`, `?Foo`), `$x is T` means exactly what PHP's will
mean, so that the emitter can write PHP's own `is` once it exists. A Phaenon-only form (`list<T>`,
shapes) keeps lowering to `\Phaenon\Type`. When PHP's proposal is voted, it is weighed here
against this section.

## 9. Decisions made while building PHP 8.6

| Topic | Decision | Why |
|---|---|---|
| composer.json's floor outside 8.5…8.6 (2026-11-02) | Below 8.5, or no lower bound at all (`*`, `<9`): the target is 8.5 and `P4008` warns on `require.php`, as for a stated target above the floor. Above 8.6 (`>=8.7`): the error `P4009` on `require.php`, and the target clamps to 8.6 for `check` while `build` writes nothing | §2 makes an unknown target an error when `phaenon.json` states it. A floor composer.json states is a fact about installs, not a request to `phn`: compiling for 8.5 is what a project admitting 8.3 gets, and the warning says what an 8.3 install would meet. A floor above 8.6 means code `phn` cannot read |
| "with a note" (2026-11-02) | No target anywhere is the warning `P4010`, attached to `phaenon.json`, composer.json or the first file; an unreadable `require.php` (`dev-master`) is `P4010` too | `phn` has no severity below a warning, and adding one for a single message is not worth it; the fix is one line |
| Codes of the target (2026-11-02) | `P4008` target above composer.json's floor (warning), `P4009` target `phn` does not support (error), `P4010` no target stated (warning); `P1002` "… needs PHP 8.6 (target 8.5)" for a construct, the one syntax-family code §3 asks for | §3's split: constructs in the syntax family, the project file in the interop family beside `P4005` |
| `"roots"` optional (2026-11-02) | A `phaenon.json` may state only `"php"`; `build` and `check` need the roots only when run without paths | `phn migrate --php 8.6` creates the file when there is none, and inventing roots would move every output into a sibling directory, a layout change nobody asked for |
| PHP's compile errors around placeholders (2026-11-02) | Reported by the checker, not the parser: `P1003` for a variadic placeholder not last or twice, placeholders with unpacking, a closure from `new`, from `?->` or as an attribute's arguments; `P2110` for a positional placeholder after a named argument | The parser ports `zendparse`, which accepts them, and the conformance oracle (`TOKEN_PARSE`) counts such files as PHP-accepted: a parser that rejected them would fail its own conformance |
| `f(...)` in the tree (2026-11-02) | A list of `...` alone, with or without a trailing comma, stays `ArgList.FirstClassCallable`; any other placeholder is an `ast.Placeholder` on its argument | PHP 8.6 compiles the sole `...` as the first-class callable it was; every pass that knows first-class callables keeps working |
| Argument count of a partial application (2026-11-02) | `P2101` and `P2102` with PHP's wording ("partial application of foo() expects exactly 3 arguments, 1 given"); a named placeholder for no parameter, or overwriting an argument, is `P2010` | One mistake, one diagnostic: these are the mistakes the codes already name |
| Pipe diagnostics (2026-11-02) | A stage needing more than one argument is `P2101` and a non-callable right side `P2109`, as in any call; a by-reference parameter receiving the piped value is the new `P2114` | `A \|> f(...)` is the call `f(A)` and PHPStan reports both as for calls (`arguments.count`, `callable.nonCallable`). `P2112` says "is not a variable", wrong for `$xs \|> sort(...)`; PHPStan has its own identifier for it (`pipe.byRef`) |
| A `void` stage feeding another (2026-11-02) | `P2205`, an error, not the warning §5.4 says | `$x \|> f(...) \|> g(...)` is the program `g(f($x))`, already `P2205`, and PHPStan reports both at level max (`function.void`): a warning would be more lenient than PHPStan |
| Pipes typed as calls (2026-11-02) | A first-class callable or a partial application with one placeholder on the right side is checked as the call PHP turns it into, for functions, methods and static methods; any other right side as `($stage)(A)` | It is what `zend_compile_pipe` compiles, so inference, closures in bound arguments and by-reference parameters behave as at run time |
| Refusals beyond §5.3's table (2026-11-02) | Also refused for 8.5 (`P1002`): a bound argument passed by reference; in a pipe, a bound argument that assigns; a bound variable named like a parameter of the closure; outside a pipe, a computed receiver or callee; `...` after a named argument when it would unpack | PHP binds by reference what an arrow function captures by value; an assignment in the closure changes its copy; renaming the parameter would add a name the programmer did not write (D9); `f(a: 1, ...$args)` is a compile error |
| The direct pipe form (2026-11-02) | `A \|> f(B, ?, C)` becomes `f(B, A, C)` only for a named function or a static method of a named class, a left side on one line with nothing else lowered in it; otherwise the closure form, which is exact too | The emitter keeps every line's number; moving a multi-line left side into the call would shift lines, and a receiver evaluated before `A` could see what `A` changed |
| Refined parameter types in a lowered closure (2026-11-02) | A parameter whose type says more than PHP declares (`list<int>`) makes the closure `function (array $xs) use ($v) { /** @var list<int> $xs */ return f($xs, $v); }` | The output must pass PHPStan at level max, and PHPStan reads no docblock on a closure, only an inline `@var` in a function body. On one line PHPStan reads the first tag only, so a second refined parameter goes unread (vault `open-items`) |
| Partial applications in constant expressions (2026-11-02) | Lowered to `static function (T $p) { return f(B, $p); }` | PHP 8.5 accepts static closures there and rejects arrow functions |
| PHPStan on 8.6 output (2026-11-02) | The emitter goldens run PHPStan on 8.5 output only when the source holds placeholders; the 8.6 output is checked with `php -l` under PHP 8.6 and by running both outputs | The parity PHPStan (2.2) does not parse PHP 8.6's placeholders yet |
