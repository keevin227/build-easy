# Plan: Live demo upgrade — Room cost heatmap + Building strip

Audience: the model that will implement this (e.g. Sonnet 5.5). Follow it literally.
Every colour, size, string and behaviour below is decided — do not substitute your own.
If something here turns out to be impossible, stop and ask the user; do not improvise.

Reply to the user in **English**.

---

## 0. What we are building (the picture to hold in your head)

The live demo (`<section id="demo">`) shows one floor of a 5-storey building as an SVG plan.
Today every room is the same dark blue and the cost colour lives only on thin wall lines.

After this change:

1. **Room cost heatmap.** In the "Cost heat" lens, every room is a **solid tile coloured by its
   material cost per m²** on a 5-step single-hue orange scale (dark = cheap, bright = expensive in
   dark theme). Each room shows its name and one number (million toman / m²). Walls, windows,
   doors and fixtures turn **neutral grey** so colour means only one thing: room cost.
   The most expensive room on the floor carries a small "▲ Highest" pill.
   Below the plan, a **scale bar legend** with real numbers replaces the two-swatch legend.
2. **Building strip.** The floor tabs (`طبقهٔ ۱ … ۵`) are replaced by a **vertical elevation of
   the building** beside the plan: 5 stacked rows, floor 5 on top, floor 1 at the bottom. Each row
   has a horizontal bar whose **length = that floor's material cost**, plus the value. The selected
   floor is highlighted and drives the plan. The most expensive floor carries "▲".
   So the whole building is visible at a glance, and the plan is one click away.

ASCII sketch of the plan pane (dark theme, desktop, LTR shown; RTL mirrors automatically):

```
┌───────────────────────────────────────────────────────────────────────┐
│ [Cost heat] [Origin] [Price level]                         (lens seg)  │
│ ┌──────────────┐ ┌───────────────────────────────────────────────────┐ │
│ │ Building     │ │ ┌────────┬──────┬──────┬────────┐                  │ │
│ │ 12.4 bn      │ │ │Bedroom │Bedr. │Bedr. │Bedroom │  rooms = solid   │ │
│ │──────────────│ │ │  3.1   │ 3.4  │ 3.4  │  3.1   │  orange tiles    │ │
│ │5 ▇▇▇▇▇▇▇ 2.9▲│ │ ├──┬─────┴──┬───┴──┬───┴──┬─────┤                  │ │
│ │4 ▇▇▇▇▇   2.4 │◀│ │Ba│ Living │Living│  Ba  │      │                  │ │
│ │3 ▇▇▇▇▇   2.4 │ │ │8.9│ 2.7   │ 2.7  │ 8.9  │      │                  │ │
│ │2 ▇▇▇▇▇   2.4 │ │ └──┴────────┴──────┴──────┴─────┘                  │ │
│ │1 ▇▇▇▇▇   2.4 │ └───────────────────────────────────────────────────┘ │
│ └──────────────┘  Room material cost per m² (million toman)            │
│                   [▆][▆][▆][▆][▆]  1.5  2.9  5.5  10.6  20.4  39        │
│                   D door · E entrance · S sanitary · K kitchen · ⚡ elec. │
└───────────────────────────────────────────────────────────────────────┘
```

Numbers in the sketch are illustrative only; real values come from the engine.

**Out of scope (do not do):** replacing D/E/S/K letters with drawing symbols, isometric views,
A/B slider, changing the pricing engine (`CAT`, `ORIG`, `LEV`, `unit`, `cost`, `totals`),
changing any other section of the proposal.

---

## 1. Repository facts you need

- Repo: `keevin227/build-easy`. Work on the branch you are given (a `claude/...` branch).
  Pushing to any `claude/**` branch **auto-publishes**: `.github/workflows/publish.yml` merges it
  into `main`, GitHub Pages rebuilds, and the run then checks the live site serves the new
  `proposal.json`. Do **not** push to `main` yourself.
- `index.html` is only the password gate. **Do not edit it.**
- `proposal.json` = `{s, i, n, c}`: the whole proposal HTML encrypted with AES-256-GCM,
  key = PBKDF2-SHA256(password, salt `s`, `n` iterations), IV `i`, `c` = base64(ciphertext‖16-byte tag).
- The password is **not** in the repo. **Ask the user for it** before starting. Never write it
  into any file, commit message, or log.
- The decrypted proposal is one HTML file (~4.6 MB) containing: inline CSS (one `<style>`),
  the sections, a `<script>window.PDFS={"fa":"…","en":"…"}</script>` line holding two
  base64 PDFs (~3 MB — never print or open that line), and the app script.
- Themes: `html[data-theme=dark|light|read]`, set by `setTheme(t)`. Languages: `body.fa` / `body.en`,
  set by `setLang(l)`. Bilingual text is always `<span class="fa">…</span><span class="en">…</span>`.
  Persian digits via `faD()`, numbers via `fmt()` / `bil()` — reuse them.
- **Class collision to avoid:** `.plan` is used both by the demo SVG (`<svg class="plan" id="plan">`)
  and by the pricing cards (`.plan{position:relative;background:…;padding:26px}`). All new demo CSS
  must be scoped with `#demo` or `#plan`, never bare `.plan`, `.strip`, `.bar`, `.stg` (all taken).
  Use the new class prefix **`bs-`** (building strip) and **`hm-`** (heatmap) for everything new.
- Chromium for Playwright: `/opt/pw-browsers/chromium` (do not run `playwright install`).
  Use `NODE_PATH=$(npm root -g)` to require the globally installed `playwright`.

---

## 2. Tooling (create these in your scratchpad, NOT in the repo)

Set `S=<your scratchpad dir>` and `PW=<password from the user>` (env var only).

### 2.1 `dec.mjs` — decrypt `proposal.json` → `work.html`
```js
import fs from 'fs'; import crypto from 'crypto';
const D = JSON.parse(fs.readFileSync(process.argv[2], 'utf8'));
const key = crypto.pbkdf2Sync(process.env.PW, Buffer.from(D.s,'base64'), D.n, 32, 'sha256');
const b = Buffer.from(D.c,'base64');
const d = crypto.createDecipheriv('aes-256-gcm', key, Buffer.from(D.i,'base64'));
d.setAuthTag(b.subarray(-16));
fs.writeFileSync(process.argv[3], Buffer.concat([d.update(b.subarray(0,-16)), d.final()]));
```
Run: `PW=… node $S/dec.mjs /home/user/build-easy/proposal.json $S/work.html`

### 2.2 `enc.mjs` — encrypt `final.html` → `proposal.json` (fresh salt + IV every time)
```js
import fs from 'fs'; import crypto from 'crypto';
const pt = fs.readFileSync(process.argv[2]);
const s = crypto.randomBytes(16), i = crypto.randomBytes(12), n = 250000;
const key = crypto.pbkdf2Sync(process.env.PW, s, n, 32, 'sha256');
const c = crypto.createCipheriv('aes-256-gcm', key, i);
const ct = Buffer.concat([c.update(pt), c.final(), c.getAuthTag()]);
fs.writeFileSync(process.argv[3], JSON.stringify({s:s.toString('base64'), i:i.toString('base64'), n, c:ct.toString('base64')}));
```

### 2.3 `pdf.cjs` — regenerate the two embedded PDFs from the edited HTML
```js
const {chromium} = require('playwright'); const fs = require('fs');
(async () => {
  const b = await chromium.launch({executablePath: '/opt/pw-browsers/chromium'});
  const pg = await b.newPage({viewport: {width: 794, height: 1123}});
  await pg.goto('file://' + process.argv[2]); await pg.waitForTimeout(1500);
  for (const l of ['fa','en']) {
    await pg.evaluate(l => setLang(l), l); await pg.waitForTimeout(800);
    await pg.pdf({path: process.argv[3] + `/new_${l}.pdf`, format: 'A4', printBackground: true,
                  preferCSSPageSize: true, margin: {top:0,bottom:0,left:0,right:0}});
  }
  await b.close();
})();
```
Then embed (replaces the `window.PDFS={...}` object in place):
```js
// embed.cjs <in.html> <pdfdir> <out.html>
const fs=require('fs');let p=fs.readFileSync(process.argv[2],'utf8');const d=process.argv[3];
const fa=fs.readFileSync(d+'/new_fa.pdf').toString('base64'),en=fs.readFileSync(d+'/new_en.pdf').toString('base64');
const re=/window\.PDFS=\{"fa":"[^"]+","en":"[^"]+"\}/; if(!re.test(p)) throw new Error('PDFS not found');
p=p.replace(re,()=>`window.PDFS={"fa":"${fa}","en":"${en}"}`); fs.writeFileSync(process.argv[4],p);
```

### 2.4 Build order (every time you change `work.html`)
1. `node pdf.cjs $S/work.html $S` → `node embed.cjs $S/work.html $S $S/final.html`
2. `PW=… node enc.mjs $S/final.html /home/user/build-easy/proposal.json`
3. Verify round-trip: `PW=… node dec.mjs /home/user/build-easy/proposal.json $S/check.html && cmp $S/final.html $S/check.html`

Edit `work.html` with small Python/Node string replacements that **assert** the old text exists
exactly once before replacing. Never open the PDFS line in an editor.

---

## 3. Data changes (inside the app script)

### 3.1 Give room-owned elements a room id
In `buildFloor(f)`, change `rooms.forEach(r=>{` to `rooms.forEach((r,ri)=>{` and add `room:ri`
to **every** `add(f,{…})` call inside that loop (floor, wet, paint, sanitary, kitchen, elec).

After the rooms loop (still inside `buildFloor`), assign each window of this floor to the room whose
rectangle contains the window's midpoint (edges inclusive):
```js
 E.filter(e=>e.f===f&&e.c==='window').forEach(e=>{const[a,b,c,d]=e.seg,mx=(a+c)/2,my=(b+d)/2;
   const ri=rooms.findIndex(r=>mx>=r[0]&&mx<=r[0]+r[2]&&my>=r[1]&&my<=r[1]+r[3]);if(ri>=0)e.room=ri});
```
Walls, doors and entrance doors stay without `room` (shared elements). Each of the 8 windows maps
to exactly one room — assert this once in your test (section 7).

### 3.2 Penthouse top floor (so the building strip shows a real difference)
All 5 floors are geometrically identical, so without this the strip shows 5 equal bars.
Add, right after `for(let f=0;f<FLOORS;f++)buildFloor(f);`:
```js
// Sample building: the top floor is a penthouse with Expensive finishes.
const PENT=['floor','wet','sanitary','kitchen','window'];
function seedPenthouse(){for(const e of E)if(e.f===FLOORS-1&&PENT.includes(e.c))over[e.id]={o:catSet[e.c].o,l:'E'}}
seedPenthouse();
```
Existing behaviour stays: the global "Mix board" buttons and "Optimise" clear `over`, which
equalises the floors — that is fine and is visible feedback in the strip.

Update the demo lead paragraph (exact strings):
- fa: `یک ساختمان مسکونی نمونهٔ ۵ طبقه (هر طبقه دو واحد، طبقهٔ پنجم پنت‌هاوس). رنگ هر اتاق هزینهٔ مصالح هر متر مربع آن را نشان می‌دهد. روی یک طبقه یا هر جزء نقشه کلیک کنید، مبدأ و سطح قیمت را عوض کنید و تغییر عدد کل را ببینید.`
- en: `A sample 5-storey residential building (two units per floor, a penthouse on floor 5). Each room's colour shows its material cost per m². Click a floor or any element on the plan, change its origin and price level, and watch the total move.`

### 3.3 Room statistics
Add these helpers next to `totals()`:
```js
const ROOMN={room:['اتاق خواب','Bedroom'],living:['نشیمن','Living'],wet:['سرویس','Bath'],kitchen:['آشپزخانه','Kitchen']};
// One entry per room on floor f: {ri, rect, kind, area (m²), cost (material, toman), rate (toman/m²), fe (floor element)}
function roomStats(f,price){const out=[];
 for(const fe of E.filter(e=>e.f===f&&e.c==='floor')){const ri=fe.room;
   const own=E.filter(e=>e.f===f&&e.room===ri);
   const c=own.reduce((a,e)=>a+(price?price(e):cost(e)),0);
   out.push({ri,rect:fe.rect,kind:fe.kind,area:fe.q,cost:c,rate:c/fe.q,fe})}
 return out}
const floorCost=f=>E.filter(e=>e.f===f).reduce((a,e)=>a+cost(e),0);
```

### 3.4 A fixed, absolute heat scale (computed once)
The scale must **not** rescale with every change, otherwise switching the whole project to Luxury
would leave the colours unchanged. Compute it once at startup, from the cheapest and the most
expensive *achievable* room rate (every uniform origin × level combination), on a
**geometric (log) scale with 5 bins**:
```js
let HEAT=null; // {edges:[6 numbers, toman/m²]}
function initHeat(){let lo=Infinity,hi=0;
 for(const o of ORIG)for(const l of LEV){const p=e=>e.q*unit(e.c,o.k,l.k);
   for(const r of roomStats(0,p)){lo=Math.min(lo,r.rate);hi=Math.max(hi,r.rate)}}
 const k=Math.pow(hi/lo,1/5);HEAT={edges:Array.from({length:6},(_,i)=>lo*Math.pow(k,i))}}
const heatBin=v=>Math.max(0,Math.min(4,Math.floor(Math.log(v/HEAT.edges[0])/Math.log(HEAT.edges[1]/HEAT.edges[0]))));
```
Call `initHeat()` once, right after `seedPenthouse()`.

---

## 4. Visual specification

### 4.1 Heat colours (validated — use exactly these)
Single hue (OKLCH h≈48°, orange), 5 ordinal steps, checked for monotone lightness, ≥0.06 ΔL
between steps and ≥2:1 contrast of the faintest step against the surface.

| step (bin) | dark theme fill | label colour on it | light & read theme fill | label colour on it |
|---|---|---|---|---|
| 0 cheapest | `#79472c` | `#ffffff` | `#f1a278` | `#14213a` |
| 1 | `#a55628` | `#ffffff` | `#e3814a` | `#14213a` |
| 2 | `#d06829` | `#04121f` | `#d06217` | `#14213a` |
| 3 | `#fb8139` | `#04121f` | `#b54900` | `#ffffff` |
| 4 most expensive | `#ffaa73` | `#04121f` | `#963300` | `#ffffff` |

```js
const HEATC={dark:{f:['#79472c','#a55628','#d06829','#fb8139','#ffaa73'],t:['#fff','#fff','#04121f','#04121f','#04121f']},
             light:{f:['#f1a278','#e3814a','#d06217','#b54900','#963300'],t:['#14213a','#14213a','#14213a','#fff','#fff']}};
const heatPal=()=>HEATC[(document.documentElement.dataset.theme||'dark')==='dark'?'dark':'light'];
```
Note the direction: in **dark** theme cheap = dark, expensive = bright (glows); in **light/read**
cheap = pale, expensive = deep. Both read "more = more ink against the background".

### 4.2 Plan in the "Cost heat" lens (`lens==='cost'`)
Rewrite only the `lens==='cost'` drawing path of `drawPlan()`. The "Origin" and "Price level"
lenses must render **exactly as today** (keep that code path untouched).

Draw order (later = on top): rooms → room labels → walls → windows → points (doors/fixtures) → badge.

- **Rooms** (one per room from `roomStats(curFloor)`):
  `<rect class="el hm-room" data-id="${r.fe.id}" data-ri="${r.ri}" x="${x+6}" y="${y+6}" width="${w-12}" height="${h-12}" rx="8" fill="${pal.f[b]}"/>`
  Solid fill, no opacity, no stroke. Selected room (`sel===r.fe.id`): add class `sel` (existing
  `.plan .sel` gives a white 6px stroke — in light/read theme override to `var(--text)`, see CSS).
  Clicking a room selects its floor element (`sel=r.fe.id; refresh()`), as today.
- **Room label** (centred in the room, `pointer-events:none`, colour `pal.t[b]`, via `style="fill:…"`):
  - line 1: room name from `ROOMN` — `font-size:30px;font-weight:700`, at `y = centreY - 14`.
  - line 2: rate in **million toman per m², one decimal** (e.g. `3.4`, fa `۳٫۴`) —
    `font-size:46px;font-weight:800`, at `y = centreY + 40`, class `hm-v`.
    Format: `const m=r.rate/1e6, s=m<10?m.toFixed(1):m.toFixed(0); lang==='fa'?faD(s).replace('.','٫'):s`.
  - No unit text on the plan; the legend states the unit.
- **Walls** (`extwall`, `intwall`): same geometry and widths as today, class `el hm-n`
  (neutral stroke, see CSS), still clickable.
- **Windows**: outer line stroke `#9fe8ff` width 26 as today; the inner line becomes class `hm-n`
  with stroke-width 8 (no heat colour).
- **Points** (`door`, `entry`, `sanitary`, `kitchen`, `elec`): same shapes, sizes and letters as today,
  but fill/stroke via class `hm-pt` (neutral) and letters via class `hm-pl`.
- **"Highest" badge**: on the room with the highest `rate` on this floor (first one on ties), a pill
  at the room's top-inline-start corner: `<g class="hm-badge">` containing
  `<rect x=… y=… width="230" height="56" rx="28"/>` and `<text>▲ گران‌ترین</text>` (en `▲ Highest`),
  `font-size:28px;font-weight:800`. Inset 20 svg units from the room's corner. In RTL (`lang==='fa'`)
  the inline-start corner is the top-**right** corner of the room rect; in English the top-left.
  Never colour-only: the ▲ and the word carry the meaning.

### 4.3 Legend (replaces the cost-lens content of `#legend`; other lenses unchanged)
```
<div class="hm-legend">
  <div class="hm-title">هزینهٔ مصالح هر متر مربع اتاق (میلیون تومان) | Room material cost per m² (million toman)</div>
  <div class="hm-scale"> 5 × <i style="background:pal.f[k]"></i> </div>
  <div class="hm-ticks"> 6 × <span>edge value</span> </div>
  <div class="hm-key">D در · E ورودی · S سرویس · K کابینت · ⚡ برق</div>   (en: D door · E entrance · S sanitary · K kitchen · ⚡ electrical)
</div>
```
- Swatches: each `flex:1`, height 12px, `border-radius:3px`, 2px gap between them, max total width 360px.
- Ticks: 6 labels under the swatch boundaries (`display:flex;justify-content:space-between`, same
  360px max width), font 12px, colour `var(--mute)`, tabular numbers. Values = `HEAT.edges` in
  million toman, formatted like the room values (one decimal below 10, integer from 10).
- The scale runs in reading direction (RTL in Persian: cheapest on the right). Do not force LTR.

### 4.4 Building strip (replaces `#floors`)
Markup: in the plan pane, replace the toolbar's `<div class="row seg" id="floors"></div>` with nothing,
and wrap the SVG + strip:
```
<div class="hm-stage">
  <div class="bs" id="bstrip" role="group" aria-label="طبقات ساختمان / Building floors"></div>
  <div class="hm-plan"><svg class="plan" id="plan" viewBox="-60 -60 2520 1420"></svg><div class="hm-tip" id="hmtip" hidden></div></div>
</div>
<div class="legend" id="legend"></div>
```
`drawStrip()` renders into `#bstrip` (call it from `drawPlan()`; remove the old
`seg(document.getElementById('floors'),…)` line):

```
<div class="bs-head"><b>ساختمان | Building</b><span>{bil(sum of floorCost)}</span></div>
for f = FLOORS-1 down to 0:
  <button class="bs-row {on if f===curFloor}" data-f="f" aria-pressed="true|false"
          aria-label="{Floor N}, {bil(value)} {toman}">
     <span class="bs-n">{N}</span>
     <span class="bs-track"><span class="bs-bar" style="width:{100*value/max}%"></span></span>
     <span class="bs-v">{bil(value)}{f===argmax ? ' ▲' : ''}</span>
  </button>
<div class="bs-hint">برای دیدن نقشه روی یک طبقه بزنید | Tap a floor to open its plan</div>
```
- `N` = floor number (fa digits in Persian). `value = floorCost(f)` (material cost, toman).
  `bil()` already returns "2.4 bn" / "۲٫۴ میلیارد".
- **Bar length encodes magnitude from zero** (never a truncated axis). Bar colour is `var(--acc)`
  for every floor — **not** the heat ramp (the heat ramp means "per m² of a room"; the strip means
  "floor total"). The selected floor's bar is `var(--acc2)`.
- `▲` marks the most expensive floor (text, not colour only).
- Click a row → `curFloor=f; refresh()`. Rows are real `<button>`s (keyboard + focus ring).

### 4.5 Room tooltip
On `pointerenter`/`pointermove` over `.hm-room`, show `#hmtip` near the pointer (8px offset,
clamped inside `.hm-plan`); hide on `pointerleave`. On touch (`pointerType==='touch'`), a tap shows
the tooltip for that room (and selects it, as the click already does).
Content (bilingual, values via `fmt`/`bil`):
```
<b>{room name} · {Floor N}</b>
Area | مساحت: {area, 1 decimal} m² | م²
Per m² | هر متر مربع: {fmt(rate)} toman | تومان
Room total | کل اتاق: {fmt(cost)} toman | تومان
Share of floor | سهم از طبقه: {round(100*cost/floorCost(curFloor))}% | ٪
```

### 4.6 CSS (append to the end of the main `<style>`; nothing else in CSS changes)
```css
#demo .hm-stage{display:grid;grid-template-columns:168px 1fr;gap:12px;align-items:stretch;margin-top:10px}
#demo .hm-plan{position:relative;min-width:0}
#demo .bs{display:flex;flex-direction:column;gap:4px;background:var(--panel2);border:1px solid var(--line);border-radius:12px;padding:10px}
#demo .bs-head{display:flex;justify-content:space-between;align-items:baseline;font-size:12px;color:var(--mute);padding:0 2px 6px;border-bottom:1px solid var(--line);margin-bottom:2px}
#demo .bs-head b{color:var(--text);font-size:13px}
#demo .bs-row{flex:1;display:grid;grid-template-columns:18px 1fr auto;align-items:center;gap:8px;min-height:36px;background:transparent;border:1px solid transparent;border-radius:8px;padding:4px 8px;color:var(--mute);font-family:inherit;font-size:12px;cursor:pointer;text-align:start}
#demo .bs-row:hover{border-color:var(--line);color:var(--text)}
#demo .bs-row.on{background:var(--panel);border-color:var(--acc);color:var(--text);font-weight:700}
#demo .bs-row:focus-visible{outline:2px solid var(--acc);outline-offset:2px}
#demo .bs-n{font-weight:800;color:var(--text)}
#demo .bs-track{height:8px;background:var(--line);border-radius:4px;overflow:hidden}
#demo .bs-bar{display:block;height:100%;background:var(--acc);border-radius:4px;transition:width .45s ease}
#demo .bs-row.on .bs-bar{background:var(--acc2)}
#demo .bs-v{font-variant-numeric:tabular-nums;white-space:nowrap}
#demo .bs-hint{font-size:11px;color:var(--mute);text-align:center;padding-top:4px}
#plan .hm-room{transition:fill .35s}
#plan .hm-n{stroke:var(--mute);stroke-opacity:.55}
#plan .hm-pt{fill:var(--panel2);stroke:var(--line);stroke-width:4}
#plan text.hm-pl{fill:var(--text);font-weight:800;font-size:32px}
#plan .hm-badge rect{fill:var(--text)}#plan .hm-badge text{fill:var(--bg);font-size:28px;font-weight:800}
html[data-theme=light] #plan .sel,html[data-theme=read] #plan .sel{stroke:var(--text)!important}
#demo .hm-tip{position:absolute;z-index:5;pointer-events:none;background:var(--panel);border:1px solid var(--line);border-radius:10px;padding:8px 12px;font-size:12px;line-height:1.7;color:var(--text);box-shadow:0 8px 24px rgba(0,0,0,.25);min-width:180px}
#demo .hm-legend{display:grid;gap:4px;margin-top:10px;font-size:12px;color:var(--mute)}
#demo .hm-title{color:var(--text);font-weight:700}
#demo .hm-scale{display:flex;gap:2px;max-width:360px}#demo .hm-scale i{flex:1;height:12px;border-radius:3px}
#demo .hm-ticks{display:flex;justify-content:space-between;max-width:360px;font-variant-numeric:tabular-nums}
@media(max-width:640px){
 #demo .hm-stage{grid-template-columns:1fr}
 #demo .bs-row{min-height:32px}
 #plan .hm-v{display:none}
}
```
On screens ≤640px the strip sits **above** the plan at full width, and the per-room numbers hide
(tap a room for the tooltip). Room names stay.

### 4.7 Theme switching must recolour the heatmap
`setTheme(t)` runs before the app is ready. Add at its end: `if(window._demoReady)drawPlan();`
and set `window._demoReady=true;` right after the final `setLang('fa');…` line at the bottom of the script.

---

## 5. Wiring summary (functions you touch)

| Where | Change |
|---|---|
| `buildFloor` | `room:ri` on room-owned adds; window → room assignment (3.1) |
| after `buildFloor` loop | `seedPenthouse(); initHeat();` (3.2, 3.4) |
| new helpers | `ROOMN`, `roomStats`, `floorCost`, `HEAT`, `heatBin`, `HEATC`, `heatPal`, `drawStrip`, tooltip handlers |
| `drawPlan` | remove `#floors` seg call; call `drawStrip()`; new cost-lens path (4.2); cost-lens legend (4.3) |
| demo HTML | toolbar keeps only `#lens`; add `.hm-stage` wrapper with `#bstrip`, `.hm-plan`, `#hmtip` (4.4) |
| `setTheme` | redraw plan when ready (4.7) |
| lead paragraph | new fa/en text (3.2) |
| CSS | append block 4.6 |

Do not change: `LENS`, origin/level lens drawing, `drawDet`, `drawCats`, `drawBars`, `drawIns`,
`drawBoq`, scenarios, budget, profit calculator, any other section.

---

## 6. Copy (all strings, both languages)

| key | fa | en |
|---|---|---|
| legend title | هزینهٔ مصالح هر متر مربع اتاق (میلیون تومان) | Room material cost per m² (million toman) |
| badge | ▲ گران‌ترین | ▲ Highest |
| strip head | ساختمان | Building |
| strip hint | برای دیدن نقشه روی یک طبقه بزنید | Tap a floor to open its plan |
| strip row aria | طبقهٔ N، X تومان | Floor N, X toman |
| tooltip rows | مساحت · هر متر مربع · کل اتاق · سهم از طبقه | Area · Per m² · Room total · Share of floor |
| room names | اتاق خواب · نشیمن · سرویس · آشپزخانه | Bedroom · Living · Bath · Kitchen |

---

## 7. Verification (all must pass before you push)

Serve the decrypted page locally for interactive tests: open `$S/work.html` via `file://`
(it is the decrypted document, no password needed). Write one Playwright script that:

1. Loads the page at 1280×900; no `pageerror` events.
2. Asserts every window element has a `room` (8 per floor × 5 floors = 40).
3. Asserts `#bstrip .bs-row` count is 5, floor 5 is on top, and floor 5's value > floor 1's value
   (penthouse; measured ≈ 4.53 bn vs 3.52 bn material cost, +29%) and carries `▲`.
4. Asserts in cost lens on floor 1: 10 `.hm-room` rects, each `fill` is one of the 5 ramp colours;
   at least 3 distinct bins are visible; kitchens have the highest rate, living rooms the lowest;
   exactly one `.hm-badge`, on a kitchen. (Reference values measured on the current engine,
   Iranian/Mid-range defaults: scale edges ≈ 2.1 · 4.0 · 7.3 · 13.5 · 24.8 · 45.8 M toman/m²;
   room rates ≈ living 3.1–3.9, bedrooms 4.4–9.7, baths 8.7, kitchens 12.2 → bins 0–2.
   The top two bins are intentionally reached only by Expensive/Luxury or pricier origins —
   the penthouse floor and the Mix board show them.)
5. Clicks floor 1 in the strip → plan redraws, the row gets `.on`, `aria-pressed="true"`.
6. Clicks Mix board origin "Turkish" → at least one room moves to a higher bin (absolute scale works).
7. Switches lens to "Origin" and "Price level" → plan looks exactly as before this change
   (no `.hm-room`, legend shows the old swatches).
8. `setTheme('light')` → room fills switch to the light ramp; `setTheme('read')` the same.
9. `setLang('en')` → all strip/legend/badge/tooltip strings in English, digits Latin.
10. Viewport 390×800: `document.documentElement.scrollWidth === 390` (no horizontal scroll),
    strip is above the plan, `.hm-v` hidden.

Then take screenshots and **look at them** (dark fa, dark en, light fa, mobile fa) of `#demo`:
the expensive rooms must be identifiable in under a second, no label may overflow its room, the
badge must not cover a room name, the strip must line up with the plan's height on desktop.
Hovering a room must show the tooltip inside the plan box.

Also check: `pdf.cjs` output has the same page count as before ±1 and the demo page in the PDF
shows the heatmap (open the page with `pdftoppm -f <n> -l <n> -png`).

---

## 8. Ship

1. Build order 2.4 (PDFs → embed → encrypt → round-trip `cmp`).
2. `git add proposal.json && git commit` (only `proposal.json` changes; message describes the
   heatmap + building strip; end with the attribution lines the session gives you).
3. `git push -u origin <your claude/ branch>` — the publish workflow merges to `main` and
   verifies the live site. Check the "Publish to main" run is green with the GitHub tools; if it
   is red, read its log and report — do not push to `main` by hand.
4. Tell the user: what changed, that the live site updates ~2–5 minutes after the run finishes,
   and the two screenshots (dark fa desktop + mobile).
