<!--
This macro block is meant to be pasted, verbatim or near-verbatim, at the top of
every tool's CLAUDE.md (BuildTrackUnified, PunchTrack, Schedule Trend). Keep it in
sync by hand when a cross-tool rule changes — there's no auto-sync between repos.
-->

## Who this is for

Chris — Superintendent at Prieb Homes, a high-volume residential new home builder,
Olathe/KC metro area. Field-based, managing multiple houses across subdivisions at
once. Builds internal tools himself because CoConstruct (the company's main system)
has no cross-house/cross-timeline intelligence — only per-house, per-assignee, or
per-vendor views. Non-technical background, learning API/JSON/dev concepts as he
builds — explain *why*, not just *what*.

## The tool ecosystem — context, not code you can see

- **BuildTrackUnified** — house milestone/bonus/closing tracker, with Scope Deviation
  (contract change items → trade emails → follow-up log) embedded inside it as its
  own section/tab. Google Sheets + Apps Script backend.
- **PunchTrack** — voice-dictated walkthrough punch items. Own repo, own backend.
- **Schedule Trend** — ingests weekly Thursday-meeting schedule PDFs (~100 houses,
  back to 2020) to spot slippage/bottleneck trends, with a vendor trust/reliability
  scoring layer. Own repo, own backend. A duration-baseline/drift layer is planned
  to be built *inside* this tool, not as a separate one.

**This file only describes the contract those other tools are expected to honor —
not their live code.** If you need to know what another tool's backend actually does
right now, that's out of scope for this repo/session; say so rather than guessing,
and trust what you find in *this* repo's own files over anything below if they ever
conflict.

## Architecture principles — do not relitigate these

- Tools stay **separate and independently backended**, linked only by shared address
  as the common key. Deliberately not one merged database — keeps each tool small and
  fast as data accumulates, and keeps a bug in one tool from taking down another.
- Creating a house in one tool does **not** auto-create linked jobs in others — those
  are opt-in, created only once a house actually reaches that real-world stage.
- Eventual direction (not yet built, reference only): one unified dashboard frontend
  that calls each tool's backend directly — still no merged database. Per-house radial
  progress rings per tool, rendered only once that house has a job in that tool. See
  each repo's own DESIGN.md if one exists for visual direction.
- **Token-efficient pattern:** send deltas + a rolling carried-forward summary to
  Claude, not full history. Code pre-filters (by neighborhood/date/etc.) before
  anything reaches the model; the model handles query-translation and narration,
  never raw dumps. Aggregate math happens in code, not token-by-token in the model.
- API cost budget: modest (~$20/mo as of mid-2026), open to more but flag anything
  that would meaningfully increase per-query spend.

## Security conventions — decided, don't re-open without asking

- Every backend requires an `APP_TOKEN` (Apps Script Script Property) on every
  request, checked via a `checkToken()` guard before any read/write/AI action runs.
  Never add a debug/status endpoint that returns the token or bypasses this check —
  that has bitten this project before.
- Every backend rate-limits via `CacheService` buckets (~60/min reads, ~20/min
  writes, ~10/min AI calls) plus a per-day AI cap tracked in Script Properties.
  This is the primary real protection, not the token — see below.
- **Known, accepted tradeoff:** `SCRIPT_URL` and `APP_TOKEN` are hardcoded in each
  tool's client-side `index.html`, which is served directly by public GitHub Pages
  repos. The token is *not* actually secret — anyone who finds the page can view-source
  it. This is a deliberate choice (rate limiting is the real backstop, not the token)
  rather than an oversight — don't "fix" it by architecting around it without asking.
- If `APP_TOKEN` is ever found to have leaked (e.g. via a debug endpoint, a bad log
  line, or being committed to a *newly-made-private* assumption that turns out
  false), rotate it in Script Properties immediately and say so.

## How Chris likes to work

- **No guessing on assumptions.** If a rubric, timeline, or business rule isn't
  confirmed, ask — don't infer and move on. Wrong assumptions are the worst-case
  failure mode, worse than the extra time spent asking.
- Prefers being interviewed thoroughly on domain rules before code gets written,
  even if that's slower up front.
- Wants to understand the underlying reasoning (cost structure, why a filter step
  exists, etc.), not just receive a working feature.

## Domain reference (construction workflow — for accuracy across all tools)

- **Stage sequence:** foundation (service pulled → hole dug → formed/poured →
  backfill) → framing → flat work → roof → rough-in (framing + MEP: E-Mech/electrical,
  P-Mech/plumbing, M-Mech/HVAC, with a "Furdown" carpentry step between P-Mech and
  E-Mech) → RI Inspect → ReRI Inspect (more progressed than RI Inspect) → sheetrock →
  trim → paint → finish trades (tile, countertops, fireplace, mirrors, hardware) →
  closing.
- **Inspection gates:** structural/foundation report → underslab plumbing inspection
  → garage portal → rough-in (incl. gas pressure test) → home efficiency rater visit
  → pre-placement concrete → combined final inspection (life-safety + exterior +
  permit-hold) → certificate of occupancy (required for lender funding/closing).
  Passing gas/electrical inspection is a prerequisite for utility meter installs.
- **Superintendent-to-neighborhood map:** Chris → Woodland Hills + Ranch Villas of
  Prairie Farms; Jason → Prairie Farms (distinct despite similar name); Jack →
  Canyon Lakes; Ashton → multifamily (only sometimes on the shared sheet).
- **Culture norm:** a job "sitting" with no schedule movement must always have an
  explainable reason.
- **Vendor trust dynamic:** some trades pad/misstate timelines (counter-adjust
  downward), some are uninvolved but want to seem informed, some are reliably
  honest — tracked per-vendor, informs vendor-facing scoring/output.

## Trade email conventions (use exactly, don't improvise format)

- Subject line = recipient's name only.
- Body opens: "Good morning. Can you please have the below listed items completed
  at the above address prior to [date]"
- Bullets as `Room: Item` (colon separator, sub-location in parens allowed, related
  fixes combined with semicolons).
- Cleaners typically scheduled the day after the deadline (day before closing).

---

## This repo: Portfolio Dashboard (new — not yet built)

A read-only aggregator that pulls status from the other four tools and shows it
in one place. **This tool has no backend and no Google Sheet of its own** — it
holds credentials for the other four tools' existing backends and calls their
read actions directly, client-side. Single-file `index.html` (CDN + Babel, no
build step), same pattern as every other tool in this ecosystem.

### Design reference
`MOCKUP-design-reference.html` in this repo (or wherever it's placed) is a static mockup
with fake sample data — it shows the intended layout, not working code. Match its
visual direction (dark theme, Barlow Condensed headers, DM Mono data, the round
per-house status dial, worst-first sort + capped section length, the Closing This
Week section) but rebuild the data layer for real against the actual backends
below. Do not reuse its JavaScript data/rendering logic as-is — it was written
against invented sample data, not the real response shapes.

### Source tools this dashboard reads from

**BuildTrack (milestone/bonus/closing)** — repo: `buildtrack-unified`, but this is
a *separate* backend from the one below in that same repo.
- `GET ?token=...` → `{ houses: [...] }`. Each house has `milestones` (keyed
  dates — includes `closing`) and `bonuses` (roughIn/co/closing/basement booleans).
- Needs: `SCRIPT_URL` + `APP_TOKEN` for this specific backend (`Code-BuildTrack.gs`
  in the buildtrack-unified repo) — **ask Chris for these, do not reuse the Scope
  Deviation token from the same repo, they're intentionally separate.**

**Scope Deviation** — repo: `buildtrack-unified` (embedded section, `Code.gs`).
- `GET ?token=...&action=listJobs` → job list. `GET ?token=...&action=getJob&slug=...`
  → one job's Items (contract change items) with `status` per item.
- Needs: `SCRIPT_URL` (same as BuildTrack milestone's host page, but a *different*
  `APPS_SCRIPT_URL`) + its own `APP_TOKEN` — **ask Chris.**

**PunchTrack** — own repo.
- `GET ?token=...&action=listJobs` → job list. `GET ?token=...&action=getJob&slug=...`
  → one job's Items (walkthrough punch items), `status` ∈ assignable | flagged |
  self_assigned | sent.
- Needs: `SCRIPT_URL` + `APP_TOKEN` — **ask Chris.**

**Schedule Trend** — own repo.
- `GET ?token=...&action=listWeeks` → available weeks. `GET ?token=...&action=getWeek&weekDate=...`
  → per-house stage/status for that week. Stage vocabulary is `STAGE_ORDER` in that
  repo's own `index.html` — reuse it rather than re-inventing a stage list here.
- Needs: `SCRIPT_URL` + `APP_TOKEN` — **ask Chris.**

### Before writing any fetch calls
This dashboard needs **eight secrets total** (SCRIPT_URL + APP_TOKEN × 4 backends)
that don't exist anywhere in this new repo yet. Ask Chris for all eight up front
rather than discovering the gap mid-build — they're not guessable and not derivable
from anything already in this repo.

### Known open design questions (ask Chris, don't assume)
- What "Open <tool>" does on click — link to that tool's existing live page
  (simplest), expand inline, or a modal. Not decided yet.
- Whether Closing This Week should also show closing-*eligibility* (inspection
  gates cleared) alongside bonus-eligibility.
- The 4 KPI tiles in the mockup (Total Houses, Need Attention, Fully Closed,
  Active Tool Jobs) are a first guess, not confirmed as what Chris wants to see
  first every morning — confirm before treating them as final.
- Refresh model: manual refresh button, not auto-polling — rate limits on the
  source backends are global per-backend, not per-caller, so a polling dashboard
  would compete with real usage of the underlying tools for that budget.
- Address matching across tools: simple normalize (case/punctuation/abbreviation),
  not a flagged-mismatch system — this was a deliberate choice, not an oversight,
  don't upgrade it without asking.
