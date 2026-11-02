# Phaenon design — agent guide

The language design documents of Phaenon. The map of every Phaenon repo, the working method and the
commit rules are in the compiler repo: read `../phaenon/AGENTS.md` first.

| Doc | Holds |
|---|---|
| `language-overview.md` | Scope: what stays from PHP, what is removed, what is added, per milestone |
| `generics.md` | Generics design; §11 records the decisions made while building M3 |
| `checker.md` | Type checker design (M2); §9 decisions, §10 open questions |
| `roadmap.md` | Decisions D1–D17, milestones M0–M7 and the PHP release milestones with their status, working method (§4) |
| `php-versions.md` | The target PHP version, following PHP releases, what each release adds and how it lowers |

- A feature is designed here before it is built. A new area gets its own doc;
  `language-overview.md` §5 names the pending ones.
- Draft text stays as written. What building a milestone settles goes into that doc's
  *Decisions made while building Mx* section (*Topic | Decision | Why*), which wins where the two
  differ; the milestone's roadmap entry records ✅ items, *Found on the way*, *Open*, *Target*.
- Dates are absolute.
- Nothing to build or test: Markdown only.
- Commit messages follow Conventional Commits (`feat:`, `fix:`, `docs:`, … with a body that says
  what changed and why), with no attribution lines and no `Co-Authored-By`: the repository will be
  public.
