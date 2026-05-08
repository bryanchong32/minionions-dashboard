# Campaigns Tab Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the Campaigns tab (Tab 4) in `minionions-dashboard/index.html` into a campaign workbench with dot calendar, dual-track engagement playbook, setup tasks, and copy-paste-ready broadcast content.

**Architecture:** Single-file vanilla HTML/CSS/JS dashboard. All changes go into `index.html`. Replace the existing campaigns panel HTML (lines 1100-1149), campaigns CSS (lines 90-104), and `loadCampaignsTab()` function (lines 2712-2786). Add new helper functions for dot calendar, timeline rendering, clipboard copy, and CSV download. No new files, no dependencies.

**Tech Stack:** HTML5, CSS3, vanilla JavaScript, Chart.js 4.4.7 (existing)

**Spec:** `docs/superpowers/specs/2026-05-08-campaigns-tab-redesign-design.md`

**Important context:**
- This is a single-file app (`index.html`, ~2800 lines). All HTML, CSS, and JS live in one file.
- The Ads tab is being worked on in a parallel session — do NOT modify any ads-related code.
- Existing API helpers: `api(endpoint)`, `apiSafe(endpoint)` — these prepend `/api/boss-view`.
- Existing formatters: `fmt(n)`, `fmtPct(n)`, `fmtRM(n)`, `shortMonth(ym)`.
- No test framework — verification is visual (open in browser, check layout).
- The existing campaigns tab (HTML lines 1100-1149, CSS lines 90-104, JS lines 2712-2800) will be fully replaced.

---

### Task 1: Replace Campaigns CSS

**Files:**
- Modify: `index.html` — CSS section (lines ~90-104, the `.timeline-*` and `.checklist-*` classes)

- [ ] **Step 1: Replace the existing campaigns CSS block** with new styles. Find the block starting with `.timeline-row` and ending with `.checklist-owner--cy` and replace with:

```css
/* ===== CAMPAIGNS TAB ===== */
.camp-selector { padding: 6px 12px; border: 1px solid var(--border); border-radius: 6px; font-size: 13px; font-family: var(--font); background: var(--surface); min-width: 240px; }
.camp-grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin-bottom: 16px; }
.camp-grid-3 { display: grid; grid-template-columns: 200px 1fr 1fr; gap: 16px; margin-bottom: 16px; }
.camp-card { background: var(--surface); border: 1px solid var(--border); border-radius: 10px; padding: 20px; }
.camp-card--overflow { overflow: hidden; }
.camp-date-chip { border-radius: 6px; padding: 5px 12px; text-align: center; }
.camp-date-chip--start { background: #EFF6FF; border: 1px solid #BFDBFE; }
.camp-date-chip--end { background: #FEF2F2; border: 1px solid #FECACA; }
.camp-date-chip__label { font-size: 8px; text-transform: uppercase; letter-spacing: 0.5px; font-weight: 600; }
.camp-date-chip__value { font-size: 14px; font-weight: 600; font-family: var(--mono); margin-top: 1px; }

/* Dot Calendar */
.dot-cal { background: var(--surface); border: 1px solid var(--border); border-radius: 10px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.06); }
.dot-cal__strip { background: var(--red); height: 10px; }
.dot-cal__holes { display: flex; justify-content: space-around; padding: 5px 20px 0; }
.dot-cal__hole { width: 10px; height: 10px; border-radius: 50%; background: var(--border-light); border: 1px solid var(--border); }
.dot-cal__body { padding: 8px 16px 14px; }
.dot-cal__days-left { text-align: center; margin-bottom: 10px; padding-bottom: 8px; border-bottom: 1px solid var(--border-light); }
.dot-cal__number { font-size: 32px; font-weight: 700; font-family: var(--mono); color: var(--red); }
.dot-cal__label { font-size: 9px; color: var(--text-3); text-transform: uppercase; letter-spacing: 1px; font-weight: 600; margin-top: 1px; }
.dot-cal__grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 3px; margin-bottom: 3px; }
.dot-cal__header { font-size: 8px; text-align: center; font-weight: 600; color: var(--text-3); }
.dot-cal__header--weekend { color: var(--red); }
.dot-cal__dot { width: 20px; height: 20px; border-radius: 50%; margin: 0 auto; cursor: default; }
.dot-cal__dot--elapsed { background: #1E40AF; }
.dot-cal__dot--remaining { background: var(--border); }
.dot-cal__dot--today { background: var(--red); box-shadow: 0 0 0 2px var(--surface), 0 0 0 3px var(--red); }
.dot-cal__torn { height: 6px; background: repeating-conic-gradient(var(--border) 0% 25%, transparent 0% 50%) 0 0 / 12px 6px; opacity: 0.6; }

/* KPI Block */
.camp-kpi { background: var(--surface); border: 1px solid var(--border); border-radius: 10px; overflow: hidden; }
.camp-kpi__top { padding: 14px; text-align: center; border-bottom: 1px solid var(--border); }
.camp-kpi__split { display: grid; grid-template-columns: 1fr 1fr; gap: 1px; background: var(--border); }
.camp-kpi__cell { background: var(--surface); padding: 12px; text-align: center; }
.camp-kpi__label { font-size: 10px; color: var(--text-3); text-transform: uppercase; letter-spacing: 0.5px; }
.camp-kpi__value { font-size: 20px; font-weight: 600; font-family: var(--mono); margin-top: 2px; }
.camp-kpi__value--lg { font-size: 28px; font-weight: 700; }
.camp-kpi__sub { font-size: 11px; color: var(--text-2); margin-top: 2px; }

/* Segments Table */
.camp-seg { background: var(--surface); border: 1px solid var(--border); border-radius: 10px; overflow: hidden; }
.camp-seg__row { display: grid; grid-template-columns: 56px 1fr 1fr; }
.camp-seg__row--header { background: var(--border-light); }
.camp-seg__row + .camp-seg__row { border-top: 1px solid var(--border-light); }
.camp-seg__cell { padding: 10px; font-size: 11px; }
.camp-seg__cell--label { font-size: 12px; font-weight: 600; }
.camp-seg__cell--center { text-align: center; font-family: var(--mono); }
.camp-seg__cell--header { font-size: 11px; font-weight: 600; color: var(--text-2); text-align: center; cursor: help; }
.camp-seg__help { font-size: 9px; color: var(--text-3); border: 1px solid var(--text-3); border-radius: 50%; width: 12px; height: 12px; display: inline-flex; align-items: center; justify-content: center; margin-left: 2px; }
.camp-seg__footer { padding: 6px 10px; border-top: 1px solid var(--border-light); font-size: 9px; color: var(--text-3); text-align: center; }

/* Setup Tasks */
.camp-task { display: flex; align-items: center; gap: 8px; padding: 6px 8px; border-radius: 6px; font-size: 12px; }
.camp-task--done { background: var(--green-bg); }
.camp-task--partial { background: var(--orange-bg); }
.camp-task--pending { background: var(--border-light); }
.camp-task__icon { font-size: 14px; }
.camp-task__label--done { text-decoration: line-through; color: var(--text-3); }
.camp-task__count { font-size: 10px; color: var(--text-3); margin-left: 4px; }
.camp-task__owner { margin-left: auto; font-size: 10px; color: white; padding: 1px 6px; border-radius: 3px; }
.camp-task__owner--claude { background: var(--purple); }
.camp-task__owner--bryan { background: var(--accent); }
.camp-task__owner--cy { background: var(--teal); }
.camp-task__owner--muted { background: var(--border); color: var(--text-2); }
.camp-subtask { font-size: 11px; display: flex; align-items: center; gap: 6px; margin-left: 30px; margin-top: 2px; }

/* Engagement Playbook */
.camp-playbook { background: var(--surface); border: 1px solid var(--border); border-radius: 10px; padding: 20px; }
.camp-track-headers { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 14px; }
.camp-track-header { display: flex; align-items: center; gap: 8px; padding-bottom: 8px; }
.camp-track-header--repeat { border-bottom: 2px solid var(--green); }
.camp-track-header--new { border-bottom: 2px solid var(--accent); }
.camp-track-dot { width: 8px; height: 8px; border-radius: 50%; }
.camp-timeline-row { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 8px; }
.camp-timeline-col { position: relative; padding-left: 24px; }
.camp-timeline-line { position: absolute; left: 7px; top: 0; bottom: -8px; width: 2px; }
.camp-timeline-line--green { background: #BBF7D0; }
.camp-timeline-line--blue { background: #BFDBFE; }
.camp-timeline-line--gray { background: var(--border); }
.camp-timeline-node { position: absolute; left: 0; top: 6px; width: 16px; height: 16px; border-radius: 50%; border: 2px solid var(--surface); z-index: 1; }
.camp-timeline-node--done-repeat { background: var(--green); box-shadow: 0 0 0 2px var(--green); }
.camp-timeline-node--done-new { background: var(--accent); box-shadow: 0 0 0 2px var(--accent); }
.camp-timeline-node--next { box-shadow: 0 0 0 2px; }
.camp-timeline-node--future { background: var(--surface); border: 2px solid var(--border); box-shadow: none; }

.camp-eng-card { border-radius: 8px; padding: 10px; }
.camp-eng-card--done-repeat { border: 1px solid #BBF7D0; background: #F0FDF4; }
.camp-eng-card--done-new { border: 1px solid #BFDBFE; background: #EFF6FF; }
.camp-eng-card--next { border-width: 2px; border-style: solid; position: relative; }
.camp-eng-card--next-repeat { border-color: var(--accent); background: #EFF6FF; }
.camp-eng-card--next-new { border-color: var(--orange); background: var(--orange-bg); }
.camp-eng-card--future { border: 1px solid var(--border); background: var(--surface); cursor: pointer; }
.camp-eng-card__date { font-size: 10px; font-family: var(--mono); color: var(--text-3); }
.camp-eng-card__title { font-size: 12px; font-weight: 500; margin-top: 2px; }
.camp-eng-card__meta { font-size: 10px; color: var(--text-2); margin-top: 2px; }
.camp-eng-badge { position: absolute; top: -8px; left: 12px; color: white; font-size: 9px; padding: 1px 8px; border-radius: 3px; text-transform: uppercase; font-weight: 600; letter-spacing: 0.5px; }

.camp-eng-detail { margin-top: 8px; background: var(--surface); border-radius: 6px; padding: 8px; }
.camp-eng-detail__label { font-size: 9px; font-weight: 600; text-transform: uppercase; letter-spacing: 0.5px; color: var(--text-3); margin-bottom: 2px; }
.camp-eng-strategy { border-left: 3px solid; padding: 8px; font-size: 10px; color: var(--text-2); line-height: 1.4; background: var(--surface); border-radius: 6px; margin-top: 8px; }
.camp-eng-copybox { background: var(--bg); border: 1px solid var(--border); border-radius: 4px; padding: 8px; font-size: 11px; line-height: 1.6; white-space: pre-wrap; position: relative; }
.camp-eng-copybtn { position: absolute; top: 3px; right: 3px; background: var(--text); color: white; border: none; border-radius: 3px; padding: 1px 6px; font-size: 9px; cursor: pointer; font-family: var(--font); }
.camp-eng-copybtn:hover { opacity: 0.8; }
.camp-caption-tabs { display: flex; gap: 2px; }
.camp-caption-tab { font-size: 9px; padding: 1px 4px; border-radius: 2px; cursor: pointer; border: 1px solid var(--border); background: var(--border-light); font-family: var(--font); }
.camp-caption-tab--active { background: var(--accent); color: white; border-color: var(--accent); }

.camp-broadcast { margin-top: 6px; background: var(--surface); border: 1px solid var(--border); border-radius: 6px; padding: 8px; }
.camp-broadcast__row { display: flex; align-items: center; justify-content: space-between; padding: 3px 6px; background: var(--bg); border-radius: 3px; font-size: 10px; margin-bottom: 3px; }
.camp-broadcast__dl { background: var(--text); color: white; border: none; border-radius: 3px; padding: 1px 6px; font-size: 9px; cursor: pointer; font-family: var(--font); }
.camp-broadcast__refresh { background: none; border: 1px solid var(--border); border-radius: 3px; padding: 1px 6px; font-size: 9px; cursor: pointer; color: var(--accent); font-family: var(--font); }

.camp-today-line { display: flex; align-items: center; margin: 16px 0; }
.camp-today-line__bar { height: 2px; flex: 1; background: var(--red); }
.camp-today-line__label { font-size: 9px; font-weight: 600; color: var(--red); text-transform: uppercase; letter-spacing: 0.5px; white-space: nowrap; padding: 0 10px; }

/* Toast */
.camp-toast { position: fixed; bottom: 24px; left: 50%; transform: translateX(-50%); background: var(--text); color: white; padding: 8px 16px; border-radius: 8px; font-size: 13px; font-family: var(--font); z-index: 9999; opacity: 0; transition: opacity 0.2s; pointer-events: none; }
.camp-toast--visible { opacity: 1; }

/* Utility */
.hidden { display: none !important; }
```

- [ ] **Step 2: Verify no CSS class conflicts** by searching for any other usage of `.camp-` prefix in the file. There should be none (existing uses `camp-` as ID prefix like `#camp-name`).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "style: add campaigns tab redesign CSS classes"
```

---

### Task 2: Replace Campaigns Panel HTML

**Files:**
- Modify: `index.html` — the `panel-campaigns` div (lines ~1100-1149)

- [ ] **Step 1: Replace the entire `panel-campaigns` div** (from `<!-- ===== TAB 4: CAMPAIGNS ===== -->` through the closing `</div>` of `panel-campaigns`) with the new HTML structure. The new HTML is rendered dynamically by JS, so the panel just needs container divs:

```html
<!-- ===== TAB 4: CAMPAIGNS ===== -->
<div class="tab-panel" id="panel-campaigns">
  <div id="campaignsLoading" class="loading">Loading campaign data</div>
  <div id="campaigns-empty" style="display:none;">
    <div style="text-align:center; padding:80px 24px; color:var(--text-3);">
      <div style="font-size:40px; margin-bottom:12px;">📋</div>
      <div style="font-size:15px; font-weight:500;">No active campaign</div>
      <div style="font-size:13px; margin-top:4px;">Campaign data will appear here when a promo is running.</div>
    </div>
  </div>
  <div id="campaigns-content" style="display:none;"></div>
</div>
```

Note: The content is now fully rendered by `loadCampaignsTab()` via `innerHTML`. This is simpler than maintaining dozens of element IDs.

- [ ] **Step 2: Add toast element** before the closing `</body>` tag:

```html
<div class="camp-toast" id="campToast"></div>
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "refactor: simplify campaigns panel HTML to dynamic rendering"
```

---

### Task 3: Implement Helper Functions

**Files:**
- Modify: `index.html` — `<script>` section, add new functions after existing helpers

- [ ] **Step 1: Add clipboard copy helper** after the existing helper functions (after `prevMonth()`):

```javascript
// ===== CAMPAIGNS HELPERS =====
function campCopy(text) {
  const toast = document.getElementById('campToast');
  const show = (msg) => { toast.textContent = msg; toast.classList.add('camp-toast--visible'); setTimeout(() => toast.classList.remove('camp-toast--visible'), 1500); };
  try {
    navigator.clipboard.writeText(text).then(() => show('Copied!')).catch(() => {
      const ta = document.createElement('textarea'); ta.value = text; ta.style.position = 'fixed'; ta.style.left = '-9999px';
      document.body.appendChild(ta); ta.select(); document.execCommand('copy'); document.body.removeChild(ta); show('Copied!');
    });
  } catch (e) { show('Copy failed'); }
}
```

- [ ] **Step 2: Add dot calendar renderer:**

```javascript
function renderDotCalendar(startDate, endDate) {
  const start = new Date(startDate + 'T00:00:00+08:00');
  const end = new Date(endDate + 'T23:59:59+08:00');
  const now = new Date();
  const today = new Date(now.toLocaleString('en-US', { timeZone: 'Asia/Kuala_Lumpur' }));
  const todayStr = today.toISOString().slice(0, 10);

  const daysLeft = Math.max(0, Math.ceil((end - now) / 86400000));
  const ended = now > end;

  // Build array of campaign days
  const days = [];
  const d = new Date(start);
  while (d <= end) {
    days.push(new Date(d));
    d.setDate(d.getDate() + 1);
  }

  // Pad to start on Monday (ISO weekday)
  const startDow = (start.getDay() + 6) % 7; // 0=Mon
  const padBefore = startDow;

  // Cap at 42 days (6 weeks) for grid
  const maxDots = 42 - padBefore;
  const displayDays = days.slice(0, maxDots);
  const overflow = days.length > maxDots ? days.length - maxDots : 0;

  // Pad after to fill last row
  const totalCells = padBefore + displayDays.length;
  const padAfter = totalCells % 7 === 0 ? 0 : 7 - (totalCells % 7);

  let dotsHtml = '';
  // Empty padding cells
  for (let i = 0; i < padBefore; i++) dotsHtml += '<div></div>';
  // Day dots
  for (const day of displayDays) {
    const ds = day.toISOString().slice(0, 10);
    const elapsed = ds <= todayStr && !ended ? true : ended;
    const isToday = ds === todayStr && !ended;
    const cls = isToday ? 'dot-cal__dot dot-cal__dot--today' : elapsed ? 'dot-cal__dot dot-cal__dot--elapsed' : 'dot-cal__dot dot-cal__dot--remaining';
    const months = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];
    const tip = months[day.getMonth()] + ' ' + day.getDate() + (isToday ? ' — TODAY' : ds === endDate ? ' — LAST DAY' : '');
    dotsHtml += `<div class="${cls}" title="${tip}"></div>`;
  }
  // Trailing padding
  for (let i = 0; i < padAfter; i++) dotsHtml += '<div></div>';

  const overflowHtml = overflow > 0 ? `<div style="text-align:center; font-size:9px; color:var(--text-3); margin-top:4px;">+${overflow} more days</div>` : '';

  return `<div class="dot-cal">
    <div class="dot-cal__strip"></div>
    <div class="dot-cal__holes"><div class="dot-cal__hole"></div><div class="dot-cal__hole"></div></div>
    <div class="dot-cal__body">
      <div class="dot-cal__days-left">
        <div class="dot-cal__number">${ended ? 'ENDED' : daysLeft}</div>
        ${ended ? '' : '<div class="dot-cal__label">Days Left</div>'}
      </div>
      <div class="dot-cal__grid" style="margin-bottom:5px;">
        <div class="dot-cal__header">M</div><div class="dot-cal__header">T</div><div class="dot-cal__header">W</div>
        <div class="dot-cal__header">T</div><div class="dot-cal__header">F</div>
        <div class="dot-cal__header dot-cal__header--weekend">S</div><div class="dot-cal__header dot-cal__header--weekend">S</div>
      </div>
      <div class="dot-cal__grid">${dotsHtml}</div>
      ${overflowHtml}
    </div>
    <div class="dot-cal__torn"></div>
  </div>`;
}
```

- [ ] **Step 3: Add segment table builder:**

```javascript
function buildSegmentTable(segments) {
  const lists = segments?.include_lists || [];
  const conversions = segments?.conversions || {};

  // Group by product
  const products = {};
  for (const list of lists) {
    const parts = list.id.split('-');
    const product = parts[0].toUpperCase();
    const type = parts.slice(1).join('-'); // 'reorder' or 'recent'
    if (!products[product]) products[product] = {};
    products[product][type] = { total: list.count || 0, converted: conversions[list.id] || 0 };
  }

  const order = ['HMG', 'BLZ', 'BGS', 'OTHER'];
  const sorted = order.filter(p => products[p]).concat(Object.keys(products).filter(p => !order.includes(p)));

  const cellHtml = (data) => {
    if (!data) return '<div class="camp-seg__cell camp-seg__cell--center" style="color:var(--text-3);">—</div>';
    const remaining = data.total - data.converted;
    return `<div class="camp-seg__cell camp-seg__cell--center"><span style="color:var(--text-3);">${remaining}/</span>${data.total} · <span style="color:var(--green); font-weight:600;">${data.converted}</span></div>`;
  };

  let rows = '';
  for (const p of sorted) {
    rows += `<div class="camp-seg__row">
      <div class="camp-seg__cell camp-seg__cell--label">${p}</div>
      ${cellHtml(products[p]?.reorder)}
      ${cellHtml(products[p]?.recent)}
    </div>`;
  }

  return `<div class="camp-seg">
    <div class="camp-seg__row camp-seg__row--header">
      <div class="camp-seg__cell"></div>
      <div class="camp-seg__cell camp-seg__cell--header" title="Reorder customers: haven't purchased in 14+ days. Need more touchpoints to convert.">Cold <span class="camp-seg__help">?</span></div>
      <div class="camp-seg__cell camp-seg__cell--header" title="Recent customers: purchased within last 14 days. Higher chance of repeat during promo.">Hot <span class="camp-seg__help">?</span></div>
    </div>
    ${rows}
    <div class="camp-seg__footer">remaining / total · <span style="color:var(--green);">converted</span></div>
  </div>`;
}
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add campaigns helper functions (clipboard, dot calendar, segments)"
```

---

### Task 4: Implement Setup Tasks Renderer

**Files:**
- Modify: `index.html` — add function in `<script>` section

- [ ] **Step 1: Add setup tasks renderer:**

```javascript
function renderSetupTasks(tasks) {
  if (!tasks || tasks.length === 0) return '';

  const statusIcon = (s) => s === 'done' ? '<span style="color:var(--green);">✓</span>' : s === 'partial' ? '<span style="color:var(--orange);">◐</span>' : '<span style="color:var(--text-3);">○</span>';
  const ownerCls = (o, done) => done ? 'camp-task__owner camp-task__owner--muted' : `camp-task__owner camp-task__owner--${o || ''}`;

  const pending = tasks.filter(t => t.status !== 'done');
  const done = tasks.filter(t => t.status === 'done');

  let html = '';

  // Pending/partial first
  for (const task of pending) {
    const cls = task.status === 'partial' ? 'camp-task camp-task--partial' : 'camp-task camp-task--pending';
    const doneCount = (task.subtasks || []).filter(s => s.status === 'done').length;
    const totalCount = (task.subtasks || []).length;
    const countHtml = totalCount > 0 ? `<span class="camp-task__count">${doneCount}/${totalCount}</span>` : '';

    html += `<div style="margin-bottom:10px;">
      <div class="${cls}">
        <span class="camp-task__icon">${statusIcon(task.status)}</span>
        <span>${task.label}</span>${countHtml}
        <span class="${ownerCls(task.owner, false)}">${task.owner || ''}</span>
      </div>`;

    for (const sub of (task.subtasks || [])) {
      const subLabel = sub.status === 'done' ? `<span style="color:var(--text-3); text-decoration:line-through;">${sub.label}</span>` : `<span>${sub.label}</span>`;
      html += `<div class="camp-subtask">${statusIcon(sub.status)} ${subLabel}</div>`;
    }
    html += '</div>';
  }

  // Completed divider
  if (done.length > 0) {
    html += '<div style="border-top:1px solid var(--border); padding-top:10px; margin-top:4px;"><div style="font-size:10px; color:var(--text-3); text-transform:uppercase; letter-spacing:0.5px; margin-bottom:6px;">Completed</div>';
    for (const task of done) {
      const doneCount = (task.subtasks || []).length;
      html += `<div style="display:flex; align-items:center; gap:8px; padding:4px 8px; margin-bottom:4px;">
        <span style="color:var(--green); font-size:14px;">✓</span>
        <span style="font-size:12px; color:var(--text-3); text-decoration:line-through;">${task.label}</span>
        <span style="font-size:10px; color:var(--text-3);">${doneCount}/${doneCount}</span>
        <span class="${ownerCls(task.owner, true)}">${task.owner || ''}</span>
      </div>`;
    }
    html += '</div>';
  }

  const totalDone = tasks.filter(t => t.status === 'done').length;
  return `<div class="camp-card" style="margin-bottom:16px;">
    <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:14px;">
      <div style="font-size:14px; font-weight:600;">Setup & Content Prep</div>
      <div style="font-size:11px; color:var(--text-3);">${totalDone}/${tasks.length} tasks</div>
    </div>
    ${html}
  </div>`;
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add setup tasks renderer with task/subtask hierarchy"
```

---

### Task 5: Implement Engagement Playbook Renderer

**Files:**
- Modify: `index.html` — add function in `<script>` section

This is the largest task — the dual-track timeline with today line, card states, and workbench content.

- [ ] **Step 1: Add engagement card renderer** (handles all 3 card states):

```javascript
function renderEngCard(eng, isNext, isArchived) {
  const track = eng.track || 'repeat';
  if (!eng.track) console.warn('Engagement', eng.id, 'missing track field, defaulting to repeat');
  const status = eng.status || 'planned';
  const isFuture = status === 'planned' && !isNext;
  const isDone = status === 'completed';

  const dateStr = eng.planned_date ? (() => {
    const d = new Date(eng.planned_date + 'T00:00:00+08:00');
    const days = ['Sun','Mon','Tue','Wed','Thu','Fri','Sat'];
    const months = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];
    return months[d.getMonth()] + ' ' + d.getDate() + ' · ' + days[d.getDay()];
  })() : 'TBD';

  const segNames = (eng.segments || []).map(s => s.replace(/-reorder|-recent/, '').toUpperCase()).join(', ');
  const exclNote = eng.exclude_converted ? ' · excl. converted' : '';

  // Done card
  if (isDone) {
    const cls = track === 'repeat' ? 'camp-eng-card camp-eng-card--done-repeat' : 'camp-eng-card camp-eng-card--done-new';
    return `<div class="${cls}">
      <div class="camp-eng-card__date">${dateStr}</div>
      <div class="camp-eng-card__title">${eng.id.toUpperCase()} · ${eng.touchpoint}</div>
      <div class="camp-eng-card__meta">${segNames}${exclNote}</div>
    </div>`;
  }

  // Future card (collapsed)
  if (isFuture) {
    return `<div class="camp-eng-card camp-eng-card--future" onclick="this.querySelector('.camp-eng-expand')?.classList.toggle('hidden')">
      <div class="camp-eng-card__date">${dateStr}</div>
      <div class="camp-eng-card__title">${eng.id.toUpperCase()} · ${eng.touchpoint}</div>
      ${eng.reasoning || eng.captions ? '<div class="camp-eng-expand hidden" style="margin-top:8px;">' + renderEngDetails(eng, isArchived) + '</div>' : ''}
    </div>`;
  }

  // Next up card (expanded)
  const cardCls = track === 'repeat' ? 'camp-eng-card camp-eng-card--next camp-eng-card--next-repeat' : 'camp-eng-card camp-eng-card--next camp-eng-card--next-new';
  const badgeBg = track === 'repeat' ? 'var(--accent)' : 'var(--orange)';
  return `<div class="${cardCls}">
    <div class="camp-eng-badge" style="background:${badgeBg};">Next</div>
    <div style="padding-top:4px;">
      <div class="camp-eng-card__date">${dateStr}</div>
      <div class="camp-eng-card__title">${eng.id.toUpperCase()} · ${eng.touchpoint}</div>
      <div class="camp-eng-card__meta">${segNames}${exclNote}</div>
      ${renderEngDetails(eng, isArchived)}
    </div>
  </div>`;
}

function renderEngDetails(eng, isArchived) {
  let html = '';

  // Strategy reasoning
  if (eng.reasoning) {
    const borderColor = (eng.track || 'repeat') === 'repeat' ? 'var(--accent)' : 'var(--orange)';
    html += `<div class="camp-eng-strategy" style="border-color:${borderColor};">
      <div class="camp-eng-detail__label">Strategy</div>
      ${eng.reasoning}
    </div>`;
  }

  // Image prompt
  if (eng.image_prompt) {
    html += `<div class="camp-eng-detail" style="margin-top:6px;">
      <div class="camp-eng-detail__label">Image Prompt</div>
      <div class="camp-eng-copybox">${eng.image_prompt}<button class="camp-eng-copybtn" onclick="event.stopPropagation(); campCopy(${JSON.stringify(eng.image_prompt).replace(/'/g, '\\u0027')})">Copy</button></div>
    </div>`;
  }

  // Captions
  if (eng.captions && Object.keys(eng.captions).length > 0) {
    const segments = Object.keys(eng.captions);
    const engId = eng.id;
    html += `<div class="camp-eng-detail" style="margin-top:6px;" id="captions-${engId}">
      <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:4px;">
        <div class="camp-eng-detail__label">Caption — ${segments[0].replace(/-reorder|-recent/, '').toUpperCase()}</div>
        <div class="camp-caption-tabs">
          ${eng.captions[segments[0]].map((_, i) => `<button class="camp-caption-tab ${i === 0 ? 'camp-caption-tab--active' : ''}" onclick="event.stopPropagation(); switchCaption('${engId}', ${i})">${i + 1}</button>`).join('')}
        </div>
      </div>
      <div class="camp-eng-copybox" id="caption-text-${engId}">${eng.captions[segments[0]][0] || ''}<button class="camp-eng-copybtn" onclick="event.stopPropagation(); campCopy(document.getElementById('caption-text-${engId}').textContent.replace('Copy','').trim())">Copy</button></div>
      <div style="margin-top:4px;">
        <select style="font-size:10px; padding:2px 6px; border:1px solid var(--border); border-radius:3px; background:var(--surface); font-family:var(--font);"
          onchange="event.stopPropagation(); switchCaptionSegment('${engId}', this.value)">
          ${segments.map(s => `<option value="${s}">${s.replace(/-reorder|-recent/, '').toUpperCase()} ${s.includes('reorder') ? 'Reorder' : 'Recent'}</option>`).join('')}
        </select>
      </div>
    </div>`;
  } else if (!eng.captions) {
    html += `<div class="camp-eng-detail" style="margin-top:6px;">
      <div class="camp-eng-detail__label">Captions</div>
      <div style="font-size:11px; color:var(--text-3); font-style:italic;">Captions pending</div>
    </div>`;
  }

  // Broadcast lists
  if (!isArchived) {
    if (eng.broadcast_lists && Object.keys(eng.broadcast_lists).length > 0) {
      const updated = eng.broadcast_lists_updated ? new Date(eng.broadcast_lists_updated).toLocaleString('en-US', { month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit', hour12: false, timeZone: 'Asia/Kuala_Lumpur' }) : 'Never refreshed';
      let listRows = '';
      for (const [segId, info] of Object.entries(eng.broadcast_lists)) {
        const label = segId.replace(/-reorder|-recent/, '').toUpperCase();
        listRows += `<div class="camp-broadcast__row">
          <span>${label}</span>
          <div style="display:flex; align-items:center; gap:6px;">
            <span style="color:var(--text-3); font-family:var(--mono);">${info.count || 0}</span>
            <button class="camp-broadcast__dl" onclick="event.stopPropagation(); campDownloadCSV('${eng.id}', '${segId}')">↓</button>
          </div>
        </div>`;
      }
      html += `<div class="camp-broadcast" style="margin-top:6px;">
        <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:4px;">
          <div class="camp-eng-detail__label">Broadcast Lists</div>
          <div style="display:flex; align-items:center; gap:4px;">
            <span style="font-size:8px; color:var(--text-3);">${updated}</span>
            <button class="camp-broadcast__refresh" onclick="event.stopPropagation(); refreshBroadcastList('${eng.id}')">↻ Refresh</button>
          </div>
        </div>
        ${listRows}
      </div>`;
    } else if (!eng.broadcast_lists) {
      html += `<div class="camp-broadcast" style="margin-top:6px;">
        <div class="camp-eng-detail__label">Broadcast Lists</div>
        <div style="font-size:10px; color:var(--text-3); font-style:italic;">Lists not generated yet</div>
      </div>`;
    }
  }

  return html;
}
```

- [ ] **Step 2: Add caption switching and broadcast refresh functions:**

```javascript
// Caption state per engagement
let captionState = {};

function switchCaption(engId, idx) {
  if (!captionState[engId]) return;
  const { segment, captions } = captionState[engId];
  const texts = captions[segment] || [];
  const el = document.getElementById('caption-text-' + engId);
  if (el && texts[idx]) {
    el.innerHTML = texts[idx] + '<button class="camp-eng-copybtn" onclick="event.stopPropagation(); campCopy(document.getElementById(\'caption-text-' + engId + '\').textContent.replace(\'Copy\',\'\').trim())">Copy</button>';
  }
  // Update active tab
  const container = document.getElementById('captions-' + engId);
  if (container) {
    container.querySelectorAll('.camp-caption-tab').forEach((t, i) => t.classList.toggle('camp-caption-tab--active', i === idx));
  }
}

function switchCaptionSegment(engId, segId) {
  if (!captionState[engId]) return;
  captionState[engId].segment = segId;
  const captions = captionState[engId].captions;
  // Update label
  const container = document.getElementById('captions-' + engId);
  if (container) {
    const label = container.querySelector('.camp-eng-detail__label');
    if (label) label.textContent = 'Caption — ' + segId.replace(/-reorder|-recent/, '').toUpperCase();
    // Reset tabs
    const tabs = container.querySelectorAll('.camp-caption-tab');
    const texts = captions[segId] || [];
    tabs.forEach((t, i) => { t.classList.toggle('camp-caption-tab--active', i === 0); t.style.display = i < texts.length ? '' : 'none'; });
  }
  switchCaption(engId, 0);
}

async function refreshBroadcastList(engId) {
  try {
    // Hit the broadcast-list endpoint to trigger server-side regeneration
    // then reload the tab to show updated counts
    await api('/campaign-broadcast-list?engagement=' + engId + '&refresh=true');
    tabDataLoaded.campaigns = null;
    await loadCampaignsTab();
  } catch (e) {
    console.error('Refresh broadcast list failed:', e);
    const toast = document.getElementById('campToast');
    toast.textContent = 'Refresh failed: ' + e.message;
    toast.classList.add('camp-toast--visible');
    setTimeout(() => toast.classList.remove('camp-toast--visible'), 2000);
  }
}

async function campDownloadCSV(engId, segId) {
  try {
    const res = await fetch(apiUrl + '/api/boss-view/campaign-broadcast-list?engagement=' + engId + '&segment=' + segId, {
      headers: { 'Authorization': 'Bearer ' + apiToken }
    });
    if (!res.ok) throw new Error('Download failed: ' + res.status);
    const blob = await res.blob();
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = segId + '-' + engId + '.csv';
    document.body.appendChild(a); a.click(); document.body.removeChild(a);
    URL.revokeObjectURL(url);
  } catch (e) {
    console.error('CSV download failed:', e);
    const toast = document.getElementById('campToast');
    toast.textContent = 'Download failed: ' + e.message;
    toast.classList.add('camp-toast--visible');
    setTimeout(() => toast.classList.remove('camp-toast--visible'), 2000);
  }
}
```

- [ ] **Step 3: Add the main playbook renderer:**

```javascript
function renderPlaybook(schedule, isArchived) {
  if (!schedule || schedule.length === 0) return '';

  const now = new Date();
  const todayStr = new Date(now.toLocaleString('en-US', { timeZone: 'Asia/Kuala_Lumpur' })).toISOString().slice(0, 10);

  // Split into tracks
  const repeatEngs = schedule.filter(e => (e.track || 'repeat') === 'repeat');
  const newEngs = schedule.filter(e => (e.track || 'repeat') === 'new');

  // Collect all unique dates, sorted
  const allDates = [...new Set(schedule.map(e => e.planned_date).filter(Boolean))].sort();

  // Find next-up for each track
  const findNext = (engs) => engs.find(e => e.status !== 'completed' && e.planned_date >= todayStr);
  const nextRepeat = isArchived ? null : findNext(repeatEngs);
  const nextNew = isArchived ? null : findNext(newEngs);

  // Group by date and track
  const byDate = {};
  for (const date of allDates) {
    byDate[date] = {
      repeat: repeatEngs.filter(e => e.planned_date === date),
      new: newEngs.filter(e => e.planned_date === date)
    };
  }

  // Determine today line position
  let todayInserted = false;

  // Build rows
  let rowsHtml = '';
  for (let i = 0; i < allDates.length; i++) {
    const date = allDates[i];
    const nextDate = allDates[i + 1];

    // Check if today line goes before this date
    if (!todayInserted && !isArchived && date > todayStr) {
      rowsHtml += `<div class="camp-today-line"><div class="camp-today-line__bar"></div><span class="camp-today-line__label">Today · ${formatShortDate(todayStr)}</span><div class="camp-today-line__bar"></div></div>`;
      todayInserted = true;
    }

    const repeatCards = byDate[date].repeat;
    const newCards = byDate[date].new;

    // Determine line colors based on status
    const hasMore = i < allDates.length - 1;
    const rLineColor = repeatCards.length > 0 && repeatCards[0].status === 'completed' ? 'camp-timeline-line--green' : 'camp-timeline-line--gray';
    const nLineColor = newCards.length > 0 && newCards[0].status === 'completed' ? 'camp-timeline-line--blue' : 'camp-timeline-line--gray';

    const renderCol = (cards, track, lineColor, nextEng) => {
      if (cards.length === 0) {
        return `<div class="camp-timeline-col" style="min-height:40px;">${hasMore ? `<div class="camp-timeline-line camp-timeline-line--gray"></div>` : ''}</div>`;
      }
      let colHtml = `<div class="camp-timeline-col">${hasMore ? `<div class="camp-timeline-line ${lineColor}"></div>` : ''}`;
      for (const eng of cards) {
        const isNext = eng === nextEng;
        const isDone = eng.status === 'completed';
        let nodeCls = 'camp-timeline-node ';
        let nodeStyle = '';
        if (isDone) {
          nodeCls += track === 'repeat' ? 'camp-timeline-node--done-repeat' : 'camp-timeline-node--done-new';
        } else if (isNext) {
          nodeCls += 'camp-timeline-node--next';
          const col = track === 'repeat' ? 'var(--accent)' : 'var(--orange)';
          nodeStyle = `background:${col}; box-shadow:0 0 0 2px ${col};`;
        } else {
          nodeCls += 'camp-timeline-node--future';
        }
        colHtml += `<div class="${nodeCls}" style="${nodeStyle}"></div>`;

        // Store caption state
        if (eng.captions) {
          const segs = Object.keys(eng.captions);
          captionState[eng.id] = { segment: segs[0], captions: eng.captions };
        }

        colHtml += renderEngCard(eng, isNext, isArchived);
      }
      colHtml += '</div>';
      return colHtml;
    };

    rowsHtml += `<div class="camp-timeline-row">
      ${renderCol(repeatCards, 'repeat', rLineColor, nextRepeat)}
      ${renderCol(newCards, 'new', nLineColor, nextNew)}
    </div>`;

    // Today line between this date and next
    if (!todayInserted && !isArchived && nextDate && todayStr >= date && todayStr < nextDate) {
      rowsHtml += `<div class="camp-today-line"><div class="camp-today-line__bar"></div><span class="camp-today-line__label">Today · ${formatShortDate(todayStr)}</span><div class="camp-today-line__bar"></div></div>`;
      todayInserted = true;
    }
  }

  // Today line after all dates
  if (!todayInserted && !isArchived && todayStr >= allDates[allDates.length - 1]) {
    rowsHtml += `<div class="camp-today-line"><div class="camp-today-line__bar"></div><span class="camp-today-line__label">Today · ${formatShortDate(todayStr)}</span><div class="camp-today-line__bar"></div></div>`;
  }

  return `<div class="camp-playbook">
    <div style="font-size:14px; font-weight:600; margin-bottom:16px;">Engagement Playbook</div>
    <div class="camp-track-headers">
      <div class="camp-track-header camp-track-header--repeat">
        <div class="camp-track-dot" style="background:var(--green);"></div>
        <div style="font-size:12px; font-weight:600; color:var(--green);">Repeat Customers</div>
        <div style="font-size:10px; color:var(--text-3); margin-left:auto;">WhatsApp</div>
      </div>
      <div class="camp-track-header camp-track-header--new">
        <div class="camp-track-dot" style="background:var(--accent);"></div>
        <div style="font-size:12px; font-weight:600; color:var(--accent);">New Customers</div>
        <div style="font-size:10px; color:var(--text-3); margin-left:auto;">Messenger</div>
      </div>
    </div>
    ${rowsHtml}
  </div>`;
}

function formatShortDate(dateStr) {
  const d = new Date(dateStr + 'T00:00:00+08:00');
  const months = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];
  return months[d.getMonth()] + ' ' + d.getDate();
}
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add engagement playbook renderer with dual-track timeline"
```

---

### Task 6: Rewrite loadCampaignsTab()

**Files:**
- Modify: `index.html` — replace the existing `loadCampaignsTab()` function (lines ~2712-2786)

- [ ] **Step 1: Replace `loadCampaignsTab()` with the new implementation** that renders all 4 levels:

```javascript
async function loadCampaignsTab() {
  const loading = document.getElementById('campaignsLoading');
  const empty = document.getElementById('campaigns-empty');
  const content = document.getElementById('campaigns-content');

  loading.style.display = 'block';
  loading.textContent = 'Loading campaign data';
  empty.style.display = 'none';
  content.style.display = 'none';

  // Try to load campaign list for selector
  let campaignList = [];
  const listData = await apiSafe('/campaign-list');
  if (listData && Array.isArray(listData) && listData.length > 0) {
    campaignList = listData;
  }

  // Load campaign state
  const slug = document.getElementById('campSelector')?.value || '';
  const endpoint = slug ? `/campaign-state?slug=${slug}` : '/campaign-state';
  const data = await apiSafe(endpoint);

  loading.style.display = 'none';

  if (!data || (!data.active && !slug) || !data.campaign) {
    empty.style.display = 'block';
    content.style.display = 'none';
    tabDataLoaded.campaigns = selectedMonth;
    return;
  }

  empty.style.display = 'none';
  content.style.display = 'block';

  // data.campaign is the full campaign state object containing:
  // .campaign (identity), .dates, .mechanics, .performance,
  // .engagement_schedule, .segments, .setup_tasks, .tenant_config, .internal_packages
  const c = data.campaign;
  const isArchived = !!(slug && data.active === false);

  // Reset caption state
  captionState = {};

  // Campaign selector
  let selectorHtml = '';
  if (campaignList.length > 1) {
    const opts = campaignList.map(cl => {
      const label = `${cl.name} — ${cl.display_name} (${cl.status === 'active' ? 'Active' : 'Archived'})`;
      const selected = cl.slug === (slug || campaignList.find(x => x.status === 'active')?.slug) ? ' selected' : '';
      return `<option value="${cl.slug}"${selected}>${label}</option>`;
    }).join('');
    selectorHtml = `<div style="margin-bottom:20px;"><select class="camp-selector" id="campSelector" onchange="tabDataLoaded.campaigns=null; loadCampaignsTab();">${opts}</select></div>`;
  }

  // Level 1: Campaign Overview
  // c.campaign holds identity (name, display_name, promo_code)
  // c.mechanics holds the campaign mechanics
  // c.dates holds orders_open, orders_close
  const camp = c.campaign || {};
  const dates = c.dates || {};
  const mech = c.mechanics || {};
  const startDate = dates.orders_open || '';
  const endDate = dates.orders_close || '';

  const startChip = startDate ? `<div class="camp-date-chip camp-date-chip--start"><div class="camp-date-chip__label" style="color:var(--accent);">Start</div><div class="camp-date-chip__value">${formatShortDate(startDate)}</div></div>` : '';
  const endChip = endDate ? `<div class="camp-date-chip camp-date-chip--end"><div class="camp-date-chip__label" style="color:var(--red);">End</div><div class="camp-date-chip__value">${formatShortDate(endDate)}</div></div>` : '';

  // Mechanics
  const mechLines = [];
  if (mech.type || mech.tiers) {
    const tier = mech.tiers?.[0];
    mechLines.push(`<strong>Type:</strong> ${mech.type || 'Free Gift'}${tier ? ` — ${tier.threshold} → ${tier.reward}` : ''}`);
  }
  if (mech.bonus) mechLines.push(`<strong>Bonus:</strong> ${mech.bonus}`);
  const products = (c.tenant_config?.product_focus || []).join(', ');
  if (products) mechLines.push(`<strong>Products:</strong> ${products}${mech.mix_match ? ' (mix & match OK)' : ''}`);
  const packages = (c.internal_packages || []).map(p => p.name).join(', ');
  if (packages) mechLines.push(`<strong>Packages:</strong> ${packages}`);

  const level1 = `<div class="camp-grid-2">
    <div class="camp-card">
      <div style="display:grid; grid-template-columns:1fr auto; gap:16px;">
        <div>
          <div style="font-size:20px; font-weight:600;">${camp.name || '—'}</div>
          <div style="font-size:12px; color:var(--text-2); margin-top:2px;">${camp.display_name || ''}</div>
          <div style="margin-top:10px;"><span style="font-size:11px; font-family:var(--mono); background:var(--border-light); padding:2px 8px; border-radius:4px; color:var(--text-2);">${camp.promo_code || ''}</span></div>
        </div>
        <div style="display:flex; flex-direction:column; align-items:flex-end; gap:6px;">
          ${startChip}${endChip}
        </div>
      </div>
    </div>
    <div class="camp-card">
      <div style="font-size:13px; font-weight:600; margin-bottom:8px;">Campaign Mechanics</div>
      <div style="font-size:12px; color:var(--text-2); line-height:1.7;">
        ${mechLines.map(l => '• ' + l).join('<br>')}
      </div>
    </div>
  </div>`;

  // Level 2: Dot Calendar + KPI + Segments
  const perf = c.performance || {};
  const totalPV = perf.promo_pv || 0;
  const totalOrders = perf.promo_orders || 0;
  const newPV = perf.new_pv || 0;
  const newOrders = perf.new_orders || 0;
  const repeatPV = perf.repeat_pv || 0;
  const repeatOrders = perf.repeat_orders || 0;
  const avgPV = totalOrders > 0 ? Math.round(totalPV / totalOrders) : 0;
  const newPct = totalPV > 0 ? Math.round(newPV / totalPV * 100) : 0;
  const repeatPct = totalPV > 0 ? Math.round(repeatPV / totalPV * 100) : 0;

  const dotCalHtml = startDate && endDate ? renderDotCalendar(startDate, endDate) : '';

  const level2 = `<div class="camp-grid-3">
    ${dotCalHtml}
    <div class="camp-kpi">
      <div class="camp-kpi__top">
        <div class="camp-kpi__label">Total PV</div>
        <div class="camp-kpi__value camp-kpi__value--lg">${fmt(totalPV)}</div>
        <div class="camp-kpi__sub">${fmt(totalOrders)} orders · avg ${fmt(avgPV)} PV</div>
      </div>
      <div class="camp-kpi__split">
        <div class="camp-kpi__cell">
          <div class="camp-kpi__label">New PV</div>
          <div class="camp-kpi__value" style="color:var(--accent);">${fmt(newPV)}</div>
          <div class="camp-kpi__sub">${newPct}% · ${fmt(newOrders)} orders</div>
        </div>
        <div class="camp-kpi__cell">
          <div class="camp-kpi__label">Repeat PV</div>
          <div class="camp-kpi__value" style="color:var(--green);">${fmt(repeatPV)}</div>
          <div class="camp-kpi__sub">${repeatPct}% · ${fmt(repeatOrders)} orders</div>
        </div>
      </div>
    </div>
    ${buildSegmentTable(c.segments)}
  </div>`;

  // Level 3: Setup Tasks
  const level3 = renderSetupTasks(c.setup_tasks);

  // Level 4: Engagement Playbook
  const level4 = renderPlaybook(c.engagement_schedule, isArchived);

  // Assemble
  content.innerHTML = selectorHtml + level1 + level2 + level3 + level4;
  tabDataLoaded.campaigns = selectedMonth;
}
```

- [ ] **Step 2: Verify** that the existing `loadTabData()` function still calls `loadCampaignsTab()` correctly (it should — the function signature hasn't changed).

- [ ] **Step 3: Also remove the old `formatTimelineDate()` function** (around line 2788) since it's replaced by `formatShortDate()`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: rewrite loadCampaignsTab with full campaign workbench rendering"
```

---

### Task 7: Visual Verification & Polish

**Files:**
- Modify: `index.html` — bug fixes from visual testing

- [ ] **Step 1: Create a mock campaign state JSON** for testing. Save to a temporary file or use the browser console to verify. The mock should exercise all states: completed engagements, next-up with captions, future collapsed, both tracks, setup tasks with mixed statuses.

- [ ] **Step 2: Start local server and test:**

```bash
cd /Users/bryanckl/Documents/minionions-dashboard
python3 -m http.server 3333
```

Open `http://localhost:3333` and navigate to the Campaigns tab.

- [ ] **Step 3: Verify each level renders correctly:**
- Level 1: Campaign name, subtitle, promo code, date chips, mechanics
- Level 2: Dot calendar (correct elapsed/remaining/today), KPIs, segments table with tooltips
- Level 3: Setup tasks sorted (pending first, done at bottom), subtasks visible
- Level 4: Dual tracks aligned, today line spans full width, next-up expanded, future collapsed, click-to-expand works

- [ ] **Step 4: Test interactive elements:**
- Copy buttons (captions, image prompt) — verify toast appears
- Caption variation tabs — switch between 1-5
- Caption segment switcher — dropdown changes content
- Future card click — expands/collapses

- [ ] **Step 5: Test edge cases:**
- Missing captions → "Captions pending" placeholder
- Missing reasoning → strategy block hidden
- Missing broadcast lists → "Lists not generated yet"

- [ ] **Step 6: Fix any CSS alignment, spacing, or rendering issues found**

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "fix: polish campaigns tab rendering after visual verification"
```

---

### Task 8: Final Cleanup

- [ ] **Step 1: Remove any dead code** — the old `formatTimelineDate()` function, any unused CSS classes from the old campaigns tab.

- [ ] **Step 2: Verify no regressions** on other tabs (Lifetime, Monthly, Ads) — just quick visual check that switching tabs still works.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "chore: remove dead campaigns tab code"
```
