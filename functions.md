# Phaenon — Functions design (draft 0.1)

Status: **draft** · 2026-11-01 · covers the spelling of declarations and closures where PHP
only allows constant expressions. Companion: `language-overview.md` §3–5, `roadmap.md` D15.

Phaenon's additions so far were types; this document adds two pieces of syntax that are not
types. Both compile to PHP a person would write, and plain PHP stays valid Phaenon (the
compatibility contract, `language-overview.md` §2).

---

## 1. `fun` — declaring a function or method

### 1.1 Spelling

```php
fun id(): T                       // public function id(): T
protected fun apply(): void       // protected function apply(): void
static fun create(): static       // public static function create(): static
final fun version(): int          // final public function version(): int
abstract protected fun run(): void
fun helper(int $x): int           // function helper(int $x): int, outside a class
```

- `fun` declares a named function or method. The modifiers PHP allows before `function` —
  `abstract`, `final`, `static`, `public`, `protected`, `private` — are allowed before `fun`.
- **No visibility means `public`.** That is PHP's own rule for a method written without one;
  `fun` makes it the normal spelling. `public fun` is allowed and means the same.
- `function` remains valid everywhere it is today, and is the only spelling of a closure:
  `function () {}` and `fn () => …` are unchanged. `fun` is for declarations with a name.
- `fun` is a **contextual keyword**: it is a keyword when the next token is a name (or `&` and
  a name) and an identifier otherwise, so a PHP project with a function or constant called `fun`
  keeps parsing. PHP's reservation of `fn` in 7.4 broke code; this does not.

### 1.2 Compilation

The emitter replaces the declaration's head — the modifiers and the keyword, on one line — with
PHP in PSR-12 order: `[abstract|final] [visibility] [static] function`, `public` written out.
Nothing else moves; line N of the output is line N of the source (`emit` invariant). When the
modifiers span several lines the keyword alone is replaced, so the line count still holds.

### 1.3 Migration

`phn migrate` rewrites every named `function` to `fun` and drops a sole `public` on the way:
`public function` → `fun`, `final public static function` → `final static fun`,
`protected function` → `protected fun`. `--keep-public` keeps `public fun`. Re-running on a
`.phn` is a no-op: a declaration already spelled with `fun` is left alone.

### 1.4 Why

- Methods are the most frequent declaration in the code Phaenon exists for (Tequila's `core`
  has 870 `public function` to 59 `protected function`), and the two words say nothing the
  reader did not assume.
- The implicit `public` is not a new rule but PHP's, made the default spelling; the emitter
  writes it out, so the PHP output stays PSR-12 and PhpStorm's "missing visibility" never fires
  on generated code.
- `fun` reads immediately to anyone who has seen Kotlin; `fn` would collide with PHP's arrow
  functions; `func` and `def` were considered and are no better once `fn` is excluded.
- Keeping `function` valid is not a second way to write the same thing by choice but the
  compatibility contract: every PHP file is a Phaenon file, and `phn migrate` moves it over.

## 2. Closures in constant expressions

### 2.1 What PHP allows

PHP 8.5 ("closures in constant expressions") admits closures in attribute arguments, constant
initializers, property and parameter defaults, enum case values and `static` variable
initializers — but only `static function` closures and first-class callables. Verified on
PHP 8.5.3:

```php
#[Map(static function (int $x): int { return $x * 2; })]   // works
#[Map(strlen(...))]                                        // works
#[Map(static fn (int $x): int => $x * 2)]                  // Constant expression contains invalid operations
#[Map(fn (int $x): int => $x * 2)]                         // the same
```

Arrow functions are excluded because they capture the enclosing scope implicitly; at class
level there is no scope to capture, and the exclusion is a rule of the syntax, not of the
semantics.

### 2.2 Phaenon

In a constant expression a closure may be written as PHP writes it anywhere else:

```php
#[Map(fn (int $x): int => $x * 2)]
final class Doubled {}

final class Config
{
    const Closure DEFAULT = fn (string $s): string => \strtoupper($s);
    public function __construct(private Closure $normalise = fn (string $s): string => \trim($s)) {}
}
```

- `fn (params): R => expr` compiles to `static function (params): R { return expr; }`.
- `function (params): R { … }` without `static` compiles with `static` added.
- The closure may not use `$this` or a `use (…)` clause: PHP rejects both in a constant
  expression (a closure there is static and captures nothing). The checker reports `P2531`
  (closure in a constant expression captures its scope) where PHP would fail at compile time;
  a variable the closure does not declare is already `P2101` undefined, since the enclosing
  scope has none.
- Line preservation: `fn` becomes `static function` and `=>` becomes `{ return` on their own
  lines; `; }` is appended after the body's last token.

### 2.3 Why

Attribute-driven libraries — `tequila/object-mapper`'s mapping attributes — are where a small
inline callback is most wanted and where PHP makes it the ugliest. The lowering writes exactly
what the programmer would have written by hand after reading the RFC, so the output is PHP of
the kind D9 asks for, and the checker's rule is PHP's rule stated before the file runs.

## 3. Editor

PhpStorm reads `fun` as the `function` keyword (the plugin's lexer hands its parser a
`function` token with the text `fun`); a method without visibility is public to PhpStorm as to
PHP, and its "missing visibility" inspection is filtered on `.phn` files. Completion offers
`fun` where a member or statement may start. A closure in an attribute is checked against the
PHP 8.5 rule by the compiler, not the editor.

## 4. Decisions

| # | Decision | Why |
|---|---|---|
| F1 | `fun` declares named functions and methods; absent visibility is `public`; `function` stays valid | §1.4 |
| F2 | `fun` is contextual, never reserved | §1.1 |
| F3 | Closures in constant expressions are written as `fn` / `function` and lowered to `static function`; `$this` and `use` are `P2531` | §2 |
| F4 | `phn migrate` writes `fun` and drops a sole `public`, `--keep-public` keeps it | §1.3 |
