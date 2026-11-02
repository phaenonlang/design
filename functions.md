# Phaenon — Functions design (draft 0.1)

Status: **draft** · 2026-11-01, extended 2026-11-02 · covers the spelling of declarations,
closures where PHP only allows constant expressions, expression bodies, `throws`, and the layout
of a declaration. Companion: `language-overview.md` §3–5, `roadmap.md` D15, D18.

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
- **A class constant without visibility is public too**: `const int LIMIT = 10` compiles to
  `public const int LIMIT = 10`, and `phn migrate` drops the `public`. `private const`,
  `protected const` and `final const` are written as in PHP.
- **The same rule holds for asymmetric visibility**: `private(set) int $x` and a promoted
  `private(set) string $id` are public, as PHP 8.4 already allows when the main visibility is
  omitted; the emitter writes `public private(set)` so the output satisfies PSR-12 tooling, and
  `phn migrate` drops the `public`.
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

## 3. Expression bodies

### 3.1 Spelling

```php
fun load(Id $id): T => $this->get($id) ?? throw new AggregateNotFound(static::aggregateClass(), $id);
fun name(): string => $this->name;
fun log(string $message): void => $this->logger->info($message);
fun fail(string $why): never => throw new LogicException($why);
static fun of(string $s): self => new self(trim($s));
```

- A named function or method — `fun` or `function` — may have **one expression** for its body:
  `=> expr;` takes the place of the block, after the return type and the `throws` clause (§4).
- The expression is **returned**, unless the return type is `void` or `never`: then it is only
  evaluated. A `never` body is a `throw` or a call that never returns, as in a block.
- **Not allowed** on constructors and destructors, whose rules on returned values are PHP's own
  (8.6 deprecates returning one), and in generators: `yield` in the expression is an error. An
  abstract or interface method has no body, so no arrow either.
- A long expression wraps **after the arrow**, the arrow starting the continuation line, one
  indentation level in:

  ```php
  fun load(Id $id): T throws AggregateNotFound
      => $this->get($id) ?? throw new AggregateNotFound(static::aggregateClass(), $id);
  ```

### 3.2 Compilation

The arrow becomes `{ return`, the final `;` becomes `; }` (`{` and `; }` alone for `void` and
`never`), on the lines they stand on: line N of the output is line N of the source.

```php
public function load(AggregateId $id): EventSourcedAggregate { return $this->get($id) ?? throw new AggregateNotFound(static::aggregateClass(), $id); }
```

The output puts the opening brace on the declaration's line, which PSR-12's brace rule
rejects, so compiled output needs that php-cs-fixer rule off, beside the line-1 rules of D12
(§5.2).

### 3.3 Why

Accessors, delegations and guards — the one-statement methods that dominate domain code — take
four lines and two braces to say one expression. Kotlin (`= expr`), C# and Scala (`=> expr`)
write them as what they are. PHP's own Short Functions RFC (2021) proposed this exact syntax and
was declined as a second way to write a function; Phaenon's answer differs because it already
has one (`fn`) and because its arrow is PHP's arrow: nothing about it is new to a reader.

## 4. `throws`

### 4.1 Spelling

```php
fun load(Id $id): T throws AggregateNotFound
fun commit(): void throws ConcurrencyConflict|StoreUnavailable
abstract fun handle(Command $c): Result throws Refusal;
```

- `throws` follows the return type (or the parameter list when there is none) and names the
  exceptions the function lets escape: a class, or a union `A|B` written as in PHP's
  `catch (A|B $e)`. Each member is a `Throwable`, or a type parameter bounded by one.
- It means what a `@throws` tag means today, and no more: documentation the checker verifies
  (each member exists and is a `Throwable`). It is **not** Java's checked exceptions: a caller
  is not required to catch or redeclare. The throw analysis on the open list (vault
  `open-items`) will read the clause when it comes.
- `throws` is a **contextual keyword**, a keyword only in that position.
- Named functions and methods only for now; a closure keeps a docblock.

### 4.2 Compilation

The emitter writes the clause as a `@throws` tag: into the declaration's docblock when there is
one, otherwise as `/** @throws A|B */` at the start of the declaration's first line, before the
modifiers, which PHP, PHPStan and PhpStorm attach to the declaration as they would on the line
above. The line count holds.

### 4.3 Migration

`phn migrate` lifts the `@throws` tags without a description into the clause, several tags into
one union; a tag with a description stays in the docblock, as a typeless `@param` keeps its
prose (D14).

### 4.4 Why

Types live in the signature (D14): the parameter and return types moved out of the docblock
with `fun`, and an exception type is a type. A bare `@throws X` line without the docblock was
considered and rejected: `@` is PHP's error-suppression operator, so inside a body the line
would be ambiguous, and it would be a third comment syntax that is neither PHP nor a
declaration, invisible to every tool but Phaenon's.

## 5. Layout

### 5.1 The canonical style

- **The opening brace of a function or method stands on the declaration's line**, as it does
  for control structures in PSR-12 already, and in Kotlin, Swift, Go, Rust, TypeScript and Java.
- **The opening brace of a class, interface, trait or enum keeps its own line** (2026-11-02,
  `line-map.md` §7): a class costs one brace line per file, and its header — generic parameters
  with bounds, an `implements` list with type arguments — is the file's longest line, where a
  trailing brace is hard to see.
- **An empty body is `{}` on the declaration's line** (2026-11-02), as PER-CS 2.0 writes it: a
  function or method without statements (`fun __construct(private EventStoreFactory $factory) {}`,
  the constructor that only promotes its parameters being the common case) and a class-like without
  members (`final class AggregateNotFound extends \DomainException {}`). A class's own-line brace
  separates its header from its members; an empty class has none, so the line is the whole
  declaration. A body with a comment inside keeps its lines.
- **A constructor keeps its body** — `fun __construct(private Foo $foo);` was considered and
  rejected: in PHP a method without a body is abstract or an interface's, which is the opposite of
  a constructor that does its work by promoting parameters (PHP forbids promotion in an abstract
  constructor for that reason); it would save two characters over `{}`, at the price of syntax of
  its own in the parser, checker, migrate and the plugin. TypeScript keeps `constructor(private x:
  X) {}` for the same case.
- **A docblock with one line of content is one line**: `/** The aggregate, or null. */`.
- This is a style, not syntax: the PSR-12 placement stays valid Phaenon. The style is what the
  PhpStorm plugin's code style for `.phn`, `phn migrate` and later `phn fmt` (M6) write.

### 5.2 Compilation

Until 2026-11-02 the emitter kept the source's layout (the line rule), so compiled output carried
the style and needed php-cs-fixer's brace rules off. Since D19 (`line-map.md` §2) the emitter puts
the braces of named functions, methods and class-likes where PSR-12 wants them and expands an
expression body into a PSR-12 body; the line map keeps every position pointing into the `.phn`.

### 5.3 Migration

`phn migrate` moves the opening brace of a function or method up to its line and a class-like's
back down to its own line where an earlier run moved it up, writes an empty body (a function or method
without statements, a class-like without members, no comment inside) as `{}`, collapses a docblock with
one line of content, and turns a body that is a single `return expr;` (or a single expression
statement under `void`) into `=> expr;`, wrapped after the arrow when the line would pass 120
columns. A body with a comment inside stays a block. `--keep-layout` keeps braces, docblocks and
bodies as they are. Re-running is a no-op.

### 5.4 Why

PSR-12's brace on its own line for classes and methods is an inheritance from PEAR, not a
reading aid: PSR-12 itself puts control-structure braces on the line. With `fun`, `=>` bodies and
`throws`, a typical accessor shrinks from six lines to one; the brace rule decides whether the
remaining blocks keep paying a line each. The cost — a php-cs-fixer exception on output and a
one-time reformatting of every migrated tree (a `.git-blame-ignore-revs` entry keeps `git blame`
useful) — is paid once. The php-cs-fixer exception went with the line rule (D19).

## 6. Editor

PhpStorm reads `fun` as the `function` keyword (the plugin's lexer hands its parser a
`function` token with the text `fun`); a method without visibility is public to PhpStorm as to
PHP, and its "missing visibility" inspection is filtered on `.phn` files for methods, constants
and `private(set)` members alike. Completion offers
`fun` where a member or statement may start. A closure in an attribute is checked against the
PHP 8.5 rule by the compiler, not the editor.

The plugin reads `throws` and its names as an erased construct (navigable, counted by Optimize
Imports), hands PhpStorm's parser a block for an expression body, and formats `.phn` with the
canonical style of §5.

## 7. Decisions

| # | Decision | Why |
|---|---|---|
| F1 | `fun` declares named functions and methods; absent visibility is `public`, on constants and `private(set)` members as well; `function` stays valid | §1.4 |
| F2 | `fun` is contextual, never reserved | §1.1 |
| F3 | Closures in constant expressions are written as `fn` / `function` and lowered to `static function`; `$this` and `use` are `P2531` | §2 |
| F4 | `phn migrate` writes `fun` and drops a sole `public`, `--keep-public` keeps it | §1.3 |
| F5 | A named function or method may have an expression body, `=> expr;`; returned unless `void` / `never`; not on constructors, destructors or generators | §3 |
| F6 | `throws A\|B` in the signature, checked to name `Throwable`s, compiled to `@throws`; not checked exceptions | §4 |
| F7 | No bare `@tag` lines: a one-line docblock, or the signature | §4.4 |
| F8 | Canonical layout: opening braces of declarations on their line, one-line docblocks; a style, not syntax; output needs php-cs-fixer's brace rule off | §5 |
| F9 | `phn migrate` lifts bare `@throws`, moves braces up, collapses one-line docblocks, writes `=>` for single-expression bodies; `--keep-layout` keeps the layout | §4.3, §5.3 |

## 8. Decisions made while building D18

| Topic | Decision | Why |
|---|---|---|
| Where `throws` is recognised (2026-11-02) | By the parser, after the parameter list or the return type of a named function or method; the lexer leaves it a `T_STRING` | PHP's grammar admits only `{` or `;` there, so the position is free. A lexer rule ("`throws` before a name") cannot tell the clause from the names PHP code gives things: `class throws extends …`, `throws instanceof X`, `foreach (throws as …)`, a trait method `throws as protected` |
| What an expression body is in the tree (2026-11-02) | The declaration keeps the arrow, and its body holds the statement the arrow stands for: a `return`, or an expression statement when the written return type is `void` or `never` | "Typed exactly as a `return` would be" then holds by construction: every pass that reads a body reads a block of one statement, and only the printer and the emitter look at the arrow. `void` and `never` are keywords, so the written type decides |
| Codes (2026-11-02) | `P1004` (syntax family, error) for an expression body the design refuses: on a constructor or a destructor, or one that yields. An arrow on an abstract or interface method is a body and keeps the codes any body gets there, `P2504` / `P2514`. `P2319` for a `throws` member that is not a `Throwable` (an unbounded type parameter included), beside `P2317` / `P2318` for `throw` and `catch`; an unknown class is `P2001` | One mistake, one diagnostic: what the grammar admits and the design forbids is the syntax family's (`P1003` is the same shape); a non-`Throwable` is the mistake the `throw` and `catch` codes already name, in a third position. A constructor's arrow is checked as a statement, so no `P1101` for a returned value follows it |
| `throws` and the target (2026-11-02) | A class newer than the target is not reported in a `throws` clause | The clause compiles to a docblock and fails nothing on the older PHP, as `catch (X)` does (php-versions.md §9) |
| PHPStan's `throws.unusedType` (2026-11-02) | Allow-listed in the checker's parity test, beside `catch.neverThrown` | PHPStan 2 reports at level max a `@throws` type the body never throws. The clause is documentation the checker verifies, not checked against the body (§4.1); the throw analysis on the open list will read it |
| Merging the tag without a line (2026-11-02) | No docblock: `/** @throws A\|B */ ` before the attributes and modifiers. A docblock whose `*/` stands alone: the tag takes that line, ` * @throws A\|B */`. A one-line docblock, or content beside the `*/`: the `*/` moves to the next line with the tag, ahead of what stands there (`/** Summary.` over ` * @throws A\|B */ public function f()`), or onto a blank line below. PHP's reflection and PHPStan read every form | §4.2 asks for the docblock and an unchanged line count; one tag per line is all PHPStan reads, so the tag needs a line of its own, and these are the lines a docblock already has. Writing a second docblock would hide the first from reflection, where PHPUnit and attribute-less libraries read annotations |
| A docblock below the attributes (2026-11-02) | `#[A] /** … */ fun f() throws X` gets `/** @throws X */ ` before the attributes, its own docblock left as written; generated generic tags go there too | PHPStan reads the comment before a declaration's attributes, PHP's reflection the one before the keyword: the tag goes where the tool that reads it looks |
| The one new exception to the line rule (2026-11-02) | A docblock on the declaration's own line (`/** Summary. */ fun f() throws X`) gains one line; when the generics lowering writes the declaration's docblock, which adds lines already, the tag joins it as one more line | Such a docblock has no next line to borrow. It is rare in source, and the canonical layout puts docblocks above their declaration |
| The long form's output (2026-11-02) | An arrow on a continuation line hands its brace to the end of the signature's line and becomes `return` (vanishes under `void` / `never`); the short form becomes `{ return expr; }` where the arrow stood | With the brace on the arrow's line, a multi-line expression's continuation lines stood one level deeper than a block's statement, and php-cs-fixer's `statement_indentation` and `method_argument_space` fired on tequila/core and tq-cli. Moving the brace up adds no line and leaves only the brace's placement to excuse |
| php-cs-fixer on compiled output (2026-11-02) | Beside D12's four line-1 rules: `braces_position` (functions, methods, closures, and the one-line blocks of expression bodies) and `class_definition` (it puts a class-like's brace on its own line). Measured on the re-laid-out core and tq-cli with their own configurations: nothing else fires, D11's `single_line_after_imports` aside where an output gets a compiler import | §5.2's "brace-placement rule" is two rules in php-cs-fixer 3.95; no docblock rule is in `@PSR12`, so the merged `@throws` forms pass |
| `throws` lifting and `--keep-layout` (2026-11-02) | Lifting `@throws` is part of the lift (`--keep-docblocks` skips it), not of the layout | It moves a type into the signature as the generic lifts do (D14); `--keep-layout` keeps braces, docblocks' shape and bodies |
| `never` bodies in migrate (2026-11-02) | A body that is one expression statement under `never` (`throw new X(…);`) becomes `=> …;` as under `void` | §3.1 evaluates the expression under both; §5.3 named `void` only |
| Limits of the layout (2026-11-02) | A docblock is not collapsed when the one line would pass 120 columns; a brace moves up whatever the width; a comment between the signature and the brace keeps the block as a comment inside does; an empty body keeps its closing brace on its own line (`) {` / `}`). A multi-line expression joining the signature's line moves its continuation lines one level out; one holding a multi-line string or heredoc, whose text that would change, wraps after the arrow instead | A collapsed docblock only saves lines; a brace has no other place short of reflowing the signature, which is `phn fmt`'s work (M6). A comment there would have nowhere to go |
| `throws` members (2026-11-02) | Class names, `self`, `static` and `parent`, and type parameters; no type arguments, no keyword types. `@throws void` and a tag naming a template left in the docblock stay in the docblock with a note | What `catch` accepts, plus the type parameters §4.1 allows; PHPStan's `@throws void` ("throws nothing") has no clause spelling |
