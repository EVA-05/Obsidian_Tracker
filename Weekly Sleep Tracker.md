---
style:
  - default
Week: 4
---

```dataviewjs
const themes = {
  default: { line: "var(--text-normal)", goal: "var(--text-accent)", bg: "var(--background-secondary)", dot: "var(--text-normal)", grid: "var(--background-modifier-border)", text: "var(--text-muted)" },
  red:     { line: "#f7768e", goal: "#ff9e9e", bg: "rgba(247,118,142,0.12)", dot: "#f7768e", grid: "rgba(247,118,142,0.15)", text: "#f7768e" },
  blue:    { line: "#7aa2f7", goal: "#a9c1f7", bg: "rgba(122,162,247,0.12)", dot: "#7aa2f7", grid: "rgba(122,162,247,0.15)", text: "#7aa2f7" },
  green:   { line: "#9ece6a", goal: "#b9de8a", bg: "rgba(158,206,106,0.12)", dot: "#9ece6a", grid: "rgba(158,206,106,0.15)", text: "#9ece6a" },
  purple:  { line: "#9d7cd8", goal: "#bb9ef7", bg: "rgba(157,124,216,0.12)", dot: "#9d7cd8", grid: "rgba(157,124,216,0.15)", text: "#9d7cd8" },
};

const rawStyle = dv.current().style;
const themeName = Array.isArray(rawStyle) ? rawStyle[0] : (rawStyle || "default");
const t = themes[themeName] || themes["default"];

// --- Week boundaries for current month (Monday-aligned) ---
const now = new Date();
const year = now.getFullYear();
const month = now.getMonth();
const daysInMonth = new Date(year, month + 1, 0).getDate();
const firstDow = new Date(year, month, 1).getDay(); // 0=Sun

const weeks = [];
let dayPtr = 1;

// Week 1: from 1st to first Sunday (partial if month doesn't start Monday)
if (firstDow !== 1) {
  const daysUntilSun = firstDow === 0 ? 0 : (7 - firstDow);
  weeks.push({ start: 1, end: 1 + daysUntilSun });
  dayPtr = 2 + daysUntilSun;
}

// Weeks 2+ (Mon-Sun), cap at 5 total
while (dayPtr <= daysInMonth) {
  if (weeks.length >= 4) {
    weeks.push({ start: dayPtr, end: daysInMonth });
    break;
  }
  weeks.push({ start: dayPtr, end: Math.min(dayPtr + 6, daysInMonth) });
  dayPtr += 7;
}

// --- Determine selected week ---
const rawWeek = dv.current().Week;
const weekNum = Number(Array.isArray(rawWeek) ? rawWeek[0] : rawWeek);
let sel;

if (weekNum >= 1 && weekNum <= weeks.length) {
  sel = weekNum - 1;
} else {
  // Default: find which week contains today
  const today = now.getDate();
  sel = 0;
  for (let i = 0; i < weeks.length; i++) {
    if (today >= weeks[i].start && today <= weeks[i].end) {
      sel = i;
      break;
    }
  }
}

const wk = weeks[sel];
const monthNames = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];
const mName = monthNames[month];
const monthNum = String(month + 1).padStart(2, "0");
const yearStr = String(year);

// --- Query and filter pages to selected week + current month ---
const pages = dv.pages('"Logs"')
  .where(p => {
    const m = p.file.name.match(/Sleep - (\d{4})-(\d{2})-(\d{2})/);
    if (!m) return false;
    if (m[1] !== yearStr || m[2] !== monthNum) return false;
    const day = parseInt(m[3]);
    return day >= wk.start && day <= wk.end;
  })
  .sort(p => p.file.name, "asc")
  .array();

// --- Header ---
dv.el("p", `**${mName} ${year}** — Week ${sel + 1} (${mName} ${wk.start}–${wk.end})`);

if (pages.length === 0) {
  dv.el("p", "No sleep logs for this week.");
} else {

// --- Build chart data ---
const data = pages.map(p => {
  const m = p.file.name.match(/Sleep - \d{4}-\d{2}-(\d{2})/);
  return {
    label: `${mName} ${parseInt(m[1])}`,
    sleep: Number(p.sleep_hours) || 0,
  };
});

const W = 600, H = 280;
const pad = { top: 30, right: 50, bottom: 40, left: 50 };
const cw = W - pad.left - pad.right;
const ch = H - pad.top - pad.bottom;
const yMin = 4, yMax = 12, goalVal = 8;
const n = data.length;

const px = (i) => pad.left + (n === 1 ? cw / 2 : (i / (n - 1)) * cw);
const py = (v) => pad.top + ch - ((v - yMin) / (yMax - yMin)) * ch;

let svg = `<svg width="100%" viewBox="0 0 ${W} ${H}" xmlns="http://www.w3.org/2000/svg" style="font-family:var(--font-monospace);">`;

// Y grid + labels
for (let v = yMin; v <= yMax; v += 2) {
  svg += `<line x1="${pad.left}" y1="${py(v)}" x2="${W - pad.right}" y2="${py(v)}" stroke="${t.grid}" stroke-width="1"/>`;
  svg += `<text x="${pad.left - 8}" y="${py(v) + 4}" fill="${t.text}" font-size="11" text-anchor="end">${v}h</text>`;
}

// Goal line
svg += `<line x1="${pad.left}" y1="${py(goalVal)}" x2="${W - pad.right}" y2="${py(goalVal)}" stroke="${t.goal}" stroke-width="1.5" stroke-dasharray="6,4"/>`;

// Area fill
let area = `M ${px(0)} ${py(data[0].sleep)}`;
for (let i = 1; i < n; i++) area += ` L ${px(i)} ${py(data[i].sleep)}`;
area += ` L ${px(n - 1)} ${py(yMin)} L ${px(0)} ${py(yMin)} Z`;
svg += `<path d="${area}" fill="${t.bg}" opacity="0.5"/>`;

// Data line
let line = `M ${px(0)} ${py(data[0].sleep)}`;
for (let i = 1; i < n; i++) line += ` L ${px(i)} ${py(data[i].sleep)}`;
svg += `<path d="${line}" fill="none" stroke="${t.line}" stroke-width="2.5" stroke-linejoin="round" stroke-linecap="round"/>`;

// Dots + value labels (flip below dot when near goal line)
const goalPy = py(goalVal);
for (let i = 0; i < n; i++) {
  const cx = px(i), cy = py(data[i].sleep);
  svg += `<circle cx="${cx}" cy="${cy}" r="4" fill="${t.dot}" stroke="var(--background-primary)" stroke-width="2"/>`;
  const labelY = Math.abs(cy - goalPy) < 25 ? cy + 20 : cy - 10;
  svg += `<text x="${cx}" y="${labelY}" fill="${t.text}" font-size="10" text-anchor="middle">${data[i].sleep}h</text>`;
}

// X-axis labels
for (let i = 0; i < n; i++) {
  svg += `<text x="${px(i)}" y="${H - pad.bottom + 18}" fill="${t.text}" font-size="11" text-anchor="middle">${data[i].label}</text>`;
}

svg += `</svg>`;
const el = dv.container.createEl("div");
el.innerHTML = svg;

}
```
