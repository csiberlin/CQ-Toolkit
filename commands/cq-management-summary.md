---
description: Regroup the existing CQ cross-solution summaries into six executive management briefs (one per quality attribute) via the CQ-Management-Summary agent. Run after the CQ-Reviews summaries exist.
argument-hint: [working-subdir]
---

# /cq-management-summary — quality-attribute management briefs

Regroups the cross-solution CQ summaries into six tight, decision-ready briefs — **Scalability, Readability, Maintainability, Security, Reliability, Test Quality** — for a tech lead deciding what to fund next. Each brief has a Goal, a "Risk if you do nothing" section, and a ranked Actions list.

This is a **separate workflow** from `/cq-plan`. `/cq-plan` produces implementation plans; this command produces management briefs from the summaries `/cq-plan` (or a standalone `CQ-Summary` run) already wrote. Run it *after* the summaries exist — it never regenerates them.

## Preconditions

- The cross-solution summaries must already exist under `<working-directory>\CQ-Reviews\summaries\`, produced by the `CQ-Summary` agent (e.g. during `/cq-plan` Step 4). If they are missing, run `/cq-plan` first.
- Read-only except for the six brief files the agent writes into `summaries\`. No production source files are modified.

## Steps

1. **Resolve the working directory.** Default to the current directory. If `$ARGUMENTS` names a subdirectory, treat it as the working directory. The summaries live under `<working-directory>\CQ-Reviews\summaries\`.

2. **Locate the input summaries.** `Glob` `CQ-Reviews/summaries/*.md` and identify the four cross-solution summaries. **Two naming conventions are in circulation — accept either:**

   | Domain | Default name (standalone `CQ-Summary`) | `/cq-plan` override name |
   |---|---|---|
   | Architecture | `CQ-Architecture-Summary.md` | `CQ-Summary-Architecture.md` |
   | CodeReview | `CQ-CodeReview-Summary.md` | `CQ-Summary-CodeReview.md` |
   | TestReview | `CQ-TestReview-Summary.md` | `CQ-Summary-TestReview.md` |
   | Top-level | `CQ-Summary.md` | `CQ-Summary.md` |

   If fewer than two of the four are present, **stop** and report which are missing — do not dispatch the agent.

3. **Dispatch the agent.** Invoke the `CQ-Management-Summary` agent via the `Agent` tool. Brief it with:
   - the working directory, and
   - the **exact input filenames you resolved in Step 2** (the agent honours an explicit override of its default `CQ-<Domain>-Summary.md` input names — be explicit so it reads the right files regardless of which producer ran).

   The agent writes the six briefs — `CQ-Scalability.md`, `CQ-Readability.md`, `CQ-Maintainability.md`, `CQ-Security.md`, `CQ-Reliability.md`, `CQ-TestQuality.md` — into `CQ-Reviews\summaries\`. Wait for it to return; do not proceed until it does.

4. **Verify & report.** `Glob` `CQ-Reviews/summaries/` and confirm all six brief files are present. Report the six absolute paths and, for each, the count of Top-6 actions the agent emitted. If any file is missing, surface the error and stop — do not claim success without the `Glob` proof.

## Constraints

- Consumes existing summaries only — never re-run `/cq-scan` or regenerate the summaries from this command.
- Do not invoke this inside `/cq-plan`; it is a deliberately separate workflow.
- The six briefs are the deliverable. Your final reply is a short confirmation (the six paths + per-file action counts), not the brief bodies inline.
