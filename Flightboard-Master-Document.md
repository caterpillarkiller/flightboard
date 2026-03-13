# Flightboard — Master Project Document
**Last Updated:** March 13, 2026 (through Chat 1)
**Developer:** Chris Stinson, CHS Consulting, LLC
**Client / Owner:** Personal project — Chris Stinson

**Chat Series Note:** Always confirm chat number when uploading a session summary.

---

## What This Project Does

A live flight information display board for any airport, designed to be shown on a large TV screen. The flight board should mimic a split-flap-style information board, commonly found in European railway stations and airports. It pulls real-time departures and arrivals from the AviationStack API and displays airline logos, flight numbers, routes, gate info, and status. The primary use case is to show near real-time flight arrival and departure information for a given airport on a large display. It replaces the need for expensive commercial flight board software by using a free API tier with smart rate-limit controls.

---

## Key Stakeholders

| Name | Role |
|------|------|
| Chris Stinson | Owner / Developer (personal project, no client) |

---

## Current Tech Stack

| Component | Technology | Notes |
|-----------|-----------|-------|
| Frontend | Pure HTML/CSS/JavaScript | Single-file app (`index.html`), no build process |
| Styling | Custom CSS + CSS Grid | TV-optimized: pure black background, bold white text |
| Fonts | Google Fonts | Orbitron (headers/clock), Share Tech Mono (data rows) |
| Flight Data | AviationStack API (Free tier) | 500 requests/month, HTTP only on free plan |
| CORS Proxy | Multi-proxy fallback system | codetabs.com → corsproxy.io → allorigins.win |
| Airline Logos | Kiwi.com Images API | Free, no key — `https://images.kiwi.com/airlines/64/{IATA}.png` |
| Hosting | GitHub Pages | https://caterpillarkiller.github.io/flightboard/ |
| Version Control | GitHub Desktop | Chris uses GUI — do NOT suggest CLI git |
| Code Editor | VS Code | Primary development environment |

**Brand Colors:**
```
--bg-primary:    #000000  ← Pure black background (TV-optimized, OLED-friendly)
--bg-secondary:  #0a0a0a  ← Deep gray for containers
--accent-yellow: #ffd700  ← Gold — primary accent, active states, highlights
--accent-orange: #ff6b35  ← Orange — secondary accent, warnings, delayed flights
--text-white:    #ffffff  ← All primary text, ALWAYS bold
--text-muted:    #cccccc  ← Secondary text, labels
```

**Environment Variables:** None. All config is hardcoded in `index.html` JavaScript.
- API key is in source code — acceptable for a free-tier personal project
- No `.env` file exists or is needed

**Local dev:** Open `index.html` directly in browser, or `python -m http.server 8000` → http://localhost:8000


**Live URL:** https://caterpillarkiller.github.io/flightboard/

---

## What's Working Right Now

- ✅ Live flight data loads from AviationStack via CORS proxy
- ✅ Multi-proxy fallback (tries 3 proxies automatically if one fails)
- ✅ Departures, Arrivals, and All Flights views
- ✅ Airport code input with Enter key support
- ✅ Geolocation detects user location and suggests 3 nearest airports
- ✅ Clickable airport suggestions auto-load flights
- ✅ Digital clock updates every second with timezone display
- ✅ Auto-refresh toggle (default OFF) — refreshes every 15 min when ON
- ✅ Active/in-flight flights highlighted in gold and sorted to top
- ✅ Flights sorted: active first, then ascending by time
- ✅ Airline names database expanded to 50+ carriers (majors, regional, cargo, international)
- ✅ Airline logos from Kiwi.com (hides gracefully if image fails to load)
- ✅ Status badges color-coded: Scheduled (blue), Boarding (yellow/blink), Departed (green), Delayed (orange), Cancelled (red)
- ✅ Flight status correctly maps AviationStack values to display labels
- ✅ XX flights displayed as "Military / General Aviation" (Charleston-specific context)
- ✅ Mobile responsive layout (grid collapses at 768px)
- ✅ Deployed and loading correctly at base GitHub Pages URL

---

## What Is NOT Done Yet

- ❌ No caching of API results — if the 500/month limit runs out, no fallback data
- ❌ No client-side result caching (considered for future if API limit becomes an issue)

---

## Architecture Notes

- **Single-file app** — all HTML, CSS, and JS lives in `index.html`. This is intentional. Do not suggest splitting into separate files or adding a build process.
- **Client-side only** — no backend, no server, no database. Do not suggest adding any of these.
- **CORS proxy required** — AviationStack free tier uses HTTP. Browsers block mixed HTTP/HTTPS requests, so a proxy rewrites the request. Three proxies are tried in order; the last successful one is remembered for the session.
- **Auto-refresh defaults to OFF** — this is intentional to protect the 100 req/month API quota. Do not change the default.
- **XX flight codes are NOT errors** — Charleston shares runways with Joint Base Charleston. ~16% of operations are military (C-17s and other military aircraft). These show as "XX" in the civilian AviationStack API. This is normal and expected. Do not filter them out.

---

## File Structure

```
flightboard/
  index.html     ← Complete application (renamed from airport-ticker.html in Chat 1) ✅
  .gitignore     ← Excludes .DS_Store
  README.md      ← Basic project description
```

**Deleted:**
- `airport-ticker.html` — renamed to `index.html` so GitHub Pages serves it at the root URL

---

## AviationStack API — Key Details

- **Free tier:** 100 requests/month, HTTP only (no HTTPS on free plan)
- **API Key:** Hardcoded in `index.html` as `AVIATIONSTACK_API_KEY`
- **Endpoint:** `http://api.aviationstack.com/v1/flights`
- **Limit param:** Set to 100 for departures/arrivals, 50 each when fetching "all"

**Flight status values from the API:**
| API Value | Displayed As | Style |
|-----------|-------------|-------|
| `active` or `en-route` | In Flight | Green, highlighted row |
| `landed` | Arrived | Green |
| `cancelled` | Cancelled | Red |
| `incident` | Delayed | Orange |
| `diverted` | Diverted | Orange |
| `scheduled` (default) | Scheduled | Blue |
| Has delay field > 0 | Delayed | Orange |

**Null field handling:**
- `flight.departure.gate` → shows "TBA"
- `flight.aircraft.registration` → shows "N/A"
- `flight.airline.iata` = "XX" → shows "Military / General Aviation"

---

## What Was Tried and Abandoned

| Approach | Why Abandoned |
|----------|--------------|
| `airport-ticker.html` as filename | GitHub Pages requires `index.html` at root to serve the base URL |
| 2-minute auto-refresh (120 sec interval) | Would burn through 100/month API quota in ~7 hours of use; replaced with 15-min interval defaulting to OFF |

---

## What NOT to Suggest (Explicitly Ruled Out)

1. Converting to React, Vue, or any JS framework
2. Adding a build process (Webpack, Vite, etc.)
3. Creating a backend server
4. Upgrading to AviationStack paid plan
5. Using `localStorage` or `sessionStorage`
6. Switching from GitHub Pages to another host
7. CLI git commands — Chris uses GitHub Desktop
8. Automated testing
9. Removing or filtering "XX" / military flights
10. Setting auto-refresh default to ON
11. Adding gradient backgrounds or animated backgrounds — pure black is required for TV display
12. Treating "XX" airline code as an error or unknown

---

## Immediate Next Steps (In Priority Order)

1. **Ask Chris what he'd like to change or add** — no pending tasks from Chat 1; project is in a working, deployed state
2. **Possible future: client-side caching** — if API quota becomes an issue, cache results for 60 seconds to reduce calls
3. **Possible future: add more airports** to the `majorAirports` suggestion array if Chris tracks other locations

---

## How Chris Works (Important for Future Sessions)

- **Not a software engineer by trade** — always provide step-by-step instructions with exact clicks and keystrokes
- Has **VS Code** for editing and **GitHub Desktop** for commits/pushes but does not know HTML
- Uses **Mac** (not confirmed — default to cross-platform instructions)
- Appreciates knowing the *why* behind decisions, not just the *what*
