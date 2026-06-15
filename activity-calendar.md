```dataviewjs
const pages = dv.pages('""');
const today = new Date();
const endDate = new Date(today);
endDate.setHours(23,59,59,999);

const startDate = new Date(endDate);
startDate.setDate(startDate.getDate() - 364);
const dow0 = startDate.getDay();
const offset = dow0 === 0 ? 6 : dow0 - 1;
startDate.setDate(startDate.getDate() - offset);

const COLORS = ["#161b22","#0e4429","#006d32","#26a641","#39d353"];
const THRESHOLDS = [0,1,3,6,10];
const DAY_NAMES = ["Mon","","Wed","","Fri","","Sun"];
const MONTH_NAMES = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"];

let currentMode = "both";

const createdCountByDate = {};
const createdByDate = {};
const modifiedCountByDate = {};
const modifiedByDate = {};

pages.forEach(p => {
  const cd = p.file.cday;
  if (cd) {
    const key = `${cd.year}-${String(cd.month).padStart(2,'0')}-${String(cd.day).padStart(2,'0')}`;
    createdCountByDate[key] = (createdCountByDate[key] || 0) + 1;
    if (!createdByDate[key]) createdByDate[key] = [];
    createdByDate[key].push({ name: p.file.name, type: "created" });
  }
  const md = p.file.mday;
  if (md) {
    const key = `${md.year}-${String(md.month).padStart(2,'0')}-${String(md.day).padStart(2,'0')}`;
    const cdKey = cd ? `${cd.year}-${String(cd.month).padStart(2,'0')}-${String(cd.day).padStart(2,'0')}` : null;
    if (key !== cdKey) {
      modifiedCountByDate[key] = (modifiedCountByDate[key] || 0) + 1;
      if (!modifiedByDate[key]) modifiedByDate[key] = [];
      modifiedByDate[key].push({ name: p.file.name, type: "modified" });
    }
  }
});

function getCountByDate(mode) {
  if (mode === "created") return createdCountByDate;
  if (mode === "modified") return modifiedCountByDate;
  const merged = {};
  const allKeys = new Set([...Object.keys(createdCountByDate), ...Object.keys(modifiedCountByDate)]);
  allKeys.forEach(k => {
    merged[k] = (createdCountByDate[k] || 0) + (modifiedCountByDate[k] || 0);
  });
  return merged;
}

function getNotesByDate(mode, key) {
  if (mode === "created") return createdByDate[key] || [];
  if (mode === "modified") return modifiedByDate[key] || [];
  return [...(createdByDate[key] || []), ...(modifiedByDate[key] || [])];
}

function getColor(count) {
  if (count === 0) return COLORS[0];
  for (let i = THRESHOLDS.length - 1; i >= 1; i--)
    if (count >= THRESHOLDS[i]) return COLORS[i];
  return COLORS[1];
}

const msPerDay = 86400000;
const totalDays = Math.round((endDate - startDate) / msPerDay) + 1;
const totalWeeks = Math.ceil(totalDays / 7);

function buildGrid(mode) {
  const notesCountByDate = getCountByDate(mode);
  const grid = Array.from({length: 7}, () => Array(totalWeeks).fill(null));
  const monthPositions = {};
  for (let i = 0; i < totalDays; i++) {
    const cur = new Date(startDate.getTime() + i * msPerDay);
    const weekIdx = Math.floor(i / 7);
    const dayIdx = i % 7;
    const key = cur.toISOString().slice(0, 10);
    const month = cur.getMonth();
    if (cur.getDate() <= 7 && !monthPositions.hasOwnProperty(month))
      monthPositions[month] = weekIdx;
    grid[dayIdx][weekIdx] = { date: key, count: notesCountByDate[key] || 0 };
  }
  return { grid, monthPositions };
}

function calcStats(mode) {
  const notesCountByDate = getCountByDate(mode);
  let total = 0, active = 0;
  Object.values(notesCountByDate).forEach(v => { total += v; active++; });
  return { total, active };
}

const containerWidth = this.container.clientWidth || 700;
const labelWidth = 36;
const daySize = Math.floor((containerWidth - labelWidth) / totalWeeks) - 2;
const cellSize = Math.max(9, Math.min(daySize, 15));
const cellStep = cellSize + 2;

const startYear = startDate.getFullYear();
const endYear = endDate.getFullYear();
const containerId = "cal-" + Date.now();
const activityId = "activity-" + Date.now();
const statsId = "stats-" + Date.now();
const gridId = "grid-" + Date.now();
const monthsId = "months-" + Date.now();

function renderMonths(monthPositions) {
  let out = '';
  Object.entries(monthPositions).forEach(([m, w]) => {
    out += `<span style="position:absolute;left:${w*cellStep}px;font-size:11px;color:var(--text-muted)">${MONTH_NAMES[parseInt(m)]}</span>`;
  });
  return out;
}

function renderGrid(grid, mode) {
  let out = '';
  for (let d = 0; d < 7; d++) {
    out += `<div style="display:flex;align-items:center;margin-bottom:1px;">`;
    out += `<span style="width:${labelWidth}px;font-size:11px;color:var(--text-muted);text-align:right;padding-right:4px;flex-shrink:0">${DAY_NAMES[d]}</span>`;
    for (let w = 0; w < totalWeeks; w++) {
      const cell = grid[d][w];
      if (!cell) {
        out += `<span style="width:${cellSize}px;height:${cellSize}px;margin:1px;display:inline-block;flex-shrink:0"></span>`;
        continue;
      }
      const color = getColor(cell.count);
      const border = cell.count === 0 ? 'border:1px solid #30363d;' : '';
      const dataAttr = cell.count > 0 ? `data-date="${cell.date}" data-count="${cell.count}"` : '';
      out += `<span ${dataAttr} title="${cell.date}: ${cell.count} notes" style="width:${cellSize}px;height:${cellSize}px;border-radius:3px;background:${color};${border}display:inline-block;margin:1px;flex-shrink:0;cursor:${cell.count > 0 ? 'pointer' : 'default'}"></span>`;
    }
    out += `</div>`;
  }
  return out;
}

function renderActivity(dateKey, formatted, notes) {
  let out = `<div style="margin-bottom:8px;">`;
  out += `<span style="font-size:13px;font-weight:500;color:var(--text-normal)">Activity</span>`;
  out += `<span style="font-size:12px;color:var(--text-muted);margin-left:10px">${formatted}</span>`;
  out += `</div>`;
  out += `<div style="border:1px solid var(--background-modifier-border);border-radius:6px;overflow:hidden;">`;
  notes.forEach((note, i) => {
    const border = i > 0 ? 'border-top:1px solid var(--background-modifier-border);' : '';
    const icon = note.type === "created" ? "📝" : "✏️";
    const label = note.type === "created" ? "created" : "modified";
    out += `<div style="${border}padding:8px 12px;display:flex;align-items:center;gap:8px;">`;
    out += `<span style="font-size:18px;line-height:1">${icon}</span>`;
    out += `<div>`;
    out += `<a class="internal-link" href="${note.name}" style="font-size:13px;color:var(--text-accent);text-decoration:none;">${note.name}</a>`;
    out += `<span style="font-size:12px;color:var(--text-muted);margin-left:8px">${label} ${dateKey}</span>`;
    out += `</div></div>`;
  });
  out += `</div>`;
  return out;
}

const { grid: initGrid, monthPositions: initMonths } = buildGrid(currentMode);
const { total: initTotal, active: initActive } = calcStats(currentMode);

const sortedAllDates = [
  ...Object.keys(createdCountByDate),
  ...Object.keys(modifiedCountByDate)
].sort().reverse();
const lastDate = sortedAllDates[0] || null;

let html = `<div style="font-family:var(--font-interface);padding:8px 0;">`;

html += `<div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:8px;flex-wrap:wrap;gap:6px;">`;
html += `<span style="font-weight:500;color:var(--text-normal)">${startYear}–${endYear} · activity</span>`;
html += `<div style="display:flex;gap:4px;">`;
for (const [id, label] of [["created","Created"],["modified","Modified"],["both","Both"]]) {
  const active = currentMode === id;
  html += `<button data-mode="${id}" style="font-size:12px;padding:3px 10px;border-radius:4px;border:1px solid var(--background-modifier-border);background:${active ? 'var(--interactive-accent)' : 'var(--background-secondary)'};color:${active ? '#fff' : 'var(--text-normal)'};cursor:pointer;">${label}</button>`;
}
html += `</div>`;
html += `</div>`;

html += `<div id="${statsId}" style="font-size:13px;color:var(--text-muted);margin-bottom:6px;">${initTotal} notes · ${initActive} active days</div>`;

html += `<div id="${monthsId}" style="position:relative;height:18px;margin-left:${labelWidth}px;margin-bottom:2px;">${renderMonths(initMonths)}</div>`;

html += `<div id="${gridId}">${renderGrid(initGrid, currentMode)}</div>`;

html += `<div style="display:flex;align-items:center;gap:3px;margin-top:8px;justify-content:flex-end;font-size:11px;color:var(--text-muted)">`;
html += `<span>Less</span>`;
COLORS.forEach(c => {
  const border = c === "#161b22" ? 'border:1px solid #30363d;' : '';
  html += `<span style="width:${cellSize}px;height:${cellSize}px;border-radius:3px;background:${c};${border}display:inline-block"></span>`;
});
html += `<span>More</span></div>`;

html += `<div id="${activityId}" style="margin-top:20px;border-top:1px solid var(--background-modifier-border);padding-top:16px;">`;
if (lastDate) {
  const d = new Date(lastDate);
  const formatted = d.toLocaleDateString("en-US", { month: "long", day: "numeric", year: "numeric" });
  const notes = getNotesByDate(currentMode, lastDate);
  html += renderActivity(lastDate, formatted, notes);
} else {
  html += `<p style="color:var(--text-muted);font-size:13px">No activity yet</p>`;
}
html += `</div>`;
html += `</div>`;

const el = dv.el("div", html);

setTimeout(() => {
  el.querySelectorAll("button[data-mode]").forEach(btn => {
    btn.addEventListener("click", () => {
      currentMode = btn.getAttribute("data-mode");

      el.querySelectorAll("button[data-mode]").forEach(b => {
        const active = b.getAttribute("data-mode") === currentMode;
        b.style.background = active ? 'var(--interactive-accent)' : 'var(--background-secondary)';
        b.style.color = active ? '#fff' : 'var(--text-normal)';
      });

      const { grid, monthPositions } = buildGrid(currentMode);
      const { total, active } = calcStats(currentMode);

      const statsEl = document.getElementById(statsId);
      if (statsEl) statsEl.textContent = `${total} notes · ${active} active days`;

      const monthsEl = document.getElementById(monthsId);
      if (monthsEl) monthsEl.innerHTML = renderMonths(monthPositions);

      const gridEl = document.getElementById(gridId);
      if (gridEl) {
        gridEl.innerHTML = renderGrid(grid, currentMode);
        attachCellListeners(gridEl);
      }
      const notesCountByDate = getCountByDate(currentMode);
      const sorted = Object.keys(notesCountByDate).sort().reverse();
      const actEl = document.getElementById(activityId);
      if (actEl && sorted.length > 0) {
        const dateKey = sorted[0];
        const d = new Date(dateKey);
        const formatted = d.toLocaleDateString("en-US", { month: "long", day: "numeric", year: "numeric" });
        const notes = getNotesByDate(currentMode, dateKey);
        actEl.innerHTML = renderActivity(dateKey, formatted, notes);
      }
    });
  });

  function attachCellListeners(container) {
    container.querySelectorAll("[data-date]").forEach(span => {
      span.addEventListener("click", () => {
        const dateKey = span.getAttribute("data-date");
        const notes = getNotesByDate(currentMode, dateKey);
        const d = new Date(dateKey);
        const formatted = d.toLocaleDateString("en-US", { month: "long", day: "numeric", year: "numeric" });
        const actEl = document.getElementById(activityId);
        if (actEl) actEl.innerHTML = renderActivity(dateKey, formatted, notes);
      });
      span.addEventListener("mouseenter", () => span.style.outline = "1px solid #58a6ff");
      span.addEventListener("mouseleave", () => span.style.outline = "none");
    });
  }

  const gridEl = document.getElementById(gridId);
  if (gridEl) attachCellListeners(gridEl);
}, 300);
```
