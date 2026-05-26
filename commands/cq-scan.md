---
description: Scan all C# solutions in the given subdirectories (or current dir) for C# projects, then dispatch CQ-Data + CQ-Architect + CQ-Reviewer per production project and CQ-Test-Reviewer per test project.
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

Create a `TaskCreate` task per project so progress is visible, then **launch all agents in a single message with multiple `Agent` tool calls** so they run concurrently.

For each **production** project, spawn THREE agents in parallel:
- `subagent_type: "CQ-Data"` — data-layer review
- `subagent_type: "CQ-Architect"` — architecture review
- `subagent_type: "CQ-Reviewer"` — code-quality review

For each **test** project, spawn ONE agent:
- `subagent_type: "CQ-Test-Reviewer"` — test-quality review

Each `Agent` prompt MUST tell the agent:
- The absolute path of the `.csproj` to analyze.
- That the **working directory** is the current shell working directory and reports must be written to `<cwd>\CQ-Reviews\<ProjectName>-CQ-<Kind>.md` (the agents already follow this convention via the `<working-directory>` placeholder in their definitions).
- `<ProjectName>` is the `.csproj` filename without extension.

Be mindful of parallelism: if there are many projects, batch the `Agent` calls in groups of 6–10 per message rather than firing hundreds at once.

### 4. Wait and report

After every batch returns, collect the report paths each agent emitted and present a summary table:

| Project | Kind | Report |
|---------|------|--------|

Mark every `TaskCreate` task as `completed` as its agent returns. Do not run `CQ-Summary` automatically — the user can invoke that separately when all per-project reports are in place.

## Constraints

- Do NOT generate or modify production code during this command — read-only review only.
- Do NOT invoke `CQ-Business-Value`, `CQ-Summary`, `CQ-Domain-Summary`, `CQ-Quality-Attributes`, or `CQ-HTML-Publisher` here. Those are separate workflows.
- Ensure the `CQ-Reviews\` directory exists in the current working directory before dispatching (use `Bash mkdir -p CQ-Reviews`).
