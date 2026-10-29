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
  key and value fits; optional keys are `?` and a missing key is not `null`.
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
| `is_a($x, Foo::class[, true])`, `is_subclass_of` | `$x & Foo` (class-string when 3rd arg) | — |
| `$x === null`, `null === $x`, `$x !== null`, `isset($x)`, `empty($x)` | as expected | |
| `$x === 'lit'` / `$x === Foo::Bar` / `$x === true` | literal / enum case | subtract literal |
| `$x`, `!$x` (truthiness) | remove `null`, `false`, `0`, `''`, `'0'`, empty array | keep falsy |
| `$a && $b`, `$a \|\| $b`, `!$p`, `($p)` | combined | |
| `count($x) > 0` / `=== 0`, `$x !== []`, `array_key_exists('k', $x)`, `in_array($x, [...], true)` | non-empty / shape key / literal union | |
| `assert($p)`, `if (!$p) throw/return/continue/break/exit` | narrowing continues after the statement | |
| `match (true) { $p => … }` arms, `switch` with `===` semantics on literal subjects | per arm | |

Narrowing applies to **paths**: `$x`, `$x->prop`, `$x['k']`, `$this->prop`, `Foo::$prop`. A
property path is invalidated by any call on the same object, any assignment to a prefix of the
path, and by `unset`. A variable path is invalidated by assignment and by reference capture
(`&$x`, `use (&$x)`, passing to a by-ref parameter, `global`).

### 3.5 Flow analysis

Abstract interpretation over the AST with an environment `var → type` plus "definitely assigned"
bits. Rules:

- Branches merge by union of each variable's type; a variable assigned on only some branches is
  *maybe undefined* after the merge, and reading it is P2 (PHPStan: "might not be defined").
- `return`, `throw`, `exit`, `continue`, `break`, calls to `never` functions end a path.
- Loops: iterate the body with the environment joined from the entry and the back edge until a
  fixed point; after 3 iterations the changing variables are widened to their declared type, or
  to the union's primitive. `foreach` keys/values come from the iterable's key/value types; a
  `foreach` over a possibly-empty iterable may not assign its variables.
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

### 4.4 Hierarchy

Built lazily per class, with cycle detection (P2 "circular inheritance"): the method table is the
class's own methods, then trait methods after adaptations (`insteadof`, `as`), then the parent's
table, then interface methods (abstract). Property and constant tables likewise. Signature
compatibility with the overridden member (parameter contravariance, return covariance, visibility,
static-ness, readonly) is checked for **checked** classes only — PHP fatals at runtime on the
same conditions, so the checker reports them as errors.

## 5. Checks

Every check has a stable code. Families: `P1` syntax and removed constructs, `P2` types and flow,
`P3` erasure/generics (M3), `P4` interop (vendor, docblocks). Numbers are allocated in the
implementation and listed in `checker-codes.md` (generated from the source, one line per code
with its message template). The M2 set, by area:

**Names** — undefined class / interface / trait / enum, function, constant, class constant,
property, static property, method, static method; `parent` without a parent; `self`/`static`
outside a class; unknown named argument; duplicate `use`; unused `use` (warning).

**Calls** — too few / too many arguments; argument type not acceptable; named argument after
unpacking; variadic/by-ref mismatches; call on `mixed`/`null`/non-object; calling a non-static
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
catch block ordering (a wider exception first makes a later catch dead, warning).

**Classes** — missing abstract method implementation in a concrete class; interface method
signature mismatch; override incompatible (LSP); `#[\Override]` on a method that overrides
nothing; missing `#[\Override]` is **not** reported (Tequila's cs-fixer owns that); abstract
method in a non-abstract class; final method overridden; trait conflict without `insteadof`; enum
with properties; enum case value type mismatch; interface constants/properties rules;
constructor promotion in an abstract constructor; readonly class with static/untyped properties;
property hooks: `get` returning the wrong type, `set` parameter type, virtual property without
backing used by reference.

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
  Phaenon deliberately lacks is Tequila's: PHPStan 2.x, level 8 in the 22 service repositories
  (`core`, `iam`, `mongodb-odm` among them), level 10 in the 17 libraries,
  `treatPhpDocTypesAsCertain: false` everywhere, no `phpstan-strict-rules`. Refinement types
  (`non-empty-string`, `positive-int`, `int<0, max>`, …) are inferred and narrowed —
  `$s !== ''` gives `non-empty-string` — but never required in source, since they cannot be
  written as native types (`generics.md` §11.2): where a declared `string` flows into a vendor
  `non-empty-string` parameter the finding stays, as in PHPStan, and the fix is a guard, not an
  annotation. Development must stay plausible for a person without an assistant; a rule that
  turns typing into a minefield is not adopted because PHPStan has it.
- **Docblock types are not certain** (2026-10-29, `tequila/mongodb-odm`). A docblock, an
  inherited generic or a `@var` tag refines what PHP declares; the refined type drives inference
  and narrowing, the PHP type is what the runtime guarantees. A diagnostic that calls a check
  impossible — an `instanceof` always true, a `=== null` always false, an `is` that can never
  hold — requires both to agree. `normalize(object $value)` in a subclass of a generic normaliser
  is typed `UnitEnum`, and its `if (!$value instanceof UnitEnum) throw` is not dead code, since
  erasure lets any object in. This is PHPStan's `treatPhpDocTypesAsCertain: false`, Tequila's
  setting everywhere; the parity harness runs PHPStan with it. The type flow still uses the
  refined type: a ternary whose condition the refined types decide takes one branch, as in
  PHPStan.
- **`@var` narrows, never widens** (2026-10-29). Above an assignment, the tag's type must be a
  subtype of what PHP declares for the value (PHPStan's `varTag.nativeType`); a literal value must
  fit the tag instead. Above any other statement, `@var T $x` retypes `$x` for what follows; above
  a `return`, an unnamed `@var T` types the returned value. A `@var` on a promoted constructor
  parameter types the parameter and the property (`phn migrate` lifts it into the declaration).
- **Extensions come from `php`** (2026-10-29). The embedded stubs are php-src's; the classes,
  functions and constants of every other extension the local `php` has loaded — ext-mongodb,
  redis, amqp, apcu — are generated from its reflection, once per extension and PHP version,
  and cached under the user's cache directory (`program.LoadExtensions`; `PHN_PHP` names another
  binary). `php` is the oracle for what exists. Reflection carries no docblocks, so a generated
  stub has no generics; the overlay refines what matters. Without a `php` on `PATH` the checker
  warns once (`P4007`) and knows php-src only.
- **A trait sees its users.** A method the trait body calls on `$this`, `self` or `static` and
  the trait does not declare is looked up in the classes using the trait, as PHPStan does by
  analysing a trait in each user's context.
- **A general array is accepted for a shape it may well be.** `array<string, string>` passes for
  `array{spec: string, declared: string}` when its keys and values can be the shape's, as PHPStan
  accepts it; `array<string, mixed>` still fails against `array<string, int>`.
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

## 10. Open questions

- **Shapes vs classes**: PHPStan's array shapes are rich (optional keys, unsealed `...`). M2
  supports sealed shapes with optional keys; unsealed shapes are read and treated as `array<K, V>`.
  M4 decides whether shapes survive as a language feature.
- **Constant evaluation**: `const A = B::C + 1` — the checker evaluates literal arithmetic over
  int/float/string/bool/null and arrays; anything else types as the declared type. Enough for
  enum backing values and most constant tables.
- **`@internal`, `@deprecated`**: parsed and stored; reporting deprecated calls is a warning behind
  `--deprecations` (default on in `check`, off in `build`).
- **Speed**: parsing all of `vendor/` for a Symfony project is ~5–10k files; the loaders are lazy
  and per-file, so a check touches hundreds, not thousands. A persistent cache is M6.
