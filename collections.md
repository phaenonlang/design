# Collections: `list<T>`, `map<K, V>` and the legacy `array` — M4 design (draft)

Status: **reviewed 2026-10-30**; §9 holds the decisions, the body is the design they settle.
**Superseded in part by D20 (2026-11-03)**: the stdlib `\Phaenon\Seq` / `\Phaenon\Map` (§3) and
the method-call syntax (§3.1) are gone from the language. The types, literals, tuples, interop,
migration and shapes (§2, §4–§6) stand; lists and maps are transformed with PHP's functions and
chained with pipes and partial application (`roadmap.md` D20, `php-versions.md` §5.4).
**Amended 2026-11-05** (§7): `array<V>`, a bare `array`, shapes and the docblock vocabulary.
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
| `array{a: int, b?: string}` (shape) | read from docblocks in `.php` and `.phn` alike, as today; **not native syntax** in M4 (§6) | `array` | the shape |
| `iterable<K, V>` | `map<K, V>\|Traversable<K, V>` (M3) | `iterable` | `iterable<K, V>` |

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
    PHPStan reports it at level 7+. Narrowing is `is_list()`-style guards, a shape docblock, or
    `as` (`$config as map<string, mixed>`, a runtime check through `\Phaenon\Type`).
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
It is spelled with parentheses in any type position (`list<(string, int)>`, `(User, bool)`,
`?(int, int)`), built with an array literal (`[$name, $age]`), read by destructuring
(`[$name, $age] = $pair`) or by index (`$pair[0]`), and erases to the shape PHPStan writes:
`array{string, int}` in docblocks, `array` natively. In the lattice it *is* the sealed positional
shape `array{0: A, 1: B}`, which the checker already has, so `zip`, `toList` and every PHP API
that returns a pair-shaped array interoperate without conversion. A tuple is at least two
elements long; `(A)` is a parenthesised type.

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
  boundary check could go.
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
    namespace.

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
- Shapes stay in docblocks (§6).
- The `array_*` → `Seq::`/`Map::` codemod is M6's `phn migrate` v1 ("where provable"), not
  M4.

Tequila `core`, the M4 target, in numbers (315 files, 19,909 lines in `src/`): 141 docblock
`array<…>` tags against 66 `list<…>`, 35 shapes, `array_values` 29 calls, `array_map` 18,
`array_key_exists` 14, `array_filter` 9; `sprintf` 234 calls (strings.md). `core` runs PHPStan
at level 8 — the same question as mongodb-odm (vault `open-items`) will come up.

## 6. Shapes

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
| Shapes (2026-11-05) | docblock vocabulary, no native syntax | a fixed-key structure that deserves a type is a class; a shape documents an ad-hoc array (§6) |
| Phaenon types in `.phn` docblocks (2026-11-05) | translated for PHPStan by the emitter; the docblock parser reads tuples | the docblock is where shapes live (§4) |

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
