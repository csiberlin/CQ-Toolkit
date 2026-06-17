# CQ-Toolkit

A Claude Code plugin that bundles a family of **C**ode-**Q**uality review agents and orchestration commands for C# solutions.

## What it does

CQ-Toolkit reads a C# repository (one or many `.sln` files) and produces a tiered set of reports:

1. **Per-solution purpose** — what the solution does and why it exists (`CQ-Business-Value`).
2. **Reviews** — Solution layout + backend architecture per solution (`CQ-Architect`); WPF-first frontend architecture per solution when a WPF project exists — WinForms/Razor/Blazor detected and labeled only (`CQ-Frontend-Architect`); Data and Code quality per production project (`CQ-Data`, `CQ-Reviewer`); Test quality per test project (`CQ-Test-Reviewer`).
3. **Cross-solution summaries** — one per domain plus a top-level brief (`CQ-Summary`, `CQ-Domain-Summary`).
4. **Management summary** — findings regrouped by quality attribute: Scalability, Readability, Maintainability, Security, Reliability, Test Quality (`CQ-Management-Summary`).
5. **Static HTML site** — browsable index of all the reports (`CQ-HTML-Publisher`).
6. **Implementation plans** — DETAILED and SUMMARY plans, one of each per CQ domain (Architecture, Frontend, CodeReview, Data, TestReview), ready for review before execution (`/cq-plan`).

All output lands in a `CQ-Reviews/` folder at the repository root, organised by concern. Nothing in the source tree is modified.

```
CQ-Reviews/
  solutions/<Solution>/    Purpose.md, Architect.md,          (one set per .sln)
                           Frontend.md (when WPF project exists)
  projects/<Project>/      Data.md, CodeReview.md,           (one set per .csproj)
                           TestReview.md
  summaries/               CQ-Summary.md + 5 per-lens summaries
  plans/                   10 implementation plans (DETAILED + SUMMARY)
  scripts/                 build_html.py, build_docx.py
  site/                    generated browsable HTML
```

## Layout

```
.claude-plugin/
  plugin.json           Plugin manifest
agents/                 Sub-agent definitions (CQ-Architect, CQ-Reviewer, ...)
commands/               Slash commands (/cq-purpose, /cq-analyze, /cq-summary,
                        /cq-management-summary, /cq-plan, /cq-all)
```

## Installation

### As a Claude Code plugin

From a Claude Code session in any project:

```
/plugin install <path-or-git-url-to-this-repo>
```

Once installed, the `cq-*` agents become available to dispatch and the `/cq-*` slash commands (`/cq-purpose`, `/cq-analyze`, `/cq-summary`, `/cq-management-summary`, `/cq-plan`, `/cq-all`) show up.

### Manual copy (fallback)

Copy `agents/cq-*.md` into your project's `.claude/agents/` and `commands/cq-*.md` into your project's `.claude/commands/`.

## How to run a review

Every review is calibrated by a **per-solution purpose file** whose `## Scale signals` table you fill in with the real load baseline. Curating that table *before* the analysis is what lets `CQ-Architect` judge "50x" claims against actual numbers instead of guesses. The full sequence is three steps.

### 1. Generate the purpose files — `/cq-purpose`

```
/cq-purpose [subdir ...]
```

Generates `CQ-Reviews/solutions/<Solution>/Purpose.md` for every solution that doesn't have one yet (existing files are preserved). Downstream agents call this the *CQ-Purpose* report; under the hood it dispatches the `CQ-Business-Value` agent. It reports each solution's **Scale-signals disposition**:

- `scaffolded` — it wrote a blank `## Scale signals` table that **you must fill in** (step 2).
- `kept` — an existing table was preserved verbatim, so there is nothing to do.

### 2. Fill in the `## Scale signals` table

Open each `CQ-Reviews/solutions/<Solution>/Purpose.md` and complete the `## Scale signals` section with the real **load baseline** — current request volume (req/s), concurrent users, device/customer counts, and growth plans. These numbers are deliberately **not** inferred from code, so a human has to supply them. This section is **user-owned**: it is `CQ-Architect`'s highest-priority input for its 50x analysis and is **preserved across re-scans**, so you only fill it in once.

### 3. Run the analysis

Run the per-unit reviews, then aggregate however you need:

```
/cq-analyze [subdir ...]      # the reviews: per-solution Architect + per-project
                              # Data / CodeReview / TestReview analysis files
/cq-summary                   # cross-solution per-domain summaries (needs /cq-analyze)
/cq-management-summary        # executive briefs by quality attribute (needs /cq-summary)
/cq-plan                      # ten implementation plans (needs /cq-analyze + /cq-summary)
```

Or run the whole pipeline in one shot:

```
/cq-all [subdir ...]          # /cq-purpose → /cq-analyze → /cq-summary → /cq-plan
```

`/cq-all` will *auto-generate* any missing purpose file and then analyze immediately — so to get a first pass calibrated against real Scale-signals numbers, do steps 1–2 first (or run `/cq-purpose` on its own) rather than letting `/cq-all` analyze a blank table. On re-runs your curated `Purpose.md` files are preserved, so `/cq-all` is the convenient choice.

### Command map

| Stage | Command | Produces |
|---|---|---|
| 1. Baseline *(once)* | `/cq-purpose` | `solutions/<S>/Purpose.md` (fill in `## Scale signals`) |
| 2. Review | `/cq-analyze` | `solutions/<S>/Architect.md`, `projects/<P>/{Data,CodeReview,TestReview}.md` |
| 3a. Technical summary | `/cq-summary` | `summaries/CQ-Summary-<Domain>.md` + top-level |
| 3b. Exec briefs | `/cq-management-summary` | `summaries/CQ-<Attribute>.md` (Scalability, Security, …) |
| 4. Plans | `/cq-plan` | `plans/IMPLEMENTATION-PLAN-{DETAILED,SUMMARY}-<Kind>.md` |
| **All** | `/cq-all` | runs stages 1 → 4 end-to-end |

## Conventions

- Reports land in `CQ-Reviews/` relative to the working directory, split by concern (`solutions/`, `projects/`, `summaries/`, `plans/`, `scripts/`, `site/`).
- `solutions/<Solution>/Purpose.md` files are preserved across re-scans — they rarely go stale and are expensive to regenerate.
- All agents are read-only across the production source tree. Output is markdown.
- No migration of an existing flat `CQ-Reviews/` is performed — the next `/cq-analyze` simply produces the new layout.

## License

TBD.
