# Campaigns Tab Redesign — Design Spec

**Created:** 2026-05-08 MYT
**Author:** Claude Code (brainstormed with Bryan)
**Status:** Awaiting approval
**Target:** minionions-dashboard (`mom.solworks.io`), Tab 4 (Campaigns)

---

## Overview

Redesign the existing Campaigns tab from a basic status view into a **campaign workbench** — a hybrid dashboard + working station that visualizes campaign-ops agent reasoning, tracks campaign progress, and provides copy-paste-ready content for broadcast execution.

**Key principle:** Read-only smart display of campaign state JSON, with a thin interactive layer for simple actions (refresh exclude lists, mark engagements sent, download CSVs).

---

## Data Architecture

### Source

Campaign-ops agent writes a JSON state file to `shared/projects/campaigns/{tenant-slug}/{campaign-slug}.json`. CRM backend serves it via `GET /boss-view/campaign-state`. Dashboard fetches and renders.

### Existing Fields Referenced

These fields already exist in the campaign state JSON and are used by the dashboard:

- `campaign.name`, `campaign.display_name`, `campaign.promo_code` — Level 1 header
- `campaign.mechanics` — Level 1 mechanics card (type, tiers, bonus, mix_match, internal_packages)
- `dates.orders_open`, `dates.orders_close` — Level 1 date chips, dot calendar computation
- `performance.promo_orders`, `performance.promo_pv`, `performance.unique_customers` — Level 2 KPIs
- `engagement_schedule[].id`, `.touchpoint`, `.planned_date`, `.status`, `.segments`, `.exclude_converted`, `.segment_status`, `.notes` — Level 4 timeline
- `segments.include_lists[].id`, `.label`, `.type`, `.product`, `.count` — Level 2 segments table (totals)

### Schema Extensions Required

The existing campaign state JSON needs these new fields:

```json
{
  "performance": {
    "promo_orders": 23,
    "promo_pv": 4280,
    "unique_customers": 20,
    "new_pv": 1498,              // NEW: PV from new customers
    "new_orders": 8,             // NEW: order count from new customers
    "repeat_pv": 2782,           // NEW: PV from repeat customers
    "repeat_orders": 15,         // NEW: order count from repeat customers
    "last_refreshed": "..."
  },
  "engagement_schedule": [
    {
      // Existing fields:
      "id": "e4",
      "touchpoint": "Voice message reminder",
      "planned_date": "2026-05-08",
      "status": "planned",
      "segments": ["hmg-reorder", "blz-reorder", "bgs-reorder"],
      "exclude_converted": true,
      "segment_status": {},
      "notes": "Personal touch before weekend",

      // NEW fields:
      "track": "repeat",           // "repeat" | "new" — determines which timeline column
      "reasoning": "Personal touch before the weekend. Voice messages have 2x open rate vs text. Scheduled Thursday so recipients hear it before Friday/Saturday shopping decisions.",
      "image_prompt": "Warm Mother's Day scene. Asian family, health supplements gift...",
      "captions": {                // keyed by segment ID — repeat track uses reorder segments, new track uses recent segments
        "hmg-reorder": ["caption variation 1", "caption variation 2", "...", "...", "..."],
        "blz-reorder": ["...", "...", "...", "...", "..."],
        "bgs-reorder": ["...", "...", "...", "...", "..."]
      },
      "broadcast_lists": {         // per-segment list metadata
        "hmg-reorder": { "count": 167 },
        "blz-reorder": { "count": 506 },
        "bgs-reorder": { "count": 75 }
      },
      "broadcast_lists_updated": "2026-05-05T09:14:00+08:00"
    }
  ],
  "setup_tasks": [                  // NEW: replaces flat setup_checklist
    {
      "id": "crm-setup",
      "label": "CRM Setup",
      "owner": "claude",
      "status": "done",             // "done" | "pending" | "partial"
      "subtasks": [
        { "label": "Update products table (promo_name, dates)", "status": "done" },
        { "label": "Test order verification (campaign selector, gift flow)", "status": "done" },
        { "label": "Segment customers → generate include CSVs", "status": "done" }
      ]
    },
    {
      "id": "promo-creatives",
      "label": "Promotion Creatives",
      "owner": "cy",
      "status": "partial",
      "subtasks": [
        { "label": "3x videos", "status": "done" },
        { "label": "30x images (6x angle × 5x variations)", "status": "pending" }
      ]
    }
  ],
  "segments": {
    // Existing: include_lists[], exclude_list
    // Segment totals are derived from include_lists[].count
    // Remaining = total - converted
    "conversions": {                // NEW: per-segment conversion count (from CRM promo orders)
      "hmg-reorder": 3,
      "blz-reorder": 12,
      "bgs-reorder": 0,
      "other-reorder": 0,
      "hmg-recent": 5,
      "blz-recent": 0
    }
  }
}
```

**Caption segment key convention:** Repeat-track engagements use reorder segment keys (`hmg-reorder`, `blz-reorder`). New-track engagements use recent segment keys (`hmg-recent`, `blz-recent`). The `captions` object only contains keys relevant to that engagement's track.

**Segment table derivation:** `total` comes from `segments.include_lists[].count`. `converted` comes from `segments.conversions[segment_id]`. `remaining = total - converted`. "Other" segment follows the same pattern — it exists in `include_lists` as `other-reorder`.

### API Endpoints

All endpoints use the short-form path. The existing `api()` helper prepends `/api/boss-view`.

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/campaign-state` | GET | Serves active campaign JSON (existing) |
| `/campaign-state?slug={slug}` | GET | Serves specific archived campaign (new) |
| `/campaign-list` | GET | Returns `[{slug, name, display_name, status}]` for selector (new) |
| `/campaign-broadcast-list?engagement={id}&segment={id}` | GET | Returns CSV file (Content-Disposition: attachment). Queries CRM orders with promo_name filter, excludes converted customers. (new) |
| `/campaign-state` | PATCH | Body: `{"engagement_id": "e4", "status": "completed"}`. Only allows updating engagement status. (new) |

**Fallback:** If `/campaign-list` returns 404, the campaign selector shows only the active campaign from `/campaign-state` as a single non-interactive option.

### Error Handling

Interactive actions (refresh, download, PATCH) follow the existing dashboard error pattern:
- Show inline error message on failure (red text near the action button)
- Do not block the rest of the page
- Log to console via `console.error()`

### Dot Calendar Edge Cases

- Campaigns up to 42 days (6 weeks) render cleanly in the grid
- Campaigns longer than 42 days: cap at 6 rows, show "+N more days" text below the grid
- Campaign ended: all dots dark navy, "ENDED" replaces days-left number

---

## Layout Structure

### Campaign Selector

Dropdown at top. Defaults to active campaign. Shows archived campaigns for review.

Format: `{name} — {display_name} ({status})`

Example: `雙親26 — Parents Day 2026 (Active)`

### Level 1: Campaign Overview

Two-column layout, side by side:

**Left card — Campaign Identity:**
- Campaign name (Chinese, large)
- Display name (English subtitle)
- Promo code (monospace chip)
- Start/End date chips stacked on the right side of the card
  - Start: blue chip with "START" label
  - End: red chip with "END" label

**Right card — Campaign Mechanics:**
- Type, bonus, products, packages as bullet points
- Read from `campaign.mechanics` in JSON

### Level 2: Progress + KPIs + Segments

Three-column layout: `Dot Calendar (200px) | KPIs (1fr) | Segments (1fr)`

**Dot Calendar (tear-off style):**
- Red top strip + two punch holes + torn bottom edge
- **Days left number on top** (large, red, monospace) with "DAYS LEFT" label
- Below: 7-column dot grid (M T W T F S S)
- Dark navy dots = elapsed days
- Gray dots = remaining days
- Red dot with red ring = today
- Weekend column headers in red
- Hover tooltip shows date on each dot
- Empty cells after campaign end date

**KPI Block:**
- Top section: Total PV (large), supporting text "23 orders · avg 186 PV"
- Bottom section: two columns
  - New PV (blue) with percentage + order count
  - Repeat PV (green) with percentage + order count

**Segments Table:**
- Three columns: product label | Cold (Reorder) | Hot (Recent)
- Column headers have (?) tooltip icons
  - Cold: "Reorder customers: haven't purchased in 14+ days. Need more touchpoints to convert."
  - Hot: "Recent customers: purchased within last 14 days. Higher chance of repeat during promo."
- Each cell shows: `remaining/total · converted` (monospace, converted in green)
- Footer row: format explainer `remaining / total · converted`
- Rows: HMG, BLZ, BGS, Other
- Dash (—) for empty segments

### Level 3: Setup & Content Prep

Single card with task/subtask hierarchy.

**Ordering:** Pending and in-progress tasks float to top. Completed tasks sink below a "Completed" divider.

**Task structure:**
- Parent task row: status icon (○ pending, ◐ partial, ✓ done) + label + count (e.g., "1/2") + owner badge
- Subtask rows: indented 30px, smaller text, same status icons
- Owner badges: colored pills — `claude` (purple), `bryan` (blue accent), `cy` (teal)

**Status icons:**
- `○` pending (gray)
- `◐` partial/in-progress (orange)
- `✓` done (green, text strikethrough, muted color)

**Completed section:**
- Below a thin divider with "Completed" label
- Compact single-line format (no subtask expansion)
- Muted styling (gray owner badge, strikethrough text)

### Level 4: Engagement Playbook

The workbench. Dual-track vertical timeline with unified date alignment.

**Track headers:**
- Side by side with track color bar
- Left: "Repeat Customers" (green) + "WhatsApp" label
- Right: "New Customers" (blue) + "Messenger" label

**Timeline structure:**
- Each date is one grid row: `1fr | 1fr`
- Both tracks share the same date rows for alignment
- Empty slots: just the vertical connector line continues, no placeholder card
- Vertical timeline line: colored for completed section (green/blue), gray for future

**Card states:**

1. **Completed** — colored border + light background, compact info (date, title, segments, sent count)
2. **Next up** — highlighted border (blue for repeat, orange for new), "NEXT" badge, auto-expanded with:
   - Strategy reasoning (glassbox) — left-border accent, italic-style
   - Image prompt — copyable block
   - Caption drafts — numbered variation tabs (1-5), segment switcher dropdown, copy button
   - Broadcast lists — per-segment rows with contact count + CSV download button + refresh button + last-updated timestamp
3. **Future** — neutral border, collapsed, just date + title. Click to expand.

**Content order inside expanded card:**
Title → Strategy Reasoning → Image Prompt → Caption Drafts → Broadcast Lists

**Today line:**
- Single red horizontal line spanning full width across both tracks
- Label: "Today · {date}" centered
- Positioned chronologically between the appropriate date rows

**Date alignment rules:**
- If only one track has an engagement on a date, the other track's column is empty space
- The vertical connector line continues through empty spaces
- This ensures both tracks are always temporally aligned

**Broadcast Lists (inside engagement card):**
- Header row: "BROADCAST LISTS" label + last-updated timestamp + Refresh button
- Refresh button: hits CRM to regenerate exclude list, updates timestamp
- Per-segment rows: segment name + contact count (monospace) + ↓ CSV download button
- Footer: "Excludes N promo buyers" note

### States

**No active campaign:** Centered empty state with icon + "No active campaign" message + last campaign summary if available.

**Active campaign:** Full layout as described above.

**Archived campaign (selected via dropdown):** Same layout but:
- Dot calendar shows all dots as elapsed (dark)
- "ENDED" badge instead of days left
- All engagements in completed state
- No refresh/download actions
- Performance KPIs show final numbers

---

## Interactive Elements (Thin C Layer)

These are the only dashboard-triggered actions. Everything else is read-only.

| Action | Trigger | API Call |
|--------|---------|----------|
| Refresh broadcast lists | Click "↻ Refresh" in engagement card | `GET /boss-view/campaign-broadcast-list?...` |
| Download CSV | Click "↓ CSV" per segment | Direct download from CRM |
| Mark engagement sent | Click status toggle (future) | `PATCH /boss-view/campaign-state` |
| Copy to clipboard | Click "Copy" on captions/prompts | Client-side `navigator.clipboard` |

---

## Technical Implementation Notes

**Codebase:** Single-file `index.html` in minionions-dashboard. Vanilla JS, no frameworks.

**Existing patterns to follow:**
- Tab switching: `switchTab(tabName)` + `loadTabData()` lazy loading
- API: `api(endpoint)` / `apiSafe(endpoint)` helpers with Bearer token auth
- State: `tabDataLoaded.campaigns` cache pattern
- Styling: CSS variables from existing design system (--accent, --green, --orange, etc.)
- Helpers: `fmt()`, `fmtPct()`, `fmtRM()`, `shortMonth()` etc.

**Key considerations:**
- Campaign selector needs campaign list endpoint — falls back to just the active campaign if endpoint not ready
- Dot calendar: computed from `dates.orders_open` and `dates.orders_close` + current date
- Today line position: computed by comparing engagement dates to current date
- Timeline alignment: merge both tracks' dates into a sorted unique date list, render one row per date
- CSV download: needs new CRM endpoint that queries orders with `promo_name` filter, excludes converted customers, returns CSV

**Does NOT require:**
- Any changes to the Ads tab (parallel session)
- New dependencies or build tools
- Changes to authentication or password gate

---

## Out of Scope

- Editing campaign state from the dashboard (beyond simple status toggles)
- Caption generation or AI features in the dashboard
- WhatsApp/LuluChat integration
- Meta Ads publishing from dashboard
- Real-time updates / websockets — standard page refresh is sufficient
