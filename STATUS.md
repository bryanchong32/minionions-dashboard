# Minionions Dashboard — Status

## 2026-05-10
- **Campaigns tab live end-to-end** — All backend endpoints deployed, campaign state pushed with real data, dashboard rendering verified
- Fixed: cancelled engagement rendering (distinct gray styling + expandable), NEXT badge logic (in_progress prioritized), setup task labels (field name mismatch), segment conversions populated
- Added: dashboard sticky notes (inline text inputs on tasks + engagement cards, saved via PATCH /campaign-notes, separate from campaign state)
- Fixed: Coolify auto-deploy webhooks configured for all 3 apps (ecomwave-crm, ecomwave-crm-staging, minionions-dashboard)
- CRM migration 071: `dashboard_notes` JSONB column on `campaign_states`

## 2026-05-08
- **Campaigns tab redesigned** — Full campaign workbench with dot calendar, dual-track engagement playbook, setup tasks, segment tracking, caption/broadcast workbench
- Spec: `docs/superpowers/specs/2026-05-08-campaigns-tab-redesign-design.md`
- Plan: `docs/superpowers/plans/2026-05-08-campaigns-tab-redesign.md`
- CRM backend endpoints deployed: `/campaign-list`, PATCH `/campaign-state`, `?slug=` param on GET `/campaign-state`
