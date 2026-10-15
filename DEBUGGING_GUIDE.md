# Scout Debugging Guide

## Quick Troubleshooting

### 🔴 Red Error Banner at Top

**What it means**: Something is critically wrong

**What to do**:
1. Read the error message
2. Follow the fix below
3. Refresh browser

---

## Common Errors & Fixes

### ❌ "TinyFish API not configured"

**Cause**: `TINYFISH_API_KEY` not in `.env`

**Fix**:
```bash
# 1. Create .env file
Copy-Item .env.example .env

# 2. Edit .env and add your key
# TINYFISH_API_KEY=sk_live_your_actual_key_here

# 3. Restart server
node --env-file=.env server.mjs

# 4. Refresh browser
```

**Verify**: Console shows `✓ TinyFish API configured`

---

### ❌ "Cannot reach API server"

**Cause**: Server not running on port 3000

**Fix**:
```bash
# 1. Check server is running
# Terminal should show: "Scout is ready at http://localhost:3000"

# 2. If not running, start it:
node --env-file=.env server.mjs

# 3. Refresh browser
# http://localhost:3000
```

**Verify**: Console shows `✓ TinyFish API configured`

---

### ❌ "Missing DOM elements: ..."

**Cause**: `index.html` is corrupted or incomplete

**Fix**:
```bash
# Verify index.html is correct
# Should have:
# - <form id="search-form">
# - <div id="status-message">
# - <div id="results-list">
# - <div class="nav-tab" data-tab="search">
# etc.

# Compare with backup or re-download
```

**Verify**: All elements load, no red banner

---

### ❌ "Search failed: ..."

**Cause**: API error during search

**Fix**:
1. Open DevTools (F12)
2. Click search
3. Look at console error message
4. Check what went wrong:

| Error | Cause | Fix |
|-------|-------|-----|
| `401 Unauthorized` | Invalid API key | Check `.env` file |
| `429 Too Many Requests` | Rate limited | Wait 5 minutes and retry |
| `500 Internal Server Error` | Server error | Check server console |
| `TimeoutError` | Search took too long | Retry with different keywords |

---

### ✅ "No matching opportunities found"

**This is OK** - means search worked but no results

**Fix**:
- Try broader keywords (remove specific tech stack)
- Try different location
- Try different role title

---

## Console Debugging

### ✅ On Startup

Open DevTools (F12 → Console tab)

Should see:
```
Scout app.js loading...
============================================================
SCOUT × TINYFISH - INITIALIZING
============================================================
✓ All DOM elements found
✓ Tab navigation initialized
✓ Preferences loaded from localStorage
✓ TinyFish API configured and ready
✓ Scout initialized successfully
============================================================
```

If you see ❌ errors instead of ✓, scroll up to find which step failed.

---

### Check DOM Elements

Type in console:
```javascript
Object.entries(DOM_ELEMENTS).forEach(([k, v]) => console.log(k, v ? '✓' : '❌'));
```

Should show all ✓

---

### Check API Status

Type in console:
```javascript
fetch("/api/status").then(r => r.json()).then(console.log);
```

Should show:
```
{ configured: true }
```

If `false`, add API key to `.env`

---

### Test Search API

Type in console:
```javascript
fetch("/api/search", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    role: "Software Engineer",
    location: "Bengaluru",
    seniority: "Entry level",
    keywords: "Python"
  })
}).then(r => r.json()).then(console.log);
```

Should return job objects with `title`, `company`, `location`, etc.

---

### Check Tab Navigation

Type in console:
```javascript
document.querySelectorAll('.nav-tab').forEach(t => console.log(t.dataset.tab, '✓'));
```

Should show: `search ✓`, `feed ✓`, `radar ✓`, `docs ✓`

---

### Check Form Elements

Type in console:
```javascript
const form = document.querySelector("#search-form");
console.log("Role input:", form.elements.namedItem("role").value);
console.log("Location input:", form.elements.namedItem("location").value);
```

Should show form field values

---

## Network Debugging

### Check API Calls

1. Open DevTools (F12)
2. Go to "Network" tab
3. Enter search
4. Look for requests:
   - `POST /api/search` → Should be 200 OK
   - Response should have `{ jobs: [...], sources: [...] }`

**If 403 or 401**:
- API key invalid
- Check `.env` file

**If 502 or 504**:
- API server error
- Check server console
- Restart server

---

## File Verification

### Check index.html

Required IDs to have:
```html
<form id="search-form">
<div id="status-message">
<div id="results-list">
<div id="results-footer">
<div id="result-count">
<select id="sort-select">
<div id="telemetry-feed">
<div id="radar-discovered">
<div id="radar-search-bar">
<div id="radar-fetched">
<div id="radar-fetch-bar">
<div id="radar-verified">
<div id="radar-agent-bar">
<button id="trigger-agent-btn">
<button id="share-x-btn">
<button id="share-li-btn">
```

Missing any? → Reload from backup

---

### Check app.js Loaded

Type in console:
```javascript
console.log(typeof DOM_ELEMENTS);
// Should print: "object"
```

If undefined: app.js not loaded, check Network tab

---

### Check server.mjs Running

Type in console:
```javascript
fetch("/api/status").then(r => console.log(r.status));
// Should print: 200
```

If error: Server not running

---

## Step-by-Step Debug Process

### 1. Is server running?
```bash
# Terminal shows: "Scout is ready at http://localhost:3000"
# If not, run: node --env-file=.env server.mjs
```

### 2. Can you reach http://localhost:3000?
```
# Should see Scout UI
# If 404, server not running
```

### 3. Open DevTools (F12)
```
# Console tab
# Should see: "✓ Scout initialized successfully"
# If not, scroll up to find error
```

### 4. Check for error banner
```
# Red bar at top?
# Read error message, follow fix above
```

### 5. Test each feature
```javascript
// Tab navigation
document.querySelector('[data-tab="search"]').click();

// Form submission
document.querySelector("#search-form").dispatchEvent(new Event('submit'));

// API call
fetch("/api/search", { /* ... */ });
```

---

## Performance Issues

### App is Slow to Load

**Cause**: Tailwind CSS from CDN

**What to do**: This is normal, first load ~2-3 seconds

**Second load**: Should be <500ms (cached)

---

### Search Takes Long Time

**Cause**: TinyFish APIs take 30-60 seconds

**This is expected**:
- Search API: ~30 seconds to discover
- Fetch API: ~60 seconds to read pages
- Agent API: ~90 seconds to verify

**Total**: 2-3 minutes for complete search

---

## Browser Compatibility

| Browser | Works? | Notes |
|---------|--------|-------|
| Chrome 90+ | ✅ | Full support |
| Firefox 88+ | ✅ | Full support |
| Safari 14+ | ✅ | Full support |
| Edge 90+ | ✅ | Full support |
| Mobile Chrome | ✅ | Responsive |
| Mobile Safari | ✅ | Responsive |
| IE 11 | ❌ | Too old |
| Chrome 80 | ❌ | Tailwind CSS requires ES6 |

---

## Still Not Working?

### Collect Debugging Info

1. **Screenshot of error banner** (if any)
2. **Console output** (F12 → Console → Copy all)
3. **Network errors** (F12 → Network → See red requests)
4. **Search query** (what role/location/keywords)
5. **Error message** (from red banner or console)

### Share With Support

Create issue with:
- Browser + version
- Steps to reproduce
- Error message
- Console logs
- Screenshots

---

## Emergency Reset

If everything breaks:

```bash
# 1. Stop server (Ctrl+C)

# 2. Backup broken files
mkdir backup
mv index.html backup/
mv app.js backup/

# 3. Restore from original
# (Re-download or copy from version control)

# 4. Start fresh
node --env-file=.env server.mjs

# 5. Test
# http://localhost:3000
```

---

## Getting Help

### Check Logs

**Server console** (where you ran `node`):
- Shows API calls
- Shows errors
- Shows port listening

**Browser console** (F12):
- Shows JavaScript errors
- Shows API responses
- Shows initialization steps

### Ask Questions

Provide:
1. What you were trying to do
2. What error you got
3. Console output (copy-paste)
4. Steps to reproduce

---

## Quick Command Reference

```bash
# Check Node.js version
node --version

# Start server
node --env-file=.env server.mjs

# Start with port override
PORT=3001 node --env-file=.env server.mjs

# Test API endpoint
curl http://localhost:3000/api/status

# Stop server
Ctrl + C
```

---

**Still stuck?** Check the console - it will tell you exactly what's wrong! 🔍
