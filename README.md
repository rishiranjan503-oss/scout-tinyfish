# Scout — Your Job Search, on Autopilot

Scout is a TinyFish-powered job and internship finder built for the "AI That Clicks" workshop at Galgotias University. Set a role, location, experience level, and optional keywords; Scout searches live company career pages, reads the content, and asks a TinyFish Agent to verify and structure matching openings. Results are ranked, deduplicated, and linked to their source pages.

## Features

- **Live job discovery** – Finds openings from company careers pages and major job boards (Lever, Greenhouse, Ashby)
- **Personalized matching** – Filters by role, location, seniority level, and keywords you care about
- **Real data only** – TinyFish Agent checks live page content for current openings; Scout does not use sample listings
- **Direct apply links** – Results link straight to the original job posting
- **Deduplicated results** – Same job listed on multiple sites? Scout shows it once, ranked by match score
- **Fresh searches** – Each search discovers pages again; confirm availability and details on the employer's page before applying

## How Scout Works

Scout uses three TinyFish APIs working together:

1. **TinyFish Search** – Discovers career pages based on your preferences
2. **TinyFish Fetch** – Reads page content from discovered URLs
3. **TinyFish Agent** – Inspects up to three distinct discovered pages and returns structured role details for matching

Scout deduplicates results and ranks them against the selected preferences. Automated extraction can miss context; use the original employer link to confirm each role is still open and that its details fit.

See [TINYFISH_USAGE.md](./TINYFISH_USAGE.md) for a detailed breakdown of how each API is used.

## Application assistant

Each result offers **Prepare application**. The user enters only the profile details they choose to share, reviews an explicit consent notice, and confirms before those values are sent to TinyFish. TinyFish Agent may fill clearly matching visible fields in its hosted browser, but is instructed never to submit, sign in, upload files, select checkboxes, or complete legal/sensitive fields. Scout shows a summary and a link to the employer page; the hosted browser itself cannot be transferred into the user's browser, so the user must review and continue on the employer site. Application details are not saved in Scout's browser storage.

Only direct job links returned by a recent Scout search can be used, and they expire after 30 minutes. This application helper is an opt-in convenience, not a guarantee of a complete or accepted application.

## Run Locally

Requires **Node.js 20 or newer**. No npm packages needed.

1. Copy `.env.example` to `.env`:
   ```powershell
   Copy-Item .env.example .env
   ```

2. Add your TinyFish API key to `.env`:
   - Get your key from [agent.tinyfish.ai/api-keys](https://agent.tinyfish.ai/api-keys)
   - Paste it into `TINYFISH_API_KEY=...`

3. Start Scout:
   ```powershell
   node --env-file=.env server.mjs
   ```
   Or if you prefer to set the environment variable first:
   ```powershell
   $env:TINYFISH_API_KEY = "your_key_here"
   node server.mjs
   ```

4. Open your browser to [http://localhost:3000](http://localhost:3000)

**Note:** Your TinyFish API key stays on the server and is never sent to the browser.

## Deployment

Scout runs as a standalone Node.js HTTP server. To deploy:

1. Ensure Node.js 20+ is installed on your server
2. Copy all files to your server
3. Set `TINYFISH_API_KEY` environment variable
4. Set `PORT` environment variable (defaults to 3000)
5. Run: `node server.mjs`

Example systemd service file for Linux:
```ini
[Unit]
Description=Scout - TinyFish Job Finder
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/scout
Environment="TINYFISH_API_KEY=your_key_here"
Environment="PORT=3000"
ExecStart=/usr/bin/node server.mjs
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## Share Your Build

Scout includes built-in share links on the "How it works" section:

- **LinkedIn:** Click "Share on LinkedIn" to open the composer. Add your demo link and tag TinyFish
- **X (Twitter):** Click "Post on X" to open the composer. Add your demo link and tag @TinyFishAI

Both tags are required for workshop submission. The app opens the X post composer with a draft; LinkedIn opens a share dialog for the current page URL. Add your public demo link and the required TinyFish tag to both posts, then publish them yourself:
- LinkedIn: Tag "TinyFish" in the post
- X: Tag "@TinyFishAI" in the post

Posting is a manual submission step; opening a share link does not publish a post.

## Architecture

- **Frontend:** Vanilla JavaScript + CSS (accessible, responsive, no dependencies)
- **Backend:** Node.js HTTP server (no Express or frameworks needed)
- **APIs:** TinyFish Search, Fetch, and Agent
- **Browser storage:** Saves user preferences to localStorage (no backend database needed)

## Endpoints

### GET /api/status
Returns whether TinyFish API is configured.

```json
{ "configured": true }
```

### POST /api/search
Searches for matching jobs based on preferences.

**Request:**
```json
{
  "role": "Product Designer",
  "location": "San Francisco",
  "seniority": "Mid level",
  "keywords": "Figma, climate tech"
}
```

**Illustrative response (values are examples, not live listings):**
```json
{
  "jobs": [
    {
      "title": "Senior Product Designer",
      "company": "climate-tech-startup",
      "location": "San Francisco, CA",
      "seniority": "Senior",
      "datePosted": "2025-01-15",
      "url": "https://climate-tech-startup.com/jobs/...",
      "description": "Join our design team...",
      "source": "climate-tech-startup.com",
      "matchScore": 92
    }
  ],
  "sources": ["climate-tech-startup.com", "patagonia.com", ...],
  "message": ""
}
```

## Testing the API

Using PowerShell:
```powershell
$body = @{
    role = "Product Designer"
    location = "San Francisco"
    seniority = "Mid level"
    keywords = "Figma"
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:3000/api/search" `
  -Method Post `
  -Headers @{"Content-Type"="application/json"} `
  -Body $body
```

## Troubleshooting

**"Add a TinyFish API key to TINYFISH_API_KEY"**
- Check that your `.env` file exists and has `TINYFISH_API_KEY=your_key_here`
- Verify the key is valid at [agent.tinyfish.ai/api-keys](https://agent.tinyfish.ai/api-keys)
- Restart the server after updating `.env`

**"A live search timed out"**
- TinyFish APIs may take up to 2 minutes for slower job sites
- Try searching again
- Use broader keywords or fewer filters

**"Scout couldn't reach the listings"**
- Check your internet connection
- Verify the TinyFish API key is valid
- Some job sites block automated requests; Scout will skip those

## What Scout Doesn't Do

- ❌ Generate or invent job listings
- ❌ Cache results (every search is live)
- ❌ Store user data beyond browser localStorage
- ❌ Show jobs from LinkedIn, Indeed, or other aggregators (focuses on direct company sites)

What Scout Does Do:

- ✅ Find real, open positions from company careers pages
- ✅ Match jobs to your preferences
- ✅ Verify each listing is current (via TinyFish Agent)
- ✅ Link directly to application pages
- ✅ Rank by relevance to what you want
- ✅ Remove duplicate listings

## License

Built for the "AI That Clicks" workshop by Team TinyFish. Use and modify freely.
