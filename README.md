# Phaenon — language design

Phaenon is a typed dialect of PHP 8.5+ that compiles to idiomatic PHP: "TypeScript for PHP".
This repository is its design: what the language is and will be, and every decision taken while
building it. The compiler, `phn`, is
[phaenonlang/phaenon](https://github.com/phaenonlang/phaenon); the playground and the downloads
are on [phaenon.dev](https://phaenon.dev).

Pre-alpha. Markdown only: there is nothing to build or test here.

## How to read it

A document is a draft followed by its decision sections. The draft is the design as it was
proposed, and it stays as written. A decision records what a review or the building of a
milestone settled, with its date and its reasons. **Where the draft and a decision differ, the
decision wins.** A superseded decision keeps its row and names what superseded it.

| Document | Holds | Decisions |
|---|---|---|
| `language-overview.md` | The scope: what stays from PHP, what is removed, what is added | in each feature's document |
| `roadmap.md` | The plan: milestones M0–M7 and the PHP release milestones with their status, the working method | §1 (D1–D28) |
| `generics.md` | Type parameters, variance, erasure to PHP, `class<T>`, interop with docblock generics | §11 |
| `collections.md` | `list<T>`, `map<K, V>`, tuples, shapes and the legacy `array` | §7 |
| `strings.md` | Text and `bytes` | §6 |
| `functions.md` | Methods without a keyword, closures in constant expressions, expression bodies, `throws`, the source layout | §7 (F1–F13), §8 |
| `concurrency.md` | `concurrently { }` blocks and `spawn` over amp v3 and Revolt; withdrawn until PHP core has a scheduler | §15 (C1–C10), §16, §17 |
| `checker.md` | The type checker: its types, program model, checks and diagnostics, PHPStan parity | §9; open questions in §10 |
| `line-map.md` | The compiled layout, and the line map that leads positions in compiled PHP back to the source | §6 |
| `tooling.md` | Distribution, the Composer plugin, `phn fix`, `phn fmt`, `phn lsp`, the editors, CI, the codemods | §14 (T1–T19) |
| `php-versions.md` | The target PHP version, following PHP's releases, lowering what is newer than the target | §9 |
| `AGENTS.md` | The guide for coding agents working in this repository (`CLAUDE.md` includes it) | |

Two names recur:
- **Tequila**, the PHP platform behind [tequila.dev](https://tequila.dev), is the codebase Phaenon
  is built to run. Its trees (`core`, `tq-cli`, `mongodb-odm` and others) are where the
  measurements in these documents come from.
- **The vault** (`open-items`, `testing`, `dogfood`, …) is the project memory in the compiler
  repository, under `.serena/memories/`.

## Proposing a change

Open a pull request that edits the draft text and adds a row to the document's decision section:
the topic, the decision, and why. A change to the language starts here, before it is built. The
maintainers decide.

## License

MIT, see [LICENSE](LICENSE).
