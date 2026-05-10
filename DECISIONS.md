# Minionions Dashboard — Decisions

## 2026-05-08 — Campaigns tab: workbench over dashboard
**Decision:** Built as a hybrid workbench (B with thin C) rather than pure dashboard or full CRUD.
**Reasoning:** Bryan needs to copy-paste captions and download CSVs during campaign execution, not just view status. Full CRUD (option C) would duplicate campaign-ops logic in the CRM backend. Hybrid B gives actionable read view + simple state toggles + CSV downloads, keeping creative work in Claude Code.

## 2026-05-08 — Dual-track engagement timeline
**Decision:** Separate Repeat (WhatsApp) and New (Messenger) tracks side by side with unified date alignment.
**Reasoning:** The two tracks have different cadences — WhatsApp needs 2-3 day spacing, Messenger can go daily. Side-by-side with shared date rows + a single today line gives clear visibility into what's coming on both channels.

## 2026-05-10 — Dashboard sticky notes: separate column, not inline in state
**Decision:** Store user notes in a separate `dashboard_notes` JSONB column, not inside the campaign state JSON.
**Reasoning:** Campaign-ops does full PUT overwrites of the `state` column. If notes lived inside `state`, every campaign-ops push would wipe user notes. Separate column = campaign-ops never touches it. Trade-off: notes are invisible to campaign-ops (can't use them as context). Accepted — notes are personal sticky notes, not system data.

## 2026-05-08 — Dynamic innerHTML rendering
**Decision:** Replaced static HTML with element IDs with full dynamic innerHTML rendering in `loadCampaignsTab()`.
**Reasoning:** The workbench has too many dynamic elements (engagement cards, caption tabs, timeline rows) for static ID binding. Single innerHTML assembly is simpler and matches the complexity of the rendering logic. Trade-off: no incremental DOM updates, but acceptable for a dashboard that reloads on tab switch.
