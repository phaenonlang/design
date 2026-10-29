# Phaenon — Language overview (draft 0.1)

Status: **draft** · 2026-10-28 · scope document; each feature gets its own design doc
(`generics.md` is the model). Companion: `roadmap.md`.

## 1. One sentence

Phaenon is PHP 8.5 with a type checker, generics, real collections, a sane standard library and
structured concurrency — compiled to the PHP you would have written by hand.

## 2. Compatibility contract

- **Every `.phn` file compiles to one `.php` file** with the same namespace, classes and functions,
  and the same observable behaviour at runtime. The compiler never changes semantics the
  programmer did not spell out.
- **PHP 8.5 is the baseline**: the output targets PHP ≥ 8.5 and uses its features (property hooks,
  asymmetric visibility, pipe operator, `clone with`) directly instead of lowering them.
- **The PHP subset round-trips**: a `.phn` file that uses only PHP syntax compiles to an
  equivalent PHP file (formatting aside). This is what the php-src `.phpt` corpus tests.
- **Mixed projects are normal**: `.phn` and `.php` files coexist in one Composer package; Phaenon
  code calls PHP code with full type information read from native types and docblocks; PHP calls
  Phaenon code without knowing (§ generics.md §4.4).
- **Runtime generics are the PHP idiom.** A generic that needs its type parameter at runtime holds
  a `class<T>` value — a constructor argument you pass, or a static getter you override in the
  subclass that Symfony wires. The compiler never adds a parameter, a member or a name the
  programmer did not write (roadmap D9, `generics.md` §11.2).

## 3. Removed from PHP

| Removed | Replacement | Why |
|---|---|---|
| `<?php` / `?>` / inline HTML | none — a `.phn` file is code from byte 0; templates are a separate file type (later, `.phn.tpl`-style) | the tag is a legacy of the template origin; it is the single most visible "PHP smell" |
| `array` as the one collection type | `list<T>` and `map<K, V>` (M4); `array` stays a **legacy type** for interop with PHP APIs, implicitly convertible both ways under checker rules | the dual nature is the main source of typing pain |
| Haystack/needle inconsistency of the stdlib | `\Phaenon\*` stdlib with predictable argument order; PHP functions remain callable | every PHP developer has typed `in_array($needle, $haystack)` wrong |
| `mbstring`-vs-byte duality | `string` is text (UTF-8); `bytes` is binary; one API | unicode-native by default |
| Untyped code paths | every declaration has a type; `mixed` must be written, never inferred silently; `declare(strict_types=1)` is implied and emitted | the checker is the product |
| `global`, `goto`, `extract`, `compact`, `$$var`, `${expr}` string interpolation, `eval` | nothing | dynamic scope defeats a type checker; all are already deprecated in spirit or in fact |
| Alternative syntax (`if: … endif;`) | braces | one syntax |
| `var`, `(integer)`/`(boolean)`/`(double)` casts, `list()` | `public`, canonical casts, `[…]` destructuring | already deprecated or redundant |

Nothing is removed from the *runtime* — these are things the Phaenon parser refuses; the
emitted PHP never contains them.

## 4. Kept from PHP (deliberately)

`$variables` · classes, interfaces, traits, enums, readonly, property hooks, asymmetric visibility ·
attributes · closures and arrow functions · `match` · named arguments · union/intersection/DNF
types · class names as strings (`Foo::class`, typed `class<T>`) · exceptions · generators ·
Fibers (as the concurrency substrate) · the request-cycle execution model · Composer · Symfony ·
PHPStan vocabulary (read from docblocks, not written as syntax).

## 5. Added

| Feature | Doc | Milestone |
|---|---|---|
| Type checker: inference, narrowing, flow analysis, diagnostics | `checker.md` (pending) | M2 |
| Generics: `<T>`, bounds, variance, erased; `class<T>` as the type of `T::class`; `is`/`as` | `generics.md` | M3 |
| `type` aliases | `generics.md` §2.1 | M3 |
| `list<T>`, `map<K, V>`, literals, `array` as legacy type, `\Phaenon\Lists` / `Maps` | `collections.md` (draft 2026-10-29) | M4 |
| String/`bytes` split, method syntax on primitives (`$s->upper()` → `\Phaenon\Strings::upper($s)`) | `strings.md` (draft 2026-10-29) | M4 |
| `\Phaenon\*` stdlib (PHP package `phaenon/runtime`) | `collections.md` §3, `strings.md` §4 | M4 |
| Structured concurrency over Fibers, no function colouring | `concurrency.md` (pending) | M5 |
| Tooling: `phn build/check/watch/fmt/migrate/lsp` | `tooling.md` (pending) | M1, M6 |

## 6. Files and build

- Extension `.phn`; one file = one compilation unit = one emitted `.php` beside it (gitignored),
  the way `.js` sits beside `.ts`. Composer autoloads the `.php`.
- `phn build` compiles a package; `phn check` type-checks without emitting; `phn watch` does both
  on change. Type errors mean **no output** for that file.
- Emitted files start with a header comment naming the source and the compiler version, and
  `declare(strict_types=1);`.

## 7. Non-goals

- A new runtime (phase 3 at most).
- Changing PHP's evaluation order, copy-on-write value semantics, reference semantics, integer or
  float behaviour. Phaenon is a *dialect*, not a different language with PHP syntax.
- Templating in v1.
