# Flightboard Codebase Audit Report

**Date:** March 14, 2026 (updated — fresh independent review)
**Audited by:** Claude Code
**Codebase:** [caterpillarkiller/flightboard](https://github.com/caterpillarkiller/flightboard)
**Deployed at:** flightboard.chs-consulting.com

---

## Executive Summary

The Flightboard is a single-file (`index.html`) Solari-style flight information board that fetches real-time flight data from the AviationStack API and displays it with a retro split-flap board aesthetic. The application has **2 critical security vulnerabilities**, **9 high-severity issues**, and multiple medium/low concerns.

This report supersedes the March 13 version and adds 3 newly identified vulnerabilities (issues #4, #5, #6 below).

| Severity | Count |
|----------|-------|
| Critical | 2 |
| High | 9 |
| Medium | 9 |
| Low | 5 |
| **Total** | **25** |

---

## Critical Issues

### 1. Hardcoded API Key Exposed in Client-Side Code
**Location:** `index.html` line 654
**Code:** `const AVIATIONSTACK_API_KEY = '8d972ea55bf625f3baf55a32910ec047';`

The AviationStack API key is embedded directly in client-side JavaScript, visible to anyone viewing page source. This key can be used by malicious actors to make unauthorized API calls, exhaust rate limits, and incur unexpected charges. **This key should be considered compromised.**

**Fix:** Move the API key to a backend server/proxy. Create a secure endpoint that validates requests and uses environment variables to store the key.

---

### 2. Insecure HTTP API Endpoint
**Location:** `index.html` line 849
**Code:** `const baseUrl = 'http://api.aviationstack.com/v1/flights';`

The API endpoint uses plain HTTP, enabling Man-in-the-Middle (MITM) attacks. The API key and flight data are transmitted in plaintext.

**Fix:** Change to `https://api.aviationstack.com/v1/flights`.

---

## High Severity Issues

### 3. DOM XSS via User-Supplied Airport Code (NEW)
**Location:** `index.html` line 1129
**Code:**
```javascript
board.innerHTML = `<div class="loading"><div class="spinner"></div><div>Loading ${airportCode.toUpperCase()}...</div></div>`;
```

`airportCode` is taken directly from user input (`document.getElementById('airportCode').value.trim()`) and inserted verbatim into `innerHTML`. Typing `<img src=x onerror=alert(document.cookie)>` in the airport code field triggers immediate JavaScript execution. This is a DOM-based XSS vulnerability exploitable by the user themselves or via a crafted URL if the page ever accepts query parameters.

**Fix:** Use `textContent` for the loading message, or sanitize `airportCode` with a `/^[A-Z0-9]{3,4}$/` regex before use.

---

### 4. XSS in Error Handler via Unsanitized Error Message (NEW)
**Location:** `index.html` line 1154
**Code:**
```javascript
board.innerHTML = `<div class="error">
    <strong>Failed to load flight data</strong><br><br>
    ${error.message}<br><br>
    ...
</div>`;
```

`error.message` is inserted directly into `innerHTML`. A compromised CORS proxy returning a response that causes `JSON.parse` to fail with an HTML-containing error message, or a network-level attacker crafting an error response, could inject arbitrary HTML/JavaScript here. Combined with the reliance on uncontrolled third-party proxies (issue #7), this is a realistic attack path.

**Fix:** Use `textContent` to set the error message, or escape HTML characters before insertion.

---

### 5. Attribute Injection via Unescaped `alt` Attribute (NEW)
**Location:** `index.html` line 1085
**Code:**
```javascript
<img src="${logoUrl}" alt="${flight.airline}" class="airline-logo" onerror="this.style.display='none'">
```

`flight.airline` comes from API data and is inserted unescaped into an HTML attribute. If AviationStack (or a compromised proxy) returns a value containing `"`, the attribute context breaks, allowing attribute injection. For example, `flight.airline = 'XX" onmouseover="alert(1)'` would add an event handler to the `<img>` element.

**Fix:** Validate `flight.airline` with `/^[A-Z0-9]{1,3}$/` before use, or use `createElement` and set attributes via DOM APIs.

---

### 6. DOM-Based XSS via Flight Data from API
**Location:** `index.html` lines 1076–1093 (`renderFlights` function)

Flight data from the API (airline names, route codes, flight numbers, gate, status, and formatted route HTML from `formatRoute()`) is inserted directly into `innerHTML` without sanitization. Specifically:
- `name` (`airlineNames[flight.airline] || flight.airline`) — falls back to raw API value
- `flight.flightNumOnly`, `flight.time`, `flight.gate`, `flight.status` — all raw API strings
- `routeHTML` from `formatRoute()` which embeds `flight.depIata` and `flight.arrIata` directly

A compromised API or CORS proxy could inject arbitrary JavaScript through any of these fields.

**Fix:** Use `textContent` for plain text values, or DOM creation methods (`createElement`, `appendChild`). If HTML is necessary, use DOMPurify.

---

### 7. Missing Content Security Policy (CSP)
**Location:** Entire application

No CSP header is set to restrict inline scripts, external resources, or script injection. All application code is inline JavaScript.

**Fix:** Add a strict CSP header:
```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' https://images.kiwi.com; font-src https://fonts.gstatic.com;
```

---

### 8. Uncontrolled Third-Party CORS Proxy Dependency
**Location:** `index.html` lines 663–667

The application relies on three third-party CORS proxies (`api.codetabs.com`, `corsproxy.io`, `api.allorigins.win`) that are not controlled by the application owner. A compromised proxy could return malicious data, steal the API key, or serve malicious HTML/JavaScript. Combined with the `innerHTML` XSS issues (#3, #4, #6), a malicious proxy response is a realistic end-to-end exploit path.

**Fix:** Implement a backend proxy server under your own control (e.g., a Cloudflare Worker or AWS Lambda).

---

### 9. No Error Handling for JSON Parsing
**Location:** `index.html` line 873
**Code:** `const data = JSON.parse(await response.text());`

No try-catch around `JSON.parse()`. Malformed JSON from any proxy crashes the function silently, and as noted in issue #4, a malicious error message could reach `innerHTML`.

**Fix:**
```javascript
let data;
try {
    data = JSON.parse(await response.text());
} catch (e) {
    throw new Error(`Invalid JSON response from proxy`); // Do NOT pass e.message to innerHTML
}
```

---

### 10. No Client-Side Rate Limiting
**Location:** `index.html` lines 1161–1180

Users can click "Load" repeatedly, potentially exhausting the monthly API quota (100–500 calls/month on AviationStack's free tier) quickly.

**Fix:** Disable the Load button for at least 10 seconds after each click; track call count and warn users when approaching limits.

---

### 11. No Response Caching
**Location:** Entire application

Every page load or manual refresh fetches fresh data. No caching means if the API fails the user sees nothing, and quota burns faster.

**Fix:** Cache flight data in memory with a 5-minute TTL and display a "last updated" timestamp in the UI. (Note: `localStorage` is explicitly ruled out by project constraints.)

---

### 12. Race Condition in Auto-Refresh
**Location:** `index.html` lines 1173–1180

If `loadFlights()` takes longer than the refresh interval, concurrent requests can overlap, duplicating API calls.

**Fix:** Add an `isLoading` guard:
```javascript
let isLoading = false;
async function loadFlights() {
    if (isLoading) return;
    isLoading = true;
    try { /* ... */ } finally { isLoading = false; }
}
```

---

## Medium Severity Issues

### 13. Unvalidated Airport Code Input
Only a minimum-length check is performed. No character-set validation allows injection of unexpected values into API queries and (critically) into `innerHTML` (see issue #3).

**Fix:** Validate with `/^[A-Z]{3,4}$/` before making any API calls or inserting into the DOM.

---

### 14. Image Loading Without Validation
Airline logo URLs are constructed from unvalidated API data: `` `https://images.kiwi.com/airlines/64/${flight.airline}.png` ``. The `alt` attribute is also unescaped (see issue #5).

**Fix:** Validate airline code format with `/^[A-Z0-9]{1,3}$/` before building the URL.

---

### 15. User Geolocation Displayed Publicly
The user's raw GPS coordinates are shown in the UI (line 812, using `textContent` — no XSS risk, but a privacy concern). The display format `📍 33.9°, -80.1°` could expose location to onlookers or screenshots.

**Fix:** Show only an approximate city name rather than coordinates; add a privacy notice.

---

### 16. Partial API Failure Not Handled Gracefully
If the departures call succeeds but arrivals fails (or vice versa), the whole request is treated as failed.

**Fix:** Use `Promise.allSettled()` and display whatever partial data was successfully retrieved.

---

### 17. Incomplete "Landed" Flight State Logic
The `Landed` status detection relies on `previousFlightData`, which can become stale if API fetches fail.

**Fix:** Add timestamps to track state transitions; implement an explicit state machine for flight statuses.

---

### 18. Time Display Lacks Timezone Context
Flight times are shown without any timezone indicator, which can be confusing.

**Fix:** Parse and retain timezone from the API response; display a timezone abbreviation (e.g., `15:30 EST`).

---

### 19. Inefficient Full DOM Re-render on Each Update
The entire board's `innerHTML` is replaced on every data update, re-triggering animations and causing potential visual flickering.

**Fix:** Use DOM methods or a diffing strategy so only changed flight rows are updated.

---

### 20. Mobile Layout Has Only One Breakpoint
Only a 768 px breakpoint is defined, with no consideration for small phones or large desktops.

**Fix:** Add breakpoints at 480 px, 768 px, 1024 px, and 1500 px.

---

### 21. API Rate Limit Responses (HTTP 429) Not Handled
No logic exists to detect or recover from `429 Too Many Requests` responses.

**Fix:** Check for 429 status and retry with exponential backoff; parse `Retry-After` headers.

---

## Low Severity Issues

### 22. No Accessibility (ARIA) Support
No ARIA labels, roles, or states on interactive elements. Toggle switches and the flight board are not keyboard-navigable. Status is communicated by color only.

**Fix:** Add ARIA labels, `role="status"` for live updates, and text alternatives for all color indicators.

---

### 23. No Transpilation for Older Browsers
Modern JS features (optional chaining `?.`, nullish coalescing `??`) are used without polyfills or transpilation.

**Fix:** Document minimum browser requirements, or add a Babel transpilation step.

---

### 24. Fragile Timezone Abbreviation Parsing
`toLocaleTimeString('en-US', { timeZoneName: 'short' }).split(' ')[2]` may fail silently in non-US locales.

**Fix:** Use `Intl.DateTimeFormat` directly with explicit options and add a try-catch.

---

### 25. Timer Not Cleared on Page Unload
The auto-refresh `setInterval` timer is not cleared when the page unloads.

**Fix:** Add `window.addEventListener('beforeunload', stopAutoRefresh)`.

---

### 26. `apiCallCount` Tracked but Never Used
`apiCallCount` is incremented (line 862) but never surfaced to the user or checked against a limit.

**Fix:** Either display the count in the UI with a warning threshold, or remove the dead code.

---

## Priority Action Plan

### Phase 1 — Immediate (Critical Security)
1. Rotate and remove the hardcoded AviationStack API key (Issue #1)
2. Switch the API endpoint to HTTPS (Issue #2)
3. Set up a backend proxy (Cloudflare Worker recommended for a static site) to protect the API key and eliminate reliance on third-party CORS proxies (Issues #1, #2, #8)
4. Sanitize user input with `/^[A-Z]{3,4}$/` before inserting into the DOM — fixes Issue #3 (user input XSS in loading message) and Issue #13
5. Sanitize all API data before inserting into the DOM — fixes Issues #4, #5, #6

### Phase 2 — Short-term (High Priority)
6. Add try-catch around `JSON.parse()` and never pass `error.message` to `innerHTML` (Issues #4, #9)
7. Add rate limiting on the Load button (Issue #10)
8. Add an `isLoading` guard to prevent concurrent requests (Issue #12)

### Phase 3 — Medium-term (Quality & Reliability)
9. Add CSP headers via meta tag (Issue #7)
10. Handle partial API failures with `Promise.allSettled()` (Issue #16)
11. Show timezone with flight times (Issue #18)
12. Improve mobile responsive breakpoints (Issue #20)
13. Handle HTTP 429 rate-limit responses (Issue #21)

---

## Summary

The Flightboard is a well-designed personal project with a polished retro aesthetic. The most urgent issue is the **exposed API key**, which should be rotated immediately. **Three additional XSS vulnerabilities** were identified beyond the original report:

- User-supplied airport code inserted unsanitized into `innerHTML` (line 1129)
- `error.message` from potentially-malicious proxies inserted into `innerHTML` (line 1154)
- Unescaped `flight.airline` in an HTML `alt` attribute enabling attribute injection (line 1085)

Combined with the reliance on uncontrolled third-party CORS proxies, the XSS issues represent a realistic end-to-end attack chain: a compromised proxy → crafted JSON → XSS in error handler or flight render. Input validation on the airport code field (3–4 uppercase letters only) would eliminate issue #3 with a single line of code.
