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

1. **Discover** project files: the paths given (`.phn`, walked) — the *checked set*.
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

### 4.2 Loaders

A `Loader` maps a fully-qualified name to a parsed declaration:

1. **Project**: the checked set, parsed in Phaenon mode.
2. **Composer**: `vendor/composer/autoload_psr4.php`, `autoload_classmap.php`, `autoload_files.php`
   (function and constant files) and `autoload_namespaces.php` (PSR-0), evaluated by walking their
   AST (string literals, `$vendorDir . '/…'`, `$baseDir`). The project's own PSR-4 roots are
   there too, which is how a `.php` file beside a `.phn` one is found. Looked-up files are parsed in
   PHP mode; a file is parsed once however many classes it declares. Composer's maps are found by
   walking up from the first checked path to the nearest `vendor/composer/`.
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
(always false), `instanceof` always true/false, dead `?->` (left never null), `??` with a
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
`code`, `message`, `fix` when present). Messages name types in PHPStan vocabulary and members as
`Class::member()`; they never mention internal node names.

## 7. Removed constructs (P1)

Per `language-overview.md` §3, in Phaenon mode the checker rejects: `global`, `goto`/labels,
`eval`, `extract`/`compact`, `$$var`, `${expr}` in strings, alternative control syntax, `var`,
`(integer)`/`(boolean)`/`(double)`/`(real)` casts, `list()`, `include`/`require` *inside functions*
(top-level `require` of a bootstrap file stays), `<?php`/`?>`/inline HTML (the parser already
refuses these). Each P1 carries a fix-it. The **parser** keeps accepting them so the php-src
conformance corpus still parses; the checker is where "Phaenon mode" bites.

## 8. Testing

1. **Golden corpus** `internal/checker/testdata/*.phn` — one file per feature; expected
   diagnostics as trailing comments on the offending line: `// want P2104` (optionally with a
   message substring). A test runs every file and demands an exact match of (line, code) sets.
   Vendor fixtures live under `testdata/vendor/` with a hand-written `autoload_psr4.php`.
2. **Type printing / subtyping tables** in `internal/types` — unit tests over pairs.
3. **Docblock grammar** — a corpus of PHPStan type strings → canonical printing, plus error cases.
4. **Stub loading** — every embedded stub parses; a few hundred well-known signatures are asserted
   (`strlen`, `array_map`, `ReflectionClass::newInstance`).
5. **PHPStan parity** (opt-in, `PHAENON_PHPSTAN=/path/to/phpstan`): for each golden file, write it as `.php`,
   run `phpstan analyse --level max --error-format json`, and assert that PHPStan's (line) set
   is covered by ours. Not the other way round: the checker may be stricter. Where PHPStan finds
   something we do not, the test fails and the case is either fixed or allowlisted with a reason.
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
  project keeps its existing suppressions; `@phaenon-ignore` is the native spelling.
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

## 10. Open questions

- **Shapes vs classes**: PHPStan's array shapes are rich (optional keys, unsealed `...`). M2
  supports sealed shapes with optional keys; unsealed shapes are read and treated as `array<K, V>`.
  M4 decides whether shapes survive as a language feature. *Decided 2026-11-05 (collections.md
  §6): they stay docblock vocabulary, with no native syntax. Amended 2026-11-15 (roadmap D29): a
  shape with a name is syntax, in the right-hand side of a `type` declaration. The lattice has
  moved on from the M2 sentence above: an unsealed shape keeps its flag (`types.Shape.Unsealed`),
  so a key it does not name reads as `mixed` where a sealed shape gives `P2313`.*
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
  `int`) and not the array, so the call is reported: stricter than PHPStan, which §9 allows.
  None of the dogfood trees holds such a call. A reading built on 2026-11-15, which wrote what the
  environment knew of the offsets into the array's type where the array is held to a declared
  one, was set aside: it gained nothing on the dogfood, and it accepted what PHPStan reports
  after a write through a reference to the array (`$alias = &$a; $alias['id'] = 'x';`), a write
  the checker's narrowing of offsets does not see either. Doing it needs the typed rest above,
  for the keys the test did not prove, and that narrowing made sound first.
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
