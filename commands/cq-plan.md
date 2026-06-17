---
description: Produce ten reviewable implementation plans (DETAILED + SUMMARY per CQ kind — Architecture, Frontend, CodeReview, Data, TestReview) from the existing CQ analysis files and cross-solution summaries. Run after /cq-analyze and /cq-summary (or use /cq-all to run the whole pipeline).
argument-hint: [working-subdir]
---

# /cq-plan — implementation plans from existing CQ reports

**Stage 4** of the workflow (`/cq-purpose` → `/cq-analyze` → `/cq-summary` → **`/cq-plan`**). Synthesises **two sets of five reviewable implementation plans** (one set derived from the per-project/solution analysis files, one set derived from the cross-solution summaries) for user review **before any code changes**. This command only *plans* — it consumes reports that already exist and never regenerates them. To run the full pipeline end-to-end, use **`/cq-all`**.

## Arguments

`$ARGUMENTS` — optional subdirectory to treat as the working directory. Default to the current directory. Reports and plans live under `<working-directory>\CQ-Reviews\`.

## Preconditions

Both input sets must already exist under `<working-directory>\CQ-Reviews\`:

- **Per-unit analysis files** from `/cq-analyze` — `solutions/<Solution>/Architect.md` (and `Frontend.md` where present), `projects/<Project>/{CodeReview,Data,TestReview}.md`.
- **Cross-solution summaries** from `/cq-summary` — the five `summaries/CQ-Summary-<Domain>.md` files.

`Glob` for both at the start. If the analysis files are missing, stop and tell the user to run `/cq-analyze`. If the summaries are missing, stop and tell the user to run `/cq-summary`. (Or run `/cq-all`, which chains every stage.) Do **not** regenerate either set from this command.

## Steps

### 1. Read the per-project reports AND the summaries

- Use `Glob` with `CQ-Reviews/solutions/*/Architect.md`, `CQ-Reviews/solutions/*/Frontend.md`, `CQ-Reviews/projects/*/CodeReview.md`, `CQ-Reviews/projects/*/Data.md`, `CQ-Reviews/projects/*/TestReview.md` to enumerate every per-unit report.
- Also enumerate the five `CQ-Reviews/summaries/CQ-Summary-<Domain>.md` files (`CQ-Summary-Architecture.md`, `CQ-Summary-Frontend.md`, `CQ-Summary-CodeReview.md`, `CQ-Summary-Data.md`, `CQ-Summary-TestReview.md`).
- Read each per-unit report and each summary. For each kind keep two structured tallies:
  - **DETAILED tally** (per-unit): one entry per finding `{ unit, severity, category, summary, suggested_fix, source_report_path, source_section }`. Group by category so the plan addresses themes rather than individual lines.
  - **SUMMARY tally** (cross-solution): one entry per promoted theme `{ theme_id (e.g. AR3), affected_solutions, severity, category, evidence_links }` — exactly the structure the summary already uses.

### 2. Create ten implementation plans (5 DETAILED + 5 SUMMARY)

Use the `superpowers:writing-plans` skill style. Write the plans directly with the `Write` tool — do NOT spawn sub-agents for this. The ten files live at:

- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-DETAILED-Architecture.md` (from `solutions/*/Architect.md` reports)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-DETAILED-Frontend.md` (from `solutions/*/Frontend.md` reports)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-DETAILED-CodeReview.md` (from `projects/*/CodeReview.md` reports)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-DETAILED-Data.md` (from `projects/*/Data.md` reports)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-DETAILED-TestReview.md` (from `projects/*/TestReview.md` reports)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-SUMMARY-Architecture.md` (from `summaries/CQ-Summary-Architecture.md`)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-SUMMARY-Frontend.md` (from `summaries/CQ-Summary-Frontend.md`)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-SUMMARY-CodeReview.md` (from `summaries/CQ-Summary-CodeReview.md`)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-SUMMARY-Data.md` (from `summaries/CQ-Summary-Data.md`)
- `CQ-Reviews/plans/IMPLEMENTATION-PLAN-SUMMARY-TestReview.md` (from `summaries/CQ-Summary-TestReview.md`)

#### 2a. DETAILED plans

For each DETAILED plan:

- Open with a **Scope** section listing each project reviewed for that kind and a count of findings addressed vs. deferred.
- Group work into **Phases** ordered by risk-adjusted value:
  1. Cheap, high-confidence cleanups (naming, dead code, formatting drift, missing braces).
  2. Foundational cross-cutting changes (e.g. logging spine, connection factory) that unblock later work.
  3. Single-file refactors (extract method, split large class, tighten visibility).
  4. Cross-file structural changes (move type, introduce interface, rename across XAML+C#).
  5. Behavioural changes that need extra test coverage before they ship.
- For every phase enumerate concrete **Tasks**, each one carrying:
  - Affected file(s) with absolute paths.
  - Originating finding (link back to the source report by short name, e.g. `<Project>-CodeReview §<section>` or `<Solution>-Architect §<section>`).
  - Acceptance criteria (what unit test, build, or coverage signal will prove it).
  - Whether new tests are required (per CLAUDE.md coverage mandate — almost always yes).
- Close with an **Out of Scope / Deferred** section listing findings intentionally skipped, with a one-line reason each.

#### 2b. SUMMARY plans

For each SUMMARY plan:

- Open with the theme count and short note that this is the theme-level plan and that the per-finding breakdown lives in the matching DETAILED plan.
- Produce **one task per promoted theme** (typically 5–8 tasks total). Each task references:
  - The theme code (e.g. `AR3`, `CR1`) and affected solutions.
  - The matching DETAILED-plan task IDs (so a reader can drill down).
  - Approach in one short paragraph (no per-file enumeration here — that lives in the DETAILED plan).
  - Acceptance criteria at the theme level (the cross-solution outcome, not per-file build signals).
- Close with a brief **Out of Scope** note pointing at the DETAILED plan for the per-finding gaps that didn't promote to themes.

#### 2c. Cross-plan task de-duplication

Several themes overlap across plans (e.g. `TimeProvider` injection appears in CodeReview-CR7, Architecture-AR6, and TestReview-TR1). For each shared task:

- Name it once with a stable identifier (e.g. `S-CR7` or `4.10`).
- In every other plan, reference it explicitly as "shared task — execute once" rather than re-describing the work.

#### 2d. Project-rule honour

Honour every project rule in `CLAUDE.md` across all ten plans:

- **Coverage ratchet:** no untested production change. Each refactor task lists its companion test work.
- **Skills:** `clean-csharp`, `unit-testing`, `aspnet-minimal-api`, `services-and-di`, `wpf-command-dialog`, `devexpress-*` skills are referenced in tasks that touch the relevant areas.
- **Roslyn refactor MCP tools** for any rename / move / extract — call this out so execution doesn't regress to text edits.

### 3. Hand off for review

- Do NOT begin implementation. Print a concise summary to the user:
  - The ten plan paths grouped by kind (Architecture, Frontend, CodeReview, Data, TestReview).
  - For each kind, the finding count, theme count, and task counts in DETAILED vs SUMMARY.
  - Top 3 findings across the whole portfolio by severity (cite the source report).
  - Recommendation: read the SUMMARY plans first to align on direction, then drill into DETAILED plans for execution.
  - The exact next command the user should run after approving a plan (e.g. `Skill superpowers:executing-plans` or `/gsd:plan-phase`).
- Wait for the user's review and explicit go-ahead before any edit.

## Constraints

- **Read-only except for the ten plan files in Step 2.** No production source files, analysis reports, or summaries may be modified — this command consumes the existing reports and writes only plans.
- Do not invoke any CQ review/summary agent here (`CQ-Business-Value`, `CQ-Architect`, `CQ-Reviewer`, `CQ-Data`, `CQ-Test-Reviewer`, `CQ-Summary`, `CQ-Management-Summary`, `CQ-HTML-Publisher`). Their outputs are inputs to this command; regenerate them via `/cq-analyze` / `/cq-summary` / `/cq-all` if they are stale.
- If a given kind's analysis produced zero reports, write a one-paragraph placeholder plan for both that kind's DETAILED and SUMMARY files explaining "no findings — nothing to plan" rather than skipping the file (downstream tooling expects all ten paths to exist). The same applies when reports exist but contain zero findings — a legitimate result under the reviewers' value bar: the plan for that kind is a short "reviewed clean — nothing to plan" note, not a padded list. Do NOT treat an empty `## Recommended Actions` or empty `## Findings` section as an error, and do NOT pull items from a report's `## Optional / stylistic (below the value bar)` section into a plan — those failed the value bar by design.
- If a per-domain summary file is missing, do NOT generate the corresponding SUMMARY plan — surface the error and stop (run `/cq-summary` to produce it).
- The plans are proposals: every task in them must be reversible until the user approves execution.
- Match the summary filenames on disk. `/cq-summary` writes `CQ-Summary-<Domain>.md` under `CQ-Reviews/summaries/`; verify the files exist with that naming before proceeding to Step 2.
