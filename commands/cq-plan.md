---
description: Clear CQ-Reviews, run /cq-scan, run /cq-summary, then produce DETAILED + SUMMARY implementation plans per CQ kind (Architecture, CodeReview, Data, TestReview) for user review before any code changes.
argument-hint: [subdir1 subdir2 ...]
---

# /cq-plan — full CQ review → eight implementation plans pipeline

End-to-end orchestrator: wipe prior CQ reports, run a fresh scan, run cross-solution summary, then synthesize **two sets of four reviewable implementation plans** (one per CQ kind, one set derived from per-project reports and one set derived from the cross-solution summary).

## Arguments

`$ARGUMENTS` — optional space-separated list of subdirectories forwarded verbatim to `/cq-scan`. If empty, scans the current working directory.

## Steps

### 1. Clear the target directory (preserving Purpose files)

- Confirm `CQ-Reviews/` exists at the cwd. If yes, delete its contents **except** any `*-CQ-Purpose.md` files (those are produced by `CQ-Business-Value` and are reused as input by the scan agents — re-generating them on every run is wasteful and they rarely go stale).
  - On bash: `Bash find CQ-Reviews -mindepth 1 -maxdepth 1 ! -name '*-CQ-Purpose.md' -exec rm -rf {} +`
- If `CQ-Reviews/` does not exist, create it with `Bash mkdir -p CQ-Reviews`.
- Verify only `*-CQ-Purpose.md` files (if any) remain (`Bash ls CQ-Reviews`).

### 2. Ensure a CQ-Purpose file exists for every solution

The downstream CQ agents (CQ-Architect, CQ-Reviewer, CQ-Data, CQ-Test-Reviewer) expect a `<Solution-Name>-CQ-Purpose.md` file to be present in `CQ-Reviews/` for the solution that contains the project under review. Generate any that are missing **before** dispatching the scan.

- Discover the set of `.sln` files exactly as `/cq-scan` does: `Glob` with pattern `**/*.sln` over each subdirectory in `$ARGUMENTS` (or `**/*.sln` from cwd when no args).
- For each `.sln`, compute the expected purpose file name:
  - `<Solution-Name>` = LAST dot-separated segment of the `.sln` file name, with `.sln` stripped.
    - `Tke.Bbx.Des.CommunicationApi.sln` → `CommunicationApi-CQ-Purpose.md`
    - `Foo.sln` → `Foo-CQ-Purpose.md`
  - Expected path: `CQ-Reviews/<Solution-Name>-CQ-Purpose.md`
- For every `.sln` whose expected purpose file is **missing** from `CQ-Reviews/`, dispatch a `CQ-Business-Value` agent (single message, multiple parallel `Agent` calls — batch in groups of 6–10 if there are many).
  - `subagent_type: "CQ-Business-Value"`
  - Brief each agent with: absolute path of the `.sln`, that the working directory is the current shell cwd, and that the report MUST land at `<cwd>\CQ-Reviews\<Solution-Name>-CQ-Purpose.md` (the agent already follows this convention — just confirm it).
- Track each dispatched agent with `TaskCreate`/`TaskUpdate`. Wait for **every** Business-Value agent to return before proceeding to Step 3.
- After they return, verify each expected purpose file now exists with `Glob CQ-Reviews/*-CQ-Purpose.md`. If any expected file is still missing, surface the gap and stop — do not run the scan against an incomplete purpose set.
- If every expected purpose file is already present at the start of this step (e.g. preserved from a prior run), skip the dispatch entirely and continue to Step 3.

### 3. Run /cq-scan

- Invoke the existing `/cq-scan` workflow by using the `Skill` tool with `skill: "cq-scan"` and `args: "$ARGUMENTS"`. Follow its instructions to completion: discover solutions, classify projects, dispatch CQ agents in parallel batches, and wait for every spawned agent to return.
- Do NOT proceed to Step 4 until every dispatched `Agent` call has returned. Track them via `TaskCreate`/`TaskUpdate` as `/cq-scan` prescribes.
- After scan, the `CQ-Reviews/` directory should contain per-project reports of the form `<Project>-CQ-<Kind>.md` for `Kind ∈ { Architect, Codereview, Data, Testreview }`.

### 4. Run /cq-summary

- Dispatch a `CQ-Summary` agent via the `Agent` tool. Brief it to produce **all four** per-domain summaries plus the top-level cross-domain summary, **using the file naming `CQ-Summary-<Domain>.md` and `CQ-Summary.md`** (NOT the agent's default `CQ-<Domain>-Summary.md` — be explicit, the agent honours an explicit override):
  - `CQ-Summary-Architecture.md` (from `*-CQ-Architect.md`)
  - `CQ-Summary-CodeReview.md` (from `*-CQ-Codereview.md`)
  - `CQ-Summary-Data.md` (from `*-CQ-Data.md` — produce even when only one project has findings; the others are 0-finding attestations)
  - `CQ-Summary-TestReview.md` (from `*-CQ-Testreview.md`)
  - `CQ-Summary.md` (top-level cross-domain summary consuming all four)
- Do NOT proceed to Step 5 until the agent returns.

### 5. Read the per-project reports AND the summaries

After scan + summary finish:

- Use `Glob` with `CQ-Reviews/*-CQ-Architect.md`, `CQ-Reviews/*-CQ-Codereview.md`, `CQ-Reviews/*-CQ-Data.md`, `CQ-Reviews/*-CQ-Testreview.md` to enumerate every per-project report.
- Also enumerate the four `CQ-Summary-<Domain>.md` files written in Step 3.
- Read each per-project report and each summary. For each kind keep two structured tallies:
  - **DETAILED tally** (per-project): one entry per finding `{ project, severity, category, summary, suggested_fix, source_report_path, source_section }`. Group by category so the plan addresses themes rather than individual lines.
  - **SUMMARY tally** (cross-solution): one entry per promoted theme `{ theme_id (e.g. AR3), affected_solutions, severity, category, evidence_links }` — exactly the structure the summary already uses.

### 6. Create eight implementation plans (4 DETAILED + 4 SUMMARY)

Use the `superpowers:writing-plans` skill style. Write the plans directly with the `Write` tool — do NOT spawn sub-agents for this. The eight files live at:

- `CQ-Reviews/IMPLEMENTATION-PLAN-DETAILED-Architecture.md` (from `*-CQ-Architect.md` reports)
- `CQ-Reviews/IMPLEMENTATION-PLAN-DETAILED-CodeReview.md` (from `*-CQ-Codereview.md` reports)
- `CQ-Reviews/IMPLEMENTATION-PLAN-DETAILED-Data.md` (from `*-CQ-Data.md` reports)
- `CQ-Reviews/IMPLEMENTATION-PLAN-DETAILED-TestReview.md` (from `*-CQ-Testreview.md` reports)
- `CQ-Reviews/IMPLEMENTATION-PLAN-SUMMARY-Architecture.md` (from `CQ-Summary-Architecture.md`)
- `CQ-Reviews/IMPLEMENTATION-PLAN-SUMMARY-CodeReview.md` (from `CQ-Summary-CodeReview.md`)
- `CQ-Reviews/IMPLEMENTATION-PLAN-SUMMARY-Data.md` (from `CQ-Summary-Data.md`)
- `CQ-Reviews/IMPLEMENTATION-PLAN-SUMMARY-TestReview.md` (from `CQ-Summary-TestReview.md`)

#### 6a. DETAILED plans

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
  - Originating finding (link back to `<Project>-CQ-<Kind>.md §<section>`).
  - Acceptance criteria (what unit test, build, or coverage signal will prove it).
  - Whether new tests are required (per CLAUDE.md coverage mandate — almost always yes).
- Close with an **Out of Scope / Deferred** section listing findings intentionally skipped, with a one-line reason each.

#### 6b. SUMMARY plans

For each SUMMARY plan:

- Open with the theme count and short note that this is the theme-level plan and that the per-finding breakdown lives in the matching DETAILED plan.
- Produce **one task per promoted theme** (typically 5–8 tasks total). Each task references:
  - The theme code (e.g. `AR3`, `CR1`) and affected solutions.
  - The matching DETAILED-plan task IDs (so a reader can drill down).
  - Approach in one short paragraph (no per-file enumeration here — that lives in the DETAILED plan).
  - Acceptance criteria at the theme level (the cross-solution outcome, not per-file build signals).
- Close with a brief **Out of Scope** note pointing at the DETAILED plan for the per-finding gaps that didn't promote to themes.

#### 6c. Cross-plan task de-duplication

Several themes overlap across plans (e.g. `TimeProvider` injection appears in CodeReview-CR7, Architecture-AR6, and TestReview-TR1). For each shared task:

- Name it once with a stable identifier (e.g. `S-CR7` or `4.10`).
- In every other plan, reference it explicitly as "shared task — execute once" rather than re-describing the work.

#### 6d. Project-rule honour

Honour every project rule in `CLAUDE.md` across all eight plans:

- **Coverage ratchet:** no untested production change. Each refactor task lists its companion test work.
- **Skills:** `clean-csharp`, `unit-testing`, `aspnet-minimal-api`, `services-and-di`, `wpf-command-dialog`, `devexpress-*` skills are referenced in tasks that touch the relevant areas.
- **Roslyn refactor MCP tools** for any rename / move / extract — call this out so execution doesn't regress to text edits.

### 7. Hand off for review

- Do NOT begin implementation. Print a concise summary to the user:
  - The eight plan paths grouped by kind.
  - For each kind, the finding count, theme count, and task counts in DETAILED vs SUMMARY.
  - Top 3 findings across the whole portfolio by severity (cite the source report).
  - Recommendation: read the SUMMARY plans first to align on direction, then drill into DETAILED plans for execution.
  - The exact next command the user should run after approving a plan (e.g. `Skill superpowers:executing-plans` or `/gsd:plan-phase`).
- Wait for the user's review and explicit go-ahead before any edit.

## Constraints

- **Read-only across Steps 1–6** except for:
  - deleting prior reports in Step 1 (Purpose files are preserved),
  - the purpose files produced by `CQ-Business-Value` agents in Step 2,
  - the report files produced by `/cq-scan` agents in Step 3,
  - the summary files produced by `/cq-summary` in Step 4,
  - the eight plan files in Step 6.
  No production source files may be modified.
- Do not invoke `CQ-HTML-Publisher` or `CQ-Management-Summary` here — those are separate workflows. `CQ-Business-Value` IS invoked in Step 2, but only for solutions whose `*-CQ-Purpose.md` file is missing.
- If `/cq-scan` produces zero reports of a given kind, write a one-paragraph placeholder plan for both that kind's DETAILED and SUMMARY files explaining "no findings — nothing to plan" rather than skipping the file (downstream tooling expects all eight paths to exist).
- If `/cq-summary` fails to produce a per-domain summary file, do NOT generate the corresponding SUMMARY plan — surface the error and stop.
- The plans are proposals: every task in them must be reversible until the user approves execution.
- Match the agent's actual summary filename. The current `CQ-Summary` agent default is `CQ-<Domain>-Summary.md`; this command **overrides** that to `CQ-Summary-<Domain>.md` via an explicit instruction in the Step 4 prompt. Verify the files on disk match before proceeding to Step 6.
