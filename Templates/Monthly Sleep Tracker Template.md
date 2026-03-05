---
style:
  - default
Month: 3
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

// --- Determine selected month ---
const now = new Date();
const year = now.getFullYear();
const monthNames = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];

const rawMonth = dv.current().Month;
const monthInput = Number(Array.isArray(rawMonth) ? rawMonth[0] : rawMonth);
const month = (monthInput >= 1 && monthInput <= 12) ? monthInput - 1 : now.getMonth();

const mName = monthNames[month];
const monthNum = String(month + 1).padStart(2, "0");
const yearStr = String(year);

// --- Query and filter pages to selected month ---
const pages = dv.pages('"Logs"')
  .where(p => {
    const m = p.file.name.match(/Sleep - (\d{4})-(\d{2})-(\d{2})/);
    if (!m) return false;
    return m[1] === yearStr && m[2] === monthNum;
  })
  .sort(p => p.file.name, "asc")
  .array();

if (pages.length === 0) {
  dv.el("p", `**${mName} ${year}** — No sleep logs for this month.`);
} else {

// --- Build chart data ---
const data = pages.map(p => {
  const m = p.file.name.match(/Sleep - \d{4}-\d{2}-(\d{2})/);
  return { day: parseInt(m[1]), sleep: Number(p.sleep_hours) || 0 };
});

const sleepVals = data.map(d => d.sleep);
const avg = sleepVals.reduce((a, b) => a + b, 0) / sleepVals.length;
const daysAtGoal = sleepVals.filter(v => v >= 8).length;

const W = 800, H = 300;
const pad = { top: 30, right: 20, bottom: 40, left: 44 };
const cw = W - pad.left - pad.right;
const ch = H - pad.top - pad.bottom;
const yMin = 4, yMax = 10, goalVal = 8;
const n = data.length;

const px = (i) => pad.left + (n === 1 ? cw / 2 : (i / (n - 1)) * cw);
const py = (v) => pad.top + ch - ((Math.min(Math.max(v, yMin), yMax) - yMin) / (yMax - yMin)) * ch;

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
svg += `<path d="${line}" fill="none" stroke="${t.line}" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>`;

// Dots
for (let i = 0; i < n; i++) {
  svg += `<circle cx="${px(i)}" cy="${py(data[i].sleep)}" r="3" fill="${t.dot}" stroke="var(--background-primary)" stroke-width="1.5"/>`;
}

// X-axis day numbers
const step = n > 20 ? 2 : 1;
for (let i = 0; i < n; i += step) {
  svg += `<text x="${px(i)}" y="${H - pad.bottom + 18}" fill="${t.text}" font-size="10" text-anchor="middle">${data[i].day}</text>`;
}

svg += `</svg>`;
const el = dv.container.createEl("div");
el.innerHTML = svg;

// --- Minimal stats ---
const stats = dv.container.createEl("div");
stats.style.cssText = "font-family:var(--font-monospace); font-size:11px; color:var(--text-muted); padding:6px 0 0; opacity:0.7;";
stats.textContent = `avg ${avg.toFixed(1)}h · ${daysAtGoal}/${n} days at goal`;

}
```
