# Flightboard Codebase Audit Report

**Date:** March 13, 2026
**Audited by:** Claude Code
**Codebase:** [caterpillarkiller/flightboard](https://github.com/caterpillarkiller/flightboard)
**Deployed at:** flightboard.chs-consulting.com

---

## Executive Summary

The Flightboard is a single-file (`index.html`) Solari-style flight information board that fetches real-time flight data from the AviationStack API and displays it with a retro split-flap board aesthetic. The application has **2 critical security vulnerabilities**, **7 high-severity issues**, and multiple medium/low concerns that should be addressed.

| Severity | Count |
|----------|-------|
| Critical | 2 |
| High | 7 |
| Medium | 9 |
| Low | 5 |
| **Total** | **23** |

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

### 3. DOM-Based XSS via Flight Data
**Location:** `index.html` lines 1076–1093 (`renderFlights` function)

Flight data from the API (airline names, routes, flight numbers) is inserted directly into `innerHTML` without sanitization. A compromised API could inject arbitrary JavaScript.

**Fix:** Use `textContent` for plain text values, or DOM creation methods (`createElement`, `appendChild`). If HTML is necessary, use DOMPurify.

---

### 4. Missing Content Security Policy (CSP)
**Location:** Entire application

No CSP header is set to restrict inline scripts, external resources, or script injection. All application code is inline JavaScript.

**Fix:** Add a strict CSP header:
```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' https://images.kiwi.com; font-src https://fonts.gstatic.com;
```

---

### 5. Uncontrolled Third-Party CORS Proxy Dependency
**Location:** `index.html` lines 663–667

The application relies on three third-party CORS proxies (`api.codetabs.com`, `corsproxy.io`, `api.allorigins.win`) that are not controlled by the application owner. A compromised proxy could return malicious data, steal the API key, or serve malicious HTML/JavaScript.

**Fix:** Implement a backend proxy server under your own control (e.g., a Cloudflare Worker or AWS Lambda).

---

### 6. No Error Handling for JSON Parsing
**Location:** `index.html` line 873
**Code:** `const data = JSON.parse(await response.text());`

No try-catch around JSON.parse(). Malformed JSON from any proxy crashes the function silently.

**Fix:**
```javascript
let data;
try {
    data = JSON.parse(await response.text());
} catch (e) {
    throw new Error(`Invalid JSON response: ${e.message}`);
}
```

---

### 7. No Client-Side Rate Limiting
**Location:** `index.html` lines 1161–1180

Users can click "Load" repeatedly, potentially exhausting the monthly API quota (100–500 calls/month on AviationStack's free tier) quickly.

**Fix:** Disable the Load button for at least 10 seconds after each click; track call count and warn users when approaching limits.

---

### 8. No Response Caching
**Location:** Entire application

Every page load or manual refresh fetches fresh data. No `localStorage` or `sessionStorage` caching, so if the API fails the user sees nothing.

**Fix:** Cache flight data in `localStorage` with a 5-minute TTL and display a "cached" indicator. Show the last-updated timestamp in the UI.

---

### 9. Race Condition in Auto-Refresh
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

### 10. Unvalidated Airport Code Input
Only a minimum-length check is performed. No character-set validation allows injection of unexpected values.

**Fix:** Validate with `/^[A-Z]{3,4}$/` before making any API calls.

---

### 11. Image Loading Without Validation
Airline logo URLs are constructed from unvalidated API data: `https://images.kiwi.com/airlines/64/${flight.airline}.png`. The `alt` attribute is also unescaped.

**Fix:** Validate airline code format with `/^[A-Z0-9]{1,3}$/` before building the URL.

---

### 12. User Geolocation Displayed Publicly
The user's raw GPS coordinates are shown in the UI, which could expose location to onlookers or screenshots.

**Fix:** Show only an approximate city name rather than coordinates; add a privacy notice.

---

### 13. Partial API Failure Not Handled Gracefully
If the departures call succeeds but arrivals fails (or vice versa), the whole request is treated as failed.

**Fix:** Use `Promise.allSettled()` and display whatever partial data was successfully retrieved.

---

### 14. Incomplete "Landed" Flight State Logic
The `Landed` status detection relies on `previousFlightData`, which can become stale if API fetches fail.

**Fix:** Add timestamps to track state transitions; implement an explicit state machine for flight statuses.

---

### 15. Time Display Lacks Timezone Context
Flight times are shown without any timezone indicator, which can be confusing.

**Fix:** Parse and retain timezone from the API response; display a timezone abbreviation (e.g., `15:30 EST`).

---

### 16. Inefficient Full DOM Re-render on Each Update
The entire board's `innerHTML` is replaced on every data update, re-triggering animations and causing potential visual flickering.

**Fix:** Use DOM methods or a diffing strategy so only changed flight rows are updated.

---

### 17. Mobile Layout Has Only One Breakpoint
Only a 768 px breakpoint is defined, with no consideration for small phones or large desktops.

**Fix:** Add breakpoints at 480 px, 768 px, 1024 px, and 1500 px.

---

### 18. API Rate Limit Responses (HTTP 429) Not Handled
No logic exists to detect or recover from `429 Too Many Requests` responses.

**Fix:** Check for 429 status and retry with exponential backoff; parse `Retry-After` headers.

---

## Low Severity Issues

### 19. No Accessibility (ARIA) Support
No ARIA labels, roles, or states on interactive elements. Toggle switches and the flight board are not keyboard-navigable. Status is communicated by color only.

**Fix:** Add ARIA labels, `role="status"` for live updates, and text alternatives for all color indicators.

---

### 20. No Transpilation for Older Browsers
Modern JS features (optional chaining `?.`, nullish coalescing `??`) are used without polyfills or transpilation.

**Fix:** Document minimum browser requirements, or add a Babel transpilation step.

---

### 21. Fragile Timezone Abbreviation Parsing
`toLocaleTimeString('en-US', { timeZoneName: 'short' }).split(' ')[2]` may fail silently in non-US locales.

**Fix:** Use `Intl.DateTimeFormat` directly with explicit options and add a try-catch.

---

### 22. Timer Not Cleared on Page Unload
The auto-refresh `setInterval` timer is not cleared when the page unloads.

**Fix:** Add `window.addEventListener('beforeunload', stopAutoRefresh)`.

---

### 23. `apiCallCount` Tracked but Never Used
`apiCallCount` is incremented but never surfaced to the user or checked against a limit.

**Fix:** Either display the count in the UI with a warning threshold, or remove the dead code.

---

## Priority Action Plan

### Phase 1 — Immediate (Critical Security)
1. Rotate and remove the hardcoded AviationStack API key
2. Switch the API endpoint to HTTPS
3. Set up a backend proxy (Cloudflare Worker recommended for a static site) to protect the API key and eliminate reliance on third-party CORS proxies
4. Sanitize all API data before inserting into the DOM

### Phase 2 — Short-term (High Priority)
5. Add rate limiting on the Load button
6. Implement `localStorage` caching with a 5-minute TTL
7. Add an `isLoading` guard to prevent concurrent requests
8. Add airport code format validation
9. Add a try-catch around JSON.parse()

### Phase 3 — Medium-term (Quality & Reliability)
10. Add CSP headers (deploy via server config or meta tag)
11. Handle partial API failures with `Promise.allSettled()`
12. Show timezone with flight times
13. Improve mobile responsive breakpoints
14. Handle HTTP 429 rate-limit responses

---

## Summary

The Flightboard is a well-designed personal project with a polished retro aesthetic. The most urgent issue is the **exposed API key**, which should be rotated immediately and moved to a backend proxy. Switching to **HTTPS** and **sanitizing DOM insertions** are the other critical fixes. The remaining issues are quality-of-life improvements that would make the application more reliable and maintainable.
