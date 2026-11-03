# Phaenon — Tooling and distribution (draft 0.1)

Status: **reviewed 2026-11-03** except §8, which waits on the method-syntax question (§14, T9) · M6. Companion: `roadmap.md` D2, D8, D10, D12, D13,
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
- **Plugins run under `--no-scripts`.** That flag only drops `composer.json`'s script listeners, not
  the event subscribers of plugins (`EventDispatcher::getListeners()`). `--no-plugins` turns plugins
  off.
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
  no second repository to keep in step.
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
    Composer credentials they already have. Slice 1 settles which part of the registry serves it.

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
  or its target changed since the last build, which a stamp in its output directory records. A
  `dump-autoload` on a warm tree then costs nothing. A full build costs little anyway: core's 800
  files take 1.4 s.
- **`phn build --vendor`** does the same by hand.
- **The root project's build can be turned off** with `extra.phaenon.build: false`, for a project
  whose own workflow builds it (Q4).

### 3.4 Versions

- **Lockstep until 1.0.** `phaenon/phn` X.Y.Z requires `phaenon/runtime` X.Y.* and suggests
  `phaenon/concurrent` X.Y.*, and `phn stdlib` regenerates both from the same tag. One compiler
  builds the whole vendor tree at install, so the runtime always matches the compiler that wrote
  every call into it.
- **A library states the compiler range it needs.** It requires `phaenon/phn` with the range its
  sources compile with (`^0.4`), as a PHPStan extension requires PHPStan.
- **`phn check` reads `installed.json`** and reports two P4xxx errors:
  - a file that uses `concurrently` where `phaenon/concurrent` is not installed (the vault's M5
    item);
  - an installed runtime package of another minor version.

### 3.5 The three in-place libraries

reflector, object-mapper and mongodb-odm still commit a `.php` beside each `.phn`, because their
dogfood branches predate D12 and D13. They move to the sibling layout and drop the committed PHP, and
core's dogfood branch consumes them as path repositories through the plugin. That is the first proof
of D13.

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
layout is already what PSR-12 wants (D19). A tool that still wants to change it reveals either a
compiler bug or a configuration choice, and the report makes either visible.

**Measure first.** Before §5 builds on this mechanism:
1. Un-format core's `.phn` by running a php-cs-fixer config with other spacing rules through
   `phn fix`.
2. Format it back the same way.
3. Count the changes carried back and refused.
4. Run one of core's Rector upgrade sets the same way and count again.

If the refused share is not small, `phn fmt` ships its first layer alone (§5).

## 5. `phn fmt`

`phn fmt` has two layers.

### 5.1 Phaenon's layout

The first layer is native, with no options:
- the canonical layout of `functions.md` §5: braces, `{}` for empty bodies, one-line docblocks;
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
- **In editors**, the LSP's formatting request runs the first layer, which takes milliseconds. The
  second layer costs a PHP process per run, so it runs on an explicit command.

### 5.4 Why not the alternatives

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
| Completion | members, including method syntax on `list`, `map`, `string` and `bytes`; names in scope; classes with their import |
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
  other files' types, through method syntax and the override-erasure rule, so building one file
  alone would be wrong.
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

## 7. CI and containers

### 7.1 Annotations and reports

- **`phn check` writes CI formats.** `--format=github` writes GitHub's workflow commands
  (`::error file=src/Foo.phn,line=12,col=5,title=P2103::…`), and `--format=checkstyle` serves other
  CI systems.
- **`phn trace` reads three more formats:**
  - GitHub workflow commands, which PHPStan writes with `--error-format=github`;
  - Checkstyle XML;
  - JUnit XML, which PHPUnit writes with `--log-junit`.

  This matters because GitHub drops annotations on files that are not in the repository. The
  compiled outputs are gitignored, so a PHPStan finding on compiled code that is not traced is not
  shown at all.
- **Coverage.** `phn trace` translates Clover and Cobertura XML too:
  - a source line counts as covered when any compiled line of a run that covers it is;
  - holds count as generated text and are not reported.

  PhpStorm's coverage view, with the plugin reading the map, comes with §6.2 if the platform allows
  it.
- **`phn build --check`** passes when its only failures are files built `--emit-on-error` whose
  outputs are current. On core these are the 9 files with deliberate findings (vault `open-items`).

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

**How a codemod runs:**
- **Opt-in, by name.** Each codemod is a rule you name (`phn migrate --codemod list-types,string-methods
  src`), and it runs on `.php` and `.phn` alike.
- **Only where the checker proves its condition.**
- **The file is checked again afterwards.** If a rewrite changes a type the checker infers anywhere
  in the file, or adds a finding, it is undone for that file and reported.

| Rule | Rewrites | Condition |
|---|---|---|
| `list-types`, `map-types` | an `array` declaration with no docblock type → `list<T>` / `map<K, V>` | every value it can hold provably is one: a private property (all writes are in the class), a private or final method's return (all its `return`s), a private method's parameter (all calls are in the class). It erases to the same `array`, so the PHP signature does not move |
| `list-methods` | `array_map($f, $xs)` → `$xs->map($f)`; `array_values(array_filter($xs, $f))` → `$xs->filter($f)`; `in_array($x, $xs, true)` → `$xs->contains($x)`; `implode($sep, $xs)` → `$xs->join($sep)` | `$xs` is a `list<T>`, and the rule's table holds exact pairs only. `array_filter` alone keeps the keys, so it is not `filter()` |
| `string-methods` | `str_contains($s, $n)` → `$s->contains($n)`; `str_starts_with` → `startsWith`; `str_ends_with` → `endsWith` | `$s` is a `string`, not a `?string`, and only exact equivalents qualify. `strlen` counts bytes where `->length()` counts code points, and `strtolower` folds ASCII where `->lower()` folds Unicode (`strings.md` §3, §4), so neither is rewritten |

**Each rule's table is reviewed with its slice.** The rule is then run on core and tq-cli; its
measure is the number of rewrites, both suites green, and output that differs only in the lowered
static calls. Whether the dogfood trees keep the rewrites is Q9.

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
| 7 | The codemods, measured on core and tq-cli |
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
9. **The codemods on the dogfood trees.** Run them on core and tq-cli for measurement, but apply them
   to the trees (large, mechanical diffs) only on a maintainer's go-ahead per rule. Confirm.
10. **The roadmap's M6 entry.** "GitHub Action" becomes the CI formats plus Tequila's reusable
    workflows (§7.2), and "Docker base image update" becomes no base-image change (§7.3). Confirm.

## 14. Decisions

Reviewed 2026-11-03.

| # | Decision | Why |
|---|---|---|
| T1 | **The repositories stay private until Tequila runs on Phaenon in production**. Until then `phaenon/*` reach Tequila and its tenants through Tequila's package registry, the binaries through a mirror it serves (§3.2), the PhpStorm plugin through a custom plugin repository, the VS Code extension as a `.vsix` | A language is ready for the public when products have been built and run in production on it, with features delivered on it continuously, not when it first compiles a codebase |
| T2 | **D13 amended**: one package, `phaenon/phn`, with the binaries as release assets pinned by checksum in the package, fetched through Composer's HTTP client (§3.2) | Composer cannot pick a package by platform, and Packagist hosts no artefacts |
| T3 | `phaenon/phn` is built from the compiler repository's root (§3.1) | one tag, one version, nothing to keep in step |
| T4 | The plugin builds the root project at `dump-autoload` too, unless `extra.phaenon.build: false` (§3.3) | `composer install` yields a runnable project, as Composer's culture expects |
| T5 | An installed package builds with its findings as warnings (§3.3) | its own CI gave the verdict; a consumer's `phn` may be stricter |
| T6 | `phn fmt` is Phaenon's layout plus the project's fixer through `phn fix`, the second layer built once the measure of §4 passes (§5) | the project's php-cs-fixer configuration stays the one authority on PHP style |
| T7 | VS Code gets a minimal client in M6 (§6.3) | the server is the cost; the client is small |
| T8 | Throw analysis in M6, unused private members if cheap, purity later (§9) | `throws` is Phaenon syntax and the editor will show it |
| T9 | *Open*: the method syntax on non-objects, and with it §8's `list-methods` and `string-methods` (roadmap, D20 proposal) | |
| T10 | Roadmap M6's "GitHub Action" is CI formats plus Tequila's reusable workflows (§7.2), its "Docker base image update" no base-image change (§7.3) | the plugin brings the binary wherever Composer runs |
