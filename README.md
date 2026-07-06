# tonggeun (통근권) — Seoul-area commute isochrone map

> Drop a single starting point and this single-file web app draws "the area you can reach within 30 / 60 / 90 minutes" on a map as an isochrone (等時圈), based on **Seoul metropolitan rail + walking (+ optional bus)**. Intended for gauging at a glance "how far can I commute from here" when choosing a home or workplace.

🔗 **Live:** https://clayborneyeounjunlee.github.io/tonggeun/

---

## ✨ Key features

- 🗺️ **Set an origin** — tap the map, search by address or place name (e.g. `강남역`, `판교`), or pick via the street-address popup (📮). The marker can also be **dragged** for fine adjustment.
- ⏱️ **Draw isochrones** — computes **subway + walking** travel times from the chosen origin and visualizes the reachable area as a union of colored circles.
- 🎚️ **30 / 60 / 90 minute band selection** — press one of three time-band buttons to display it. Each band's threshold (minutes) can be edited directly as a number.
- 🚶 **Two-layer display: walk reach / 🚌 bus reach** — draws a darker area reachable by walking alone and a lighter, wider area when buses are included (approximated), overlaid and visually distinguished.
- 📍 **Reachable-station dots** — places a dot on each reachable station; clicking a dot shows the estimated travel time in an InfoWindow as `station name · ~NN min`.
- ⚙️ **Adjustable computation parameters** — walking speed, bus inclusion / average speed / wait time, transfer penalty, and maximum walking time from a station, all via sliders. Recomputes immediately on change.
- 🌗 **Dark mode** + 🌐 **Korean/English toggle (EN⇄한)** — toggled via icon buttons in the header. The theme is shared with the moa hub apps.
- 💾 **Automatic state persistence** — saves the last origin, settings, language, and theme in the browser and restores them on the next visit.
- ◈ **Back to the moa hub** button.

> ⚠️ Results are **estimates based on standard station-to-station travel times**. Actual headways, transfer walking, congestion, and real-time bus routes are not reflected. Use only to roughly gauge your commute range.

---

## 🧱 Tech stack / languages

| Item | Details |
|---|---|
| Languages | Pure **HTML + CSS + Vanilla JavaScript** (**no** frameworks or build tools) |
| JS style | Classic script (embedded directly in `<script>`, IIFE + `"use strict"`). **Not ES modules**, no `import`/`export` |
| File count | **Just 2** — `index.html`, `subway-data.js` (data injected via the global `window.SUBWAY`) |
| Map SDK | **Kakao Maps JavaScript SDK** — `//dapi.kakao.com/v2/maps/sdk.js`, **dynamically loaded at runtime** with `autoload=false` + `libraries=services` |
| Address popup | **Kakao (Daum) postal-code service** — `//t1.kakaocdn.net/mapjsapi/bundle/postcode/prod/postcode.v2.js` (no key required, free) |
| Geocoding/search | Kakao `services.Geocoder` (address ↔ coordinates), `services.Places` (keyword search) |
| Fonts | System font stack — `Pretendard`, `Apple SD Gothic Neo`, `Malgun Gothic`, `Noto Sans KR` (local fallbacks, no web-font embedding) |
| Styling | CSS custom properties (design tokens) + `data-theme` dark mode, `env(safe-area-inset-*)` support (mobile/notch) |
| i18n | Hand-rolled dictionaries (`I18N.ko` / `I18N.en`) + a `T()` helper + `data-i18n*` attribute scanning |
| Algorithms | Hand-implemented **Dijkstra** on a **binary min-heap (MinHeap)** + Haversine distance |
| Backend | **None** (fully static, client-only) |

The two Kakao SDKs above are the only CDN libraries; there are no other external dependencies such as jQuery, chart libraries, or polyfills.

---

## 🏗️ System architecture

### File layout

- **`index.html`** — a single file containing the UI markup, all CSS, and the entire application logic (one IIFE).
- **`subway-data.js`** — global data in the form `window.SUBWAY = { stations:[...], edges:[...] }`. Loaded via `<script src="subway-data.js">` **before** the logic in `index.html`.

### Boot flow

```
index.html loads
  └─ (head) hub-theme applied immediately → prevents dark-mode flash
  └─ postcode.v2.js (postal-code SDK) loads
  └─ subway-data.js loads → injects window.SUBWAY
  └─ main IIFE runs
       ├─ restore language/settings/origin from localStorage
       ├─ build COORD map (station name → [lat, lng])
       ├─ applyI18n(): fill text/placeholder/title/aria of data-i18n* elements
       ├─ bind controls (search, bands, slider, toggles, dark mode, language, key modal)
       └─ startMap()
            ├─ if localStorage["tg_kakaokey"] is missing → show key-input modal
            └─ if present → loadKakao(key) injects the SDK dynamically → initMap()
                 └─ create map → click/tile-load listeners → restore last origin
```

### Core modules and functions (inside index.html)

| Area | Function | Role |
|---|---|---|
| Graph construction | `buildAdj(transferPenalty)` | Builds an adjacency list of `station␟line` nodes from `edges`. Adds **transfer-penalty** edges between different line nodes of the same station |
| Distance | `haversine(la1,lo1,la2,lo2)` | Great-circle distance (m) between two coordinates |
| Access time | `accessMin(d)` | One leg between home/work and a station (minutes). The **faster** of walking vs (wait + bus) |
| Reach radius | `walkReach(t)` / `busReach(t)` | Radius (m) reachable by walking/bus on the final leg with the remaining time |
| Shortest paths | `computeArrival(oLat,oLng)` | **Dijkstra** — returns a `Map` of arrival times (minutes) from the origin to every station |
| Data structure | `MinHeap` | Binary min-heap for Dijkstra (push/pop implemented by hand) |
| Rendering | `render()` | Calls `drawLayer()` per band to draw walk-reach and bus-reach circles on the map and show reachable-station dots |
| Rendering | `drawLayer(Tmin, radiusFn, opacity, z)` | Draws the union of circles, deduplicated via grid snapping to **limit stacked-opacity buildup** |
| Map | `initMap` / `loadKakao(key)` / `startMap` | Map initialization, dynamic SDK loading, branching on key presence |
| Search | `doSearch()` / `openPostcode()` / `reverseGeo()` | Address search with keyword-search fallback, postal-code popup, reverse geocoding |
| Compute triggers | `setOrigin()` / `recompute()` / `fitToReach()` | Set origin → recompute → fit the view to the reachable area |

### State management & routing

- **State** lives in a single settings object `S` inside the IIFE closure plus module-level variables (`lastArrival`, `lastOrigin`, `map`, `marker`, etc.). No reactive framework — events update state and call `render()`/`recompute()` directly.
- **No routing** — a single screen. No URL parameters or hash routing; all state persists via `localStorage` only.

### How the isochrone is computed (summary)

1. Compute the straight-line distance from the origin to each station and derive the "station entry time" with `accessMin()`. Stations within the maximum access time are queued as Dijkstra start points (if all are too far, connect to at least the single nearest station).
2. Run Dijkstra on the `station␟line` node graph to get each station's **minimum arrival time (minutes)** (edge weight = station-to-station minutes, plus a penalty on transfers).
3. For each band threshold `Tmin`, draw circles from the origin itself and from the reachable stations, with radii reachable by walking/bus in the "remaining time" (`Tmin - arrival time`). The union of these circles is the isochrone.

---

## 🗂️ Data

All line data is hardcoded in the single file `subway-data.js`. The source is stated in the comment at the top of the file:

> `/* Seoul metropolitan-area subway data · Source: github.com/ledyx/KoreaMetroGraph (crawled from Seoul Metro / wiki) */`

### Size (measured from the code)

| Item | Count |
|---|---|
| Stations | **604** |
| Edges | **684** (680 base + 4 missing-edge fixes at the end of the file) |
| Line codes | **23** |

### Schema

**stations** — an array of `[station name, latitude, longitude]` tuples:

```js
window.SUBWAY = {
  stations: [
    ["녹천", 37.644799, 127.051269],
    ["고속터미널", 37.50481, 127.004943],
    ["양재", 37.484147, 127.034631],
    // … 604 in total
  ],
  ...
};
```

**edges** — an array of `[from station, to station, minutes, line code]` tuples:

```js
  edges: [
    ["서동탄", "병점", 5, "1"],
    ["강남", "역삼", 2, "2"],
    ["양재", "남부터미널", 3, "3"],
    // … 684 in total (interpreted as undirected)
  ]
```

At the very end of the file, real adjacent segments missing from the original crawl are appended as a **runtime patch**:

```js
/* ── Missing-edge fix: real adjacent segments absent from the original crawled data ── */
window.SUBWAY.edges.push(
  ["부천","소사",3,"1"],
  ["강남","신논현",2,"S"],
  ["신논현","논현",2,"S"],
  ["논현","신사",2,"S"]
);
```

### Line code → line (edge count)

Edge counts confirmed from the fourth field (line code) of `edges` and the actual data:

| Code | Line | Edges |
|---|---|---|
| `1` | Line 1 | 96 |
| `2` | Line 2 | 49 |
| `3` | Line 3 | 42 |
| `4` | Line 4 | 47 |
| `5` | Line 5 | 50 |
| `6` | Line 6 | 38 |
| `7` | Line 7 | 47 |
| `8` | Line 8 | 16 |
| `9` | Line 9 | 29 |
| `A` | AREX (Airport Railroad) | 11 |
| `K` | Gyeongui–Jungang Line | 54 |
| `G` | Gyeongchun Line | 23 |
| `B` | Suin–Bundang Line | 35 |
| `S` | Sinbundang Line (Shinbundang) | 15 |
| `SU` | Suin Line segment | 12 |
| `KK` | Gyeonggang Line | 10 |
| `I` | Incheon Line 1 | 28 |
| `I2` | Incheon Line 2 | 26 |
| `U` | Uijeongbu LRT | 14 |
| `E` | Yongin EverLine | 14 |
| `UI` | Ui–Sinseol Line | 12 |
| `W` | Seohae Line | 11 |
| `M` | Incheon Airport Maglev segment | 5 |

> Line codes come from the data file; in the code they are used only to separate graph edges by line (the same station is split into `station␟line` nodes so a transfer penalty can be applied). Line-name labels are not shown separately in the UI.

### Runtime data-structure transforms

- On app start, coordinates are indexed as `COORD = Map<station name, [latitude, longitude]>`.
- `buildAdj()` builds an adjacency list (`Map`) keyed by `station␟line` strings, plus a map of the lines each station belongs to (`stationLines: Map<station, Set<line>>`). The separator is Unicode `␟` (U+241F).

---

## 💾 Storage / DB

**No server DB or Firebase.** All state is stored only in the browser's **localStorage**. (Even as a guest/offline, computation works as long as the data file and a saved key are present; map tiles and address search do require the Kakao SDK and an internet connection.)

### localStorage keys

| Key | Purpose | Format/example |
|---|---|---|
| `tg_kakaokey` | **Kakao JavaScript key** (stored only on this device) | string |
| `hub-theme` | Light/dark theme (**shared** with the moa hub apps) | `"light"` / `"dark"` |
| `tg_lang` | Interface language | `"ko"` / `"en"` |
| `tg_settings` | Computation settings object | JSON — see below |
| `tg_origin` | Last origin | JSON `{lat, lng, label}` |

Default `tg_settings` (per the code):

```js
{ walk:4, transfer:4, maxwalk:15, dots:true,
  bus:false, busSpeed:12, busWait:5,
  bands:[30,60,90], on:[true,true,true] }
```

> In the code, `S`'s initial `on` is `[true,true,true]`, but on startup a "single selection" rule is enforced (whenever exactly one band is not enabled), so the effective initial display is corrected to `[false,true,false]` (= 60 minutes).

| Field | Meaning | Default | Range (UI) |
|---|---|---|---|
| `walk` | Walking speed (km/h) | 4 | 3–6 |
| `transfer` | Transfer penalty (min) | 4 | 0–10 |
| `maxwalk` | Max walk from a station (min) | 15 | 5–30 |
| `dots` | Show reachable-station dots | true | on/off |
| `bus` | Include buses (approximate) | false | on/off |
| `busSpeed` | Average bus speed (km/h) | 12 | 8–20 |
| `busWait` | Average bus wait (min) | 5 | 0–12 |
| `bands` | The 3 time bands (min) | `[30,60,90]` | free input |
| `on` | Which band is enabled (single selection enforced) | initial `[true,true,true]` → corrected to `[false,true,false]` | — |

> Only **single band selection** is allowed (to avoid overlap and excessive opacity). If the saved value violates the rule, the app corrects it to `[false,true,false]` (= 60 minutes).

---

## 🌐 External APIs & dependencies

Unlike its sibling apps, this is the **only app that requires an external map API key**.

### 1) Kakao Maps JavaScript SDK — **key required**

- Loaded via: `//dapi.kakao.com/v2/maps/sdk.js?appkey=<KEY>&autoload=false&libraries=services`
- Features used: map, marker (draggable), Circle overlays, InfoWindow, ZoomControl, `services.Geocoder` (address ↔ coordinates), `services.Places` (keyword search).
- **Where to enter the key:** on first launch (or via ⚙️ Settings → "Change Kakao key"), paste your **JavaScript key** into the modal; it is saved to `localStorage["tg_kakaokey"]`. The key is not hardcoded in the code — each user provides their own.
- **How to obtain and configure (as shown in the modal):**
  1. Log in at [developers.kakao.com](https://developers.kakao.com) → My Applications → Add an application
  2. Select the app → **Product settings → Kakao Map → toggle ON** (required since Dec 2024)
  3. Copy the **JavaScript key** from the app keys (not the REST key)
  4. Register your deployed URL under **Platform → Web → Site domains** (e.g. `https://clayborneyeounjunlee.github.io`)
  5. Paste the key into the modal → save
- If the map renders as a gray area only, it is usually because (a) the Kakao Map product is not enabled or (b) the site domain is not registered (the app shows a toast explaining this).

### 2) Kakao (Daum) postal-code service — **no key required**

- `postcode.v2.js` is loaded statically in the `<head>`. The 📮 button opens a street-address popup, and the selected address is geocoded back into the origin.

> No other services — TravelTime/Google/Skyscanner/exchange rates/Web Speech, etc. — are used.

---

## ▶️ Running locally

There is no build step at all. There is no `package.json` either, so just serve it with any **static file server**. (Opening directly via `file://` may be unreliable due to Kakao SDK domain restrictions, so a local server is recommended.)

```bash
# From the repository folder — any static server works
python -m http.server 8000
#   or
npx serve .
#   or
npx http-server -p 8000
```

Open `http://localhost:8000` in the browser → enter your Kakao **JavaScript key** on first launch.
To use the map locally, you must also register `http://localhost:8000` (the port you use) under **Web site domains** in the Kakao developer console.

---

## 🚀 Deployment

**GitHub Pages** static hosting.

1. Push this repository to GitHub (default branch `main`).
2. Repository **Settings → Pages** → Source: `main` branch root (`/`).
3. Register the deployment URL (e.g. `https://<username>.github.io/tonggeun/`) under **Web site domains** in the Kakao developer console, or map tiles will not render properly.

No build step or CI — files are served as-is.

---

## 📁 File structure

```
tonggeun/
├── index.html        # The entire app — UI markup + CSS design tokens/dark mode + logic (IIFE)
│                     #   · i18n (ko/en), Dijkstra+MinHeap, Kakao map/search/postal code,
│                     #   · isochrone rendering (walk/bus reach, 2 layers), localStorage state
└── subway-data.js    # Seoul-area subway data: window.SUBWAY = { stations[604], edges[684] }
                      #   · 4 missing segments patched via push at the end of the file
```

- This repository contains **no** `package.json`, `firebase.json`/`.firebaserc`, `.gitignore`, or server/route code (purely static, 2 files).

---

## 🔗 Related apps (moa hub)

- The **◈** button in the header → moa hub: https://clayborneyeounjunlee.github.io/moa/
- Shares the **design token system** (color variables, `--radius`, shadows, etc.) and the dark-mode key (`hub-theme`) with its sibling apps. Per the code comments, it uses the same common UI system as `moa/haru/kanade/dday`.
