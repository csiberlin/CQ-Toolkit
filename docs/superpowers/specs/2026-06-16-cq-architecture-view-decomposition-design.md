# CQ Architecture-View Decomposition — Design

**Date:** 2026-06-16
**Status:** Approved (design); implementation plan pending
**Topic:** Split the single per-solution architectural review into a Solution-Layout view, a Backend-Architecture view, and a WPF Frontend-Architecture view.

## Problem

`CQ-Architect` reviews each solution **as a whole** and applies a backend-centric checklist (Minimal API, AuthN/AuthZ, EF, 50x scalability) to everything in it. Real solutions mix tiers — backend (ASP.NET), frontend (WPF / WinForms / Razor / Blazor), shared libraries, and unit tests. Consequences:

- A WPF/Razor frontend is judged against backend criteria, or simply ignored — there is no frontend architectural view at all.
- A pure-desktop solution still gets a backend-shaped report with a meaningless 50x verdict.
- The tech-neutral "solution layout" concern (project boundaries, dependency direction across tiers) is implicit and easily lost.

Tests are already out of scope for `CQ-Architect` (owned by `CQ-Test-Reviewer`), so they are not part of this redesign except as items the layout view *notes* structurally.

## Decisions

- **Approach B (hybrid).** `CQ-Architect` owns **Solution Layout + Backend**; a new agent **`CQ-Frontend-Architect`** owns the WPF view. Layout stays bundled with backend because it is short, tech-neutral, and shares the cross-tier dependency-direction reasoning. One new agent, one new summary domain, one new plan kind.
- **WPF is the only first-class frontend.** WinForms / Razor / Blazor are **detected and labeled only** — `CQ-Frontend-Architect` emits a one-paragraph "no specialized review defined for this project type" report and invents no findings. This mirrors the tech-conditional pattern already used by `CQ-Data`.
- **Frontend theme prefix: `FR`.**
- **Frontend responsiveness/perf routes to the Reliability bucket** in the management summary (the six buckets have no "responsiveness" axis, and "Scalability" means portfolio load, not desktop UI latency).
- The new agent is **born with the cross-lens coordination discipline** already added to the other reviewers (Cross-Lens Flags, Considered-but-not-reported, owner-of-last-resort routing, load-independent severity).

## Section 1 — Project classification & solution archetype (`cq-scan`, `cq-plan`)

`cq-scan` Step 2 today decides only **test vs production**. Extend it: every **production** project also gets a **tier**.

| Tier | Detection signal |
|---|---|
| Backend | `Microsoft.NET.Sdk.Web`, `Microsoft.AspNetCore.*`, worker/host SDK, or the web/service entrypoint |
| Frontend (WPF) | `<UseWPF>true</UseWPF>`, references `PresentationFramework` / `DevExpress.Xpf.*`, or contains `.xaml` files |
| Frontend (other) | `<UseWindowsForms>true</UseWindowsForms>` (WinForms), Blazor, Razor/MVC — **labeled only** |
| Library | production project matching none of the above (class library, domain, shared kernel) |

Derive a **solution archetype** from the tier mix: `backend-only` | `desktop-only` | `mixed` | `library-only`. The archetype gates which deep-dive sections run.

Dispatch rules:

- `CQ-Architect` — one per solution (dispatch unchanged), briefed with the archetype + tier map.
- `CQ-Frontend-Architect` — **new**, dispatched per solution **only when a WPF frontend project exists**. If only "other" frontend tech exists, it is skipped and the layout view notes it. Brief: WPF project path(s), `<Solution-Name>`, working directory.
- `CQ-Reviewer` / `CQ-Data` / `CQ-Test-Reviewer` — unchanged, still per project.

`cq-plan` Step 2 discovery applies the same classification so the Frontend dispatch and the Purpose lookup line up.

## Section 2 — `CQ-Architect` rescoped to Layout + Backend

`CQ-Architect` changes from "review the whole solution" to **solution layout + backend architecture**.

- New mandatory **`## Solution Layout`** section (renders for every archetype):
  - Project inventory with tier labels and the solution-archetype verdict.
  - Cross-tier dependency-direction check (dependencies point inward: UI → application → domain ← infrastructure). Flag a WPF project referencing EF / `HttpClient` directly, a library depending on a UI assembly, or the backend referencing the desktop project.
  - Shared-kernel / "big ball of mud" assessment at the solution level.
  - Test-project presence (structure only; content is `CQ-Test-Reviewer`'s).
- Existing backend dimensions (data flow, validation, AuthN/AuthZ, domain, error-handling strategy, observability, evolvability, **50x scalability**) move under **`## Backend Architecture`**, which renders **only when backend projects exist**. For `desktop-only` / `library-only`, replace it with one line: *"No backend projects — backend architecture and 50x stress test not applicable."* The 50x stress test is explicitly backend-only.
- **Frontend internals leave `CQ-Architect`.** It no longer judges WPF/frontend concerns beyond noting them in the layout view and flagging cross-tier dependency violations. A new non-overlap clause points WPF architecture at `CQ-Frontend-Architect`.
- Findings gain a **`Tier: Layout | Backend`** metadata field.
- The Coverage map splits into a Layout block (always) and a Backend block (gated on archetype, marked `not applicable` when no backend).

## Section 3 — New agent `CQ-Frontend-Architect` (WPF-first)

New file `agents/cq-frontend-architect.md`. Same scaffolding as the other reviewers: mandatory-written-deliverable contract, relative-path conventions, Step 0 (Purpose) + Step 0b (project conventions / skills) loading, MCP-or-grep tool-preference table, value bar, Coverage map, self-review + Verification log, output-discipline/citation rules, **and the cross-lens coordination block** (Cross-Lens Flags, Considered-but-not-reported, owner-of-last-resort routing to `CQ-Architect`, load-independent severity).

Per solution, writes `solutions/<Solution>/Frontend.md`.

**Tech-conditional behavior:**
- WPF present → full WPF review (below).
- Only other UI tech → one-paragraph report: *"Detected `<tech>`; no specialized architectural review defined for this project type."* No invented findings.
- No frontend → agent not dispatched.

**WPF review dimensions:**
1. **MVVM separation** — View/ViewModel/Model boundaries; minimal code-behind (view concerns only); no business logic in code-behind or XAML; ViewModels reach infrastructure (EF/`HttpClient`) only through services.
2. **Data binding strategy** — `INotifyPropertyChanged` correctness; `DataContext` flow; binding-error hygiene; ViewModel-shaped properties vs heavy converters; DevExpress `DXBinding` where the project skill mandates it.
3. **Commands** — `ICommand` / `DelegateCommand` / DevExpress POCO commands vs code-behind event handlers; honoring the project's `BaseCommand` convention.
4. **Threading & dispatcher** — UI-thread blocking (sync I/O, `.Result` on the UI thread), `Dispatcher.Invoke` misuse, async patterns that keep the UI responsive, background-work model.
5. **View lifecycle, navigation & DI** — window/page/UserControl structure, consistent navigation pattern, ViewModels resolved via DI not `new`.
6. **Resources, styles & theming** — `ResourceDictionary` organization, style/theme consistency, DevExpress theme usage.
7. **Frontend performance (replaces 50x for desktop)** — UI virtualization on large grids/lists, binding / visual-tree perf, image/resource loading. *Responsiveness*, not throughput.
8. **Lifetime & leaks** — event-handler / static-event leaks, weak-event usage, `IDisposable` / unsubscribe discipline.
9. **State management** — app-level state, messenger / event-aggregator vs tight coupling.

**Non-overlap contract:**
- `CQ-Frontend-Architect` owns WPF **architecture & patterns** (MVVM as a strategy, binding/command/threading/navigation models, leak patterns).
- `CQ-Reviewer` keeps per-file **code quality** in the same projects (naming, method size, logging, a single mis-named ViewModel property).
- `CQ-Architect` owns the cross-tier dependency-direction call in the layout view.
- `CQ-Data` still owns data access even when reached from a ViewModel.
- Overlaps resolve through `## Cross-Lens Flags`. WPF/DevExpress skills (`wpf-command-dialog`, `devexpress-*`, `DXBinding`) load in Step 0b and feed `## Project-Convention Deviations`.

## Section 4 — Downstream ripple (summaries + plans)

- **`cq-domain-summary`** — domains become `{Architecture, Frontend, CodeReview, TestReview}`. New row: Domain `Frontend` → glob `CQ-Reviews\solutions\*\Frontend.md` → output `CQ-Frontend-Summary.md`, unit = solution, theme prefix `FR`. (Architecture domain still reads `Architect.md`, now = layout + backend.) Add `FR` to the legend block.
- **`cq-summary`** — Phase A dispatches **four** sub-agents instead of three; the reference-nomenclature legend gains `FR<n>`; cross-domain `X<n>` themes may now span Frontend. The Step 4b cross-lens reconciliation set automatically includes `Frontend.md` once the per-unit glob includes it.
- **`cq-plan`** — produces **ten** plans (adds DETAILED + SUMMARY Frontend), naming `CQ-Summary-Frontend.md` per the existing override convention; discovery classifies tiers as in Section 1.
- **`cq-management-summary`** — gains `CQ-Frontend-Summary` as a fifth input plus a **Frontend-Architect → bucket** mapping. WPF findings route into existing production buckets (no new bucket): MVVM / coupling / navigation → **Maintainability**; XAML / naming / resource organization → **Readability**; UI-thread blocking / dispatcher misuse / leaks → **Reliability**; binding / virtualization perf → **Reliability** (responsiveness, noting it is not portfolio "Scalability").

## Section 5 — Citation nomenclature & cross-lens wiring

- Per-unit short-name for the new report: `solutions\<Sln>\Frontend.md` → **`<Sln>-Frontend`** (folder + lens basename, no `CQ-` infix, no `.md`). Self-citations: `` `<Sln>-Frontend §Findings #N` ``.
- Summary short name: `CQ-Frontend-Summary.md` → `Frontend-Summary`; themes `FR<n>`.
- `CQ-Frontend-Architect` carries the same Cross-Lens Flags / Considered-but-not-reported / owner-of-last-resort / load-independent-severity rules already in the other reviewers, so it is consistent from day one.

## Affected files

- `commands/cq-scan.md` — tier classification + archetype + Frontend dispatch.
- `commands/cq-plan.md` — tier-aware discovery; ten plans; `CQ-Summary-Frontend.md`.
- `agents/cq-architect.md` — `## Solution Layout` section, archetype-gated `## Backend Architecture`, `Tier` field, frontend-out-of-scope clause, Coverage-map split.
- `agents/cq-frontend-architect.md` — **new** WPF-first architecture agent.
- `agents/cq-domain-summary.md` — Frontend domain (`FR`), glob/output table, legend.
- `agents/cq-summary.md` — four domain sub-agents, `FR` legend, nomenclature.
- `agents/cq-management-summary.md` — fifth input + Frontend → bucket mapping.
- `README.md` — document the new lens and archetype gating (if it enumerates lenses).

## Non-goals / out of scope

- No specialized review for WinForms / Razor / Blazor (detected + labeled only).
- No standalone Solution-Layout agent (that was Approach C); layout stays inside `CQ-Architect`.
- No change to `CQ-Reviewer` / `CQ-Data` / `CQ-Test-Reviewer` scope beyond the new non-overlap clause naming `CQ-Frontend-Architect`.
- No build/render-script changes assumed (the publishing layer is not shipped in this repo).

## Acceptance criteria

1. A `desktop-only` WPF solution produces: an `Architect.md` with a populated `## Solution Layout` and a "backend not applicable" line (no 50x verdict), plus a `Frontend.md` with WPF findings.
2. A `backend-only` solution produces an `Architect.md` with layout + backend sections and **no** `Frontend.md` (agent not dispatched).
3. A `mixed` solution produces both, with cross-tier dependency violations flagged in the layout view.
4. A solution whose only UI is WinForms produces a one-paragraph `Frontend.md` stating no specialized review applies.
5. `cq-summary` emits a `CQ-Frontend-Summary.md` with `FR<n>` themes; `cq-management-summary` consumes it and routes WPF findings into Maintainability/Readability/Reliability.
6. Citations to the new report resolve as `<Sln>-Frontend §Findings #N`.

## Open questions — resolved

- Frontend theme prefix → `FR`.
- Frontend responsiveness bucket → Reliability.
