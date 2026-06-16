# CQ Architecture-View Decomposition Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split the single per-solution architectural review into a Solution-Layout + Backend view (`CQ-Architect`) and a new WPF-first `CQ-Frontend-Architect` view, and ripple the change through scan, summaries, and plans.

**Architecture:** Approach B (hybrid) from the design spec. `cq-scan` classifies each production project into a tier (backend / WPF / other-UI / library) and derives a solution archetype that gates deep dives. `CQ-Architect` keeps layout + backend; a new agent owns WPF; a fourth summary domain (`Frontend`, prefix `FR`) and a fourth plan kind flow downstream. WinForms/Razor/Blazor are detected and labeled only.

**Tech Stack:** Markdown agent/command definitions under `agents/` and `commands/`. No compiled code, no test runner — verification is structural (`Grep`/`Read`) against the spec's acceptance criteria.

**Spec:** `docs/superpowers/specs/2026-06-16-cq-architecture-view-decomposition-design.md`

**Branch:** `feat/cq-frontend-architecture-view` (already created, off the cross-lens coordination commit).

**Convention for every task below:** locate edit points by **section heading**, not line number — line numbers drift as the files change. "Verify" steps use `Grep` because there is no test harness. Commit after each task.

---

## Task 1: `cq-scan` — tier classification, archetype, Frontend dispatch

**Files:**
- Modify: `commands/cq-scan.md` (sections "### 2. Classify each project" and "### 3. Dispatch agents in parallel")

- [ ] **Step 1: Add tier classification after the test-vs-production rule**

In `### 2. Classify each project`, after the existing test/production rule, append a tier sub-classification for production projects. Insert this block:

```markdown
For each **production** `.csproj`, also assign a **tier** (read the `.csproj` once — reuse the read from the test/production check):

- **Backend** — SDK is `Microsoft.NET.Sdk.Web`, OR it references `Microsoft.AspNetCore.*` / a worker SDK, OR it is the web/service host entrypoint.
- **Frontend (WPF)** — `<UseWPF>true</UseWPF>` is set, OR it references `PresentationFramework` / `DevExpress.Xpf.*`, OR the project folder contains `.xaml` files.
- **Frontend (other)** — `<UseWindowsForms>true</UseWindowsForms>` (WinForms), Blazor, or Razor/MVC. **Labeled only** — no specialized lens runs on it.
- **Library** — a production project matching none of the above (class library, domain, shared kernel).

From the per-project tiers, derive the **solution archetype** for each solution: `backend-only` | `desktop-only` | `mixed` | `library-only`. Record the tier map and archetype — they are passed to `CQ-Architect` and decide whether `CQ-Frontend-Architect` runs.
```

- [ ] **Step 2: Update the dispatch rules to pass archetype and add the Frontend agent**

In `### 3. Dispatch agents in parallel`, replace the per-solution dispatch bullet so it briefs the archetype and adds the conditional Frontend agent. Change:

```markdown
Per **solution** (`.sln` or `.slnx`), spawn ONE agent:
- `subagent_type: "CQ-Architect"` — architecture review of the whole solution.
```

to:

```markdown
Per **solution** (`.sln` or `.slnx`):
- `subagent_type: "CQ-Architect"` — solution-layout + backend architecture review. Brief it with the **solution archetype** and the **tier map** (which projects are backend / WPF / other-UI / library) in addition to the solution path and `<Solution-Name>`. It writes `<cwd>\CQ-Reviews\solutions\<Solution-Name>\Architect.md`.
- `subagent_type: "CQ-Frontend-Architect"` — **only when the solution has at least one Frontend (WPF) project.** Brief it with the WPF project path(s), the `<Solution-Name>`, and the working directory. It writes `<cwd>\CQ-Reviews\solutions\<Solution-Name>\Frontend.md`. Skip this agent entirely for solutions whose only frontend is "other" tech (WinForms/Razor/Blazor) — the layout view notes those.
```

- [ ] **Step 3: Update the command's frontmatter description**

Replace the `description:` line in the frontmatter so it mentions the layout/backend split and the frontend lens:

```markdown
description: Scan all C# solutions in the given subdirectories (or current dir), then dispatch CQ-Architect (solution-layout + backend) per solution, CQ-Frontend-Architect per solution that has a WPF project, CQ-Reviewer (+ CQ-Data where a data layer exists) per production project, and CQ-Test-Reviewer per test project.
```

- [ ] **Step 4: Verify**

Run: `Grep` for `Frontend (WPF)`, `solution archetype`, and `CQ-Frontend-Architect` in `commands/cq-scan.md`.
Expected: all three present; the dispatch section lists `CQ-Frontend-Architect` as conditional on a WPF project.

- [ ] **Step 5: Commit**

```bash
git add commands/cq-scan.md
git commit -m "cq-scan: classify project tiers, derive solution archetype, dispatch CQ-Frontend-Architect"
```

---

## Task 2: `CQ-Architect` — rescope to Solution Layout + Backend

**Files:**
- Modify: `agents/cq-architect.md` (frontmatter description; "## Scope of review"; the report template under "## Output"; "## Boundary…"; "## Rules")

- [ ] **Step 1: Update the frontmatter description**

Replace the `description:` so it scopes the agent to layout + backend and disclaims frontend:

```markdown
description: Reviews a C# solution's layout (project boundaries, cross-tier dependency direction, archetype) and its backend architecture (data flow, validation, authn/z, domain, error strategy, observability, evolvability, 50x scalability). WPF frontend architecture is owned by CQ-Frontend-Architect. Use for a solution-layout + backend architectural review.
```

- [ ] **Step 2: Add the archetype input to the invocation context**

Near the top (after "MANDATORY DELIVERABLE" or in a short "Invocation contract" note — match the file's existing structure), add:

```markdown
## Invocation context — archetype & tier map

The orchestrator passes you the **solution archetype** (`backend-only` | `desktop-only` | `mixed` | `library-only`) and a **tier map** (each production project tagged backend / frontend-WPF / frontend-other / library). Use them to gate sections:

- The `## Solution Layout` section ALWAYS renders.
- The `## Backend Architecture` section and the 50x stress test render **only when the archetype includes backend projects** (`backend-only` or `mixed`). For `desktop-only` / `library-only`, omit them and state "No backend projects — backend architecture and 50x stress test not applicable."
- You do NOT review WPF/frontend internals — that is `CQ-Frontend-Architect`. You only (a) inventory frontend projects in the layout view and (b) flag cross-tier dependency-direction violations involving them.

If the orchestrator did not pass an archetype, infer it from the `.csproj` SDKs / `UseWPF` / `.xaml` presence yourself and say so.
```

- [ ] **Step 3: Add a Solution Layout dimension at the top of "## Scope of review"**

Insert as the new dimension **1** (renumber the existing list, or add as "1. Solution layout" and let the existing "Architecture & layering" become a backend-leaning item — keep it simple: add the new item and note the rest are backend dimensions):

```markdown
1. **Solution layout (always — tech-neutral)** — project inventory with tier labels (backend / frontend-WPF / frontend-other / library / test) and the solution-archetype verdict. Cross-tier dependency direction: dependencies point inward (UI → application → domain ← infrastructure). Flag a WPF/UI project referencing EF Core / `HttpClient` / the DB directly, a class library depending on a UI assembly, or the backend referencing a desktop project. Shared-kernel / "big ball of mud" assessment at the solution level. Note test-project presence structurally (content is CQ-Test-Reviewer's).

The remaining dimensions below are **backend architecture** — they render only when the archetype includes backend projects (see Invocation context).
```

- [ ] **Step 4: Add the frontend-out-of-scope clause to the boundary section**

In "## Boundary with CQ-Reviewer and CQ-Data (non-overlap contract)" (or a new short clause beside it), add:

```markdown
- **WPF / frontend architecture — NOT YOUR SCOPE.** MVVM separation, data binding strategy, command patterns, dispatcher/threading, view lifecycle/navigation, XAML resource organization, and frontend perf are owned by **CQ-Frontend-Architect**. You only inventory frontend projects in `## Solution Layout` and flag cross-tier dependency-direction violations. Cite `<Sln>-Frontend` if a frontend finding sharpens a layout finding.
```

- [ ] **Step 5: Restructure the Output report template**

In the report template under "## Output", insert `## Solution Layout` immediately after `## Architectural Overview`, and wrap the existing backend dimensions under `## Backend Architecture`. Add the template blocks:

```markdown
## Solution Layout

**Archetype:** backend-only | desktop-only | mixed | library-only

| Project | Tier | Role |
|---|---|---|
| <relative\path.csproj> | Backend / Frontend (WPF) / Frontend (other) / Library / Test | <one line> |

**Cross-tier dependency direction:** <verdict — any inward-rule violations, each cited file:line>
**Solution structure:** <big-ball-of-mud vs clean; shared kernels; test-project presence>

## Backend Architecture

(Render only when the archetype includes backend projects. Otherwise replace this whole section with: "No backend projects — backend architecture and 50x stress test not applicable.")

<the existing backend findings, coverage rows, and 50x stress analysis go here>
```

- [ ] **Step 6: Add the `Tier` field to the Finding template and split the Coverage map**

In the Finding template add a line under the metadata block:

```markdown
**Tier:** Layout | Backend
```

In the `## Coverage map` template, split the table into a Layout block (always) and a Backend block (marked `not applicable` for desktop-only/library-only):

```markdown
### Layout coverage
| Dimension | Verdict |
|---|---|
| Project inventory & archetype | clean / <N> findings |
| Cross-tier dependency direction | clean / <N> findings |
| Solution structure / shared kernels | clean / <N> findings |

### Backend coverage
(Mark every row `not applicable` when the archetype has no backend projects.)
| Dimension | Verdict |
|---|---|
| Data flow | clean / <N> findings / not applicable |
| … (existing backend dimensions) | … |
```

- [ ] **Step 7: Add gating rules to "## Rules"**

```markdown
- **Solution Layout always renders; Backend Architecture + 50x render only for backend-bearing archetypes.** For desktop-only/library-only solutions, state backend is not applicable rather than forcing a backend-shaped review or a 50x verdict.
- **Do not review WPF/frontend internals** — that is CQ-Frontend-Architect. Inventory frontend projects and flag cross-tier dependency violations only.
- Every finding carries a `Tier: Layout | Backend` field.
```

- [ ] **Step 8: Verify**

Run: `Grep` for `## Solution Layout`, `## Backend Architecture`, `**Tier:**`, `CQ-Frontend-Architect`, and `not applicable` in `agents/cq-architect.md`.
Expected: all present; backend section is explicitly archetype-gated.

- [ ] **Step 9: Commit**

```bash
git add agents/cq-architect.md
git commit -m "CQ-Architect: rescope to Solution Layout + archetype-gated Backend; defer WPF to CQ-Frontend-Architect"
```

---

## Task 3: New agent `agents/cq-frontend-architect.md` (WPF-first)

**Files:**
- Create: `agents/cq-frontend-architect.md`

**Scaffold source:** copy the *structure* of `agents/cq-data.md` (the tech-conditional precedent) and apply the mechanical substitutions below, then replace the domain content with the WPF content in Step 2. Reproduce — do not summarize — these sections from `cq-data.md`, adapting the wording: MANDATORY DELIVERABLE, Invocation contract, Path conventions, Step 0 (Purpose), Step 0b (project conventions), Tool preference table, The value bar, the cross-lens block (Cross-Lens Flags + "Lens ownership is not a reason to demote" + load-independent severity), Self-review before writing (incl. `### Considered but not reported`), Output discipline (citation rules / table-cell / heading shape / backtick globs), and Rules.

**Mechanical substitutions when copying the scaffold:**
- Agent `name`: `CQ-Frontend-Architect`. Domain noun: "WPF frontend architecture".
- Deliverable path: `<working-directory>\CQ-Reviews\solutions\<Solution-Name>\Frontend.md`. The **unit is the solution** (like CQ-Architect), not the project — `<Solution-Name>` is the last dot-separated segment of the `.sln`/`.slnx` name, extension stripped.
- Self-citation form: `` `<Sln>-Frontend §Findings #N` ``.
- `tools:` same list as `cq-data.md` (Read, Glob, Grep, Write, Bash, the codebase-memory MCP tools).
- Owner-of-last-resort routing for secrets/config → **CQ-Architect** (unchanged).

- [ ] **Step 1: Create the file with frontmatter + scaffold**

```markdown
---
name: CQ-Frontend-Architect
description: Reviews the WPF frontend architecture of a C# solution — MVVM separation, data binding strategy, command patterns, dispatcher/threading, view lifecycle & navigation, resource/theming organization, frontend performance (responsiveness), lifetime/leaks, and state management. WPF-first; WinForms/Razor/Blazor are detected and labeled only. Use for a frontend architectural review of a C# solution that has a WPF project.
tools: Read, Glob, Grep, Write, Bash, mcp__codebase-memory-mcp__search_graph, mcp__codebase-memory-mcp__get_code_snippet, mcp__codebase-memory-mcp__search_code, mcp__codebase-memory-mcp__trace_path, mcp__codebase-memory-mcp__index_status, mcp__codebase-memory-mcp__index_repository
---

You are a senior WPF frontend architect reviewing the presentation tier of a C# solution. You focus on frontend architecture and patterns, not line-by-line code quality (that is CQ-Reviewer) and not backend/data/test concerns.
```

Then paste the adapted scaffold sections listed above.

- [ ] **Step 2: Add the tech-conditional gate**

```markdown
## Tech detection — WPF-first, others labeled only

Detect the frontend tech from the target project(s):

- **WPF** (`<UseWPF>true</UseWPF>`, `PresentationFramework` / `DevExpress.Xpf.*` references, `.xaml` files) → run the full WPF review below.
- **Only other UI tech** (WinForms `<UseWindowsForms>true</UseWindowsForms>`, Blazor, Razor/MVC) → write a one-paragraph report: "Detected `<tech>`; no specialized architectural review is defined for this project type in the CQ toolkit." Fill the Coverage map with `not applicable` and invent NO findings.
- **No frontend project at all** → you should not have been dispatched; write a one-line report saying so and stop.
```

- [ ] **Step 3: Add the WPF "## Scope of review" dimensions**

```markdown
## Scope of review (WPF)

1. **MVVM separation** — View/ViewModel/Model boundaries; code-behind minimal (view concerns only); no business logic in code-behind or XAML; ViewModels reach infrastructure (EF/`HttpClient`) only through injected services, never directly.
2. **Data binding strategy** — `INotifyPropertyChanged` correctness; `DataContext` flow; binding-error hygiene; ViewModel-shaped properties vs heavy `IValueConverter` logic; DevExpress `DXBinding` where the project skill mandates it.
3. **Commands** — `ICommand` / `DelegateCommand` / DevExpress POCO commands vs event handlers in code-behind; honoring the project's `BaseCommand` convention.
4. **Threading & dispatcher** — UI-thread blocking (sync I/O, `.Result` / `.Wait()` on the UI thread), `Dispatcher.Invoke` misuse, async patterns that keep the UI responsive, background-work model.
5. **View lifecycle, navigation & DI** — window/page/UserControl structure, a consistent navigation pattern, ViewModels resolved through DI not `new`.
6. **Resources, styles & theming** — `ResourceDictionary` organization, style/theme consistency, DevExpress theme usage.
7. **Frontend performance (responsiveness)** — UI virtualization on large grids/lists, binding / visual-tree perf, image/resource loading. This is the desktop analogue of the backend 50x stress test — reason about *responsiveness*, not throughput. There is NO 50x analysis here.
8. **Lifetime & leaks** — event-handler / static-event leaks, weak-event usage, `IDisposable` / unsubscribe discipline.
9. **State management** — app-level state, messenger / event-aggregator vs tight coupling.
```

- [ ] **Step 4: Add the non-overlap contract**

```markdown
## Boundary with CQ-Architect, CQ-Reviewer, CQ-Data (non-overlap contract)

- **CQ-Frontend-Architect owns** WPF architecture & patterns: MVVM separation as a strategy, binding/command/threading/navigation models, resource/theming structure, frontend perf, leak patterns.
- **CQ-Architect owns** solution layout + the cross-tier dependency-direction call (it flags a ViewModel referencing EF directly at the *layout* level; you own the *pattern* recommendation).
- **CQ-Reviewer owns** per-file code quality in the same projects — naming, method size, logging, a single mis-named ViewModel property.
- **CQ-Data owns** any data access, even when reached from a ViewModel.
- Overlaps resolve through `## Cross-Lens Flags`; secrets/config route to CQ-Architect (owner of last resort).
```

- [ ] **Step 5: Add the Output report template**

```markdown
## Output

Write the report to `<working-directory>\CQ-Reviews\solutions\<Solution-Name>\Frontend.md`.

# CQ-Frontend-Architect Report

**Solution:** <relative path to the `.sln`/`.slnx`>
**Frontend tech:** WPF | WinForms (labeled only) | Razor (labeled only) | Blazor (labeled only) | none
**Date:** <YYYY-MM-DD>
**WPF projects:** <list>

## Frontend Overview
<short paragraph: shell type, MVVM framework (Prism/MVVM Toolkit/DevExpress/hand-rolled), DI container, navigation style>

## Summary Verdict
- **MVVM discipline:** Good | Acceptable | Risky — <one sentence>
- **Responsiveness:** Good | Acceptable | Risky — <one sentence, naming the worst UI-thread risk>

## Coverage map
| Dimension | Verdict |
|---|---|
| MVVM separation | clean / <N> findings / not applicable |
| Data binding strategy | clean / <N> findings |
| Commands | clean / <N> findings |
| Threading & dispatcher | clean / <N> findings |
| View lifecycle / navigation / DI | clean / <N> findings |
| Resources / styles / theming | clean / <N> findings |
| Frontend performance | clean / <N> findings |
| Lifetime & leaks | clean / <N> findings |
| State management | clean / <N> findings |

## Findings
### 1. <Issue title>
**Category:** MVVM | Binding | Commands | Threading | Navigation/DI | Resources/Theming | Performance | Leaks | State management
**Severity:** High | Medium | Low

**Bad example** (`<relative\path.cs|.xaml>:<line>`):
(code block)

**Why it's a problem:** <one paragraph>
**Cost of inaction:** <concrete consequence at this app's context>

---

## Recommended Actions
## Optional / stylistic (below the value bar)
## Cross-Lens Flags
## Project-Convention Deviations
## Verification log
```

(For the shared sections — Recommended Actions, Optional/stylistic, **Cross-Lens Flags** with proposed-owner table, Project-Convention Deviations, Verification log with `### Considered but not reported` — reuse the exact shapes from `cq-data.md`, substituting `<Sln>-Frontend` self-citations and CQ-Architect/CQ-Reviewer/CQ-Data as the proposed-owner options.)

- [ ] **Step 6: Verify**

Run: `Grep` for `## Cross-Lens Flags`, `Considered but not reported`, `MVVM separation`, `no 50x`, `labeled only`, and `Frontend.md` in `agents/cq-frontend-architect.md`.
Expected: all present; the tech gate, WPF dimensions, boundary, and output template are all there; the cross-lens block matches the other reviewers.

- [ ] **Step 7: Commit**

```bash
git add agents/cq-frontend-architect.md
git commit -m "Add CQ-Frontend-Architect: WPF-first frontend architecture review"
```

---

## Task 4: `cq-domain-summary` — add the Frontend domain

**Files:**
- Modify: `agents/cq-domain-summary.md` (the domain intro line; the Invocation-contract domain table; Reference-nomenclature prefix table; the embedded legend block; Output H1 line)

- [ ] **Step 1: Widen the domain set**

Change the opening "You own a single domain `D ∈ {Architecture, CodeReview, TestReview}`" to `D ∈ {Architecture, Frontend, CodeReview, TestReview}`. In the same paragraph, note that `Architecture` and `Frontend` read per-**solution** reports; `CodeReview`/`TestReview` read per-**project**.

- [ ] **Step 2: Add the Frontend row to the input-glob table**

In the Invocation-contract table, add:

```markdown
| `Frontend`     | `CQ-Reviews\solutions\*\Frontend.md`  | solution | `CQ-Frontend-Summary.md`          |
```

- [ ] **Step 3: Add `FR` to the theme-prefix table and the legend block**

In the Reference-nomenclature prefix table add a row `| `Frontend` | `FR` | `### FR3 — <title>` |`. In the citations table add `| `solutions\<Sln>\Frontend.md` | `<Sln>-Frontend` |`. In the verbatim legend block (under "### Legend block to embed verbatim") add the line `- `FR<n>` = Frontend summary theme`.

- [ ] **Step 4: Verify**

Run: `Grep` for `Frontend`, `FR<n>`, `<Sln>-Frontend`, and `CQ-Frontend-Summary.md` in `agents/cq-domain-summary.md`.
Expected: domain set, glob table, prefix table, citations table, and legend block all include Frontend.

- [ ] **Step 5: Commit**

```bash
git add agents/cq-domain-summary.md
git commit -m "cq-domain-summary: add Frontend domain (FR prefix, solutions/*/Frontend.md)"
```

---

## Task 5: `cq-summary` — four domain sub-agents + FR nomenclature

**Files:**
- Modify: `agents/cq-summary.md` (MANDATORY DELIVERABLE list; Reference-nomenclature code table + legend; Phase A dispatch; Inputs; citations table)

- [ ] **Step 1: Add the Frontend summary to the deliverables and inputs**

In "MANDATORY DELIVERABLE", add `summaries\CQ-Frontend-Summary.md` to the list of per-domain files written by a `cq-domain-summary` sub-agent. In the Phase-B inputs list add `summaries\CQ-Frontend-Summary.md`.

- [ ] **Step 2: Add `FR` to the nomenclature code table and the legend block**

In the `<DD><n>` code table add `| `FR<n>` | Frontend summary theme | `Frontend-Summary` |`. In the citations short-name table add `| `solutions\<Sln>\Frontend.md` | `<Sln>-Frontend` |` and `| `summaries\CQ-Frontend-Summary.md` | `Frontend-Summary` |`. In the verbatim legend block add `- `FR<n>` = Frontend summary theme`.

- [ ] **Step 3: Dispatch four sub-agents in Phase A**

In "### 2. Phase A", change "three `cq-domain-summary` sub-agents" to **four** (Architecture, Frontend, CodeReview, TestReview), in a single parallel message. Update the prompt template's domain list and the "Joining and validation" check to expect four files. Update the strict write-order note ("Phase A dispatches three…") to four.

- [ ] **Step 4: Verify**

Run: `Grep` for `CQ-Frontend-Summary`, `FR<n>`, `Frontend-Summary`, and `four` in `agents/cq-summary.md`.
Expected: deliverables, legend, and Phase-A dispatch all reflect four domains.

- [ ] **Step 5: Commit**

```bash
git add agents/cq-summary.md
git commit -m "cq-summary: dispatch four domain summaries incl. Frontend (FR); add FR nomenclature"
```

---

## Task 6: `cq-management-summary` — fifth input + Frontend bucket mapping

**Files:**
- Modify: `agents/cq-management-summary.md` (Inputs; citation short-name table; Source-agent Category mapping; Process Step 2)

- [ ] **Step 1: Add the Frontend summary as an input**

In "## Inputs", add `CQ-Frontend-Summary.md` to the read list and to the short-name citation table as `| `CQ-Frontend-Summary.md` | `Frontend-Summary` |`. Update the "fewer than two of the four" guard to "fewer than two of the five".

- [ ] **Step 2: Add a Frontend-Architect Category → bucket mapping table**

After the existing per-agent mapping tables, add:

```markdown
**Frontend-Architect — primary bucket per Category:**

| Frontend Category | Primary bucket | Secondary (when to override) |
|---|---|---|
| MVVM | Maintainability | — |
| Binding | Maintainability | Readability if purely about call-site/XAML clarity |
| Commands | Maintainability | — |
| Threading | Reliability | — |
| Navigation/DI | Maintainability | — |
| Resources/Theming | Readability | — |
| Performance | Reliability | (desktop responsiveness — not portfolio "Scalability") |
| Leaks | Reliability | — |
| State management | Maintainability | — |
```

Add `FR<n>` to the summary-theme handling: FR themes are **production** code (never Test Quality) and route via this table.

- [ ] **Step 3: Add FR extraction to Process Step 2**

In "## Process", Step 2 extraction list, add: "From `Frontend-Summary`: `### FR<n>` themes."

- [ ] **Step 4: Verify**

Run: `Grep` for `Frontend-Summary`, `FR<n>`, and `Frontend Category` in `agents/cq-management-summary.md`.
Expected: input, citation table, mapping table, and extraction step all present.

- [ ] **Step 5: Commit**

```bash
git add agents/cq-management-summary.md
git commit -m "cq-management-summary: consume CQ-Frontend-Summary; map FR findings into prod buckets"
```

---

## Task 7: `cq-plan` — tier-aware discovery + ten plans

**Files:**
- Modify: `commands/cq-plan.md` (Step 4 summary filenames; Step 5 globs; Step 6 plan file list + 6a/6b; Step 7 handoff; Constraints)

- [ ] **Step 1: Add the Frontend summary to Step 4**

In "### 4. Run /cq-summary", add `summaries/CQ-Summary-Frontend.md` (from `solutions/*/Frontend.md`) to the override-named outputs, and note `CQ-Summary` now produces five summary files (four per-domain + top-level).

> **Pre-existing anomaly — do NOT fix here.** `cq-plan` Step 4 also lists `CQ-Summary-Data.md`, but `cq-summary`/`cq-domain-summary` define only three domains today (Architecture, CodeReview, TestReview) and do not dispatch a Data domain sub-agent. That Data-summary mismatch predates this work. Leave it exactly as-is — this task only **adds** the Frontend domain alongside the existing ones. If the user later wants the Data domain reconciled, that is a separate change.

- [ ] **Step 2: Add Frontend reports to Step 5 enumeration**

In "### 5", add `CQ-Reviews/solutions/*/Frontend.md` to the `Glob` enumeration and `CQ-Reviews/summaries/CQ-Summary-Frontend.md` to the summary enumeration.

- [ ] **Step 3: Add the two Frontend plan files in Step 6**

Add to the plan-file list:

```markdown
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-DETAILED-Frontend.md` (from `solutions/*/Frontend.md` reports)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-SUMMARY-Frontend.md` (from `summaries/CQ-Summary-Frontend.md`)
```

Update the prose that says "two sets of four" → "two sets of five" and "eight implementation plans" / "eight plan paths" → "ten" throughout (frontmatter description, intro, Step 6 heading, Step 7 handoff, Constraints). The DETAILED/SUMMARY task shapes in 6a/6b apply unchanged to the Frontend kind.

- [ ] **Step 4: Update the frontmatter description and Step 7 handoff**

Frontmatter `description:` → "…produce DETAILED + SUMMARY implementation plans per CQ kind (Architecture, Frontend, CodeReview, Data, TestReview)…". Step 7 lists ten plan paths grouped by kind.

- [ ] **Step 5: Verify**

Run: `Grep` for `Frontend`, `ten`, and `CQ-Summary-Frontend` in `commands/cq-plan.md`. Then `Grep` for the literal `eight` — expected: zero remaining matches that refer to the plan count.
Expected: five kinds and ten plans throughout; no stale "eight"/"four" plan-count references.

- [ ] **Step 6: Commit**

```bash
git add commands/cq-plan.md
git commit -m "cq-plan: add Frontend kind — five summaries, ten plans, tier-aware discovery"
```

---

## Task 8: README — document the new lens

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Read the README and locate where lenses/agents are enumerated**

Run: `Read` `README.md`. Find the section listing the agents / review lenses and the `CQ-Reviews` output layout.

- [ ] **Step 2: Add CQ-Frontend-Architect and the layout/backend split**

Document: `CQ-Architect` now produces Solution Layout + Backend (archetype-gated, 50x backend-only); `CQ-Frontend-Architect` produces a WPF frontend view at `solutions/<Solution>/Frontend.md` (WinForms/Razor/Blazor detected and labeled only); the Frontend summary domain (`FR`) and the tenth/eleventh plan files. Match the README's existing list/table style; do not restructure unrelated sections.

- [ ] **Step 3: Verify**

Run: `Grep` for `CQ-Frontend-Architect` and `Frontend.md` in `README.md`.
Expected: both present in the agent list and the output-layout description.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "README: document CQ-Frontend-Architect and the layout/backend split"
```

---

## Task 9: End-to-end consistency pass

**Files:** none modified unless a gap is found.

- [ ] **Step 1: Cross-file citation consistency**

Run: `Grep` (repo-wide, `agents/` + `commands/`) for `Frontend-Summary`, `<Sln>-Frontend`, and `CQ-Frontend-Summary`.
Expected: the short-name `<Sln>-Frontend` (per-unit) and `Frontend-Summary` (summary) are used consistently; no `CQ-` infix in per-unit citations; no `.md` suffix in citations.

- [ ] **Step 2: Domain-count consistency**

Run: `Grep` for `three` and `four` across `agents/cq-summary.md` and `agents/cq-domain-summary.md`.
Expected: no stale "three domain summaries" references; all say four.

- [ ] **Step 3: Plan-count consistency**

Run: `Grep` for `eight` across `commands/cq-plan.md`.
Expected: no remaining plan-count "eight" (all updated to ten).

- [ ] **Step 4: Walk the spec's acceptance criteria**

Re-read the spec's "Acceptance criteria" (1–6) and confirm each is satisfiable from the edited prompts: desktop-only → layout + "backend not applicable" + Frontend.md; backend-only → no Frontend.md; mixed → both + cross-tier flags; WinForms-only → one-paragraph Frontend.md; summary emits `CQ-Frontend-Summary.md` with FR themes consumed by management-summary; citations resolve as `<Sln>-Frontend §Findings #N`. Note any gap and add a fix task.

- [ ] **Step 5: Final commit (if anything was adjusted)**

```bash
git add -A
git commit -m "Frontend-view decomposition: final consistency pass"
```

---

## Out of scope / deferred

- No specialized WinForms / Razor / Blazor review (detected + labeled only).
- No standalone Solution-Layout agent (layout lives inside CQ-Architect).
- No build/render-script changes (publishing layer not shipped in this repo).
- The pre-existing `CQ-Summary-Data.md` mismatch in `cq-plan` (a Data domain summary requested but never produced by `cq-summary`'s three sub-agents) is left untouched — reconciling it is a separate change.
- Behavioral validation by running the agents against a real WPF solution is a manual follow-up, not part of this prompt-only plan.
