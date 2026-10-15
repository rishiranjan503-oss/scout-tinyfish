# Scout Quick Start Guide

Get Scout running in 3 minutes.

## Prerequisites

- Node.js 20 or newer ([download here](https://nodejs.org))
- TinyFish API key ([get one here](https://agent.tinyfish.ai/api-keys))

## Setup

1. **Clone or download Scout**
   ```powershell
   cd "path\to\scout"
   ```

2. **Create `.env` file**
   ```powershell
   Copy-Item .env.example .env
   ```

3. **Add your API key**
   - Open `.env` in your editor
   - Replace `your_tinyfish_api_key` with your actual key
   - Save the file

4. **Start Scout**
   ```powershell
   node --env-file=.env server.mjs
   ```

   You should see:
   ```
   Scout is ready at http://localhost:3000
   ```

5. **Open your browser**
   - Navigate to [http://localhost:3000](http://localhost:3000)
   - You should see Scout's home page

## First Search

1. Enter a job role (e.g., "Product Designer")
2. (Optional) Add location, experience level, keywords
3. Click "Find my next move"
4. Wait 30-60 seconds while Scout searches live career pages
5. Review results with match scores

## How Results Work

Each job card shows:
- **Job title** (e.g., "Senior Product Designer")
- **Company** (first letter in a colored box)
- **Location** (with 📍 icon)
- **Seniority level** (if available)
- **Match score** (0-99%, based on your preferences)
- **Posted date** (if available)
- **"View opportunity"** link (opens the original job page)

Results are deduplicated and sorted by match score by default. Use the "Sort by" dropdown to order by most recent instead.

## Tips

- **Be specific about role:** "Product Designer" finds more relevant jobs than "Designer"
- **Use keywords:** Add skills or tools you want (e.g., "Figma, React, climate tech")
- **Save preferences:** Scout saves your search preferences in the browser; they persist between sessions
- **Check multiple sources:** Each result shows which company/site it came from
- **Apply directly:** Every link goes straight to the original job posting

## Understanding the Search

When you search, Scout:

1. **Discovers** 8 live career pages matching your preferences
2. **Reads** the content from each page using TinyFish Fetch
3. **Verifies** job listings with the TinyFish Agent (asks: "Is this really open? Does it match?")
4. **Scores** each job based on role, location, seniority, keywords
5. **Deduplicates** same jobs found on multiple sites
6. **Ranks** by match score (best matches first)
7. **Returns** top 30 results with direct apply links

This is why Scout takes 30-60 seconds—it's searching live pages in real-time, not pulling from a cached database.

## Troubleshooting

**Scout won't start:**
- Check Node.js version: `node --version` (must be 20+)
- Check `.env` file exists in the same folder as `server.mjs`
- Check `TINYFISH_API_KEY` is set correctly (no quotes needed)

**Search returns no results:**
- Try a broader job role or location
- Remove keywords and search again (keywords are optional)
- Some job sites rate-limit automated searches; try again in 5 minutes

**Search times out:**
- First search often takes longest (up to 2 minutes)
- Subsequent searches are usually faster
- Some job sites are slower to respond

**Want to stop Scout:**
- Press `Ctrl + C` in the terminal

## Next Steps

- Customize your search preferences and save them
- Try different roles and locations
- Share your find! Use the LinkedIn and X links in the app
- Read [TINYFISH_USAGE.md](./TINYFISH_USAGE.md) to learn how Scout uses TinyFish APIs

## Need Help?

- Check [README.md](./README.md) for detailed documentation
- Verify your [TinyFish API key](https://agent.tinyfish.ai/api-keys) is valid
- Ensure your internet connection is working

Happy hunting! 🎯
