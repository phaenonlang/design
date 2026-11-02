# Phaenon — Global plan (draft 0.1)

Status: **draft** · 2026-10-28 · companion: `generics.md`

## 0. What Phaenon is (one paragraph)

A typed dialect of PHP 8.5+ that **compiles to idiomatic PHP** — "TypeScript for PHP". Adoptable
file-by-file in any Composer project (`.phn` beside `.php`), with full access to the PHP ecosystem
(Symfony, PHPStan, FrankenPHP, every extension) because the output *is* PHP. The product is the
**type checker**; the syntax is the packaging; the native runtime is a possible phase 3, not a
prerequisite. The first real codebase is Tequila (~50 repos, Symfony, MongoDB, NATS).

## 1. Decisions so far

| # | Decision | Why | Where |
|---|---|---|---|
| D1 | Transpile to PHP, not a new runtime | ecosystem is the moat; Hack/HHVM is the cautionary tale; an oracle (`php`) exists for every behaviour | conversation 2026-10-28 |
| D2 | Compiler written in **Go** | single static binary shipped through a Composer plugin (esbuild-via-npm model); fast watch mode/LSP; frontend reusable by a future native runtime | — |
| D3 | Generics: real `<T>` syntax, erased; `class<T>` for class names as values; `is`/`as` operators — `reified` removed 2026-10-29 (`generics.md` §11.2) | `generics.md` | |
| D4 | Emitted PHP carries PHPStan-vocabulary docblocks; PHPStan `level: max` on output is part of compiler CI | output must read as well-typed modern PHP | `generics.md` §4.2, §8.4 |
| D5 | Strings stay primitive; method-call syntax on primitives compiles to static calls (`"a"->upper()` → `\Phaenon\Str::upper("a")`) | 95% of the "string object API" win, no value-semantics change | conversation |
| D6 | Concurrency: structured concurrency over Fibers, **no function colouring** (no `async fn`). Revised 2026-11-01: the loop is **Revolt** and the awaitable **amp v3's `Future`**, Phaenon ships no loop of its own; its contribution is syntax, structure and checking | PHP's synchronous readability is the point; Fibers already exist. Two loops in one process cannot share a socket wait, and amp's libraries are the only non-blocking I/O PHP has, so a Phaenon loop would exclude them (`concurrency.md` §1, C1) | `concurrency.md` |
| D7 | **Own parser**: hand-written lexer (state machine ported from `zend_language_scanner.l`), recursive-descent + Pratt expressions (precedence from `zend_language_parser.y`), typed AST node families, speculative parse for `<…>(` | z7zmey/php-parser evaluated 2026-10-28: PHP 7.4 only, goyacc LALR (no backtracking, poor recovery), untyped `Vertex` AST, dead since 2021 — nothing to keep. VKCOM/noverify excluded (Russian company) | — |
| D8 | Extension **`.phn`**, CLI **`phn`** (`phn build\|check\|watch\|fmt\|migrate\|lsp`), project **`phaenon`**, GitHub org **`phaenonlang`** (`phaenon` is taken), Composer vendor `phaenon/*` to be checked on Packagist | short and typeable like `tsc`; `.phn` reads as PHP-family; `phaenon.dev` being bought; org `phaenonlang` | — |
| D9 | **The emitted PHP contains no identifier the programmer did not write**: no hidden parameters, no synthesised members, no renamed locals. The compiler adds only the header line, `declare(strict_types=1)`, docblocks, and `phaenon/runtime` calls where the source used `is` / `as` on a type PHP cannot test | the output must read as PHP a person wrote; `reified`'s hidden constructor parameter broke that and fought Symfony DI (2026-10-29) | `generics.md` §11.2 |
| D10 | **A file with type errors produces no output.** `phn build` withholds the `.php` of a file the checker rejects and exits non-zero; `--emit-on-error` writes it anyway for a codebase that adopts the checker while it fixes its findings (2026-10-29) | the type checker is the product, and a build that writes what it rejected teaches people to ignore it; `tsc` emits by default and its users switch `noEmitOnError` on in every strict project | `language-overview.md` §6 |
| D11 | **Generated PHP imports what it uses.** *(Placement superseded by D19: since 2026-11-02 the import takes a line of its own; the shadowing rule stands.)* A class the compiler introduces — `\Phaenon\Seq`, `\Phaenon\Str`, `\Phaenon\Type` — is imported with `use` and called by its short name (`Seq::map(…)`), never as an inline FQCN. When the short name is already taken in the file (a user import, a class of the namespace, or, since 2026-11-02, the first segment of a relative qualified name in its code or docblocks, which PHP would resolve through the import), the FQCN is used for that file (2026-10-30). **The import joins the file's own `use` block** (2026-11-01), on the line of a neighbouring class import: in its alphabetical place when the block is sorted (php-cs-fixer's order: case-insensitive, `\` read as a space), after the last class import otherwise; a file without imports gets it at the start of the line where the first statement after the `namespace` begins, a file without a namespace on the compiler's line 1. Line N of the `.php` stays line N of the `.phn`. Sharing a line breaks one php-cs-fixer rule, `single_line_after_imports` (plus `blank_line_between_import_groups` when the block holds only function or constant imports): an output that gets an import needs that rule off beside D12's four line-1 rules, and an import written by hand needs nothing | the output must read as PHP a person wrote (D9), and people import. Until 2026-11-01 the import sat on line 1 after the `namespace` statement, which php-cs-fixer's `blank_line_after_namespace` rejects and, in a file with imports, `no_extra_blank_lines` too (tq-cli's dogfood): two broad rules, against one rule about imports alone. The line count rules out the placement a person would choose (a line of its own), so the import shares the line a person would have put it next to | `collections.md` §3.1 |
| D12 | **Sources compile into a sibling output directory, named in `phaenon.json`** *(the line-1 header superseded by D19)*: `{"roots": {"src": ".src", "tests": ".tests"}, "exclude": [...]}`. `phn build` writes `src/A/B.phn` → `.src/A/B.php`, copies every other file under a root (fixtures, YAML) over, and removes outputs with no source behind them; Composer's PSR-4, PHPUnit, PHPStan and php-cs-fixer point at the outputs, which are gitignored, and the IDE indexes the sources only. Without a project file `phn build` still writes beside the source (adoption, a file at a time) or under `--out` (2026-11-01) | A `.php` beside each `.phn` declares every class twice for the IDE and doubles every directory. The output is a *sibling at the same depth* and not a nested mirror (`build/src/…`) so that a compiled file's `__DIR__` sits where its source's does: `__DIR__.'/../config/services.yaml'` and `\dirname(__DIR__, 3)` — Symfony bundles, Rector configs, test kernels — keep resolving without the emitter rewriting `__DIR__`, which would put text in the output the programmer did not write (D9). The price is paid by php-cs-fixer: line 1 carries the open tag, the marker, `declare` and the namespace together, so four PSR-12 rules about line 1 are off on the outputs | `README` § Composer |
| D13 | **A Phaenon library ships its `.phn` sources and no compiled PHP.** Consumers build it on install: a Composer plugin (M6) runs `phn build` in every installed package that has a `phaenon.json`, with the `phn` binary pinned by the package and distributed as a Composer package per platform; the one-time plugin trust question is answered for everyone by `config.allow-plugins` in the consumer's `composer.json`, as for `symfony/flex`. Until the plugin exists, a consumer runs `phn build` in `vendor/<pkg>` itself (2026-11-01) | The emitter is deterministic: with `phn` and the library pinned, the output is a byte-for-byte reproducible cache, and committing a cache invites drift and doubles every review. Install hooks are Composer's culture, unlike npm's (where packages ship `dist/`); the one real cost, a hard dependency on the plugin for anyone consuming Phaenon code, is accepted | `roadmap.md` M6 |
| D14 | **Docblock prose without a type**: a parameter description is written `@param $x - prose` (the dash, as in JSDoc), a property's `@var T prose` becomes the property's docblock `/** prose */`; `phn migrate` writes both forms and the emitter puts the types back and drops the dash (2026-11-01) | Types live in the signature, so a docblock type is redundant and drifts. `@param $x prose` is valid PSR-19 and phpdoc-parser (`TypelessParamTagValueNode`), but PhpStorm reads the first word of the prose as the type (an undefined class, a wrong-type warning, and `prose|string` as the parameter's type); the dash is the one spelling every tool reads the same way | `README` § migrate |
| D15 | **`fun` declares a named function or method, and a method without visibility is public**: `fun id(): T` compiles to `public function id(): T`, `protected fun` to `protected function`; `function` stays valid (every PHP file is a Phaenon file) and `phn migrate` rewrites it, dropping a sole `public` unless `--keep-public`. In a constant expression — an attribute argument, a constant, a default — a closure is written `fn (…) => …` or `function (…) {…}` and compiled to the `static function` PHP 8.5 accepts there; `$this` or `use` in one is `P2531` (2026-11-01) | The declaration Phaenon's code consists of mostly is `public function`, two words that say nothing the reader did not assume; the implicit `public` is PHP's own rule, and the emitter writes it out so the output stays PSR-12. PHP 8.5 admits only `static function` closures in constant expressions because arrow functions capture implicitly; at class level there is nothing to capture, so the spelling is a restriction of syntax the compiler can lift while the checker keeps PHP's rule | `functions.md` |
| D16 | **`concurrently { … }` and `spawn`**: a block owns the tasks spawned inside it and cannot be left while one runs; the first failure cancels the siblings and is rethrown at the block's end; blocks are expressions (`return concurrently { … };`); the handle is `Amp\Future<T>`, awaited with `->await()`; the block lowers to `Scope::run(function () use (…) { … })` on a second package `phaenon/concurrent` (2026-11-01) | Structure is what amp lacks (`await(array_map(async(...), $jobs))` leaks the siblings of a failed job); every identifier in the output is the programmer's (D9) because the block is a closure in the output and the scope is ambient | `concurrency.md` §15 |
| D17 | **Phaenon follows PHP, and a project compiles for a target.** A PHP release's syntax and library are Phaenon's from the release on; each release is a milestone named after it. The target is `phaenon.json`'s `"php"`, otherwise the lowest version composer.json's `require.php` admits; `phn migrate --php X.Y` writes it. A construct newer than the target is lowered when the lowered PHP is exact and idiomatic and refused otherwise; a library symbol newer than the target is reported; deprecations are reported whatever the target. PHP 8.5 stays the floor at least until it leaves active support (2027-12-31); the floor is revisited before the repositories are published | Every PHP file is a Phaenon file, so holding back what PHP ships would make Phaenon a dialect of an old PHP. The target lives in `phaenon.json` first because migrating to Phaenon and to a newer PHP is often one move, and falls back to composer.json because that is where a project already says which PHP it runs on. Keeping 8.5 lets Tequila build while its images move to 8.6 and lets a project adopt Phaenon without upgrading PHP first; the lowerings it costs are few | `php-versions.md` |
| D18 | **Expression bodies, `throws`, and the brace on the declaration's line** (2026-11-02; compilation and class braces amended by D19): `fun name(): string => $this->name;` compiles to `{ return …; }` on the same lines; `throws A\|B` follows the return type and compiles to `@throws`, verified to name `Throwable`s but not enforced on callers; the canonical layout puts a declaration's opening brace on its line and a one-line docblock on one line, written by the plugin, `phn migrate` and later `phn fmt`, while PSR-12 placement stays valid. No bare `@tag` lines | One-statement methods took four lines and two braces; exception types are types and belong in the signature with the others (D14); PSR-12's own-line brace for declarations is a PEAR inheritance. The cost is a php-cs-fixer brace exception on compiled output, beside D12's, and one reformatting of each migrated tree | `functions.md` §3–5 |
| D19 | **The compiled PHP is laid out freely, and a line map points back into the source** (2026-11-02): the emitter writes the header, the imports it adds, the docblocks it writes, expanded expression bodies and declaration braces as PSR-12 wants them, compiles `__LINE__` to the source line, and ends every compiled file with a `// phaenon-map v1 …` comment of runs and holds; `phn trace`, the PhpStorm plugin (navigation and debugging) and `phaenon/runtime` (`LineMap`, a Monolog processor, a Sentry integration) read it. The canonical source layout keeps a class-like's brace on its own line; only functions and methods put it on the declaration's line | The line rule cost the output its PSR-12 layout one feature at a time (D11, D12, D18, the generic docblock) and a growing list of php-cs-fixer exceptions, while saving only the line half of a mapping the tools need anyway: the code that runs is `.src/Foo.php`, so links, breakpoints and coverage translate the file in any case. Twig's compiled templates and their debug info are PHP's precedent | `line-map.md` |

## 2. Repository layout

```
~/labs/phaenon/
├── php-src/        reference clone of php/php-src — read-only (grammar, stubs, .phpt corpus)
├── design/         this directory → becomes repo phaenon/design (specs, decisions, RFC-style docs)
├── phaenon/        the compiler, Go — monorepo:
│   ├── cmd/phaenon          CLI: build, check, watch, fmt, migrate, lsp
│   ├── internal/token|lexer|ast|parser
│   ├── internal/types|checker|infer|docblock   (docblock = vendor PHP type reader)
│   ├── internal/emit                           PHP printer + docblock emitter
│   ├── internal/lsp
│   ├── stdlib/                                 PHP stdlib signatures from php-src *.stub.php + generics overlay
│   └── conformance/                            golden tests, phpt harness, PHPStan-on-output
├── runtime/        Composer package phaenon/runtime (PHP): \Phaenon\Type, \Phaenon\Str, \Phaenon\List_/Map helpers, event loop
└── composer-plugin/  phaenon/composer-plugin — fetches the binary per OS/arch, compiles on install/dump-autoload (later, M6)
```

Names are placeholders; the GitHub org and the binary name are open.

## 3. Milestones

Each milestone has a **design doc first** (like `generics.md`), a **test corpus**, and a **Tequila
dogfood target**. Nothing is "done" until the Tequila target's own test-suite passes on compiled
output.

### M0 — Foundations: parse PHP, print PHP, prove it round-trips
**Status 2026-10-28: done except the `phn build` deliverable (moved to M1, it is Composer work).**
Lexer 23,536/23,536 token-identical to PHP 8.5.3; parser 23,226/23,226 PHP-accepted files parse;
printer round trip 23,393/23,393 token-identical; execution comparison 4,546/4,546 `Zend/tests`
files behave identically (680 tests that observe line numbers are not run — see M1).
- `design/language-overview.md`: what stays from PHP, what is removed (tags, `array` as a type —
  demoted to legacy, haystack/needle stdlib…), what is added (generics, `list`/`map`, `type`,
  `is`/`as`, string methods, concurrency). Scope only; details per feature doc.
- Go module skeleton, lexer, **hand-written recursive-descent parser** for PHP 8.5 (the `.y` file is
  the spec; hand-written for error recovery and LSP), full-fidelity AST (comments, positions).
  - Evaluate forking an existing Go PHP parser first (VKCOM/php-parser from noverify,
    z7zmey/php-parser): a week of evaluation can save months — or cost months if the AST is wrong
    for a type checker. Decide by end of week 1.
- Printer that emits PHP; **phpt harness**: parse → print → run with `php` → compare output with
  the original for the ~24k `.phpt` files (minus ext-specific ones). Target: 100% of `Zend/tests`
  that don't need unavailable extensions.
- Deliverable: `phaenon build` turns a `.phn` (= PHP without `<?php`) into byte-equivalent PHP.

### M1 — First dogfood: a Tequila library in `.phn`
- ✅ `phn build`: `.phn` (= PHP without `<?php`) → PHP, in place (`Foo.php` beside `Foo.phn`) or
  `--out DIR`; `--check` for CI.
- ✅ **Line-preserving emission**: the emitter is patch-based (the source passes through byte for
  byte, every edit keeps its line count), so line N of the `.php` is line N of the `.phn`. Line 1
  carries `<?php`, the `@generated by phaenon` marker and `declare(strict_types=1)` (injected unless
  the file declares `strict_types` itself). Proven by `TestEmitExecution`: 4,843/4,843 `Zend/tests`
  files produce byte-identical output after migrate + build, line numbers included.
- ✅ `phn migrate` v0: mechanical `.php` → `.phn` (strips `<?php`, a leading
  `declare(strict_types=1);`, a trailing `?>`; refuses templates and generated files).
- ✅ Composer integration v0: in-place output needs no `composer.json` change; libraries commit the
  generated `.php` (consumers need no build step) and run `phn build --check` in CI; php-cs-fixer
  skips `@generated by phaenon` files. A Composer plugin / binary distribution is M6.
- ✅ Target: `tequila/reflection` (18 files) compiled from `.phn` on branch `phaenon-dogfood`; phpunit
  (25 tests), PHPStan level 10 and php-cs-fixer pass.
- Found on the way: in Phaenon mode `foreach ($x as $y)` collided with the `as` operator (fixed with
  a no-`as` context for the subject); labels named with contextual keywords (`out:`) (fixed);
  `Closure(int): R` lexes `(int)` as a cast token (M3 must handle cast tokens in type position);
  declaring `class is` / `function is` is a PHP 8.5 deprecation that Phaenon turns into an error.
- Open: Phaenon-only syntax is rejected by the emitter until M3; `.phn` tooling (php-cs-fixer,
  editors) reads files without `<?php` as inline HTML — formatting `.phn` needs `phn fmt` (M6).

### M2 — Type checker core (no generics syntax yet) — done 2026-10-28
- ✅ `phn check`: name resolution, class hierarchy (traits, interfaces, enums, hooks, asymmetric
  visibility, readonly), the PHPStan type lattice (`internal/types`), docblock reader
  (`internal/docblock`), program model with lazy loaders (`internal/program`): project files,
  Composer autoload maps, php-src stubs embedded with a Phaenon overlay (`stdlib/`).
- ✅ Local bidirectional inference with contextual typing of closures; narrowing on `instanceof`,
  `is_*`, `===`, `isset`, `count`, `in_array`, `array_key_exists`, `assert`, `@phpstan-assert`;
  loops to a fixed point; try/catch; `match` exhaustiveness.
- ✅ Docblock generics already: `@template` inference at call sites (fresh identities per call,
  foreign templates inferred, generic first-class callables, conditional return types).
- ✅ Diagnostics `P1`/`P2`/`P4` with stable codes, source line + caret, `--json`,
  `@phpstan-ignore`; `phn build` withholds files with type errors.
- ✅ Target met: **0 findings on `tequila/reflection` (18 files) and `tequila/object-mapper`
  `src` (23 files)** — the same result as PHPStan level 10 on both. Golden corpus plus an opt-in
  PHPStan cross-check (`PHAENON_PHPSTAN=…`) that fails when PHPStan reports something the corpus
  does not; 0.1–0.5 s per library.
- Found on the way (see `checker.md` §9): PHPStan keeps property narrowings across calls and
  infers templates that are not in scope; `switch` must not narrow scalars; bare `Closure` is
  accepted for any signature; PHPStan crashes (internal error) on a deliberately broken enum.
- Open: throw analysis (dead catches), purity/unused-result rules, missing-type severity, the
  LSP (M6). The PHPStan comparison allow-lists those identifiers explicitly.

### M3 — Generics (per `generics.md`) — ✅ done 2026-10-28
- ✅ Syntax in the program model: `class Foo<T of Bar>`, `extends Base<T>`, `implements I<T>`,
  function / method / closure `<T>`, file-level `type` aliases (generic, namespace-scoped), `map<K, V>`.
- ✅ Checker: arity and bounds of type arguments, `in`/`out` positions, static members, generic
  property redeclaration, erased-T-as-value rule, `is` narrowing and `as` / `as?`, explicit type
  arguments at call sites, reified inference with closed-type requirement (`P3001`–`P3009`).
- ✅ Emitter: erasure to native types + PHPStan docblocks (merged into existing ones),
  override-erasure for parameters and returns, reified → leading `string|\Phaenon\Type` parameter /
  `$__T` property / synthesised binding constructors, `is`/`as` lowering (native where PHP can),
  alias expansion. Golden test compiles `.phn` → `.php`, runs `php -l` and PHPStan level max.
- ✅ `phaenon/runtime` (`~/labs/phaenon/runtime`, private `phaenonlang/runtime`): `\Phaenon\Type`
  with 12 PHPUnit tests and PHPStan max clean.
- Found on the way: PHPStan reads one docblock tag per line, so generic declarations cost
  output lines (decision in `generics.md` §11); `Foo::class(...)` is not callable syntax, hence
  `Type::of()`; GoLand rewrites `go.mod`'s module line when it opens the project.
- ✅ `phn migrate` lifts generic docblocks into syntax (`@template` → `<T of B>`, `@extends` /
  `@implements` → clause arguments, `@param` / `@return` / `@var` → declared types, `T[]` →
  `array<T>`, `array<K, V>` → `map<K, V>`, `$this` → `static`); a tag is lifted only when its
  type has a Phaenon spelling and erases to the declared native type, the rest stay with a note.
  Hyphenated names (`class-string<T>`, `non-empty-string`, `array-key`, `positive-int`, …) became
  native syntax for it; `@param $x prose` is the prose-only form the emitter fills back in.
- ✅ Dogfood: `tequila/reflection` (`phaenon-dogfood`) and `tequila/object-mapper`
  (`phaenon-dogfood`, 51 files) migrated with real generics; both suites, PHPStan and php-cs-fixer
  pass on the compiled output. The checker inherits a parent's generic signature into an
  undocumented override, as PHPStan does for docblocks.
- Open: `--runtime-guards`, Symfony DI warning, `array-key` / shapes as native syntax (M4),
  descriptor hoisting, translating Phaenon vocabulary inside user docblocks; a constructor-level
  `@template` used only by a promoted property stays a docblock (it is a class type parameter in
  disguise — a per-class design decision for a maintainer, not a mechanical one); object-mapper's
  `tests/` carry 16 pre-existing findings outside PHPStan's `paths` (Symfony container `get()`
  typing, untyped closure parameters).
- 2026-10-29: `reified` removed (keyword, hidden `string|\Phaenon\Type` parameters, `$__T`,
  synthesised binding constructors, the planned Symfony DI warning, codes `P3005`/`P3006`), and
  the hyphenated PHPStan names dropped from the syntax. Replaced by `class<T>`, the type of
  `T::class`, and by the runtime-generics idioms Tequila already writes by hand (a `class<T>`
  constructor parameter, an overridden static getter, an attribute-carried class name). The
  hidden parameter fought Symfony DI and made the output read as machine-generated; the PHPStan
  vocabulary was a migration shortcut, not a language decision. `generics.md` §11.2, D9.
- 2026-10-29, `tequila/mongodb-odm` (124 files, PHPStan level 8): migrated and built on
  `phaenon-dogfood`; its suite (162 tests), PHPStan and php-cs-fixer pass on the compiled output.
  The first `phn check` reported 150 findings in `src`; 110 of them were the compiler's and are
  fixed (`checker.md` §9, 2026-10-29 entries): the classes of ext-mongodb were unknown, so
  `php`'s loaded extensions are now reflected into stubs; impossibility diagnostics now need the
  PHP types to agree with the refined ones (`treatPhpDocTypesAsCertain: false`); `@var` narrows
  instead of being checked for acceptance, retypes above any statement and types a `return`; a
  promoted property's `@var` reaches the parameter (checker and migrate); a docblock above an
  attribute reaches the declaration; `$r[$k] = $v` types `$r`; `P3008` lets static methods use
  class parameters; enum interface properties, callable strings, `new $class` on abstract
  bounds, by-reference closure uses, `new class (…)` arguments, trait methods through users,
  `iterable` against `array|Traversable`, and conditional returns with defaults (`explode`,
  `microtime`, `array_shift`, `array_map`, `preg_replace`). The remaining 41 (39 in `src`) are
  PHPStan's level-9/10 findings on a level-8 library — `mixed` from Symfony config arrays and
  untyped `array` parameters. Decided: the library went to level max on `main` (PR #45, merged
  2026-10-30); rebuilt on it, `phn check` reports **0 errors** after 13 more checker gaps went
  (`class_exists` proving a class-string, static-method asserts, constant-key paths, `?? null`
  narrowing, `array_key_first` on non-empty arrays, nested index assignment).

### M4 — Collections and strings — ✅ done 2026-11-01
- `list<T>` / `map<K,V>` as first-class types with literals; `array` as a legacy type that
  interoperates (design doc: migration ergonomics for the array-as-both-things codebase).
- `\Phaenon\*` stdlib with consistent argument order, written in PHP (runtime package) with
  Phaenon signatures; string method syntax, `bytes` vs `string`, grapheme-aware unicode API over
  `intl`.
- Target: `tequila/core` (the heaviest array user).
- 2026-10-30, first slice (`collections.md`, `strings.md` reviewed the same day): method syntax on
  `list`, `map` and `string` receivers compiled to `Seq::` / `Map::` / `Str::` static calls with the
  class imported (D11); the stdlib written in Phaenon and read as stubs; the tuple type `(A, B)`;
  the `bytes` primitive. Found on the way: findings inside a loop body were lost whenever the loop
  needed a second pass (the fixed point discarded them, the dedupe set remembered them) — fixed,
  the three dogfood libraries re-checked at 0; `$a?->b->c` short-circuits as PHP does.
- 2026-10-30, second slice: `bytes` in use — the `(bytes)` cast (erased), `P2530` for `.` and
  interpolation, `P2305` for `===` against text, the overlay typing PHP's binary producers and
  consumers, `\Phaenon\Bytes` and `Str::bytes`; `map`'s key bound `P2518` with the implied
  `K of int|string`; `P2516` an error. Dogfood libraries unchanged at 0.
- 2026-10-30, third slice: a null receiver (`?->`, a short-circuited chain) compiles to a guarded
  static call when the receiver is a side-effect-free path, else `P2519`; `\Stringable`
  receivers get the `Str` methods; `phn migrate` lifts `array{A, B}` to `(A, B)`; `phn stdlib`
  materialises the compiled stdlib for the runtime package; `phn build --no-types` is gone.
  Fixed: `phn build` compiled a reparsed tree, so the CLI had never lowered method syntax.
- 2026-11-01, fourth slice (`functions.md`): `fun` as the spelling of a named function or method,
  `public` implied; closures in constant expressions lowered to the `static function` PHP 8.5
  takes; `phn migrate` writes `fun`. Decided before M5 so that no later migration is redone.
- **Closed 2026-11-01.** ✅ `list<T>` / `map<K, V>` with literals and `array` as the legacy type,
  the tuple type `(A, B)`, `bytes`; ✅ the stdlib in Phaenon (`Seq`, `Map`, `Str`, `Bytes`,
  `phaenon/runtime` with 95 tests) behind method syntax on `list`, `map`, `string`, `bytes` and
  `\Stringable` receivers; ✅ `fun`, implied `public` (methods, constants, `private(set)`
  members), closures in constant expressions (D15); ✅ `phaenon.json` and the sibling output
  layout (D12), sources-only libraries (D13), docblock prose spellings (D14); ✅ the PhpStorm
  plugin parses every construct, with navigation, completion, find-usages and rename inside
  erased types. *Target:* `tequila/core` at 812 `.phn` on a `main` at PHPStan level max — suite
  (1188 tests), PHPStan max and cs green on the output, `phn check` at 11 findings, all PHPStan
  rules deliberately not matched; the three earlier libraries respelled with identical output.
  *Found on the way:* findings inside loop bodies lost to the fixed point; `phn build` compiling
  a reparsed tree; PhpStorm's docblock token outside its comment set; PhpStorm reading a bare
  `@param $x prose` as a type (D14); `[...$a][0]` unparsed by PhpStorm 2025.1. *Open:* the
  checker items in the vault's `open-items` (regex narrowing, `Closure::call`, the 11 core
  findings), the Composer plugin (M6), moving the three earlier dogfood branches to the sibling
  layout.

### M5 — Concurrency — ✅ done 2026-11-01
- Design: `concurrency.md` (reviewed 2026-11-01): `concurrently { }` blocks and `spawn` over
  amp v3 / Revolt, `Amp\Future<T>` handles, cooperative cancellation and `timeout:`, a second
  package `phaenon/concurrent`, checker codes `P26xx`.
- Order: the FrankenPHP worker-mode check of Revolt between requests (time-boxed), goldens,
  checker, emitter, the runtime package.
- Target: `tq-cli` first (its fifteen `Api::concurrently` fan-outs on amp already; suite, PHPStan
  and cs on the compiled output), then `tequila/nats` with a `RevoltTransport` and the
  single-reader change to `Connection`. The roadmap's original order (nats first) was reversed
  at the review: nats is synchronous by its own ADR and has no fan-out to express.
- **Closed 2026-11-01.** ✅ `concurrently (opts) { … }` as expression and statement, `spawn expr`
  and `spawn { … }`, checker `P2601`–`P2606`, the emitter's `Scope::run` / `Scope::spawn` with a
  computed `use` clause; ✅ `phaenon/concurrent` (`phaenonlang/concurrent`), compiled from
  `Scope.phn`, 14 tests on the real amp, PHPStan max; ✅ the FrankenPHP worker-mode check (400
  requests, the loop persisting between them); ✅ the plugin's token-level approximation.
  *Target:* `tq-cli` compiled from 859 `.phn` with its fifteen fan-outs as blocks — 2052 tests
  green on the output, PHPStan 10 and cs clean, `phn check` at 0 over `src` and `tests` (378 at
  the first build: checker gaps closed in three merges, real findings fixed in tq-cli);
  `tequila/nats` with `RevoltTransport`, the pump lease and an opt-in reader task, 175 tests
  against the server (tequila/nats#13, the ADR amended in tequila/platform#255). *Found on the
  way:* `?(A, B)` did not parse; the first capture rule wrote `&$path` on loop variables; closure
  types gained `T...` and `T=` (`generics.md` §11.3); numeric string keys are int keys
  (`checker.md` §9); `.` joins two `bytes` (`strings.md` §6); the plugin's stubs went stale on
  every upgrade, its first suspend function hung on Kotlin 2.3 metadata, and PhpStorm's Optimize
  Imports removed imports only erased constructs used — all fixed. *Open:* the plugin's
  expression forms (a PSI element, M6), `phn check` against `composer.json` for a missing
  `phaenon/concurrent`, `Scope::first` / `Scope::settle` when a use site asks, the checker's two
  parity gaps found on tq-cli's tests (vault `open-items`), a formatter for `.phn` in PhpStorm.

### PHP 8.6 — the first release milestone (started 2026-11-02)
- Design: `php-versions.md`: the target (D17), lowering or refusing what is newer than it, partial
  function application typed and lowered, pipe diagnostics, readonly defaults, `#[\Override]` on
  constants, the 8.6 deprecations, a library that knows each symbol's version, and `is` kept a
  superset of PHP's coming pattern match.
- Order: the target and the 8.6 grammar (lexer, parser, printer, conformance against the
  `PHP-8.6` corpus under PHP 8.6); the library by version and the non-PFA checker items; partial
  application in the checker and the emitter; the plugin.
- Target: before PHP 8.6's release (2026-11-19). The Tequila trees stay on target 8.5 and
  exercise the lowering; tq-cli also builds with `"php": "8.6"` and its suite runs on PHP 8.6.
- Closes on PHP 8.6.0's release day, once the plugin handles the 8.6 syntax:
  - `stdlib/constants.php` regenerated from the released build;
  - the conformance suites run once more against it.

### M6 — Tooling and distribution
- LSP server in the Go binary; VS Code + PhpStorm plugins; `phaenon fmt`; `phaenon migrate` v1
  (codemods: arrays → list/map where provable, string functions → methods).
- Composer plugin (D13: builds every installed package with a `phaenon.json` on install, `phn`
  pinned and shipped as a Composer package per platform), GitHub Action, Docker base image update
  in `operations-container-images`.
- Target: `php-service-template` ships `.phn` by default.

### M7 — Tequila on Phaenon
- Service by service: `iam` first (largest, most representative), then the rest via the template.
- Public: spec, playground (compile in the browser via WASM build of the Go compiler), docs site.

### Phase 3 (not planned) — native runtime
Only if the language exists, has a conformance suite, and a measured reason. The Go frontend is
reusable; the backend is a separate decision then.

## 4. Working method

- **Design doc → golden tests → implementation → PHPStan-on-output → dogfood.** Every feature.
- `design/decisions.md`: append-only log (date, decision, why, alternatives) — the table in §1 moves
  there once there are more than ten.
- **Oracle-driven correctness**: for everything that is "PHP", `php` itself is the test oracle
  (phpt harness). For everything Phaenon adds, golden files + hand-written semantics tests.
- **Token economy**: mechanical, parallelisable work (phpt classification, stub conversion, codemod
  corpora) goes to subagents; design and checker core stay in the main thread.
- The maintainers' work: design review and decisions, Tequila-side migrations, the bits of PHP
  semantics that need a person who has been bitten (and the ones the oracle can't settle: taste).

## 5. First steps (this week)

1. `rm ~/labs/phaenon/php-src/go.mod` — the stray module file in the reference clone.
2. Write `design/language-overview.md` (scope; 2–3 pages).
3. Bootstrap the `phaenon` Go module (`cmd/phn`); start the lexer from `Zend/zend_language_scanner.l` and
   the Pratt table from `zend_language_parser.y`.
4. Build the phpt round-trip harness before the parser is "finished" — it is the compass for M0.
5. Register `phaenon.dev` + the GitHub org; pick the licence.

## 6. Open questions

- In-place output (`.php` beside `.phn`, gitignored) vs separate output directory — affects
  Symfony's autoloading, PHPStan runs, Docker builds. Lean: in-place, gitignored, like `.js` beside `.ts`.
- Licence (MIT? Apache-2?), governance once it's public.
- Whether the `design/` docs are published as the spec from day one (recommended: yes, public repo,
  "RFC-style" docs — it costs nothing and makes feedback possible).
