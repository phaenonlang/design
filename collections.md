# Collections: `list<T>`, `map<K, V>` and the legacy `array` — M4 design (draft)

Status: **reviewed 2026-10-30**; §9 holds the decisions, the body is the design they settle.
**Superseded in part by D20 (2026-11-03)**: the stdlib `\Phaenon\Seq` / `\Phaenon\Map` (§3) and
the method-call syntax (§3.1) are gone from the language. The types, literals, tuples, interop,
migration and shapes (§2, §4–§6) stand; lists and maps are transformed with PHP's functions and
chained with pipes and partial application (`roadmap.md` D20, `php-versions.md` §5.4).
**Amended 2026-11-05** (§7): `array<V>`, a bare `array`, shapes and the docblock vocabulary.
**Amended 2026-11-15** (§6, §7; `roadmap.md` D29): a shape gets syntax with a name, in a `type`
declaration, and an alias belongs to its namespace; built in the compiler (§6.9) and in both
editors (`tooling.md` §6.2, §6.6) the same day.
Nothing is implemented yet beyond what M3 already has (the keywords `list` and `map` in type
positions, their erasure to `array`, the lattice's array types).

## 0. One paragraph

PHP has one collection type, `array`, that is at once an ordered sequence and a dictionary, and
every PHP type checker spends half its vocabulary (`list<T>`, `array<K, V>`, `non-empty-list`,
shapes) telling the two apart in docblocks. Phaenon makes the distinction the type system's:
`list<T>` is an ordered sequence, `map<K, V>` a dictionary, and both **are PHP arrays at
runtime** — nothing is boxed, copy-on-write value semantics and every `array_*` function stay
exactly as they are. `array<mixed>` is the honest spelling for "a PHP array I know nothing more
about", the way `mixed` is for values; a bare `array` names no element type at all. On top sit a stdlib with a predictable
argument order (`\Phaenon\Seq`, `\Phaenon\Map`) and method-call syntax on collection values
that compiles to it, so `$users->filter(fn (User $u) => $u->active)` reads like a language with
collections and runs like PHP.

## 1. Goals and non-goals

### Goals

- **The dual nature ends at the type level.** A declaration says `list<Event>` or
  `map<string, Handler>`; the checker knows the difference everywhere PHPStan does and in a few
  places it does not (literals, mutations, iteration).
- **Zero runtime cost, total interop.** A `list<T>` *is* an array: passed to `array_values`, to
  Symfony, to `json_encode`, to MongoDB, received back from any PHP API typed `array` or with a
  PHPStan docblock. Roadmap D1: no new runtime.
- **Tequila's docblocks become syntax.** Across Tequila's `src/` trees `list<…>` appears on 2,007
  lines and `array<…>` on 1,959 (generics.md §11.2). `phn migrate` already lifts both; M4 makes
  the result pleasant to work in.
- **A stdlib a person can guess.** Collection first, callback last, the same names on lists and
  maps where the operation is the same, and method-call syntax so the class name never needs
  typing (roadmap M4; the string half is `strings.md`).

### Non-goals (v1)

- A collection *class library* (Doctrine Collections, Laravel's `Collection`, `Ds\*`): classes
  would box the array, break interop with every `array` API and change value semantics.
- Immutable or persistent collections, typed sets, ordered maps beyond what PHP arrays already
  are. `Set<T>` is a `map<T, true>` idiom until a measured need appears.
- New *runtime* representation or behaviour of any kind. Everything in this document erases to
  arrays and `\Phaenon\*` static calls.

## 2. The types

| Phaenon | What it is | Erases to | Docblock |
|---|---|---|---|
| `list<T>` | keys `0 .. n-1` in order, values `T` (PHPStan's `list<T>`) | `array` | `list<T>` |
| `map<K, V>` | keys `K ⊆ int\|string`, values `V`, any order | `array` | `array<K, V>` |
| `array<V>` | a PHP array of `V`, list or keyed: the type `map<int\|string, V>`, spelled for "the keys do not matter" | `array` | `array<V>` |
| `array` | no element type written: in an override, the type of the method it overrides; elsewhere in a `.phn` declaration, `P2516`. A PHP API typed `array` gives `array<mixed>` | `array` | none |
| `non-empty-list<T>` / `non-empty-map<K, V>` | refinements the checker *infers* (`count($xs) > 0`, a literal, `$xs[] = …`) and reads from docblocks; written natively as `list<T>` / `map<K, V>` | `array` | `non-empty-list<T>` / `non-empty-array<K, V>` |
| `array{a: int, b?: string}` (shape) | native syntax **in the right-hand side of a `type` declaration and nowhere else** (2026-11-15, §6; not native syntax at all from M4 until then); read from docblocks in `.php` and `.phn` alike, as today. A static type, erased at compile time | `array` | the shape |
| `iterable<K, V>` | `map<K, V>\|Traversable<K, V>` (M3) | `iterable` | `iterable<K, V>` |

*2026-11-15 (roadmap D30):* `list` and `map` are written in lower case. `List<int>` in a type
position is still a list, PHP reserving the word for its own `list()`, and is `P1006`, which `phn
fmt` rewrites. `map` is a name PHP leaves free, so `Map<string, int>` is a class name like any
other: the class, interface, enum or alias of that name, or, where none resolves, `P2001` saying
that Phaenon's type is written `map`. A lower-case `map` is Phaenon's type even in a file that
imports a class `Map`, which that file names `Map`. In a `.phn` docblock (§4) the translation
for PHPStan reads `map<…>` and `non-empty-map<…>` in lower case only; `List<int>` there is
PHPStan's own word, which PHPStan reads in any case.

Decisions proposed with the table:

- **`map`'s key bound is `int|string`**, implicit: `map<K, V>` without a bound on `K` in a
  generic declaration means `K of int|string`; `map<bool, V>` is a type error (`P2518`). PHP
  would cast the key; Phaenon refuses the type. `array-key` stays docblock vocabulary and is
  written `int|string` when a key type is meant generically (generics.md §11.2).
- **A key whose type is known to be numeric is an int key** (2026-11-01, checker.md §9): PHP
  stores `'42'` as `42`, so a literal integer string, a `decimal-int-string` (what `(string)` of
  an `int` is) and the int part of a `numeric-string` are int keys, in writes, reads, literals
  and `array_combine`-style results; `$out[(string) $k] = $v` with `$k: int|string` builds a
  `map<int|string, …>`. A general `string` key stays a `string` key, a hole kept on purpose
  for the reasons checker.md §9 gives.
- **Subtyping.** `list<T> <: map<int, T> <: array`; `list<T>` is covariant in `T`; `map<K, V>` is
  covariant in **both** `K` and `V`. Arrays are values (copy-on-write), there is no aliasing
  through which a covariant key could be abused, so `map<'a'|'b', Admin> <: map<string, User>`
  is sound. This answers generics.md §3.3's open question (PHPStan agrees in practice: it treats
  array key and value types covariantly).
- **A bare `array` in a declaration names no element type** (2026-11-05; the first form of this
  rule called it deliberate and fine).
  - In an override it takes the docblock type of the method it overrides, as PHPStan reads it.
  - Anywhere else in a `.phn` file it is `P2516`, the error for a parameter with no type at all:
    "`mixed` must be written, never inferred silently" (language-overview.md §3), and
    `array<mixed>` is the explicit "any array". A bare `iterable`, and an `array` inside another
    type (`list<array>`), are the same.
  - Why the change: the emitter writes no docblock for a bare `array`, so PHPStan at level max
    reported `missingType.iterableValue` on the output of a source `phn check` had passed. The
    emitted PHP must pass level max.
  - Passing an `array<mixed>` where a `list<T>` or `map<string, V>` is expected is an error, as
    PHPStan reports it at level 7+. Narrowing is PHP's tests (`is_array()`, `array_is_list()`), a
    shape docblock, a `@var` tag, or a guard function with `@phpstan-assert` (2026-11-09, roadmap
    D26; until 2026-11-08 also `as`, `$config as map<string, mixed>`, a runtime check through
    `\Phaenon\Type`).
- **`array<V>` is the spelling of "an array of `V`, list or keyed"** (2026-11-05). It is the type
  `map<int|string, V>`, which includes every list since `list<T> <: map<int, T>`; but `map` reads
  as a dictionary, and the point of this spelling is that the keys do not matter. It is PHPStan's
  own one-argument form, so the docblock is the same text. It takes one argument: with a key type,
  it is a `map<K, V>`.

### 2.1 Literals

Literals already type as shapes in the checker (checker.md §3.1); M4 fixes the rules a
programmer sees:

| Literal | Type | Widens to |
|---|---|---|
| `[]` | the empty array: both a `list{}` and a `map` with no keys | whatever the context asks: `list<T>` or `map<K, V>` |
| `[1, 2, 3]` | `list{1, 2, 3}` | `list<int>` |
| `['a' => 1, 'b' => 2]` | `array{a: 1, b: 2}` | `map<string, int>` |
| `[1, 'a' => 2]` | a shape with mixed keys | `map<int\|string, int>` |
| `[...$xs, ...$ys]` | `list<T>` when both are lists, `map` otherwise | — |

Widening happens where a shape meets a declared type: assignment to a typed property or
parameter, `return`, a join in flow analysis. A shape stays a shape as long as the checker can
track it, which is what makes `$opts['limit']` precise after `$opts = ['limit' => 10]`.

### 2.2 Mutation

The checker follows PHPStan, whose rules match what the runtime does:

| Statement | On `list<T>` | On `map<K, V>` |
|---|---|---|
| `$xs[] = $v` | stays `non-empty-list<T\|V>` | `non-empty-map<int\|K, V\|V'>` (PHP appends with the next int key) |
| `$xs[$i] = $v`, `$i: int` | becomes `non-empty-map<int, T\|V>` — a gap may have appeared | stays a map |
| `$xs[$k] = $v`, `$k: string` | becomes `non-empty-map<int\|string, …>` | stays a map, `K\|string` |
| `unset($xs[$i])` | becomes `map<int, T>` | stays a map |
| `array_values($xs)`, `array_map`, `array_filter(list)` | `list<T>` · `list<U>` · `map<int, T>` (keys survive `array_filter`) | per the overlay |

A list that lost its shape is still a PHP array, so nothing breaks at runtime; the type just
tells the truth. `\Phaenon\Seq::values($xs)` (or `array_values`) is how a list is reasserted.
`$xs[$i] = $v` with `$i` provably in `0 .. count($xs)` keeps the list — the checker knows
`count()` ranges and literal indexes.

### 2.3 Iteration, destructuring, indexing

- `foreach ($list as $i => $v)` types `$i` as `int<0, max>` and `$v` as `T`; `foreach ($map as
  $k => $v)` types `$k` as `K`.
- `[$a, $b] = $pair` requires a list (or a shape with keys 0 and 1); `['x' => $x] = $point`
  requires a map or shape with that key.
- `$list[$i]` is `T` when `$i` is known in range (a literal index into a shape, or a guarded
  `count()`), `T|null`-and-a-warning otherwise? **No:** PHPStan reports a possibly missing offset
  only with `reportPossiblyNonexistentGeneralArrayOffset`, which Tequila does not set, and
  PHP returns `null` with a warning. Phaenon keeps PHPStan's default: `$list[$i]` is `T`, and
  `$list[$i] ?? $default` is the idiom for "maybe". Revisit after `core` (open question §7).

### 2.4 Tuples

`(A, B)` is a **tuple type**: a positional, fixed-length list whose elements have their own types.
Fixed means exact: `(string, int)` is two elements, and a value with three is not one, as a
tuple in Python is not. (PHPStan's `list{string, int}` admits a longer value; the checker is
stricter there, which parity allows. A docblock shape's optional entry, `list{int, 2?: bool}`,
still lets a shorter value in.)
It is spelled with parentheses in any type position (`list<(string, int)>`, `(User, bool)`,
`?(int, int)`), built with an array literal (`[$name, $age]`), read by destructuring
(`[$name, $age] = $pair`) or by index (`$pair[0]`), and erases to the shape PHPStan writes:
`array{string, int}` in docblocks, `array` natively. In the lattice it *is* the sealed positional
shape `array{0: A, 1: B}`, which the checker already has, so `zip`, `toList` and every PHP API
that returns a pair-shaped array interoperate without conversion. A tuple is at least two
elements long; `(A)` is a parenthesised type.

*A type only (2026-11-09, roadmap D26).* Nothing tests a tuple at runtime: a value is a tuple
because the code that built it says so, as in PHP, and a test the programmer writes out
(`count($pair) === 2`, `is_int($pair[0])`) narrows as PHPStan narrows it.

## 3. The stdlib: `\Phaenon\Seq` and `\Phaenon\Map`

Written in PHP in the runtime package (`phaenon/runtime`), with full Phaenon signatures in their
docblocks so the checker types every call precisely; the stub embedded in the compiler is
generated from the package (closing [[gotcha-runtime-stub-drift]]). Names follow Go's `slices`
and `maps` packages: a plural noun per collection kind, functions named for the operation.

Argument order, everywhere: **the collection first, then the operands, callbacks last.**

```php
namespace Phaenon;

final class Seq
{
    public static function map<T, U>(list<T> $xs, Closure(T): U $f): list<U>;
    public static function filter<T>(list<T> $xs, Closure(T): bool $keep): list<T>;
    public static function reduce<T, A>(list<T> $xs, A $initial, Closure(A, T): A $f): A;
    public static function first<T>(list<T> $xs): ?T;
    public static function last<T>(list<T> $xs): ?T;
    public static function contains<T>(list<T> $xs, T $needle): bool;
    public static function indexOf<T>(list<T> $xs, T $needle): ?int;
    public static function slice<T>(list<T> $xs, int $offset, ?int $length = null): list<T>;
    public static function concat<T>(list<T> ...$lists): list<T>;
    public static function reverse<T>(list<T> $xs): list<T>;
    public static function sort<T>(list<T> $xs, ?Closure(T, T): int $cmp = null): list<T>;
    public static function sortBy<T, K of int|string>(list<T> $xs, Closure(T): K $key): list<T>;
    public static function unique<T of int|string>(list<T> $xs): list<T>;
    public static function flatten<T>(list<list<T>> $xs): list<T>;
    public static function chunk<T>(list<T> $xs, int $size): list<list<T>>;
    public static function zip<A, B>(list<A> $as, list<B> $bs): list<(A, B)>;        // a tuple, §2.4
    public static function keyBy<T, K of int|string>(list<T> $xs, Closure(T): K $key): map<K, T>;
    public static function groupBy<T, K of int|string>(list<T> $xs, Closure(T): K $key): map<K, list<T>>;
    public static function values<T>(map<int|string, T> $xs): list<T>;   // the way back from a map
    public static function isEmpty(list<mixed> $xs): bool;
    public static function count(list<mixed> $xs): int;   // alias of count(); exists for the method form
}

final class Map
{
    public static function get<K of int|string, V>(map<K, V> $m, K $key): ?V;
    public static function has<K of int|string>(map<K, mixed> $m, K $key): bool;
    public static function keys<K of int|string>(map<K, mixed> $m): list<K>;
    public static function values<V>(map<int|string, V> $m): list<V>;
    public static function map<K of int|string, V, U>(map<K, V> $m, Closure(V, K): U $f): map<K, U>;
    public static function filter<K of int|string, V>(map<K, V> $m, Closure(V, K): bool $keep): map<K, V>;
    public static function merge<K of int|string, V>(map<K, V> ...$maps): map<K, V>;   // later keys win, as `+` reversed
    public static function with<K of int|string, V>(map<K, V> $m, K $key, V $value): map<K, V>;
    public static function without<K of int|string, V>(map<K, V> $m, K ...$keys): map<K, V>;
    public static function toList<K of int|string, V>(map<K, V> $m): list<(K, V)>;
    public static function fromList<K of int|string, V>(list<(K, V)> $pairs): map<K, V>;
    public static function isEmpty(map<int|string, mixed> $m): bool;
}
```

Every function is a thin wrapper over the `array_*` function or loop a PHP developer would have
written; semantics are PHP's (copy on write, `===` for `contains`, stable sort). The emitter may
later inline the trivial ones (`Seq::map` → `array_map` with the arguments swapped) — an
optimisation, not a semantic question, and not in M4.

### 3.1 Method-call syntax

A method call on a value whose static type is a `list`, a `map` (or a `string`, see
`strings.md`) resolves to the stdlib class of that type and compiles to the static call:

```php
$active = $users->filter(fn (User $u) => $u->active)->map(fn (User $u) => $u->email);
//  →   Seq::map(Seq::filter($users, fn (User $u) => $u->active), fn (User $u) => $u->email)
//      with `use Phaenon\Seq;` imported by the compiler (roadmap D11)
```

Rules:

- The receiver's static type decides, at compile time. `->` on an array or a string is a runtime
  error in PHP, so no PHP program changes meaning.
- A receiver typed as a union of a collection and an object (`list<int>|Countable`) is an error
  (`P2519`): narrow first. `array` (legacy) has no methods: say `list` or `map`.
- Only the stdlib's methods exist; there are no user-defined extension methods in M4.
- The result type is the function's return type, so chains type through.
- **A receiver that may be null** — `$s?->length()`, or `$u?->name->upper()` where PHP would
  skip the whole chain — compiles to a guard: `null === $s ? null : Str::length($s)` (in the
  else branch the chain's `?->` are plain `->`, it is known non-null there). The receiver is
  written twice, so it must be a path with no side effects: a variable, a property or constant
  path, an index by a literal or a variable. A call in the receiver is `P2519`: assign it to a
  variable first. (Decided 2026-10-30: a hidden temporary would be a variable the programmer did
  not write, D9; the guard reads as PHP a person writes.)
- A `\Stringable` receiver has the `Str` methods after its own: `$id->startsWith('u-')` compiles
  to `Str::startsWith($id, 'u-')` (the `Str` functions take `string|\Stringable`) when the class
  declares neither `startsWith()` nor `__call()`. A `string|Id` union is a string receiver.

The method form is sugar, nothing more: `Seq::map($xs, $f)` and `$xs->map($f)` are the same
call, and PHP code reads the output as the static call.

## 4. Interop with PHP code

- **To PHP:** a `list<T>` or `map<K, V>` parameter erases to `array`, with the docblock PHPStan
  reads (`list<T>` / `array<K, V>`), so PHPStan on the output keeps the distinction. PHP callers
  pass arrays; the checker's `--runtime-guards` (generics.md §6, not built) would be where a
  boundary check could go *(withdrawn 2026-11-08, generics.md §11.4)*.
- **From PHP:** a vendor function returning `array` is `array` (legacy); one with a docblock is
  typed by it, as today. The overlay keeps teaching the checker what PHPStan knows by extension
  (`array_map` preserving lists, `explode` returning a non-empty list, …).
- **`array_*` functions stay callable** from `.phn`; nothing is removed. The stdlib exists for
  the argument order and the method form, not to forbid anything.
- **Docblocks in `.phn` speak Phaenon** (decided 2026-11-05; M4 named the translation and did not
  build it). A docblock is where shapes live (§6), so it must hold every Phaenon type. The emitter
  rewrites Phaenon vocabulary in user docblocks into PHPStan's, in place:
  - `map<K, V>` → `array<K, V>`, `non-empty-map` → `non-empty-array`;
  - `class<T>` → `class-string<T>`;
  - a tuple `(A, B)` → `array{A, B}`;
  - `Closure(A): R` → `\Closure(A): R`, which PHPStan would otherwise resolve in the file's
    namespace;
  - the name of a `type` alias → the type it stands for (2026-11-15, §6.5), since PHPStan knows
    no alias of a namespace.

  *Amended 2026-11-15 (§6):* a shape with a name lives in a `type` declaration. The docblock is
  where the shapes without one live, and it may name an alias.

  The docblock parser reads tuples. Before, `phn check` passed `@param array{by: map<string,
  int>} $x`, and PHPStan on the output reported `App\map` as an unknown class; a tuple did not
  parse, and its tag was dropped with a warning.

## 5. Migration

`phn migrate` already lifts `@param list<T>` / `array<K, V>` / `T[]` into `list` / `map`
(generics.md §11.1). M4 adds nothing to the lift; it decides what the lift leaves behind:

- `V[]`, `array<V>`, `array<array-key, V>` and `array<int|string, V>` become `array<V>`; an
  `array<K, V>` with any other key becomes `map<K, V>`; `array-key` elsewhere is written
  `int|string` (2026-11-05: `array-key` used to keep its tag, with the note "no Phaenon
  spelling").
- A bare `array` with no docblock type becomes `array<mixed>`, unless the declaration overrides one
  whose type it inherits (2026-11-05). The opt-in codemods narrow first what they can prove
  (tooling.md §8.2).
- Shapes stay in docblocks (§6). *Amended 2026-11-15:* a shape that a `@phpstan-type` or
  `@psalm-type` tag names is lifted with its name into a `type` declaration; a shape written
  inline in a `@param`, `@return` or `@var` stays a docblock, since the tool cannot invent a name
  (§6.6).
- The `array_*` → `Seq::`/`Map::` codemod is M6's `phn migrate` v1 ("where provable"), not
  M4.

Tequila `core`, the M4 target, in numbers (315 files, 19,909 lines in `src/`): 141 docblock
`array<…>` tags against 66 `list<…>`, 35 shapes, `array_values` 29 calls, `array_map` 18,
`array_key_exists` 14, `array_filter` 9; `sprintf` 234 calls (strings.md). `core` runs PHPStan
at level 8 — the same question as mongodb-odm (vault `open-items`) will come up.

## 6. Shapes

**Decided 2026-11-05, amended 2026-11-15 (roadmap D29): a shape gets syntax only with a name.**
`array{id: int, name?: string}` is Phaenon syntax in the right-hand side of a `type` declaration
and nowhere else. Everywhere else a shape is docblock vocabulary, as the decision of 2026-11-05
(§6.7) had it for every shape. Built in the compiler on 2026-11-15; §6.9 holds what building it
settled, and wins where it differs from §6.1 to §6.6.

**A shape is a static type, erased at compile time.** It compiles to `array` in the signature and
to the shape in the docblock PHPStan reads, and to nothing that runs: no check, no descriptor, no
class. A value is a `Row` because the code that built it says so, as a value is a tuple (§2.4).

### 6.1 Spelling

```php
namespace App\Orders;

type Row = array{id: int, name?: string};
type Order = array{id: int, lines: list<array{sku: string, qty: int}>};
type Page<T> = array{items: list<T>, total: int, next?: string};

final class Orders
{
    find(int $id): ?Order { … }

    page(int $offset): Page<Row> { … }
}
```

- **Where.** A shape literal stands in the right-hand side of a `type` declaration
  (`generics.md` §2.1): as the whole type, or anywhere inside it, in a type argument, a union, a
  nullable type, a tuple, a closure type or another shape. Between `type Name =` and its `;` a
  brace after `array` can only be a shape.
- **Nowhere else.** In a parameter, return, property or constant type, in a type argument written
  outside a `type` declaration, and in the bound or the default of a type parameter, `array{` is
  a syntax error whose message says what to do: "a shape is written in a `type` declaration:
  name it". In a return type before a body the brace opens the body, as in PHP, and the parser
  meets the first entry as a statement. No Phaenon body begins as a shape's entry does (`id:` is
  a label, which Phaenon does not have: `P1001`), so the same message is given there.
- **Keys** are the ones the docblock grammar reads (`shapeKey` in the compiler's
  `internal/docblock`):
  - a name, PHP's reserved words included (`class`, `default`, `list`), as after `->`;
  - a quoted string, single or double and without interpolation, for a key that is not a name:
    `'$set'`, `'a-b'`. A docblock also reads `a-b` unquoted, as PHPStan does; the lexer has no
    such token, so the syntax quotes it;
  - an integer, `0` or `-1`. A quoted key that is a decimal integer is that integer's key, as in
    §2.
- **`?` after the key** marks the entry optional: the key may be absent. `name?: string` and
  `name: ?string` differ as in PHPStan, an absent key against a present `null`.
- **Every entry has a key**, and a key is written once. A positional shape is a tuple, `(A, B)`
  (§2.4), inside an alias as anywhere.
- **`...` as the last entry** leaves the shape unsealed: `array{id: int, ...}` admits keys it
  does not name. The lattice and the docblock parser already have it (`types.Shape.Unsealed`,
  set from `parseShape`, printed back by the lattice's printer): reading a key a sealed shape
  does not name is `P2313`, on an unsealed one it is `mixed`. Since it may hold any key it does
  not name, with any value, it is no shape that gives such a key a type (`checker.md` §3.3).
  PHPStan's typed form, `...<K, V>`, is not read by the docblock parser today (`P4001`, and the
  tag is dropped), so it is not syntax either (`checker.md` §10).
- **A trailing comma** is allowed, and a shape may span lines, an entry per line. `array{}` is
  the empty array.
- **Only `array{…}`.** `list{…}`, `non-empty-array{…}` and `object{…}` stay docblock vocabulary:
  the first is what a tuple spells, the second a refinement the checker infers (§2), the third
  documents a `stdClass`, which is rare.

The type is the lattice's shape, the one a docblock shape and an array literal already are
(§2.1, `checker.md` §3.1), with the rules a docblock shape has today (`checker.md` §9). It is
structural: an alias names a type and makes no new one, so `Row` and a docblock's
`array{id: int, name?: string}` are one type, and two aliases with the same right-hand side are
interchangeable.

### 6.2 Why with a name, and only with one

- **An inline shape cannot be read by the tools that read tokens.** The one ambiguous position is
  the one that matters most, a return type before a body: `): array {` is where PHP's own grammar
  has a block, and `id:` at its start is a label. Verified on the compiler of 2026-11-15:
  `find(int $id): array{id: int} {` parses as a body. An inline form would need a trial parse,
  reading to the matching brace to see whether a body follows. The compiler's parser could make
  one. The editors' token-level readers cannot: the PhpStorm plugin's lexer erases Phaenon's
  types before PhpStorm's parser sees them, and the VS Code grammar is TextMate rules. After
  `type Name =` there is no block to mistake the brace for, and both readers already take a
  `type` declaration whole, up to its `;`.
- **The reasoning of 2026-11-05 survives**, which is why this amends that decision and does not
  reverse it. A structure that deserves a type of its own is a class; a shape documents an ad-hoc
  array, and PHP's ease there, a shape as a comment and not a declaration, is kept for every
  array nobody names. A shape that gets syntax is one somebody found worth a name.
- **A name reads better than a repeated literal.** `find(int $id): Order` against the whole
  shape spelled out in every signature that passes one; and the alias is the one place to
  change. TypeScript's `type` and PHPStan's own `@phpstan-type` exist for this.
- **The cost of 2026-11-05 shrinks to the alias.** "The largest grammar addition for the fewest
  sites" was about a shape in every type position. Here the grammar grows in one production, the
  right-hand side of `type`, and the keys are the docblock parser's.
- **Still true:** a shape cannot be checked at runtime without runtime type descriptors, and
  Phaenon has none (roadmap D24, D26). The name changes nothing there.

### 6.3 An alias is its namespace's

A named shape that a public method returns is useless unless the caller's file can name it. Until
now a `type` alias was seen only in the file that declared it: verified on the compiler of
2026-11-15, another file of the same namespace gets `P2001 unknown class App\Pair`. So an alias
now belongs to its namespace, as a class does: any file of the project finds it by its qualified
name, and a file of another namespace imports it with `use`.

```php
namespace App\Http;

use App\Orders\Order;

final class OrderController
{
    show(int $id): Order { … }
}
```

`generics.md` §11.6 holds the rules: how an alias is found, a name declared twice, what a file of
aliases compiles to, and what crosses a package's boundary.

### 6.4 Compilation

What an alias does today (`generics.md` §11, "Type aliases"), unchanged: the `type` declaration
leaves no trace in the output, the signature gets the erased type, and the docblock the alias
expanded, with no `@phpstan-type`.

```php
// Orders.phn
find(int $id): Row => $this->rows[$id];

// .src/Orders/Orders.php
/**
 * @return array{id: int, name?: string}
 */
public function find(int $id): array
{
    return $this->rows[$id];
}
```

- A class a shape names is written by its fully-qualified name in every docblock, whatever the
  using file imports. The emitter does this for an expanded alias already.
- The output repeats the shape at every use. One place to change is the source's gain; the
  compiled PHP is what a PHPStan project writes when it has no `@phpstan-type`.
- Nothing is checked at run time, and `phn` adds no test of its own (roadmap D26). A value that
  comes from outside the checked code, a decoded payload or a row from a driver, is a `Row` once
  a test the programmer writes, a `@var` or a guard function with `@phpstan-assert` says so
  (§2), as for any array type.

### 6.5 Docblock shapes stay

`@param array{name: string, age?: int} $row` stays the way to write a shape nobody named, read in
`.phn` as in `.php`: PHP code read from `vendor/`, and the one-off array that a name would only
add ceremony to.

- `@phpstan-type` and `@phpstan-import-type` stay read, class-scoped as PHPStan has them. In a
  `.phn` the `type` declaration is the spelling; the tags are what PHP code has.
- **A docblock may name an alias** (2026-11-15). An inline shape that holds a named one, as in
  `@return array{total: int, lines: list<Line>}`, resolves `Line` as a signature does, and the
  emitter expands it for PHPStan with the rest of the vocabulary (§4). Today such a name is read
  as a class and reaches the output unexpanded.
- Inside a class, its own `@phpstan-type` wins over an alias of the namespace with the same
  name: the order in which the compiler resolves a name today, a type parameter, then the
  class's docblock aliases, then the rest.

### 6.6 Migration

`phn migrate` lifts the aliases PHP code already has:

- **`@phpstan-type Row array{…}`** (`@psalm-type` and the `=` form too) becomes
  `type Row = array{…};` before the class that carried the tag, under the usual rule
  (`generics.md` §11.1): the type has a Phaenon spelling, its members translated as in any lift
  (`class-string<T>` → `class<T>`, `array<string, V>` → `map<string, V>`). An alias that is not a
  shape (`@phpstan-type Json array<string, mixed>`) lifts the same way.
- **`@phpstan-import-type Row from Orders`** goes: when `Orders` is in another namespace it
  becomes `use Other\Namespace\Row;`, with `as Alias` kept, and in the same namespace nothing is
  needed. It is lifted only when the alias it imports was.
- **A name that is taken stays a tag.** PHPStan's aliases are scoped to a class, so two classes
  of one namespace may each declare a `Row`. At the namespace's level the two collide
  (`generics.md` §11.6), and so does an alias that bears the name of a class of its namespace.
  Such an alias is not lifted, and neither is one whose right-hand side names an alias that
  stayed: the tags stay, read as today, with a note naming the other declaration. Which of the
  two is renamed is a decision for a person.
- **An inline shape stays a docblock.** The tool cannot invent a name, and `@param array{…} $row`
  remains valid.

What this leaves, counted in the trees on 2026-11-15: core and mongodb-odm hold nine shapes under
a `@phpstan-type` name (five and four), and tq-cli's four docblock aliases name
`array<string, mixed>`. Against the 180 docblock shapes of §6.7, nearly all stay docblocks until
somebody names them by hand.

### 6.7 The decision of 2026-11-05

*Amended 2026-11-15 by §6.1–§6.6: "no native syntax" now holds for a shape without a name. The
text as decided, where "named shapes" meant shapes with named keys:*

**Decided 2026-11-05: shapes stay docblock vocabulary, with no native syntax.** A shape documents
the keys of an ad-hoc array: the configuration tree a bundle receives, a decoded payload, a data
provider's rows. A structure with fixed keys that deserves a type of its own is a class. The ease
of PHP's arrays, where a shape is a comment rather than a declaration, is a feature worth keeping.
So `@param array{name: string, age?: int} $row` stays the way to write one, read in `.phn` as in
`.php`.

- **Numbers.** After the migrations, the declarations of core, tq-cli and mongodb-odm carry 180
  named shapes in docblocks (81, 85 and 14), the largest kind of docblock type left in the
  dogfood. Before them, the M4 draft counted 35 in core and 27 in mongodb-odm.
- **The pair is the exception**, and it has a type: the tuple `(A, B)` of §2.4.
- **What follows:** the docblock is first-class. Phaenon types inside a shape are translated for
  PHPStan (§4), and `@phpstan-type` names a shape used in several places.
- **Still true from the M4 draft:** native syntax would be the largest grammar addition for the
  fewest sites, and a shape cannot be checked at runtime without runtime type descriptors.

### 6.8 Open

`checker.md` §10 for the shape itself (the typed rest of an unsealed shape, keys a sealed shape
does not name); `generics.md` §9, questions 9 to 13, for the alias (aliases of sources
outside the program, whether a type remembers its alias, `@phpstan-type` in the output, and
what a run over given paths does not see).

Found while building (2026-11-15), about the lattice's shape, which the syntax made easier to
write and did not change. *Closed the same day* (`checker.md` §9, *A shape is held as PHPStan
holds it*): an unsealed shape and `array<string, mixed>` accepted where a shape is expected, the
suite that holds what a shape accepts to PHPStan (`shapes-held.phn`, the docblock spelling of
`shapes.phn`'s cases), and the inference of a type argument through a shape (`generics.md`
§3.4). What is left:

- **A shape is still checked more leniently than PHPStan checks it**, in three places, each a
  parity gap of the kind `checker.md` §9 forbids, none of them new with the syntax:
  - a key is still read from a shape after `unset` removed it;
  - reading a key an unsealed shape does not name gives `mixed` without a word, where PHPStan
    reports an offset that might not exist;
  - destructuring an optional key (`['name' => $n] = $row`) is accepted as if the key were
    there.
- **What a test proves of an offset stays with the offset** (`checker.md` §10). After
  `isset($a['id']) && is_int($a['id'])`, PHPStan passes an `array<string, mixed>` for
  `array{id: int}`, and the checker, which narrows `$a['id']` and not `$a`, reports it: stricter
  than PHPStan, as the parity rule allows, and found nowhere in the dogfood trees.

### 6.9 What building it settled (2026-11-15)

The compiler has named shapes as §6.1 to §6.6 describe them. Building them settled what the text
left unsaid; where a sentence here differs from one above, this one is what the compiler does.
The rules of the alias itself are `generics.md` §11.7.

- **A quoted key is taken as written.** The docblock grammar reads the text between the quotes
  and nothing else, so the syntax does the same, and an escape sequence in a key (`\'`, `\\`,
  `\n` in double quotes) is a syntax error: with one, the two spellings of a shape would be two
  different keys, and `phn migrate` could not move a shape without rewriting it. A backslash that
  escapes nothing is itself, so `'App\Entity\User'` is a key.
- **An integer key is one PHP stores as an integer.** Unquoted, a key is a decimal integer
  without a leading zero (`0`, `-1`); `007`, `0x10` and `1_000` are syntax errors. Quoted, a key
  is that integer only where PHP itself would store it as one: `'7'` is the key 7, while
  `'07'`, `'-0'` and `'+1'` are strings, as PHP and PHPStan have them. The docblock reader
  follows the same rule; it had read all three as integers.
- **The brace after `array` in a return type.** §6.1 says the first entry is met as a
  statement, and a label gives the message. An entry with an optional key begins as a valid PHP
  statement does (`FOO ?: f();`), and an empty shape is an empty body. So the parser takes the
  brace for a shape only when what follows it can begin nothing else: `key:`, a quoted or
  integer key and its colon, `...`; `key?:` when no `;` stands directly inside the braces;
  `{}` when a body, an arrow or, on a method without a body, a `;` follows it. It then reads
  the shape, so that the body after it is read as a body, and reports the shape when the file
  is parsed, an earlier syntax error still being the first. Everywhere else a brace after
  `array` outside a `type` declaration is the shape's error at once.
- **Hyphenated names are not syntax, so two of §6.1's three exclusions are caught by name.**
  `list{…}` and `object{…}` in a `type` declaration are syntax errors that say they stay
  docblock vocabulary, and so are `non-empty-array{…}` and `non-empty-list{…}`, recognised as
  the sequence of tokens they are. A malformed entry is reported for what it is: a key a
  docblock writes bare and the syntax quotes (`a-b`), a number that is no integer key, an entry
  without a key, the end of the file.
- **A shape that names no key is emitted as `array<mixed>`.** `array{...}` is any array, and
  PHPStan at level max reports the shape for the value type it does not state. The emitted
  docblock holds the same type with it stated; diagnostics print the shape as written.
- **The expansion holds no word PHPStan would resolve in the using file.** A class is written
  by its qualified name, `Closure` among them (`\Closure(int): int`, and `\Closure` in the
  signature), and `bytes` is written `string`, as for a type spelled in a signature (§4).
- **The expansion is written for where the alias stood.** An alias stands for a type of any
  form, and `?` or a neighbour in a union would bind to a part of it:
  - `?Alias` is the nullable of what the alias stands for, as a type: over a union,
    `int|string|null`, in the signature and in the docblock; over a nullable type, `?int` in
    the signature and `int|null` in the docblock, not `??int`; over an intersection,
    `(\A&\B)|null` in both, PHP's DNF; over a closure signature, `?\Closure` and
    `(\Closure(int): int)|null`. `?` stays before a type that takes it, `?array{id: int}`;
  - among the members of a union or an intersection, what the alias stands for is bracketed
    where it would merge with its neighbours: an intersection in a union
    (`int|(\A&\B)`), a closure signature in either, a union in an intersection. A union
    among the members of a union stays bare;
  - a closure type's return is bracketed whenever its text is a union, written or expanded:
    `\Closure(U): (int|string)`. Without the brackets PHPStan reads the union as
    `(\Closure(U): int)|string`.

  The same holds in a docblock written by hand: `?Value` and `Both|null` are expanded with
  their place known.
- **A docblock's mention of an alias written over several lines is expanded on one.** The
  emitter rewrites a docblock in place and keeps its line count, and an expansion has no line
  break. So a generic alias written with its type arguments on lines of their own is expanded
  on a single line, and the lines the mention gave up stay in the docblock as bare ` *` lines
  before the tag:

  ```
   * @param Page<            *
   *     Customer            *
   * > $p                    * @param array{items: list<\App\Customer>, total: int} $p
  ```

  What follows the type on its last line, the parameter's name, stays beside the type, and a
  bare line above a tag changes nothing PHPStan reads. A docblock that opens on the tag's line
  opens on a line of its own instead. The alias never reaches the output by name.
- **`phn migrate`**, beyond §6.6:
  - an alias is lifted with the aliases it names, or not at all; one that names itself, or
    that mentions a type parameter of its class (`@phpstan-type Boxed array{value: T}`), stays;
  - a tag with a description after its type stays, since a `type` declaration has no place
    for it;
  - a name is taken, besides the cases of §6.6, when the file imports another type under it,
    and when the class declares it twice in different letter cases;
  - a signature that a lifted alias types is lifted with it: `@return Row` over `array`
    becomes `: Row`;
  - an import from the class's own namespace under another name becomes `use NS\Row as R;`;
  - an import is lifted when the alias it names is lifted by the same run, or already is a
    `type` of that namespace from an earlier one;
  - an alias that a class outside the run still imports stays a tag, with a note naming the
    importer, as an import stays whose alias did: neither side is left naming what the other
    no longer declares. To know, the run reads the project's other files that hold an import
    tag: with a project file, the files of its roots; without one, as in a PHP project
    migrated for the first time, the files of the directories the nearest `composer.json`
    autoloads, under `autoload` and `autoload-dev` (PSR-4 and PSR-0 directories, the
    classmap's entries, `files`). Where neither says where the project's code is, the files
    that may import an alias cannot be known, and no alias is lifted: each stays a tag with a
    note that says so;
  - without the program's other files nothing is lifted, since whether the name is free
    cannot be known; and a run over part of a namespace does not see the tags of the classes it
    never loads, so a collision it missed is `P2511` at the next check.

## 7. Decisions (reviewed 2026-10-30)

| Question | Decision | Why |
|---|---|---|
| Pairs and tuples | A tuple type `(A, B)` (§2.4), erasing to the positional shape `array{A, B}` | the spelling people want, at the cost of nothing: it *is* the shape PHPStan already writes, destructuring already reads it, no boxing |
| `$list[$i]` on an unknown index | `T`, PHPStan's default; `$list[$i] ?? $default` is the idiom for "maybe" | Tequila does not enable `reportPossiblyNonexistentGeneralArrayOffset`; a warning on every index would turn typing into a minefield |
| Names | **Singular** class names, as PHP idiom wants: `\Phaenon\Seq` (`List` is reserved), `\Phaenon\Map`, `\Phaenon\Str` (`String` is reserved), `\Phaenon\Bytes`; namespace `\Phaenon`. Amends roadmap D5's `Str` spelling (unchanged) and the plural first draft | `new Strings()` reads as nonsense; the class name appears in generated PHP and in static calls |
| `Map::merge` | last wins, like `array_replace` | what every other language's merge does |
| Method syntax on `array` (legacy) | none; narrow to `list` or `map` first | a legacy array is a value one narrows before using |
| `P2516` (missing declared type) | an error in `.phn` for parameters, properties and returns | `mixed` is one word; "never inferred silently" (language-overview.md §3) |
| Sorting | returns a copy; PHP's by-reference `sort()` stays callable | value semantics all the way through the stdlib |
| Generated code | imports and short names (`Seq::map`), roadmap D11 | reads as PHP a person wrote |
| Wrapper objects (`make()` over ext-ds, Symfony-String-like objects) | not adopted; method syntax on the native types gives the chain, a lazy pipeline object can come later as a library | `Ds\*` has reference semantics and is a PECL dependency; wrapping at every boundary would copy O(n) per call, invisibly |
| Editor support | the PhpStorm plugin must complete and resolve the methods on `list`, `map`, `string` and `bytes` receivers in the same milestone | a method syntax nobody's editor knows is not shipped |
| `array<V>` (2026-11-05) | the native spelling of "an array of `V`, list or keyed", the type `map<int\|string, V>` | `map` reads as a dictionary; PHPStan's own one-argument form keeps the docblock the same text (§2) |
| A bare `array` (2026-11-05) | names no element type: an override inherits its parent's, anywhere else in `.phn` it is `P2516`. Supersedes "deliberate and fine" | the output failed PHPStan max on sources `phn check` passed (§2) |
| Shapes (2026-11-05) | *(Amended 2026-11-15 by "Named shapes" below: a shape with a name has syntax; for every other shape this row stands.)* docblock vocabulary, no native syntax | a fixed-key structure that deserves a type is a class; a shape documents an ad-hoc array (§6) |
| Phaenon types in `.phn` docblocks (2026-11-05) | translated for PHPStan by the emitter; the docblock parser reads tuples. *2026-11-15: the name of a `type` alias too, expanded (§6.5)* | the docblock is where shapes live (§4) |
| Tuples at runtime (2026-11-09) | a type only, with no runtime test; the tuple test of `is` (`generics.md` §11.4) went with the operator (roadmap D26) | the shape of a value at runtime is the programmer's code, as in PHP; a tuple's members are fixed by the type, which is what the checker needs |
| Named shapes (2026-11-15, roadmap D29; amends "Shapes") | `array{id: int, name?: string}` is syntax in the right-hand side of a `type` declaration, as the whole type or nested in it, and nowhere else; in any other type position it is a syntax error whose message says to name it (§6.1) | an inline shape is ambiguous where it matters most, a return type before a body (`): array {` is PHP's block), and the editors' token-level readers cannot make a trial parse, while after `type Name =` a brace can only be a shape; the reasoning of 2026-11-05 survives, since a shape that gets syntax is one that got a name; a name reads better than a repeated literal and is one place to change (§6.2) |
| A shape's entries (2026-11-15) | keys as the docblock grammar reads them: a name, a quoted string, an integer; `?` for an optional key; every entry keyed, a key written once; `...` last for an unsealed shape; only `array{…}`, no `list{…}` or `object{…}` (§6.1) | one grammar for the docblock and the syntax, so `phn migrate` moves a shape without rewriting it; the lattice and the docblock parser already have the unsealed flag, and not PHPStan's typed rest; a positional shape is a tuple (§2.4) |
| Shapes at runtime (2026-11-15) | a static type, erased at compile time: `array` in the signature, the alias expanded in the docblock, no `@phpstan-type`, no check of any kind (§6.4) | what an alias does today (`generics.md` §11); Phaenon adds no runtime behaviour (roadmap D24, D26), and a shape cannot be tested without runtime type descriptors |
| An alias is its namespace's (2026-11-15) | a `type` alias is found from any file of the project by its qualified name and imported with `use`; the rules are `generics.md` §11.6 (§6.3) | a named shape a public method returns is useless unless the caller's file can name it; until now an alias was seen only in its own file |
| Docblock shapes (2026-11-15) | stay, in `.phn` as in `.php`, and may name an alias, which the emitter expands (§6.5) | PHP interop, and the one-off array a name would only add ceremony to |
| `phn migrate` and shapes (2026-11-15) | lifts `@phpstan-type` and `@psalm-type` into `type` declarations and `@phpstan-import-type` into `use`; a name another class of the namespace declares, or a class bears, stays a tag with a note; an inline shape stays a docblock (§6.6) | the tool cannot invent a name, nor choose which of two declarations is renamed. Of the 180 docblock shapes the dogfood holds, nine have a name today: nearly all stay docblocks until named by hand |

## 8. Checker and emitter work (once reviewed)

- Checker: `map` key bound (`P2518`), method resolution on collection types (`P2519`), `P2516`
  to error, mutation rules of §2.2 where they differ from today, destructuring on lists/maps,
  spreading.
- Emitter: method form → static call; docblock vocabulary translation (§4).
- Runtime package: `Seq`, `Map` with tests; the compiler's embedded stub generated from them.
- Goldens: checker `collections.phn`, emitter `collections.phn` → `.php` under PHPStan max.
- Dogfood: `tequila/core`.
- 2026-10-30, later: `?->` and short-circuited chains guarded (above), `\Stringable` receivers,
  `phn migrate` lifting `array{A, B}` to `(A, B)`; `phn build --no-types` removed (method syntax
  needs types; `--emit-on-error` is the way to build a file with findings). Found on the way:
  `phn build` reparsed the source instead of compiling the checked tree, so no method call had
  ever been lowered by the CLI; the goldens now compile through the CLI's path.
- 2026-10-30: `P2518` with the implied `K of int|string` for an unbounded key parameter
  (`program.ImplyMapKeyBounds`, functions, methods, classes, aliases and closures), `P2516` an
  error — done.
- 2026-11-06: **a tuple is exact.** The lattice had followed PHPStan, where `list{int, string,
  string}` is a `list{int, string}`; found by the `list-types` audit, which saw tq-cli pass a
  four-element record where `(string, string, string)` was declared. A tuple admits no longer
  value now (`types.IsTuple`), and a shorter one only for entries a docblock marks optional.
  Python's tuples, the ones most people know by that name, are exact; a tuple of unknown length
  is a `list<T>`. The checker is stricter than PHPStan here, which §9 of `checker.md` allows.
- 2026-11-15: named shapes and namespace-level aliases designed (§6, `generics.md` §11.6,
  roadmap D29) and built in the compiler (§6.9, `generics.md` §11.7), then in both editors
  (`tooling.md` §6.2, §6.6). The roadmap's *Named shapes* entry lists what is left: the
  documentation, the dogfood trees.
