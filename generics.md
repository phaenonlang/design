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
   programmer asked for it, at a cost the programmer can see.
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

$id = fn<T>(T $x): T => $x;                              // generic closure (erased only)

type UserMap = map<Uuid, User>;                          // type alias
type Result<T, E of Throwable = RuntimeException> = Ok<T> | Err<E>;
```

Vocabulary choices and why:

| Choice | Alternatives | Why this one |
|---|---|---|
| `T of Bound` | `T extends Bound` (TS/Java), `T as Bound` (Hack), `T: Bound` (Rust) | It is literally what PHPStan docblocks write (`@template T of Foo`) — every PHP developer already reads it, and the emitted docblock is the same text. |
| `in T` / `out T` | `+T`/`-T` (Scala), `@template-covariant` | Declaration-site variance like C#/Kotlin/Hack; `out` = produces T (covariant), `in` = consumes T (contravariant). Maps 1:1 onto `@template-covariant` / `@template-contravariant`. |
| `type X = …` | `alias`, `typedef` | TS/Hack/Rust convention; `type` is not a reserved word in PHP, so it's a contextual keyword at statement start (same trick as `enum` in PHP 8.1). |

### 2.2 Type positions

Anywhere PHP accepts a type, Phaenon accepts a generic type: parameters, returns, properties,
property hooks, `static`/`self`, union and intersection members, nullable:

```php
private readonly map<string, list<Event<T>>> $streams;
public function get(Uuid $id): ?T;
public function all(): iterable<T>;
public function with(Closure(T, int): bool $pred): static;
public function __construct(private readonly class<T> $aggregateClass) {}  // the type of T::class (§11.2)
```

PHP's grammar never puts an *expression* in a type position, so `<`/`>` here are unambiguous.

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
3. **From the bound / default** when nothing else constrains T.
4. Otherwise: for erased T, infer `mixed` with warning `P2xxx: type argument not inferable`;
   for reified T, **error** — an explicit argument is required (a runtime value must exist).

Closure arguments are checked *after* the other arguments are unified (TS's "deferred" closures),
so `$users->map(fn($u) => $u->name)` knows `$u: User` without annotation.

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

### 3.6 Erased type parameters at runtime — the rule

Inside a body, an **erased** `T` can be used as a type (declarations, casts for the checker) but not
as a **value**: `new T`, `T::class`, `$x instanceof T`, `$x is T`, `T::staticMethod()` are compile
errors with a fix-it: "declare `T` as `reified`, or test against the bound". This is what makes
erasure honest instead of surprising.

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
| `class-string<T>` | `string` | `class-string<T>` |
| `static` / `self` / `$this` | same | `static` … |
| type alias | the alias expanded | `@phpstan-type` emitted once + used by name |

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
5. **Type aliases with reified use.** `type Ids = list<Uuid>; $x is Ids` expands fine; aliases
   containing reified parameters of an enclosing class — allowed or not?
6. **Should `is` on a container be deep by default?** Deep is correct and the cost is explicit;
   Hack chose shallow for `is` on generics (`vec<_>`) because deep was too costly in practice.
   Possible compromise: `is list<int>` deep, `is list<_>` shallow (wildcard = don't check elements).
7. **Monomorphisation as an opt-in optimisation** for closed instantiation sets (e.g. a final
   reified class instantiated only from Phaenon code) — same semantics, no hidden parameter.
8. **Diagnostics numbering and messages** — define the `P1xxx` (syntax) / `P2xxx` (inference) /
   `P3xxx` (erasure misuse) / `P4xxx` (interop) families when the checker exists.

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
| `array-key`, shapes | Not native syntax in M3. A bound needs `int\|string`; a shape (`array{a: int}`) can only come from a docblock or an alias body — and it cannot be tested at runtime. | Hyphenated names and `{}` in types are an M4 question (strings/collections). |
| Type aliases | `type X = …;` is file-level, namespace-scoped, generic (`type Result<T, E = \Throwable> = …`), expanded inline in emitted docblocks (no `@phpstan-type`). | PHPStan has no file-level aliases; inline expansion is what PHPStan sees either way. |
| `Closure` | Written as `\Closure` in emitted signatures and docblocks. | The checker accepts a bare `Closure` everywhere; PHP in a namespace would resolve it to `App\Closure`. |
| `is` | Native where PHP has a check — `instanceof`, `is_int()`, `=== null`, `?Foo` and `A\|B` on a simple operand — otherwise `\Phaenon\Type::…->check()`. Deep by default (open question 6: decided). | Zero-cost when PHP can do it; the runtime cost is visible where it exists. |
| `as` / `as?` | Always the runtime: `->assert()` / `->orNull()`. | One precise `TypeError` message for every shape; the dependency is already there for structural `is`. |
| `is !T` | Dropped; write `!($x is T)`. | Nothing a negation prefix adds. |
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
| Docblocks in `.phn` | Passed through untouched; a `map<…>` written in a `@var` in Phaenon source reaches PHPStan as is. Types belong in the syntax in `.phn` files. | Rewriting user prose is a later migration concern. |

### 11.1 Migrating docblock generics (`phn migrate`, 2026-10-28)

| Decision | Choice | Why |
|---|---|---|
| What is lifted | `@template[-covariant\|-contravariant] T [of B] [= D]` → `<[out\|in] T of B = D>` on classes, interfaces, traits, functions, methods (constructors included); `@extends` / `@implements` → arguments on the matching clause; `@param`, `@return`, property `@var` → the declared type. `@phpstan-*` / `@psalm-*` variants too. | These are the tags that *are* generics in PHPStan code. `@var` on statements, `@phpstan-type`, `@method`, `@mixin` stay (no Phaenon construct for them yet). |
| Safety rule | A tag is lifted only when (1) its type has a Phaenon spelling and (2) it **erases to exactly the declared native type**; an untyped parameter accepts only a type that erases to `mixed`; an untyped return is never refined. Everything else stays, with a note on stderr saying why. | The emitted PHP must keep the signature PHP saw before: migrating must not change what overrides, callers or reflection see. |
| Spelling | `T[]` → `array<T>`; `array<K, V>` → `map<K, V>` (`non-empty-array` → `non-empty-map`); `X\|null` → `?X`; `$this` → `static`; `\Closure(A): R` → `Closure(A): R`; class names resolved through `use` so `\App\Row` and an imported `Row` match. Not spellable: `callable(…)` signatures, shapes, literals, `int<a, b>`, conditional types, optional/variadic closure parameters. | What a human would write in `.phn`. |
| Hyphenated names | `class-string<T>`, `non-empty-string`, `array-key`, `positive-int`, `key-of<T>`, … are native type syntax: the parser joins `name-name` written without spaces in a type position; `A - B` stays arithmetic. They erase through the resolved type (`string`, `int\|string`, …). | `class-string<T>` is the generic type Tequila uses most; without it the lift would skip half of reflector. |
| Prose-only `@param` | A lifted `@param T $x prose` becomes `@param $x prose` (the type is in the signature); the emitter fills the type back in on output, and the docblock reader accepts the form. `@return T prose` and `@var T prose` keep their type too (no unambiguous type-less form). | Like TypeScript's JSDoc: prose stays, types are not repeated. |
| Emptied docblocks | Removed when only the frame is left; blank `*` lines at the edges collapse; generated tags are inserted **before** the first remaining tag, so `@throws` keeps its place after `@return`. | Round-tripping a PHPStan file through migrate + build gives back the same docblock order. |
| Inheritance | A child method without docblock types inherits the parent's **effective** type when its own native type equals the parent's erasure (`string $x` under `class-string<T> $x`), through `@extends`/`implements` substitution. | PHPStan's implicit `@inheritDoc` applies to the lifted syntax as it did to docblocks; `CachedReflector::reflect(string)` keeps `class-string<ClassType>`. |
| Promoted properties | A method-level `@template` that a promoted parameter's type mentions stays in the docblock (with its `@param`), noted as "make it a class type parameter". | A property cannot carry a method type parameter; whether the class should become generic is a design decision per class, not a mechanical one. |
| CLI | `phn migrate` takes `.php` (strip + lift) and `.phn` (lift only, rewritten in place when changed); `--keep-docblocks` is the mechanical v0. Notes print as `file:line: kept @tag …`. | One command for new and already-migrated projects. |

Still open after M3: `--runtime-guards` (§6), monomorphisation (§7), hoisting descriptors out
of hot paths (`static $__t ??= …`), and a `P3010`-style warning for an erased parameter that
nothing constrains (§3.4 item 4) — PHPStan is silent there, so the checker is too for now. The
Symfony DI warning of §5.4 went with `reified` (§11.2).

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
| **`\Phaenon\Type` narrowed** | The runtime serves structural `is` / `as` only (`$m is list<int>`), where PHP has no check of its own. It never appears in a signature. `Type::from()` and `Type::instantiate()` are removed. | What is left is an honest library call at the site that asked for a deep check; nothing hidden. |
| **Emitter principle** | Roadmap D9: the emitted PHP contains no identifier the programmer did not write — no hidden parameters, no synthesised members, no renamed locals. The compiler adds only the header line, `declare(strict_types=1)`, docblocks, and `phaenon/runtime` calls where the source used `is` / `as` on a type PHP cannot test. | The output must read as PHP a person wrote; "generated-looking" PHP would be the complaint that follows Phaenon forever. |
