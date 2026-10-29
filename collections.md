# Collections: `list<T>`, `map<K, V>` and the legacy `array` — M4 design (draft)

Status: **draft for review**, 2026-10-29. Nothing here is implemented beyond what M3 already
has (the keywords `list` and `map` in type positions, their erasure to `array`, the lattice's
array types). The decisions are proposals until they move into a *Decisions* section.

## 0. One paragraph

PHP has one collection type, `array`, that is at once an ordered sequence and a dictionary, and
every PHP type checker spends half its vocabulary (`list<T>`, `array<K, V>`, `non-empty-list`,
shapes) telling the two apart in docblocks. Phaenon makes the distinction the type system's:
`list<T>` is an ordered sequence, `map<K, V>` a dictionary, and both **are PHP arrays at
runtime** — nothing is boxed, copy-on-write value semantics and every `array_*` function stay
exactly as they are. `array` remains as the *legacy type*, the honest spelling for "a PHP array I
know nothing more about", the way `mixed` is for values. On top sit a stdlib with a predictable
argument order (`\Phaenon\Lists`, `\Phaenon\Maps`) and method-call syntax on collection values
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
| `array` | the legacy type: `map<int\|string, mixed>` — PHP's `array` as the checker knows it | `array` | `array` |
| `non-empty-list<T>` / `non-empty-map<K, V>` | refinements the checker *infers* (`count($xs) > 0`, a literal, `$xs[] = …`) and reads from docblocks; written natively as `list<T>` / `map<K, V>` | `array` | `non-empty-list<T>` / `non-empty-array<K, V>` |
| `array{a: int, b?: string}` (shape) | read from docblocks in `.php` and `.phn` alike, as today; **not native syntax** in M4 (§6) | `array` | the shape |
| `iterable<K, V>` | `map<K, V>\|Traversable<K, V>` (M3) | `iterable` | `iterable<K, V>` |

Decisions proposed with the table:

- **`map`'s key bound is `int|string`**, implicit: `map<K, V>` without a bound on `K` in a
  generic declaration means `K of int|string`; `map<bool, V>` is a type error (`P2518`). PHP
  would cast the key; Phaenon refuses the type. `array-key` stays docblock vocabulary and is
  written `int|string` when a key type is meant generically (generics.md §11.2).
- **Subtyping.** `list<T> <: map<int, T> <: array`; `list<T>` is covariant in `T`; `map<K, V>` is
  covariant in **both** `K` and `V`. Arrays are values (copy-on-write), there is no aliasing
  through which a covariant key could be abused, so `map<'a'|'b', Admin> <: map<string, User>`
  is sound. This answers generics.md §3.3's open question (PHPStan agrees in practice: it treats
  array key and value types covariantly).
- **`array` is allowed in declarations**, and it means what it says. "`mixed` must be written,
  never inferred silently" (language-overview.md §3) applies: a parameter with no type at all is
  `P2516`, which M4 turns from a warning into an **error** in Phaenon files (checker.md §9 said
  M4 decides); a parameter written `array` is deliberate and fine, like one written `mixed`.
  Passing an `array` where a `list<T>` or `map<string, V>` is expected is an error, as PHPStan
  reports it at level 7+; narrowing is `is_list()`-style guards, a shape docblock, or `as`
  (`$config as map<string, mixed>`, a runtime check through `\Phaenon\Type`).

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
tells the truth. `\Phaenon\Lists::values($xs)` (or `array_values`) is how a list is reasserted.
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

## 3. The stdlib: `\Phaenon\Lists` and `\Phaenon\Maps`

Written in PHP in the runtime package (`phaenon/runtime`), with full Phaenon signatures in their
docblocks so the checker types every call precisely; the stub embedded in the compiler is
generated from the package (closing [[gotcha-runtime-stub-drift]]). Names follow Go's `slices`
and `maps` packages: a plural noun per collection kind, functions named for the operation.

Argument order, everywhere: **the collection first, then the operands, callbacks last.**

```php
namespace Phaenon;

final class Lists
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
    public static function zip<A, B>(list<A> $as, list<B> $bs): list<array{A, B}>;   // or a Pair class — §7
    public static function keyBy<T, K of int|string>(list<T> $xs, Closure(T): K $key): map<K, T>;
    public static function groupBy<T, K of int|string>(list<T> $xs, Closure(T): K $key): map<K, list<T>>;
    public static function values<T>(map<int|string, T> $xs): list<T>;   // the way back from a map
    public static function isEmpty(list<mixed> $xs): bool;
    public static function count(list<mixed> $xs): int;   // alias of count(); exists for the method form
}

final class Maps
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
    public static function toList<K of int|string, V>(map<K, V> $m): list<array{K, V}>;
    public static function fromList<K of int|string, V>(list<array{K, V}> $pairs): map<K, V>;
    public static function isEmpty(map<int|string, mixed> $m): bool;
}
```

Every function is a thin wrapper over the `array_*` function or loop a PHP developer would have
written; semantics are PHP's (copy on write, `===` for `contains`, stable sort). The emitter may
later inline the trivial ones (`Lists::map` → `array_map` with the arguments swapped) — an
optimisation, not a semantic question, and not in M4.

### 3.1 Method-call syntax

A method call on a value whose static type is a `list`, a `map` (or a `string`, see
`strings.md`) resolves to the stdlib class of that type and compiles to the static call:

```php
$active = $users->filter(fn (User $u) => $u->active)->map(fn (User $u) => $u->email);
//  →   \Phaenon\Lists::map(\Phaenon\Lists::filter($users, fn (User $u) => $u->active), fn (User $u) => $u->email)
```

Rules:

- The receiver's static type decides, at compile time. `->` on an array or a string is a runtime
  error in PHP, so no PHP program changes meaning.
- A receiver typed as a union of a collection and an object (`list<int>|Countable`) is an error
  (`P2519`): narrow first. `array` (legacy) has no methods: say `list` or `map`.
- Only the stdlib's methods exist; there are no user-defined extension methods in M4.
- The result type is the function's return type, so chains type through.

The method form is sugar, nothing more: `Lists::map($xs, $f)` and `$xs->map($f)` are the same
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
- **Docblocks in `.phn`:** a `@var map<string, int>` written in Phaenon source reaches PHPStan
  in the output. M4 decides: the emitter **translates** Phaenon vocabulary in user docblocks
  (`map<K, V>` → `array<K, V>`, `class<T>` → `class-string<T>`), so a docblock is never wrong for
  PHPStan. (Open item since M3.)

## 5. Migration

`phn migrate` already lifts `@param list<T>` / `array<K, V>` / `T[]` into `list` / `map`
(generics.md §11.1). M4 adds nothing to the lift; it decides what the lift leaves behind:

- A bare `array` with no docblock stays `array` (legacy) and is not a finding.
- Shapes stay in docblocks (§6).
- The `array_*` → `Lists::`/`Maps::` codemod is M6's `phn migrate` v1 ("where provable"), not
  M4.

Tequila `core`, the M4 target, in numbers (315 files, 19,909 lines in `src/`): 141 docblock
`array<…>` tags against 66 `list<…>`, 35 shapes, `array_values` 29 calls, `array_map` 18,
`array_key_exists` 14, `array_filter` 9; `sprintf` 234 calls (strings.md). `core` runs PHPStan
at level 8 — the same question as mongodb-odm (vault `open-items`) will come up.

## 6. Shapes

Tequila writes 35 shapes in `core`, 27 in mongodb-odm — configuration arrays, `[$a, $b]` pairs,
data provider rows. Proposal for M4: **shapes stay docblock vocabulary**, read in `.phn` as in
`.php`, and are not given native syntax yet. Reasons: a shape is usually a class that has not
been written (checker.md §10, *shapes vs classes*); native syntax would be the single largest
grammar addition of M4 for the smallest number of sites; and a shape cannot be tested at runtime
without the runtime descriptors. The one shape idiom worth a type is the pair (`zip`,
`toList`); §7 asks whether `array{K, V}` or a `Pair<K, V>` class carries it. Revisit with the
`core` numbers after the migration.

## 7. Open questions (for review)

1. **Pairs and tuples.** `Lists::zip` and `Maps::toList` return `list<array{A, B}>`. A native
   tuple type (`(A, B)`) is grammar; a `\Phaenon\Pair<A, B>` class is boxing. Proposal: the
   shape, as PHPStan writes it, until shapes are decided.
2. **`$list[$i]` on an unknown index**: PHPStan's default (`T`) or `?T`? Proposal: PHPStan's.
3. **Names.** `Lists`/`Maps`/`Strings` after Go's `slices`/`maps`/`strings`, or `Lst`/`Map`/`Str`
   (roadmap D5 wrote `\Phaenon\Str::upper`)? Proposal: the plural nouns; D5 to be amended.
4. **`Maps::merge` semantics** — last wins (like `array_replace`) or first wins (like `+`)?
   Proposal: last wins, since that is what every other language's merge does.
5. **Method syntax on `array`** (legacy): none, or `Maps`' methods? Proposal: none; a legacy
   array is a value one narrows before using.
6. **`P2516` as an error**: for parameters only, or for properties and returns too? Proposal: all
   three in `.phn`; `mixed` is one word.
7. **Sorting**: in place (PHP's `sort()` by reference) or returning a copy? Proposal: a copy; the
   by-reference `sort()` stays callable.

## 8. Checker and emitter work (once reviewed)

- Checker: `map` key bound (`P2518`), method resolution on collection types (`P2519`), `P2516`
  to error, mutation rules of §2.2 where they differ from today, destructuring on lists/maps,
  spreading.
- Emitter: method form → static call; docblock vocabulary translation (§4).
- Runtime package: `Lists`, `Maps` with tests; the compiler's embedded stub generated from them.
- Goldens: checker `collections.phn`, emitter `collections.phn` → `.php` under PHPStan max.
- Dogfood: `tequila/core`.
