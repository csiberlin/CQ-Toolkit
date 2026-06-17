---
description: Generate the per-solution Purpose.md (business value, solution profile, domain glossary, and the user-owned Scale signals baseline) for every C# solution in the given subdirectories, via the CQ-Business-Value agent. Run this once before /cq-analyze, then fill in each Scale signals table.
argument-hint: [subdir1 subdir2 ...]
---

# /cq-purpose — generate the per-solution purpose baseline

**Stage 1** of the workflow (**`/cq-purpose`** → `/cq-analyze` → `/cq-summary` → `/cq-plan`). For every C# solution, ensure a `CQ-Reviews\solutions\<Solution-Name>\Purpose.md` exists — the *CQ-Purpose* report the downstream review agents read for business context and severity calibration. You generally run this **once per repository**; the files are preserved across re-scans.

## Arguments

`$ARGUMENTS` — optional space-separated list of subdirectories. If empty, work from the **current working directory** recursively.

## Steps

### 1. Discover solutions

Discover the set of solution files exactly as `/cq-analyze` does: `Glob` with patterns `**/*.sln` **and** `**/*.slnx` over each subdirectory in `$ARGUMENTS` (or from cwd when no args). Both the classic `.sln` and the new XML `.slnx` formats are supported. Exclude solution files under `.vs/`, `.git/`, `bin/`, `obj/`, `node_modules/`, and `.claude/worktrees/` (caches and isolated copies, not the canonical solution). If a folder has both a `.sln` and a `.slnx` for the same solution, treat them as one (prefer the `.slnx`).

### 2. Compute the expected purpose path per solution

- `<Solution-Name>` = LAST dot-separated segment of the solution file name, with the `.sln` / `.slnx` extension stripped.
  - `Acme.Research.Platform.MessagingApi.sln` → `solutions/MessagingApi/Purpose.md`
  - `ReportTool.slnx` → `solutions/ReportTool/Purpose.md`
  - `Foo.sln` → `solutions/Foo/Purpose.md`
- Expected path: `CQ-Reviews/solutions/<Solution-Name>/Purpose.md` (relative to the cwd).

### 3. Generate the missing purpose files

- For every solution whose expected purpose file is **missing**, dispatch a `CQ-Business-Value` agent. Launch them in a single message with multiple parallel `Agent` calls (batch in groups of 6–10 if there are many). Track each with `TaskCreate` / `TaskUpdate`.
  - `subagent_type: "CQ-Business-Value"`
  - Brief each agent with: the absolute path of the solution file (`.sln` / `.slnx`), that the working directory is the current shell cwd, and that the report MUST land at `<cwd>\CQ-Reviews\solutions\<Solution-Name>\Purpose.md` (the agent already follows this convention — just confirm it).
- Solutions whose purpose file **already exists** are skipped — they are not regenerated. (To force a regenerate, delete the file first; `CQ-Business-Value` preserves any operator-curated `## Scale signals` section verbatim even on a rewrite.)
- Wait for **every** dispatched agent to return. Then `Glob CQ-Reviews/solutions/*/Purpose.md` and confirm each expected file now exists. If any is still missing, surface the gap and stop.

### 4. Report — and tell the user to fill in Scale signals

For each solution, report the purpose-file path and the agent's **Scale-signals disposition**:

- `scaffolded` — a **blank** `## Scale signals` table was written. The user MUST open that `Purpose.md` and fill the table with the real **load baseline** (current req/s, concurrent users, device/customer counts, growth plans). These numbers are not in the code, so a human supplies them, and `CQ-Architect`'s 50x analysis treats them as its highest-priority source.
- `kept` — an existing `## Scale signals` section was preserved verbatim; nothing to do.
- already-present (skipped) — the file existed; report it unchanged.

End with the explicit next step: **fill in every `scaffolded` `## Scale signals` table, then run `/cq-analyze`** (or `/cq-all` to run the rest of the pipeline end-to-end).

## Constraints

- Read-only across the production source tree. The only files written are the `solutions/<Solution-Name>/Purpose.md` reports produced by `CQ-Business-Value`.
- Do NOT dispatch any other CQ agent here (no `CQ-Architect`, `CQ-Reviewer`, `CQ-Data`, `CQ-Test-Reviewer`, `CQ-Summary`). Those belong to `/cq-analyze` and later stages.
- Never overwrite an existing purpose file's operator-provided `## Scale signals` numbers — `CQ-Business-Value` handles that preservation; this command simply skips files that already exist.
- Ensure `CQ-Reviews\` exists in the cwd before dispatching (`Bash mkdir -p CQ-Reviews`).
