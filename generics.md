# Phaenon — Generics design (draft 0.1)

Status: **draft for discussion** · 2026-10-28 · supersedes nothing

Phaenon is a typed PHP dialect compiled to PHP ("TypeScript for PHP"). This document designs
generics for it: syntax, type-system semantics, how they compile to PHP, and — the hard part —
how much of them survives at runtime. The guiding constraint is the framing itself: **whatever we
emit must be plain PHP that Symfony, PHPStan, PhpStorm and a PHP caller understand.**

---

## 1. Goals and non-goals

### Goals

1. Generics are **real syntax**, checked by the Phaenon type checker — never docblock comments
   in Phaenon source.
2. Emitted PHP is **readable, idiomatic and typed the way the ecosystem types today**
   (`@template` docblocks), so the output is a first-class citizen of a Composer project.
3. A **meaningful runtime story**: `$x instanceof T`, `new T(...)`, `$x is list<int>` — where the
   programmer asked for it, at a cost the programmer can see. *Narrowed by §11.2, §11.4 and
   §11.5: the runtime story is PHP's own, a `class<T>` value the code holds and PHP's tests.*
4. **Zero cost when not asked for.** Code that never reifies a type parameter compiles to exactly
   the PHP a careful human would have written.
5. **Interop both ways** without ceremony: Phaenon calls PHP generics (docblock-typed vendor code)
   with full checking; PHP calls Phaenon generics without knowing they are generic.

### Non-goals (v1)

- Higher-kinded types, conditional types, template-literal types, type-level arithmetic.
- Overloading on type arguments.
- Specialisation / monomorphisation as the primary strategy (see §7 for why it's a fallback idea).
- Generics on `trait`s and `enum`s (deferred; see §9).

---

## 2. Syntax

### 2.1 Declaration site — `<T>` after the name

```php
final class Collection<T> implements IteratorAggregate<int, T> { ... }

interface Mapper<in TIn, out TOut> { ... }               // variance: in / out

final class Repository<T of AggregateRoot> { ... }       // upper bound: `of`

final class Cache<K of string|int = string, V = mixed>   // defaults

function first<T>(list<T> $items): ?T { ... }

public function map<U>(Closure(T): U $fn): Collection<U> { ... }

$id = fn<T>(T $x): T => $x;                              // generic closure: not available yet (P1007, issue #28)

type UserMap = map<Uuid, User>;                          // type alias
type Result<T, E of Throwable = RuntimeException> = Ok<T> | Err<E>;
```

Vocabulary choices and why:

| Choice | Alternatives | Why this one |
|---|---|---|
| `T of Bound` | `T extends Bound` (TS/Java), `T as Bound` (Hack), `T: Bound` (Rust) | It is literally what PHPStan docblocks write (`@template T of Foo`) — every PHP developer already reads it, and the emitted docblock is the same text. |
| `in T` / `out T` | `+T`/`-T` (Scala), `@template-covariant` | Declaration-site variance like C#/Kotlin/Hack; `out` = produces T (covariant), `in` = consumes T (contravariant). Maps 1:1 onto `@template-covariant` / `@template-contravariant`. |
| `type X = …` | `alias`, `typedef` | TS/Hack/Rust convention; `type` is not a reserved word in PHP, so it's a contextual keyword at statement start (same trick as `enum` in PHP 8.1). |

*2026-11-15 (§11.6, roadmap D29):* an alias belongs to its namespace, and another file imports it
with `use`; its right-hand side is the one place a shape is written, `type Row = array{id: int,
name?: string};` (`collections.md` §6).

*2026-11-15 (roadmap D30):* `type`, `of`, `in` and `out` are written in lower case, as every word
Phaenon adds is. Where the parser takes the word, another case is still read as it (`TYPE Row =
int;` declares the alias, `<T OF Item>` bounds `T`) and is `P1006`, which `phn fmt` rewrites; a
name of that spelling (`Type::make()`, a type parameter `In`) stays a name. `class<T>` and
`list<T>` likewise: PHP reserves both words, so `CLASS<T>` and `List<int>` in a type position can
only be Phaenon's, written in another case.

### 2.2 Type positions

Anywhere PHP accepts a type, Phaenon accepts a generic type: parameters, returns, properties,
property hooks, `static`/`self`, union and intersection members, nullable:

```php
private readonly map<string, list<Event<T>>> $streams;
public function get(Uuid $id): ?T;
public function all(): iterable<T>;
public function with(Closure(T, int): bool $pred): static;
public function __construct(private readonly class<T> $aggregateClass) {}  // the type of T::class (§11.2)
private readonly Closure(string, string...): (int, string) $kubectl;      // a variadic parameter, a tuple result
private readonly Closure(string, int=): void $log;                       // an optional parameter (§11.3)
```

PHP's grammar never puts an *expression* in a type position, so `<`/`>` here are unambiguous.

A closure type's parameters are types, never names: `T=` marks an optional parameter, `T...` the
variadic one (last, at most one), as PHPStan spells them in `callable(…)` and `Closure(…)`; after
an optional parameter every parameter is optional or variadic. By-reference parameters have no
spelling and stay in a docblock (§11.3).

### 2.3 Call site — explicit type arguments

```php
$repo  = new Repository<User>($store);
$users = Collection::of<User>($rows);
$first = first<User>($users);
$x     = $repo->get<Address>($id);
```

Explicit type arguments are **rare by design**: inference (§3.4) covers the common cases, and
reified parameters (§5) that cannot be inferred are a compile error, not a guess.

### 2.4 The `<` / `>` ambiguity — smaller than in TypeScript

The classic objection to `<>` generics in PHP is `foo<Bar>($x)` reading as `foo < Bar > ($x)`.
In PHP it mostly cannot:

- **Relational operators are non-associative** in PHP's grammar
  (`%nonassoc '<' T_IS_SMALLER_OR_EQUAL '>' T_IS_GREATER_OR_EQUAL`, `Zend/zend_language_parser.y:72`).
  `1 < 2 > 3` is a *parse error* in PHP 8.5. Therefore `name<Type>(…)`, `new Name<Type>(…)`,
  `$o->m<Type>(…)`, `C::m<Type>(…)` and `Name<Type>::m(…)` have **no valid comparison reading at all**.
  Single-type-argument calls are unambiguous.
- **The only genuine ambiguity is a multi-argument type list inside an argument list or array
  literal**: `f(a<B, C>(1))` is valid PHP today meaning `f(a < B, C > (1))` — two boolean arguments
  (verified: `const A=1; const B=2; f(A<B, B>(1))` → `[true, true]`).

Resolution (the C#/TypeScript rule): when the parser sees `Identifier <`, it attempts to parse a
type-argument list; it commits to the generic reading **only if the matching `>` is immediately
followed by `(`**. Otherwise it backtracks and parses a comparison. Consequences:

- `f(A<B, C>(1))` changes meaning in Phaenon. It is a shape no sane codebase contains (two
  constant comparisons, one against a parenthesised expression, in one argument list). The compiler
  emits diagnostic `P1xxx: ambiguous '<' — parenthesise the comparison` when the type-argument reading
  fails to type-check *and* the comparison reading would have parsed, so the migration is loud.
- `>>` is split by the lexer when closing nested type arguments (`map<string, list<int>>`).
- Explicit type arguments are only allowed on a *named* callee (function name, `new Name`,
  `->name`, `::name`). `$fn<T>($x)` is rejected — use inference or a typed intermediate variable.

#### Alternatives considered and rejected

| Syntax | Example | Why not |
|---|---|---|
| Square brackets (Go, Scala, Python) | `Repository[User]`, `first[User]($xs)` | Collides with array access and constant-array indexing (`foo[Bar](…)` is valid PHP: index a constant, call the result). In type positions `list[int]` reads like an array shape. Worse ambiguity than `<>`, and alien to PHP eyes. |
| Turbofish (Rust) | `first::<User>($xs)` | Unambiguous, but `::` already means "static member" in PHP; `Foo::<T>::create()` is unreadable. |
| Keyword-prefixed | `new Repository of User(…)` | Novel, no precedent, awkward in type positions. |
| Parenthesised | `Repository(User)` | Indistinguishable from a call. |
| No call-site type args at all | inference only | TS proves you need an escape hatch (`parse<User>($json)` with nothing to infer from). |

**Decision: `<T>` everywhere, C# disambiguation rule at call sites.** The PHP-specific finding
(non-associative relational operators) makes this strictly safer than the same decision was in TS.

---

## 3. Type-system semantics

### 3.1 Kinds of type parameters

| Declared on | Scope | Erased (default) | `reified` allowed |
|---|---|---|---|
| class | instance members (not `static` members — like Java/C#) | ✅ | ✅ (§5.3) |
| interface | its methods' signatures | ✅ | ❌ (no constructor to carry it) |
| function / method | its body | ✅ | ✅ (§5.2) |
| closure expression | its body | ✅ | ❌ (v1) |
| type alias | the alias | n/a (expands at check time) | n/a |

### 3.2 Bounds

- `T of X` — X may be any Phaenon type (class, interface, union, `object`, scalar).
  Unbounded means `T of mixed`.
- Members accessible on a `T`-typed value are those of the bound.
- Bounds may reference other parameters (`K of array-key, V of Entity<K>`), but not cyclically.

### 3.3 Variance

- Declared with `in`/`out` on class and interface parameters; invariant by default.
- The checker enforces usage positions: `out T` may not appear in parameter positions of public
  members; `in T` may not appear in return positions. Properties force invariance (read + write).
- Functions and methods have no variance annotations (not meaningful).
- Subtyping: `Collection<Admin> <: Collection<User>` iff `T` is `out`. `list<T>` and
  `iterable<T>` are `out`; `map<K, V>` is `out V`, invariant `K` (PHP array keys are compared by
  value, so covariance in K would be sound too — open question §9).

### 3.4 Inference

Local, bidirectional, no global Hindley–Milner:

1. **From arguments** — unify each argument's type with the parameter type (`first($users)` with
   `list<User>` → `T = User`).
2. **From the expected type** (contextual typing, like TS) — `/** @var Collection<User> */`-free:
   `$c: Collection<User> = Collection::empty();` infers `T = User` from the declared variable type,
   return type, or parameter type at the use site.
   A partial application and a first-class callable are the exception: what is expected of
   them is a closure type, which says what their open parameters will receive and joins what
   the bound arguments determine. The callee's return type is not unified with it; the closure
   is checked against it once the type parameters are chosen, and only a literal written in a
   bound argument yields to it (`php-versions.md` §5.2).
3. **From the bound / default** when nothing else constrains T.
4. Otherwise: for erased T, infer `mixed` with warning `P3010: type argument not inferable`
   (built 2026-11-07, §11);
   for reified T, **error** — an explicit argument is required (a runtime value must exist).

Closure arguments are checked *after* the other arguments are unified (TS's "deferred" closures),
so `$users->map(fn($u) => $u->name)` knows `$u: User` without annotation.

*2026-11-15 (`checker.md` §9, "A shape is held as PHPStan holds it"):* a shape parameter is
unified as PHPStan's `ConstantArrayType::inferTemplateTypes()` unifies it, so that
`first<T>(Page<T> $page): ?T` with a `Page<Row>` gives `T = Row`:
- against a shape, entry by entry by key, a tuple by position, an array literal's inferred shape
  included. The keys the argument has beyond the parameter's are passed over. An entry a sealed
  argument lacks binds nothing, so an absent optional entry leaves its parameter to the others;
  an unsealed argument may hold it with any value, and binds `mixed`;
- against a general array, the entries' types, as one union, from the array's value type,
  whatever its keys: `array<int, string>` makes the `T` of `array{value?: T}` a `string`. An
  unsealed parameter's values take in its rest, `mixed`, and bind nothing, as in PHPStan, which
  reports the parameter unresolved;
- a union of arguments binds from each member.

Where the entries' union has members without a type parameter, PHPStan leaves out of `T` what
they already cover (`T|int` against `int|string` gives `T = string`); the checker binds the whole
union, a wider `T`, which makes the check of the result stricter.

### 3.5 Runtime type tests: `is` and `as`

PHP's `instanceof` only works on class types. Phaenon keeps `instanceof` unchanged and adds two
operators (precedent: Hack):

```php
if ($x is list<int>)       { /* $x: list<int> here — narrowing */ }
if ($x is ?User)           { ... }
if ($x is T)               { /* only if T is reified — else compile error P3xxx */ }
$y = $x as map<string, Money>;   // TypeError on failure, typed as map<string, Money> after
$y = $x as? Money;               // null on failure
```

Semantics are structural and **deep by definition** (`list<int>` walks the list). The cost is
visible at the call site, which is the point: generic runtime checks exist, they are not free, and
the syntax says so. `is` on a scalar/class type is O(1) and compiles to a native check
(`is_int`, `instanceof`).

*Amended 2026-11-08 (§11.4, roadmap D23): deep is gone. `is` and `as` test only what PHP tests in
O(1); `$x is list<int>` is `P3011`, `$x is list` compiles to `\array_is_list()`.*

*Withdrawn 2026-11-09 (§11.5, roadmap D26): `is`, `as` and `as?` leave the language, and a test is
written as PHP writes it. The examples above, as far as PHP can test them:*

```php
if (is_array($x) && array_is_list($x)) { /* $x: list<mixed>, its elements unchecked */ }
if ($x === null || $x instanceof User) { /* $x: ?User */ }
$y = $x instanceof Money ? $x : null;    // $y: ?Money
```

### 3.6 Erased type parameters at runtime — the rule

Inside a body, an **erased** `T` can be used as a type (declarations, casts for the checker) but not
as a **value**: `new T`, `T::class`, `$x instanceof T`, `$x is T`, `T::staticMethod()` are compile
errors with a fix-it: "declare `T` as `reified`, or test against the bound". This is what makes
erasure honest instead of surprising. *(No `reified` since §11.2, and no `is` since §11.5,
2026-11-09.)*

---

## 4. Compilation to PHP (erasure)

### 4.1 Type erasure table

| Phaenon type | Emitted native PHP type | Emitted docblock |
|---|---|---|
| `T` (unbounded) | `mixed` | `T` |
| `T of Foo` | `Foo` | `T` |
| `T of int\|string` | `int\|string` | `T` |
| `?T of Foo` | `?Foo` | `?T` |
| `list<X>` | `array` | `list<X>` |
| `map<K, V>` | `array` | `array<K, V>` |
| `iterable<X>` | `iterable` | `iterable<X>` |
| `Collection<X>` | `Collection` | `Collection<X>` |
| `Closure(A): R` | `Closure` | `Closure(A): R` |
| `Closure(A, B=, C...): R` | `Closure` | `Closure(A, B=, C...): R` — PHPStan's own markers, read unchanged (§11.3) |
| `class-string<T>` | `string` | `class-string<T>` |
| `static` / `self` / `$this` | same | `static` … |
| type alias | the alias expanded | `@phpstan-type` emitted once + used by name *(§11: expanded inline, no `@phpstan-type`; so is a named shape, §11.6)* |

### 4.2 Docblocks are the interop contract

Every erased signature is emitted with PHPStan/Psalm-vocabulary docblocks:

```php
/**
 * @template T of AggregateRoot
 * @template-covariant TEvent of DomainEvent
 */
final class EventStream
{
    /**
     * @param T $root
     * @param list<TEvent> $events
     * @return static
     */
    public function append(AggregateRoot $root, array $events): static { ... }
}
```

This means: PHPStan at `level: max` on the output is part of the compiler's test suite (an
emitted file that PHPStan rejects is a compiler bug), PhpStorm understands the output, and
a PHP colleague reads it as "well-typed modern PHP", not as generated code.

Optionally (`--emit-type-attributes`), the compiler also emits machine-readable attributes
(`#[\Phaenon\Generic('T', of: AggregateRoot::class)]`, `#[\Phaenon\TypeOf('list<TEvent>')]`) for
reflection-driven tooling (serializers, Tequila's `reflection`/`object-mapper`) that would rather
not parse docblocks. Docblocks remain the primary channel because they need no runtime dependency.

### 4.3 The override-erasure rule (a PHP-specific trap)

```php
class Base<T>            { public function accept(T $x): void {} }
class IntBox extends Base<int> { public function accept(int $x): void {} }
```

Naïve erasure gives `Base::accept(mixed $x)` and `IntBox::accept(int $x)` — PHP rejects the child
at compile time (parameters are contravariant; `int` narrows `mixed`). TypeScript never meets this
because JS has no runtime signatures. Rule:

- A method that **overrides or implements** a generic signature erases its parameter types to the
  **parent's erased types**, keeps the precise type in the docblock, and (under `--runtime-guards`,
  §6) gets a shallow assertion inserted as the first statement.
- Return types may narrow (PHP allows covariant returns), so `IntBox::accept(): int` stays `int`.
- Properties are invariant in PHP, so a generic property's erased type is fixed by the
  declaring class (`mixed` or the bound) and redeclarations are rejected by the checker.

### 4.4 What a PHP caller sees

A generic, non-reified Phaenon function or class compiles to a function or class with native
PHP types and docblocks — a PHP caller cannot tell it was generic, and PHPStan checks the call as
it would any vendor code. **No hidden parameters, no runtime dependency, no wrapper.**

---

## 5. Runtime generics: `reified`

*Superseded 2026-10-29: `reified` was removed, see §11.2. The text below is the draft as written.*

### 5.1 The idea

The question "can we have `$x instanceof T` for real?" has exactly one honest answer in a
transpile-to-PHP design: **only if a runtime value representing `T` exists.** The programmer
asks for that value with `reified`, and the compiler threads it through — the same mechanism as
Hack's `reify`, Kotlin's `reified inline`, and what C#/.NET do under the hood (a type handle
passed with the instance).

The payoff for interop is large: the compiled form of a reified type parameter is **the
`class-string<T>` idiom PHP already uses today**. `$repo->get<User>($id)` in Phaenon compiles to
`$repo->get(User::class, $id)` — a PHP caller writes exactly what they write now.

### 5.2 Reified function and method parameters

```php
function decode<reified T of object>(string $json): T
{
    $data = json_decode($json, true);
    return $this->hydrator->hydrate($data, to: T);      // T as a value
}

$user = decode<User>($payload);
```

Compiles to:

```php
/**
 * @template T of object
 * @param class-string<T>|\Phaenon\Type<T> $T
 * @return T
 */
function decode(string|\Phaenon\Type $T, string $json): object
{
    $T = \Phaenon\Type::from($T);
    $data = json_decode($json, true);
    return $this->hydrator->hydrate($data, to: $T);
}

$user = decode(User::class, $payload);
```

Rules:

- Reified parameters become **leading** parameters, in declaration order, before the visible ones
  (leading, not trailing, so optional/variadic visible parameters keep working and PHP's
  "required after optional" deprecation never triggers). Named-argument calls stay valid; the
  hidden parameter's name is the type parameter's name (`T:`), which is also how a PHP caller can
  pass it by name.
- The accepted PHP type is `string|\Phaenon\Type` when the bound is a class/interface
  (so `User::class` works from PHP), `\Phaenon\Type` otherwise (e.g. `reified T` used for
  `list<int>`-style structural types). The first statement normalises to a `Type`.
- `T` as a **value** in the body means the `Type` object: `T->check($x)`, `T->name()`,
  `new T(...)`, `T::class` (→ `$T->className()`), `$x instanceof T` / `$x is T` (→ `$T->check($x)`).
- `new T(...)` is allowed when the bound is a class type; the constructor is checked against the
  **bound's** constructor. PHP lets subclasses change constructors, so this is as (un)sound as
  PHP's own `new $className(...)` — a `TypeError`/`ArgumentCountError` at runtime is the worst case.
- Static methods may have their own reified parameters; they may **not** use the class's type
  parameters (no instance to carry them).

### 5.3 Reified class parameters

```php
final class Repository<reified T of AggregateRoot>
{
    public function __construct(private readonly EventStore $store) {}

    public function get(Uuid $id): T
    {
        $root = $this->store->load($id);
        if (!$root is T) { throw AggregateNotFound::for(T, $id); }
        return $root;
    }

    public function fresh(Uuid $id): T { return new T($id); }
}
```

Compiles to:

```php
/** @template T of AggregateRoot */
final class Repository
{
    /** @var \Phaenon\Type<T> */
    private readonly \Phaenon\Type $__T;

    /** @param class-string<T>|\Phaenon\Type<T> $T */
    public function __construct(string|\Phaenon\Type $T, private readonly EventStore $store)
    {
        $this->__T = \Phaenon\Type::from($T);
    }

    /** @return T */
    public function get(Uuid $id): AggregateRoot
    {
        $root = $this->store->load($id);
        if (!$this->__T->check($root)) { throw AggregateNotFound::for($this->__T, $id); }
        return $root;
    }

    /** @return T */
    public function fresh(Uuid $id): AggregateRoot { return $this->__T->instantiate($id); }
}
```

- The type argument is the **leading constructor parameter**; it is stored in `$__T`
  (readonly, private). All instance members see it as `T`.
- Reified parameters of a class are **invariant** (a runtime value can't be variance-checked).
- `Repository<User>` and `Repository<Admin>` are the **same PHP class** — `instanceof Repository`
  works, static state is shared, reflection sees one class. This is the big difference from
  monomorphisation (§7).

### 5.4 Binding subclasses — how DI and PHP callers construct reified classes

A DI container (Symfony autowiring) cannot invent a type argument. The idiomatic answer is one
Tequila already uses by hand: a concrete subclass binds the parameter.

```php
final class UserRepository extends Repository<User> {}
```

Compiles to:

```php
final class UserRepository extends Repository
{
    public function __construct(EventStore $store)
    {
        parent::__construct(User::class, $store);
    }
}
```

The compiler synthesises the constructor (or rewrites the explicit one to pass the bound type to
`parent::__construct`). `UserRepository` has a plain, autowirable signature. The checker **warns**
(`P4xxx`) when a reified generic class is a Symfony service without a binding subclass, since
that will fail at container compile time — the diagnostic says so and names the fix.

Inheritance chain: a generic subclass may re-export the parameter (`class CachedRepository<reified T of AggregateRoot> extends Repository<T>`) — the compiled constructor forwards `$T`.

### 5.5 The runtime library `\Phaenon\Type`

*Removed 2026-11-08 (§11.4, roadmap D23–D24): the compiled output calls no runtime; the hoisting
sketched below was measured and dropped.*

Shipped as the Composer package `phaenon/runtime` (plain PHP, no extension, required only by
output that uses `reified`, `is`/`as` on structural types, or `--runtime-guards`). It is a
small immutable type-descriptor tree mirroring the checker's types:

```
Type::class(string $fqcn, Type ...$args)        Type::int()  ::string()  ::float()  ::bool()
Type::list(Type $elem)                           Type::map(Type $key, Type $value)
Type::nullable(Type $t)                          Type::union(Type ...$members)
Type::literal(mixed $v)                          Type::mixed()

$t->check(mixed $v): bool          // deep for containers, O(n); shallow() for O(1) top-level
$t->assert(mixed $v): mixed        // TypeError with a precise message, returns $v
$t->instantiate(mixed ...$args)    // class types only
$t->className(): string            // class types only
$t->name(): string                 // 'list<int>', 'Repository<User>' — for messages
$t->equals(Type $o): bool, isSubtypeOf(Type $o): bool
```

Descriptors are **interned** (`Type::class(User::class)` returns a shared instance), and the
compiler hoists closed types out of hot paths (`static $__t ??= Type::list(Type::int())`), so the
per-call cost is a property read, not an allocation. `Type::from(string|Type)` accepts a
class-string for PHP-caller ergonomics.

Reflection of a reified type at runtime (`T->name()`) gives precise names in error messages — a
practical win over PHP today, where `AggregateNotFound` can only say "User" if you passed it.

### 5.6 Cost model (what `reified` buys and costs)

| | erased `T` | `reified T` |
|---|---|---|
| Runtime dependency | none | `phaenon/runtime` |
| Signature change | none | one leading `string\|Type` param per reified T |
| `$x instanceof T`, `new T`, `T::class` | ✖ compile error | ✔ |
| Per-call overhead | 0 | one `Type::from()` (interned lookup) |
| Variance | `in`/`out` allowed | invariant |
| PHP caller | unaware | passes `Foo::class` — today's idiom |
| DI | n/a | via binding subclass (§5.4) |

---

## 6. Runtime guards (dev mode)

*Withdrawn 2026-11-08 (§11.4): never built; a guard on a structural type is the deep check by
another name.*

Erasure means a PHP caller can pass `list<string>` into a `list<int>` parameter and nothing stops
it at runtime. The checker is the enforcement for Phaenon-to-Phaenon calls; for the boundary with
untyped code the compiler offers `--runtime-guards=boundary|all|off`:

- `boundary` (recommended in dev/test): shallow `Type::assert()` on parameters of **public**
  members and on values coming from `as`-free deserialisation points the compiler can see.
- `all`: also private members and property writes — for debugging type-hole hunts.
- `off` (recommended in prod): emitted code is identical to a human's.

Guards never change behaviour of correct programs, and the output with `off` has no dependency on
`phaenon/runtime` unless `reified`/`is` is used.

---

## 7. Alternatives considered for runtime generics

| Strategy | What it is | Why not primary |
|---|---|---|
| **Pure erasure** (TypeScript, Java) | no runtime representation ever | fails goal 3; "you can't `instanceof T`" is the complaint that would follow Phaenon forever |
| **Monomorphisation** (C++ templates, Rust) | emit `Repository__User extends Repository` per instantiation; `instanceof T` becomes `instanceof User` literally | needs whole-program knowledge (type args can come from PHP callers and from other generics at runtime); specialising a library's generic for an app's type means shipping library source to the consumer's compile (how Rust does it, and why its builds are slow); class identity splits (`Repository<User>` ≠ `Repository<Admin>` as PHP classes, separate static state); mangled names in stack traces and DI configs; unbounded instantiation with polymorphic recursion. Kept as a **possible optimisation** behind `reified` later (a known-closed set of instantiations could be specialised with identical semantics) |
| **Dictionary passing** (Haskell, Swift witness tables) | pass a vtable of operations instead of a type | strictly more general than we need; PHP has dynamic dispatch already, so passing the *type* is the vtable |
| **Runtime type metadata on every value** (boxed values) | wrap `list<int>` in a `TypedList` object | changes value semantics of arrays, costs everywhere, and breaks every PHP function that takes `array` |
| **Reified by default** (C#) | every generic carries its type | hidden params on every generic signature → PHP callers must pass types for things they don't care about; variance lost; runtime dependency everywhere |

**Decision: erased by default, `reified` opt-in, compiled to the `class-string<T>` idiom.**
It is the only option that keeps goal 4 (zero cost by default) and goal 5 (interop) while giving
a *real* `instanceof T` where it is asked for.

*Superseded 2026-10-29: the idiom stays, the opt-in does not — a `class<T>` value the programmer
writes replaces the compiled hidden parameter, see §11.2.*

---

## 8. Interop details

### 8.1 Calling PHP generics from Phaenon

The checker reads vendor `.php` as already typed: native types first, docblocks as refinement
(`@template`, `@param`, `@return`, `@var`, `@phpstan-type`, `@psalm-*`, `@template-covariant`).
PHP's own standard library comes from php-src's `*.stub.php` files (native signatures) overlaid
with a generics map for the functions that are generic in spirit (`array_map`, `array_filter`,
`iterator_to_array`, …) — the same overlay PHPStan maintains. A vendor `class-string<T>` parameter
plus `@return T` is understood as a reified-style call site, so `$container->get(User::class)` is
typed `User` without annotation.

### 8.2 Calling Phaenon from PHP

| Phaenon declaration | PHP caller writes |
|---|---|
| `first<T>(list<T> $xs): ?T` | `first($xs)` — unaware |
| `decode<reified T of object>(string $json): T` | `decode(User::class, $json)` |
| `new Repository<User>($store)` (reified) | `new Repository(User::class, $store)` or `new UserRepository($store)` |
| `$x is list<int>` | n/a (compiled to `Type::list(Type::int())->check($x)`) |

*The last row went with §11.4 and §11.5 (2026-11-09): a type test is PHP's own, and a PHP caller
and a Phaenon one write the same.*

### 8.3 Symfony specifics

- Autowiring: reified classes need a binding subclass (§5.4) or an explicit `arguments:` entry
  with `User::class`; the checker warns.
- Serializer / Messenger / Validator read erased native types and docblocks exactly as today.
- Attributes with generic types in Phaenon (`#[MapTo<User>]`) are erased to `#[MapTo]` +
  a `class-string` argument if the attribute declares a reified parameter — attributes are
  constructed by reflection, so this is just §5.2 applied.

### 8.4 PHPStan on the output

Part of the compiler's CI: every golden-file test's emitted PHP must pass PHPStan at `level: max`
with the Phaenon-generated docblocks. This is how we keep "the output is idiomatic typed PHP"
honest over time.

---

## 9. Open questions (to iterate on)

1. **`class-string<T>` spelling.** Hyphenated names are odd as real syntax. Candidates:
   keep `class-string<T>` (PHPStan familiarity) · `classname<T>` (Hack) · `T::class` as a type.
2. **Covariant map keys.** PHP compares keys by value; `map<Admin, V>` can't exist (objects aren't
   keys), so `K` is always `array-key`-ish — should `K` be `out`?
3. **Generic traits.** Traits copy code into the using class; a generic trait could take its
   parameters from the using class (`use Timestamps<T>`) — easy to erase, unclear value. Deferred.
4. **Reified closures / generic closure expressions with `reified`.** Would need a hidden
   bound variable; doable, deferred.
5. **Type aliases with reified use.** *(Closed with `reified`, §11.2, and no `is` since §11.5.)*
   `type Ids = list<Uuid>; $x is Ids` expands fine; aliases
   containing reified parameters of an enclosing class — allowed or not?
6. **Should `is` on a container be deep by default?** Deep is correct and the cost is explicit;
   Hack chose shallow for `is` on generics (`vec<_>`) because deep was too costly in practice.
   Possible compromise: `is list<int>` deep, `is list<_>` shallow (wildcard = don't check elements).
   *Answered again 2026-11-08 (§11.4): shallow, and only where PHP has the test. Closed 2026-11-09
   (§11.5): there is no `is`; PHP's tests are what a program writes.*
7. **Monomorphisation as an opt-in optimisation** for closed instantiation sets (e.g. a final
   reified class instantiated only from Phaenon code) — same semantics, no hidden parameter.
8. **Diagnostics numbering and messages** — define the `P1xxx` (syntax) / `P2xxx` (inference) /
   `P3xxx` (erasure misuse) / `P4xxx` (interop) families when the checker exists.

Asked by §11.6 (2026-11-15), about aliases of a namespace:

9. **Aliases of sources outside the program.** For an installed Phaenon package, and for the
   rest of the project in a run over given paths, §11.6 finds aliases by reading the `type`
   declarations under a PSR-4 prefix, the first time a name under it is not a class. The cost
   of that reading on a large tree (core has 800 files) is not measured. The alternative is a
   rule of placement: an alias sits in the file PSR-4 gives its name, so that it is found as a
   class is. That costs nothing to find, and it takes away the natural place for a type,
   beside the class that returns it. To be settled by measuring when the table is built.
   *Measured 2026-11-15, on core's 811 sources under two prefixes:* a run over one path that
   names something which is no class reads them in about 25 ms, and a run that names classes
   alone reads nothing; a check of the whole project, which holds the sources already, pays one
   walk of their directories, about 8 ms in a second. At that cost the reading stays and no rule
   of placement is made (§11.7). The question is kept for a tree an order of magnitude larger.
10. **Does a type remember its alias?** An alias expands when it is resolved, so a diagnostic and
    the server's hover print the expansion: `expects array{id: int, name?: string}`, not
    `expects Row`. A named shape would read better by its name. That asks the lattice to carry
    a name that plays no part in subtyping or equality.
11. **`@phpstan-type` in the output.** Inline expansion repeats a long shape at every use, where
    a person would have written `@phpstan-type` once per class. Emitting the tag on each class
    that uses an alias, with the expansion as its body, would shorten the docblocks and need no
    `@phpstan-import-type` between classes. Not taken now: §11's reason holds, and the
    expansion is already built. To be weighed on the first dogfood tree that names its shapes.

Left by building §11.6 (2026-11-15), about a run over given paths, which checks the files it was
given and loads the rest by name:

12. **A name declared twice is not reported in a run over a path.** `P2511` is reported at the
    second declaration when its file is checked. In a run over a path the other file is read on
    demand and not checked, so of two aliases of one name the run reports the collision only when
    both files are among its paths. A run over the project reports it always. The same is true
    of two classes.
13. **An alias outside the directories of a PSR-4 prefix is not found on demand.** The reading
    of §11.7 follows the PSR-4 prefixes that have a source root: a source Composer loads
    through `files` or a classmap, and a source of a project without a project file, are read
    only when something names a class of theirs. A run over the project has them.

---

## 10. Worked example (Tequila-flavoured)

```php
// Phaenon
interface EventStore<out TEvent of DomainEvent>
{
    public function load(Uuid $id): list<TEvent>;
    public function append(Uuid $id, list<TEvent> $events, int $expectedVersion): void;
}

abstract class AggregateRepository<reified T of AggregateRoot, TEvent of DomainEvent>
{
    public function __construct(private readonly EventStore<TEvent> $store) {}

    public function get(Uuid $id): T
    {
        $events = $this->store->load($id);
        if ($events === []) { throw AggregateNotFound::of(T, $id); }   // T->name() in message
        $root = T::reconstitute($events);                                // static call on reified T
        return $root;
    }

    public function save(T $root): void
    {
        $this->store->append($root->id(), $root->releaseEvents(), $root->version());
    }
}

final class UserRepository extends AggregateRepository<User, UserEvent> {}
```

```php
// Emitted PHP
/** @template-covariant TEvent of DomainEvent */
interface EventStore
{
    /** @return list<TEvent> */
    public function load(Uuid $id): array;
    /** @param list<TEvent> $events */
    public function append(Uuid $id, array $events, int $expectedVersion): void;
}

/**
 * @template T of AggregateRoot
 * @template TEvent of DomainEvent
 */
abstract class AggregateRepository
{
    /** @var \Phaenon\Type<T> */
    private readonly \Phaenon\Type $__T;

    /**
     * @param class-string<T>|\Phaenon\Type<T> $T
     * @param EventStore<TEvent> $store
     */
    public function __construct(string|\Phaenon\Type $T, private readonly EventStore $store)
    {
        $this->__T = \Phaenon\Type::from($T);
    }

    /** @return T */
    public function get(Uuid $id): AggregateRoot
    {
        $events = $this->store->load($id);
        if ($events === []) { throw AggregateNotFound::of($this->__T, $id); }
        $root = ($this->__T->className())::reconstitute($events);
        return $root;
    }

    /** @param T $root */
    public function save(AggregateRoot $root): void
    {
        $this->store->append($root->id(), $root->releaseEvents(), $root->version());
    }
}

/** @extends AggregateRepository<User, UserEvent> */
final class UserRepository extends AggregateRepository
{
    public function __construct(EventStore $store) { parent::__construct(User::class, $store); }
}
```

Compare with what Tequila writes by hand today: the same shape, minus the hand-written
`class-string` plumbing and the docblock duplication — and with `get()` actually checked.

---

## 11. Decisions made while building M3 (2026-10-28)

What the implementation settled that the draft above left open or got wrong. The draft text is
kept as written; this section wins where they differ.

| Topic | Decision | Why |
|---|---|---|
| **Line preservation** | A generic declaration adds lines to the output: its docblock. Everything else stays line-exact. | PHPStan reads one tag per line (`/** @template T @param T $x */` keeps only the first tag — verified against PHPStan 2.2). An inline single-tag docblock would save a line but is not how PHP is written; readable output (goal 2) wins over M1's line-exactness for these declarations. Docblocks that already exist are extended in place; tags already present are not repeated. |
| Keyed arrays | `map<K, V>` is the native keyword; it erases to `array` and prints as `array<K, V>` in docblocks. `non-empty-map` likewise. | `array<K, V>` in native syntax reads as "PHP array with magic"; `map` says what it is. |
| `array-key`, shapes | *(Amended 2026-11-15, §11.6 and `collections.md` §6: a shape is native syntax in the right-hand side of a `type` declaration, and nowhere else.)* Not native syntax in M3. A bound needs `int\|string`; a shape (`array{a: int}`) can only come from a docblock or an alias body — and it cannot be tested at runtime. | Hyphenated names and `{}` in types are an M4 question (strings/collections). |
| Type aliases | *(Amended 2026-11-15 by §11.6: an alias belongs to its namespace, is found from any file of the project and imported with `use`. Generic and expanded inline as before.)* `type X = …;` is file-level, namespace-scoped, generic (`type Result<T, E = \Throwable> = …`), expanded inline in emitted docblocks (no `@phpstan-type`). | PHPStan has no file-level aliases; inline expansion is what PHPStan sees either way. |
| `Closure` | Written as `\Closure` in emitted signatures and docblocks. | The checker accepts a bare `Closure` everywhere; PHP in a namespace would resolve it to `App\Closure`. *Narrowed 2026-11-06 (`checker.md` §9): where the type reaches PHP as written, a bare `Closure` in a namespace that does not import it is `P2001`; erased positions keep reading it as `\Closure`.* |
| `is` | *(Superseded by §11.4 and §11.5.)* Native where PHP has a check — `instanceof`, `is_int()`, `=== null`, `?Foo` and `A\|B` on a simple operand — otherwise `\Phaenon\Type::…->check()`. Deep by default (open question 6: decided). | Zero-cost when PHP can do it; the runtime cost is visible where it exists. |
| `as` / `as?` | *(Superseded by §11.4 and §11.5.)* Always the runtime: `->assert()` / `->orNull()`. | One precise `TypeError` message for every shape; the dependency is already there for structural `is`. |
| `is !T` | *(Superseded by §11.5: `!is_int($x)`, `!($x instanceof Foo)`.)* Dropped; write `!($x is T)`. | Nothing a negation prefix adds. |
| Reified call sites | The checker records the final binding of each reified parameter per call site; the emitter turns it into the leading argument (`\App\User::class`, `$T`, `$this->__T` or a descriptor). A binding must be a class type or a reified parameter in scope — `P3005` otherwise. Explicit arguments (`decode<User>()`) and inference from arguments both work. | The emitter never re-infers; one source of truth. |
| Runtime API | `Type::of(User::class)` not `Type::class(…)` (`Foo::class` is syntax); `from()` accepts `string\|Type`; `orNull()` is `as?`; `classString()` added; `literal()` covers scalars and enum cases; descriptors are interned by their printed name. | |
| Variance checks | `out` / `in` positions are checked on non-private, non-static members; constructors are exempt; `readonly` properties count as output positions; closures and array keys follow PHPStan. | Parity with PHPStan's template variance rules. |
| Static members | A static property cannot use the class's type parameters (`P3008`): one slot is shared by every instantiation. A static method may: late static binding resolves them, and `static::aggregateClass(): class<T>` is one of the idioms of §11.2 (relaxed 2026-10-29; the first cut forbade both). | Nothing carries the parameter into a shared slot; a static call always has a class. |
| Where `reified` may appear | Classes, functions, methods. Not interfaces, traits, enums, closures, type aliases (`P3006`). A reified parameter is invariant. | |
| Override erasure | Implemented for parameters (widen to the parent's erased type) **and** return types (narrow to the parent's when the child's erasure would be wider). | A child's `T` erasing to `mixed` under a parent's `User` return would be a PHP fatal. |
| Generic properties | A property whose type mentions a class parameter cannot be redeclared in a subclass (`P3007`). | Property types are invariant in PHP; the declaring class fixes the erasure. |
| Binding subclasses | `final class UserRepository extends Repository<User> {}` gets a synthesised constructor forwarding the parent's parameters with `User::class` first; an explicit constructor must call `parent::__construct(...)`, which gets the argument inserted. Re-exported parameters (`extends Repository<T>`) forward `$this->__T`. | §5.4 as designed. |
| Closures and reified `$T` | A `function` closure inside a reified function that mentions `T` gets `$T` added to its `use (...)`; arrow functions capture by value anyway; in classes `$this->__T` needs nothing. | |
| Diagnostics | `P3001` arity · `P3002` bound · `P3003` erased T used as a value · `P3004` variance · `P3005` reified not inferable / not closed · `P3006` reified not allowed here · `P3007` generic property redeclared · `P3008` class parameter in a static property · `P3009` type arguments on a non-generic callee. | Open question 8: answered. |
| Docblocks in `.phn` | *Superseded 2026-11-05: translated for PHPStan (collections.md §4).* Passed through untouched; a `map<…>` written in a `@var` in Phaenon source reaches PHPStan as is. Types belong in the syntax in `.phn` files. | Rewriting user prose is a later migration concern. |

### 11.1 Migrating docblock generics (`phn migrate`, 2026-10-28)

| Decision | Choice | Why |
|---|---|---|
| What is lifted | `@template[-covariant\|-contravariant] T [of B] [= D]` → `<[out\|in] T of B = D>` on classes, interfaces, traits, functions, methods (constructors included); `@extends` / `@implements` → arguments on the matching clause; `@param`, `@return`, property `@var` → the declared type. `@phpstan-*` / `@psalm-*` variants too. | These are the tags that *are* generics in PHPStan code. `@var` on statements, `@phpstan-type`, `@method`, `@mixin` stay (no Phaenon construct for them yet). *2026-11-15 (§11.6): `@phpstan-type` and `@psalm-type` have one, the `type` declaration, and `@phpstan-import-type` is `use`; both are lifted where the name is free in its namespace.* |
| Safety rule | A tag is lifted only when (1) its type has a Phaenon spelling and (2) it **erases to exactly the declared native type**; an untyped parameter accepts only a type that erases to `mixed`; an untyped return is never refined. Everything else stays, with a note on stderr saying why. *2026-11-02:* "erases to" means what the emitter writes, the override-erasure rule (§4.3) included: in a method that overrides or implements a generic signature, `@param T` lifts when T widens to the parent's compiled parameter type, as `AbstractEventSourcedRepository::save(Aggregate)` under `AggregateRepository::save(T)` does; migrate reads the parent from the same program model `phn check` builds. | The emitted PHP must keep the signature PHP saw before: migrating must not change what overrides, callers or reflection see. |
| Spelling | `T[]`, `array<array-key, V>` and `array<int\|string, V>` → `array<V>`; `array<K, V>` with any other key → `map<K, V>` (`non-empty-array` → `non-empty-map`); `array-key` elsewhere → `int\|string` (2026-11-05, collections.md §5); `X\|null` → `?X`; `$this` → `static`; `\Closure(A): R` → `Closure(A): R`, its `T=` and `T...` parameters kept, its parameter names dropped (§11.3); class names resolved through `use` so `\App\Row` and an imported `Row` match. Not spellable: `callable(…)` signatures, named shapes, literals, `int<a, b>`, conditional types, by-reference closure parameters. *2026-11-15 (§11.6): a shape with named keys is spellable as the right-hand side of a `type` declaration, so a `@phpstan-type` lifts; written inline in a `@param`, `@return` or `@var` it still has no spelling.* | What a human would write in `.phn`. |
| Hyphenated names | `class-string<T>`, `non-empty-string`, `array-key`, `positive-int`, `key-of<T>`, … are native type syntax: the parser joins `name-name` written without spaces in a type position; `A - B` stays arithmetic. They erase through the resolved type (`string`, `int\|string`, …). | `class-string<T>` is the generic type Tequila uses most; without it the lift would skip half of reflector. |
| Prose-only `@param` | A lifted `@param T $x prose` becomes `@param $x prose` (the type is in the signature); the emitter fills the type back in on output, and the docblock reader accepts the form. `@return T prose` and `@var T prose` keep their type too (no unambiguous type-less form). | Like TypeScript's JSDoc: prose stays, types are not repeated. |
| Emptied docblocks | Removed when only the frame is left; blank `*` lines at the edges collapse; generated tags are inserted **before** the first remaining tag, so `@throws` keeps its place after `@return`. | Round-tripping a PHPStan file through migrate + build gives back the same docblock order. |
| Inheritance | A child method without docblock types inherits the parent's **effective** type when its own native type equals the parent's erasure (`string $x` under `class-string<T> $x`), through `@extends`/`implements` substitution. | PHPStan's implicit `@inheritDoc` applies to the lifted syntax as it did to docblocks; `CachedReflector::reflect(string)` keeps `class-string<ClassType>`. |
| Promoted properties | A method-level `@template` that a promoted parameter's type mentions stays in the docblock (with its `@param`), noted as "make it a class type parameter". | A property cannot carry a method type parameter; whether the class should become generic is a design decision per class, not a mechanical one. |
| CLI | `phn migrate` takes `.php` (strip + lift) and `.phn` (lift only, rewritten in place when changed); `--keep-docblocks` is the mechanical v0. Notes print as `file:line: kept @tag …`. | One command for new and already-migrated projects. |
| **`P3010`: a type argument nothing determines** (2026-11-07) | A warning at a call whose callee has a type parameter in its return type that no argument, no closure, no explicit argument and no expected type determined, and that has no bound to fall back on: the result is typed `mixed` from there (`$b = emptyBox();`, `first([])`). Silent when an argument the checker could not read is the reason, reported where it is if at all: a `mixed` or an unknown, a `never` (an unreachable branch), a generic class written without its arguments, or a callable parameter given anything but a closure with a signature (`[$this, 'method']`, a string) or given a closure that returns nothing (`run(fn() => $this->log())`, whose result is not used). | PHPStan is silent and infers `mixed`, which then flows through the program unchecked; the warning names the one place a declared type fixes it. The silences keep it honest on the dogfood: with them the four Tequila trees (core, mongodb-odm, object-mapper, reflector) report none, without them core reported 25 void closures passed to `run()` and mongodb-odm two `iterator_to_array()` of an unreachable branch. |

`--runtime-guards` (§6), monomorphisation (§7) and hoisting descriptors out of hot paths
(`static $__t ??= …`) stayed open after M3 and closed on 2026-11-08 without being built (§11.4). The `P3010` warning for an erased parameter that nothing
constrains (§3.4 item 4) is built (2026-11-07, the last row above). The Symfony DI warning of §5.4 went with
`reified` (§11.2).


### 11.2 `class<T>`, and the end of `reified` (2026-10-29)

Settled after the migrate step, when the compiled output of reflector and object-mapper was read
as a PHP developer would read it. Where this table and §11.1 differ, this one wins.

| Topic | Decision | Why |
|---|---|---|
| **`reified` is removed** | No keyword, no hidden leading parameters, no `$__T` property, no synthesised or rewritten constructors, no binding subclasses (§5.4), no Symfony DI warning. Codes `P3005` and `P3006` are retired and reserved, never reused. `P3003` (erased type parameter used as a value) stays; its message now points to `class<T>`. | The hidden parameter fought Symfony DI — constructors are the wiring surface — and made the output read as machine-generated (`string\|\Phaenon\Type $T`, `$this->__T->check()`). Tequila already carries `T` at runtime by hand, in three string-typed, DI-transparent idioms (next row), and `reified` compiled to one of them anyway. Across Tequila's 51 `src/` trees (4,963 files): `instanceof $x` on 24 lines, `new $x` on 13, `is_a()` on 37, `is_subclass_of()` on 25 — every one with the class string already in hand; `::class` on 1,932 lines in 721 files. Not one site needs a hidden type parameter. |
| **A constructor's type parameter may type a promoted property** (2026-10-29) | `public function __construct<C of object>(private Reflector<C> $reflector)` is legal and is how a dependency whose type arguments do not matter to the class is declared: the property holds a `Reflector<C>` for some `C`, and each use infers `C` again from the arguments (§9, *foreign templates are inferred*). It compiles to the PHPStan form Tequila writes by hand — `@template C of object` and `@param Reflector<C> $reflector` on the constructor — and PHPStan reads the promoted property the same way. `phn migrate` lifts these templates like any other (it used to keep them with a note: three in object-mapper, nine in mongodb-odm). No "any arguments" spelling is added: PHPStan's projections (`Reflector<*>`, `Reflector<covariant Foo>`) turn the parameters in input positions into `never`, which is not what a dependency holder means, and a bare `Reflector` would make PHPStan report `missingType.generics` on the output. | The existential type is what the code means; PHPStan already has the spelling and the inference for it, and so does the checker. |
| **Runtime generics are written, not synthesised** | A generic that needs `T` at runtime holds a `class<T>` value, in one of the idioms PHP already uses: (a) a constructor parameter `class<T> $aggregateClass` (Tequila `core`'s `InMemoryAggregateRepository`); (b) an abstract static getter `aggregateClass(): class<T>` overridden by the concrete subclass, which is the DI service (`core`'s `AbstractEventSourcedRepository`); (c) a class name carried by an attribute and read by a factory that does `new $repositoryClass(...)` (`mongodb-odm`'s `AttributeBasedRepositoryFactory`). The checker types all three precisely; the language introduction (phaenon.dev) shows them. | The PHP idiom is readable, autowirable and already understood by every PHP developer and by PHPStan; the compiler has nothing to add to it but types. |
| **`class<T>`** | A native type: the type of the expression `T::class`, a subtype of `string`. Bare `class` is any class name (`class<object>`). Erases to `string`; prints in emitted docblocks as `class-string<T>`. Parsed from the `class` keyword in a type position (Phaenon mode), so no name-joining rule exists. With `$c` of type `class<T>`, the checker narrows `$x instanceof $c` to `T`, types `new $c(...)` as `T` and checks `$c::method()` against `T` — as it already did for `class-string<T>` read from docblocks. | Rejected: a runtime class object (Python-style) — breaks attribute arguments (constant expressions only), `services.yaml` strings and `===` at every PHP boundary. Rejected: keeping `class-string<T>` as the one hyphenated exception — a special parser rule for a single name. `class<T>` reads as "a class of T" and mirrors `T::class`. |
| **Hyphenated PHPStan names are not syntax** | `non-empty-string`, `positive-int`, `array-key`, `non-empty-list`, `int<a, b>`, `key-of<T>`, … are read by the type lattice from docblocks — in `.php` and `.phn` alike, as refinements of a native type, exactly as PHPStan reads them — but cannot be written as native types. The surface stays small on purpose: PHP's native types, `list<T>`, `map<K, V>`, `Closure(A): R`, `class<T>`, type parameters and aliases, plus what M4 adds deliberately. `array-key` is written `int\|string`; M4 decides the key bound of `map`. `phn migrate` lifts `class-string<T>` to `class<T>` and keeps the other hyphenated tags in docblocks with a note. Supersedes the "Hyphenated names" row of §11.1. | PHPStan is a tool's vocabulary, not a type-system design; the Go rule applies — the general type unless the use case demands otherwise — or the number of types explodes. Tequila's own `src/` writes `list<…>` on 2,007 lines against 22 `non-empty-string`, 17 `positive-int`, 67 `array-key` and 4 for all the rest combined. The §11.1 row was justified by the migration (reflector's lift), not by the language. |
| **`\Phaenon\Type` narrowed** | The runtime serves structural `is` / `as` only (`$m is list<int>`), where PHP has no check of its own. It never appears in a signature. `Type::from()` and `Type::instantiate()` are removed. *Removed altogether 2026-11-08 (§11.4).* | What is left is an honest library call at the site that asked for a deep check; nothing hidden. |
| **Emitter principle** | Roadmap D9: the emitted PHP contains no identifier the programmer did not write — no hidden parameters, no synthesised members, no renamed locals. The compiler adds only the header line, `declare(strict_types=1)`, docblocks, and `phaenon/runtime` calls where the source used `is` / `as` on a type PHP cannot test. *The last exception ended 2026-11-08 (§11.4, D24).* | The output must read as PHP a person wrote; "generated-looking" PHP would be the complaint that follows Phaenon forever. |

### 11.3 Optional and variadic closure parameters (2026-11-01)

Settled on the tq-cli dogfood, where `phn migrate` left `@var \Closure(string ...$args): array{int,
string}` docblocks over `\Closure` properties because a closure type's parameters were types only.

| Topic | Decision | Why |
|---|---|---|
| **`T=` and `T...` in a closure type** | `Closure(string, int=): void` has an optional second parameter, `Closure(string, string...): (int, string)` a variadic last one. A variadic parameter is last and at most one; after an optional parameter every parameter is optional or variadic (the parser rejects the rest). The type erases to `\Closure` with the signature spelled the same way in the docblock: `@var \Closure(string, string...): array{int, string}`. The checker treats a call on such a value as a call on a function with those parameters (`()` on `Closure(string, string...)` is P2101, an optional parameter may be omitted) and checks a closure literal against the type parameter by parameter: a variadic or optional parameter of the type is satisfied only by a variadic parameter or one with a default, as PHPStan rules (`Closure(string...)` does not accept `fn (string $s)`, which cannot be called with no argument). `phn migrate` lifts them; a by-reference parameter keeps the docblock, as before. | These are PHPStan's own markers for `callable(…)` and `Closure(…)` docblock types, so the erased docblock is read by PHPStan unchanged and `phn migrate` lifts the forms Tequila writes: tq-cli's eleven `Closure(string ...$args)` kubectl signatures and the `bool=` of its http boundary. By-reference closure parameters are rare enough (none in Tequila's sources) not to deserve syntax. |
| **No parameter names** | `Closure(string $stdin, string ...$args)` is not Phaenon; the lift drops the names. | Phaenon's closure type never had them, and in a type they add nothing a checker or a reader needs: the closure literal names its own parameters. PHPStan accepts the nameless form. |

### 11.4 Native tests only, and the end of `\Phaenon\Type` (2026-11-08)

Settled when the descriptor hoisting of §5.5 was designed and measured, and the output was read
again as a PHP developer reads it. Two findings closed it. PHPStan types every `static` variable
`mixed`, whatever its initialiser, so a hoisted descriptor costs two compiler lines per function
(`/** @var Type<list<int>> $__t */` above `static $__t = …;`) in output whose promise is the PHP a
person would have written. And the saving was never where the cost is: with opcache on PHP 8.5,
one million calls, `list<int>` inline against hoisted was 231 against 79 ns on an empty array,
682 against 531 at ten elements and 4637 against 4463 at a hundred — the deep walk dominates
from ten elements on, and the operator that hides it reads as O(1). Roadmap D23 and D24. Where this
table and the text above differ, this table wins.

*Superseded 2026-11-09 by §11.5 (roadmap D26): `is`, `as` and `as?` left the language. The table
stays as the record of what each test compiled to, which is what `phn fmt` writes when it rewrites
a leftover one (`tooling.md` §5.6).*

| Topic | Decision | Why |
|---|---|---|
| **Checkable types** | `is` and `as` accept a type PHP tests in O(1), and compile to that test: `int` → `\is_int($x)`, `float`, `string`, `bool` likewise; `null` → `$x === null`; `array`, and `map` with `mixed` key and value → `\is_array($x)`; `object` → `\is_object($x)`; `callable` → `\is_callable($x)`; `iterable` → `\is_iterable($x)`; a class, interface, enum, `self` or `static` → `$x instanceof Foo`; `Closure` → `$x instanceof \Closure`; `?T` → `$x === null \|\| …`; `A\|B` → `\|\|`, `A&B` → `&&` of the members' tests; a literal or an enum case → `$x === 'a'`, `$x === Suit::Hearts`; `class<Foo>` → `\is_string($x) && \is_a($x, Foo::class, true)` and bare `class` → `\is_string($x) && (\class_exists($x) \|\| \interface_exists($x))`, as since §11.2; `list` → `\is_array($x) && \array_is_list($x)`; a tuple `(int, string)` → `\is_array($x) && \array_is_list($x) && \count($x) === 2 && \is_int($x[0]) && \is_string($x[1])`. A conjunct the operand's static type already settles is dropped: on an `array` operand, `is list` is `\array_is_list($x)` alone. | Every form is the line a PHP developer writes, and PHPStan narrows each of them on the output; a tuple's test is O(arity), its members fixed by the type. |
| **Uncheckable types: `P3011`** | `list<T>` with `T` other than `mixed`; `map<K, V>` with a key or value other than `mixed`; a `Closure(A): R` with a signature; `bytes` (a PHP string at runtime; `strings.md` §6 makes `as string` on a `bytes` operand the one exception); a tuple with an uncheckable member; `Foo<Bar>` whose arguments do not follow from the operand's static type; an alias expanding to any of these. The message names the test that exists: "`list<string>` cannot be tested at runtime in O(1): test `is list` and build the typed value, or check the elements yourself". No opt-out, no depth allowance. A type parameter stays `P3003`. | The deep check is writable in Phaenon as it is in PHP — `\is_array($x) && \array_is_list($x) && \array_all($x, fn ($v) => \is_string($v))` — and PHPStan 2.1 narrows through neither it nor a library call, so the hand-written form tells the truth about what the checker knows afterwards. Hack chose shallow `is` on generics for the same reason (§9 question 6, answered twice). |
| **`Foo<Bar>`** | Allowed, and compiled to `$x instanceof Foo`, when the operand's static type fixes the arguments: `Repository<User>\|null` tested with `is Repository<User>`, or a union whose `Repository` member has them. Otherwise `P3011`, with the fix "test `is Repository` and read the arguments from the value, or hold a `class<T>` (§11.2)". After `instanceof` on an operand that does not fix them, the type's parameters are their bounds, as PHPStan has it. | Kotlin's rule for erased generics: the test is allowed where erasure loses nothing. |
| **`as` and `as?`** | `$x as int` → `\is_int($x) ? $x : throw new \TypeError('expected int, got ' . \get_debug_type($x))`; `$x as? int` → `\is_int($x) ? $x : null`. `as?` is the operator only when the `?` touches `as`; `$x as ?int` is `as` with the nullable type, throwing on anything but an int or null (2026-11-08: the parser took the `?` either way, so a space changed the meaning). A compound type uses the same `is` form in the condition. A test the static type settles is reported as always true or always false, as `is` is today. | `cond ? $x : throw …` is PHP 8's own idiom for a checked value; `get_debug_type()` gives the message `TypeError` would have given. |
| **The operand: `P3012`** | A compound test, an `as` or an `as?` writes its operand more than once, so the operand is a variable, a property fetch or a static property fetch, a constant-index read of one of these (`$row['id']`, `$this->items[0]`), or a chain of them; never a call, a `new`, an operator or an assignment. Anything else is `P3012`: "the operand of `as` is evaluated twice; assign it to a variable first". A single-function test (`is int`, `is Foo`) takes any expression. | A compiler temporary (`\is_int($__v = f()) ? $__v : …`) is the machine-written output D9 forbids; binding the value first is what the hand-written line does. |
| **Narrowing** | After `is list`, the operand is `list<mixed>`, or `list<T>` when it was `array<T>`; after `is array` it is `array<mixed, mixed>`, or what it was; after `as string` on `bytes`, `string`. A hand-written `array_all(...)` narrows nothing beyond that. | Parity with PHPStan on the output (`checker.md` §9): the checker may be stricter than PHPStan, never more lenient, and PHPStan narrows `is_array`, `array_is_list`, `is_int` and `instanceof` but not `array_all`. |
| **`\Phaenon\Type` removed** | The class, its stub in the compiler and its tests leave `phaenon/runtime`; the emitter writes no call into any package (roadmap D24). §5.5's hoisting and §6's `--runtime-guards` are withdrawn unbuilt; §7's monomorphisation closes with them. | A guard on `list<T>` is the deep check by another name, and a guard on a scalar is the native parameter type PHP already enforces. Monomorphisation was an optimisation of reified instantiations, which went with §11.2. |

### 11.5 Types, not operators (2026-11-09)

Roadmap D26. §11.4 kept `is` and `as` to the tests PHP has; read again as a PHP developer reads a
`.phn`, what was left was a second spelling of those tests, an operator that compiled to the
function every reader already knew, under the word PHP reserves for a pattern match of its own
(`php-versions.md` §8). Where this table and the text above differ, this table wins.

| Topic | Decision | Why |
|---|---|---|
| **`is`, `as` and `as?` leave the language** | A type test is written as PHP writes it: `is_string($x)`, `$x instanceof Foo`, `$x === null`, `is_array($x) && array_is_list($x)`, `count($x) === 2`, `in_array($x, ['a', 'b'], true)`; a checked value is `is_int($x) ? $x : throw new \TypeError(…)`, PHP 8's own idiom. The emitter lowers no test: what the source writes is the output. | Phaenon adds no operation PHP lacks, so a reader knows what `is_string($x)` does without learning anything. Every test §11.4 admitted has a PHP form, the one it compiled to, and PHPStan narrows through it on the output as the checker does in the source. |
| **Narrowing** | Through PHP's tests, as PHPStan narrows (`checker.md` §3.4, §9): `is_array()` gives `array<mixed>` or what the operand was, `array_is_list()` a list, `instanceof Foo` a `Foo` whose type arguments are what the operand's static type fixes and otherwise their bounds, a literal comparison the literal. A hand-written element check (`array_all($x, fn ($v) => is_string($v))`) narrows nothing beyond `list<mixed>`, as in PHPStan. | Parity on the output: the checker may be stricter than PHPStan, never more lenient. |
| **A guard of one's own** | A function or method that tests a type says so with `@phpstan-assert`, `@phpstan-assert-if-true` or `@phpstan-assert-if-false` (the `@psalm-` and bare `@assert` spellings too), which the checker has honoured since M2, the exact form `=T` since 2026-10-31 (`checker.md` §9): `/** @phpstan-assert-if-true list<string> $x */ function isStringList(mixed $x): bool`. The Phaenon types in the tag are translated for PHPStan as in any `.phn` docblock (`collections.md` §4). | It is PHP's settled way to name a test, read by PHPStan, Psalm and PhpStorm alike. A syntax of Phaenon's for it would be a guard only Phaenon's tools read. |
| **Tuples** | A type only: `(int, string)` in declarations, lifted from `array{…}` by `phn migrate`, with no runtime test. A value is a tuple because the code that built it says so, as in PHP; a test written out by hand narrows as PHPStan narrows it. | The shape of a value at runtime is the programmer's code. |
| **Type arguments** | `$x instanceof Repository` tests the class; the arguments come from the operand's static type or from a `class<T>` the code holds (§11.2), as §11.4's `Foo<Bar>` row had it. | Erasure: the arguments are not there to test. |
| **Codes** | `P3011` and `P3012` are retired and reserved, never reused. A leftover `is`, `as` or `as?` expression is `P1001`, the code of a construct removed from Phaenon, as a leftover `concurrently` block is, with the PHP test in its message. `phn fmt` rewrites it into the PHP §11.4's table gives (`tooling.md` §5.6), so the diagnostic is left only where no test compiled. | One mistake, one diagnostic: the removed construct has its code. The trees migrated before 2026-11-09 hold such tests, and their migration must be mechanical. |
| **The word `is`** | Not reserved: a name `is` compiles, with PHP 8.6's deprecation reported as for `let` (`php-versions.md` §9). `as` stays PHP's, in `foreach`, `use` and trait adaptations. When a PHP release ships `$x is <pattern>`, it is Phaenon's from that release with PHP's meaning (roadmap D17). | PHP reserves the word for its own construct, which Phaenon's operator would have collided with. |

### 11.6 An alias is its namespace's, and names a shape (2026-11-15)

Roadmap D29. A shape gets syntax with a name (`collections.md` §6), and the name is a `type`
alias. A named shape that a public method returns is useless unless the caller's file can name
it, and until now an alias was seen only in the file that declared it: on the compiler of
2026-11-15 another file of the same namespace gets `P2001 unknown class App\Pair`
(`internal/program`: a file's `Scope` holds its aliases, and `FileAlias` looks nowhere else).
Built in the compiler on 2026-11-15, and in both editors the same day (`tooling.md` §6.2,
§6.6). Where this table and the text above differ, this table wins; where §11.7, which holds
what building it settled, differs from this table, §11.7 wins.

| Topic | Decision | Why |
|---|---|---|
| **An alias belongs to its namespace** | `type Row = …;` declares `App\Orders\Row`, as `class Orders` declares `App\Orders\Orders`. A name in a type position resolves as a class name does, through the file's imports and else in its namespace, never falling back to the global one, and what it finds is a class or an alias. A file of the same namespace names the alias bare; any other writes `use App\Orders\Row;` (with `as`, in a group or a comma list, as for a class) or the qualified name. A type parameter in scope and, inside a class, its own `@phpstan-type` come first, as today. Supersedes "file-level" in §11's *Type aliases* row | It is the rule a PHP developer already knows for every other name a namespace holds, and it needs no new import form: `use` names a namespace's member. A separate `use type` was weighed and not taken: a second keyword for the reader, and for the tools a second table of imports, to tell apart two things that cannot share a name anyway (next row). |
| **How an alias is found** | By its declaration, never by a file name: an alias does not reach PHP, so nothing autoloads it. *The files of the program*: a run over the project (`phn check` and `phn build` without paths, the server) reads every file of the roots before it checks any (`Program.AddFile`, `collect`), so the aliases go into a table by qualified name beside the classes, wherever they are declared; a file may hold several, beside classes or alone. *Sources outside the program*: an installed Phaenon package, read from its `.phn` (`tooling.md` §3.3, T18), and the rest of the project in a run over given paths (`phn check src/Http`). There a class is loaded when it is named, through Composer's maps (`ComposerLoader.FindClass`), and an alias has no file to be found by. So the first name that the maps give no class for, under a PSR-4 prefix whose directories hold `.phn` sources, makes the checker read the `type` declarations of those directories, once per program. *A PHP package* declares none | There is no autoload for types, and no need of one. A run over one path must find what a run over the project finds, or a file's findings would depend on how `phn` was called. The reading is paid only by a name that is not a class, which is an alias or an error, so sources without aliases cost nothing. Its cost on a large tree is not measured (§9, question 9) |
| **One name, one declaration** | A name declared as an alias and as a class, interface, enum or trait of the same namespace is an error, reported at the alias; so is an alias declared twice, in one file or two. Names are compared as class names are, without regard to case. The code is in the P2 family, with the names: either `P2511`, which already reports a class, a function or an alias "already declared", or a new P2xxx code, assigned when it is built | One table of names is what makes `use` unambiguous. Today the two do not meet: `type Thing = int;` beside `final class Thing {}` in one file is accepted and the alias wins in type positions; `type local` and `type Local` are two aliases, matched by case as PHPStan matches `@phpstan-type` names. Sharing the classes' table means sharing their rule |
| **A type, not a class** | An alias stands in type positions and in `use`, nowhere else: `new Row`, `$x instanceof Row`, `Row::class`, `catch (Row $e)`, `extends Row` and an attribute `#[Row]` are errors, `P2001` as today, with a message that says the name is a type alias. It is structural: it names a type and makes no new one | An alias is erased. PHP would look for a class of that name and find none; the checker says so before PHP does |
| **No alias mentions itself** | `type Tree = array{value: int, children: list<Tree>};` is an error, directly or through another alias: a new P2xxx code, assigned when built | The alias is expanded into every docblock that uses it (below), and a recursive one has no expansion; PHPStan refuses a circular `@phpstan-type` too. Today the inner mention silently reads as `mixed`, and the docblock says `list`. A recursive structure is a class |
| **`use` of an alias** | Counts as used when a type position or a docblock of the file names it (`P2013` otherwise). The emitter leaves it out of the output: the statement, or that item of a group or a comma list | Nothing in the compiled PHP names an alias, and an import of a name that does not exist is not PHP a person wrote |
| **Erasure, unchanged** | The `type` declaration leaves no trace; a type position gets the erased native type (`array` for a shape) and the docblock the alias expanded, its class names fully qualified; no `@phpstan-type` is emitted. Supersedes nothing: it is §11's *Type aliases* row, now also true across files | PHPStan's aliases are scoped to a class and imported class by class; an alias of a namespace has no counterpart there, so the expansion is what PHPStan would see either way. The emitter already writes an expanded alias with fully-qualified names, so the text is right in whichever file uses it. The cost is a long docblock at every use (§9, question 11) |
| **A file that holds only `type` declarations** | Compiles to an output with nothing in it: the header, `declare(strict_types=1)`, the `namespace` statement and the line map, which is what the compiler writes for such a file today. No class is declared there, so no autoloader ever includes it | Every `.phn` has its `.php` (`language-overview.md` §2), and the tools lean on that: `phn build` removes an output with no source behind it (D12), and a library's gate compares the tree with what a build writes (D28). A source without an output would be an exception in each of them, for a file PHP never loads. The output depends on nothing (D24) and is valid PHP. A team that adopts the compiled tree as its source deletes these files |
| **Across a package's boundary** | The name crosses with the sources and not with the compiled PHP. A consumer written in Phaenon reads an installed Phaenon package from its `.phn` (D13; T18 for a library that commits its outputs) and imports its aliases like its classes. Compiled PHP holds no alias: a PHP consumer, PHPStan and PhpStorm reading a library's `.src/` see the expanded shape in each docblock, and `Acme\Orders\Row` names nothing there | Erasure. A library of D28 gives its PHP consumers the shape and not the name; one of them that wants a name declares its own `@phpstan-type` over the same shape |
| **Generic aliases** | Unchanged: `type Page<T> = array{items: list<T>, total: int};`, bounds and defaults as before, the implied `K of int\|string` for a map key (`collections.md` §2). A shape may use the alias's type parameters; the shape's syntax stands in the right-hand side only, not in a bound or a default | Nothing in them depended on the file |
| **`phn migrate`** | Lifts `@phpstan-type` and `@psalm-type` into `type` declarations before the class that carried them, and `@phpstan-import-type` into `use`, or into nothing within one namespace. PHPStan's aliases are scoped to a class, so a name that two classes of a namespace declare, or that a class of the namespace bears, is not lifted: the tags stay, read as today, with a note. An inline docblock shape stays a docblock (`collections.md` §6.6). Amends §11.1's *What is lifted* and *Spelling* rows | The tags are the aliases PHP code has. The tool cannot invent a name for a shape, nor decide which of two declarations is renamed |
| **Tools** | `phn fmt` spaces a shape as it spaces type arguments (`tooling.md` §5.1). `phn lsp` goes to an alias's declaration from any file and completes aliases in type positions with their `use`, as it completes classes (`tooling.md` §6.1). The PhpStorm plugin and the VS Code grammar read a shape inside a `type` declaration, which both already take whole, and the plugin resolves an alias's name across files (`tooling.md` §6.2, §6.6) | A syntax nobody's editor knows is not shipped (`collections.md` §7) |

### 11.7 What building §11.6 settled (2026-11-15)

The compiler has aliases as §11.6 describes them, and named shapes (`collections.md` §6.9).
What the table left unsaid, or said otherwise, is here.

| Topic | Decision | Why |
|---|---|---|
| **How an alias is found, outside the program** (amends §11.6) | A name under a PSR-4 prefix whose directories have a source root (the project file's, or an installed Phaenon package's) is first loaded as a class is, from the file the maps give it: that file holds the class, or the alias of that name, or neither, and only its declarations say which. A name that is neither a class nor an alias then has the `type` declarations of the prefix's directories read, once. Of those sources only the ones that may declare an alias are parsed, which a scan of their text decides: `type`, a name, then `=` or `<`, the keyword in any letter case and with the blanks and comments the lexer passes over between the tokens *(the keyword is written in lower case since roadmap D30, but the parser still reads `TYPE Row = …;` as a declaration, to report it, so the scan keeps finding it)*. The scan may report a file that declares nothing and never misses one the parser reads a declaration in. A directory without a source root holds PHP, which declares no alias, and is never read | "The first name that the maps give no class for" was not enough: an alias kept in the file PSR-4 gives its name (`Row.phn` for `type Row`) has a file in the maps, was taken for a class to be loaded later, and was `P2001` in a run over a path while the project's run found it; and it was found after all once another alias of the directory had been read. A path's findings must not depend on how `phn` was called, nor on the order of the names. "Sources without aliases cost nothing" did not survive either: any tree names something that is no class sooner or later, and the first such name costs a walk of the directories (§9, question 9, has the numbers) |
| **On a collision the class wins** (amends *One name, one declaration*) | A name declared as an alias and as a class-like is `P2511` at the alias, the message naming the class's file and line, and in type positions the name is read as the class. An alias declared twice is `P2511` at the second, naming the first. The code is the one that already said "already declared"; no new one was needed | With the alias winning, as it did while aliases were looked up first, every `new Thing()` and `$thing->size` in every file that imports the name failed far from the mistake, and one name meant two things in one file, a type where it was written as a type and a class where it was instantiated. With the class winning there is one finding, at the declaration that has to change |
| **An alias under a name its file imports** | `use App\Models\Row;` followed by `type Row = …;` is `P2511` at the alias, the message quoting the import and its line. The name stays the import's | PHP refuses a class declared under a name its file imports, for the reason that holds here: the name would mean two things in one file. Unreported, the import wins in a type position, as for any class name, and the file's own alias is never what its own signatures mean |
| **No alias mentions itself** (the code) | `P2015`, at each alias of the cycle: `type Ping = list<Pong>;` and `type Pong = ?Ping;` are two findings | Each of the two declarations is wrong on its own, and a reader of one file should not need the other to learn it |
| **PHP names no alias** | A type written in a `.php` file, in a signature or a docblock, is never looked up as an alias: there it is a class or nothing | PHP has no aliases and would look for a class; PHPStan reads the docblock the same way. It also keeps the docblocks of the vendor code out of the lookup |
| **A `type` declaration takes its docblock with it, and takes no attribute** (amends *Erasure, unchanged*) | The docblock above a `type` declaration is removed from the output with the declaration, and so are the comments between the two (`/** … */`, `// note`, `type Row = …;`): PHP attaches a doc comment to the next declaration across other comments, so one left behind a note would still be the next class's. An attribute on one is a syntax error | "Leaves no trace" has to include the comment: left behind, it documents whatever declaration comes next, and a `@deprecated` over an alias deprecated the class after it for PHPStan and for reflection. An attribute had the same fate, landing on the next function, and an erased name has nothing that could carry one |
| **An import that only erased names use is left out of the output** (amends *`use` of an alias*) | Besides the import of an alias itself, the emitter leaves out an import whose every use is in text the output does not have: a namespace through which only aliases are named (`use App\Orders;` for `Orders\Row`), a class named only among an alias's type arguments (`Page<Customer>`, whose expansion writes the class by its qualified name), only in a `type` declaration, or only among the type arguments of a call, and a docblock mention the emitter expands away. An import the output still uses anywhere stays, in a signature, a `new`, an attribute, a docblock or a `{@see}`; so does one nothing uses at all, which is the checker's to report (`P2013`). A use is a use of a class here: an unqualified name where PHP looks for a function or a constant (`user()`, `Limit`) is none, whatever it spells, since PHP resolves those in tables of their own; and an import is its namespace block's, so a use of the name in another block of the file is none either | An unused import in the output is not PHP a person wrote, and it is not harmless: a fixer that removes unused imports takes it out of the output, `phn fix` carries the removal into the source, and the source, which does use the import, no longer compiles. The uses are found by the checker's own walk over the tokens. The emitter is stricter about what one is: the checker, deciding whether to warn, errs on the side of a use and does not tell the kinds of import apart, while the emitter has to agree with a fixer that does, or the import it keeps for a function's name is the one the fixer removes |
| **A docblock's mention of an alias** | Resolved where the docblock stands: a type parameter of the declaration it documents and the class's own `@phpstan-type` come first, then the alias, as in a signature. It is expanded in place (`collections.md` §6.9) | The emitter translates a docblock's vocabulary textually; the alias is the one word that needs the program to translate |
| **The language server** | Go to definition, from a type position or from the `use`; hover, the declaration with the type it stands for, its type parameters and its docblock's summary; completion, among the classes, with the import. Completion offers the aliases the program has read, so those of an installed package appear once something names one | The server reads the project, whose aliases it has; a package's are read on demand, as its classes are |
| **`phn migrate`** | `collections.md` §6.9 holds the rules that building the lift added to §6.6 | They are about the tags, which are shapes' more often than not |
