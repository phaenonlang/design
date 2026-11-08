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
| `A \|> f(B, ?, C)` or `A \|> f(...)` that folds (below) | `f(B, A, C)`, `f(A)`; a run of them as nested calls | nothing is evaluated out of order |
| `A \|> f(B, ?, C)`, otherwise | `A \|> (fn (T $p) => f(B, $p, C))` | the closure is called at once, so `B` and `C` run after `A`, as in 8.6 |
| `f(B, ?)` outside a pipe, with every bound argument a literal, a constant or a variable | `fn (T $p) => f(B, $p)` | an arrow function captures variables by value when it is created |
| `f(B, ?)` outside a pipe, with another bound argument (a call, a property, an operator) | refused: "assign it to a variable first, or target PHP 8.6" | lowering would call it at each invocation, or need a name the programmer did not write |
| `f(B, ...)` | `fn (T2 $b, T3 $c = …) => f(B, $b, $c)`, the remaining parameters copied from the declaration; `fn (mixed ...$args) => f(B, ...$args)` for a variadic or internal function whose defaults are not expressible | same rules for the bound arguments |
| `f(b: ?)` | `fn (T $b) => f(b: $b)` | |

`$p` stands for the called function's parameter name (§3), and `T` for its declared type, written
so that the closure rejects what the partial application would reject; a by-reference parameter
keeps its `&`.

**A pipe folds** when PHP evaluates it as the call it makes: `A` goes by value, and the callee and
the arguments written before the placeholder compute the same before `A` as after it. An argument
does when it is a literal, a constant, a variable `A` does not write, a closure that captures none
of those by value, or a first-class callable of a named function; a callee when it is a named
function (not `compact()`, `extract()`, `func_get_args()` or `assert()`), a static method of a named
class, or a method of `$this` or of a variable `A` does not write. `A` writes a variable by
assigning it, capturing it by reference, or passing it to a parameter the checker cannot show to
be by value. Arguments after the placeholder are evaluated after `A` either way. The checker
decides it for every pipe (the same rule the `pipes` codemod applies in the other direction,
`tooling.md` §8.1) and the emitter writes what it decided:

```php
$users |> array_map(fn (User $u) => $u->email, ?) |> array_filter(...) |> array_values(...)
// 8.5 →
array_values(array_filter(array_map(fn (User $u) => $u->email, $users)))
```

- **A method of a property does not fold**: 8.6 reads the property after `A`, and a call in `A`
  may assign it. `implode(', ', $rows) |> $this->log->line(...)` stays a pipe on 8.5.
- **A first-class callable after a pipe that stays one stays a stage** too — PHP 8.5 runs it as
  written, and `A |> (fn($x) => …) |> f(...)` reads better than `f(A |> (fn($x) => …))`.
- **Layout.** A folded run goes on one line when that fits in 120 columns (a closure over several
  lines keeping its lines), otherwise one argument per line with a trailing comma, PER's
  multi-line argument list, a call around a single inner call staying on its line
  (`array_values(array_filter(` … `))`). Its lines map to the pipeline's first line
  (`line-map.md`).
- **A run that cannot move stays a pipeline**: a comment between its stages or arguments, or a
  multi-line string that would move sideways.

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
- **reports** a `void` stage feeding another stage (`P2205`, an error: §9).

Not adopted:
- a topic variable for the piped value (Hack's `$$`);
- first-argument insertion (Elixir's `|> f(1)` meaning `f($x, 1)`).

Either would make Phaenon's `|>` mean something other than PHP's, and partial application covers
the uses of both. Since D20 pipes are the left-to-right form for every value, lists and strings
included — the method syntax is gone — and `phn migrate` writes them (`tooling.md` §8).

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

*Amended 2026-11-09 (roadmap D26): Phaenon has no `is` of its own any more, so there is nothing
to keep compatible. `is` is a name again, deprecated as PHP 8.6 deprecates it (§9). When a PHP
release ships `$x is <pattern>`, the construct is Phaenon's from that release with PHP's meaning
(§1), and the checker narrows through it as through the tests it stands for; it is weighed here
when the proposal is voted, as any PHP addition is.*

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
| Where the stubs come from (2026-11-02) | The embedded stubs are every `*.stub.php` of the newest supported release branch, `PHP-8.6`, written by the generator `stdlib/gen` (`go generate ./stdlib`), which reads php-src through `git archive` only. They had been a copy of `master` | A release branch is what users run; `master` already carries the next release (`connection_aborted(): bool`, readonly Reflection names, `IntlNumberFormatter`), which an 8.6 program would be typed against |
| What the version table holds (2026-11-02) | Generated Go (`stdlib/versions_gen.go`): the functions, classes (interfaces, traits, enums), global constants, and the methods, class constants and enum cases of classes the floor already has, that 8.6 added; parameters added to existing functions and methods; the `#[\Deprecated]` attributes 8.6 added. Not private members, not the members of a new class, not php-src's test extensions (`zend_test`, `dl_test`, `skeleton`). PHP 8.6 removes no symbol | A new class's members come with it; a private member is no API. Deprecations are reported from the stubs' attribute whatever the target, so the table lists them for reference only |
| The library's codes (2026-11-02) | `P4011` (error), the one interop code §3 asks for: "clamp() needs PHP 8.6 (target 8.5)", "method ReflectionProperty::isReadable() needs …", "parameter $salt_length of openssl_sign() needs …". `P4012` (warning), §6's notice: the local php whose extensions are reflected is older than the target | A use PHP 8.5 cannot run is an error, as the unknown function it would otherwise be; the stale extension signatures are a risk, not a mistake in the code |
| Where `P4011` applies (2026-11-02) | Calls, `new`, static access, class constants, `extends` and `implements`, attributes, declared types; not `X::class`, `instanceof X` or `catch (X)` | PHP 8.5 fails on the first set (on a declared type, at every call); the second fails nothing and is how code guards what 8.6 adds (`class_exists(Duration::class)`). PHPStan on 8.5 takes them for misspelt names; the parity test ignores that, with the reason |
| Polyfills (2026-11-02) | A function, constant or class that project or vendor code also declares is available on every target; the stub's types still apply | A polyfill package (`symfony/polyfill-php86`) makes the symbol exist on 8.5; reporting it would refuse code that runs |
| Version guards (2026-11-02) | No `P4011` inside a branch that `function_exists('f')`, `class_exists(C::class)` and kin, `method_exists($x, 'm')`, `defined('C')` or a `PHP_VERSION_ID` comparison proves, in `else` branches of the negated forms, nor after an early exit such as `if (PHP_VERSION_ID < 80600) { return …; }`; literal names and ids only | Code for several PHP versions tests for what it uses, and PHPStan reads `function_exists()` the same way. Guards in expressions: next row |
| Guards in expressions (2026-11-06) | Read as in `if`: the arms of `?:` (the first under the condition, the second, and the default of `c ?: d`, under its negation), the right operand of `&&` under the left one and of `\|\|` under the left's negation, in a condition as in a value; the arms of `match (true)`, each condition under the negations of those before it, an arm's body under its condition when it has one, `default` under the negation of every condition | They are the same branches written as expressions: `PHP_VERSION_ID >= 80600 ? clamp(…) : …` and `function_exists('clamp') && clamp(…)` run `clamp()` only where it exists. An arm with several conditions runs when any holds, so it proves none of them alone |
| `#[\Override]` and the target's library (2026-11-02) | An `#[\Override]` method whose only parent is a method the target lacks (`SessionHandler::validateId()` on 8.5) is `P4011` | PHP 8.5 checks the attribute when it loads the class and finds nothing to override: a fatal error |
| Library deprecations (2026-11-02) | Read from `#[\Deprecated]` (stubs and code) as well as `@deprecated`, on functions, methods, constants, class constants and enum cases; a stub's attribute holds when the overlay retypes the declaration; `P4006` takes PHP's wording: "function is_double() is deprecated since 8.6, use is_float() instead" | The stubs' 196 attributes went unread, so no library deprecation was reported; PHP's own message is the one a developer will meet in logs |
| The language deprecations' code (2026-11-02) | One code, `P1101` (warning), in a new group P11xx for constructs PHP deprecates; a message per construct; reported whatever the target, behind `--no-deprecations` | Deprecated constructs are removals announced, the syntax family's matter; one code for the family, as `P4006` is one for the library |
| Lines of the deprecations (2026-11-02) | PHP's, except a generator constructor or destructor: reported on each `yield`, where PHPStan reports it, not on the method's line where PHP does | The `yield` is what makes the method a generator, and the line PHPStan flags must carry the diagnostic |
| `is` as a name (2026-11-02) | *(Superseded by the next row, 2026-11-09.)* Reported only where it reaches PHP as a string, through `define()` and `class_alias()`; `let` everywhere | Phaenon parses `is` as the type test (§8), so declaring or importing a name `is` is already a syntax error |
| `is` as a name (2026-11-09) | Reported everywhere, as `let` is: a class, interface, trait, enum, function or constant named `is`, and an alias in `use` or `class_alias()`, is `P1101` | Roadmap D26 withdrew the type test, so `is` is an identifier again and the declarations PHP 8.6 deprecates compile; the deprecation is reported whatever the target (D17) |
| `is_a()` with a class name (2026-11-02) | `P1101` when the first argument's type admits a string (mixed aside) and `$allow_string` is false or, for `is_a()`, omitted | `is_a()`'s default is false; PHP 8.6 raises the deprecation when a string reaches it, `is_a('K', K::class)` included |
| Comparing two `Time\Duration` (2026-11-02) | No overlay: the checker admitted comparisons of objects already, as PHP (through the class's handler) and PHPStan do. What it lacked was an object compared with an int or a float, now `P2301` (GMP and `BcMath\Number` aside), in conditions as elsewhere | §6 assumed object comparisons were refused. PHP converts the object to a number with a Notice, and PHPStan reports it (`smaller.invalid`, `equal.invalid`, `spaceship.invalid`) |
| The rest of the 8.6 library (2026-11-02) | Overlay types for `clamp<T>(T, T, T): T` and for the event lists of `Io\Poll\Context::add()` and `Watcher::modify*()`, `non-empty-array<Event>`; the rest is typed by its stubs | `clamp()` returns one of its arguments unchanged (`clamp(1.5, 0, 1)` is the int `1`); PHP 8.6 throws a TypeError for any other events array, the empty one included |
| Readonly defaults and `__clone` (2026-11-02) | On an 8.6 target, assigning a readonly property that has a default is `P2206` in the constructor as elsewhere, but not in `__clone`; on an 8.5 target the default alone is reported (`P1002`) | PHP 8.6 lets `__clone` reinitialise any readonly property (the 8.3 rule) and throws in the constructor; on 8.5 the default is the mistake, and the assignment is what fixing it leads to |
| Readonly defaults and the constructor's duty (2026-11-06) | A readonly property with a default needs no assignment in the constructor (`P2515`, `checker.md` §9); on an 8.5 target its default is `P1002` and the property is not reported as uninitialised too | On 8.6 the default initialises it before the constructor runs; on 8.5 the default is the one mistake |
| `#[\Override]` on constants: the code (2026-11-02) | `P2503`, the methods' code, titled now "#[Override] on a member that overrides nothing"; a parent's private constant does not count; a trait's constant is judged in each class using it; reported on the constant, where PHP names the class | The same mistake as for a method; PHP's link-time check behaves so |
| `#[\Override]` on properties (2026-11-06) | Checked as PHP 8.5 checks it, with `P2503`: a parent class's property that is not private, or an interface's, static or not, promoted properties included; a class's own property redeclaring a trait's overrides nothing; a trait's property is judged in each class using it. Not checked: `#[\Override]` on a property hook, which names a parent's hook | PHP 8.5, the floor, added the attribute's property target, and PHPStan 2.2 reports the same cases (`property.override`); a hook's attribute is a method's, against hooks the checker does not model per parent |
| Magic methods on enums (2026-11-02) | Every magic method but `__call`, `__callStatic`, `__invoke` and, from 8.6, `__debugInfo` is `P2514` on an enum, like its properties; `__debugInfo()` is `P1002` on an 8.5 target | The checker said nothing, where PHP refuses them at compile time and PHPStan reports them (`enum.magicMethod`) |
| `constants.php` (2026-11-02) | Regenerated by `stdlib/gen -php` from PHP 8.6.0-dev, the procedure that reproduces the 8.5.3 file; to be regenerated from 8.6.0 | The values are the newest release's; the 8.6 constants had none |
| PHPStan parity for 8.6 goldens (2026-11-02) | A golden whose header says `// target: 8.6` is analysed by PHPStan running on a php 8.6 (`PHAENON_PHP86`, or `PHAENON_PHP` when it is one); findings that are PHPStan 2.2's gaps are ignored per file, each with its reason | PHPStan 2.2 knows 8.6's new functions and classes only by reflection, not the members 8.6 added to existing ones, and refuses readonly defaults, `#[\Override]` on constants and `__debugInfo()` on enums on every version |
