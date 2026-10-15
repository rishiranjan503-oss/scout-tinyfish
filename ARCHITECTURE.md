# Scout Architecture

## System Overview

Scout is a full-stack job search application with a simple but powerful architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                        User Browser                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Frontend (app.js + index.html + styles.css)          │  │
│  │  - Search form (role, location, seniority, keywords) │  │
│  │  - Results display (job cards, sorting)              │  │
│  │  - Match score ranking                               │  │
│  │  - Browser localStorage (preferences)                │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         ↓ HTTP POST /api/search (JSON)
         ↑ HTTP 200 + results (JSON)
┌─────────────────────────────────────────────────────────────┐
│                    Scout Server (Node.js)                   │
│                    (server.mjs)                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ HTTP Server                                          │  │
│  │  - GET /         → serve index.html                 │  │
│  │  - GET /styles.css → serve styles.css               │  │
│  │  - GET /app.js   → serve app.js                     │  │
│  │  - POST /api/search → handle job search             │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Search Logic (server.mjs)                           │  │
│  │  1. Parse user preferences                          │  │
│  │  2. Build search query                              │  │
│  │  3. Call TinyFish Search API                        │  │
│  │  4. Get discovered URLs (up to 8)                   │  │
│  │  5. Call TinyFish Fetch API                         │  │
│  │  6. Read page content (markdown)                    │  │
│  │  7. Call TinyFish Agent API (parallel x4)           │  │
│  │  8. Extract structured job data                     │  │
│  │  9. Score and deduplicate                           │  │
│  │  10. Return ranked results                          │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         ↓ HTTPS API calls (with X-API-Key header)
         ↑ JSON responses
┌─────────────────────────────────────────────────────────────┐
│                   TinyFish APIs (Cloud)                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 1. TinyFish Search                                   │  │
│  │    https://api.search.tinyfish.ai                    │  │
│  │    Finds career pages based on query                │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 2. TinyFish Fetch                                    │  │
│  │    https://api.fetch.tinyfish.ai                     │  │
│  │    Reads URLs and returns markdown content           │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 3. TinyFish Agent                                    │  │
│  │    https://agent.tinyfish.ai/v1/automation/run       │  │
│  │    Runs automation goal (extract jobs)              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         ↓ HTTPS to job portals and company sites
         ↑ HTML responses
┌─────────────────────────────────────────────────────────────┐
│                Career Pages (External)                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Lever           (jobs.lever.co)                      │  │
│  │ Greenhouse      (boards.greenhouse.io)               │  │
│  │ Ashby           (jobs.ashbyhq.com)                   │  │
│  │ Company sites   (any company careers page)           │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow: A Complete Search

### User Action: Submit search form

```javascript
{
  "role": "Product Designer",
  "location": "San Francisco",
  "seniority": "Mid level",
  "keywords": "Figma, design systems"
}
```

### Step 1: Server receives POST /api/search

```javascript
// server.mjs: receiveSearch()
const preferences = {
  role: "Product Designer",
  location: "San Francisco",
  seniority: "Mid level",
  keywords: "Figma, design systems"
}
```

### Step 2: Build TinyFish Search Query

```javascript
const query = `"Product Designer" "Mid level" "Figma, design systems" 
  jobs careers San Francisco 
  (site:jobs.lever.co OR 
   site:boards.greenhouse.io OR 
   site:jobs.ashbyhq.com OR 
   site:careers.*)`;
```

### Step 3: Call TinyFish Search

```
GET https://api.search.tinyfish.ai?query=...&location=San Francisco
Headers: X-API-Key: [your_key]
Timeout: 30 seconds
```

**Returns:**
```json
{
  "results": [
    { "url": "https://example.com/careers/product-designer", "title": "..." },
    { "url": "https://another.com/jobs", "title": "..." },
    ...  // up to 8 results
  ]
}
```

**Filtering:** Remove social media, non-HTTPS, block YouTube/LinkedIn/Facebook

### Step 4: Call TinyFish Fetch

```
POST https://api.fetch.tinyfish.ai
Headers: X-API-Key: [your_key], Content-Type: application/json
Body: { 
  "urls": [ /* 8 discovered URLs */ ],
  "format": "markdown"
}
Timeout: 60 seconds
```

**Returns:**
```json
{
  "results": [
    { 
      "url": "https://example.com/careers/product-designer",
      "text": "# Careers at ExampleCo\n## Product Designer\nJoin our design team...",
      "status": "success"
    },
    ...
  ]
}
```

**Content:** Markdown formatted page content (up to 3500 chars per page)

### Step 5: Call TinyFish Agent (Parallel × 4)

For each of the first 4 discovered pages:

```
POST https://agent.tinyfish.ai/v1/automation/run
Headers: X-API-Key: [your_key], Content-Type: application/json
Body: {
  "url": "https://example.com/careers/product-designer",
  "goal": "Inspect this careers page and find current job openings relevant to: 
    role 'Product Designer', location 'San Francisco', seniority 'Mid level', 
    keywords 'Figma, design systems'. Return ONLY valid JSON with shape 
    {\"jobs\":[{\"title\":\"...\",\"company\":\"...\",\"location\":\"...\",
    \"seniority\":\"...\",\"datePosted\":\"...\",\"url\":\"https://...\",
    \"description\":\"...\"}]}. Extract no more than 5 real, currently open 
    roles. Use direct apply/job URLs. Never invent openings, fields, links, 
    or dates.\n\nPage content fetched for verification:\n[markdown content]"
}
Timeout: 90 seconds (4 parallel calls)
```

**Returns:**
```json
{
  "result": {
    "jobs": [
      {
        "title": "Product Designer",
        "company": "ExampleCo",
        "location": "San Francisco, CA",
        "seniority": "Mid level",
        "datePosted": "2024-10-14",
        "url": "https://example.com/jobs/product-designer-123",
        "description": "Join our design team to create..."
      }
    ]
  }
}
```

### Step 6: Process Results

```javascript
// Combine all Agent responses
let allJobs = [
  ...page1_agent_result,
  ...page2_agent_result,
  ...page3_agent_result,
  ...page4_agent_result
];

// Score each job
allJobs = allJobs.map(job => {
  const matchScore = calculateScore(job, preferences);
  return { ...job, matchScore };
});

// Deduplicate by URL
const seen = new Set();
const unique = allJobs.filter(job => {
  const key = job.url.toLowerCase().replace(/[?#].*$/, "");
  if (seen.has(key)) return false;
  seen.add(key);
  return true;
});

// Sort by match score
unique.sort((a, b) => b.matchScore - a.matchScore);

// Return top 30
return unique.slice(0, 30);
```

### Step 7: Return to Frontend

```json
{
  "jobs": [
    {
      "title": "Product Designer",
      "company": "ExampleCo",
      "location": "San Francisco, CA",
      "seniority": "Mid level",
      "datePosted": "2024-10-14",
      "url": "https://example.com/jobs/product-designer-123",
      "description": "Join our design team...",
      "source": "example.com",
      "matchScore": 92
    },
    // ... more results, up to 30
  ],
  "sources": ["example.com", "another.com", ...],
  "message": ""
}
```

### Step 8: Frontend Renders Results

```javascript
// app.js receives results
// Displays job cards with:
// - Title and company
// - Location, seniority, date
// - Match score
// - Direct link to apply
// - Ability to sort by match or date
```

## Matching Algorithm

Each job gets scored 0-99 based on these factors:

### 1. Role Relevance (65% weight)

```javascript
// Split role into words (e.g., "Product Designer" → ["product", "designer"])
const roleWords = preferences.role.toLowerCase().split(/\W+/);

// Count how many words match in job content
const matchingWords = roleWords.filter(word => 
  jobContent.includes(word)
).length;

// Score: (matching / total) × 100 × 0.65
const roleScore = (matchingWords / roleWords.length) * 65;
```

### 2. Keywords (20% weight)

```javascript
// If no keywords specified, get 10 points
// If keywords specified, count matches
const keywordWords = preferences.keywords.toLowerCase().split(/[,\s]+/);
const matchingKeywords = keywordWords.filter(word => 
  jobContent.includes(word)
).length;

const keywordScore = keywordWords.length 
  ? (matchingKeywords / keywordWords.length) * 20 
  : 10;
```

### 3. Location Match (10% weight)

```javascript
// Match if:
// - No location preference
// - Location mentions the preferred city
// - Job is remote/anywhere
const locationMatch = !preferences.location || 
  jobContent.includes(preferences.location) ||
  /remote|anywhere|worldwide/i.test(job.location);

const locationScore = locationMatch ? 10 : 0;
```

### 4. Seniority Match (5% weight)

```javascript
// Match if:
// - No seniority preference
// - Seniority level is explicitly mentioned
// - Entry level preference matches "junior/graduate/associate" in content
const seniorityMatch = !preferences.seniority ||
  jobContent.includes(preferences.seniority.toLowerCase()) ||
  (preferences.seniority === "Entry level" && 
   /junior|entry|graduate|associate/i.test(jobContent));

const seniorityScore = seniorityMatch ? 5 : 0;
```

### Final Score

```javascript
const finalScore = roleScore + keywordScore + locationScore + seniorityScore;
const cappedScore = Math.min(99, Math.round(finalScore)); // 0-99
```

## Deduplication Strategy

Jobs are deduplicated by comparing:

1. **Primary:** Full URL (normalized)
   ```javascript
   const key = job.url
     .toLowerCase()
     .replace(/[?#].*$/, "")  // Remove query strings and fragments
     .replace(/\/$/, "");      // Remove trailing slashes
   ```

2. **Fallback:** Title + Company (if URL missing)
   ```javascript
   const key = `${job.title.toLowerCase()}|${job.company.toLowerCase()}`;
   ```

Duplicates are filtered during processing, but if the same URL appears:
- **First appearance** is kept (highest match score wins)
- **Subsequent appearances** are removed

## Error Handling

Scout handles errors gracefully:

### Missing API Key
```
Status: 503 Service Unavailable
Message: "Add a TinyFish API key to TINYFISH_API_KEY on the server, then restart Scout."
```

### Search Timeout
```
Status: 504 Gateway Timeout
Message: "A live search timed out. Please try again."
```

### API Error
```
Status: 502 Bad Gateway
Message: "[Error from TinyFish API]"
```

### No Results
```
Status: 200 OK
Message: "Scout checked live pages, but couldn't verify a matching open role. 
Try changing the role, location, or keywords."
```

### Missing Role
```
Status: 400 Bad Request
Message: "Enter a role to start your search."
```

## Performance Characteristics

| Operation | Timeout | Parallel | Cached? |
|-----------|---------|----------|---------|
| Search discovery | 30s | No | No |
| Fetch pages | 60s | Yes (8 URLs) | No |
| Agent verification | 90s | Yes (4 URLs) | No |
| Score & dedupe | <100ms | N/A | N/A |
| **Total** | **~2 min** | Sequential | **No** |

**Why no caching?**
- Job listings change hourly
- Results are time-sensitive
- Scout prioritizes freshness over speed

## Code Organization

### server.mjs (~250 lines)

```javascript
// 1. HTTP Server Setup (port, static files)
function serveStatic(request, response)

// 2. JSON Parser (safe, bounded)
async function readJson(request)

// 3. TinyFish API Wrapper (error handling, timeouts)
async function tinyfish(url, options, timeout)

// 4. Job Extraction (parse Agent response, normalize data)
function extractJobs(agentResponse, fallbackUrl)

// 5. Scoring Algorithm (role, keywords, location, seniority)
function scoreJob(job, preferences)

// 6. Deduplication (URL-based, score-ranking)
function uniqueJobs(jobs, preferences)

// 7. Main Search Logic (orchestration)
async function searchJobs(preferences)

// 8. Server Routes (/api/search, /api/status, static files)
const server = createServer((request, response) => {
  if (request.method === "POST" && request.url === "/api/search") {
    // Handle search
  }
  if (request.method === "GET" && request.url === "/api/status") {
    // Check if configured
  }
  // Serve static files
})
```

### app.js (~200 lines)

```javascript
// 1. Form Handling
// - Submit event listener
// - Parse preferences
// - Save to localStorage

// 2. Display Functions
// - showStatus() – loading/error states
// - renderResults() – job cards
// - escapeHtml() – security

// 3. Sorting & Filtering
// - Sort by match score (default)
// - Sort by date posted
// - Render on each change

// 4. Accessibility
// - aria-busy attribute
// - aria-live region
// - Semantic HTML
// - Keyboard navigation
```

### index.html (~150 lines)

```html
<!-- 1. Header with branding -->
<!-- 2. Hero section -->
<!-- 3. Search panel (form) -->
<!-- 4. Results panel (live region) -->
<!-- 5. How it works section (3 steps) -->
<!-- 6. Share strip (LinkedIn, X) -->
<!-- 7. Footer -->
```

### styles.css (~400 lines)

```css
/* 1. CSS Variables (colors, fonts) */
/* 2. Base styles (reset, body) */
/* 3. Header styles */
/* 4. Hero section */
/* 5. Search panel */
/* 6. Results panel */
/* 7. Job cards */
/* 8. Status states (loading, error, empty) */
/* 9. Responsive breakpoints (768px, 620px) */
/* 10. Accessibility (prefers-reduced-motion) */
```

## Deployment Checklist

- [x] Node.js 20+ required
- [x] No external dependencies (npm packages)
- [x] Environment variable: `TINYFISH_API_KEY`
- [x] Environment variable: `PORT` (optional, defaults to 3000)
- [x] Static file serving (HTML, CSS, JS)
- [x] CORS not needed (same origin)
- [x] No database required (stateless)
- [x] No frontend build step required
- [x] Error handling for network failures
- [x] Bounded request size (12KB max)

## Future Improvements (Out of Scope)

- [ ] Caching with Redis (for user analytics)
- [ ] User accounts and saved searches
- [ ] Email alerts when new jobs posted
- [ ] Browser extension
- [ ] Mobile app
- [ ] More job boards (Indeed, Glassdoor, etc.)
- [ ] Salary range extraction
- [ ] Company insights (size, funding, industry)
- [ ] Application tracking
- [ ] Resume matching

Scout's current scope is intentionally narrow: discover live jobs, verify with AI, match to user preferences, deliver clean results. That's what the workshop requires, and it's what actually works.

---

**Version:** 1.0  
**Last Updated:** October 2024  
**Built with:** Node.js, TinyFish APIs, Vanilla JS/CSS
