# Phaenon — Tooling and distribution (draft 0.1)

Status: **reviewed 2026-11-03** (decisions §14) · M6. Companion: `roadmap.md` D2, D8, D10, D12, D13,
D17, D19 and §3 (M6); `line-map.md` (positions in compiled code); `functions.md` §5 (the canonical
source layout).

This document is the design. §13 lists the questions of the review, §14 the decisions. Where it departs from what the
roadmap assumed (D13's "a Composer package per platform", M6's "GitHub Action" and "Docker base
image update") it says so and why.

---

## 0. One paragraph

M6 makes Phaenon something a project installs, not something a developer builds beside it. One
Composer package, `phaenon/phn`, brings the compiler binary and a Composer plugin. The plugin
compiles every installed package that has a `phaenon.json`, and the project itself, when Composer
writes the autoloader. So `composer require` is the whole setup, and a library ships only its `.phn`.

PHP's own tools keep working on Phaenon code because they keep working on the compiled PHP.
Positions already come back to the source through the line map (D19). With M6, *edits* come back
too: `phn fix` runs a rewriting tool such as php-cs-fixer or Rector over the compiled tree with the
project's own configuration, and carries its changes back into the `.phn`. `phn fmt` is Phaenon's
canonical layout plus that.

`phn lsp` puts the checker into editors:
- in PhpStorm, as the source of the findings `phn check` reports, beside the plugin's PSI, which
  keeps navigation, refactoring and debugging;
- in VS Code, as the whole of the language support.

CI annotations land on `.phn` lines. The target is `php-service-template` shipping `.phn`.

## 1. The facts that shape it

Read and measured on 2026-11-02.

**Composer** (2.9.5):
- **No per-platform dependencies.** npm's `os` / `cpu` fields with `optionalDependencies`, the
  model D2 named after esbuild, have no Composer counterpart. A package is installed on every
  platform or on none.
- **Packagist hosts no artefacts.** It builds every package from a VCS repository, so a package per
  platform would be a git repository with each release's binary committed into it.
- **Plugins run under `--no-scripts`, the autoload events do not.** That flag drops
  `composer.json`'s script listeners, not the event subscribers of plugins
  (`EventDispatcher::getListeners()`), but the autoload generator dispatches `pre-autoload-dump`
  only when scripts run: `install --no-scripts` builds nothing, and the `dump-autoload` that follows
  it in the template does (§3.3). `dump-autoload --dry-run` dispatches the event, and builds.
  `--no-plugins` turns plugins off.
- **An unlisted plugin is asked about or refused.** A plugin missing from `config.allow-plugins` is
  a question in an interactive run. In a non-interactive run the install fails
  (`PluginBlockedException`).
- **The autoloader needs the outputs first.** `dump-autoload --classmap-authoritative` and
  `--optimize` scan the PSR-4 directories while the autoloader is written, so the compiled outputs
  must already exist.

**`phn`**:

| Measure | Value |
|---|---|
| Binary | 8.8 MB, static (`CGO_ENABLED=0`); one machine cross-compiles every platform |
| `phn check` on core (800 `.phn`) | 2.1 s, 171 MB resident |
| `phn check` on one core file | 0.33 s |
| `phn build --check` on core | 1.4 s |
| `phn check` on tq-cli (859 `.phn`) | 1.6 s |

Vendor code is loaded lazily, and nothing is cached between runs.
- **No third-party Go modules.** `go.mod` has no requirements (`AGENTS.md`): no fsnotify, and the
  LSP's JSON-RPC is written on the standard library.
- **Sources over outputs, root project only.** The program model already reads the `.phn` behind a
  project's outputs (`ComposerLoader.Sources`), but only for the root project.

**Tequila**:
- **The template's Dockerfile builds in two steps.** Its vendor stage runs `composer install --no-dev
  --no-scripts --no-autoloader`, copies `src/`, `config/` and the rest, then runs `composer
  dump-autoload --classmap-authoritative --no-dev`. The base image is `php-builder:8.5`: php-cli,
  the extensions and Composer.
- **CI runs through reusable workflows.** They are `php-ci-checks.yaml` and `php-ci-tests.yaml` in
  `operations-github-actions`, mirrored into each org as `tequila-gha`. Composer authenticates to
  GitHub with an App installation token scoped to that mirror, and to `composer.tequila.dev` with
  the job's federated Tequila token.
- **Core's php-cs-fixer config rewrites code.** It is `@PSR12` plus the `@PHP80Migration:risky` to
  `@PHP83Migration` sets: `void_return`, `use_arrow_functions`, `modernize_strpos` and other fixers
  that change code, not only layout.
- **The template upgrades with Rector.** It runs core's upgrade sets (`tools/rector`, `rector.php`),
  which is how a service moves between core pre-releases.
- **`bin/init` only knows `.php`.** It rewrites placeholders in `*.php` under `src/`, `tests/` and
  `contracts/src/`.

**Editors**:
- **The checker stays on the command line.** The PhpStorm plugin gives `.phn` PhpStorm's PHP
  toolchain through PSI, but the checker's findings appear only in the terminal.
- **PhpStorm has an LSP client API** since 2023.2. The plugin builds against 2025.1.
- **VS Code has nothing for `.phn`.** Its PHP extensions (Intelephense, PHP Tools) read PHP files
  that start with the open tag.

**Repositories**: `phaenonlang/*` stay private until they are published under MIT, which happens
once Tequila runs on Phaenon in production (T1). Until then everything in this document reaches its
users through private channels: Tequila's package registry, a custom plugin repository, a `.vsix`.

## 2. Goals and non-goals

### Goals

1. A project adopts Phaenon with `composer require phaenon/phn`, and a library ships `.phn` only
   (D13) and is consumed with `composer require` alone.
2. Every PHP tool a project runs keeps working with its own configuration:
   - analysers and test runners on the outputs, positions mapped back (D19);
   - rewriting tools on the outputs, edits mapped back (§4).
3. The checker's findings show in the editor while typing, in PhpStorm and in VS Code.
4. CI annotations land on `.phn` lines.
5. The same `phn` and the same sources give the same output on every platform.

### Non-goals (M6)

- A formatter that replaces php-cs-fixer as the authority on PHP style (§5).
- Debugging `.phn` in VS Code (§6.3).
- A persistent cache of vendor declarations: §1's numbers say parsing is not the bottleneck. It
  comes back when a project shows the need, M7's services first.
- LSP features beyond §6.1's list: references, rename, semantic tokens, inlay hints.

## 3. Distribution

### 3.1 One package: `phaenon/phn`

- **Built from the compiler repository itself.** It is a Composer package of type
  `composer-plugin`: `composer.json` at the repository root, the plugin's PHP under `composer/`, and
  `.gitattributes` marking everything else `export-ignore`. The dist archive then holds the plugin
  alone, some tens of KB, without the Go sources. A compiler tag is a package version, and there is
  no second repository to keep in step. GitHub's source archives of a tag are made the same way and
  hold the plugin alone too, so a release also publishes the whole tree as `phn-X.Y.Z-src.tar.gz`;
  `go install …@vX.Y.Z` is unaffected (Go ignores `export-ignore` in module zips).
- **It provides `vendor/bin/phn`.** This is a small PHP launcher that runs the binary and passes
  through the arguments, the standard streams and the exit status. If the binary is missing on first
  use (after an install with `--no-plugins`), the launcher fetches it the way the plugin would.
- **It requires `phaenon/runtime` at the same minor version** (§3.4).
- **It goes in `require`, not `require-dev`.** The production build (`composer install --no-dev`)
  is where outputs are made. The running application never calls it.

### 3.2 The binary

- **One release asset per platform.** `phaenonlang/phaenon` releases carry `darwin-arm64`,
  `darwin-amd64`, `linux-amd64`, `linux-arm64` and `windows-amd64`. The Linux builds are static and
  run on glibc and musl (Alpine) alike.
- **The lock pins the binary.** The package carries each asset's SHA-256 (`composer/checksums.json`),
  so `composer.lock`, which pins the package, pins the binary too.
- **The plugin fetches and verifies it.**
  - It downloads the asset for the running platform through Composer's own HTTP client, which
    honours its `auth.json`, proxies and TLS settings.
  - It keeps the asset in Composer's cache directory and verifies it before every use.
  - It installs it into `vendor/phaenon/phn/bin/`.
  - A checksum mismatch fails the install.
- **Builds are reproducible**: `-trimpath`, the Go toolchain pinned in `go.mod`, no VCS stamping, and
  the version set by the linker. So the checksums can be known before the tag:
  1. The release PR's workflow builds the five binaries and commits their checksums into the PR.
  2. The tag's workflow builds them again, refuses to publish unless they match, and uploads them.

  Anyone can rebuild a release and compare.
- **Two overrides:**
  - `PHN_BINARY=/path/to/phn` replaces the download with another binary, such as a compiler built
    from a checkout (how the dogfood trees use one today, from `PATH`) or a distribution's package.
  - `extra.phaenon.binary-url` in the root `composer.json` names a mirror with the same file names,
    for builds that cannot reach GitHub. While the repositories are private (T1) this is how
    Tequila's CI and its tenants get the binary: from a mirror Tequila's registry serves, with the
    Composer credentials they already have. The mirror is the registry's release-asset door, which
    already serves `tq` and the Flex recipes behind the Composer login. The value is a template:
    `{version}` and `{file}` are replaced, and a value with neither is a base URL the file name is
    appended to.

**Why not a Composer package per platform** (D13's wording)? Composer cannot pick a package by
platform (§1), so a plugin would have to choose and fetch the binary anyway. On top of that,
Packagist would need one git repository per platform with every release's binary committed into it.

**Why not one package holding every binary?** It would make every install download about 45 MB
(5 × 8.8 MB).

Downloading release assets is what PHP's tools that wrap a Go or Rust binary already do: RoadRunner's
`rr get-binary`, the SymfonyCasts Tailwind and Sass bundles, Laravel Octane's FrankenPHP. They fetch
when first run; `phaenon/phn` fetches at install time and pins what it fetches. **(Amends D13.)**

### 3.3 Building at install

- **The plugin builds when Composer writes the autoloader.**
  1. It subscribes to `pre-autoload-dump`.
  2. It builds every installed package that has a `phaenon.json`, then the root project if it has
     one.
  3. Composer then writes the autoloader over outputs that exist, `--classmap-authoritative`
     included.

  An `install --no-autoloader` builds at the `dump-autoload` that follows, so the template's
  Dockerfile works in its present order (§7.3).
- **It writes the autoload maps first.** phn reads the vendor code through `vendor/composer`'s maps,
  which a first install has not written yet at `pre-autoload-dump` and an update has left stale. So
  the plugin dumps them once, plain (no optimisation, no scripts, the lock's suffix), and Composer's
  own dump then overwrites them with the outputs in place.
- **Nothing to build, nothing done.** Without an installed package or a root project that has a
  `phaenon.json`, the plugin neither fetches the binary nor runs it.
- **A compiler checkout is never written into.** With `phaenon/phn` a path repository onto the
  compiler's checkout, the binary is `PHN_BINARY` or the checkout's own `bin/phn`, the developer's
  build.
- **Installed packages are read from their sources.** The program model already maps a project's
  outputs back to its sources (`ComposerLoader.Sources`). That mapping extends from the root project
  to every installed package with a `phaenon.json`. A package's build then sees its dependencies'
  generic signatures as written, and the order of the builds does not matter.
- **An installed package's findings never withhold its output.** They are printed as warnings, as
  `--emit-on-error` does.
  - D10 is a rule for the code a project owns. A library's own CI already gave its verdict, and the
    consumer's `phn` may be newer and stricter than the one the library was released with.
  - The root project builds exactly as `phn build` does.
- **Each package builds for its own target** (D17: its `phaenon.json`, otherwise its composer.json's
  `require.php`), the target its CI tested.
- **Unchanged packages are skipped.** A package is rebuilt only when its sources, the `phn` version
  or its target changed since the last build, or the installed set did, which a stamp in each of its
  output directories records (`.phn-build`: the compiler's version and the digest of its binary,
  the target, the digest of `installed.json`, the digest of the root's files). A
  `dump-autoload` on a warm tree then costs nothing. A full build costs little anyway: core's 800
  files take 1.4 s.
- **`phn build --vendor`** builds the installed packages by hand, as the plugin does before
  `phn build` builds the root. A package's program reads the vendor code through the root project's
  `vendor/composer`, not through a vendor directory of its own (a path repository's checkout has
  one), and a root its dist archive leaves out (`tests/`) is skipped.
- **The root project's build can be turned off** with `extra.phaenon.build: false`, for a project
  whose own workflow builds it (Q4).

### 3.4 Versions

- **Lockstep until 1.0.** `phaenon/phn` X.Y.Z requires `phaenon/runtime` X.Y.* and suggests
  `phaenon/concurrent` X.Y.*, and `phn stdlib` regenerates both from the same tag. One compiler
  builds the whole vendor tree at install, so the runtime always matches the compiler that wrote
  every call into it.
- **A library states the compiler range it needs.** It requires `phaenon/phn` with the range its
  sources compile with (`^0.4`), as a PHPStan extension requires PHPStan.
- **`phn check` reads `installed.json`** and reports two errors, `P4013` and `P4014`:
  - a file that uses `concurrently` where `phaenon/concurrent` is not installed (the vault's M5
    item);
  - an installed runtime package of another minor version.

### 3.5 The three in-place libraries

reflector, object-mapper and mongodb-odm still commit a `.php` beside each `.phn`, because their
dogfood branches predate D12 and D13. They move to the sibling layout and drop the committed PHP, and
core's dogfood branch consumes them as path repositories through the plugin. That is the first proof
of D13.

Done in slice 1 (2026-11-03): the three moved, and core's dogfood branch takes them as path
repositories of their dogfood checkouts, built by the plugin at every autoload dump; core's suite,
PHPStan max and php-cs-fixer pass on them, a `--classmap-authoritative` dump with their outputs
deleted included.

## 4. PHP's rewriting tools: `phn fix`

D19 carried positions from the compiled PHP back to the source. M6 carries *edits* back the same
way. A tool that rewrites PHP — php-cs-fixer, Rector, PHP_CodeSniffer's `phpcbf` — then keeps
working on a Phaenon project with the configuration the project already has, and runs where that
configuration already points: the output roots.

```sh
phn fix -- tools/php-cs-fixer/vendor/bin/php-cs-fixer fix
phn fix -- tools/rector/vendor/bin/rector process
```

1. **Build and remember.** `phn build` writes the outputs and keeps, for each, the emitter's list of
   **verbatim spans**: the regions copied from the source byte for byte, with their offsets on both
   sides. The emitter is patch-based, so these spans are simply the complement of its edits; nothing
   new is computed.
2. **Run the tool.** The command runs as given and rewrites outputs.
3. **Map the changes back.** Each rewritten output is diffed against what `phn build` wrote.
   - A change inside one verbatim span, or between two spans with nothing generated in between, is
     applied to the source at the mapped offset.
   - A change that touches generated text is not carried back. Generated text is a docblock the
     compiler wrote, the `public function` of a `fun`, an erased type or an import the compiler
     added. The change is reported with its position traced to the source:
     `src/Foo.phn:12: php-cs-fixer changed generated text (phpdoc_align); not applied`.
4. **Rebuild and check.** `phn build` runs again, so the outputs are compiled sources once more.
   Where the tool has a check mode, the command then runs a second time in it, and any change it
   still wants is reported. A tool and the compiler therefore cannot loop.

A file the command rewrites outside the output roots is a real file and keeps the change. Examples
are Rector's `config/` and `rector.php` itself.

What is never carried back is narrow on purpose. Generated text belongs to the compiler, and its
layout is already what PER Coding Style wants (D19, D22). A tool that still wants to change it
reveals either a compiler bug or a configuration choice, and the report makes either visible.

**The project's fixer follows PER** (D22, 2026-11-04): `@PER-CS3x0`, or a later PER set. The
compiled layout is PER Coding Style 3.0's, and PSR-12 contradicts it in one place that is the
source's text: an empty body `{}`, which `@PSR12` expands to `{` and `}` on lines of their own.
`phn fix` would carry that expansion into the `.phn`, against `phn fmt`'s layout. `@PSR12` also
wants `fn (` where the compiler writes `fn(`.

**Text a lowering moves is carried back too** (2026-11-04). A folded pipe (`php-versions.md`
§5.3) writes its operands, the programmer's text, into a call of the compiler's layout. The
source's runs in that text are verbatim spans where they land, out of the source's order, so a
change of an operand is carried like any other; the indentation the fold gives a continuation
line is the compiler's. The carried changes are applied in the source's order, and a deletion
across spans whose source range holds text that lands elsewhere is refused. Before, `@PER-CS3x0`
on core and tq-cli left 13 and 117 changes (`.` spacing, `fn (`, trailing commas) uncarried in
folded pipes; now none, and one `phn fix` converts either tree to PER with nothing left after
the rebuild.

**Measure first.** Before §5 builds on this mechanism:
1. Un-format core's `.phn` by running a php-cs-fixer config with other spacing rules through
   `phn fix`.
2. Format it back the same way.
3. Count the changes carried back and refused.
4. Run one of core's Rector upgrade sets the same way and count again.

If the refused share is not small, `phn fmt` ships its first layer alone (§5).

Done in slice 3 (2026-11-03), with these settlements:
- **The verbatim spans are the emitter's copied segments**, which the line map's property test
  already tracked: `emit.FileVerbatim` returns them with the output. `phn fix` builds as
  `phn build --emit-on-error` does, without printing findings, and keeps them in memory; nothing is
  written beside the outputs.
- **A change is found by aligning tokens.** Both outputs are lexed as PHP and aligned on their
  tokens other than whitespace. A change is the text between two tokens both keep: a respaced gap,
  a rewritten token or a run of them. It is narrowed to the bytes that differ. A change is carried
  whole or not at all.
- **What is carried:**
  - a change that lies within one verbatim span;
  - a run of a repeated byte, such as indentation, that can slide into one span;
  - a deletion whose two ends lie in verbatim text, which takes the source between them along: a
    method Rector removes goes with the head and the body the compiler wrote for it.

  A replacement across compiler text is refused, since its new text is PHP the `.phn` may not
  take. A copied plain-PHP file is a source and takes every change. A file the tool creates or
  removes in an output root is reported, and the rebuild undoes it.
- **Rounds instead of a check mode.** Some compiler text follows the source: the indentation of
  an expression body's continuation lines, the `return` line, the blank line after the header. A
  change of it is refused while the source is still the old one, and becomes the source's own
  change once the rebuild has written that text anew. So `phn fix` rebuilds and runs the tool
  again, carries what has become the source's, and stops after three rounds or when a round
  carries nothing.

  The last run reports what the tool still changes, at its source line, and the outputs are put
  back as built. Running the tool itself needs no knowledge of its check mode, so any tool fits.
  The exit status is 1 when something is left, or the tool's own status when it failed.
  `--check` writes no source and reports every change: the CI form of the second layer of
  `phn fmt` (§5.3).
- **A reported change names its rule** (2026-11-04) when the tool's own report says which rule
  it was: `src/Box.phn:7: php-cs-fixer (binary_operator_spaces) changes: "" → " "`.
  - **Asking for the report.** `phn fix` adds the options itself, unless the command already
    picks a report format: `--format=json -v` to php-cs-fixer's `fix`, `--output-format=json` to
    Rector's `process`. It reads the report from the tool's standard output and doesn't print
    it, unless the tool fails.
  - **Rector** names each change's rector with the line of the node it rewrote. A change goes to
    the rector whose node, starting on that line, is the innermost that encloses the change.
  - **php-cs-fixer** names only the fixers applied to each file. A change in a file one fixer
    changed is that fixer's. Otherwise, before printing, `phn fix` runs each candidate fixer
    alone (`--rules`, the project's configuration otherwise) over copies of the outputs, and
    names the fixers that alone make exactly that change. One run per candidate fixer, not per
    change: on a sample with four fixers, the whole report took 1.9 s.
  - **Otherwise the report says "one of"** and lists the file's rules. That happens when no
    single fixer makes the change alone (a configured option that differs from the default),
    or when several rectors name the same node.
  - Any other tool's changes are reported as before, without a rule.
- **A method's modifiers stay the source's text** when they are already in PSR-12 order: the
  emitter replaces only `fun`. The output is the same byte for byte (core's 800 and tq-cli's 859
  outputs), but Rector's removal of a `static` is now carried. It used to be one of 344 refusals.
- **The measure, on a copy of core's tree:**

  | Run | Carried | Refused in round 1 | Left after the rounds |
  |---|---|---|---|
  | php-cs-fixer, other spacing (tabs, same-line braces, `( spaced )` parentheses, `) : type`, `declare( x = 1 )`, closure spacing, two blank lines before `namespace`) | 74,844, then 1,246 and 9 | 10,102 | 8,585, all compiler text this configuration lays out against PSR-12 |
  | php-cs-fixer, core's configuration, back | 74,087, then 451 and 8 | 5,060 | **0**: exit status 0 |
  | core's `1.0-alpha.15` Rector upgrade set | 1 | 0 | 0 |
  | Rector's dead-code, code-quality and type-declaration sets | 2,450 in 229 files | 25 | 3: types Rector adds to the closures the compiler writes for pipelines |

  With the project's own configuration nothing is refused in the end. The round-tripped `.phn`
  equals the original except in two kinds of layout the compiler lays out itself, so the fixer
  never sees them:
  - a blank line before the first line (794 files), which the header absorbs;
  - a class-like brace left on the declaration's line (756), which the compiler moves to its own.

  Both are Phaenon layout, `phn fmt`'s first layer (§5.1). **The gate passes:** `phn fmt` ships
  both layers.

## 5. `phn fmt`

`phn fmt` has two layers.

### 5.1 Phaenon's layout

The first layer is native, with no options:
- the canonical layout of `functions.md` §5: braces, `{}` for empty bodies, one-line docblocks;
- spaces for the tabs that `functions.md` §5.5 forbids (D21, 2026-11-04), before anything else,
  so every later step works on spaces;
- the spacing of Phaenon's own constructs, which no PHP tool knows.

| Construct | Canonical spelling |
|---|---|
| type parameters | `class Box<T of Item>`, `fun first<T>(list<T> $xs): ?T` |
| type arguments | `list<User>`, `map<string, list<int>>`: no space inside `<>`, one after a comma |
| tuple and closure types | `(int, string)`, `Closure(int, string...): bool` |
| expression body | `fun name(): string => $this->name;`, a continuation one level in |
| `throws` | `fun find(Id $id): User throws NotFound\|Conflict {` |
| `is`, `as` | `$x is list<int>`, `$x as? User` |
| `concurrently`, `spawn` | `concurrently (timeout: 5) {`, `spawn $job->run()` |

This layer changes whitespace only, plus the brace and `{}` forms, which are layout too. Turning a
block into `=> expr` stays `phn migrate`'s job, because it changes the code.

PhpStorm's Reformat Code writes this same layer (the plugin's formatter), and the two share their
golden files.

### 5.2 The project's PHP style

The second layer runs through `phn fix` with the project's fixer:
- named in `phaenon.json` as `"fmt": ["tools/php-cs-fixer/vendor/bin/php-cs-fixer", "fix"]`;
- or found where the template and core keep it.

A project without a fixer gets the first layer only.

### 5.3 How it runs

- **Commands:** `phn fmt` formats the roots, `phn fmt FILE…` given files, and `phn fmt --check` runs
  both layers for CI without writing anything.
- **php-cs-fixer on the outputs** stays the check it is today. `phn fmt` is what makes the source
  pass it without editing by hand, and Composer's `cs` script becomes `phn fmt`.
- **A fixer that undoes the first layer is named** (2026-11-04): when the first layer has to lay
  a carried change out again — `@PSR12` expanding an empty body `{}` — `phn fmt` names the files,
  rebuilds the outputs from the sources it kept in Phaenon's layout, and fails, since the layers
  would disagree on every run. A PER set keeps the layout (§4, D22).
- **In editors**, the LSP's formatting request runs the first layer, which takes milliseconds. The
  second layer costs a PHP process per run, so it runs on an explicit command.

### 5.4 Done in slice 3 (2026-11-04)

With these settlements:
- **The first layer is migrate's layout without the bodies**, plus the spacing of Phaenon's own
  syntax and no blank line before the first line (`internal/format`):
  - braces, `{}` and one-line docblocks come from `migrate.LayoutBlocks`, which `phn migrate`
    runs too. Turning a block into `=> expr` stays migrate's.
  - The spacing touches only the gaps within a line and leaves a construct with a comment inside
    as it is. It covers:
    - type parameter lists, type arguments (on types, calls and `new`), tuple and closure types,
      and type aliases;
    - `throws`, `is`, `as` and `as?`;
    - an expression body's `=>`;
    - `concurrently` and `spawn`.

    The tokens must come out as they went in (a `>>` counts as the `>` `>` it closes).
  - The rest of a `.phn` is the project's PHP style, which the second layer writes.
- **What the first layer leaves to the second:**
  - the indentation of an expression body's continuation lines, which php-cs-fixer sees in the
    output and `phn fix` carries back in its rounds (§4);
  - the spacing of PHP syntax around Phaenon's, such as `list<T>  $items`.
- **The second layer is `phn fix` with the project's fixer**, in this order:
  - phaenon.json's `"fmt"` (`[]` turns the layer off);
  - `tools/php-cs-fixer/vendor/bin/php-cs-fixer fix`, where the template and core keep it;
  - `vendor/bin/php-cs-fixer fix`.

  `phn fmt FILE…` hands the fixer those files' outputs. The first layer runs again after the
  fixer, which can leave what only it lays out.
- **`phn fmt --check`** reports the files out of the first layer's layout and every change the
  fixer would make at its `.phn` line, writes nothing, and fails on either.
- **Proof:**
  - on the measure's copy of core, the tree un-formatted through `phn fix` (§4) comes back from
    one `phn fmt` equal to the original, byte for byte, all 800 files;
  - the first layer changes nothing in the five dogfood trees (1,853 `.phn`), and formats each
    file to itself a second time;
  - core's and tq-cli's `composer cs` became `phn fmt`, `cs:check` `phn fmt --check`, and both
    pass.
- **PhpStorm's Reformat Code** writes the first layer with the editor work (§6.2). Its golden files
  are `internal/format/testdata` here. Done in slice 4 (§6.5), which also fixed the braces of
  one-statement blocks in the first layer.

### 5.5 Why not the alternatives

- **A native formatter for all of PHP?** php-cs-fixer's configurations are where PHP projects keep
  their style, and most of core's is migration fixers that rewrite code. A second style authority
  would disagree with the first on the very output that first one checks.
- **php-cs-fixer on the `.phn` itself**, with an open tag prepended and Phaenon's syntax masked as
  comments? The fixers move and respace comments, and every one of them would be looking at code
  that is not the code. On the compiled output, their assumptions hold.

## 6. Editors

### 6.1 `phn lsp`

**How it runs:**
- **One server per project.** It speaks LSP over stdio, with JSON-RPC written on the standard
  library. The project is the nearest `phaenon.json`, otherwise `composer.json`.
- **The project's own compiler.** The editor starts the server from the project's `vendor/bin/phn`,
  so it checks with the compiler the project builds with.
- **The program model stays in memory:** the project's sources, vendor code loaded lazily as
  `phn check` loads it, and the open buffers.
- **An edit re-checks its own file** against that model: 0.33 s cold today, less once warm.
- **A save re-checks the whole project** in the background (2 s on core) and publishes every file's
  diagnostics. The next save cancels a check still running.

**Features in M6:**

| Feature | What it gives |
|---|---|
| Diagnostics | the checker's codes and messages, with "declared here" as related information |
| Hover | the type in Phaenon's spelling, and the docblock |
| Go to definition | into `.phn`, vendor `.php` and the stubs |
| Document symbols | the file's outline |
| Completion | members; names in scope; classes with their import; after `\|>`, the functions whose first parameter takes the left side's type, inserted as `f(?, …)` |
| Signature help | the parameters of the call being typed |
| Formatting | §5's first layer |
| Build on save | see below |

**What the checker needs for this.** It grows two queries: the type at an offset, answered by
re-checking the enclosing function with a probe, and the symbol at an offset. Checking one file
must also become separable from checking the project; it nearly is already, since files are checked
one by one against the declarations.

**Build on save.** When the editor asks for it (a setting, on by default in the plugin), the server
writes the outputs after each save, as `phn build` would:
- **The whole project is built, but only changed outputs are written.** A file's output depends on
  other files' declarations — a parent's signature for the override-erasure rule, a called
  function's parameters for a partial application lowered for 8.5 — so building one file alone
  would be wrong.
- **Files with errors are withheld** (D10).

So a PHPUnit run from the IDE, or a FrankenPHP worker in dev, sees the source that was just saved.

**Later:** references, rename, workspace symbols, semantic tokens, inlay hints for inferred types,
and code actions beyond a diagnostic's own fix.

### 6.2 PhpStorm

- **Diagnostics only from the server.** The plugin starts `phn lsp` through PhpStorm's LSP API for
  **diagnostics and their quick-fixes only**. Its PSI keeps everything else: navigation, completion,
  refactorings, formatting, debugging. The server never answers a question PhpStorm answers.
- **Duplicates show twice at first.** Where a PhpStorm inspection and a checker finding report the
  same thing, both show for now; the plugin's highlight filter learns the pairs from the dogfood
  trees.
- **Build on save is on by default**, with a status-bar item for the last build.
- **Distribution.** A custom plugin repository (an `updatePlugins.xml` URL in PhpStorm's settings),
  so updates arrive without installing zips by hand; the JetBrains Marketplace once the repositories
  are public (T1).
- **The `.php`-beside-`.phn` case** from the vault: in the adoption layout without `phaenon.json`, a
  `FileTypeOverrider` hides from the project a `.php` that has a `.phn` sibling and a `@generated`
  header.

### 6.3 VS Code

- **A new repository, `phaenonlang/vscode`**, holding:
  - the TextMate grammar the plugin bundles;
  - a language configuration (brackets, comments, indentation);
  - an LSP client, `vscode-languageclient` (the standard-library-only rule is the compiler's, not
    the extension's);
  - commands to build, to check, and to run the selection or the terminal's text through
    `phn trace`.
- **VS Code users get exactly §6.1's list.** Unlike PhpStorm, VS Code has no PHP engine for `.phn`
  beside the server.
- **Not in M6: debugging.** vscode-php-debug's path mappings map directories and cannot change the
  file extension, the same problem PhpStorm had (`line-map.md` §1). Debugging would need a debug
  adapter in front of Xdebug's that maps positions through the line map, a piece of work of its own.
- **Publication waits for the repositories to be public**, both to the VS Code Marketplace and to
  Open VSX. Until then the extension ships as a `.vsix`.

### 6.4 `phn watch`

`phn watch` builds and checks on change, for terminals and for editors without the server:
- it uses the same engine as the server's build on save;
- it polls the roots' modification times every 300 ms, since the standard library has no file
  events and a few thousand `stat` calls per poll are cheap;
- it prints findings as `phn check` does.

### 6.5 Done in slice 4 (2026-11-04)

With these settlements:

**The server keeps no model between checks** (`internal/lsp`; the project side is `cmd/phn/lsp.go`):
- **A program per check, made cheap.** Every check builds a program of the whole project: the open
  buffers as they are, the other files as on disk. Three things are kept between programs
  (`cmd/phn/share.go`):
  - the parsed trees (`program.ParseCache`): a file whose text is unchanged is not parsed again;
  - the extensions' stubs, since asking `php` for them took 352 ms of a one-file check;
  - Composer's maps, read again when a file of `vendor/composer` changes.

  Nothing writes into a tree after the parse, so programs share trees, also across goroutines.
  Replacing a file in a living model was the alternative: the model is built to be added to, and
  its lazily built signatures and loaded vendor classes would all need invalidating.
- **Measured on core (800 `.phn`) and tq-cli (859):**

  | Check | core | tq-cli |
  |---|---|---|
  | First diagnostics of an opened file (`php` asked for its extensions) | 0.63 s | 0.77 s |
  | An edit's file, after the 150 ms pause in typing | 50 ms more | 50 ms more |
  | The whole project on save, published | 1.06 s | 1.26 s |

  `phn check` on core takes 2.1 s; the parse cache is the difference.
- **The project** is the nearest directory above the working directory with a `phaenon.json`, else
  with a `composer.json`. Without a `phaenon.json` only the open files are checked, and nothing is
  built.

**Answers come from the newest check**:
- **The checker's two queries became what the check records.** Instead of a probe re-checking the
  enclosing function, the check records the type it gave every expression of the edited file
  (`checker.Options.Observe`). Hover, go to definition and member completion read those types and
  the program model (`symbolAt`).
- **Completion and signature help check a copy of the buffer.** The word at the cursor becomes a
  placeholder name, closed with up to two brackets and a `;` until the text parses:
  `$this->repo->` is checked as `$this->repo->__phnProbe;`.
- **An unfinished line elsewhere takes nothing away.** When the buffer does not parse, hover, go to
  definition, and the completion of names and variables use the newest version that did.
- **Class completion offers what Composer can load.** Its source is the classmap plus one walk of
  the PSR-4 directories per set of maps (`ComposerLoader.ClassNames`). A class the file doesn't
  import comes with its `use`.
- **Hover prints Phaenon's spelling** (`types.PrintPhaenon`): `map<string, User>`, `class<T>`,
  `?User`, `(int, string)`, `$this` as its class.
- **Go to definition into the stubs** opens a read-only copy written under the user's cache
  directory.

**Diagnostics:**
- **"declared here"** is related information (`checker.Diagnostic.Related`). It points at the
  parameter an argument doesn't fit (P2103) and at the method an override breaks (P2502).
- **The quick-fix of P1005** replaces the tabs (D21).

**Build on save**:
- **After a save only.** Opening the project or a change on disk re-checks without building. A
  save's build waits for the check that finally completes, so a later save or change doesn't lose
  it.
- **Built from what is saved.** With an unsaved buffer open, the outputs come from a second program
  of the disk alone; otherwise the save's check is built as it is (`buildJob` takes a checked
  program).
- **Reported with `phaenon/build`**, a notification of the server's own: files, outputs written,
  files withheld, duration.

**PhpStorm** (`../jetbrains`: `PhaenonLsp.kt`, `PhaenonCompiler.kt`, `PhaenonLayout.kt`):
- **Diagnostics only, by what the server advertises.** The plugin passes `"features": "diagnostics"`,
  and the server then advertises diagnostics and code actions alone. Overriding the descriptor's
  support properties is not enough: PhpStorm 2026.2 replaced them with an `LspCustomization`
  whose defaults turn on every feature the server advertises, from document symbols to inlay
  hints. PhpStorm 2025.1, which the plugin builds against, reads the properties, which the plugin
  sets as well.
- **The LSP part is optional**, on `com.intellij.modules.ultimate`, the module both 2025.1 and
  2026.2 declare for the API.
- **The compiler** is the setting (Settings | Languages & Frameworks | Phaenon), else
  `$PHN_BINARY`, else the project's `vendor/bin/phn`, else `phn` on the PATH. The same page
  turns build on save off. The status bar shows the last build.
- **Reformat Code is PhpStorm's PHP style plus the first layer.** It runs `phn fmt --stdin` twice:
  - before the PHP formatter, which wraps lines by their length, so it measures the spacing the
    layer writes (`map< string,list<int> >` made a signature wrap);
  - after it, as an edit of its own right after the reformat.

  Only a reformat of the whole file runs it, never one of a range (after completion, an inserted
  import, Implement Methods).
- **The shared goldens:** Reformat Code turns the input and the golden into the same text. That
  text is in Phaenon's layout, and it differs from the golden in whitespace only. It isn't the
  golden itself: `phn fmt`'s first layer leaves PHP's own spacing around Phaenon's syntax to the
  project's fixer (`list<T>  $items`), and PhpStorm's PHP style tidies it.
- **Spaces, never tabs** (D21): the plugin's indent options for a `.phn` turn tabs off and
  override the PHP language's own options, as `.editorconfig` does. The PHP formatter's blocks
  read the language's options otherwise.

**`phn watch`** checks and builds at start and after each change, polling every 300 ms
(`--interval`), and prints findings as `phn check` does. `--no-build` only checks.

**Found on the way:**
- **`phn fmt` left a one-statement body's brace on its own line.** The layout pass leaves that
  brace to the arrow that migrate's second pass writes, which `phn fmt` never runs. The first
  layer now treats every block as a block (`migrate.LayoutBlocks`; golden `format/blocks.phn`).
  The dogfood trees were all migrated, so their bodies were arrows already, and nothing changed
  there.
- **Paths are matched under every spelling of the root.** macOS's `/var` is `/private/var`: a
  buffer named through one spelling was a second copy of the project's file, declared twice
  (P2511). URIs go back to the client in its own spelling.

## 7. CI and containers

### 7.1 Annotations and reports

- **`phn check` writes CI formats.** `--format=github` writes GitHub's workflow commands
  (`::error file=src/Foo.phn,line=12,endLine=12,col=5,endColumn=9,title=P2103::…`), and
  `--format=checkstyle` serves other CI systems. `--format=json` is the `--json` of M2, which
  stays as its short form.
- **`phn trace` reads three more formats:**
  - GitHub workflow commands, which PHPStan writes with `--error-format=github`;
  - Checkstyle XML;
  - JUnit XML, which PHPUnit writes with `--log-junit`.

  This matters because GitHub drops annotations on files that are not in the repository. The
  compiled outputs are gitignored, so a PHPStan finding on compiled code that is not traced is not
  shown at all.
- **Coverage.** `phn trace` translates Clover and Cobertura XML too:
  - every compiled line counts as the source line its forward lookup gives, holds included;
  - the compiled lines of one source line merge into one, covered when any of them is, with the
    largest of their counts;
  - the totals beside the lines (Clover's metrics, Cobertura's line rates) follow the merge.

  PhpStorm's coverage view, with the plugin reading the map, comes with §6.2 if the platform allows
  it.
- **`phn build --check`** passes when its only failures are files built `--emit-on-error` whose
  outputs are current. On core these are the 9 files with deliberate findings (vault `open-items`).

Done in slice 2 (2026-11-03), with these settlements:
- **Holds count in coverage.** The draft left them out as generated text. A coverage run shows
  otherwise. The executable lines a coverage driver reports on holds are code the source line is:
  - an expression body's `return …;`;
  - an empty body's closing brace (`fun __construct(…) {}`);
  - the continuation lines of a folded pipeline.

  Generated text that is not code (the header, imports, docblocks) is never executable, so it never
  shows up. On reflector, 5 of 73 statements sit on holds, and leaving them out would drop those
  source lines from the report. On tq-cli, 507 of 26,272 compiled lines merge into the source
  lines they came from.
- **The largest count, not the sum.** PHPUnit's count is the number of tests that ran the line.
  The compiled lines of one source line are mostly run by the same tests, so a sum would overstate,
  and the largest count is the one figure the merge knows to be true.
- **Columns are dropped where a line is mapped**: a workflow command's `col` and `endColumn`, a
  Checkstyle error's `column`. The map holds lines, and on an edited line a compiled column is not
  the source's.
- **An annotation names its file by its path in the repository**: from the root of the git
  repository that holds it (the nearest directory above the file with a `.git`), not from the
  working directory. GitHub matches the path against the repository, and in a project kept in a
  subdirectory of its repository the working directory is not the root. In a tree without git,
  which `actions/checkout` downloads where git is missing, the path starts at `GITHUB_WORKSPACE`.
  `phn trace` names the file of each command it rewrites the same way. A Checkstyle report keeps
  its paths relative to the working directory.
- **A report is told by its root element.** `phn trace` reads stdin a line at a time, as before,
  unless the input opens as XML. Such input is read whole:
  - `checkstyle` is Checkstyle;
  - `testsuites` or `testsuite` is JUnit;
  - `coverage` with a `project` is Clover, any other `coverage` is Cobertura.

  Other XML is text. The document is written back byte for byte except what points into a
  compiled file with a map. A rewritten tag is escaped as libxml2, which writes PHPUnit's and
  PHPStan's reports, escapes it.
- **Cobertura's `<source>` becomes the directory that holds both trees** (the project root for
  `.src/` → `src/`). Every class's filename is rewritten relative to it, so the filenames read as
  repository paths, which is how GitLab matches them. A package or class that PHPUnit named after
  its file is renamed with it.
- **What stays the compiled file's:** Clover's `loc` and `ncloc`. A class's Clover metrics follow
  the merge only when its file declares one class: Clover does not say which class a line belongs
  to.
- **Proof on the dogfood trees:**
  - tq-cli's suite, run under PCOV in a throwaway container:
    - all 2,054 JUnit test cases land on their `fun` line;
    - of the 449 positions in the failure texts, 418 now point into a `.phn` and 31 into vendor
      PHP, none into a compiled tree;
    - all 569 Cobertura classes resolve to a `.phn`, and the root's totals equal the sum of the
      lines.
  - Core: `phn check --format=github` writes its 19 findings as annotations, and
    `phn build --check --emit-on-error` passes with its 9 files.
- The P4xxx checks on installed packages, which §11 put in this slice, came with slice 1 (§3.4).

### 7.2 GitHub Actions

**No separate action.** With the plugin, every project has `vendor/bin/phn` after `composer
install`, so nothing else needs to install it.

**Tequila's reusable workflows** (`operations-github-actions`: `php-ci-checks.yaml`,
`php-ci-tests.yaml`) gain three steps for a project with a `phaenon.json`:
- `phn check --format=github`;
- `phn fmt --check`;
- PHPStan and PHPUnit with their annotations passed through `phn trace`.

A `setup-phn` action waits until a project runs `phn` without Composer. **(Amends roadmap M6's
"GitHub Action".)**

### 7.3 Containers

- **The base images need no change.** The binary arrives with `composer install` and stays in
  BuildKit's Composer cache mount between builds.
- **A service's Dockerfile** copies `phaenon.json` and the source roots before `dump-autoload`, which
  builds them. Its `.dockerignore` keeps local outputs out of the build context.
- **The image ships the `.phn` beside the outputs.** The Sentry integration reads its context lines
  from them (`line-map.md` §4.3).

**(Amends roadmap M6's "Docker base image update".)**

## 8. `phn migrate` v1: codemods

Each codemod is a named rule. A rule rewrites only where its condition is proven, runs on `.php`
and `.phn` alike, and is checked afterwards: if a rewrite changes a type the checker infers
anywhere in the file, or adds a finding, it is undone for that file and reported.

### 8.1 `pipes` — on by default (D20)

Nested calls become a pipeline, PHP 8.5's `|>` with PHP 8.6's partial application:

```php
$emails = array_values(array_filter(array_map(fn (User $u) => $u->email, $users), fn (?string $e) => $e !== null));
// →
$emails = $users
    |> array_map(fn (User $u) => $u->email, ?)
    |> array_filter(?, fn (?string $e) => $e !== null)
    |> array_values(...);
```

**What is a chain.** A call, a call in one of its arguments, a call in one of that one's, and so on.
- **Length.** It must be at least three calls long; `--pipes=2` lowers that to two, and
  `--keep-calls` turns the rule off.
- **Why three.** Two calls with an outer call that does not transform the data read worse as a
  pipe: `sprintf('%d items', count($xs))`, `max(0, min(10, $x))`.

**How it is written.**
- **A pipeline follows the data.** A stage takes the piped value into its first parameter, or into
  the data parameter of PHP's functions that take it after their other arguments: `array_map`,
  `implode`/`join`, `explode`, `mb_split`, the `preg_` functions, `str_replace`/`str_ireplace`,
  `in_array`, `array_search`, `array_key_exists`. A call that takes the value anywhere else ends
  the chain there: `sprintf('%d items', count($xs))` and `self::assertSame($expected, f($x))` stay
  calls, as does `max($width, strlen($s))`.
- **The head.** It is the innermost call's subject when that call has exactly one argument other
  than literals, constants and closures, and the call may take it from a pipe (`array_map(fn …,
  $users)` → `$users |> array_map(fn …, ?)`, `trim($s)` → `$s |> trim(...)`). Otherwise the head
  is the innermost call itself.
- **A stage that takes only the piped value** is a first-class callable, `f(...)`.
- **Any other stage** is a partial application with `?` where the piped value stood: positional,
  or `name: ?` where the call names its arguments.
- **Layout.** Three or more stages, or a line that would pass 120 columns, put one stage per line,
  `|>` first, one level in. A shorter pipeline stays on one line.

**When a chain is left alone.** It is not rewritten when the rewrite would not be exact:
- **The order of evaluation would change.** A pipeline evaluates the piped value before a stage's
  other arguments and before a method's receiver; the nested call evaluated them in source order.
  So every argument written before the nested one must be a literal, a constant, a variable the
  inner calls do not assign, or a closure that captures none by value. The receiver of a method
  stage must be `$this`, such a variable, or a property of either that its class declares without
  hooks and the inner calls do not assign (`… |> $this->output->writeln(...)`). A method the
  inner calls run could still reassign that property: the codemod takes it as not happening, and
  the compiler, which stays exact, keeps such a stage a pipe stage on 8.5 (`php-versions.md`
  §5.3).
- **The piped parameter is by reference** (`sort`, `preg_match`'s `$matches`): PHP refuses
  `|>` there.
- **The nested argument is spread or unpacked**, or follows a spread.
- **The chain is in a constant expression**, or a call in it is `compact()`, `extract()` or
  `func_get_args()`.

**What PHP makes of it.** A pipeline stage that is a first-class callable or a partial application
compiles to a direct call (php-src's `pipe_optimization_*` tests), so stack traces gain no closure
frames. For an 8.5 target a pipe that PHP evaluates as the call it makes is written as that call,
a run of them as the nested calls (`php-versions.md` §5.3): every pipeline the codemod writes
compiles back to the calls it was written from, except at a property receiver, which stays a pipe
stage.

**Measured and applied (2026-11-03).** The `.phn` of the five dogfood trees, the compiler's `exclude`
left out:

| Tree | Pipelines | Left as calls | Most frequent reasons |
|---|---|---|---|
| reflector | 0 | 6 | not the data parameter |
| object-mapper | 0 | 0 | |
| mongodb-odm | 4 | 28 | not the data parameter 18, evaluation order 10 |
| core | 72 | 214 | not the data parameter 103, evaluation order 73 |
| tq-cli | 594 | 716 | not the data parameter 429, evaluation order 114, operand of an operator 71 |

None was undone by the check, `phn check` reports the same findings, and every tree's suite,
PHPStan and php-cs-fixer pass on the compiled output. In the 8.5 output the only pipes left are at
property receivers, in 3 files of core and 19 of tq-cli, each stage a first-class callable such as
`… |> $this->output->writeln(...)` except one arrow function in core.

### 8.2 `list-types`, `map-types` — opt-in

An `array` declaration with no docblock type becomes `list<T>` or `map<K, V>` when every value it can
hold provably is one:
- a private property, whose writes are all in the class;
- the return of a private or final method, from all its `return`s;
- a private method's parameter, whose calls are all in the class.

It erases to the same `array`, so the PHP signature does not move.

## 9. Analysis rules

The vault parks three groups of PHPStan rules under M6. PHPStan reports each at level max, and the
parity test allow-lists each today:

| Group | PHPStan identifiers | Proposal |
|---|---|---|
| Throw analysis | `catch.neverThrown`, `throws.unusedType`, checking `@throws` | **In M6.** `throws` is Phaenon syntax (D18), and the editor will show its findings |
| Unused private members | `method.unused`, `property.unused`, `classConstant.unused` | In M6 if cheap (a scan per class), otherwise later |
| Purity and unused results | `*.resultUnused`, `void.pure`, `@phpstan-pure` | Later: it needs a purity model, which the vault's "method-call path across impure calls" item needs as well |

## 10. The target: `php-service-template` on `.phn`

**What changes in the template:**

| Area | Change |
|---|---|
| Sources | `src/` and `tests/` become `.phn` |
| Project file | a `phaenon.json` builds them into `.src/` and `.tests/` |
| Autoload and tools | PSR-4, `autoload-dev`, PHPUnit, PHPStan and php-cs-fixer point at the outputs |
| `composer.json` | `require` gains `phaenon/phn`; `allow-plugins` gains `phaenon/phn: true` |
| Composer scripts | `cs` → `phn fmt`; Rector through `phn fix` |
| Dockerfile | adjusted as §7.3 says |
| `.gitignore`, `.dockerignore` | list the outputs |
| `bin/init` | rewrites placeholders in `*.phn` too |
| CI | the reusable workflows' steps from §7.2 |
| Inherited `CLAUDE.md` | the guide a service inherits shows the idioms in Phaenon |

**What stays as it is:**
- `contracts/` stays PHP. The contracts-codegen workflow and the subtree split read it as PHP;
  revisit after M6.
- `tequila/core` stays PHP on `main`, and services consume it as PHP.
- `iam` is M7.

**The proof.** A service born from the working tree (`bin/init` on a throwaway copy) passes:
1. its suite, PHPStan and php-cs-fixer;
2. its image build;
3. a local deploy (OrbStack);
4. an exception thrown in the running service that the Monolog processor logs at its `.phn` line.

**Dependency:** Tequila's CI and the tenants install `phaenon/*` through Tequila's package registry
and the binary through its mirror (T1, §3.2). The registry needs read access to the `phaenonlang`
repositories for that, a maintainer's step when slice 1 gets there.

## 11. Order

| # | Slice |
|---|---|
| 1 | Release pipeline, `phaenon/phn` with the plugin, installed packages read from their sources, the versions in lockstep; the three in-place libraries on the sibling layout, consumed by core's dogfood through the plugin |
| 2 | CI formats and `phn trace`'s new readers (coverage included), `build --check` with `--emit-on-error`, the P4xxx checks on installed packages |
| 3 | `phn fix`, measured first (§4); then `phn fmt` |
| 4 | `phn lsp`, its diagnostics in PhpStorm, build on save; `phn watch` |
| 5 | The VS Code extension |
| 6 | Throw analysis (and unused private members, per Q8) |
| 7 | `list-types` and `map-types`, measured on core and tq-cli (`pipes` came with D20) |
| 8 | The template on `.phn`, `phaenon/*` installed through Tequila's registry |

**Why this order:**
- **Distribution first**, because everything else is consumed through it: the editor's server is the
  project's `vendor/bin/phn`, and the template needs the plugin.
- **`phn fix` before `phn fmt`**, because `phn fmt`'s second layer *is* `phn fix`.
- **Editors after the tools they call.**
- **Codemods last**: they depend on nothing else and carry the largest risk of churn in the dogfood
  trees.
- **The template at the end**, since it exercises everything above.

## 12. Alternatives considered

| Alternative | Why not |
|---|---|
| A Composer package per platform (D13's wording) | Composer cannot select by platform, and Packagist would need the binaries committed into git (§3.2) |
| One package with every binary | ~45 MB per install and per release in git |
| A PHAR or a WASM build of `phn` | Go does not compile to a PHAR. WASM under a PHP WASM runtime is slow, and nothing in PHP's ecosystem runs that way |
| A native formatter for all of PHP | Two style authorities, and core's style is mostly code-rewriting migration fixers (§5) |
| php-cs-fixer on masked `.phn` | Comment-moving fixers, and fixers that see code that is not the code (§5) |
| The LSP answering everything in PhpStorm | It would discard the refactorings, inspections and debugger integration the plugin's PSI already has |
| A persistent vendor cache now | Core's full check takes 2.1 s and one file 0.33 s, and the server keeps the model in memory |
| fsnotify for `phn watch` | `go.mod` has no requirements (the maintainers' decision), and polling is enough |
| A `setup-phn` GitHub Action | The plugin already puts `phn` in every Composer project |

## 13. Open questions for the review

1. *(T1)* **When do the `phaenonlang` repositories go public?** Packagist, release assets without tokens,
   and the two editor marketplaces all need public repositories. Tequila's CI reads private GitHub
   repositories only through an App token scoped to its `tequila-gha` mirror.
   - (a) At M6's start.
   - (b) Before slice 8: slices 1–7 run privately, the dogfood trees use path repositories and
     Composer's GitHub token, and the repositories are published before the template, whose
     consumers include tenants' CI.
   - (c) After M6, with `composer.tequila.dev` serving `phaenon/*` and the Tequila App granted the
     `phaenonlang` repositories in the meantime.

   Recommendation: (b). Publication does not block the work, and the template should not teach
   tenants a private source.
2. **D13's amendment.** Release assets pinned by checksum in `phaenon/phn` instead of a package per
   platform (§3.2). Recommendation: yes.
3. **Where `phaenon/phn` lives.** The compiler repository's root (one tag, one version) or a
   separate `phaenonlang/composer-plugin`. Recommendation: the root.
4. **Whether the plugin builds the root project at `dump-autoload`.** If it does (opt-out:
   `extra.phaenon.build: false`), `composer install` yields a runnable project, which is Composer's
   culture. If it does not, the root is built only by `phn build`, the editor or `phn watch`.
   Recommendation: build it.
5. **Installed packages build with findings as warnings** (§3.3). Confirm.
6. **`phn fmt`'s second layer through `phn fix`** and the measure-first gate (§4, §5). Confirm.
7. **VS Code's scope in M6.** Recommendation: §6.3's minimal client, since the server is the cost
   and the client is small. The alternative is to defer the extension to M7's public launch.
8. **Analysis rules** (§9). Recommendation: throw analysis in M6, unused private members if cheap,
   purity later.
9. *(T9)* **The codemods on the dogfood trees.** Asked as: run them on core and tq-cli for
   measurement, apply them only on a maintainer's go-ahead per rule. Answered with D20 instead.
10. **The roadmap's M6 entry.** "GitHub Action" becomes the CI formats plus Tequila's reusable
    workflows (§7.2), and "Docker base image update" becomes no base-image change (§7.3). Confirm.

## 14. Decisions

Reviewed 2026-11-03.

| # | Decision | Why |
|---|---|---|
| T1 | **The repositories stay private until Tequila runs on Phaenon in production**, and possibly Buzok too. Until then `phaenon/*` reach Tequila and its tenants through Tequila's package registry, the binaries through a mirror it serves (§3.2), the PhpStorm plugin through a custom plugin repository, the VS Code extension as a `.vsix` | A language is ready for the public when products have been built and run in production on it, with features delivered on it continuously, not when it first compiles a codebase |
| T2 | **D13 amended**: one package, `phaenon/phn`, with the binaries as release assets pinned by checksum in the package, fetched through Composer's HTTP client (§3.2) | Composer cannot pick a package by platform, and Packagist hosts no artefacts |
| T3 | `phaenon/phn` is built from the compiler repository's root (§3.1) | one tag, one version, nothing to keep in step |
| T4 | The plugin builds the root project at `dump-autoload` too, unless `extra.phaenon.build: false` (§3.3) | `composer install` yields a runnable project, as Composer's culture expects |
| T5 | An installed package builds with its findings as warnings (§3.3) | its own CI gave the verdict; a consumer's `phn` may be stricter |
| T6 | `phn fmt` is Phaenon's layout plus the project's fixer through `phn fix`, the second layer built once the measure of §4 passes (§5) | the project's php-cs-fixer configuration stays the one authority on PHP style |
| T7 | VS Code gets a minimal client in M6 (§6.3) | the server is the cost; the client is small |
| T8 | Throw analysis in M6, unused private members if cheap, purity later (§9) | `throws` is Phaenon syntax and the editor will show it |
| T9 | **D20**: no method syntax on values that are not objects. §8's `pipes` rule is on by default (three calls or more, `--pipes=2`, `--keep-calls`) and ships before the first slice; the `list-methods` and `string-methods` rules of the draft are gone with the method syntax | `roadmap.md` D20 |
| T10 | Roadmap M6's "GitHub Action" is CI formats plus Tequila's reusable workflows (§7.2), its "Docker base image update" no base-image change (§7.3) | the plugin brings the binary wherever Composer runs |
| T11 | **For an 8.5 target a pipe that PHP evaluates as the call it makes is written as that call**, a run of them as nested calls; any other pipe stays one, its partial applications arrow functions called at once (`php-versions.md` §5.3) | a pipeline the codemod wrote compiles back to the code it was written from; the arrow functions, the closures with an inline `@var` a `list` parameter needed and the closure each of them created at every call are gone from the 8.5 output |
| T12 | A method stage's receiver may be a property its class declares without hooks, when the inner calls do not assign it (§8.1); the 8.5 lowering keeps such a stage a pipe stage | `… \|> $this->output->writeln(...)` ends most of tq-cli's pipelines (92 of the 239 chains the order rule refused); that a method the inner calls run reassigns the property is the codemod's assumption, not the compiler's |
| T13 | A stage takes the piped value only into its first parameter or into the data parameter of PHP's functions that take it last (§8.1) | a pipeline follows the data: `count($xs) \|> sprintf('%d items', ?)` and `actual \|> … \|> self::assertSame(expected, ?)` read worse than the calls |
