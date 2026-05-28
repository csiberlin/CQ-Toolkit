---
description: Scan all C# solutions in the given subdirectories (or current dir), then dispatch CQ-Architect per solution, CQ-Reviewer (+ CQ-Data where a data layer exists) per production project, and CQ-Test-Reviewer per test project.
argument-hint: [subdir1 subdir2 ...]
---

# /cq-scan — bulk CQ review across all C# projects

You are about to launch CQ review agents over every C# project in every C# solution found under the given subdirectories.

## Arguments

`$ARGUMENTS` — optional space-separated list of subdirectories to scan. If empty, scan the **current working directory** recursively.

## Steps

### 1. Discover solutions and projects

- Use `Glob` with pattern `**/*.sln` over each requested subdirectory (or `**/*.sln` from cwd when no args). Collect the absolute path of each `.sln` file.
- For each `.sln`, parse out the contained `.csproj` paths. The `.sln` contains lines like:
  `Project("{FAE04EC0-...}") = "ProjectName", "relative\path\Project.csproj", "{GUID}"`
  Extract the second quoted token, resolve it relative to the `.sln`'s folder, and de-duplicate the absolute paths across all solutions.
- Skip `.csproj` paths that do not exist on disk.

### 2. Classify each project

For each `.csproj` decide **test** vs **production**:

- **Test project** if ANY of these is true:
  - File name matches (case-insensitive) `*.Tests.csproj`, `*.Test.csproj`, `*Tests.csproj`, `*.UnitTests.csproj`, `*.IntegrationTests.csproj`.
  - The `.csproj` content contains `<IsTestProject>true</IsTestProject>` or a `PackageReference` to any of: `Microsoft.NET.Test.Sdk`, `xunit`, `NUnit`, `MSTest.TestFramework`.
- Otherwise → **production**.

Read each `.csproj` once with `Read` to do this classification.

### 3. Dispatch agents in parallel

The review unit differs by lens: **Architecture is per solution**, the **code lenses are per project**. Create a `TaskCreate` task per unit of work so progress is visible, then **launch agents in a single message with multiple `Agent` tool calls** so they run concurrently.

Per **solution** (`.sln`), spawn ONE agent:
- `subagent_type: "CQ-Architect"` — architecture review of the whole solution.

For each **production** project, spawn:
- `subagent_type: "CQ-Reviewer"` — code-quality review (always).
- `subagent_type: "CQ-Data"` — data-layer review, **only if** the project's `.csproj` references a data-access package (`Microsoft.EntityFrameworkCore.*`, `Dapper*`, `NHibernate`, `Npgsql`, `Microsoft.Data.SqlClient`, `System.Data.SqlClient`, `MySql.Data`, `Oracle.ManagedDataAccess.*`, `RepoDB*`, `linq2db`). Skip it for projects with no data layer.

For each **test** project, spawn ONE agent:
- `subagent_type: "CQ-Test-Reviewer"` — test-quality review.

Each `Agent` prompt MUST tell the agent that the **working directory** is the current shell working directory (the agents resolve the `<working-directory>` placeholder to it), plus:
- **CQ-Architect** — the absolute path of the `.sln` and its `<Solution-Name>` (last dot-separated segment of the `.sln` file name, extension stripped). It writes `<cwd>\CQ-Reviews\solutions\<Solution-Name>\Architect.md`.
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
