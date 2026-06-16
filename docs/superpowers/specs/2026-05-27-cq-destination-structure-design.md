# CQ-Toolkit Destination Directory Structure — Design

**Date:** 2026-05-27
**Status:** Approved for planning

## Problem

All CQ-Toolkit output lands in a single flat `CQ-Reviews/` folder, distinguished only by filename prefix (`<Solution>-CQ-<Lens>.md`). Three problems:

1. **Too many files in one folder.** A repo with many solutions/projects becomes a wall of files.
2. **A unit's reports are scattered.** The reports for one solution are interleaved alphabetically with every other solution's reports.
3. **Mixed concerns share one namespace.** Per-unit reviews, cross-cutting summaries, implementation plans, and build scripts all live together.

A secondary inconsistency: the HTML output is *already* nested per-solution (`HTML/<Solution>/`), while the markdown is flat — the two halves of the toolkit disagree about structure. The docs also flip-flop between "solution" and "project" as the review unit (`cq-plan.md` says `<Project>-CQ-<Kind>.md`; the agents write `<Solution-Name>-CQ-…`).

## Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Group output by **unit** (folder per unit), not flat-by-filename. | Solves pains 1 & 2; aligns markdown with the already-nested HTML. |
| D2 | Split top-level by **concern**: `solutions/ projects/ summaries/ plans/ scripts/ site/`. | Solves pain 3. |
| D3 | The review unit is the **project (`.csproj`)** for the per-code lenses — a granularity change, not just a rename. | User intent: unify on "project". |
| D4 | **Hybrid lens scope.** `Data` / `CodeReview` / `TestReview` run per-`.csproj`. `Architect` + `Purpose` stay per-solution. | Architecture is inherently cross-project; a library `.csproj` has no independent business purpose. |
| D5 | `projects/` is **flat**, keyed by project name (`projects/<Project>/`), not nested under solution. | Multi-`.sln` repos share `.csproj`s; flat reviews each project once and dedupes shared ones. Project names are normally unique repo-wide. |
| D6 | Build scripts live in `scripts/`, separate from the generated `site/`. | `site/` is regenerated and wiped; source scripts must not live in a regenerated dir. |
| D7 | `HTML/` is renamed `site/`. | Names the artifact by what it is, not its format. |

## Target structure

```
CQ-Reviews/
  solutions/
    <Solution>/                     # one set per .sln (cross-cutting lenses)
      Purpose.md                    # preserved across re-scans
      Architect.md
  projects/
    <Project>/                      # one set per .csproj (per-code lenses)
      Data.md                       # production projects with a data layer
      CodeReview.md                 # production projects
      TestReview.md                 # test projects
  summaries/
    CQ-Summary.md                   # top-level cross-lens brief
    CQ-Summary-Architecture.md
    CQ-Summary-CodeReview.md
    CQ-Summary-Data.md
    CQ-Summary-TestReview.md
  plans/
    IMPLEMENTATION-PLAN-DETAILED-<Lens>.md   # x4
    IMPLEMENTATION-PLAN-SUMMARY-<Lens>.md    # x4
  scripts/
    build_html.py
    build_docx.py
  site/                             # generated HTML (was HTML/)
    index.html
    solutions/<Solution>/index.html, Architect.html
    projects/<Project>/Data.html, CodeReview.html, TestReview.html
```

A unit folder holds **only the lenses that apply** to it: test projects get `TestReview.md`; production projects get `Data.md` (where a data layer exists) and `CodeReview.md`; every solution gets `Purpose.md` and `Architect.md`.

### Filename scheme

Inside a unit folder the filename is just the lens (`Architect.md`, `Data.md`, `CodeReview.md`, `TestReview.md`, `Purpose.md`). The unit name is the folder, so the redundant `<Unit>-CQ-` prefix is dropped. This mirrors the HTML output, which already uses `<Solution>/Architect.html`.

## Project discovery & lens routing

Driven by `/cq-scan` (and `/cq-plan` which wraps it):

1. Discover `.sln` files as today.
2. For each solution, enumerate its `.csproj` files (parse the `.sln`, or glob).
3. Apply the existing **production/test classification** (per-csproj; see `[[scan-classification-model]]`):
   - **Test projects** → `TestReview` only.
   - **Production projects** → `CodeReview` always; `Data` where a data layer is present.
4. Run `Architect` + `Purpose` once **per solution** (unchanged scope).

Per-`.csproj` reviewer agents (`CQ-Data`, `CQ-Reviewer`, `CQ-Test-Reviewer`) are re-scoped: their invocation contract takes a **project** (`.csproj` path) rather than a solution folder, and they write to `projects/<Project>/<Lens>.md`. `CQ-Architect` and `CQ-Business-Value` keep solution scope and write to `solutions/<Solution>/{Architect,Purpose}.md`.

## Reference & linkification rework

The citation scheme is coupled to the flat filename and must change — but only the **derivation rule**, not the surface form.

- **Short-name derivation changes from filename-split to folder+basename.** Today: glob `CQ-Reviews/*.md`, split on `-CQ-` to get `{Solution, ReportType}`, and drop the `CQ-` prefix / `.md` suffix the files carry. New: the short name is `<folder>-<basename-without-.md>` — e.g. `solutions/Catalog/Architect.md` → `Catalog-Architect`; `projects/CatalogApi.WebApi/CodeReview.md` → `CatalogApi.WebApi-CodeReview`.
- **The in-prose citation surface form is unchanged.** `` `Catalog-Architect §Findings #5` `` and `` `Architecture-Summary §AR2` `` look exactly as before, so existing examples in the agent prose do not need rewording.
- **Remove the "files carry `CQ-` prefix and `.md` on disk" wording.** Per-tier report files no longer carry `CQ-`. The "drop `CQ-`" instruction in the nomenclature blocks is replaced by the folder+basename rule. Summary files keep their `CQ-Summary-<Lens>.md` names (their short name `<Lens>-Summary` / `Architecture-Summary` is unaffected — confirm the mapping is stated explicitly).
- **`cq-summary` enumeration step** (currently "Glob `CQ-Reviews/*.md` and split on `-CQ-`") becomes: glob `CQ-Reviews/solutions/*/*.md` and `CQ-Reviews/projects/*/*.md`; derive `{unit, lens}` from folder + basename.
- **`§Findings #N` and `§<Code>` anchors are unaffected.**

## HTML site (two-tier)

`build_html.py` (spec owned by `cq-html-publisher.md`) is reworked:

- **Input** is now the two nested tiers (`solutions/*/*.md`, `projects/*/*.md`) instead of flat files. The flat→nested translation the script does today is removed (input is already nested).
- **Output tree** mirrors the tiers: top-level `index.html` from `CQ-Summary.md`; a per-solution page (`Purpose` + `Architecture`, with links down to that solution's project reports); per-project pages for the per-code lenses.
- **Cross-reference linkification** map reparses the new short names → new page paths (`Catalog-Architect §Findings #3` → `solutions/Catalog/Architect.html#findings-3`; `CatalogApi.WebApi-CodeReview §Findings #2` → `projects/CatalogApi.WebApi/CodeReview.html#findings-2`).
- **Solution→project linkage:** the per-solution `index.html` lists and links the project reports belonging to that solution. The solution→project mapping comes from `.sln` enumeration at scan time; the publisher reads it from the directory layout plus (if needed) a small manifest written by `/cq-scan`.

## Wipe-but-preserve logic

`/cq-plan` wipes prior output before a fresh run. Updated rule: wipe everything under `CQ-Reviews/` **except** `solutions/*/Purpose.md` (the only preserved artifact, per the existing convention that purpose files are expensive to regenerate and rarely go stale). `scripts/` is preserved (source, not output).

## Files touched

- `agents/cq-business-value.md` — write path → `solutions/<Solution>/Purpose.md`; preserve-path wording.
- `agents/cq-architect.md` — write path → `solutions/<Solution>/Architect.md`; Purpose-lookup path; citation derivation wording.
- `agents/cq-data.md`, `agents/cq-reviewer.md`, `agents/cq-test-reviewer.md` — re-scope to a single `.csproj`; write path → `projects/<Project>/<Lens>.md`; Purpose-lookup path; citation wording.
- `agents/cq-summary.md`, `agents/cq-domain-summary.md` — enumeration globs (two tiers); short-name derivation rule; write dir → `summaries/`.
- `agents/cq-html-publisher.md` — two-tier input/output; linkification map; run from `scripts/`, output to `site/`.
- `commands/cq-scan.md` — project discovery + production/test routing; per-project dispatch; report-location wording.
- `commands/cq-plan.md` — same discovery/routing; plan output → `plans/`; wipe-but-preserve rule; resolve `<Project>` vs `<Solution>` wording.
- `README.md` — Layout + Conventions sections; note the no-migration behavior.

## Out of scope

- **No migration** of existing `CQ-Reviews/` folders in target repos — it is regenerated output; the next `/cq-scan` produces the new layout. The README notes this.
- **No change to the content or criteria** of any lens — only its review unit (D3/D4) and on-disk location.

## Open questions

None. All decisions resolved during brainstorming (D1–D7).
