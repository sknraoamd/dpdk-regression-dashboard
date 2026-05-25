# DPDK Regression Dashboard

A self-contained, single-file HTML dashboard that connects directly to your **GitHub Projects v2** board and provides regression test tracking, data quality checking, and multi-level field inference — with zero backend, zero install, and zero data sent to any third party.

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

If a field name changes in GitHub, update the string here. Use `debug-github-fields.html` to inspect current field names and types at any time.

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

Links are established via GitHub's native **sub-issue parent** relationship (`content.parent.url`). If a parent issue is in the same project it is linked automatically.

When a native parent link is missing, a **title-similarity fallback** attempts to link items by matching benchmark and platform keywords between parent and child titles (see Field Inference below).

---

## Field Inference & Inheritance

For regression testing tickets, three fields are required: **DPDK Version**, **Platform**, and **Benchmark Type**. When a field is blank, the dashboard attempts to resolve it automatically through five passes in order:

### Pass 1 — GitHub field value
The field is set directly on the project item in GitHub. This is the authoritative source and is always used first.

### Pass 2 — ↑ Aggregated from children (UP)
If a parent is missing a field but its descendants have it, the dashboard takes a **majority vote** across all descendants (not just direct children — the full subtree). The most common value wins.

Example: an EPIC is missing Platform, but 8 of its 9 Sub-Tasks have `Platform = Turin Classic` → the EPIC is resolved as `Turin Classic`.

### Pass 2b — ↔ Shared from siblings
If an item is still missing a field after the UP pass, it checks its **siblings** (other items with the same parent). Majority vote across siblings applies.

This handles the case where a group of Sub-Tasks all lack DPDK Version but share the same parent — if even one sibling has the value set (or resolves it from a comment), all others get it.

### Pass 3 — ↓ Inherited from ancestor (DOWN)
The item walks up its full ancestor chain (parent, grandparent, great-grandparent…) looking for a value. At each ancestor it tries three things in order:

1. **Stored field value** on the ancestor
2. **Title inference** — scans the ancestor's title for known DPDK versions, platform names, and benchmark names
3. **First comment inference** — scans the ancestor's first comment body

When a value is found, it is written back onto the ancestor so all siblings of the child automatically benefit from the same resolution without re-scanning.

Example of the full chain this solves:
```
Issue #9099  "DPDK-25.07 Regression Suite for Turin Classic"
  └── Issue #9100  "Regression Testing: Turin Classic 9755 for DPDK Applications"
        └── Issue #9101  "[ST-Turin Classic] Testpmd with rxd/txd=1024 for PF and VF"
```
- #9099 has `"DPDK-25.07"` in its title but no DPDK Version field set
- #9100 has no DPDK version anywhere in title; version is in its first comment
- #9101 has neither
- Pass 3 walks from #9101 → #9100 (no value, no title match) → checks #9100's comment → still none → #9099 title → finds `25.07` → writes it to #9099, #9100, and #9101

### Pass 4 — ⚡ Inferred from own title
Scans the item's own title for DPDK version patterns, platform names, and benchmark names.

DPDK version matching tries three strategies in order:
1. Known version list: `25.11`, `25.07`, `26.03`, `24.11`, `24.07`, `23.11`, `23.07`, `24.03`
2. Prefixed pattern: `dpdk-v25.07`, `dpdk 25.07`, `dpdk-25.07`
3. Bare `XX.YY` pattern anywhere in text

### Pass 5 — 💬 Inferred from own first comment
The item's own first comment body is scanned using the same patterns as Pass 4. Intentionally limited to the first comment only to avoid picking up unrelated discussion.

---

Items resolved via Passes 2–5 are **not** counted as truly missing, but are flagged in the Data Quality tab so the team can set them as proper GitHub fields.

---

## Title-Similarity Linking

When GitHub's native parent link is absent, the dashboard attempts to link items by comparing title keywords.

**Primary match** (score 10+): child and candidate parent share at least one benchmark keyword AND at least one platform keyword.

**Fallback A** (score 5+): candidate parent has no platform keyword in its title at all (e.g. `"Regression Testing: DPDK-25.07 for Testpmd"` — has benchmark but no platform). Match on benchmark alone if the child has both.

**Fallback B** (score 5+): same logic for missing benchmark keyword.

Safeguards that prevent false links:
- Patterns must be ≥ 4 characters to avoid short false matches
- Cycle detection: a link is rejected if it would make an item its own ancestor
- Same-type leaf nodes (e.g. two Sub-Tasks) cannot be linked to each other

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

To add a benchmark, add an entry to `BENCHMARK_TAXONOMY`. Longer/more-specific patterns are sorted first automatically to prevent short patterns swallowing longer ones.

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

`"Turin Dense"` and `"Turin Classic"` are matched before bare `"Turin"`. `"[ST-Turin"` in titles like `"[ST-Turin Classic] Testpmd"` is also recognised directly.

---

## Dashboard Tabs

### Overview
Four charts, all updating when filters change:
- **Status distribution** — donut showing Completed / Active / Blocked / Backlog split
- **Items by Platform** — horizontal stacked bar, status breakdown per platform
- **Items by DPDK Version** — Completed vs Blocked per version
- **Items by Priority** — P0–P3 breakdown by status

### Benchmark Matrix
A heatmap of **Benchmark Type × Platform**. Each cell shows the worst-case status (Blocked > Active > Completed > Backlog) plus a mini breakdown (e.g. `3 Blocked · 12 Active · 30 Completed`).

**Click any cell** to open a drill-down panel below the matrix showing every item in that cell, sorted by severity (Blocked first), with direct GitHub links and full field columns. Use this to investigate unexpectedly high blocked counts.

### All Items
Full table of every item in the current filter — title (linked), DPDK Version, Platform, Benchmark Type, Ports, Phase, Priority, Status.

### ⚠ Data Quality
Scoped automatically to **Regression Testing** items (matched by `Category = "Regression testing"`). Falls back to all filtered items if no regression items are found.

**Summary row:**
- **Truly missing** — no value found by any inference method; needs a human
- **Needs field update** — value was resolved (via inheritance, title, or comment) but the GitHub field is empty; open the ticket and set it
- **Fully resolved** — all 3 fields set directly as GitHub fields

**Per-field cards** show the count of truly missing items and how many were resolved by each source (parent, children, sibling, title, comment).

**Needs field update table** — lists every item where inference found a value but the field isn't set, with coloured source pills (↓ from parent, ↑ from children, ↔ from sibling, ⚡ from title, 💬 from comment).

**Truly missing table** — includes a **First Comment (fetched)** column showing the raw comment text received from GitHub. This is used to diagnose whether comments are being fetched at all. If it shows `⚠ No comment fetched`, the PAT needs `Issues: Read-only` repository permission added.

The tab button shows a live badge count of truly missing items.

#### Export CSV
**⬇ Export CSV** downloads all regression items with columns for each field value and its source. Columns include:
- `DPDK Version`, `DPDK Source` (field / inherited from parent / aggregated from children / shared from sibling / inferred from title / inferred from first comment / missing)
- Same pair for `Platform` and `Benchmark Type`
- `Missing Fields` — semicolon-separated list of fields with no value found
- `Needs Field Update` — fields resolved by inference but not yet set in GitHub

---

## Filters

The sidebar has two groups:

**Core** — DPDK Version, Platform, Benchmark Type, Ports, Status, Phase
**Planning** — Sprint, Priority, Issue Type, Category

All filters apply simultaneously. The search box filters by title substring in real time. **Reset** clears all filters.

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

### Add a new platform
```js
{ patterns: ["naples", "naples25"], canonical: "Naples" },
```

### Add a new benchmark
```js
{ patterns: ["flow-perf","flow_perf","flowperf"], canonical: "flow-perf", category: "NIC Based" },
```

### Add a new DPDK version to inference
```js
const KNOWN_DPDK = ["25.11","25.07","26.03","24.11","24.07","23.11","23.07","24.03","26.07"];
```

### Change the project
```js
const ORG            = "AMD-DEAE-CEME";
const PROJECT_NUMBER = 19;
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Request failed" | Token expired or wrong permissions | Regenerate PAT with Org → Projects: Read-only and Repo → Issues: Read-only |
| Comments show "⚠ No comment fetched" | PAT missing Issues: Read-only | Add Repository → Issues: Read-only to the PAT and regenerate |
| "Maximum call stack size exceeded" | Circular parent links in data | Already fixed — if it reappears, report which issue numbers are involved |
| Charts not showing | Chart.js CDN blocked | Table, matrix, and data quality still work fully |
| Fields all blank | Field names changed in GitHub | Run `debug-github-fields.html` and update `FIELDS` in the script |
| DPDK version not inferred from title | Version written in unexpected format | Check the actual text and add the pattern to `KNOWN_DPDK` or widen the regex |
| Platform not inferred | Platform name not in taxonomy | Add the variant to `PLATFORM_TAXONOMY` |
| Benchmark not inferred | Benchmark name not in taxonomy | Add the variant to `BENCHMARK_TAXONOMY` |
| Parent-child inheritance not working | Native sub-issue link not set in GitHub | Use GitHub's "Add sub-issue" button, OR ensure title keywords overlap for title-similarity fallback |
| Grandparent value not reaching grandchildren | Title-similarity only links one level | Native GitHub sub-issue links work across all levels; title-similarity only links one hop |

---

## Companion Tool

`debug-github-fields.html` — open in a browser, paste your PAT, click **Inspect Fields**. Lists every field in your project with its exact name, data type, and all option values. Use this whenever you suspect a field name has changed or before adding new fields to `FIELDS`.

---

## Architecture Notes

- **No backend** — all fetching is client-side via `fetch()` to `api.github.com`
- **No localStorage** — token uses `sessionStorage` only, cleared on tab close
- **Pagination** — fetches all project items in 100-item pages automatically; no item limit
- **Single file** — entire dashboard (HTML + CSS + JS) in one file; only external dependency is Chart.js CDN which degrades gracefully
- **Theme** — dark/light toggle stored in `sessionStorage`, resets on tab close
- **Inference is read-only** — inferred values are never written back to GitHub; they exist only in the browser session for display and CSV export
