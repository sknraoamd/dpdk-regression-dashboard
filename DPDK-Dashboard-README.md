# DPDK Regression Dashboard

A self-contained, single-file HTML dashboard that connects directly to your **GitHub Projects v2** board and provides regression test tracking, sprint burndown, effort tracking, data quality checking, timeline management, and a regression health score — with zero backend, zero install, and zero data sent to any third party.

---

## Quick Start

1. Download `dashboard-dpdk-regression-tests.html`
2. Open it in any modern browser (Chrome, Edge, Firefox)
3. Create a fine-grained PAT (see below)
4. Paste the token, click **⚡ Load Dashboard**

No server, no npm, no config files.

---

## Creating a GitHub Token

The dashboard uses GitHub's GraphQL API and needs a **fine-grained Personal Access Token (PAT)** — classic PATs are blocked by your org.

**Steps:**

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. Click **Generate new token**
3. Set **Resource owner** to `AMD-DEAE-CEME`
4. Under **Organization permissions** → **Projects** → set to **Read-only**
5. Under **Repository permissions** → **Issues** → set to **Read-only** *(required for first-comment inference)*
6. Click **Generate token** and copy it immediately

> **Why Issues: Read-only?**
> The `Projects: Read-only` permission covers all project fields and item metadata, but issue comments are stored on the repository, not the project. Without `Issues: Read-only`, `comments(first:1)` returns an empty array and first-comment inference cannot run. If comments show as `⚠ No comment fetched` in the Data Quality tab, this is the cause.

**Token security:**
- Stored only in `sessionStorage` — disappears when you close the browser tab
- Never sent anywhere except directly to `https://api.github.com/graphql`
- No tracking, analytics, or third-party calls except Chart.js (CDN, gracefully degrades if blocked)

---

## What Gets Fetched

All items from **Project #19** in `AMD-DEAE-CEME` are fetched using paginated GraphQL requests (100 items per page). For each item:

| Data | GraphQL source | Used for |
|---|---|---|
| Title, URL, state | `content.title / url / state` | Display, title inference |
| Issue number | `content.number` | Parent lookup fallback |
| Parent issue | `content.parent.url` | Building the inheritance tree |
| First comment | `content.comments(first:1)` | Field inference fallback |
| All project fields | `fieldValues` (single-select, text, number, date, iteration) | Filters, KPIs, charts, matrix |

---

## Field Mapping

Configured in the `FIELDS` constant at the top of the script:

```js
const FIELDS = {
  dpdk:      "DPDK Version",    // SINGLE_SELECT: 25.11, 25.07, 26.03
  platform:  "Platform",        // SINGLE_SELECT: Sorano, Turin Classic, etc.
  benchmark: "Benchmark type",  // SINGLE_SELECT: NIC Based, Memory, NIC and Memory
  ports:     "Ports",           // SINGLE_SELECT: PF, VF
  status:    "Status",          // SINGLE_SELECT: Todo, In Progress, Done, ...
  sprint:    "Sprint Name",     // SINGLE_SELECT: Sprint-January2025, ...
  priority:  "Priority",        // SINGLE_SELECT: P0, P1, P2, P3
  issuetype: "Issue Type",      // SINGLE_SELECT: EPIC, Main-Task, Sub-Task, ...
  category:  "Category",        // SINGLE_SELECT: Regression testing, ...
  phase:     "Phase Type",      // SINGLE_SELECT: Phase 1, Phase 2, Phase 3, Patches
};
```

Date fields used by the Timeline tab:

```js
const DATE_FIELDS = {
  plannedStart: "Planned Start date",
  plannedEnd:   "Planned End date",
  actualStart:  "Actual Start Date",
  actualEnd:    "Actual End Date",
};
```

Effort fields used by the Sprint & Effort tab:

```js
const EFFORT_FIELDS = {
  planned:   "Planned Efforts",
  actual:    "Actual Efforts",
  remaining: "Remaining Efforts",
};
```

If any field name changes in GitHub, update the string here. Use `debug-github-fields.html` to inspect current field names and types at any time.

---

## Status Buckets

GitHub status values are normalised into four display buckets:

| Bucket | Colour | GitHub statuses matched |
|---|---|---|
| ✓ Completed | Green | Done, Completed, Results Analysis in progress |
| ⧖ Active | Amber | In Progress, In Review, Runs |
| ✗ Blocked | Red | Blocker, Re-Run, Dependency |
| ○ Backlog | Grey | Todo, Backlog, On Hold, Sprint |

To change mappings, edit `STATUS_MAP` near the top of the script.

---

## Issue Type Hierarchy

The dashboard understands your project hierarchy for field inheritance:

```
EPIC / Milestone
  └── Main-Task / Feature
        └── Sub-Task / Validation / Defect / Improvement / User Story
```

Links are established via GitHub's native **sub-issue parent** relationship (`content.parent.url`). If a parent issue is in the same project it is linked automatically. When not, a title-similarity fallback attempts to link items by matching benchmark and platform keywords.

---

## Field Inference & Inheritance

For regression testing tickets, three fields are required: **DPDK Version**, **Platform**, and **Benchmark Type**. When a field is blank, the dashboard resolves it automatically through five passes in priority order:

### Pass 1 — GitHub field value
The field is set directly on the project item. Always the authoritative source.

### Pass 2 — ↑ Aggregated from children (UP)
If a parent is missing a field but its descendants have it, majority vote across the full subtree determines the value.

### Pass 2b — ↔ Shared from siblings
Items with the same parent check siblings for missing field values (majority vote). Handles groups of Sub-Tasks where only one has a field set.

### Pass 3 — ↓ Inherited from ancestor (DOWN)
Each item walks its full ancestor chain (parent, grandparent, great-grandparent…). At each ancestor it tries:
1. Stored field value
2. Title inference (scans ancestor title for known patterns)
3. First comment inference (scans ancestor's first comment body)

When a value is found on an ancestor via title or comment, it is written back onto that ancestor so all siblings benefit automatically.

**Example — three-level chain:**
```
Issue #9099  "DPDK-25.07 Regression Suite for Turin Classic"
  └── Issue #9100  "Regression Testing: Turin Classic 9755 for DPDK Applications"
        └── Issue #9101  "[ST-Turin Classic] Testpmd with rxd/txd=1024 for PF and VF"
```
Pass 3 walks from #9101 → #9100 (no value, title has no version) → #9099 title → finds `25.07` → writes it to #9099, #9100, and #9101.

### Pass 4 — ⚡ Inferred from own title
Scans the item's own title using three strategies:
1. Known version list: `25.11`, `25.07`, `26.03`, `24.11`, `24.07`, `23.11`, `23.07`, `24.03`
2. Prefixed pattern: `dpdk-v25.07`, `dpdk 25.07`, `dpdk-25.07`
3. Bare `XX.YY` pattern anywhere in text

### Pass 5 — 💬 Inferred from own first comment
The item's first comment body is scanned using the same patterns. Limited to first comment only to avoid picking up unrelated discussion.

Items resolved via Passes 2–5 are flagged in the Data Quality tab so the team can set them as proper GitHub fields.

---

## Title-Similarity Linking

When GitHub's native parent link is absent, the dashboard links items by comparing title keywords.

**Primary match** (score 10+): child and candidate parent share at least one benchmark keyword AND at least one platform keyword.

**Fallback A** (score 5+): candidate parent has no platform keyword in its title (e.g. `"Regression Testing: DPDK-25.07 for Testpmd"`). Match on benchmark alone if child has both.

**Fallback B** (score 5+): same logic when candidate has no benchmark keyword.

Safeguards:
- Patterns must be ≥ 4 characters to avoid short false matches
- Cycle detection: a link is rejected if it would make an item its own ancestor (iterative check, no recursion)
- Same-type leaf nodes cannot be linked to each other

---

## Benchmark Taxonomy

### NIC-based

| Canonical | Patterns matched |
|---|---|
| testpmd | testpmd, test-pmd, test_pmd |
| l3fwd | l3fwd |
| l3fwd-graph | l3fwd-graph, l3fwd_graph, l3fwdgraph, l3fwd graph |
| l3fwd-power | l3fwd-power, l3fwd_power, l3fwd power |
| Packet-distributor | packet-distributor, pkt-dist, distributor |
| Event-Dev | event-dev, eventdev, event_dev, event dev |
| Qos-Meter | qos-meter, qosmeter, qos meter |
| Qos-Scheduler | qos-scheduler, qosscheduler, qos scheduler |
| Ipsec-secGW | ipsec-secgw, ipsec secgw, ipsec |
| vhost (PVP) | vhost pvp, vhost-pvp, vhost_pvp |
| vhost (V2V) | vhost v2v, vhost-v2v, vhost_v2v |
| vhost | vhost |
| Virtio PVP | virtio pvp, virtio-pvp, virtio_pvp |
| virtio (V2V) | virtio v2v, virtio-v2v, virtio_v2v |
| virtio | virtio |
| BB-Dev | bb-dev, bbdev, bb_dev |

### NIC and Memory (combined)

| Canonical | Patterns matched |
|---|---|
| NIC and Memory | nic and in-memory, nic and memory, nic & memory, in-memory benchmark, dpdk applications |

### Memory-based

| Canonical | Patterns matched |
|---|---|
| crypto | crypto |
| memcpy | memcpy, mem-cpy, memory copy |
| DMA perf | dma perf, dma-perf, dma_perf, dma |
| Compression_perf | compression, compress perf, compression-perf |
| LPM-Hash | lpm-hash, lpm hash, lpmhash |
| core-lib | core-lib, core lib, corelib, core-libraries, core libraries, core library |

To add a benchmark, add an entry to `BENCHMARK_TAXONOMY`. Longer/more-specific patterns are sorted first automatically.

---

## Platform Taxonomy

| Canonical | Patterns matched |
|---|---|
| Turin Dense | turin dense, turin-dense, turindense |
| Turin Classic | turin classic, turin-classic, st-turin classic, [st-turin, turin |
| Sorano | sorano |
| Genoa | genoa |
| Siena | siena |
| All Platforms | all platforms, allplatforms |

`"Turin Dense"` and `"Turin Classic"` are matched before bare `"Turin"`. `"[ST-Turin"` in titles like `"[ST-Turin Classic] Testpmd"` is recognised directly.

---

## Dashboard Tabs

### 🩺 Regression Health Score
Displayed as a banner above all KPI cards. Scoped to regression testing items only (`Category = "Regression testing"`). Computed as a weighted score:

| Component | Weight | Formula |
|---|---|---|
| Pass Rate | 40% | Completed items ÷ total regression items |
| On-Time Rate | 35% | Non-delayed scheduled items ÷ all scheduled items |
| Data Quality | 25% | Items with all 3 fields resolved ÷ total regression items |

Displays a 0–100% score, letter grade (A/B/C/D/F), colour-coded bar, and three component pills. Updates whenever filters change.

### Overview
Four charts, all updating when filters change:
- **Status distribution** — donut: Completed / Active / Blocked / Backlog
- **Items by Platform** — horizontal stacked bar, status breakdown per platform
- **Items by DPDK Version** — Completed vs Blocked vs Backlog per version
- **Items by Priority** — P0–P3 breakdown by status

### Benchmark Matrix
Heatmap of **Benchmark Type × Platform**. Each cell shows worst-case status (Blocked > Active > Completed > Backlog) plus a mini breakdown (e.g. `3 Blocked · 12 Active · 30 Completed`).

**Click any cell** → drill-down panel opens below the matrix showing every item in that cell, sorted by severity, with direct GitHub links.

### All Items
Full table of every item in the current filter with direct GitHub links, DPDK Version, Platform, Benchmark Type, Ports, Phase, Priority, Status.

### 🏃 Sprint & Effort
Two sections, all filtered by Sprint / Issue Type / Platform selectors at the top.

**Sprint Burndown:**
- 5 KPI cards: Total, Completed, Active, Blocked, Backlog
- Line chart: % completed and % blocked trend across all sprints
- Stacked bar chart: absolute item counts per sprint

**Effort Tracking** (requires `Planned Efforts` and `Actual Efforts` fields to be set on items):
- 4 summary cards: Planned days, Actual days, Remaining days, Variance
- Burn progress bar: actual vs planned, colour-coded blue → amber → red
- Per-item table sorted by worst variance first, with mini burn bars per row

### 📅 Timeline
Filtered by Date Range / DPDK Version / Platform / Priority selectors.

**Delayed Items** — past planned end date, not completed. Sorted oldest overdue first. Colour-coded overdue severity: 🔴 >30 days, 🟠 >14 days, 🟡 >7 days.

**Ongoing Items** — active items, sorted by soonest deadline. Shows days remaining (colour-coded), and a progress bar showing % of planned duration elapsed.

Tab button shows a live badge count of delayed items.

### ⚠ Data Quality
Scoped to `Category = "Regression testing"` items. Falls back to all filtered items if no regression items are found.

**Three categories:**
- **✗ Truly missing** — no value found by any method; needs human input
- **⚠ Needs field update** — value resolved by inference but GitHub field is empty; open ticket and set it
- **✓ Fully resolved** — all 3 fields set directly in GitHub

Per-field summary cards show counts resolved by each source (parent, children, sibling, title, comment).

The truly missing table includes a **First Comment (fetched)** column — if it shows `⚠ No comment fetched`, the PAT needs `Issues: Read-only` added.

**⬇ Export CSV** downloads all regression items with field values and sources for triage.

---

## Filters

### Sidebar (global — affects all tabs)
**Core:** DPDK Version, Platform, Benchmark Type, Ports, Status, Phase
**Planning:** Sprint, Priority, Issue Type, Category
**Search:** title substring, real-time

### Timeline tab (local)
Date Range (presets: current month, next month, ±1/2/3 months, custom range), DPDK Version, Platform, Priority

### Sprint & Effort tab (local)
Sprint, Issue Type, Platform

---

## Extending the Dashboard

### Add a required field to Data Quality checks
```js
const REQUIRED_FIELDS = [
  { key: 'dpdk',      label: 'DPDK Version' },
  { key: 'platform',  label: 'Platform' },
  { key: 'benchmark', label: 'Benchmark Type' },
  { key: 'ports',     label: 'Ports' },   // ← add like this
];
```

### Change what counts as a Regression Testing item
```js
const REGRESSION_CATEGORIES = ["regression testing", "regression", "patch-validation"];
```

### Add a new DPDK version to inference
```js
const KNOWN_DPDK = ["25.11","25.07","26.03","24.11","24.07","23.11","23.07","24.03","26.07"];
```

### Add a new platform
```js
{ patterns: ["naples", "naples25"], canonical: "Naples" },
```

### Add a new benchmark
```js
{ patterns: ["flow-perf","flow_perf"], canonical: "flow-perf", category: "NIC Based" },
```

### Change the project
```js
const ORG            = "AMD-DEAE-CEME";
const PROJECT_NUMBER = 19;
```

### Adjust health score weights
```js
// In renderHealthScore():
const score = Math.round((passRate * 0.40 + onTimeRate * 0.35 + dqRate * 0.25) * 100);
// Change the three multipliers — they must sum to 1.0
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Request failed" | Token expired or wrong permissions | Regenerate PAT with Org → Projects: Read-only and Repo → Issues: Read-only |
| Comments show "⚠ No comment fetched" | PAT missing Issues: Read-only | Add Repository → Issues: Read-only to the PAT and regenerate |
| "Maximum call stack size exceeded" | Circular parent links in data | Already fixed with iterative BFS — if it reappears, report which issue numbers are involved |
| Charts not showing | Chart.js CDN blocked | Table, matrix, data quality, and effort tracking still work fully |
| Fields all blank | Field names changed in GitHub | Run `debug-github-fields.html` and update `FIELDS` / `DATE_FIELDS` / `EFFORT_FIELDS` |
| DPDK version not inferred | Version in unexpected format | Check the actual text and add the pattern to `KNOWN_DPDK` or widen the regex |
| Platform not inferred | Platform name not in taxonomy | Add the variant to `PLATFORM_TAXONOMY` |
| Benchmark not inferred | Benchmark name not in taxonomy | Add the variant to `BENCHMARK_TAXONOMY` |
| Parent-child inheritance not working | Native sub-issue link not set | Use GitHub's "Add sub-issue" button, or ensure title keywords overlap for title-similarity fallback |
| Effort tracking shows no data | Planned/Actual Efforts fields not set | Set `Planned Efforts` and `Actual Efforts` NUMBER fields on issues |
| Health score not showing | No items match regression category | Check `REGRESSION_CATEGORIES` and ensure `Category` field is set on items |
| Sprint burndown flat | Items have no sprint assigned | Assign items to a sprint via the `Sprint Name` field |
| Timeline shows no items | No planned dates set | Set `Planned Start date` and `Planned End date` on active issues |

---

## Potential Enhancements

The following features have been identified as high-value additions for future development, grouped by effort and impact.

### High Impact — Ready to Build

**Column sorting on all tables**
Click any column header to sort ascending/descending. Zero new data needed. Particularly useful in Timeline (sort by "Overdue By") and Effort Tracking (sort by variance). Estimated effort: low.

**Persistent filters via URL hash**
Save filter state into the URL (`#dpdk=25.07&platform=Sorano`) so a filtered view can be shared by copying the link. Pure client-side — no backend needed. Estimated effort: low.

**Timeline CSV export**
The Data Quality tab has CSV export; Timeline doesn't. Delayed items are frequently shared with managers. One-click export of delayed/ongoing items with all date fields. Estimated effort: low.

**Auto-refresh**
Toggle to reload data automatically every 5/10/30 minutes. Useful when the dashboard is displayed on a shared monitor during sprints. Estimated effort: low.

**Keyboard shortcuts**
`Shift+R` to refresh, `1–6` to switch tabs, `Escape` to close drill-down panels. Small improvement but significant during daily standups. Estimated effort: low.

### Medium Impact — Moderate Effort

**Milestone tracking view**
A dedicated tab for `Issue Type = Milestone` items showing each milestone, its child completion percentage, planned end date, and on-time status. Your project already has milestone issue types and date fields. Estimated effort: medium.

**Benchmark completion heatmap**
Variant of the current matrix showing **% complete** per cell rather than worst-case status. More nuanced — a cell with 18 of 20 done looks very different from 2 of 2 done. Estimated effort: medium.

**Stale items detector**
Flag items that have been `In Progress` for more than N configurable days without an `Actual End Date`. Surfaces forgotten or abandoned tasks. Add as a section in the Data Quality tab. Estimated effort: medium.

**Effort data quality check**
Extend Data Quality to flag regression items where `Planned Efforts` is zero or missing — the same pattern as the current DPDK/Platform/Benchmark check. Estimated effort: low-medium.

**Trend over sprints (pass/block rates)**
A line chart in the Overview tab showing pass rate, blocked rate, and completion rate across all sprints. Answers "are we improving sprint over sprint?" without manually comparing filters. Estimated effort: medium.

**Effort vs schedule scatter plot**
X-axis: days overdue. Y-axis: effort variance. Items in the top-right quadrant (both time and effort over budget) are the highest-risk items and deserve escalation. Estimated effort: medium.

### Larger Features

**Gantt chart view**
Use planned start/end dates to draw a Gantt chart per sprint or platform. D3.js is already available. Would make the Timeline tab significantly more visual and easier to present. Estimated effort: high.

**Collapse/expand issue type groups in All Items table**
Group rows by Issue Type (EPIC → Main-Task → Sub-Task) with collapsible sections. Makes the hierarchy visible without opening GitHub. Estimated effort: medium-high.

**Multi-project support**
Load from more than one project number simultaneously and merge results. Useful if regression work is split across multiple GitHub projects or teams. Estimated effort: high.

**Duplicate detection**
Surface items with near-identical titles (e.g. `[ST-Sorano] Testpmd` appearing twice with different issue numbers) as potential duplicates in the Data Quality tab. Estimated effort: medium.

**GitHub write-back**
A "Fix it" button next to inferred items in Data Quality that calls the GitHub API to set the field value directly — turning the dashboard from read-only into an active data-cleaning tool. Requires a token with write permissions. Estimated effort: high.

**Dark/light theme persistence**
Currently theme resets on tab close (sessionStorage). Storing in localStorage would keep the preference across sessions. Estimated effort: trivial.

---

## Companion Tool

`debug-github-fields.html` — open in a browser, paste your PAT, click **Inspect Fields**. Lists every field in your project with its exact name, data type, and all option values. Use this whenever you suspect a field name has changed or before adding new fields to the dashboard config.

---

## Architecture Notes

- **No backend** — all fetching is client-side via `fetch()` to `api.github.com`
- **No localStorage** — token uses `sessionStorage` only, cleared on tab close
- **Pagination** — fetches all project items in 100-item pages automatically; no item limit
- **Single file** — entire dashboard (HTML + CSS + JS) in one file; only external dependency is Chart.js CDN which degrades gracefully
- **Inference is read-only** — inferred values are never written back to GitHub; they exist only in the browser session for display and CSV export
- **Cycle-safe inheritance** — parent-child resolution uses BFS (not recursion) with a 200-step depth limit; cycle detection prevents infinite loops regardless of data shape
- **Theme** — dark/light toggle stored in `sessionStorage`, resets on tab close
