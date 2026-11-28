# Phaenon — Type checker (M2 design)

Status: **draft 0.1** · 2026-10-28 · companions: `generics.md` (M3 builds on this), `roadmap.md`

## 0. One paragraph

The checker is the product. M2 delivers a checker for the PHP subset of Phaenon — native types
plus the PHPStan docblock vocabulary — strong enough to replace PHPStan `level: max` on the first
Tequila libraries. It is local, bidirectional, flow-sensitive, and reads the rest of the program
(vendor `.php`, php-src stubs) through the same parser. Generics *syntax* is M3; the type
representation and the docblock `@template` inference are M2, because every vendor signature
already carries them.

## 1. Goals and non-goals

**Goals (M2)**

- `phn check` on `tequila/reflection` and `tequila/object-mapper` reports the same findings as
  PHPStan at level 10 — or better. Every disagreement becomes a test case.
- Type errors mean no output: `phn build` runs the checker first.
- Diagnostics are precise enough for an editor: file, line, column, span, code, fix-it where known.
- The vendor reader needs **no configuration**: Composer's autoload maps say where classes live;
  the compiler ships the php-src stubs.

**Non-goals (M2)**

- Generic syntax, variance, erasure, `reified`, `is`/`as` → M3.
- `list<T>` / `map<K, V>` as *first-class* types with literals → M4. M2 types them as PHPStan does
  (`list<T>` and `array<K, V>` are array types).
- Whole-program optimisation, dead-code elimination, incremental caching → M6.
- PHPStan's extension ecosystem (dynamic return type extensions). A handful of core functions get
  hand-written signatures instead (§5.4).

## 2. Architecture

```
internal/types      type representation, printing, subtyping, union/intersection normalisation
internal/docblock   PHPDoc parser: tags + the PHPStan type grammar → ast-free type syntax tree
internal/program    the program model: classes, functions, constants, their signatures;
                    loaders (project .phn, Composer vendor .php, embedded php-src stubs);
                    name resolution; hierarchy, trait flattening, method tables
internal/checker    binder + expression typing + flow analysis + diagnostics
stdlib/php-src/     copy of php-src *.stub.php (+ its LICENSE), embedded with go:embed
```

Pipeline for one `phn check`:

1. **Discover** project files: the paths given (`.phn`, walked) — the *checked set*. Since
   0.8.0 a plain `.php` file under a root is in it too, checked as PHP, for `phn check` and
   `phn build` alike (one collector feeds both, `phn watch` and the language server). Left out:
   the outputs, a `.php` that stands beside a `.phn` of the same name, and a file under an
   `exclude`d path, which the build copies without reading. An installed package's plain `.php`
   is read for what it declares and not reported on.
2. **Declare**: parse every checked file; collect declarations into `program.Program`. Nothing
   else is parsed yet.
3. **Resolve on demand**: a class, function or constant referenced from a checked file that is not
   in the checked set is looked up through the loader chain: project → Composer autoload maps →
   stubs. Each vendor file is parsed at most once; each declaration's signature is built lazily.
4. **Check** every checked file: bind names, type declarations, type bodies, emit diagnostics.
5. **Report**: sorted by file then position; exit code 1 on any error.

Vendor code is never *checked*, only *read*. The checked set is the unit of blame.

## 3. Types

### 3.1 The lattice

Set-theoretic, after PHPStan: a type is a set of runtime values; `mixed` is every value, `never`
is none; a union is a set union and an intersection a set intersection. Subtyping is set
inclusion. The representation (package `types`):

| Family | Types | Notes |
|---|---|---|
| top / bottom | `mixed`, `never` | `void` is `never`-like in value position and reported (P2) |
| null | `null` | `?T` is sugar for `T\|null` |
| bool | `bool`, `true`, `false` | `bool` = `true\|false` |
| int | `int`, literal `3`, range `int<0, max>` | ranges because `count()`, `strlen()`, `array-key` need them |
| float | `float`, literal `1.5` | `int` is a subtype of `float` in **parameter** position only (PHP widening) — §3.3 |
| string | `string`, literal `'a'`, `non-empty-string`, `numeric-string`, `class-string`, `class-string<T>`, `callable-string`, `literal-string` | |
| array | `array<K, V>`, `list<V>`, `non-empty-array`, `non-empty-list`, shapes `array{a: int, b?: string}`, `callable-array` | key type ⊆ `array-key` = `int\|string` |
| object | `object`, class `Foo`, `Foo<T, U>`, `static`, `self`, `$this`, enum case `Foo::Bar` | class types carry type arguments from M2 — vendor docblocks have them |
| callable | `callable`, `callable(A, B=): R`, `Closure(A): R`, `pure-callable` | |
| iterable | `iterable<K, V>` | = `array<K, V>\|Traversable<K, V>` |
| other | `resource`, `scalar` (= `int\|float\|string\|bool`), `array-key`, `numeric` | |
| generic | type parameter `T` (with bound), conditional types (`T is X ? A : B`) read but treated as their union | |

Representation rules: unions are flat, sorted, deduplicated and *simplified* (`int|3` → `int`;
`true|false` → `bool`; `A|B` where `A <: B` → `B` for object types). Intersections contain only
object types (`A&B`) or `callable&object`; an intersection of two unrelated final classes is
`never`. Literal types are kept until a union exceeds 8 members, then widened to the primitive
(PHPStan does the same).

### 3.2 Printing

Types print in PHPStan vocabulary (`array<string, Foo>`, `list<int>`, `'a'|'b'`, `Foo<T>`); a type
prints the same whether it came from native syntax or a docblock. This is also what M3's docblock
emitter prints. Native-printable types (`int|string`, `?Foo`, `Foo&Bar`) have a second printer
for the emitted PHP signature.

### 3.3 Subtyping

`IsSubtype(a, b)`, structural on the lattice, nominal on classes:

- class `A <: B` iff `B` is `A`, an ancestor of `A`, an interface `A` implements (transitively), or
  `object`. Enums and anonymous classes are classes. Every class `<: object`.
- `A<X> <: B<Y>` iff `A <: B` and the arguments compare according to the parameter's variance
  (`@template-covariant` / `out`, `@template-contravariant` / `in`, invariant by default). M2
  implements this because vendor docblocks declare variance; M3 adds the syntax.
- `callable(A): R <: callable(B): S` iff `B <: A` (contravariant parameters) and `R <: S`.
- shapes: `array{a: int} <: array{a: int|string}`; a shape is a subtype of `array<K, V>` when every
  key and value fits; optional keys are `?` and a missing key is not `null`. *2026-11-15 (§9, "A
  shape is held as PHPStan holds it"):* a key more than the expected shape names is fine, but for
  a tuple, which is exact (`collections.md` §2.4). An unsealed shape may hold any key it does not
  name, with any value: against a shape, an entry it does not name must be optional and take
  `mixed`; against an iterable it is iterated with what it does not name, so
  `array{a: int, ...}` is no `iterable<string, int>`; against `array<K, V>` only its named
  entries are compared, as PHPStan compares them. A general array is a shape only when the shape
  is unsealed and each entry is optional and takes the array's values where its keys reach it.
  `IsAcceptable` adds what PHPStan accepts of a general array that might be the shape.
- `int <: float` **only** in parameter and return positions under PHP's widening rule, exposed as
  `IsAcceptable(arg, param)` — never in `IsSubtype`. `strict_types=1` is the only mode.
- literals: `3 <: int`, `'a' <: string`, `'Foo' <: class-string` iff `Foo` is a known class.
- `static` and `$this` resolve against the current class (`$this <: static <: self`), and
  `static` in a return type of class `C` is `C` for the caller when the receiver is `C`.

### 3.4 Narrowing

A *predicate* is a boolean expression the checker understands; evaluating it yields two
environments (then/else). Predicates recognised in M2:

| Predicate | then | else |
|---|---|---|
| `$x instanceof Foo` / `$x instanceof $cls` | `$x & Foo` (or the class-string's class) | `$x \ Foo` |
| `is_int/is_float/is_string/is_bool/is_array/is_object/is_null/is_iterable/is_callable/is_scalar/is_numeric/is_countable/is_resource($x)` | intersect | subtract |
| `is_a($x, Foo::class[, true])`, `is_subclass_of` | `$x & Foo`, or `$x & (Foo\|class-string<Foo>)` when `$allow_string` (`is_subclass_of`'s default) | — |
| `$x === null`, `null === $x`, `$x !== null`, `isset($x)`, `empty($x)` | as expected | |
| `$x === 'lit'` / `$x === Foo::Bar` / `$x === true` | literal / enum case | subtract literal |
| `$x`, `!$x` (truthiness) | remove `null`, `false`, `0`, `''`, `'0'`, empty array | keep falsy |
| `$a && $b`, `$a \|\| $b`, `!$p`, `($p)` | combined | |
| `count($x) > 0` / `=== 0`, `$x !== []`, `array_key_exists('k', $x)`, `in_array($x, [...], true)` | non-empty / shape key / literal union | |
| `array_is_list($x)` | a list of what the array held | — |
| a call whose callee has `@phpstan-assert T $x`, `@phpstan-assert-if-true` or `@phpstan-assert-if-false` (`@psalm-` and `@assert` alike) | the argument as the tag says: after the call, or in the branch the tag names | the negation, where the tag says |
| `mb_check_encoding($b, 'UTF-8')` on a `bytes` operand (2026-11-09) | `string&bytes` (`strings.md` §6) | `bytes` |
| `assert($p)`, `if (!$p) throw/return/continue/break/exit` | narrowing continues after the statement | |
| `match (true) { $p => … }` arms, `switch` with `===` semantics on literal subjects | per arm | |

Narrowing applies to **paths**: `$x`, `$x->prop`, `$x['k']`, `$this->prop`, `Foo::$prop`. A
property path is invalidated by any call on the same object, any assignment to a prefix of the
path, and by `unset`. A variable path is invalidated by assignment and by reference capture
(`&$x`, `use (&$x)`, passing to a by-ref parameter, `global`).

*Since 2026-11-09 (roadmap D26, §9) these predicates are how a Phaenon program tests a type: `is`
and `as` are gone, and PHP's tests are the only ones.*

*2026-11-20 (§9, "A narrowed type parameter is the parameter still"): a value of a type parameter
is narrowed in its bound and stays the parameter, `T of int` or `T&Row`; the table's intersect and
subtract are then those of the bound, and the cases where the parameter is lost are listed there.*

### 3.5 Flow analysis

Abstract interpretation over the AST with an environment `var → type` plus "definitely assigned"
bits. Rules:

- Branches merge by union of each variable's type; a variable assigned on only some branches is
  *maybe undefined* after the merge, and reading it is P2 (PHPStan: "might not be defined").
- `return`, `throw`, `exit`, `continue`, `break`, calls to `never` functions end a path.
- Loops: iterate the body with the environment joined from the entry and the back edge until a
  fixed point; after 3 iterations the changing variables are widened to their declared type, or
  to the union's primitive. `foreach` keys/values come from the iterable's key/value types; a
  `foreach` over a possibly-empty iterable may not assign its variables, and one over an array
  that holds nothing never runs its body: the body is checked, the state after the loop is the
  one before it (PHPStan's, for an array not iterable at least once), and `P2409` says so.
- `try`: catch blocks see the union of the environments at every statement of the body (any
  statement may throw); `finally` sees the union of all of them.
- Closures and arrow functions start from a fresh environment with their parameters, plus `use`
  variables (by-value copies, typed as at the capture point) — arrow functions capture the
  enclosing environment read-only by value.
- `static` locals start as their initialiser's type unioned with their declared docblock type.
- Variables assigned in `isset`-guarded expressions, `??=`, `list()`/`[...]` destructuring,
  `foreach`, `catch`, `global` (rejected, §7) and by-reference parameters are all definitions.

### 3.6 Inference

Bidirectional: `typeOf(expr, expected)`. The expected type comes from the assignment target's
declared type, a parameter type, a return type, an array literal's expected element type, or a
property's type. It is used to:

- give closure and arrow-function parameters their types from an expected `callable(...)` /
  `Closure(...)` type (`array_map(fn ($u) => $u->name, $users)` — the closure is typed *after* the
  other arguments, so `@template` inference from `$users` runs first);
- pick literal vs widened types for array literals (`['a' => 1]` is a shape unless the expected
  type is `array<string, int>`);
- infer docblock `@template` arguments on calls (unify argument types with parameter types, then
  the expected type with the return type), per `generics.md` §3.4.

Declared types win over inferred ones at assignment: a property or parameter keeps its declared
type after assignment, narrowed by the assigned value's type only until the next invalidation.
Local variables with no declaration are typed by their assignments (union over the flow). An
assignment to a local with a `@var` docblock on the statement uses the docblock type.

## 4. The program model

### 4.1 Declarations

`program.Program` owns every declaration ever loaded, keyed by lower-cased fully-qualified name:

| Decl | Fields (summary) |
|---|---|
| `Class` | kind, flags (abstract/final/readonly), parent, interfaces, traits (+ adaptations), type params (from `@template` in M2, syntax in M3), constants, properties (incl. hooks, asymmetric visibility, promoted), methods, enum cases + backing type, attributes, file + span, doc |
| `Function` | params, return type, by-ref, type params, attributes, `never`-ness, file + span |
| `Constant` | type, value if literal |
| `Param` | name, type, default (type + literal), by-ref, variadic, promoted, attributes |
| `Method`/`Property` | as above + visibility, static, abstract, final, readonly, declaring class, `@phpstan-*` overrides |

A declaration's **effective type** is: the docblock type when it is a subtype of the native type
(or the native type is absent); otherwise the native type plus P4 "docblock disagrees with
signature". `@phpstan-param` / `@phpstan-return` / `@phpstan-var` win over `@param` etc.; `@psalm-*`
are accepted as aliases. `@template` without a bound is `mixed`.

*2026-11-26: the docblock type and the native type are combined as PHPStan combines them
(`TypehintHelper::decideType`), for a parameter, a return type, a property, a promoted
constructor parameter and a constant. The docblock type is the type where it refines the native
one, and each member of a native union that shares no value with it is added back:
`@param list<string>` on `?array $xs` is `list<string>|null`, and `@param positive-int` on
`int|string $n` is `int<1, max>|string`. Until then the docblock stood alone, so a `null` the
declaration admits was refused (`return null`, a property's `null` default) and a use that
forgets the `null` passed (`count($xs)`). "Shares no value" is PHPStan's answer, member by
member of the docblock's type: two classes neither of which extends the other, neither being
an interface, share none (a class has one parent), so `@param Foo` on `Foo|Bar` keeps `Bar`; an
interface, which a subclass may implement, is not added back. Where the native type admits no
`null`, the docblock's `null` is left out and reported (`P4002`); a parameter whose default is
`null` is nullable first, as PHP reads `array $xs = null`. PHP's own declarations keep their
docblocks as written: the stubs and the overlay stand for PHPStan's function map, whose types
are whole. One branch of PHPStan's rule is not followed: an array docblock without a key on a
native `iterable` stays an array, where PHPStan reads an iterable of its values. That is
stricter (a `Traversable` passed for it is refused), and waits for an `iterable` narrowed by
`instanceof \Traversable` to leave the array PHPStan's leaves.*

### 4.2 Loaders

A `Loader` maps a fully-qualified name to a parsed declaration:

1. **Project**: the checked set, parsed in Phaenon mode.
2. **Composer**: `vendor/composer/autoload_psr4.php`, `autoload_classmap.php`, `autoload_files.php`
   (function and constant files) and `autoload_namespaces.php` (PSR-0), evaluated by walking their
   AST (string literals, `$vendorDir . '/…'`, `$baseDir`). The project's own PSR-4 roots are
   there too, which is how a `.php` file beside a `.phn` one is found. Looked-up `.php` files are
   parsed in PHP mode, `.phn` sources in Phaenon mode; a file is parsed once however many classes
   it declares. Composer's maps are found by walking up from the first checked path to the nearest
   `vendor/composer/`. The entries of `autoload_files.php` are read before the checked set
   (2026-11-17): one under an output directory of the project is read from its source, the `.phn`
   or the copied `.php`, as a class's file is, and one the run checks itself is left to it, so
   that a function a `"files"` entry loads from `.src/functions.php` is declared once and not
   again from its own output (`P2511` until then, at every build after the first).
3. **Stubs**: php-src `*.stub.php`, embedded. Quirks handled: `#if … #endif` lines lex as comments,
   so duplicate declarations appear — the first wins; `UNKNOWN` defaults mean "optional, no
   literal"; `@alias` copies the target's signature; `@tentative-return-type` and the other
   generator tags are ignored; the few `@param`/`@return` docblocks are honoured.

The stub set is overridden by a **signature overlay** (`stdlib/overlay/*.php`, Phaenon-maintained)
that adds the types PHP cannot express: `array_map` returning `list<R>` for a list, `array_filter`
keeping keys, `count` returning `int<0, max>`, `array_keys` → `list<K>`, `in_array`/`array_key_exists`
as predicates, `sprintf` → `string`, `preg_match` by-ref `$matches` shapes… The overlay is a plain
PHP file with docblocks; it is the M4 stdlib's ancestor.

### 4.3 Name resolution

PHP's rules, exactly: `namespace` scope, `use` / `use function` / `use const` with aliases and
group use, fully-qualified `\Foo`, relative `namespace\Foo`; unqualified **function and constant**
names fall back to the global namespace at runtime, so the checker resolves them to whichever
exists (namespaced first) and reports P2 when both exist with different signatures. Reserved
type names (`int`, `string`, … `mixed`, `never`, `iterable`, `self`, `static`, `parent`, `null`,
`false`, `true`) never resolve to classes. Case-insensitive for classes, functions and keywords;
case-sensitive for constants, properties and variables.

*2026-11-15 (roadmap D30):* the keywords meant here are PHP's. Phaenon's own words are written
in lower case: in a type position `map` and `bytes` are Phaenon's types, and `Map` or `Bytes` a
class name resolved as any other; a word that only Phaenon has in its place (`throws`, `type`,
`of`, `class<T>`, …) is read in any case there, and in another than lower case it is `P1006`.

### 4.4 Hierarchy

Built lazily per class, with cycle detection (P2 "circular inheritance"): the method table is the
class's own methods, then trait methods after adaptations (`insteadof`, `as`), then the parent's
table, then interface methods (abstract). Property and constant tables likewise. Signature
compatibility with the overridden member (parameter contravariance, return covariance, visibility,
static-ness, readonly) is checked for **checked** classes only — PHP fatals at runtime on the
same conditions, so the checker reports them as errors.

*2026-11-13: which declarations a method is compared with, and how Phaenon's types are held where
they say more than PHP's, are in §9 ("An override is held to every declaration it overrides").*

## 5. Checks

Every check has a stable code. Families: `P1` syntax and removed constructs, `P2` types and flow,
`P3` erasure/generics (M3), `P4` interop (vendor, docblocks). Numbers are allocated in the
implementation and listed in `checker-codes.md` (generated from the source, one line per code
with its message template). The M2 set, by area:

**Names** — undefined class / interface / trait / enum, function, constant, class constant,
property, static property, method, static method; `parent` without a parent; `self`/`static`
outside a class; unknown named argument; duplicate `use`; unused `use` (warning).

**Calls** — too few / too many arguments; argument type not acceptable; named argument after
unpacking; a named argument or placeholder for a parameter an argument or placeholder before it
fills (PHP's "Named parameter $x overwrites previous argument", `P2010` as for an unknown name,
`php-versions.md` §9); variadic/by-ref mismatches; call on `mixed`/`null`/non-object; calling a non-static
method statically and vice versa; instantiating an abstract class, interface, or enum; calling a
private/protected member from outside; `new` with wrong constructor arguments; invoking a
non-callable; first-class callable on an unknown function.

**Assignments and types** — assigned value not acceptable by the property/parameter/return
type; `void` function result used; missing `return` in a non-void function (flow-checked);
`return` with value in `void`; `never` function that returns; readonly property written outside
its declaring scope or twice; write to a `private(set)`/`protected(set)` property from outside;
property type and `@var` disagree; promoted property redeclared; uninitialised typed property
read before assignment in a constructor; `static` property access through an instance.

**Expressions** — binary operator on incompatible operands (`array + int`, `string * string`),
`.` on arrays/objects without `__toString`, offset access on non-arrays, `foreach` over a
non-iterable, `match` with no matching arm when the subject type is exhausted (error) or
non-exhaustive on an enum (warning, as PHPStan), comparison `===` between types with no overlap
(always false), `instanceof` always true/false, dead `?->` (left never null, or a property fetch
that `??`, `isset()` or `empty()` reads, which forgive the null themselves: `P2307` for both,
PHPStan's `nullsafe.neverNull`), `??` with a
left side that can't be null, `isset` on a never-null expression, casts that PHP rejects
(`(int) object`), `clone` of a non-object, string offset on non-string, `yield` outside a
generator-typed function, `$this` in a static context.

**Flow** — variable possibly undefined; variable certainly undefined; unreachable statement after
`return`/`throw`/`never` call (warning); always-true / always-false condition (warning);
catch block ordering (a wider exception first makes a later catch dead, warning); `foreach`
over an array that holds nothing (warning).

**Classes** — missing abstract method implementation in a concrete class; interface method
signature mismatch; override incompatible (LSP); a property redeclared against its parent's
static-ness, readonly-ness, visibility or native type (PHP's class-link rules, the first broken
reported in PHP's order); `#[\Override]` on a method, a property or a
constant that overrides nothing; missing `#[\Override]` is **not** reported (Tequila's cs-fixer
owns that); abstract method in a non-abstract class; final method overridden; trait conflict
without `insteadof`; enum with properties; enum case value type mismatch; interface
constants/properties rules; constructor promotion in an abstract constructor; readonly class with
static/untyped properties; property hooks: `get` returning the wrong type, `set` parameter
type, virtual property without backing used by reference.

**Removed from PHP (P1, §7)** — reported for checked files in Phaenon mode only.

**Interop (P4)** — docblock type unparsable (the native type is used); docblock type not a subtype
of the native type; `@template` referenced but undeclared; unknown class in a vendor docblock
(the type becomes `mixed` with a warning once per class); Composer autoload maps missing (then
only stubs are known — warning).

Severity: errors stop `phn build`; warnings do not. `phn check --strict` promotes warnings.

## 6. Diagnostics format

```
src/ObjectMapper.php:41:24: error P2104: argument 1 of Tequila\Reflection\ReflectorInterface::reflect() expects class-string<ClassType>, string given
    40 |         if (is_string($target)) {
    41 |             $target = $this->reflector->reflect($targetClassName)->newInstanceWithoutConstructor();
       |                                                ^^^^^^^^^^^^^^^^
```

Positions are 1-based line and column (bytes; the LSP layer converts to UTF-16 later). `phn check
--json` emits one object per diagnostic (`file`, `line`, `col`, `endLine`, `endCol`, `severity`,
`code`, `message`, and last `kind`, why the finding is reported: `"language"`, `"parity"` or
`"extra"`, §9). Messages name types in PHPStan vocabulary and members as `Class::member()`; they
never mention internal node names.

## 7. Removed constructs (P1)

Per `language-overview.md` §3, in Phaenon mode the checker rejects: `global`, `goto`/labels,
`eval`, `extract`/`compact`, `$$var`, `${expr}` in strings, alternative control syntax, `var`,
`(integer)`/`(boolean)`/`(double)`/`(real)`/`(binary)` casts and `(unset)`, `list()`,
`<?php`/`?>`/inline HTML (the parser already refuses these). Each P1 carries a fix-it.
*`include`/`require` inside functions stood in this list until 2026-11-17 and was never refused;
they stay (roadmap D32).* The **parser** keeps accepting them so the php-src
conformance corpus still parses; the checker is where "Phaenon mode" bites.

## 8. Testing

1. **Golden corpus** `internal/checker/testdata/*.phn` — one file per feature; expected
   diagnostics as trailing comments on the offending line: `// want P2104` (optionally with a
   message substring). A test runs every file and demands an exact match of the (line, code,
   kind) sets, the kind being whether the finding is an extra check, which its line marks after
   the code (`// want P2301 extra`, `// want P2103 extra "substring"`). It runs each file a
   second time with extra checks off, where the marked findings must be gone and every other
   one must stay, word for word and in its place.
   Vendor fixtures live under `testdata/vendor/` with a hand-written `autoload_psr4.php`.
2. **Type printing / subtyping tables** in `internal/types` — unit tests over pairs.
3. **Docblock grammar** — a corpus of PHPStan type strings → canonical printing, plus error cases.
4. **Stub loading** — every embedded stub parses; a few hundred well-known signatures are asserted
   (`strlen`, `array_map`, `ReflectionClass::newInstance`).
5. **PHPStan parity** (opt-in, `PHAENON_PHPSTAN=/path/to/phpstan`): for each golden file,
   compile it, run `phpstan analyse --level max --error-format json` on the output, its lines
   read through the line map, and assert that PHPStan's (line) set is covered by ours. *Until
   2026-11-19 a file of plain PHP was written as `.php` and read as it stood; a parity finding
   is one the output draws (roadmap D33), and the output is not the source read as PHP where the
   emitter rewrites plain PHP, as it does an overriding method's native types (generics.md
   §4.3).* Not the other way round: the checker may be stricter. Where PHPStan finds something
   we do not, the test fails and the case is either fixed or allowlisted with a reason. Since
   roadmap D33 the other direction is held too: a finding the corpus expects on a line PHPStan
   does not flag is a language rule (its code says so), or an extra check, which the corpus
   marks on its line (`// want P2301 extra`), or one of the findings the run's own table names,
   by file, code and line, with the reason it stands unmarked: an extra check by what it says,
   which the place that reports it cannot prove to be one (§9, §10), what PHP itself refuses,
   something of the source that the emitter rewrites, an artifact of PHPStan's line or version,
   a false finding waiting for its fix. Any other fails the run, and so do an entry of the table
   whose line has no such finding, and a line whose findings are all marked extra that PHPStan
   reports as well. Where PHPStan reports such a line only under an identifier the allow-list
   has, the line is named in a table of its own with that identifier, and one that is not named
   fails the run. That PHP refuses a line is not taken on the entry's word: the run loads the
   compiled file under php, again without each declaration php stopped at, and php has to refuse
   that file for that line, or throw there when code the table names is run. A line PHPStan
   reports only under an identifier the allow-list names for the file, or for that line of it,
   is PHPStan's in this direction. The files with syntax only Phaenon has, which the run
   otherwise leaves to the emitter's goldens, are compiled and held to the last rule where they
   carry marks. The second check of the corpus, with extra checks off, is item 1's.
6. **Dogfood**: `phn check` on `tequila/reflection` (`.phn`) and `tequila/object-mapper`
   (migrated on a branch) must report nothing; then a set of planted bugs must be caught.

## 9. Decisions made while building M2

Each of these came out of a disagreement with PHPStan on real code; the parity harness (§8.5)
turned it into a test.

- **Templates are per call.** A callee's type parameters get fresh identities at every call
  site, so the `T` of `first<T>()` is never confused with the `T` of the method calling it.
- **Foreign templates are inferred.** A type parameter that belongs to neither the callee nor
  the current scope — a constructor's `@template` kept on a promoted property, an interface's
  parameter on an unparameterised receiver — is inferred at the call, as PHPStan does. The
  promoted property keeps the constructor's templates rather than their bounds for that reason.
- **Generic first-class callables stay generic.** `array_map(self::make(...), $items)` resolves
  `make`'s templates from the array's element type, then the map's result from `make`'s return.
- **Conditional return types** (`$x is class-string<C> ? C : object`) are evaluated at the call
  site in M2 already; the design deferred them to M3, Symfony's container made them necessary.
- **Docblocks are inherited** by a method without its own from the method it overrides.
- **Property narrowings survive calls.** `$this->x` stays non-null across `$this->log()`; only
  an assignment to the path or a prefix of it invalidates. Sound it is not; PHPStan does the
  same and every codebase relies on it.
- **`switch` does not narrow scalars** (it compares loosely); enum cases and `null` do.
- **A bare `Closure` or `callable` is accepted for any signature.** The mismatch, if any, is
  reported at the call.
- **`class-string` is `class-string<object>`.**
- **One mistake, one diagnostic.** An expression already reported (undefined variable, unknown
  function) has the error type, which is accepted everywhere.
- **`@phpstan-ignore` is honoured** (next line, or the same line when trailing), so a migrated
  project keeps its existing suppressions: it silences every finding of its line whatever
  identifiers it names, and is never reported, since no table maps PHPStan's identifiers to
  Phaenon's codes. `@phaenon-ignore` is the native spelling, and it **names the codes it
  silences** (since 0.8.0): `@phaenon-ignore P2103, P2202 (reason)` silences those codes on
  its line, or on the next when it stands alone, and no other; the codes run to the first
  word that is none. `@phaenon-ignore-line` and `@phaenon-ignore-next-line` silence every
  finding of the line. A native comment that silences nothing is the warning `P4015`, as
  PHPStan reports its own (`ignore.unmatchedIdentifier`, `ignore.unmatchedLine`): once for
  each named code that is not reported on the line, once for a whole-line form on a line
  without findings, once for a comment that names no code. A comment is held only to the
  codes the run reports: `phn build`, which reports no deprecations, does not judge a comment
  for one, nor the whole-line forms; `phn check` judges every comment of a checked file.
  The extra checks a run leaves out are the exception: they are found in both modes and left
  out when the check ends, so a comment is held to them with the switch off too (below,
  *What off promises*).
  Nothing silences `P4015` itself, or a stale suppression could hide behind another. Nor does
  any comment silence Phaenon's syntax, `P1000` to `P1007` (since 0.8.0): a comment accepts a
  finding about what code does, and does not make other text Phaenon. Silenced, such a finding
  let the build fail on the file all the same with a less clear message, or write the text as
  it stood, `(integer)` or a construct the target cannot run. A native comment that names one
  of these codes is `P4015`; PHP's deprecations (`P1101`) stay silenceable, since the code
  runs. A tag counts where it starts the comment's text or follows a space or the comment's
  marks. Until 0.8.0 either spelling silenced its whole line and nothing reported a stale one.
  **Nor does any comment silence a finding about code PHP refuses as compiled** (2026-11-26):
  a compile error, or a fatal error when the file is loaded and its classes are declared or
  linked. Silenced, a method narrower than the one it overrides (`P2502`) was written into a
  class php refuses to link, and the compiled file did not run at all. The mark is made
  finding by finding, and only where the checker knows for certain that PHP refuses the code:
  a silenced finding about code php runs costs nothing, while a mark on one fails a build
  that loads. So a finding is compared as the compiled file writes both sides, a type
  parameter erased to its bound: `P2502` of a native signature is marked and that of a
  docblock alone is not, `P2204` under a native `void` and not under `@return void`. A rule
  PHP checks when it links a class is marked only for a class php links whenever it loads the
  file: one declared at the top of its file with nothing but declarations before it, whose
  parents, interfaces and traits are all known as the target's PHP declares them. A class
  after another top-level statement, one declared under a condition, an anonymous class and
  one with an ancestor the checker does not know are left to php, and their findings stay
  silenceable. An erased type parameter used as a value (`P3003`) is silenced by no comment
  either: `new T()`, `T::class` and `instanceof T` compile to PHP that never does what the
  source says. A native comment that names the code of such a finding on its line is `P4015`
  and says which of the two it is; `@phpstan-ignore` and the whole-line forms go on silencing
  the line's other findings. Every mark of the checker corpus is held to php, which must
  refuse the compiled code for the reason the code names (`TestRefusedByPHP`).
  **Nor a finding about an override the compiled file writes with another type than the
  source** (2026-11-26). Where the native type an override writes does not fit the method it
  overrides, the override-erasure rule (`generics.md` §4.3) compiles the type that fits, and
  `P2502` is all that says so: the body is checked against the type the source writes, which
  the compiled method does not have. Silenced, `(): iterable` over `(): array` compiled to
  `(): array`, a compile error in PHP under a body that yields, from a build that reported
  nothing; `(string $x)` over `(int $x)` compiled to `(int|string $x)`, whose body is handed
  an int; `(): float` under `Bounded<int>` compiled to `(): int|string`, which no float
  leaves. This is the second way code compiles to PHP doing something else than it reads,
  and its `P4015` is worded as `P3003`'s. The mark is on the slot the rule rewrote, in a
  static method and in a class declared when its code runs as anywhere else, since the
  emitter rewrites those too. What compiles as written stays a comment's to accept: a type
  that a docblock or Phaenon's syntax alone narrows, and a native type the erased
  declaration admits though its type argument does not (`get(): string` under `Box<int>`,
  whose compiled `get(): mixed` it fits).
  **Nor a finding about a line that throws whenever it runs** (2026-11-27; decided, and not
  built: §10). PHP compiles and loads such a file, so neither reason above holds. The reason
  is the checker's own: Phaenon is stricter than PHP by design, and a comment accepts what
  code does, which here is nothing but the throw, and a `throw` says that without a
  finding. That PHP runs the file up to the line is no reason to write it. The mark is made
  case by case, where all three hold:
  - *The line throws on every run that reaches it*, an `Error` of PHP's. A notice or a
    warning after which the code goes on is no throw.
  - *That is certain from the syntax and the declarations alone*, never from a type the
    checker inferred or a docblock states, which may be wrong where the code is right.
  - *The declaration is PHP's or the project's own.* A vendor package's is that of the
    version installed, and a library that supports two versions of a dependency rightly
    writes a call that only the other one runs.

  The first members, each as php answers it:
  - a constant that refers to itself, directly or through others (`P2016`);
  - a named argument that overwrites the argument before it, and a name no parameter has
    where the callee takes no variadic arguments (`P2010`);
  - a by-reference argument PHP refuses: a literal, a constant, an operator's result, an
    assignment, a cast, a `match`, a `clone` (`P2112`);
  - a piped value passed by reference (`P2114`).

  Not marked, and a comment's to accept as before: a call's result or a `new` passed by
  reference (`P2112` too), which PHP passes with a notice, as in `end(explode('.', $name))`;
  and each of the above where the callee is declared in a vendor package.

  Most findings about types say that a line throws when it runs, a `TypeError` under
  `strict_types`, and they stay silenceable: they rest on types, and a test that hands a
  wrong value to see it refused is valid code. So this is a list kept by judgment, and a
  member is added with the reason why no valid program writes it. Too few arguments to a
  function the checker knows is such a candidate, not taken yet.

  **The mark is an extra check** (below, *Extra checks are a kind of finding*). The finding
  is a parity finding and is reported in both modes. With `"extra-checks": false` a comment
  silences it as it silences PHPStan's finding of the same line. That is also the way out
  should the checker be wrong about a line: a false finding that nothing silences would
  stop a build with nothing to do about it. For the same reason a mark waits for the
  finding under it to be right: `$xs |> array_multisort(...)` was reported as `P2114` when
  this was decided, though PHP runs it.

  Every mark of the checker corpus is held to php by running the line, which must throw the
  `Error` the message names, as `TestRefusedByPHP` holds the others to php by loading the
  file. A native comment that names the code of such a finding on its line is `P4015`, in
  a third wording.
- **The overlay patches, it does not redeclare.** An overlay class merges into the stub or
  vendor class of the same name: its docblocks and the methods it lists win, the rest stays.
  That is how `ReflectionClass<T>` and PHPUnit's narrowing assertions are described.
- **Missing declared types are warnings** (`P2516`) until M4's collections make the
  replacement for untyped arrays available.
- **The rule set is Phaenon's own list, not a PHPStan level** (2026-10-29). The checks of §5
  are the specification; PHPStan is the oracle for what each one means on PHP code, not for
  which ones exist. The parity harness (§8) keeps running PHPStan at level max on the emitted
  goldens as a quality bar for generated code — cheap, because level 10 adds "no implicit
  `mixed`", which Phaenon forbids by language rule anyway. The reference profile for what
  Phaenon deliberately lacks is Tequila's: PHPStan 2.x, level 8 in most repositories, level 10 in
  the rest, `treatPhpDocTypesAsCertain: false` everywhere, no `phpstan-strict-rules`. Refinement types
  (`non-empty-string`, `positive-int`, `int<0, max>`, …) are inferred and narrowed —
  `$s !== ''` gives `non-empty-string` — but never required in source, since they cannot be
  written as native types (`generics.md` §11.2): where a declared `string` flows into a vendor
  `non-empty-string` parameter the finding stays, as in PHPStan, and the fix is a guard, not an
  annotation. Development must stay plausible for a person without an assistant; a rule that
  turns typing into a minefield is not adopted because PHPStan has it.
- **Docblock types are not certain** (2026-10-29, `tequila/mongodb-odm`). A docblock, an
  inherited generic or a `@var` tag refines what PHP declares; the refined type drives inference
  and narrowing, the PHP type is what the runtime guarantees. A diagnostic that calls a check
  impossible — an `instanceof` always true, a `=== null` always false, an `is` that could never
  hold (until D26) — requires both to agree. `normalize(object $value)` in a subclass of a
  generic normaliser is typed `UnitEnum`, and its `if (!$value instanceof UnitEnum) throw` is
  not dead code, since erasure lets any object in. This is PHPStan's
  `treatPhpDocTypesAsCertain: false`, Tequila's setting everywhere; the parity harness runs
  PHPStan with it. The type flow still uses the refined type: a ternary whose condition the
  refined types decide takes one branch, as in PHPStan.
- **`@var` narrows, never widens** (2026-10-29). Above an assignment, the tag's type must be a
  subtype of what PHP declares for the value (PHPStan's `varTag.nativeType`); a literal value must
  fit the tag instead. Above any other statement, `@var T $x` retypes `$x` for what follows; above
  a `return`, an unnamed `@var T` types the returned value. A `@var` on a promoted constructor
  parameter types the parameter and the property (`phn migrate` lifts it into the declaration).
- **Extensions come from `php`** (2026-10-29). The embedded stubs are php-src's; the classes,
  functions and constants of every other extension the local `php` has loaded — ext-mongodb,
  redis, amqp, apcu — are generated from its reflection, once per extension version, PHP version
  and generator, and cached under the user's cache directory (`program.LoadExtensions`; `PHN_PHP`
  names another binary). *Since 2026-11-15* a cached file's name also carries a hash of the
  script that writes it, so that a fix to the generator reaches every machine with nothing to
  delete by hand: a `phn` whose script differs reads none of the files an older one wrote and
  writes its own. Those older files are left in place, since a machine may run two versions of
  `phn`, which would otherwise delete each other's. `php` is the oracle for what exists.
  Reflection carries no docblocks, so a generated stub has no generics; the overlay refines what
  matters. Without a `php` on `PATH` the checker warns once (`P4007`) and knows php-src only.
- **A trait sees its users.** A method the trait body calls on `$this`, `self` or `static` and
  the trait does not declare is looked up in the classes using the trait, as PHPStan does by
  analysing a trait in each user's context. *Properties too* (2026-11-06): a property the body
  uses on `$this` and does not declare is every user's declaration, not the first one's, since
  the body runs in each. A read has the union of the users' types and narrows as in a class's
  own body; a write must suit each user's type (`P2201` names the user it does not suit), and the
  readonly and set-visibility rules apply in the user's scope, so a trait's constructor
  initialises its user's readonly property. A user without the property is `P2005` naming that
  user; a property no user declares stays `P2005` on the trait. One reading of PHPStan's is not
  taken: a comparison that one user's type alone decides (`=== null` on a property only another
  user declares nullable) is not reported as impossible, because the trait's code serves both;
  PHPStan, analysing per user, reports it in that user's context. Static properties through
  `self::` and `static::` are not looked through yet (vault `open-items`).
- **A readonly property is initialised by its constructor** (2026-11-06, `P2515`, PHPStan's
  `property.uninitializedReadonly`, `MissingReadOnlyPropertyAssignRule`). A readonly property of
  a class — every property of a `readonly class`, and those a trait brings — that has no default
  and is not promoted must be assigned on every way out of the constructor; a way out by `throw`
  or a `never` call needs nothing, a constructor that always throws initialises nothing, and so
  does a class without a constructor of its own (none, or a parent's, which runs in the parent's
  scope). A method of the class the constructor calls on `$this`, `self` or `static`, and those
  it calls in turn, count, conditional or not, as PHPStan counts them: the property is
  initialised at run time, and the one mistake, the write outside the constructor, is `P2206`
  already. A read in the constructor before every way has assigned it is `P2515` as well (PHP
  throws), `isset()` and `??` included, as PHPStan has it. The finding sits on the declaration,
  in the trait for a trait's property, where PHPStan files it and where its ignore comments are.
  On PHP 8.6 a default is the initialisation; on 8.5 the default is `P1002` and not also
  `P2515`. Not matched yet: PHPStan's "already assigned" (`assign.readOnlyProperty`), a second
  assignment in the constructor or one in a loop.
- **A declared type names classes that exist** (2026-11-06, `P2001`, PHPStan's `class.notFound`).
  A class in a parameter, return, property or class constant type, of a function, a method, a
  closure or an arrow function, a member of a union, intersection or nullable type, and in the
  positions Phaenon's syntax adds and the emitter writes into docblocks (type arguments,
  `class<T>`, closure signatures, tuples), must exist; PHP resolves it when it checks a value,
  so every call fails otherwise. Names a class docblock declares or imports
  (`@phpstan-type`, `@phpstan-import-type`) are aliases, not classes. A bare `Closure` is
  Phaenon's name for `\Closure` where the type is erased (a type argument, a signature: the
  emitter writes `\Closure`), but where the type reaches PHP as written — a parameter, return,
  property or constant type — PHP resolves it in the namespace, so without `use Closure;` it is
  `P2001` naming the class PHP would look for (`App\Closure`). This narrows `generics.md` §11's
  "the checker accepts a bare `Closure` everywhere": the emitter passes plain PHP through, so
  the checker reads it as PHP does. A class newer than the target is `P4011`, not `P2001`.
- **A general array is accepted for a shape it may well be.** *(Superseded 2026-11-15 by "A shape
  is held as PHPStan holds it", below: PHPStan 2.2 and 2.3 report the call.)*
  `array<string, string>` passes for `array{spec: string, declared: string}` when its keys and
  values can be the shape's, as PHPStan accepts it; `array<string, mixed>` still fails against
  `array<string, int>`.
- **Static methods may use the class's type parameters** (`P3008` stays for static properties
  only): late static binding resolves them, which is how the `aggregateClass(): class<T>` idiom
  works (`generics.md` §11.2).
- Smaller parity points settled the same day: `$r = []; $r[$k] = $v` types `$r` from its first
  entry, and an empty literal is not widened in a loop; a `const array M = […]` is typed by its
  value; `Suit::Hearts->value` is the literal; `UnitEnum::$name` and `BackedEnum::$value` exist;
  `'strval'` is a callable; `new $class()` on a `class<T>` never fails on `T` being abstract;
  `null !== ($x = f())` narrows `$x`; a closure's by-reference uses write back; `new class (…)`
  passes its arguments to the constructor; `iterable<K, V>` is `array<K, V>|Traversable`; a
  conditional return type sees the defaults of omitted arguments (`explode()`, `microtime()`,
  `array_shift()`, `array_map()` preserving lists, `preg_replace()`).
- **What the `tequila/core` dogfood taught (2026-10-30**, first pass over the plain PHP sources,
  754 → 538 findings on 810 files, PHPStan parity kept): PHPStan's **object shapes**
  (`Logger&object{records: list<string>}`) are a type, matched against a class's declared
  properties; **a method call without arguments is a narrowable path** (`assertInstanceOf(…,
  $e->getPrevious())`), dropped as soon as another method is called on the same receiver;
  `match ($event::class) { Opened::class => … }` narrows `$event`; a **typed by-reference
  parameter retypes its argument** (`?Context &$seen`) and what it writes is not certain
  (checker.md's native rule) — so `is_string($out)` after an out parameter is not dead; `$x !==
  $y` with `$y` one of several values learns nothing in the else branch; `??` is reported as
  never-null only for a certain variable, never for a property or offset (uninitialised, missing:
  PHPStan's `isset` reading); a callable returning `mixed` takes a `void` closure; `(array)` is
  typed per member; a mixed array key is reported on write, as PHPStan does at level max; a
  spread argument binds to the variadic parameter and is checked as its elements; `[]` has no
  keys or values (`never`), so `array_push` and a loop over writes type the array by what was
  added. Overlay: `hrtime`, `parse_url`, `array_push`/`array_unshift`, `mb_substitute_character`,
  `STDIN`/`STDOUT`/`STDERR`, `PHP_INT_SIZE` as `4|8`.
- **Second pass the same day** (538 → 502 on the migrated `.phn` tree, 388 outside the PHPStan and
  Rector tooling directories): a **by-reference parameter follows PHPStan's functionMap
  distinction** — php-src's are read-write (`sort`, `end`, `ksort` leave the type) unless the
  overlay says `@param-out`, user code's are writes (the parameter's type, `mixed` when untyped);
  **passing a value as an argument drops the narrowed method-call paths** under it (`render($event)`
  may change `$event->getResponse()`); `is_array($callable)` is the callable array `array{0:
  object|string, 1: string}`, `is_string` the callable string; `'' !== $s` leaves
  `non-empty-string`; a range comparison (`$n < 1`) is dead only when both operands are certain
  (`@param positive-int` over `int` is not); `self<T>` and `static<T>` in a generic class check
  against its parameters; a tuple may start with a nullable element, `(?T, string)`; **`new Bag()`
  infers the class's type arguments from the expected type** (`$this->bag = new Bag()` with the
  property typed) and generalises inferred literals (`new ArrayObject(['a'])` is
  `ArrayObject<int, string>`, as PHPStan does); a `never` type argument fits any invariant
  parameter (`new ArrayObject([])`); a `foreach` over a non-empty array exits through a `continue`
  as well as through the body's end. Overlay `spl.php`: `Traversable`, `Iterator`,
  `IteratorAggregate`, `ArrayAccess`, `ArrayObject`, `ArrayIterator` and `Generator` with PHPStan's
  template parameters, so lifted `@implements IteratorAggregate<K, V>` is syntax that resolves.
- **Third pass** (502 → 363 on 775 files, the analyser fixture directories excluded with the new
  `--exclude`): a **typed constant keeps its literal** (`const int VERSION = 1` as a default is
  `1`, so `@param positive-int` accepts it); **`[$this, 'method']` is a callable** array where a
  `callable` is expected, as PHPStan accepts the shape; `match ($this)` in an enum is exhaustive
  over its cases; a property is found on any member of an intersection (`Refusal&UnitEnum`'s
  `$name`); a trait's body may use the protected members of its users' ancestors
  (`static::getContainer()` in a trait every user of which extends `KernelTestCase`); **`continue`
  in a `for` runs the step**, so `for ($i = 1; ; ++$i)` does not loop on `$i = 1`;
  `RecursiveIteratorIterator` forwards unknown methods to the inner iterator (`__call` in the
  overlay).
- **After the third pass (2026-10-31):** `json_encode()` with `JSON_THROW_ON_ERROR` among its
  flags (a `|` chain of constants, positional or named) cannot return `false`, as PHPStan's
  dynamic return type extension knows; `@phpstan-assert[-if-true|-if-false] =T $x` is the
  **exact** form (the variable becomes `T`, not the part of its type that is a `T`), which the
  docblock reader rejected with `P4001`; a union absorbs an intersection of one of its members
  (`T|(T&int)` is `T`, what `is_int($x) ? … : …` on a type parameter produced); `strcmp()` and
  `strncmp()` are binary-safe and take `string|bytes` (strings.md §6).
- **Fourth and fifth pass (2026-10-31, 175 → 11 on 774 files, with `main` at PHPStan level max):**
  every finding was traced to a rule PHPStan applies, and the fixtures `inference.phn` and
  `aliases.phn` pin them against PHPStan. **Closures:** a parameter's declared type is narrowed
  by what the caller passes (`fn (array $row)` over `list<array{id: int}>` reads an int) and a
  declared return type is refined by what the body returns; a callable string is the function it
  names; `array_map` over several arrays gives its callback one value of each; `array_filter`
  keeps what its callback's condition (or a predicate such as `is_string(...)`) narrows to.
  **Narrowing:** `is_string($data[$field] ?? null)` with `$field` one of a few literals narrows
  each offset; destructuring reads narrowed offsets; a method or static call with constant or
  variable arguments is a path (`assertNotNull($e->last(X::class))`), `$events[$i]` too, dropped
  when the variable is assigned; `assertTrue($cond)` applies the condition; a `/** @var */`
  before any statement retypes the variable; a by-reference closure use is also what the body
  assigns to it. **Types:** `mixed` is accepted for an unbounded type parameter; a class with
  `__invoke()` is callable; `new self()` satisfies `static` in a final class; `static` on a
  `class-string<T>` receiver is `T`; conditional types resolve however deep; a class yields what
  the `Traversable<K, V>` it inherits says; a shape with more keys than the parameter names is
  accepted; a shape `is_array()` made of a callable stays callable; `min()`/`max()` give ranges,
  `sprintf()` with text a non-empty string, `filter_var()` what its filter validates,
  `preg_match()` with `PREG_OFFSET_CAPTURE` pairs. **Type parameters:** a constructor's
  `@template T` kept on a promoted property stays open so a call through the property infers it
  (the odm's `ReflectorInterface<ClassType, UnitType> $reflector`), but is its bound when the
  property is passed on as a value, as PHPStan reads it; such foreign parameters get fresh
  identities per call so a caller's `T` and a constructor's `T` never meet by name; a signature
  inherited from a parent or an interface brings its templates into scope; a parameter left to
  its default binds its template to the default's type (`B $behavior = self::EXCEPTION` decides
  `B is 0|1`); `self::`/`static::`/`parent::` are told apart by what is written, so
  `parent::method()` maps the parent's parameters; `ReflectionClass` is covariant and
  `isSubclassOf(Foo::class)` makes it `ReflectionClass<Foo>`. **Program:** php-src's `UNKNOWN`
  constants carry the values PHP defines (`stdlib/constants.php`, the extension loader for the
  rest), the machine-dependent ones typed, not valued, as PHPStan's `dynamicConstantNames`;
  `class_alias()` at the top level makes the old name the class and the `if (false)` declaration
  is not checked; PSR-4 lazy loading finds `Foo.phn`; a type written with type arguments is not
  refined by a `@var` beside it; `?string $x = null` keeps its null under a `@param string`.
  **Deliberately not matched** (vault `open-items`): PHPStan's regex analysis (a `preg_match`
  subject becoming `non-empty-string`), `Closure::call()` rebinding `$this`, expression-level
  throw points in `try`, parameter unions over a union receiver, phpredis's `multi()` chain,
  `@template B of self::*_REFERENCE`.
- **A key whose type is known to be numeric is an int key** (2026-11-01, tq-cli's `Val::map()`).
  PHP stores a string that is a canonical decimal integer as an int key (`'123'`, `'-7'` become
  ints; `'0123'`, `'1.5'`, `' 1'`, `'1e3'`, `'-0'` and anything beyond `PHP_INT_MAX` stay
  strings: `ZEND_HANDLE_NUMERIC_STR`), and under `strict_types` such a key handed on to a
  `string` parameter throws `TypeError`. The checker applies the conversion wherever the key's
  type proves it: a literal (`$b['42'] = 1` is `array{42: 1}`, as in PHPStan); a
  `decimal-int-string`, PHPStan's name for a canonical integer string and what `(string)` of an
  `int` is typed (an `int` literal casts to its literal); the int part of a `numeric-string`
  key, `int|numeric-string`, as PHPStan has it. A `(string)` cast written as the key is looked
  through member by member, before a union absorbs the decimal string into `string`:
  `$out[(string) $k] = $v` with `$k: int|string` writes an `int|string` key, so a helper that
  declares `map<string, mixed>` while building its result that way gets the finding at its own
  `return`, where the declaration is untrue, not at its callers. Reads (`$a[(string) $id]`
  reads the int key), array literals with computed keys, and the array types built by
  substituting a template into a key position (`array_combine`, `array_fill_keys`,
  `array_flip` over decimal strings) follow the same rule. PHPStan applies it to constants
  and to `decimal-int-string` keys it writes, but loses the int part of `(string) (int|string)`
  and keeps a `decimal-int-string` key after template substitution: there the checker is
  stricter. **A general `string` key stays a `string` key** — the residual hole, kept on
  purpose: full soundness would make every `$out[$name] = …` produce `int|string` keys, so a
  `map<string, …>` could hardly be built from anything but literals. Hack closed the hole with
  a `dict` that never converts its keys, which PHP's arrays cannot be; TypeScript documents a
  hole of the same kind for index access. A string key read back from an array (`foreach`,
  `array_keys`) is never a decimal integer, so the common re-keying loops are sound as they
  are; a later analysis rule (M6) may flag string keys that come from input if the hole bites.
- **Offsets and iteration on what may have neither** (2026-11-02, tq-cli's tests). An offset
  read, write, append or `unset` on `mixed`, and an offset read on `object`, `iterable` or a
  type parameter bounded by nothing, `mixed` or `object`, is `P2302`; destructuring `mixed` is
  `P2302`; a `foreach` over `mixed`, `object` or such a type parameter is `P2303`. PHPStan
  reports each at level max (`offsetAccess.nonOffsetAccessible`, `offsetAccess.nonArray`,
  `foreach.nonIterable`; the `object` and `iterable` reads already at level 7), inside
  `isset()`, `??` and `empty()` too: those forgive a missing key, not a value without keys. A
  narrowed path is never reported — `is_array($m)`, `isset($m['k'])` for that same offset,
  `array_key_exists('k', $m)` (which narrows `mixed` to an array with the key), and `$a[$k]`
  with `$k` one of a few literals after a check narrowed those offsets one by one — nor is an
  undefined variable under `isset()` or on the left of a nested write, where PHP creates the
  array. A chain reports every level that may be refused: `$m['a']['b']` is two findings, as in
  PHPStan and as `$x = $m['a']; $x['b']` is, because guarding `$m` leaves `$m['a']` mixed.
  Not matched: PHPStan reports `$r = []; $r['a']['b'] = 1` (the offset of `array{}` read as
  mixed), which PHP runs without complaint. (An append to an `object` that seemed to be a
  PHPStan silence on the first probe is reported by it too; the probe had written an offset of
  the same variable before, which PHPStan then trusts.)
- **A variadic is a map inside its function** (2026-11-02). `T ...$xs` holds
  `array<int<0, max>|string, T>`, not `list<T>`: since PHP 8.0 named arguments collect under
  their names (`f(a: 'x')` hands the body `['a' => 'x']`), and PHPStan types the parameter so
  on PHP ≥ 8.0 (2.2 and 2.3; `list<T>` with `phpVersion` below 8.0; 2.1 says
  `array<int|string, T>`). Handing it to a `list<…>` parameter is a finding; `array_values($xs)`
  is the list, and method syntax on a variadic is `Map`'s (`$xs->values()`). What a caller
  passes is unchanged — `T` values, or a spread of any iterable of `T` — and the same holds in
  closures, including a closure literal checked against `Closure(T...)`.
- **The rest of the family: members, spreads, destructuring, `unset`** (2026-11-02, PHPStan
  2.2.5 at level max). A method call, a property (read, write, under `isset()` and `??` too) or
  a static member through `$x::` on `mixed` is `P2105` (`method.nonObject`,
  `property.nonObject`, `staticMethod.nonObject`, `classConstant.nonObject`,
  `staticProperty.nonObject`); a method or property of `object` is `P2006` / `P2005`
  (`method.notFound`, `property.notFound`; outside `isset()` and `??`, where PHPStan is silent
  on `object`), unless `method_exists()` / `property_exists()` on the same path proved the
  member (PHPStan's `hasMethod` / `hasProperty`). Spreading `mixed`, `object` or an open type
  parameter is `P2303` (`arrayUnpacking.nonIterable`, `argument.unpackNonIterable`), and the
  spread's elements are then not checked again. Destructuring anything but an array or an
  `ArrayAccess` is `P2302` with the whole type in the message — a scalar, a string, `null`,
  `array|false`, `?array`, `iterable`, `Traversable`, `object` (`offsetAccess.nonArray`).
  `unset()` of an offset reports what reading it would on an object type, and on a type made
  only of scalars, strings and `null`, where PHP throws, says so (`unset.offset`); PHPStan stays
  silent on `int|string`, the checker does not. An offset on an interface or abstract class that
  is no `ArrayAccess` is `P2302` like any other class — the exemption of 2026-10-30 believed
  PHPStan silent there; 2.2.5 reports it — and on any class under `isset()`, `??` and `empty()`
  too (scalars stay silent there, as in PHPStan); an intersection with one `ArrayAccess` member
  has offsets. A member or offset of a value already reported (an unknown property, `$this` in
  a static context) has the error type, so a chain reports its first mistake only.
- **A static variable is what its `@var` says, `mixed` without one** (2026-11-02), as PHPStan
  types it. Its value outlives the call, so the initialiser describes the first call only:
  `static $cache = []` then `$cache[$k] = …` holds entries on every later call, which the
  initialiser's `array{}` denied (the checker used it until now). Inferring the union of the
  initialiser and every value the body leaves behind would be sound and more precise, but needs a
  fixpoint over the whole body for a construct none of the 1,800 dogfood files uses; PHP leaves
  a static variable no native type, so the `@var` tag is its declaration, in line with the rule
  that declarations carry types (`P2516`). The findings an undeclared static variable then gets
  where it is used say that its declaration is missing, which is true; PHPStan's are the same.
- **What PHPStan infers and core relied on** (2026-11-05, core's last eleven findings). Each is
  PHPStan's reading, taken from its source, because the code it accepts must check here too:
  - *A successful `preg_match()` narrows its subject* to a `non-empty-string` when the constant
    pattern's every match consumes a character (`RegexGroupParser`). The pattern is read for its
    least match length; a conditional, recursion, free spacing or `\K` narrows nothing.
  - *A method on a union receiver is one merged method* (`UnionTypeMethodReflection`): each
    parameter takes the union of the members' types, the result the union of their results, when
    the members' methods line up. Otherwise each member is checked.
  - *A benevolent result* (PHPStan's function map, `__benevolent<…>`: phpredis, DOM's factories,
    `Closure::bind()`) takes a method call on its object members. Only directly: kept in a
    variable, it is an ordinary union, which is stricter.
  - *`Closure::call($o)` binds `$this`* to `$o` and its class's scope (`enterClosureCall`).
  - *A nested call infers its own templates*: contextual typing of its result leaves out an
    expected type that mentions a template the enclosing call is still inferring, so a
    constructor's template kept on a property resolves in `$m->a($m->b($x))` as in `$m->b($x)`.
  - *A constant wildcard matches as a glob*: `Foo::*_SUFFIX` and `Foo::*PART*` as well as
    `Foo::PREFIX_*`.
  - *A catch starts from where the try body can first throw*: after the leading statements that
    throw nothing, with the variables the first throwing statement binds by reference before it
    calls anything.
  - *An assertion on a call decides the call's remembered result*: a docblock assertion is not
    certain, so a later `assertFalse()` of a call an earlier `assertTrue()` fixed is not
    impossible, and what follows is reachable.
  - *`is_subclass_of()` takes a class name unless told not to*
    (`IsSubclassOfFunctionTypeSpecifyingExtension`): `$allow_string` defaults to true there and
    to false for `is_a()`, so a string it answers true for is `class-string<Foo>`. Read as
    `is_a()`, the string narrowed to `never`, and an array keyed by it held nothing.
- **Type tests are PHP's own** (2026-11-09, roadmap D26, `generics.md` §11.5). With `is` and `as`
  gone, the predicates of §3.4 are the only type tests, and the checker narrows through them as
  PHPStan does. A guard of the programmer's own is a function or method with `@phpstan-assert`,
  `@phpstan-assert-if-true` or `@phpstan-assert-if-false`, honoured since M2 on functions,
  methods and static methods (the exact `=T` form above, PHPUnit's assertions through the
  overlay); nothing is added for it. One predicate is Phaenon's reading of a PHP call:
  `mb_check_encoding($b, 'UTF-8')`, with the encoding written as a literal naming UTF-8, narrows a
  `bytes` operand to `string&bytes` in its then-branch and leaves it `bytes` in the else-branch
  (`strings.md` §6); without the literal it narrows nothing, since `mb_internal_encoding()` decides
  at run time. PHPStan has no `bytes`, so parity is not at stake there. A leftover `is` or `as` is
  `P1001`, and `P3011` and `P3012` are retired.
- **An override is held to every declaration it overrides, and to Phaenon's types** (2026-11-13,
  completed 2026-11-15; `P2502`, PHPStan's `method.childParameterType` and
  `method.childReturnType`). A method is compared with the nearest declaration up its parent
  classes, even where an interface sits nearer, and with every interface's declaration of it; the
  first one it breaks is the one reported, so a method that breaks its parent and an interface
  the parent implements is one finding, against the parent. A parent class's private method is
  overridden by nothing, and a farther parent's method is the business of the nearer class that
  overrides it, but for a constructor's signature.

  A constructor is free of a concrete parent's signature, as PHP leaves it, and is held to the
  abstract constructor that parent implements, the nearest one up the parent classes however far
  up it sits, and to every interface's. PHP holds it to the top-most abstract constructor, its
  prototype; the two differ only where a nearer abstract constructor widened a farther one, and
  there the override is held to the nearer, as PHPStan holds it to the nearest constructor's
  signature. A constructor's `final` and its visibility are compared with the nearest
  constructor's, as any method's are.

  PHP's rule is applied to the written types erased, docblock types included, not to the native
  types PHP sees, and stays as `main` had it: an override whose erased written type does not fit
  the overridden one's is an error. Beyond it an override is an error where its written types are
  certainly apart from the overridden member's. So `foo(list<string> $x)` under
  `foo(list<int> $x)` is an error, while `map<string, mixed>` or a shape where a vendor parent has
  a bare `array`, `class<User>` for its `class-string` and a method type parameter the override
  renames all pass. A written type that may fit and does not erase to a fit is an error all the
  same: `User|null` for `User`, or `iterable<int>` for `list<int>`, both under a native `mixed`,
  is `P2502`, where PHPStan, which reports the "no" of its three-valued comparison and lets the
  "maybe" pass unless `reportMaybesInMethodSignatures` is on, is silent. Two types are apart part
  by part:
  - a type parameter stands for its bound, and a union is apart from a type when each of its
    members is;
  - arrays, shapes and iterables are compared by key and value, entry by entry where a shape
    names them;
  - two classes neither of which extends the other are apart, unless one is an interface that a
    subclass of the other could add;
  - a type argument follows its variance, and an invariant one must be the same type, except
    `int|string`, which cannot be told from PHPStan's benevolent `array-key`, and one with a
    member the comparison cannot decide (`mixed`, `never`, `static`), which goes with any;
  - two different enum cases are apart, of one enum or of two; an alias constant of a case is
    that case (`const Red = self::Hearts` makes `Suit::Red` a `Suit::Hearts`); a case meets a
    class as its enum does;
  - a class is compared with a generic ancestor by the arguments it passes up its `@extends` and
    `@implements` (`ArrayIterator<int, Cat>` is no `Iterator<int, Dog>`). An ancestor given no
    arguments decides nothing. A generic class written without its arguments is the class with
    its bounds, and an argument left out takes its bound with the arguments before it: the
    `TIterator` of a `LimitIterator<int, Dog>` is an `Iterator<int, Dog>`;
  - a Traversable class is compared with an iterable by the key and value its Traversable
    ancestor is given (`Generator<int, Cat>` is no `iterable<Dog>`), read from its `@implements`
    and `@extends`, where PHPStan reads its `getIterator()`'s declared type. The two part in both
    directions: a class declaring `@implements IteratorAggregate<int, Open>` whose
    `getIterator()` returns a bare `Iterator` is reported against an `iterable<string, Open>`,
    and PHPStan is silent; a class passing `IteratorAggregate` no arguments whose `getIterator()`
    declares an `Iterator<int, Cat>` passes against an `iterable<Dog>`, which PHPStan reports.
    `IteratorAggregate::getIterator()` is declared `Traversable`, as PHP and PHPStan declare it;
  - callables are compared by signature.

  Static methods are held to the same rule, as PHP holds their signatures to their parent's.
  PHPStan reports a static method whose native types break its parent's and lets its docblock
  types pass; the checker holds them to the docblock rule as well, stricter than PHPStan, as the
  parity rule allows. Why: a list, a map, a tuple and a shape all erase to `array`, so the erased
  comparison alone passed overrides whose compiled docblocks PHPStan rejects, and the checker may
  not be more lenient than PHPStan. The rule on the erased written types is kept, though it is
  stricter than PHPStan: comparing the native types instead, with the test of two types being
  apart for the rest, would lose real findings, such as a `Repository<User>` whose `find()` says
  `?User` where its parent's promises the `T` that is `User` there, and that test does not decide
  every kind PHPStan's "no" covers (a class that is not Traversable against an iterable), which
  the erased rule does. Its known rough edge is `?T` for `T`: it passes where `T` is unbounded,
  both erasing to `mixed`, and is reported where `T` is bounded or closed, so an override of
  PHP's own `current()` or `key()`, or of an `offsetGet()` declared without null, that adds
  `null` is an error under a closed argument (`ArrayIterator<int, User>`). The nearest class
  method comes first because it is the one PHP links the class against, the method its parent
  holds under the name, wherever an interface sits; every interface's follows because PHP links
  the class against each of them, and a class whose two interfaces disagree fits one at best. What
  the check does not reach yet is in §10.
- **A shape is held as PHPStan holds it** (2026-11-15; `P2103`, `P2201`, `P2202`, PHPStan's
  `argument.type`, `assign.propertyType`, `return.type`). Building named shapes found the checker
  more lenient than PHPStan about what a shape accepts, in two places no suite could see: the
  parity run reads no file with a `type` declaration, and none it reads passed a general or an
  unsealed array for a shape (`collections.md` §6.8). The rules are PHPStan's, established by
  probes of what it reports at level max and by its source (`ConstantArrayType::accepts()`, and
  `inferTemplateTypes()` for `generics.md` §3.4); PHPStan 2.2 and 2.3 give the same answers.
  PHPStan asks the value it is given whether it has each key the shape names, yes, maybe or no;
  it lets a maybe pass for an optional entry only, and then holds the value that would be there
  to the entry's type with no maybe allowed:
  - *A general array* might have any key of its key type. An entry its keys can reach must be
    optional, but offset 0 of a list that is not empty, and must accept the array's value type
    outright: `array<string, mixed>` and `array<string, int|string>` fail an `int` entry. An
    entry its keys cannot reach is passed over when optional. So `array<string, int>` fails
    `array{a: int}` and passes for `array{a?: int}`, `array{a?: int, ...}` and `array{}`, and
    `non-empty-list<int>` passes for `list{int}`. The rule of 2026-10-29 above accepted any
    general array whose keys and values could be the shape's; no parity run held it to PHPStan.
  - *An unsealed shape* may hold any key it does not name, with any value: PHPStan's answer for
    such a key is maybe, and the value `mixed`. So `array{id: int, ...}` fails
    `array{id: int, name?: string}` and passes for `array{id: int}` and
    `array{id: int, name?: mixed}`. A sealed shape with more keys than the expected one passes,
    as before.
  - *A list shape takes a list*, as `list<V>` does. Here the checker is stricter than PHPStan,
    which holds a general array to the entries alone: an `array<int, int>` may be `[5 => 1]`,
    which no list is.
  - *A shape unpacked into an array literal keeps its entries*, as PHP spreads them and PHPStan
    types them: integer keys numbered anew from where the literal stands, string keys kept and a
    later one winning, an optional string entry joined with what the literal held under its key.
    `[...$pair]` of a tuple is the tuple; it had been a general array that the old rule happened
    to accept.
  - *`intval()` of a bool or of null is `int<0, 1>`*, PHPStan's `0|1`. mongodb-odm caches a pair
    under the int of a flag, `$cache[$class][intval($allowAbstract)] = …`; under the stub's `int`
    the write made a general array of the pair, which the stricter rule refused, four findings
    PHPStan does not make on the compiled factories.

  Why PHPStan's rules, which the parity rule would let be stricter: they are its reading of
  "might have the key", which is sound, and code PHPStan passes at level max should pass here.
  The one acceptance that may fail at run time, an unsealed shape for `array<K, V>`, was already
  the checker's and is PHPStan's; it stays. The six dogfood trees give the same findings, and the
  same compiled output, before and after. `shapes-held.phn` holds every case to PHPStan in the
  docblock spelling, `shapes.phn` the same cases in the syntax of a named shape, wrong programs
  among them, which the emitter's goldens, being correct programs, could not.
- **The long casts are refused where they stand** (2026-11-16; `P1001`). §7 removes them, but
  the lexer reads `(integer)` as the token of `(int)`, and the cast's typing returned for that
  token before the report behind it ran: all five compiled as written, and `(real)` reached an
  output PHP 8.5 refuses to parse. The report now sits in the file-level pass of the withdrawn
  spellings, since a cast in an array key is typed on another path and one in dead code on
  none. It is made once per cast, in any letter case and with the blanks PHP's scanner admits,
  and names the cast normalised. `(unset)` is refused with them: PHP 8.0 removed it, and the
  message says what to write, `null` for its value or `unset()` as a statement. `phn fmt` and
  `phn migrate` write the canonical cast for a long one; `(unset)` has no certain rewrite and
  stays as written.
- **A null for a built-in's omitted argument is refused** (2026-11-16; `P2103`, PHPStan's
  `argument.type`). php-src's stubs declare `implode(string|array $separator, ?array $array =
  null)` and `strtr(string $string, string|array $from, ?string $to = null)`: the null stands
  for the argument left out, the one-argument `implode()` and the two-argument `strtr()`.
  Reading the stub as written accepted `implode(',', $nullable)`, which PHP throws a TypeError
  for and PHPStan reports, its function map giving each call form a signature of its own. An
  argument given for such a parameter is now held to the type without null. The parameters
  are a table in the checker, found by running PHPStan 2.3 on a nullable argument for every
  function and method its map gives several signatures and the stubs a `?T $x = null`:
  `implode()` and `join()`'s `$array`, `strtr()`'s `$to`, `phpversion()`'s `$extension`,
  `strtok()`'s `$token`, `stream_context_set_option()`'s `$option_name`, `PDO::query()`'s
  `$fetchMode`, `XSLTProcessor::setParameter()`'s `$value` and `ReflectionMethod`'s `$method`.
  It holds for the stubs' symbols only: a function of the same name in user code is read as
  declared. PHPStan's other refusals by call form (a separator that is no string before an
  array, a string alone) are not modelled yet. The lowering that showed the gap is in
  `php-versions.md` §5.3.
- **`unset()` of an offset types the array as PHPStan does** (2026-11-16; `P2103`, `P2201`,
  `P2206`, `P2313`). `collections.md` §2.2 has a list become a map at `unset($xs[$i])`; the
  compiler left it a list, so the list went on to a `list<T>` parameter where PHPStan reports
  `argument.type`. The statement had reused the narrowing of a failed `array_key_exists()`,
  which also made `never` of a shape whose required key was unset, and after that anything was
  accepted. The array now takes PHPStan 2.3's type: a list is an `array<int<0, max>, T>`; no
  array is known to be non-empty afterwards; a shape loses the entry a constant key names, or
  has optional every entry a key that is not constant may name, and is no list afterwards. An
  offset inside another array writes each array above it. Where the root is a property the
  unset is an assignment to it and checked as one; a property holding an `ArrayAccess` object
  is not written. One place is stricter than PHPStan, as the parity rule allows: where an
  array's values are shapes, PHPStan writes back only the entries the shape still has and so
  keeps the entry removed (`unset($rows[$k]['name'])`), and the checker gives the rows the
  shape without it.
- **An offset a shape might lack is `P2321`** (2026-11-16; PHPStan's `offsetAccess.notFound`,
  "might not exist"). Reading an optional entry, a key an unsealed shape does not name, a
  destructuring of either and a by-reference argument that reads the offset all checked clean.
  `P2321` stands beside `P2313` as `P2009` "might not be defined" stands beside `P2008`: `P2313`
  is the offset no value of the type has, `P2321` one that some value may lack. It is reported
  where PHPStan reports: a constant key or a key of a few literals, every array the type may
  be a shape, and one of them with the key optional, unsealed without it, or without it beside
  others that have it (that last case was `P2313` on the shape that lacks it). It is not
  reported where a general array is among the types, for an offset another variable names
  unless its type is a union of literals, for a key taken from the array itself, inside
  `isset()` and `empty()`, on a write, or for a reference that creates the key. The read is
  typed as the entry, so the missing guard is the one finding. *Since 2026-11-17:* a type parameter bounded by a shape is
  read as its bound, and a write into such a variable makes it the written shape, no longer
  the parameter; an `ArrayAccess` beside the shapes is read by the key its `offsetSet()` takes
  (it may hold any such key, so the shapes decide, and a key it does not take it has not),
  while any other object beside them is `P2302` alone; a reference to an offset,
  `$x = &$r['tags'][0]`, writes that offset only, and the arrays it is in are read.
  - *A test narrows the array, not a path.* A test of a constant offset, `isset()`,
    `!empty()`, `?? throw`, `!== null` or a type test, gives the array holding it the entry as
    present, its value narrowed, and so each array above it up to the variable or property,
    as PHPStan's `HasOffsetValueType` does. Until then a test narrowed the path `$r['name']`
    alone, and the first write to another key, an increment or a copy forgot it: a false
    `P2202` on `main`, and a false `P2321` had the new code been built on it. This is the
    reading §10 set aside on 2026-11-15, taken up for shapes only, where the narrowed type is a
    shape and needs no typed rest; `unset()` takes the key out again, under any spelling. What
    §10 held against it, a write through a reference to the array that the checker did not
    see, is seen since 2026-11-17 (below).
  - *A write through a reference to a variable writes both* (2026-11-17; `P2313`, `P2321`).
    After `$alias = &$r`, an assignment to either variable, a write to an offset of it or an
    `unset()` of one is applied to the other, through a chain of references as well, so that
    `unset($alias['name'])` takes the key from `$r`. The reach is PHPStan's
    (`IntertwinedVariableByReferenceWithExpr`): a reference to an offset (`$x = &$r['k']`) and
    one a closure takes with `use (&$r)` are not followed, and a link made on one way into a
    join only is gone after it. PHPStan keeps the link after `unset($a)` and after
    `$a = &$c`, where PHP breaks it; the checker keeps it too, since following PHP there would
    leave it more lenient than PHPStan on those lines.
  - *An offset is keyed by the key PHP stores.* `$a['1']`, `$a[1]` and `$a[0x1]` are one key,
    and so are `$a['name']` and `$a[self::NAME]`; a constant read through `static::` is one
    only in a final class.
  - *Other tests PHPStan understands narrow as well*, each found as a false finding on guarded
    code: a remembered test (`$has = array_key_exists('k', $a)`, `$n = $a['k'] ?? null` where
    `$n` is then not null), `match` and `switch` over a test or over `count()`, `count()`
    covering every entry of a positional shape, a non-empty shape whose one entry is optional,
    what a closure captures and what `array_filter()`'s callback proved.
  - *A key of a few literals reads the entries it may name* (2026-11-17). `$r[$k]` with `$k`
    typed `'a'|'c'` is `P2321` where one of the entries may be missing and `P2313` where none
    exists, as for a single key; `true`, `false` and a float literal are the int PHP stores
    them under. A key taken from the array itself is one it has, in the forms PHPStan follows:
    `foreach (array_keys($r) as $k)` with the call written as the iterable,
    `$k = array_key_first($r)` or `array_key_last($r)` (at once where the array cannot be empty,
    else where `$k` is not null), `array_search()` where the result is not false,
    `array_find_key()` where it is not null, `array_rand()` on an array that cannot be empty,
    and `$r[array_key_first($r)]` on a variable that cannot be empty. A variable holding
    `array_keys()` is not followed, nor an assignment a condition compares with null, as
    PHPStan follows neither. `array_key_exists($k, $r)` with such a key says the array has
    one of the keys, and `isset($r[$k])` narrows the offset as written as well as each
    literal's.
  - *A change takes out of a remembered test what it makes stale, and no more* (2026-11-17).
    `$ok = isset($a['k']) && $x !== null;` followed by `sort($a)`, `$a['j'] = 5`,
    `unset($a['j'])` or `unset($a)` dropped everything `$ok` implied, so `$x->m()` under
    `if ($ok)` was a false `P2105`. The change removes the paths and variables it changes from
    the test and keeps what the test says of the others; a join keeps what both sides still
    say. This holds for every narrowing remembered through a variable, not only offsets.
  - *An array a loop, a spread, `array_merge()` or a write under a key of a few literals builds
    keeps its shapes* (2026-11-17). A loop keeps every key a shape of the array has. A spread
    and `array_merge()` collapse an argument that is a union of shapes into one, its keys
    optional where a shape lacks them. PHP numbers integer keys anew: a spread without a
    string key is a list, its last entries optional, and `array_merge()` is a list where
    every argument has integer keys alone. `+` and `array_replace()` keep the keys, and are
    a list where every operand is one (a shape whose keys number its entries from 0 is, with
    at most its last entry optional): the longer list's numbers follow the shorter's. An
    append after optional integer keys may land at each number they leave open
    (`list{0: 1, 1?: 2}` and 3 is `list{0: 1, 1: 2|3, 2?: 3}`). A
    write under a key of several literals outside a loop adds each as an optional key of
    every shape. A join keeps the union, which `P2321` reads as PHPStan reads the shape it
    merges them into.

    Joins double the shapes of an array. Below 4096 entries in all, sixteen times PHPStan's
    `ARRAY_COUNT_LIMIT`, they stay as they are. Past it, a write, an `unset()` of an offset
    and an array `+` first merge two shapes one key tells apart into one where the key is
    optional, which loses nothing and is what
    PHPStan's joins do as they go: ten independent `if ($c) { $o['k'] = …; }` are one shape
    with ten optional keys. Past it still, as after ten if/else that each write one of two
    keys, these steps follow in turn: shapes with the same keys and the same tags merge; each
    tag's shapes collapse into one; all of them collapse into one; and only a shape past the
    limit itself becomes a general array. A tag is a key every shape has as one constant
    (`kind: 'a'`), and shapes of different tags stay apart until the last two steps. Those
    two lose what values of different keys said together; PHPStan has a general array there
    already, from 256 entries, and reads its offsets as benevolent unions, which the checker
    has not. `+` pairs each shape of one operand with each of the other, so it bounds the
    pairs as well: where they would pass the limit, each operand's shapes one key tells
    apart merge first, and then each operand collapses into one shape, after which a tag no
    longer tells an operand's shapes apart in the result (PHPStan loses it there too).
  - *A by-reference argument* reads the offset as PHPStan has it. A parameter of user code
    reads it where its declared PHP type takes no null, which is what PHP creates the missing
    key as: `int`, `array`, and not `mixed`, `?array` or no type. One of PHP's own reads it
    where PHPStan's function map has it read and written (`sort()`, `reset()`) or not by
    reference at all (`array_multisort()`); every other creates the key (`preg_match()`'s
    `$matches`, `parse_str()`'s `$result`). The table is derived from the function map.
  - *Where PHPStan reports and the checker is still silent*: a `foreach` over a literal list
    writing under its value as key, which PHPStan runs once for each entry; a key the lattice
    holds as an integer range (`$c ? 1 : 2` is `int<1, 2>`); a key whose union holds a member
    that is no literal (`'a'|'c'|int`); an enum's backing value as a key; a positional item
    after the spread of a shape with optional integer entries (`[...$a, 9]`). (`extract()` is
    a removed construct, `P1001`, and a reference between two variables is followed since
    2026-11-17, above.)

  `offset-might-not-exist.phn` and `unset-offsets.phn` hold each case to PHPStan line by line.
  The six dogfood trees give the same findings before and after both.

- **Extra checks are a kind of finding, with a switch** (2026-11-17, roadmap D33). The checker
  reports three kinds of finding: language rules, parity findings and extra checks. The first
  two are always reported. A parity finding is one the compiled output draws without Phaenon:
  PHP refuses the code, or PHPStan reports it. An extra check is a finding PHPStan does not
  make at level max with `treatPhpDocTypesAsCertain: false`, on by default, off with
  `"extra-checks": false` in `phaenon.json` or `--no-extra-checks` on `phn check`,
  `phn build` and `phn watch`.

  - *What may be one.* A finding that is true and that PHPStan leaves out by choice or by
    imprecision: the code can fail when it runs, or a declared type promises less than the code
    relies on. It is a finding PHPStan does not make of that code, which is not the same as a
    line PHPStan is silent on: a rule of PHPStan's that the checker deliberately lacks (the
    allow-list of §8) may speak on the line of an extra check about something else, as
    `return.unusedType` does on a static method whose documented return type the extra check is
    about. The parity run names each such line with the identifier. The members, each tagged
    where it is reported, with the rule PHPStan judges the place by:
    - `**` on a string, which throws for a string that is no number; PHPStan reports the other
      arithmetic operators on `string`, and this one (`binaryOp.invalid`) only on a string it
      knows the value of, an array, a class or a `mixed`. The string is an untouched parameter
      typed by PHP alone (below), a `string` with or without int, float and null beside it,
      and the other operand is a number written out, an untouched parameter declared an int
      or a float, or another such string: PHPStan reports the expression once for both
      operands;
    - a static method held to the docblock rule of overrides, where the native types the
      source writes fit as PHP compares them: `false` and `true` apart from `bool`, a parameter
      nullable by its default of `null`, a missing return type fitting no declared one.
      PHPStan's `method.childParameterType` and `method.childReturnType` are not applied to the
      documented types of static methods at level max;
    - a list shape that takes no general array, and a tuple that takes no longer one, where a
      value is held to a declared type and PHPStan judges the place by the acceptance of the
      value's type: an argument (`argument.type`), a return (`return.type`), a yield
      (`generator.keyType`, `generator.valueType`), an assignment to a property
      (`assign.propertyType`), the default of a property (`property.defaultValue`) and of a
      parameter (`parameter.defaultValue`), a constant (`classConstant.value`), and the body of
      an arrow function, which PHPStan holds to the function's own return type or, without
      one, to the place the closure is given. The value is an untouched parameter or a
      literal, and the two rules are told by comparing a second time as PHPStan compares
      array shapes. The assignment of an offset is no such place: PHPStan has a rule of its
      own there (`offsetAssign.valueType`), which refuses the general array that an argument
      takes for the same list shape, and the finding stays a parity finding;
    - a power or a sum of integers given directly as an argument, or returned, where the
      value the checker computes does not fit the declared type and PHPStan's wider type does
      (a float where an int is declared): PHPStan types a power of two ints as a benevolent
      union of int and float and a sum as an int, and computes the value only where both
      operands are known values. One operand is an untouched parameter typed by PHP alone, of
      which PHPStan knows no value, and the other an int written out or an untouched
      parameter. An inline `@var` over a returned expression gives the type itself, and the
      expression under it tags nothing.

    Of the other places this section calls stricter than PHPStan, the rows of
    `unset($rows[$k]['name'])` are an extra check by what they say that cannot be marked yet
    (§10), and the array an `is_int($a['id'])` did not narrow is none: that call cannot fail,
    so the finding is a false one. A finding that claims something false is a defect and is
    fixed, not classified. An extra check ends nothing that could find a parity finding: a
    static method that draws one against its parent class is still compared with its
    interfaces, and native types that are apart there are reported beside it.
  - *What proves one* (2026-11-19). An extra check is a claim that PHPStan makes no such
    finding of the code and php runs it, and the place that reports it has to prove the claim.
    Where the checker cannot, the finding is a parity finding and stays with the switch off: a
    doubt costs a line in the parity run's table (§8), never a finding the switch hides. Two
    things are no proof. The checker's type of a value, because PHPStan knows more wherever
    the lattice has let go: it folds a call on constants (`strtolower('ABC')` is `'abc'`,
    `count([1, 2, 3])` is 3), reads a constant's value (`PHP_EOL`, `__FUNCTION__`), and keeps
    forty literals apart where the lattice has an `int` or a `string`. **A range never counts
    by its type**: the lattice writes a union of consecutive values as a range, and PHPStan
    computes with each of a hundred and thirty of them. And what the checker has followed of
    a variable through its function. A mark carried from the function's entry and dropped at
    a write, a narrowing, a condition, a call was built first, and each review found one more
    way by which PHPStan learns of a value that the flow did not model (an assertion tag an
    interface declares for the method called, a condition kept with another variable, an
    assignment under an inline `@var`, a variable written through `$$name`), while comparing
    the marks at a loop's head made nested loops with `continue` exponential. A list of
    positions in which the function may read the parameter, however often, was built second,
    and failed on what such a list cannot see: PHPStan remembers what it was told of an
    *expression* as it does of a variable, so that after `assert(is_float($i ** -1))`, or
    `$ok = is_float($i ** -1); if ($ok)`, a second `$i ** -1` is a float to it and no
    benevolent union. Every one of these needs a second place where the variable, or an
    expression that holds it, is seen. The proof is therefore **the only occurrence**, by the
    syntax of the function alone, computed once and conservative by construction:
    - a declaration compared with a declaration, where no value is involved: the static
      method's rule;
    - a literal, which both read alike;
    - an **untouched parameter**: the read the finding is about is its only occurrence in the
      body of its function. Three things hold then. *The type comes from this one
      declaration*: a method that any class, interface or trait of its hierarchy also declares
      is left out, a constructor included, and so is a trait's own method, because PHPStan
      inherits a docblock by more routes than the checker follows (a trait's abstract method,
      an interface where the body comes from a trait, a parent's constructor) and the rule
      does not chase them; a closure's or an arrow function's parameter, which PHPStan types
      from where the closure is passed; a promoted, a variadic and a by-reference parameter;
      one the docblock names more than once (`@param` beside `@phpstan-param`). *The text of
      the body holds `$name` once*, and that occurrence is a read in one of the places below:
      a docblock above a statement, a comment, a closure's `use` and a nested parameter of
      the name are occurrences too. *The body has nothing that writes a variable without
      naming it*: `$$name`, `extract()`, `compact()`, `get_defined_vars()`, `parse_str()`,
      `eval`, `include` and `require`, `goto`. Where the claim is that PHPStan knows no value,
      the parameter is also **typed by PHP alone**: its docblock does not so much as name it,
      since a native type names no set of values and a docblock may list them. A default
      changes nothing: PHPStan does not take it for the value. A finding over two variables
      (`$i ** $j`, `$s ** $t`) needs each to be untouched, so `$s ** $s` is none.

    *Where the one occurrence may stand.* The list is default-deny: a place is on it because
    PHPStan was given the function with the finding there and was silent on the line.
    - An operand of an arithmetic operator (`+ - * / % **`, unary `+` and `-`) or of `.`, and
      a part of an interpolated string, wherever the result stands that is no condition: an
      argument (of a callee with an assertion tag too), the right-hand side of an assignment,
      the value of a compound assignment, an element of an array literal.
    - What a `return`, an `echo` or a `print` gives, and the body of an arrow function, the
      variable alone or as an element of an array literal there; what a `yield` gives, the
      variable alone.
    - The whole right-hand side of a plain assignment to a property, instance or static.
    - An argument of a call, the variable alone, **outside every loop**. PHPStan judges an
      argument before it applies the callee's assertion tags and before the type a parameter
      taken by reference leaves in the variable, so the one occurrence is judged by its
      declared type. In a loop PHPStan's second pass sees what the parameter by reference
      left, and reports the argument; the checker cannot know from the call that there is
      such a parameter.
    - Around any of these: a statement under a condition that does not hold the variable, a
      loop, a `try` with its `catch` and `finally`, a branch of a ternary, an arm of a
      `match`, a generator.

    Off the list: a target of any assignment, `++`, a reference, `unset`, `global`, `static`,
    a `foreach`, a `catch`; the variable in an array literal that is an argument; any place in
    a condition or in what narrows (`if`, `while`, `for`, `switch`, the subject and the
    conditions of a `match`, the condition of `?:`, `??`, `&&`, `||`, a comparison,
    `instanceof`, `isset`, `empty`, `assert`); the right-hand side of an assignment to a
    variable; an offset or a member of it, a cast, a bitwise operator; a closure. Some of
    these were run and PHPStan was silent (a cast, a condition); they stay off, since no
    finding the checker makes needs them.

    The rule costs every function that names the parameter twice: `echo $s ** 2; echo $s **
    3;` are two parity findings, which the parity run's table names. A wider rule would have
    to keep apart, by syntax, every two occurrences that stand in structurally equal
    expressions or one inside the other, and to show by runs that PHPStan remembers nothing
    else; it is not built.

    The places that may report an extra check are listed in the compiler, each with the proof
    it rests on and the PHPStan rule above, and a test fails for a place that is not listed.
    A place PHPStan judges by a rule of its own is no place for an extra check until that
    rule has been run with the values in question.
  - *An override whose native types are apart* (2026-11-19) is a parity finding, static or
    not, and no extra check. PHP would refuse the types as written. Compiled, they are
    rewritten until they fit where one type can (generics.md §4.3: `(): iterable` over
    `(): array` to `array`, `array $x` under `iterable $x` to `iterable`, under a parameter
    without a type to `mixed`), so the output has no such signature for php or PHPStan to
    refuse; where no rewriting helps (a missing return type, `bool` for `false`), both refuse
    it still. What PHPStan reports of a rewritten one is the body the new signature refuses,
    on the line of its `return`, or nothing where the body happens to fit both. The checker
    holds the body to the types the source writes, so the finding is all that stands before a
    compiled method whose signature its body breaks, and nothing at the report proves that
    the body fits. It is a finding about the source: reported in both modes, and named by its
    line in the parity run's table where PHPStan passes the output. `#[\Override]` on a
    constant or an enum case that no parent declares is the same kind of finding: compiled for
    8.5, which knows the attribute on methods and properties only, the emitter leaves it out
    there and php loads the file; compiled for 8.6 it is PHP's compile error.
  - *How a finding gets its kind.* By its code where the whole code is a language rule: the
    `P1xxx`, `P3xxx` and `P4xxx` codes, and of the `P2xxx` an unused import (PHPStan sees the
    import and has no rule for it, and nothing can fail: a warning of Phaenon's own), a type
    alias that mentions itself, the codes of `throws` and `nothrow`, the missing declared
    types, a map key that is no int or string, bytes used as text. A table beside the codes
    lists them, and a test holds it complete. At the place it is reported otherwise, through
    one entry point for a parity finding and another for an extra check: the same code, `P2301`
    or `P2103`, is a parity finding in one place and an extra check in another. A finding only
    Phaenon's source can draw under such a code, as bytes given for text under `P2103`, is a
    parity finding by kind, which changes nothing about when it is reported. `--format=json`
    carries the kind as its last member, `"kind"`, a JSON string: `"language"`, `"parity"` or
    `"extra"`. The language server gives each diagnostic the same word as `data.kind`. The
    text, GitHub and Checkstyle formats do not carry it.
  - *What off promises.* That the findings marked extra are silent, and nothing about the
    rest: the allow-list of PHPStan rules the checker lacks stays as it is, and a finding
    PHPStan gains or loses in a later version moves between the kinds when the parity run
    says so. Silent is whole: no line, no count, no part in the exit status, and a file whose
    only findings they were is built. The extra checks are found in both modes and left out
    when the check ends, not where they are found, so that a check with the switch off is the
    check with it on, less its extra checks. An ignore comment is therefore held to them in
    both modes: a `@phaenon-ignore` that silences an extra check has something to silence, and
    is no unused comment (`P4015`) while they are off; one that names a code nothing draws on
    its line is `P4015` either way. This is where the extra checks differ from the
    deprecations a run leaves out: those the run does not look for, and holds no comment to
    them.
  - *Who reads the switch.* `--no-extra-checks` is taken by `phn check`, `phn build` and
    `phn watch`, and wins over the file for the run. `phn lsp`, `phn fix` and `phn fmt` go by
    the project file; they and the commands that report no findings refuse the flag, as
    `phn build --vendor` does. `phn migrate` checks with the defaults, and the playground,
    which has no project file, has the extra checks on.
  - *Releases.* A new extra check is a feature and comes in a minor release; closing a gap
    against PHPStan stays a fix (roadmap D31).
  - *An installed package* is checked with its own project file's setting; its findings are
    warnings either way.

- **A constant read through `static::` is its declared type** (2026-11-17). `static::X`,
  `$this::X` and `$object::X` name the constant of the class the value has when the code runs,
  which a subclass may override. Outside a final class (an enum is one) such a constant is its
  declared type, the native one or its `@var`, and mixed without one, as PHPStan types it;
  a final constant, an interface's included, `self::X` and `ClassName::X` are the value
  written. Until then the value written in the class where the expression stood was read, so
  `(static::FLAGS | 4) === 15` was "always false" in a class whose subclass made it true. A
  private constant read through `static::` is mixed too: php gives a subclass's own value
  there, or an error, and PHPStan reports the access itself
  (`staticClassAccess.privateConstant`), a rule the checker does not have yet.
- **A docblock's word for a type is a class name in a declaration** (2026-11-16 for
  `integer`, `double` and `boolean`; 2026-11-17 for `scalar`, `numeric`, `number`, `noreturn`
  and `resource`). Where a type reaches PHP as written (a parameter, a return, a property, a
  constant), PHP reads these words as the names of classes, and every value fails the type.
  There the word is the class or the type alias of that name that an import or the namespace
  gives, in any letter case, and `P2001` where none does, with PHP's spelling of the type in
  the message (`int|float|string|bool` for `scalar`, `never` for `noreturn`; PHP declares no
  type for a resource). PHPStan reports the same as a class that does not exist. Where the
  emitter writes the type into a docblock (a type argument, a closure's signature, a tuple,
  an alias's right-hand side) the words are the types they mean, as PHPStan reads a docblock,
  unless an alias or a class of the name is in scope, which wins there too (roadmap D29).
- **A generic closure is refused until it is designed** (2026-11-17; `P1007`, design issue
  #28). The parser reads `fn<T>(T $x): T => $x` and `function<T>(…)`, and `generics.md` gives
  the closure's type parameters no meaning yet: the checker read the signature in the
  enclosing scope, where `T` was a class of the namespace, and the body in the closure's own.
  Such a closure is one finding at its type parameters, in every form the parser reads, and
  nothing else follows from the parameters, in a type position or as a class's name
  (`instanceof T`, `T::class`, `new T()`). `P1007` is a code for syntax whose meaning is not
  designed yet; `P1001` stays the code of what was removed. A generic function or method, or
  the closure with concrete types, says what such a closure could.
- **A narrowed type parameter is the parameter still** (2026-11-20; `generics.md` §3.2; not
  built yet, §10). A test does not change a value, so a value of `T` that a test narrowed is a
  value of `T`. Before this rule the checker made an intersection of it, `T&int`, `T&Row`,
  `T&null`, which was no `T`
  where one was expected: a false `P2202` at a `return $a` and a false `P2103` at an argument
  after `is_int($a)`, `$a instanceof Row` or `$a === null`. It took nothing away from a
  parameter either: after `!is_int($a)` under `T of list<int>|int` the value was the whole `T`,
  so an append to it was `P2302`, and a member after `$a !== null` under `T of ?Row` was
  `P2105`. Truthiness read a parameter as an object: `if ($a)` was `P2402`, "always true", under
  any `T`. PHPStan has none of these findings. An attempt to mend it in the lattice, with `X&T`
  a subtype of `T` and what is taken away written `T&rest`, was withdrawn: the first made the
  checker more lenient than PHPStan wherever it builds an intersection with a parameter and
  PHPStan has none, the second lost the parameter in every invariant position. The rule below is
  PHPStan's own, read from probes of PHPStan 2.3.4 (some 2,000 functions: a bound, a test, a
  branch, a place that holds the value to `T`), and it is decided where a value is narrowed, not
  where two types are compared.
  - *Three forms.* A value of a type parameter is the parameter as declared, `T`; the
    parameter with a narrower bound, `T of int`; or a meet, `T&Row`, the values of `T` that
    are a `Row`. The type says which, and its bound is in each form what the value is known
    to be. Whatever reads a parameter reads that bound: members, offsets, iteration, `?->`,
    truthiness, operators. So `$a->id` after `$a !== null` reads a `Row`, `?->` on a
    `T of ?Row` is no `P2307`, and `if ($a)` under `T of ?int` has two branches. A fourth
    outcome is no form: the test leaves a type without the parameter.
  - *Which test gives which* depends on the bound, and on whether the test takes a part of
    the bound away or meets it with a type.

    | The bound is | A part taken away: the false branch of a type test, `!== null`, `!== []`, `!== 'x'`, a truthy value, `> 0` | Met with a type: `is_*()`, `instanceof`, `=== null`, `=== 5`, `=== Suit::Hearts`, `in_array(…, true)`, a falsy value, an assert tag |
    |---|---|---|
    | several types: `list<int>\|int`, `?Row`, `int\|string` | `T of` the rest | `T of` the members met, each with the type: `T of int`, `T of Row`, `T of 5` |
    | one type: none, `mixed`, `object`, a class, an enum, `int`, `string`, `bool`, a list | `T of` the rest where the lattice can write it (`T of non-empty-list<int>`, `T of int<1, max>`), `T` where it cannot (`mixed` without `int`, a class without a subclass, an enum without a case) | `T&X`: `T&Row`, `T&int`, `T&5`, `T&Suit::Hearts` |
    | one array that is no list, a shape, `iterable` | the same | the array met, without `T`, where it is another array: `=== []`, `=== ['id' => 1]`, a falsy value, `array_is_list()`, `is_array()` on an iterable |

    A test that refines the one type a bound is, and names no other, is a meet:
    `count($a) > 0`, and a key found present by `array_key_exists()` or `isset()`, give
    `T&non-empty-array<K, V>` on a general array or a list. On a shape an entry found present
    is a narrower bound, `T of array{id: int, name: string}`, and a test of an entry's value
    leaves the shape without `T`. A value already narrowed is narrowed again by the form it
    has: `T of Animal`, what `!== null` leaves of `T of ?Animal`, has one type, so
    `instanceof Dog` makes it `T&Dog`, and a meet stays a meet. The other operand of an
    identity is narrowed by the same rule: `$x === $a` makes an `int $x` a `T of int` under
    `T of int|string`, leaves `$x = ['a' => 1]` what it was under `T of array<string, int>`,
    and makes a value that was written and is then found identical to a `T` a `T` again. A
    `switch` on the value and an identity of two values of `T` narrow nothing, and `==`
    narrows what it did (`== null`), by the same rule. A test spelled another way is the
    same test: `assert()`, an early return, `match (true)`, a ternary, `?? throw`,
    `@phpstan-assert`, a property or an offset as the path tested.
  - *Where PHPStan loses the parameter, the checker loses it too.* Each such case is a
    finding PHPStan draws on the compiled file, and the checker is never more lenient than
    PHPStan there (roadmap D33). Beside the arrays of the table there are two. Identity with
    `true`, also through `in_array($a, [true], true)`, under a bound of several types leaves
    `true`, where `false` keeps `T`. And a bound of several types narrowed to a callable, a
    `class-string` or a float literal alone is one PHPStan has no template type for: it
    forgets the bound, and the value is `T of mixed`, refused where `T` is expected
    (`return $a` after `$a !== null` under `T of callable|null`) and taken in an invariant
    position. None of this is true of the value, which is the one that was tested; it is
    what PHPStan 2.3.4 does, and the corpus holds each case. When a PHPStan release keeps the
    parameter there, the parity run names the line as the checker's alone, and the case goes.
  - *Subtyping.* A narrowed form is a `T` where the declared `T` is expected: returned,
    given as an argument, in `list<T>` or a covariant argument. But for the forgotten bound,
    that holds by how the form was made and asks nothing of the bounds. Against a narrowed
    form the bounds decide: `T` is no `T of int`, and `T of int` is a `T&int`. Against any
    other type a parameter is its bound, as before. In an invariant position, `Box<T>`, the
    parameter with a narrower bound is the parameter, whatever the bound, and a meet is not:
    `new Box($a)` is a `Box<T>` after `is_int($a)` under `T of list<int>|int`, and after
    `$a instanceof Row` under `T of object` it is a `Box<T&Row>`, which is none. That is
    PHPStan's rule, where two template types of one name and scope are one argument of an
    invariant parameter and an intersection is another type. Both halves are needed: the
    first is what the withdrawn attempt broke, the second is a finding PHPStan draws. Two
    narrowed forms are equal when they are the same form with equal bounds. The declared
    parameter is equal by its name alone, as before, so a bound that names its own parameter
    is never walked.
  - *Joins.* Where branches meet, each form is a member of the union, and the declared `T`
    absorbs the forms of its own parameter. After `if ($a instanceof Row) { … }` under
    `T of object` the value is `T`. After `if (is_int($a)) { … } else { … }` under
    `T of list<int>|int` it is `T of int|T of list<int>`: a `T` where one is expected, and
    no `T` in `Box<T>`, as in PHPStan. The ways a condition fails or holds are not branches:
    after `!(A && B)` and after `A || B`, a value that was the parameter with a bound of
    several types is the parameter with the union of the bounds its ways left, since PHPStan
    narrows it by the whole condition (`T of ''|null` after `!(is_string($a) && $a !== '')`
    under `T of ?string`).
  - *Inference.* A type argument inferred from a narrowed value is the form, with the one
    exception PHPStan makes where it generalizes a literal: `T of 5` gives `Box<int>`, the
    parameter lost.
  - *`class-string<T>`* follows its parameter: found a subclass by
    `is_a($c, Dog::class, true)` or `is_subclass_of()`, it is `class-string<T&Dog>`, and
    `new $c()` a `T&Dog`; found identical to `Dog::class` it is that literal, without `T`;
    and `$a::class === Dog::class` narrows `$a` as `instanceof` does.
  - *Printing.* In a message and in the editor's hover a form prints as what it is:
    `T of int`, `T&Row`, `T of int|T of list<int>`. The declared parameter prints `T`, as
    before. In a docblock the compiler writes from an inferred type (the `@var` of a lowered
    closure, `php-versions.md` §5.3) the narrower bound has no spelling and is written `T`,
    which is how PHPStan compares it there; a meet is written `T&\App\Row`, which PHPStan
    reads as the intersection it inferred itself. No declaration holds a narrowed form, so
    signatures, `@template` tags and erasure (`generics.md` §4.1) are untouched.
  - *Writes.* A write into a narrowed value is a write into what it is known to be, and the
    variable holds the array written afterwards, no longer `T`, as after a write into any
    value of a type parameter (`generics.md` §3.2). `$a[] = 1` after `!is_int($a)` under
    `T of list<int>|int` appends to a `list<int>`, and `return $a` is then `P2202` with the
    write named, where PHPStan reports it too. A write into `T&array`, what `is_array($a)`
    makes of an unbounded `T`, is the same. A property written on a narrowed object leaves
    the value the form it had.
  - *Stricter than PHPStan, as before the rule.* A meet with a literal in an invariant
    position: where PHPStan infers `Box`'s argument it generalizes `5&T of int` to
    `T of int`, and the checker keeps `Box<T&5>`, which is no `Box<T>`. Taking it would need
    the bound the meet started from, which the form does not keep. And a general array
    declared non-empty and found identical to a constant array, where PHPStan keeps
    `array{a: 1}&T` and drops the parameter for the same bound once a test made it
    non-empty. The parity run's table names both.
  - *Built in this order.* The rule on writes comes first: without it a narrowed value that
    is written stays `T`, and `return $a` after the write would pass where PHPStan reports
    it. The meet of `string` with `callable` is mended before the rest, since a bound is
    narrowed by the lattice's own meet and that one is `never` today (`is_callable($s)` on a
    string is `P2402`, "always false"). Then the forms, with nothing producing them; what is
    taken away; what is met; the refinements, the readers of a bound, and `class-string<T>`.
    The corpus files are `narrowed-params.phn` (bounds, tests, and the places that hold a
    value to `T`), `narrowed-params-joins.phn` and `narrowed-params-writes.phn`, each branch
    holding a finding that would be missing were the branch dead, and each file held to
    PHPStan in both directions.
  - *Measured* on a prototype of the lattice change, since discarded, with the rule on
    writes under it. Two expectations of the checker corpus change, the two false findings
    that rule's table names: an append after `!is_int()`, and a written value found
    identical to a `T`. Of the probe lines that draw a false finding without the rule, over
    500 draw none; no line that PHPStan reports and the checker reported is passed; and the
    one finding added on a line PHPStan passes came from a reader the prototype had not
    adapted, a method looked up on each class of a bound that is an intersection. The six
    Tequila trees report what they did.
- **A division and a power are typed as PHPStan types them** (2026-11-26). A division or a
  power with a float operand is a float, as PHP computes it: `$f / 2` and `(float) $a / $b`
  were `int|float`, refused wherever an int is not coerced to a float. Of two plain ints,
  `$a / $b` and `$a ** $b` are PHPStan's *benevolent* `int|float`, which the lattice now marks
  and messages write `(int|float)`. Until then `$a ** $b` was an `int`, which no float the
  power may be was held against, and `$a / $b` an ordinary `int|float`, refused where an int
  is declared though PHPStan passes it.
  - *Accepted where one of its members is*: as an argument, a return, a property, and among
    the values of an array or a shape, each such union decided where the comparison reaches
    it. Not inside a class's type argument, which PHPStan holds as a template. *Held as an
    ordinary union where a type is held*, as a `@param-out` is.
  - *Where the mark goes* is PHPStan's. It stays through `+`, `-`, `*`, `**`, unary minus
    and the steps, beside a literal, a range or another such union; a division of it by a
    plain int, or of a plain int by it, is ordinary (`$total / $users / $days`). Where two
    ways meet and one holds an ordinary `int|float`, the union is ordinary. It stays through
    a template, a callback's return and a read through a reference, and a type parameter
    bounded by `int` or `float` is inferred as the member its bound takes.
  - *Written to a property PHP declares `int` or `float`*, the value is remembered as that
    member, as PHPStan remembers it. So that this draws no "always false", a type test of a
    property or an offset the function itself wrote or narrowed reports nothing until a call
    may have changed it, which is PHPStan's rule for any expression whose type its scope
    holds (`ImpossibleCheckTypeHelper::isSpecified`), and not about arithmetic alone.
  - *`abs()`, `pow()` and `array_sum()`* follow: the absolute value of the argument's type;
    a power as `**`; the sum of the values of an array written out, with `0` for one that may
    be empty.
  - *Where the checker differs.* It computes with no range, so a plain int divided by a
    literal or a range is an ordinary union here and benevolent in PHPStan (stricter). The
    other way it is more lenient, and known: where both operands are plain ints to the
    checker and ranges to PHPStan, as the results of `time()` and `mb_strlen()` are, the
    division is benevolent here and ordinary there; the way out is the ranges in the overlay.
    A power with an exponent certainly below zero, and an int to a float exponent, stay the
    float PHP computes. And an object passed to a function keeps what the flow knew of its
    properties, where PHPStan forgets it, so a type test after such a call is not reported
    (§10).
- **A function's body is held to its `@param-out` types** (2026-11-26; `P2213`). A by-reference
  parameter documented `@param-out T` is held to `T` inside its own function or method, as
  PHPStan holds it (`paramOut.type`); until then the checker read the tag at a call alone, as
  the type the argument holds afterwards.
  - *Every write must leave a `T`*, and is reported where it is made: an assignment, compound
    or by reference, a write to an offset and an `unset()` of one, a destructuring, a
    `foreach`, a step, and an argument passed by reference to a callee that writes it. A sort
    of PHP's own that keeps the keys writes the array back as PHPStan types it, a list
    becoming an array of `int<0, max>` keys. A write into a value of a type parameter bounded
    by an array leaves the array written (`generics.md` §3.2), so an append to a `T of
    list<int>` documented `@param-out T` is reported: the tag to write is the bound.
  - *Where the body ends*, a parameter that may still hold the value it was passed must hold a
    `T`. The places are PHPStan's: the last statement, or, when that is an `if`, the last
    statement of each branch, and the `if` itself when it has no `else`. A statement after
    which the body cannot go on, a `switch`, a `try` or a loop whose every way returns, is
    held at the returns inside it, and what a `finally` block writes counts at each of them.
    A parameter written on every way to the end is not checked there again: one mistake, one
    finding.
  - *Not held*, because PHPStan does not hold them: a `return` before the last statement
    anywhere else, and the bodies of closures and arrow functions.
  - *Handed over unchecked*: a parameter a reference is taken to, or that is `static` or
    `global`, and one passed to `settype()` or, where it may hold an object or a resource, to
    a function of PHP's own with side effects that reads it, which PHPStan forgets as well.
    Any other function of PHP's own writes what its `@param-out` says, held to the tag at the
    call, or reads the parameter and writes nothing ("What PHP's own functions write",
    below).
  - *Inherited* (2026-11-26). A by-reference parameter whose own docblock has no `@param-out`
    takes the tag of the method it overrides, as PHPStan merges one docblock into another:
    from one declaration, the one PHPStan takes the docblock from (the first that has a
    docblock, its own or an inherited one, of the method in the parent's table, then of the
    class's immediate interfaces, then an abstract trait method), so that a parent whose
    docblock lacks the tag passes on none though an interface beside it has one; tag by tag,
    a docblock of the override's own keeping the tag of a parameter it does not document with
    `@param-out`; by position, a renamed parameter taking the tag and the subject of a
    conditional type renamed with it (a constructor's by name); the parent's type parameters
    mapped through `@extends` and `@implements`. The override's body is held to the inherited
    tag, the finding naming the declaration whose docblock has it, and a call through the
    override reads the tag's type. How `@param` is inherited does not change. A trait's body
    is held to the trait's own tags alone (§10).
  The corpus file is `param-out-body.phn`, each form held to PHPStan in both directions; the
  parity run names the lines where PHPStan reports the value passed in for a statement every
  way out of which writes the parameter. None of the dogfood trees documents `@param-out`.
- **What PHP's own functions write** (2026-11-26). A by-reference argument of a function of
  PHP's own holds, once the call returns, what PHPStan 2.3.4 leaves in it, for a caller
  anywhere and not only in a body held to a `@param-out`. Until then it kept the type it had,
  but for the twenty parameters the overlay had tagged: after `parse_str($s, $out)` a caller
  read whatever `$out` held before. PHPStan's signature map has two kinds of by-reference
  parameter, and the checker follows it for each of the 213 in the stubs.
  - *One that creates its variable* carries a `@param-out` in the overlay
    (`stdlib/overlay/writes.php`): the type of PHPStan's stub or of its extension, and
    `mixed` where it has neither, whatever PHP is known to write there. `mixed` is the
    oracle's answer, and a narrower tag would pass code PHPStan reports; the overlay takes a
    better type when PHPStan does.
  - *One the function reads* is no write: the argument keeps its type. Where PHPStan writes
    it back all the same, so does the checker. `sort()`, `rsort()`, `usort()` and `shuffle()`
    leave a list, non-empty where the array was. The sorts that keep the keys leave them, a
    list's as `int<0, max>`. `array_pop()` and `array_shift()` keep the keys, and take the
    last or the first entry from a shape. `array_splice()` leaves what PHPStan's
    `spliceArray()` makes of the array, but for an offset or a length past the end, which it
    reads as PHP does. `array_walk()` with a closure that takes the value by reference
    leaves the values the closure leaves where it returns, ends or stops, as PHPStan gathers
    them: a throw inside a `try` is no such place, and each is taken after the `finally`
    blocks it leaves through.
  - *What `preg_match()` writes* is a list where the pattern is known at the call and names
    no group, under flags known there: PHP numbers the groups without a gap. A named group,
    a backtracking verb that carries a name, which adds the key `'MARK'`, and a pattern the
    checker cannot read as PHP will (an escape PHP decodes in a double-quoted string) keep
    the keys `int|string`. PHPStan reads the pattern into a shape, group by group; the
    checker does not, so under `PREG_UNMATCHED_AS_NULL` every value may be null where
    PHPStan knows which groups are optional: stricter there, never more lenient.
  The corpus file is `param-out-php-writers.phn`. One finding came to the dogfood trees with
  it, a `(string)` cast of what `openssl_pkey_export()` wrote, which is `mixed` to PHPStan
  too.

## 10. Open questions

- **An object passed to a call keeps what was known of its properties** (2026-11-26).
  After `$o->s = 'x'; mutate($o);` the checker still has `$o->s` as `'x'`, and
  `$o->s === 'y'` is `P2305`, "always false", though `mutate()` may have written it; PHPStan
  forgets the properties of an argument, as it forgets `$this`'s after a method call, which
  the checker does too. A false finding in ordinary code, and since a type test of a written
  property reports nothing (§9, "A division and a power …"), also a leniency: `is_string($o->s)`
  after such a call is reported by PHPStan and not here.

- **Which findings are extra checks** (roadmap D33). The inventory was what the parity run
  found as the lines the checker flags and PHPStan does not; each is now a language rule by
  its code, marked as an extra check, or named in the run's table with its reason (§8). What
  stays open are the true findings above PHPStan that cannot be marked where they are
  reported, because nothing there proves that PHPStan has nothing to say (§9, *What proves
  one*): the difference lies in a type inferred statements earlier, and the finding is drawn
  by the ordinary check of an argument, a return or a receiver, which cannot see where the
  type came from; or the value is one of which PHPStan may know more than the checker. They
  are reported in both modes. The families known:
  - an operator on a key read from an array whose key type was not written (`$key + 1` under
    `foreach ($arr as $key => $v)` with `array<mixed>`), which PHPStan passes as a benevolent
    union and the lattice reads as a plain `int|string`;
  - a function map's benevolent union kept in a variable, where PHPStan keeps it benevolent;
  - a string cast of a key that is an int key all the same, which PHPStan reads as a string
    key;
  - arithmetic, `**` on a string and the two rules of array shapes on a value that is no
    untouched parameter and no literal (§9, *What proves one*): a local, a range the docblock
    declares, a parameter its function names more than once, a closure's parameter, a method's
    parameter where another declaration of its hierarchy exists. PHPStan passes most of them,
    and the checker cannot tell which: the lattice writes a union of consecutive values as a
    range and widens a long one to int, and PHPStan computes with both;
  - a longer tuple assigned to an offset, which PHPStan's own rule for that place passes
    while it refuses the general array;
  - the step of a string that PHPStan has as a benevolent union: a key of
    `array<array-key, V>`, and a numeric string once it is stepped;
  - a narrowing PHPStan makes by a function's short name, where a namespace has a function of
    its own under it;
  - what a static call forgets of `$this`;
  - the rows after `unset($rows[$k]['name'])`;
  - a Traversable class read as the iterable its `@implements` says, where PHPStan reads its
    `getIterator()`;
  - a value of `T of array<…>` held to `T` after a write PHPStan follows without losing `T` (a
    key written in one straight line, an optional entry `unset()` takes from a shape bound;
    returned, yielded, passed or assigned): the write and the place name the variable twice, and
    nothing at the place tells the line from those PHPStan reports (`generics.md` §3.2).

  To tell them, a type would have to carry where it came from. The run's table names those
  the corpus has, beside what the emitter rewrites of the source (§9) and the false findings
  that wait for their fix.

- **`@param-out`** (2026-11-26). The body is held to the tag (§9). The checker is still more
  lenient than PHPStan in these places. Of what PHP's own functions write (§9), `settype()`
  leaves the argument its type where PHPStan has the converted one, and `array_splice()`,
  `array_shift()` and `array_push()` on a shape with optional or unsealed entries leave a
  general array where PHPStan keeps the entries. A trait's body is not held
  to a tag that a class using the trait inherits (§9, *Inherited*): PHPStan checks the body
  in the context of each such class and reports a write that breaks that class's tag, while
  the checker checks a trait's body once, as the trait, where `$this`, `static` and the
  users' properties are the trait's, and held to a user's tag that one reading reported code
  PHPStan passes (`$out = clone $this` under a user's `@param-out Copyable`). A tag that
  names a template of the parent's method reads `mixed` at a call through the override,
  where PHPStan reads the argument's type. And a by-reference parameter without a tag is not
  held to its declared type (`parameterByRef.type`).

- **A line that throws whenever it runs: decided, and not built** (2026-11-27). §9 takes a
  finding about such a line out of a comment's reach, as an extra check. Until it is built a
  comment silences `P2016`, `P2010`, `P2112` and `P2114` like any finding. The build marks
  the members §9 lists, adds the test that runs each mark of the corpus under php, the third
  wording of `P4015`, and the switch's reading. It comes after `array_multisort()` and the
  other functions whose parameters are by reference only where they can be are read as PHP
  reads them, so that no mark stands on a false `P2114` or `P2112`. Open beside it: which
  other findings are members (too few arguments to a known function; `new` of an abstract
  class, an interface or an enum), each to be weighed for a valid program that writes it.
- **A narrowed type parameter: decided, and not built** (2026-11-20; rewritten 2026-11-26).
  §9, "A narrowed type parameter is the parameter still", says which test keeps the parameter
  and in which of three forms (`T`, `T of int`, `T&Row`), so what was asked here is answered.
  The checker does not follow the rule yet. In 0.8.0 a value of `T` that a test narrowed is
  still an intersection (`T&int`, `T&Row`, `T&null`), which is no `T` where one is expected
  and takes nothing away from a bound, and the findings §9 names as false are reported:
  `P2202` at a `return $a` and `P2103` at an argument after `is_int($a)`, `$a instanceof Row`
  or `$a === null`; `P2302` at an append after `!is_int($a)` under `T of list<int>|int`;
  `P2105` at a member after `$a !== null` under `T of ?Row`; `P2402`, "always true", at
  `if ($a)` under any `T`. PHPStan reports none of them. What stays open once the rule is
  built is listed below ("What PHPStan knows of a narrowed type parameter and the checker
  does not").

- **Shapes vs classes**: PHPStan's array shapes are rich (optional keys, unsealed `...`). M2
  supports sealed shapes with optional keys; unsealed shapes are read and treated as `array<K, V>`.
  M4 decides whether shapes survive as a language feature. *Decided 2026-11-05 (collections.md
  §6): they stay docblock vocabulary, with no native syntax. Amended 2026-11-15 (roadmap D29): a
  shape with a name is syntax, in the right-hand side of a `type` declaration. The lattice has
  moved on from the M2 sentence above: an unsealed shape keeps its flag (`types.Shape.Unsealed`),
  so a key it does not name reads as `mixed` where a sealed shape gives `P2313`. Since 2026-11-16
  that read is `P2321` unless a test stands guard (§9).*
- **The typed rest of an unsealed shape** (2026-11-15). PHPStan writes `array{id: int,
  ...<string, int>}` for further keys of a known type. The docblock parser reads the bare `...`
  only: the typed form is `P4001`, and its tag is dropped. Named shapes take the grammar the
  docblock has (`collections.md` §6.1), so neither spelling has it. Reading it means a key and a
  value type on the lattice's shape, in the docblock and in the syntax alike. The rule that then
  applies is PHPStan 2.3's, probed on 2026-11-15: an expected shape with a typed rest holds to it
  each key the given shape has beyond the named ones, and the given shape's own rest, so
  `array{id: int, ...}` fails `array{id: int, ...<string, mixed>}`, its rest's keys being
  `int|string`; PHPStan 2.2 lets that one pass.
- **What a test proves of an offset** (2026-11-15). After `isset($a['id'])` and
  `is_int($a['id'])`, PHPStan's type of `$a` is `array<string, mixed>&hasOffsetValue('id',
  int)`, which passes for `array{id: int}`. The checker narrows the offset (`$a['id']` is an
  `int`) and not the array, so the call is reported: stricter than PHPStan, which §9 allowed
  when this was written. By roadmap D33 it is no extra check: the call cannot fail, so the
  finding is a false one, reported in both modes until it is fixed.
  None of the dogfood trees holds such a call. A reading built on 2026-11-15, which wrote what the
  environment knew of the offsets into the array's type where the array is held to a declared
  one, was set aside: it gained nothing on the dogfood, and it accepted what PHPStan reports
  after a write through a reference to the array (`$alias = &$a; $alias['id'] = 'x';`), a write
  the checker's narrowing of offsets does not see either. Doing it needs the typed rest above,
  for the keys the test did not prove, and that narrowing made sound first. *For a shape this
  is done since 2026-11-16 (§9, An offset a shape might lack): there the narrowed type is the
  shape with the entry present, which needs no typed rest. The general array is as described
  here. The write through a reference between two variables is seen since 2026-11-17 (§9);
  one through a reference to an offset, or through a closure's `use (&$a)`, is not, by PHPStan
  either.*
- **What an included file does to the scope** (2026-11-17, roadmap D32). `include` and `require`
  inside a function run the file in the function's scope: it reads and may write the local
  variables. The checker types the function as if the statement touched none of them, so what
  the file assigns or changes is not seen. A file that only returns a value, or declares
  classes, is the common use and has no such effect.
- **Keys a sealed shape does not name** (2026-11-15). A literal with a key more than the shape
  names is accepted where the shape is expected (§9), while reading that key from the shape is
  `P2313`. A shape somebody named is a contract in a way an ad-hoc docblock is not, and
  TypeScript refuses the excess key of a literal for that reason. Whether a `type` declaration's
  shape should refuse it too, stricter than PHPStan as the parity rule allows, is open; it would
  give the syntax and the docblock different rules for one type, which §6.1 of `collections.md`
  avoids.
- **Constant evaluation**: `const A = B::C + 1` — the checker evaluates literal arithmetic over
  int/float/string/bool/null and arrays; anything else types as the declared type. Enough for
  enum backing values and most constant tables.
- **`@internal`, `@deprecated`**: parsed and stored; reporting deprecated calls is a warning behind
  `--deprecations` (default on in `check`, off in `build`).
- **Speed**: parsing all of `vendor/` for a Symfony project is ~5–10k files; the loaders are lazy
  and per-file, so a check touches hundreds, not thousands. A persistent cache is M6.
- **Element checks** (2026-11-09). A hand-written check of a list's elements
  (`array_all($xs, fn ($x) => is_string($x))`, a `foreach` that throws) narrows nothing beyond
  `list<mixed>`, as in PHPStan 2.1 (`generics.md` §11.4), so its result is stated with a `@var` or
  a guard function's `@phpstan-assert`. If PHPStan learns to narrow through `array_all()`, the
  checker follows.
- **An override under `nothrow`** (2026-11-15, `functions.md` §4.5). Nothing holds an override
  that writes its own `throws` clause to the method it overrides (`tooling.md` §9.1: covariance
  is deliberately not checked, as in PHPStan's defaults). Under a method that declares `nothrow`
  this leaves a hole a reader will not expect: a `catch` around a call through the parent's type
  is reported dead (`P2403`) while a subclass declares and throws the exception. Should an
  override of a `nothrow` method be refused a `throws` clause, and a body that throws? It would
  be a rule of `nothrow`'s own, stricter than PHPStan as the parity rule allows, and for the
  docblock form it would hold PHP code to a contract PHPStan does not enforce. Open; the
  dogfood trees hold no `@throws void` to measure it on.
- **Where an inherited throws declaration is looked for** (2026-11-15, `functions.md` §4.5). A
  method that declares nothing takes the declaration of a method it overrides, and the checker
  looks for it in the order PHPStan inherits a docblock in: the method the parent class's table
  holds, with what that method inherits in turn, then the class's own interfaces in the order
  written, each asked for the method it has, inherited ones included. A constructor reads the
  constructor in its parent's table first, concrete or not, and a private one passes on nothing.
  One shape is left, where the body of a method that declares nothing is held to nothing here
  while PHPStan 2.3.4 reports `throws.void` in it, which the parity rule forbids:
  - *A trait's method.* A trait's body is checked once, in the trait, where no class stands over
    it, while PHPStan analyses it in the context of each class that uses the trait. A trait's
    method that implements an interface method saying `nothrow` is therefore held to nothing,
    though a call to it through the class reads the interface's `nothrow`, so a `catch` around
    the call is still `P2403`, as PHPStan finds it. Nor is a class's method held to an abstract
    trait method that says `nothrow` and that the method implements.

  Holding a trait's body in the context of each class that uses the trait, as PHPStan does,
  closes it.
- **What the override check does not reach** (2026-11-15, §9). Each is more lenient than PHPStan
  or PHP, and each is left for a change of its own:
  - *Object shapes and intersections* are not decided by the test of two types being apart:
    `object{a: string}` for an `object{a: int}`, or `Box<string>&Tag` for a `Box<int>&Tag`,
    passes, where PHPStan 2.3.3 reports `method.childReturnType`.
  - *`int|string` and `array-key`* are one union on the lattice, so an invariant argument written
    `int|string` passes as PHPStan's benevolent `array-key` would: `Box<int|string>` for a
    `Box<int>` passes, which PHPStan reports. Telling them apart needs the type model to mark the
    benevolent union, as PHPStan does.
  - *What a trait brings in* is compared in part. A trait's abstract constructor, used by an
    abstract class, binds the classes below it as any abstract constructor does. Not compared, not
    even by PHP's rule on the erased types: a method a trait brings in against the parent's
    method it overrides, a class's own method against the abstract one its trait declares, and a
    method a trait brings in against an interface of the class; nor *a parent's method against an
    interface the child adds* (`class C extends P implements I {}`, where `P::foo()` does not fit
    `I::foo()`). PHP refuses to link each such class.
  - *Subtyping outside overrides does not map inherited type arguments*: it steps up from a class
    to a generic ancestor without the arguments the class passes it, so `ArrayIterator<int, Cat>`
    still passes where `Iterator<int, Dog>` is declared, for a return or an argument, where
    PHPStan reports `return.type`. The override check asks the test of two types being apart for
    that reason; the rest of the checker does not yet.
  - *Inference from bounds is one pass.* A class template the constructor's arguments leave
    unbound takes what the bound of a bound template says of it, as PHPStan infers it; a template
    named only in the bound of a template so inferred stays at its bound. A template's default
    that names an earlier template (`@template U = T`) is taken as written, `T` not replaced by
    the argument it took.
  - *The SPL overlay types what binds the templates, not every member.* The generic SPL, DOM and
    date classes have PHPStan's templates and the members that carry them in (the wrapping
    iterators' constructors, `WeakReference::create()`, the DOM lists' accessors, the read
    methods PHPStan types); the rest keep PHP's types, where PHPStan's stubs say more:
    `SplObjectStorage::attach()` and `getInfo()`, the lists' `push()` and `pop()`, `DatePeriod`'s
    getters. `SplStack` and `EmptyIterator` take no arguments. The overlay patches methods only,
    so `DOMNode::$childNodes` and `$attributes` are PHP's bare `DOMNodeList` and
    `DOMNamedNodeMap`, which yield `DOMNameSpaceNode|DOMNode` and `DOMNode` where PHPStan reads
    `DOMNode` and `DOMAttr`.
  - *An override of PHP's own methods that return null past the end.* `current()` and `key()` of
    an iterator are `TValue` and `TKey`, and the `offsetGet()` the overlay declares without null
    is `TValue`, as PHPStan reads them, though PHP returns null past the end or for an offset not
    held. An override that adds `null` is reported under a closed argument by the rule on the
    erased types (§9), where PHPStan passes it. Whether `T|null` for a closed `T`, under a native
    `mixed` on both sides, should pass is open.
- **What PHPStan knows of a narrowed type parameter and the checker does not** (2026-11-20, §9,
  "A narrowed type parameter is the parameter still"). Each stands as it stood before that
  rule, and none is made worse by it:
  - *A test of an entry's value under a general array bound.* `is_int($a['k'] ?? null)` or
    `($a['k'] ?? null) === 1` under `T of array<string, mixed>` leaves PHPStan with the
    array, the parameter lost, and `return $a` is `return.type` there. The checker narrows
    the offset and not a general array (above, "What a test proves of an offset"), so the
    value stays `T` and the return passes: more lenient than PHPStan.
  - *A refinement a function makes.* `str_contains()`, `ctype_digit()` and `strlen($a) > 0`
    on a `T of string`, `method_exists()` and `property_exists()` on a `T of object`, are
    meets in PHPStan (`T of string&numeric-string`), no `T` in `Box<T>`. The checker narrows
    nothing by them and has the whole `T` in an invariant position: more lenient there, and
    nowhere else.
  - *The false branch of `@phpstan-assert-if-true`.* PHPStan 2.3.4 takes the asserted type
    away where the call answers false; the checker narrows that branch only where a tag
    names it (`-if-false`). After `if (isInt($a)) { return $a; }` under `T of list<int>|int`
    an append is still `P2302`: a false finding, and one a plain `list<int>|int` draws as
    well, as `P2202` where the list is returned.
  - *A subtracted type.* The lattice has no `mixed` without `int`, no `Animal` without
    `Dog`, no `Suit` without a case under a parameter's bound, so what such a test takes
    away leaves `T` whole. Nothing is refused for it.
  - *The bound a meet started from.* A meet keeps what the value is known to be, not the
    bound it had: enough for every rule of §9 but PHPStan's generalization of a literal in
    an inferred type argument (`5&T of int` to `T of int`), where the checker stays stricter.
