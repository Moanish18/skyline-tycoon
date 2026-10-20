# 🏙️ Skyline Tycoon

A free idle tycoon game in **pure HTML/CSS/JS**: one file, no dependencies, no build step. Build 25 floors, hire staff, **Rebrand** for permanent Fame, and climb the skyline. Plays fully offline and installs as a PWA. Desktop only (keyboard + mouse).

**Play:** `https://moanish18.github.io/skyline-tycoon/`

## Features
- 25 floors with level milestones (income x2), upgrades, 6 unlockable characters, 8 achievements
- Prestige ("Rebrand") with permanent Fame and a Fame shop
- Offline earnings, combo tapping, bonus drones, synthesised sound (no audio files)
- Animated canvas skyline, day/night cycle, growing tower, particles
- Save export/import, auto-save, installable PWA

## Controls
| Key | Action |
|---|---|
| `Space` / `Enter` / click character | Tap to earn |
| `1`-`9`, `0` | Buy floors 1-10 |
| `B` | Cycle buy amount (x1 / x10 / x100 / Max) |
| `U` | Buy next upgrade |
| `Q` / `E` | Previous / next unlocked character |
| `T` | Cycle right-hand tabs |
| `M` | Mute / unmute |

## Run locally
```bash
# Option 1: just open index.html in a browser (service worker is disabled on file://)
# Option 2: serve it (enables PWA/offline testing)
python3 -m http.server 8000      # then visit http://localhost:8000
```

## Deploy to GitHub Pages
1. Push all files in this folder to a repo with `index.html` at the root.
2. **Settings → Pages → Deploy from a branch → `main` / root.**
3. Visit `https://<user>.github.io/<repo>/`.
4. After every update, bump the cache name in `sw.js` (`const C='skyline-v2'` → `v3`) so players get the new version.

---

# Developer documentation

## Project structure
```
index.html            The entire game: HTML + CSS + JS in one file
manifest.webmanifest  PWA metadata (name, colours, icons)
sw.js                 Service worker: network-first, cache fallback (offline play)
icon.svg / icon-*.png App icons
README.md             This file
```

## How the game works
```
 input (mouse/keys) ──► state `S` ──► calc() ──► inc (income/sec), gm (global multiplier)
                          ▲                         │
   loop(): gain(inc*dt) ──┘                         ▼
   every frame:  drawBG · drawTW · drawFX      every 100ms: updUI()
   every 5s / tab hidden / unload: save() ──► localStorage
```
- **`S`** is the single source of truth (see [Save data](#save-data)). Everything else is derived from it.
- **`calc()`** recomputes `inc` and `gm` from `S`. Call it after changing anything that affects income (levels, upgrades, fame, character, achievements).
- **`loop()`** runs every animation frame: adds income, runs timers, draws the three canvases. UI text updates at 10 Hz via `updUI()` to stay cheap.
- Static DOM (floor rows) is built **once**; `updUI()` only edits text and classes. The right panel is rebuilt by `renderR()` only when something changes (tab, purchase, rebrand).

## `index.html` map
Sections appear in this order inside the `<script>`:

| Section | Key names | What it does |
|---|---|---|
| Constants | `KEY`, `N`, `G` | Save key, floor count, cost growth |
| Data tables | `NM`, `IC`, `F`, `MS`, `CH`, `SH`, `UN`, `ACH` | All content and balance numbers |
| Formatting | `fmt`, `SUF` | Number abbreviations (K, M, B ... Vg, then `1.23e66`) |
| State | `load`, `save`, `S` | Defaults, validation, persistence |
| Economy | `calc`, `costN`, `maxN`, `buy`, `tapVal`, `gain`, `pending`, `rebrand` | All game rules |
| Sound & IO | `tone`, `snd`, `exportSave`, `importSave` | Web Audio synth, save codes |
| UI | `renderR`, `updUI`, `els` | DOM building and refresh |
| Graphics | `drawBG`, `drawTW`, `drawFX`, `burst`, `spawnDrone` | Canvas layers and particles |
| Input | `keydown` handler | Hotkeys |
| Boot | `offline`, `loop` | Startup, offline earnings, main loop |

## Game data (what to edit and what it changes)

### Floors
```js
const NM=['Lemonade Stand', ...];   // 25 names
const IC=['🍋', ...];               // 25 emoji icons
const F=NM.map((n,i)=>({n,ic:IC[i],c:15*Math.pow(13,i),b:Math.pow(9,i)}));
```
- `c` = base cost, `b` = income per level per second (before multipliers).
- Cost ratio (13) vs income ratio (9) between tiers sets how much slower each deeper floor pays back. Raise the cost ratio to slow late game; lower it to speed up.

### Cost growth and milestones
```js
const G=1.1;                          // each level costs 10% more than the last
const MS=[25,50,100,200,300,400,500,750,1000]; // each reached level = floor income x2
```
Past the last entry, milestones continue every 500 levels (`nextMs`). Lower `G` (e.g. 1.07) makes mass-levelling cheaper and the game faster.

### Characters
```js
{n:'Banker', e:'🧑‍💼', u:15, tap:1, inc:1.5, cost:1, d:'+50% income'}
```
| Field | Meaning |
|---|---|
| `u` | Fame needed to unlock |
| `tap` | Tap-power multiplier |
| `inc` | Global income multiplier |
| `cost` | Floor cost multiplier (lower = cheaper) |

### Upgrades
```js
const ucost=k=>1e3*Math.pow(35,k);   // cost of upgrade #k; each gives all income x2
const unm=k=>UN[k]||'Cosmic Contract '+(k-UN.length+1);
```
Upgrades are endless. Past the named list they get generated names.

### Fame shop (permanent)
```js
{n:'Golden Touch', d:'x2 tap power', c:l=>5*Math.pow(3,l), m:50}  // c = cost at level l, m = max level
```
Effects live in the economy code: `shop[0]` tap (`tapVal`), `shop[1]` offline cap (`offline`), `shop[2]` cost discount (`cmult`), `shop[3]` drone speed (`loop`).

### Achievements
```js
['Millionaire','Earn $1M lifetime',()=>S.all>=1e6]   // [name, description, test function]
```
Each gives +5% income (`calc`). Saved by **index**, so only append new ones to the end; reordering would mislabel old saves.

## Core formulas
```js
// Cost to buy n levels of floor i
costN = F[i].c * G**S.lv[i] * (G**n - 1)/(G-1) * cmult()
cmult = CH[S.ch].cost * max(.3, 1 - .03*S.shop[2])

// Income per second
inc = Σ F[i].b * lv[i] * 2**milestonesReached(lv[i])   *   gm
gm  = 2**S.up * CH.inc * (1 + .1*S.fe) * (1 + .05*S.ach.length) * (boost>0 ? 2 : 1)

// Tap value
tapVal = (1 + inc*.08) * CH.tap * 2**S.shop[0] * (1 + min(combo,60)/30)

// Prestige
pending = max(0, floor(sqrt(S.all/1e7)) - S.fe)    // Fame gained on Rebrand
```

## Balance cheat sheet
| Change this | Where | Effect |
|---|---|---|
| `G` | constants | Level cost growth. Lower = faster |
| `15*Math.pow(13,i)` | `F` | Deeper floors cost more. Lower base ratio = faster |
| `Math.pow(9,i)` | `F` | Deeper floors earn more. Higher = faster |
| `MS` | data | When floors double |
| `1e3*Math.pow(35,k)` | `ucost` | Upgrade price curve |
| `1e7` | `pending()` **and** the "Next Fame needs" text in `renderR` | Fame difficulty. Change both together |
| `.1*S.fe` | `calc` | Income per Fame point |
| `.05*S.ach.length` | `calc` | Income per achievement |
| `.08`, `60`, `30` | `tapVal` | Tap power share, combo cap, combo strength |
| `*.5` and `8` (hours) | `offline` | Offline earning rate and cap |
| `40+Math.random()*50` | `loop` | Seconds between drones |
| `1e300`, `1e290` | `gain`, `calc` | Overflow safety caps |

## Save data
Stored in `localStorage` under `KEY='skylineTycoon_v1'`.
```js
S = {
  v:2,            // save version
  cash, run, all, // cash now / earned this run / earned lifetime (never resets)
  fe, sp,         // Fame earned in total / Fame spent in shop
  lv:[...25],     // level of each floor
  up,             // upgrades bought this run
  shop:[0,0,0,0], // Fame-shop levels (permanent)
  ch,             // selected character
  ach:[],         // unlocked achievement indexes (permanent)
  taps, mute, last, t0
}
```
- **Rebrand** resets `cash`, `run`, `lv`, `up` and keeps everything else.
- `load()` repairs corrupted/NaN values. **Warning:** it resets `lv` if its length differs from `N`, so if you add floors, migrate old saves (pad `lv` with zeros) before changing `N`.
- Changing `KEY` gives every player a fresh start.
- Saves are per browser and per origin. On GitHub Pages all projects under `<user>.github.io` share one origin, so keep the key unique.
- Players can back up through **Fame tab → Export/Import save code** (base64 JSON).

## Extending the game

**Add a floor:** add a name to `NM` and an emoji to `IC`, set `N` to the new count, and pad old saves (see warning above). `F` generates itself.

**Add a character:** append to `CH`. Unlock order is by array position, so keep `u` ascending.

**Add an achievement:** append to `ACH` (end only).

**Add a Fame-shop item:** append to `SH`, then use `S.shop[index]` where the effect applies. `S.shop` has 4 slots by default, and `load()` resets it if shorter than 4, so extend the default array in `load()` too.

**Add a sound:** add an entry to `snd` using `tone(freq, seconds, waveform, volume, delay)` and call it where needed.

**Change the look:** colours are CSS variables at the top of the `<style>`:
```css
:root{--g:#ffd34d;   /* gold accent */
      --p:rgba(14,18,44,.74); /* panel background */
      --b:rgba(255,255,255,.13); /* borders */
      --a:#5eead4;}  /* teal accent */
```
Sky colours are in `drawBG`, tower colours in `drawTW` (both use `hsl()`).

## Rendering layers
| Layer | Element | Drawn by | Notes |
|---|---|---|---|
| Background | `canvas#bg` | `drawBG` | Sky, stars, 3 parallax skyline layers, day/night |
| UI | DOM panels | `updUI`, `renderR` | Glass panels, z-index 1 |
| Tower | `canvas#tw` (280x250) | `drawTW` | One segment per owned floor |
| Effects | `canvas#fx` | `drawFX`, `burst` | Floating numbers and coins, click-through, z-index 60 |

Particles are capped at 300 in `burst`. Drones are DOM elements (`spawnDrone`) animated with a CSS transition.

## PWA and offline
- `sw.js` is **network-first with cache fallback**, so players get updates when online and still play offline.
- It is only registered over `http(s)` (never `file://`).
- To force a fresh cache after a release, bump `const C='skyline-v2'`.
- The manifest uses the PNG icons for installability; `icon.svg` is the favicon.

## Debugging in the console
Top-level variables are reachable from DevTools:
```js
S.cash += 1e12                // free cash
S.lv[4] += 50; calc()         // levels
S.fe += 100; calc(); renderR()// Fame
spawnDrone()                  // trigger a drone
JSON.stringify(S)             // inspect the save
```

## Known limitations
- Numbers are plain JS doubles, capped at 1e300. Not a true big-number system.
- Long-term pacing is untested by simulation. Balance via the knobs above.
- Desktop only; no mobile controls.
- Cross-device sync is manual (save codes).

## Ideas for contributors
Floor managers/automation, a second prestige layer, random market events, big-number library, auto-play balance simulator, more characters and achievements.

## License
Add a `LICENSE` file (MIT recommended) before publishing.
