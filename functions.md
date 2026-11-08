# Phaenon — Functions design (draft 0.1)

Status: **draft** · 2026-11-01, extended 2026-11-02, §1 rewritten 2026-11-09 · covers the
spelling of declarations, closures where PHP only allows constant expressions, expression bodies,
`throws`, and the layout of a declaration. Companion: `language-overview.md` §3–5, `roadmap.md`
D15, D18, D27.

Phaenon's additions so far were types; this document adds two pieces of syntax that are not
types. Both compile to PHP a person would write, and plain PHP stays valid Phaenon (the
compatibility contract, `language-overview.md` §2).

---

## 1. Declaring a method or a function

*Rewritten 2026-11-09 for roadmap D27. From 2026-11-01 until then a named function or method was
declared with `fun` (D15; F1, F2 and F4 in §7); §1.5 says what it was and why it went.*

### 1.1 Spelling

```php
abstract class Report
{
    __construct(private Clock $clock) {}               // public function __construct(…) {}

    title(): string => 'Report';                       // public function title(): string
    protected static slug(string $title): string => strtolower($title);
    final printedAt(): DateTimeImmutable => $this->clock->now();  // final public function …
    first<T>(list<T> $rows): ?T => $rows[0] ?? null;   // public function first(array $rows): mixed
    abstract protected body(): string;                 // abstract protected function body(): string
    public function static(): string => static::class; // named after a modifier: PHP's `function`
}

// a function keeps `function`
function headline(Report $report): string => strtoupper($report->title());
```

- **A method is a name and its parameters.** A declaration at class-member level — in the body
  of a class, interface, trait or enum, anonymous classes included — is a method, written
  `[attributes] [modifiers] [&]name[<type parameters>](parameters)[: type] [throws A|B]` and
  followed by its body: a block, an expression body `=> expr;` (§3), or `;` for an abstract or
  interface method. `&` before the name returns by reference, as `function &name()` does.
- **How the parser tells it from a property.** After the attributes and modifiers, a name
  followed by `(`, or by a balanced `<…>` and then `(`, is a method; anything else begins a
  property's type (`Foo $x`, `?Foo $x`, `list<int> $xs`, `(A&B)|null $x`). A constant starts
  with `const` and its type or name; a case and a trait use start with `case` or `use` followed
  by a name, so `case(` and `use(` are methods. Apart from the modifiers, `var` and `const`, no
  PHP class member starts with a name followed by `(`, so the parse is deterministic, with a
  lookahead of one token after the name but for a generic method's `<…>`.
- **No visibility means `public`.** That is PHP's own rule for a method written without one,
  made the normal spelling; `public` may be written and means the same. The emitter writes it
  out (§1.2).
- **A class constant without visibility is public too**: `const int LIMIT = 10` compiles to
  `public const int LIMIT = 10`, and `phn migrate` drops the `public`. `private const`,
  `protected const` and `final const` are written as in PHP.
- **The same rule holds for asymmetric visibility**: `private(set) int $x` and a promoted
  `private(set) string $id` are public, as PHP 8.4 already allows when the main visibility is
  omitted; the emitter writes `public private(set)` so the output satisfies PHP's style tooling
  (php-cs-fixer's `modifier_keywords`), and
  `phn migrate` drops the `public`.
- **PHP's `function` stays valid on every method**, since every PHP file is a Phaenon file, and
  is required where the bare name would need a longer lookahead or would mislead a reader:
  - a name that is a modifier, `static`, `abstract`, `final`, `public`, `protected`, `private`
    or `readonly`, or `var`: `public static (A&B)|null $x` and `var (A&B)|null $x` are
    properties with a DNF type, so `public static (` is not read as a method named `static`.
    A scan to the closing parenthesis could tell them apart (a parameter list is empty or holds
    a `$`), but the lookahead stays one token, and the declaration stays readable. `var` is a
    construct removed from Phaenon (`P1001`), which the parser must still read to report it;
  - `const`: `const (A&B)|null C = null;` is a constant with a DNF type;
  - `Closure`: `Closure(int): string $f` is a property with a closure type (`generics.md` §2.2);
    only the unqualified `Closure` is read so, not `closure()`;
  - `function` and `fn`, the two names whose bare form is valid PHP of another meaning:
    `fn(int $x): int => $x;` is an arrow function, and `function(): int { … };` a closure.

  Every other name needs no keyword, PHP's reserved words included: `list()`, `use()`,
  `case()`, `new()`, `match()`, `print()`.
- **A function keeps `function`.** Every named declaration that is not at class-member level is
  a function and is PHP's `function name(…)`, a function declared inside a method's body
  included: at statement level `process($order) {` reads as a call. What Phaenon adds to
  functions — type parameters after the name, `throws`, an expression body — it adds to
  `function`.
- **Closures are PHP's**: `function () use (…) {}` and `fn () => …` are unchanged. So are
  property hooks: `get` and `set` inside a property's braces are hooks, not methods, and they
  are the precedent: PHP 8.4 declares a hook as `set(string $value) { … }`, without a keyword.
- **`fun` is no keyword.** It is an ordinary identifier, so PHP code that declares or calls a
  function `fun()` compiles. A declaration still spelled with `fun` is `P1001`, the code of a
  construct removed from Phaenon, and its message names the rewrite, which `phn fmt` makes
  (`tooling.md` §5.6).

### 1.2 Compilation

The emitter writes a method's head in PER order, `[abstract|final] [visibility] [static]
function`, inserting `function` before the name and `public` where the source writes no
visibility: PER Coding Style 3.0 requires a declared visibility on every method, and a compiled
tree is checked with a PER set (roadmap D22). The source may leave `public` out, as PHP allows;
the output never does. Modifiers the source already writes in PER order stay its text
(`tooling.md` §4). Since D19 the head is laid out freely, and the line map keeps every position
pointing into the `.phn`.

```php
protected static slug(string $title): string => strtolower($title);
// compiles to
protected static function slug(string $title): string
{
    return strtolower($title);
}
```

### 1.3 Migration

`phn migrate` lifts `function` off every method and drops a sole `public` on the way: `public
function title()` → `title()`, `final public static function create()` → `final static
create()`, `protected function apply()` → `protected apply()`. `--keep-public` keeps `public`.
A method named as §1.1 lists keeps `function`, and so does every function, one inside a
method's body included. Re-running on a `.phn` is a no-op. A tree migrated before 2026-11-09 holds
`fun`, which `phn fmt` rewrites: dropped before a method's name, `function` where §1.1 requires
the keyword. `phn fmt` also lifts `function` off a method written the PHP way, wherever §1.1
allows it, and leaves the modifiers as written, so a tree has one spelling of the keyword and PHP
code pasted into a `.phn` is idiomatic after one format (`tooling.md` §5.6).

### 1.4 Why

- Methods are the most frequent declaration in the code Phaenon exists for (Tequila's `core`
  had 870 `public function` to 59 `protected function` when it was migrated; its `.phn` tree
  declares 3,414 methods), and the keyword says nothing the reader does not see: a name followed
  by its parameters, in a class body, is a method.
- `public static function create()` takes most of a line, and most of a phone screen in a
  review. The declaration without a word is what TypeScript, Java and C# readers expect, and
  PHP's newest accessor syntax already declares this way: a property hook is `set(string $value)
  { … }`. The context, a class body or a property's braces, is what tells a declaration from a
  call, and the exceptions of §1.1 are the names where a one-token lookahead or a reader would
  need more.
- The implicit `public` is not a new rule but PHP's, made the default spelling; the emitter
  writes it out, so the PHP output stays PER and PhpStorm's "missing visibility" never fires
  on compiled code.
- A function keeps PHP's keyword: at statement level, a method's body included, a bare name and
  parentheses are a call, and a reader should not have to find the brace to tell the two apart.
- Keeping `function` valid on methods is not a second way to write the same thing by choice but
  the compatibility contract: every PHP file is a Phaenon file, and `phn migrate` or `phn fmt`
  moves it over.

### 1.5 `fun` (2026-11-01 – 2026-11-09)

D15 spelled every named function and method with `fun`: `fun id(): T` compiled to `public
function id(): T`, `protected fun` to `protected function`, and `phn migrate` rewrote `function`
to it. It was a contextual keyword, a keyword only when a name followed, so a PHP function or
constant named `fun` kept parsing. It read immediately to anyone who had seen Kotlin; `fn` would
have collided with PHP's arrow functions, and `func` and `def` were no better. D27 took it out:
a method without any keyword has the same gain without a word to learn, and Phaenon adds no
keyword (roadmap D26). The dogfood trees declared about 10,000 methods with it, which `phn fmt`
rewrites.

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
use Closure;

#[Map(fn (int $x): int => $x * 2)]
final class Doubled {}

final class Config
{
    const Closure DEFAULT = fn (string $s): string => \strtoupper($s);
    __construct(private Closure $normalise = fn (string $s): string => \trim($s)) {}
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
- The closure's parameter and return types are erased and documented as anywhere else
  (`generics.md` §4): `fn (list<string> $tags): bool` in an attribute compiles to
  `/** @param list<string> $tags */ static function (array $tags): bool`. The same holds for
  every constant expression a declaration carries outside its body: attribute arguments,
  parameter defaults, property defaults, constants and enum case values.

### 2.3 Why

Attribute-driven libraries — `tequila/object-mapper`'s mapping attributes — are where a small
inline callback is most wanted and where PHP makes it the ugliest. The lowering writes exactly
what the programmer would have written by hand after reading the RFC, so the output is PHP of
the kind D9 asks for, and the checker's rule is PHP's rule stated before the file runs.

## 3. Expression bodies

### 3.1 Spelling

```php
// in a class body
load(Id $id): T => $this->get($id) ?? throw new AggregateNotFound(static::aggregateClass(), $id);
name(): string => $this->name;
log(string $message): void => $this->logger->info($message);
fail(string $why): never => throw new LogicException($why);
static of(string $s): self => new self(trim($s));
```

- A named function or method — a method with or without `function`, a function with it — may
  have **one expression** for its body:
  `=> expr;` takes the place of the block, after the return type and the `throws` clause (§4).
- The expression is **returned**, unless the return type is `void` or `never`: then it is only
  evaluated. A `never` body is a `throw` or a call that never returns, as in a block.
- **Not allowed** on constructors and destructors, whose rules on returned values are PHP's own
  (8.6 deprecates returning one), and in generators: `yield` in the expression is an error. An
  abstract or interface method has no body, so no arrow either.
- A long expression wraps **after the arrow**, the arrow starting the continuation line, one
  indentation level in:

  ```php
  // in a class body
  load(Id $id): T throws AggregateNotFound
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
(§5.2). *Since D19 (2026-11-02) the line rule is gone: an expression body compiles to a body
laid out as PER wants it, the brace and the `return` on lines of their own (§1.2, §5.2), and
the line map points back.*

### 3.3 Why

Accessors, delegations and guards — the one-statement methods that dominate domain code — take
four lines and two braces to say one expression. Kotlin (`= expr`), C# and Scala (`=> expr`)
write them as what they are. PHP's own Short Functions RFC (2021) proposed this exact syntax and
was declined as a second way to write a function; Phaenon's answer differs because it already
has one (`fn`) and because its arrow is PHP's arrow: nothing about it is new to a reader.

## 4. `throws`

### 4.1 Spelling

```php
// in a class body
load(Id $id): T throws AggregateNotFound
commit(): void throws ConcurrencyConflict|StoreUnavailable
abstract handle(Command $c): Result throws Refusal;
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
with the migration, and an exception type is a type. A bare `@throws X` line without the
docblock was considered and rejected: `@` is PHP's error-suppression operator, so inside a body
the line would be ambiguous, and it would be a third comment syntax that is neither PHP nor a
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
  function or method without statements (`__construct(private EventStoreFactory $factory) {}`,
  the constructor that only promotes its parameters being the common case) and a class-like without
  members (`final class AggregateNotFound extends \DomainException {}`). A class's own-line brace
  separates its header from its members; an empty class has none, so the line is the whole
  declaration. A body with a comment inside keeps its lines.
- **A constructor keeps its body** — `__construct(private Foo $foo);` was considered and
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
expression body into a block; the line map keeps every position pointing into the `.phn`. Since
D22 (2026-11-04) the layout it writes is PER Coding Style 3.0's: the same braces, and an empty
body `{}` left on the declaration's line, where this layout and PER both put it.

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
reading aid: PSR-12 itself puts control-structure braces on the line. With methods without a
keyword, `=>` bodies and `throws`, a typical accessor shrinks from six lines to one; the brace
rule decides whether the remaining blocks keep paying a line each. The cost — a php-cs-fixer
exception on output and a one-time reformatting of every migrated tree (a
`.git-blame-ignore-revs` entry keeps `git blame` useful) — is paid once. The php-cs-fixer
exception went with the line rule (D19).

### 5.5 Whitespace is spaces (D21)

Unlike the rest of §5, this is syntax, not style.

- **A tab is allowed in two places only:**
  - the content of a string literal, a heredoc's or nowdoc's text included;
  - the text of a comment.
- **Everywhere else a tab is `P1005`, an error**, and the file is not compiled (D10):
  - a line's indentation, and the whitespace between tokens;
  - the indentation of a comment's continuation lines, such as the ` * ` lines of a docblock;
  - the indentation of a heredoc's or nowdoc's closing marker, and the same prefix of its body
    lines, which PHP strips with it;
  - whitespace inside a token, as in `(\tint)` or `yield\tfrom`.
- **One diagnostic per file**, at its first tab, with the number of lines that have one.
- **Four spaces per indentation level** is the canonical layout that the plugin, `phn migrate`
  and `phn fmt` write. Only the tab is checked; the width of the indentation is the project's
  fixer's to check (`tooling.md` §5.2).

**Migration and formatting.** `phn migrate` and `phn fmt` replace each forbidden tab with the
spaces up to the next tab stop, four columns apart. The code then looks as it did in an editor
that shows a tab as four columns.
- A heredoc's closing indentation and the matching prefix of its body lines become the same
  spaces, so the string's value does not change.
- A tab in a string's content or a comment's text stays.
- A docblock's indentation is part of the comment that `ReflectionClass::getDocComment()` and the
  like return, so replacing a tab there changes that string. Migrate's one-line docblocks already
  change it, and the tag parsers that read docblocks (PHPUnit, Doctrine, PHPStan) ignore
  indentation.

**Compilation** is unchanged: a source without such tabs compiles to output without them. A plain
`.php` file copied into the output is not Phaenon and keeps its tabs.

**In PhpStorm** the plugin's code style for `.phn` indents with spaces, whatever the project's PHP
style says, and Reformat Code replaces the tabs as `phn fmt` does.

**Why:**
- **The PHP community settled it.** PER Coding Style 3.1 §2.4, PHP-FIG's successor to PSR-12,
  says: "Code MUST use an indent of 4 spaces for each indent level, and MUST NOT use tabs for
  indenting." PSR-2 already said so in 2012.
- **A language that leaves it open hands the question to every project**, and an option to every
  tool. The compiler would have to follow each source's choice in what it writes: docblocks,
  `{}` bodies, expanded expression bodies. In a tab-indented source it wrote 633 docblocks and 426
  empty bodies with spaces (`tooling.md` §4's measure).
- **An error, not only a formatter's rule**: a formatter rule stops holding as soon as someone
  doesn't run the formatter. As syntax, the rule holds for every `.phn` that compiles. Go settled
  the question with its formatter, and Python 3 rejects inconsistent tabs. Phaenon takes the PHP
  community's answer and makes it part of the language.
- **Strings and comment text keep their tabs.** A tab in a string is part of its value. A
  comment's text is prose, and a table pasted into it may need its tabs. A comment's indentation
  is layout: its lines follow the code's.

## 6. Editor

PhpStorm reads a method without a keyword as PHP's: the plugin's lexer, which hands PhpStorm's
PHP parser a `.phn` as PHP, inserts a `function` token where such a method stands (until
2026-11-09 it handed `fun` over as a `function` token with the text `fun`). A method without
visibility is public to PhpStorm as to PHP, and its "missing visibility" inspection is filtered
on `.phn` files for methods, constants and `private(set)` members alike. A closure in an
attribute is checked against the PHP 8.5 rule by the compiler, not the editor.

The plugin reads `throws` and its names as an erased construct (navigable, counted by Optimize
Imports), hands PhpStorm's parser a block for an expression body, and formats `.phn` with the
canonical style of §5.

## 7. Decisions

| # | Decision | Why |
|---|---|---|
| F1 | *(Superseded by F11 on 2026-11-09 for the keyword; the implied `public` stands.)* `fun` declares named functions and methods; absent visibility is `public`, on constants and `private(set)` members as well; `function` stays valid | §1.4 |
| F2 | *(Superseded by F12 on 2026-11-09: `fun` is no keyword at all.)* `fun` is contextual, never reserved | §1.1 |
| F3 | Closures in constant expressions are written as `fn` / `function` and lowered to `static function`; `$this` and `use` are `P2531` | §2 |
| F4 | *(Superseded by F13 on 2026-11-09 for the keyword; dropping a sole `public` and `--keep-public` stand.)* `phn migrate` writes `fun` and drops a sole `public`, `--keep-public` keeps it | §1.3 |
| F5 | A named function or method may have an expression body, `=> expr;`; returned unless `void` / `never`; not on constructors, destructors or generators | §3 |
| F6 | `throws A\|B` in the signature, checked to name `Throwable`s, compiled to `@throws`; not checked exceptions | §4 |
| F7 | No bare `@tag` lines: a one-line docblock, or the signature | §4.4 |
| F8 | Canonical layout: opening braces of declarations on their line, one-line docblocks; a style, not syntax; output needs php-cs-fixer's brace rule off | §5 |
| F9 | `phn migrate` lifts bare `@throws`, moves braces up, collapses one-line docblocks, writes `=>` for single-expression bodies; `--keep-layout` keeps the layout | §4.3, §5.3 |
| F10 | **Whitespace is spaces** (2026-11-04, D21): a tab outside a string's content or a comment's text is `P1005`, an error; `phn migrate` and `phn fmt` replace such tabs with spaces to tab stops four columns apart, `--keep-layout` included; the plugin's `.phn` code style indents with spaces | §5.5 |
| F11 | **Methods without a keyword** (2026-11-09, roadmap D27): a declaration at class-member level is a method, `[modifiers] [&]name[<T>](…)` and its body; no visibility is `public`, written out by the emitter as PER requires; PHP's `function` stays valid on a method and is required for a name that is a modifier, `var`, `const`, `Closure`, `function` or `fn`; every other named declaration is a function and keeps `function`, one inside a method's body included | §1.1, §1.2, §1.4 |
| F12 | **`fun` leaves the language** (2026-11-09, roadmap D27): an identifier again; a leftover `fun` declaration is `P1001`, which `phn fmt` rewrites | §1.1, §1.5 |
| F13 | `phn migrate` lifts `function` off methods and drops a sole `public`, `--keep-public` keeps it; functions and the names of F11 keep `function` | §1.3 |

## 8. Decisions made while building D18

| Topic | Decision | Why |
|---|---|---|
| Where `throws` is recognised (2026-11-02) | By the parser, after the parameter list or the return type of a named function or method; the lexer leaves it a `T_STRING` | PHP's grammar admits only `{` or `;` there, so the position is free. A lexer rule ("`throws` before a name") cannot tell the clause from the names PHP code gives things: `class throws extends …`, `throws instanceof X`, `foreach (throws as …)`, a trait method `throws as protected` |
| What an expression body is in the tree (2026-11-02) | The declaration keeps the arrow, and its body holds the statement the arrow stands for: a `return`, or an expression statement when the written return type is `void` or `never` | "Typed exactly as a `return` would be" then holds by construction: every pass that reads a body reads a block of one statement, and only the printer and the emitter look at the arrow. `void` and `never` are keywords, so the written type decides |
| Codes (2026-11-02) | `P1004` (syntax family, error) for an expression body the design refuses: on a constructor or a destructor, or one that yields. An arrow on an abstract or interface method is a body and keeps the codes any body gets there, `P2504` / `P2514`. `P2319` for a `throws` member that is not a `Throwable` (an unbounded type parameter included), beside `P2317` / `P2318` for `throw` and `catch`; an unknown class is `P2001` | One mistake, one diagnostic: what the grammar admits and the design forbids is the syntax family's (`P1003` is the same shape); a non-`Throwable` is the mistake the `throw` and `catch` codes already name, in a third position. A constructor's arrow is checked as a statement, so no `P1101` for a returned value follows it |
| `throws` and the target (2026-11-02) | A class newer than the target is not reported in a `throws` clause | The clause compiles to a docblock and fails nothing on the older PHP, as `catch (X)` does (php-versions.md §9) |
| PHPStan's `throws.unusedType` (2026-11-02) | Allow-listed in the checker's parity test, beside `catch.neverThrown` | PHPStan 2 reports at level max a `@throws` type the body never throws. The clause is documentation the checker verifies, not checked against the body (§4.1); the throw analysis on the open list will read it |
| The throw analysis reads the clause (2026-11-05) | As it reads a `@throws` tag: a call throws what its callee's clause names, so a caller's catch nothing reaches is `P2403`; a member the body never throws is `P2404` where no override can replace the body (private and final methods, final classes, functions). Still not checked exceptions (`tooling.md` §9.1) | The row above deferred this to the throw analysis; it matches PHPStan's `catch.neverThrown` and `throws.unusedType`, so both left the parity test's allow-list |
| Merging the tag without a line (2026-11-02) | No docblock: `/** @throws A\|B */ ` before the attributes and modifiers. A docblock whose `*/` stands alone: the tag takes that line, ` * @throws A\|B */`. A one-line docblock, or content beside the `*/`: the `*/` moves to the next line with the tag, ahead of what stands there (`/** Summary.` over ` * @throws A\|B */ public function f()`), or onto a blank line below. PHP's reflection and PHPStan read every form | §4.2 asks for the docblock and an unchanged line count; one tag per line is all PHPStan reads, so the tag needs a line of its own, and these are the lines a docblock already has. Writing a second docblock would hide the first from reflection, where PHPUnit and attribute-less libraries read annotations |
| A docblock below the attributes (2026-11-02) | `#[A] /** … */ f() throws X` gets `/** @throws X */ ` before the attributes, its own docblock left as written; generated generic tags go there too | PHPStan reads the comment before a declaration's attributes, PHP's reflection the one before the keyword: the tag goes where the tool that reads it looks |
| The one new exception to the line rule (2026-11-02) | A docblock on the declaration's own line (`/** Summary. */ f() throws X`) gains one line; when the generics lowering writes the declaration's docblock, which adds lines already, the tag joins it as one more line | Such a docblock has no next line to borrow. It is rare in source, and the canonical layout puts docblocks above their declaration |
| The long form's output (2026-11-02) | An arrow on a continuation line hands its brace to the end of the signature's line and becomes `return` (vanishes under `void` / `never`); the short form becomes `{ return expr; }` where the arrow stood | With the brace on the arrow's line, a multi-line expression's continuation lines stood one level deeper than a block's statement, and php-cs-fixer's `statement_indentation` and `method_argument_space` fired on tequila/core and tq-cli. Moving the brace up adds no line and leaves only the brace's placement to excuse |
| php-cs-fixer on compiled output (2026-11-02) | Beside D12's four line-1 rules: `braces_position` (functions, methods, closures, and the one-line blocks of expression bodies) and `class_definition` (it puts a class-like's brace on its own line). Measured on the re-laid-out core and tq-cli with their own configurations: nothing else fires, D11's `single_line_after_imports` aside where an output gets a compiler import | §5.2's "brace-placement rule" is two rules in php-cs-fixer 3.95; no docblock rule is in `@PSR12`, so the merged `@throws` forms pass |
| `throws` lifting and `--keep-layout` (2026-11-02) | Lifting `@throws` is part of the lift (`--keep-docblocks` skips it), not of the layout | It moves a type into the signature as the generic lifts do (D14); `--keep-layout` keeps braces, docblocks' shape and bodies |
| `never` bodies in migrate (2026-11-02) | A body that is one expression statement under `never` (`throw new X(…);`) becomes `=> …;` as under `void` | §3.1 evaluates the expression under both; §5.3 named `void` only |
| Limits of the layout (2026-11-02) | A docblock is not collapsed when the one line would pass 120 columns; a brace moves up whatever the width; a comment between the signature and the brace keeps the block as a comment inside does; an empty body keeps its closing brace on its own line (`) {` / `}`). A multi-line expression joining the signature's line moves its continuation lines one level out; one holding a multi-line string or heredoc, whose text that would change, wraps after the arrow instead | A collapsed docblock only saves lines; a brace has no other place short of reflowing the signature, which is `phn fmt`'s work (M6). A comment there would have nowhere to go |
| `throws` members (2026-11-02) | Class names, `self`, `static` and `parent`, and type parameters; no type arguments, no keyword types. `@throws void` and a tag naming a template left in the docblock stay in the docblock with a note | What `catch` accepts, plus the type parameters §4.1 allows; PHPStan's `@throws void` ("throws nothing") has no clause spelling |
