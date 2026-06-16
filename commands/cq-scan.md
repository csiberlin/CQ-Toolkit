---
description: Scan all C# solutions in the given subdirectories (or current dir), then dispatch CQ-Architect (solution-layout + backend) per solution, CQ-Frontend-Architect per solution that has a WPF (Frontend) project, CQ-Reviewer (+ CQ-Data where a data layer exists) per production project, and CQ-Test-Reviewer per test project.
argument-hint: [subdir1 subdir2 ...]
---

# /cq-scan — bulk CQ review across all C# projects

You are about to launch CQ review agents over every C# project in every C# solution found under the given subdirectories.

## Arguments

`$ARGUMENTS` — optional space-separated list of subdirectories to scan. If empty, scan the **current working directory** recursively.

## Steps

### 1. Discover solutions and projects

- Use `Glob` to find **both** solution formats: pattern `**/*.sln` **and** `**/*.slnx` over each requested subdirectory (or from cwd when no args). Collect the absolute path of each solution file.
- **Exclude tooling / output / isolated-copy directories** from the results: skip any solution file whose path contains a `.vs/`, `.git/`, `bin/`, `obj/`, `node_modules/`, or `.claude/worktrees/` segment. Visual Studio caches a copy of the `.slnx` under `.vs/`, and git worktrees hold isolated copies — neither is the canonical solution, and reviewing them duplicates work.
- For each remaining solution file, parse out the contained `.csproj` paths **according to its format**:
  - **Classic `.sln`** (MSBuild text format) — contains lines like:
    `Project("{FAE04EC0-...}") = "ProjectName", "relative\path\Project.csproj", "{GUID}"`
    Extract the **second** quoted token (the path). Ignore solution-folder rows — their path has no `.csproj` extension.
  - **New `.slnx`** (XML format) — `Read` the file and parse it as XML. It contains `<Project Path="relative/path/Project.csproj" />` elements (sometimes nested inside `<Folder>…</Folder>`). Extract the `Path` attribute of every `<Project>` element whose value ends in `.csproj`. Do NOT apply the `.sln` line-grep above to a `.slnx`.
  - In both cases, resolve each extracted path relative to the **solution file's** folder, and de-duplicate the absolute `.csproj` paths across all solutions.
- If a folder contains both a `.sln` and a `.slnx` for the same solution, **prefer the `.slnx`** and skip the matching `.sln` so the solution is not reviewed twice.
- Skip `.csproj` paths that do not exist on disk.

### 2. Classify each project

For each `.csproj` decide **test** vs **production**:

- **Test project** if ANY of these is true:
  - File name matches (case-insensitive) `*.Tests.csproj`, `*.Test.csproj`, `*Tests.csproj`, `*.UnitTests.csproj`, `*.IntegrationTests.csproj`.
  - The `.csproj` content contains `<IsTestProject>true</IsTestProject>` or a `PackageReference` to any of: `Microsoft.NET.Test.Sdk`, `xunit`, `NUnit`, `MSTest.TestFramework`.
- Otherwise → **production**.

For each **production** `.csproj`, also assign a **tier** (read the `.csproj` once — reuse the read from the test/production check):

- **Backend** — SDK is `Microsoft.NET.Sdk.Web`, OR it references `Microsoft.AspNetCore.*`, OR it references `Microsoft.Extensions.Hosting` (worker / service host).
- **Frontend (WPF)** — `<UseWPF>true</UseWPF>` is set, OR it references `PresentationFramework` / `DevExpress.Xpf.*`. (Fallback: if neither is present, a quick `Glob` of the project folder for `*.xaml` is permitted as a tie-breaker.)
- **Frontend (other)** — `<UseWindowsForms>true</UseWindowsForms>` (WinForms), Blazor, or Razor/MVC. **Labeled only** — no specialized lens runs on it.
- **Library** — a production project matching none of the above (class library, domain, shared kernel).

From the per-project tiers, derive the **solution archetype** for each solution: `backend-only` | `desktop-only` | `mixed` (two or more of the backend / WPF / other-UI tiers present) | `library-only`. Record the tier map and archetype — they are passed to `CQ-Architect` and decide whether `CQ-Frontend-Architect` runs.

Read each `.csproj` once with `Read` to do this classification.

### 3. Dispatch agents in parallel

The review unit differs by lens: **Architecture is per solution**, the **code lenses are per project**. Create a `TaskCreate` task per unit of work so progress is visible, then **launch agents in a single message with multiple `Agent` tool calls** so they run concurrently.

Per **solution** (`.sln` or `.slnx`):
- `subagent_type: "CQ-Architect"` — solution-layout + backend architecture review. Brief it with the **solution archetype** and the **tier map** (which projects are backend / WPF / other-UI / library) in addition to the solution path and `<Solution-Name>`. It writes `<cwd>\CQ-Reviews\solutions\<Solution-Name>\Architect.md`.
- `subagent_type: "CQ-Frontend-Architect"` — **only when the solution has at least one Frontend (WPF) project.** Brief it with the WPF project path(s), the `<Solution-Name>`, and the working directory. It writes `<cwd>\CQ-Reviews\solutions\<Solution-Name>\Frontend.md`. Skip this agent entirely for solutions whose only frontend is "other" tech (WinForms/Razor/Blazor) — the layout view notes those.

For each **production** project, spawn:
- `subagent_type: "CQ-Reviewer"` — code-quality review (always).
- `subagent_type: "CQ-Data"` — data-layer review, **only if** the project's `.csproj` references a data-access package (`Microsoft.EntityFrameworkCore.*`, `Dapper*`, `NHibernate`, `Npgsql`, `Microsoft.Data.SqlClient`, `System.Data.SqlClient`, `MySql.Data`, `Oracle.ManagedDataAccess.*`, `RepoDB*`, `linq2db`). Skip it for projects with no data layer.

For each **test** project, spawn ONE agent:
- `subagent_type: "CQ-Test-Reviewer"` — test-quality review.

Each `Agent` prompt MUST tell the agent that the **working directory** is the current shell working directory (the agents resolve the `<working-directory>` placeholder to it), plus:
- **CQ-Architect** — the absolute path of the solution file (`.sln` or `.slnx`) and its `<Solution-Name>` (last dot-separated segment of the solution file name, with the `.sln` / `.slnx` extension stripped — e.g. `GxReport.slnx` → `GxReport`, `Tke.Bbx.Des.CommunicationApi.sln` → `CommunicationApi`). It writes `<cwd>\CQ-Reviews\solutions\<Solution-Name>\Architect.md`.
- **CQ-Reviewer / CQ-Data / CQ-Test-Reviewer** — the absolute path of the target `.csproj`, its `<Project-Name>` (`.csproj` filename without extension), and the `<Solution-Name>` of the owning solution (used for the Purpose lookup and severity calibration). They write `<cwd>\CQ-Reviews\projects\<Project-Name>\<Lens>.md` (`<Lens>` ∈ `CodeReview` / `Data` / `TestReview`).

Be mindful of parallelism: if there are many units, batch the `Agent` calls in groups of 6–10 per message rather than firing hundreds at once.

### 4. Wait and report

After every batch returns, collect the report paths each agent emitted and present a summary table:

| Unit (solution / project) | Kind | Report |
|---------------------------|------|--------|

Also surface the solution-wide **coverage gap**: list any production project that has **no** corresponding test project — this per-project review can't see that, so report it here.

Mark every `TaskCreate` task as `completed` as its agent returns. Do not run `CQ-Summary` automatically — the user can invoke that separately when all per-project reports are in place.

## Constraints

- Do NOT generate or modify production code during this command — read-only review only.
- Do NOT invoke `CQ-Business-Value`, `CQ-Summary`, `CQ-Domain-Summary`, `CQ-Management-Summary`, or `CQ-HTML-Publisher` here. Those are separate workflows.
- Ensure the `CQ-Reviews\` directory exists in the current working directory before dispatching (use `Bash mkdir -p CQ-Reviews`).
