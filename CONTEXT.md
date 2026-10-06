# SyncDrive — Project Context File (v0.4)

> Paste this entire file into another AI (ChatGPT, Claude, Gemini, etc.) to continue development from where we left off.

---

## 🎯 Objective

**SyncDrive** is a Signal Sync GPS — a navigation app that tells a driver:

- **When to leave** (departure time) so they hit the most traffic signals on green
- **What speed to maintain** (a steady, moderate speed — 25–45 km/h) to avoid red lights entirely
- **Why this is faster**: driving at 35 km/h with zero red light stops is faster than driving at 50 km/h with 3–4 stops

The core proof:
```
Normal GPS:   2.6 min driving  +  2.5 min waiting at reds  =  5.1 min total
SyncDrive:    3.8 min driving  +  0 min waiting            =  3.8 min total ✅
```
Slower speed + zero stops = faster total journey.

---

## 🗺️ Scope

- **City**: Chandigarh, India — Sector 17, 22, 35, 43, PGI, Elante Mall, Tribune Chowk area
- **Tech**: Pure HTML + CSS + JavaScript, **single file** (`index.html`), no build step, no server needed
- **Map tiles**: Stadia Maps `alidade_smooth_dark` — **free, no API key needed**
- **Map library**: Leaflet.js 1.9.4 (CDN)
- **Font**: Inter (Google Fonts CDN)
- **Data**: All fake/hardcoded — signal timings, routes, distances

---

## 📁 File Structure

```
Project/
├── index.html     ← entire app (map + sidebar + all logic)
└── CONTEXT.md     ← this file
```

---

## 🏗️ Architecture

### Three-pane layout (Apple Maps style)
```
[ Nav Rail (200px) ] [ Directions Panel (320px) ] [ Map (flex:1) ]
```

### Nav Rail
- SyncDrive logo: white wave + sync arrows on black circle background
- Nav items: Search, Guides, Directions (active, with blue left border)
- Bottom: live clock + city name

### Directions Panel
- Drive / Walk / Cycle mode tabs (Apple style, blue active tab)
- "From / To" grouped input with blue dot (origin) + red pin (destination)
- "Arrive by" chip + time picker
- Blue "Sync Route" button
- Scrollable results body with Apple-style grouped list rows

### Map
- Dark Stadia tiles (navy/dark grey — Apple Maps look)
- Place labels: dark pill labels with blur backdrop
- Signal markers: small circles, grey default → green/red after calculation
- GreenWave route: blue polyline (`#0a84ff`) — Apple Maps active route colour
- Normal GPS ghost: faint red dashed polyline (same path)
- Speed HUD: bottom-right overlay, large speed number

---

## 📍 Places (8 locations)

| Key | Name | Lat | Lng |
|-----|------|-----|-----|
| `sec17` | Sector 17 Plaza | 30.7395 | 76.7836 |
| `sec22` | Sector 22 Chowk | 30.7296 | 76.7818 |
| `sec35` | Sector 35 Bus Stand | 30.7248 | 76.7670 |
| `sec43` | Sector 43 ISBT | 30.7080 | 76.7840 |
| `pgi` | PGI Hospital | 30.7657 | 76.7794 |
| `elante` | Elante Mall | 30.7062 | 76.8014 |
| `isbt43` | ISBT Sector 43 | 30.7080 | 76.7840 |
| `tribune` | Tribune Chowk | 30.7018 | 76.7914 |

---

## 🚦 Traffic Signals (16 fake signals)

Each signal object:
```js
{ id, name, lat, lng, cycle (total seconds), gS (greenStart offset), gD (green duration) }
```

| id | name | cycle | gS | gD |
|----|------|-------|----|----|
| c1 | Jan Marg & Madhya Marg | 90 | 0 | 42 |
| c2 | Sector 17-22 Divider | 70 | 12 | 32 |
| c3 | Udyog Path Crossing | 80 | 5 | 38 |
| c4 | Sector 22 Inner Circle | 60 | 20 | 28 |
| c5 | Himalaya Marg Signal | 75 | 8 | 35 |
| c6 | Sector 34-35 Crossing | 90 | 30 | 40 |
| c7 | Sector 35 Market Entry | 60 | 0 | 25 |
| c8 | Airport Rd & Sec 43 | 80 | 15 | 36 |
| c9 | Sector 43 Light Point | 70 | 5 | 30 |
| c10 | PGI Gate Signal | 90 | 10 | 45 |
| c11 | Madhya Marg & PGI Rd | 75 | 25 | 33 |
| c12 | Sector 11 Junction | 60 | 0 | 28 |
| c13 | Dakshin Marg & Sec 35 | 85 | 18 | 38 |
| c14 | Sec 35-43 Connector | 75 | 5 | 32 |
| c15 | Elante Entry Signal | 90 | 22 | 40 |
| c16 | Tribune Chowk Signal | 70 | 10 | 30 |

---

## 🛣️ Routes (20 bidirectional)

All pairs defined (both directions):
- sec17 ↔ sec22
- sec22 ↔ sec35
- sec17 ↔ sec43
- sec17 ↔ pgi
- sec22 ↔ pgi
- sec35 ↔ sec43
- sec22 ↔ elante
- sec43 ↔ elante
- sec35 ↔ tribune
- sec17 ↔ tribune

Each route has: `sigs[]` (signal ids), `km` (distance), `pts[][]` (Leaflet polyline coords), `steps[]` (turn-by-turn strings).

---

## 🧠 Algorithms

### `optimise(route, km, arrSec)` — GreenWave optimizer

```
Speeds tested: [25, 28, 30, 32, 35, 38, 40, 42, 45] km/h
Search window: up to 25 minutes before latest departure
Step size:     15 seconds

For each (speed, departure_offset) combination:
  driveSec = (km / speed) * 3600
  dep = arrivalTime - driveSec - offset

  Simulate journey through each signal:
    phase = (dep + timeToSignal) % signalCycle
    if green → count++
    if red   → wait = time until next green; add to elapsed

  total = driveSec + totalWait
  score = total - greens * 0.5   ← lower is better

Return best {dep, speed, driveSec, waitSec, total, greens, details[]}
```

### `simNG(route, km, gwDep)` — Normal GPS simulation

```
Speed: 50 km/h (city rushing speed)
Departure: same as GreenWave departure (fair apples-to-apples comparison)
Same signal positions, same departure time → different phases because different speed

For each signal:
  phase = (gwDep + timeToSignal_at_50kmh + elapsed) % cycle
  if red → wait = time until next green; add elapsed

total = driveSec_at_50 + totalWait
```

### Signal phase math
```js
// Is a signal green at a given phase?
isGreen(sig, ph) = ph >= sig.gS && ph < (sig.gS + sig.gD)

// How long to wait if red?
waitSec(sig, ph) = ph < sig.gS
  ? sig.gS - ph                    // green hasn't started yet this cycle
  : sig.cycle - ph + sig.gS        // past green, wait for next cycle
```

---

## 🎨 Design System (Apple Maps dark)

```
--bg:       #1c1c1e   (Apple dark surface)
--bg2:      #2c2c2e   (secondary surface — list groups)
--bg3:      #3a3a3c   (tertiary)
--bg4:      #48484a   (signal default colour)
--border:   rgba(255,255,255,.08)
--border2:  rgba(255,255,255,.12)
--text:     #f5f5f7
--sub:      #aeaeb2
--muted:    #636366
--accent:   #0a84ff   ← Apple blue, PRIMARY UI accent (buttons, active route)
--green:    #30d158   ← used ONLY for "signal is green" indicator
--red:      #ff453a   ← red lights, destination pin
--amber:    #ffd60a   ← ETA time display
```

**Green usage policy**: green (`#30d158`) is used ONLY where it means "this signal is green". All UI chrome, buttons, active states use Apple blue (`#0a84ff`).

---

## 📊 Results UI Structure (after calculation)

1. **Departure card** — large leave-at time, blue clock icon
2. **Speed card** — big speed number, "Slower — saves time" badge, vs note
3. **Quick stats list** — Arrive at / Distance / Travel time / Signals hit green
4. **"Why It's Faster" section**:
   - Side-by-side total time: Normal GPS (left) vs SyncDrive (right) with saved badge
   - Time breakdown bars: blue = driving, red = waiting at reds
   - Detail rows: Speed / Red lights / Time lost waiting / Fuel usage
5. **Signal timeline** — each signal with green/red dot + badge + arrival time
6. **Turn-by-turn steps**

---

## ✅ What's Working (v0.4)

- [x] Apple Maps three-pane layout (nav rail + panel + map)
- [x] SyncDrive logo (white wave + sync arrows, black background)
- [x] Drive/Walk/Cycle mode tabs
- [x] Stadia dark tiles (no API key, navy look)
- [x] 8 places, 16 signals, 20 routes
- [x] Green wave optimizer (25–45 km/h, minimise total time)
- [x] Normal GPS simulation (50 km/h, same departure, hits reds)
- [x] Time breakdown bars (visual proof slower = faster)
- [x] Apple-style grouped list rows throughout
- [x] Green used only for signal-is-green semantics
- [x] Blue active route polyline on map
- [x] Speed HUD overlay bottom-right
- [x] Toast notifications
- [x] Live clock

---

## 🔜 Potential Next Steps

### High priority
- [ ] **Animated car** moving along route at recommended speed
- [ ] **"Leave Now" mode** — use current system time, find best speed immediately
- [ ] **Countdown** — "Leave in X minutes" live timer in the panel
- [ ] **More Chandigarh locations** — Rock Garden, Sukhna Lake, Chandigarh Railway Station, IT Park

### Medium priority
- [ ] **Real routing** — integrate OSRM or GraphHopper (free, open-source) for actual road-following paths
- [ ] **Peak hour profiles** — signal timings change at 8–10am and 5–8pm rush hours
- [ ] **Congestion factor** — reduce recommended speed during peak traffic
- [ ] **Multiple waypoints** — add stop along route (like Apple Maps "Add Stop")

### Polish
- [ ] Mobile responsive (sidebar becomes bottom sheet)
- [ ] Share route link (state in URL hash)
- [ ] PWA manifest for installable app

---

## 🛠️ How to Run

1. Open `index.html` in any modern browser — no server, no install needed
2. Select From / To locations (8 Chandigarh places)
3. Set arrival time
4. Click **Sync Route**
5. Map draws the blue route + ghost red normal-GPS path
6. Panel shows: departure time, recommended speed, comparison table, signal timeline

---

## 📦 Dependencies (CDN only, no installs)

| Library | Version | URL |
|---------|---------|-----|
| Leaflet.js | 1.9.4 | unpkg.com |
| Stadia dark tiles | — | tiles.stadiamaps.com (free, no key) |
| Inter font | — | fonts.googleapis.com |

---

*SyncDrive v0.4 — Chandigarh region — context file updated after Apple Maps UI redesign.*
