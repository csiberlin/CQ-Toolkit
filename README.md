# CQ-Toolkit

A Claude Code plugin that bundles a family of **C**ode-**Q**uality review agents and orchestration commands for C# solutions.

## What it does

CQ-Toolkit reads a C# repository (one or many `.sln` files) and produces a tiered set of reports:

1. **Per-solution purpose** — what the solution does and why it exists (`CQ-Business-Value`).
2. **Reviews** — Architecture per solution (`CQ-Architect`); Data and Code quality per production project (`CQ-Data`, `CQ-Reviewer`); Test quality per test project (`CQ-Test-Reviewer`).
3. **Cross-solution summaries** — one per domain plus a top-level brief (`CQ-Summary`, `CQ-Domain-Summary`).
4. **Management summary** — findings regrouped by quality attribute: Scalability, Readability, Maintainability, Security, Reliability, Test Quality (`CQ-Management-Summary`).
5. **Static HTML site** — browsable index of all the reports (`CQ-HTML-Publisher`).
6. **Implementation plans** — DETAILED and SUMMARY plans, one of each per CQ domain, ready for review before execution (`/cq-plan`).

All output lands in a `CQ-Reviews/` folder at the repository root, organised by concern. Nothing in the source tree is modified.

```
CQ-Reviews/
  solutions/<Solution>/    Purpose.md, Architect.md          (one set per .sln)
  projects/<Project>/      Data.md, CodeReview.md,           (one set per .csproj)
                           TestReview.md
  summaries/               CQ-Summary.md + 4 per-lens summaries
  plans/                   8 implementation plans (DETAILED + SUMMARY)
  scripts/                 build_html.py, build_docx.py
  site/                    generated browsable HTML
```

## Layout

```
.claude-plugin/
  plugin.json           Plugin manifest
agents/                 Sub-agent definitions (CQ-Architect, CQ-Reviewer, ...)
commands/               Slash commands (/cq-scan, /cq-plan)
```

## Installation

### As a Claude Code plugin

From a Claude Code session in any project:

```
/plugin install <path-or-git-url-to-this-repo>
```

Once installed, the `cq-*` agents become available to dispatch and `/cq-scan` / `/cq-plan` show up as slash commands.

### Manual copy (fallback)

Copy `agents/cq-*.md` into your project's `.claude/agents/` and `commands/cq-*.md` into your project's `.claude/commands/`.

## Typical workflow

```
/cq-plan                # discovers solutions, ensures purpose files exist,
                        # runs the full scan, produces eight implementation plans
```

Or run the pieces individually:

```
/cq-scan [subdir ...]   # per-project reviews only
# (then dispatch CQ-Summary, CQ-Management-Summary, CQ-HTML-Publisher as needed)
```

## Conventions

- Reports land in `CQ-Reviews/` relative to the working directory, split by concern (`solutions/`, `projects/`, `summaries/`, `plans/`, `scripts/`, `site/`).
- `solutions/<Solution>/Purpose.md` files are preserved across re-scans — they rarely go stale and are expensive to regenerate.
- All agents are read-only across the production source tree. Output is markdown.
- No migration of an existing flat `CQ-Reviews/` is performed — the next `/cq-scan` simply produces the new layout.

## License

TBD.
