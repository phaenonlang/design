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
| D3 | Generics: real `<T>` syntax, erased by default, `reified` opt-in compiled to the `class-string<T>` idiom; `is`/`as` operators | `generics.md` | |
| D4 | Emitted PHP carries PHPStan-vocabulary docblocks; PHPStan `level: max` on output is part of compiler CI | output must read as well-typed modern PHP | `generics.md` §4.2, §8.4 |
| D5 | Strings stay primitive; method-call syntax on primitives compiles to static calls (`"a"->upper()` → `\Phaenon\Str::upper("a")`) | 95% of the "string object API" win, no value-semantics change | conversation |
| D6 | Concurrency: structured concurrency over Fibers, **no function colouring** (no `async fn`) | PHP's synchronous readability is the point; Fibers already exist; purely syntax + stdlib loop | conversation; design doc pending |
| D7 | **Own parser**: hand-written lexer (state machine ported from `zend_language_scanner.l`), recursive-descent + Pratt expressions (precedence from `zend_language_parser.y`), typed AST node families, speculative parse for `<…>(` | z7zmey/php-parser evaluated 2026-10-28: PHP 7.4 only, goyacc LALR (no backtracking, poor recovery), untyped `Vertex` AST, dead since 2021 — nothing to keep. VKCOM/noverify excluded (Russian company) | — |
| D8 | Extension **`.phn`**, CLI **`phn`** (`phn build\|check\|watch\|fmt\|migrate\|lsp`), project **`phaenon`**, GitHub org **`phaenonlang`** (`phaenon` is taken), Composer vendor `phaenon/*` to be checked on Packagist | short and typeable like `tsc`; `.phn` reads as PHP-family; `phaenon.dev` being bought; org `phaenonlang` | — |

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

### M3 — Generics (per `generics.md`)
- Syntax, checker (bounds, variance, inference), erasure + docblock emission, override-erasure rule,
  `reified` + `phaenon/runtime` `\Phaenon\Type`, `is`/`as`, binding subclasses, Symfony DI warning.
- Target: `tequila/object-mapper`, then `tequila/mongodb-odm` (repositories = the reified case).

### M4 — Collections and strings
- `list<T>` / `map<K,V>` as first-class types with literals; `array` as a legacy type that
  interoperates (design doc: migration ergonomics for the array-as-both-things codebase).
- `\Phaenon\*` stdlib with consistent argument order, written in PHP (runtime package) with
  Phaenon signatures; string method syntax, `bytes` vs `string`, grapheme-aware unicode API over
  `intl`.
- Target: `tequila/core` (the heaviest array user).

### M5 — Concurrency
- Design doc: structured concurrency over Fibers (`concurrently { }` scopes, awaitables,
  cancellation, timeouts), event loop in the runtime package, interop with amp v3 and Symfony.
- Target: `tequila/nats` (sans-I/O core + non-blocking transport is the natural first adopter),
  `tq-cli` fan-outs.

### M6 — Tooling and distribution
- LSP server in the Go binary; VS Code + PhpStorm plugins; `phaenon fmt`; `phaenon migrate` v1
  (codemods: arrays → list/map where provable, string functions → methods).
- Composer plugin, GitHub Action, Docker base image update in `operations-container-images`.
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
- The maintainers' work: design review and decisions, Tequila-side migrations, the bits of PHP semantics that
  need a human who has been bitten (and the ones the oracle can't settle: taste).

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
