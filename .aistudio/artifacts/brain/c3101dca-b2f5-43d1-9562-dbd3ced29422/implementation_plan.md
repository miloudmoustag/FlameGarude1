# FlameGuard: Scientific Honesty & Human-Centered Redesign Plan

FlameGuard transforms NASA microgravity combustion experiments into an honest, navigable intelligence workspace built around real Physical Sciences Informatics (PSI) records and primary ISS flight media. This plan executes **Job A (Scientific Honesty)** across the backend, ML engine, and documentation first, followed by **Job B (Creative Multi-Scene Redesign)** across the frontend experience.

---

### User Review & Critical Decisions

> [!IMPORTANT]
> All existing Express API endpoints and database tables remain intact and backward-compatible. New metadata fields and the dataset registry table extend existing responses without breaking callers.

- **Verified AI Model Configuration**: Checked against the `@google/genai` SDK documentation — `gemini-4-hydrogen` is the verified active model identifier for text synthesis. When `GEMINI_API_KEY` is absent or unavailable, the assistant transparently operates in **Offline Citation Mode** using local BM25 retrieval over verified NASA excerpts.
- **Strict Out-of-Range Refusal in Risk Lab**: Rather than extrapolating probabilities outside the observed NASA test envelope (`oxygen_pct`, `flow_velocity_cm_s`, `pressure_kpa`), predictions will withhold probability and display a hatched *"Outside the range of NASA data we have"* gauge listing the out-of-bounds parameter(s) and exact training bounds.
- **Dataset Registry vs. Imported Experiments**: Newly cataloged NASA missions from the compact seed (`SAME`, `FLEX-2`, `CFI`, `ACME-*`, `DAFT/DAFT-2`, `SLICE`, `SIBAL`) will be stored in a dedicated SQLite `datasets_registry` table and surfaced in the Investigation Library with a `NEW` indicator and `data not yet imported` status until full PSI CSVs are loaded.

---

### 1. Overview & Core Concept

- **What It Does**: Combines a SQLite-backed NASA combustion repository, a strictly bounded logistic-regression + random-forest demonstration classifier, a synchronized ISS flight telemetry walkthrough, and a citation-grounded Q&A notebook.
- **Target Audience**: Space Apps judges, aerospace students, researchers, and fire-safety engineers who need quick intuitive understanding of why flames behave differently in orbit alongside strict transparency about data limits.
- **Key Value**: Replaces dense dashboard walls with an immediate physical intuition (*1g teardrop vs. 0g sphere*) and prevents misleading ML overconfidence by pairing every metric with its exact sample size (`n`), cross-validation method, uncertainty band, and nearest historical experiments.

---

### 2. User Experience & Visual Design

- **Key User Flows**:
  1. **Landing Scene**: Opens directly on a full-viewport live flame simulation (SVG + `motion`) with a `1g Earth` vs. `0g Microgravity` gravity toggle, the core message (*"On Earth fire rises. In orbit it doesn't. Here is why that matters for astronauts."*), and a single **"Start the investigation"** action. (`prefers-reduced-motion` pauses animated flicker while keeping gravity state comparison clear.)
  2. **Scene Navigation Shell**: Left navigation rail on desktop and thumb-friendly bottom dock on mobile (`>=380px`) switching smoothly via `AnimatePresence` across **Story**, **Explore**, **Compare**, **Risk Lab**, **Ask**, **Library**, **Report**, and **Limits**. Heavy visualization scenes lazy-load with skeleton states.
  3. **Story (BASS Sample 147 Test B1)**: 5-step guided walkthrough synced with the uploaded ISS flight video and progressive fan/radiometer telemetry chart (`10:10:06` `"Fan to zero"` at `t=30s`, flame shrinks, extinction at `10:10:13` / `t=37s`), annotated with lab-notebook margin notes.
  4. **Explore (Flammability Map)**: Interactive scatter plot (`oxygen_pct` on X, `flow_velocity_cm_s` on Y) with diagonal hatch patterns marking sparse/unexplored regions, glossary tooltips (`quenching`, `concurrent flow`, `normoxic`), a *"Why does this matter?"* takeaway line, an accessible text summary, and a click-to-open side drawer showing full record provenance and explicit `"not in source"` markers for `NULL` fields.
  5. **Compare**: Side-by-side experiment cards paired with a draggable morph slider that transitions between two selected NASA runs, showing flame regime, measured parameters, and plain-language captions.
  6. **Risk Lab (Research Demonstration Prototype)**: Persistent honesty banner at top, binary-only scope notice (*sustained vs. extinction only — not fire size, spread rate, or severity*), spacecraft-instrument sliders with **Exploration Cabin** and **Fan Shutdown** presets, metric cards showing sample count (`n = ... , leave-one-group-out CV`) beside ROC-AUC / F1 / Brier, uncertainty interval, 3 nearest NASA experiments with Euclidean distance, and a hatched *"no data"* gauge state whenever inputs exceed training min/max bounds.
  7. **Ask (Research Notebook)**: Clean lab-notebook chat layout with suggested prompt chips, clickable citation chips linking directly to NASA sources/DOIs, and a clear banner whenever running in offline citation mode.
  8. **Library, Report & Limits**: Library lists all cataloged PSI missions (`datasets_registry` `NEW` items + imported investigations, ProofSheet PDFs, telemetry video, and CSV import drawer); Report exports PDF & CSV including mandatory **Limitations** and exact sample counts; **Limits & Honesty** dedicates a focused scene to small sample size, missing ProofSheet O2/flow telemetry, non-peer-reviewed status, and non-operational boundaries.
- **Visual Identity & Theme**:
  - *Aesthetic Direction*: *"Mission control meets a lab notebook"* — dark telemetry clarity warmed by amber combustion highlights and understated handwritten-style chart margin annotations.
  - *Color Tokens*: Deep space base (`#070B14`), elevated slate panels (`#0D1424`), amber/orange flame accents (`#F59E0B`, `#F97316`), cool-flame cyan (`#38BDF8`), WCAG AA slate text hierarchy (`#F8FAFC`, `#CBD5E1`, `#94A3B8`).
  - *Typography*: `Chakra Petch` for scene headings, `Plus Jakarta Sans` for approachable explanations, `JetBrains Mono` (`tabular-nums`) for telemetry/bounds/metrics, plus italic serif/handwritten accent notes for lab-margin callouts.

---

### 3. Key Product Decisions & Trade-Offs

- **Decision 1 — Honesty Guardrails Inside ML Engine & API**:
  - *Chosen Approach*: Compute exact `[min, max]` bounds for `oxygen_pct`, `flow_velocity_cm_s`, and `pressure_kpa` directly from the SQLite-backed training matrix, return them in the evaluation and prediction responses, return `null` probability when out of range, and rename unearned `"validated"` labels across code and UI to accurate terms (*"complete-feature training records"* / *"NASA benchmark runs"*).
  - *Why*: Computing bounds dynamically guarantees the guardrails automatically update whenever users import additional PSI CSV datasets, and refusing out-of-range predictions prevents silent linear/tree extrapolation.
  - *Alternatives Considered*: Clamping inputs silently in the UI — rejected because silent clamping hides real data gaps from users.
- **Decision 2 — Dedicated `datasets_registry` Table Loaded from Compact Seed**:
  - *Chosen Approach*: Add a `datasets_registry` table populated idempotently from the provided compact seed JSON, tracking `status` (`existing` vs. `NEW`) and imported experiment counts separately from the `experiments` table.
  - *Why*: Lets the Investigation Library transparently display upcoming NASA datasets (`SAME`, `FLEX-2`, `CFI`, `ACME-*`, `DAFT/DAFT-2`, `SLICE`, `SIBAL`) with `"data not yet imported"` status without fabricating fake rows inside `experiments`.
- **Decision 3 — Lazy-Loaded Single-Screen Scenes over Vertical Dashboard Stack**:
  - *Chosen Approach*: Split heavy Recharts and interactive scenes via `React.lazy` + `Suspense` inside a responsive fixed shell (`AnimatePresence` cross-fade) rather than rendering all analytics on one scrolling page.
  - *Why*: Keeps initial load fast on the animated Hero scene, gives every research task dedicated focus, and keeps mobile navigation uncluttered down to `380px`.

---

### 4. Technical Architecture & Data Strategy

- **Architecture & Component Diagram**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    React 19 + Motion App Shell                          │
│  [Landing Hero: 1g Teardrop ↔ 0g Sphere] ──► "Start the investigation"  │
│                                                                         │
│  Rail (Desktop) / Bottom Dock (380px+):                                 │
│  ┌───────┬─────────┬─────────┬──────────┬─────┬─────────┬────────┬────┐ │
│  │ Story │ Explore │ Compare │ Risk Lab │ Ask │ Library │ Report │Lim.│ │
│  └───┬───┴────┬────┴────┬────┴────┬─────┴──┬──┴────┬────┴───┬────┴─┬──┘ │
└──────┼────────┼─────────┼─────────┼────────┼───────┼────────┼──────┼────┘
       │        │         │         │        │       │        │      │
       ▼        ▼         ▼         ▼        ▼       ▼        ▼      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│             Express API (/api/* — Backward-Compatible)                  │
│  • /api/overview & /api/investigations (includes datasets_registry)     │
│  • /api/experiments, /api/telemetry, /api/import                        │
│  • /api/ml/evaluation & /api/ml/predict (bounds check + 3 nearest runs) │
│  • /api/chat (Gemini SDK `gemini-4-hydrogen` or Offline Citation Mode)  │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      SQLite Persistent Store                            │
│  investigations | experiments | measurements | findings | documents     │
│  telemetry_series | predictions | datasets_registry (from compact seed) │
└─────────────────────────────────────────────────────────────────────────┘
```

- **Data Model & Integrity Rules**:
  - Missing measurements remain SQL `NULL` and render explicitly as `"not in source"` across tables, drawers, and exports.
  - `datasets_registry` links each cataloged NASA investigation (`id`, `name`, `platform`, `topic`, `source`, `status`) to imported `experiments` counts so empty missions display `"data not yet imported"`.
  - Every finding, chart data point drawer, and assistant response displays its clickable NASA PSI / NTRS / DOI citation.
- **Interactive State & Accessibility Mapping**:
  - Each scene handles empty states with explanatory sentences, keyboard focus states, WCAG AA contrast, plain-language *"Why does this matter?"* captions under charts, accessible summary blocks, and reduced-motion fallbacks.
  - Build and type checks (`npm run lint` / `tsc --noEmit` + Vite production compile) will be verified incrementally as Job A and Job B are implemented, finishing with a changed-file inventory and a 5-line judge demo script.
