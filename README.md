# nps-ai-assistant

**An AI generated school briefing app, in two parts: an Actions driven data pipeline that publishes to GitHub Pages, and an optional self hosted Express backend.**

nps-ai-assistant is the original version of a daily briefing system for the NPS parent portal. The root of the repository is the pipeline part: GitHub Actions scrapes the school portal, pulls Gmail and news, asks an AI model to summarize the day, and commits the resulting JSON snapshots into `public/data`. GitHub Pages serves the front end, which is a single static page that reads those snapshots. The `backend/` folder holds a fuller server variant that adds Google Calendar sync and a midnight WhatsApp driven class timetable importer, along with its own hosted front end.

## How the pipeline works

The sync workflow runs three times a day (7:00, 13:00, and midnight IST), plus manual triggers:

| Script | What it does |
|---|---|
| `scripts/scrape-nps.mjs` | Playwright logs into the parent portal and captures notices, circulars, and homework data into MongoDB |
| `scripts/sync-gmail.mjs` | Pulls recent mail through OAuth into MongoDB |
| `scripts/sync-news.mjs` | Fetches daily headlines from the news API |
| `scripts/generate-briefing.mjs` | Sends the day's data through Groq and writes the briefing narrative |
| `scripts/export-data.mjs` | Exports items, events, and the briefing to JSON snapshots in `public/data/` |
| `scripts/lib/mongo.mjs` | Shared MongoDB connection helper |

After export, the workflow commits any changed snapshots with a `data: auto-sync` message and pushes, which redeploys the Pages site with fresh content. The reminder comments in the workflow note that one calendar sync path (the Aakash WhatsApp to school calendar flow) runs on a phone via Termux, not in Actions.

## The front end

`public/` is the whole site: `index.html`, `app.js`, and `styles.css`, plus `public/data/*.json` snapshots that the synchronous pipeline refreshes:

| Snapshot | Contents |
|---|---|
| `briefing.json` | The AI generated day narrative |
| `calendar.json` | Today and tomorrow's calendar events |
| `holidays.json` | Holiday list |
| `items.json` | Synced items (circulars, homework, mail, news) |

The page renders like a small daily paper: a briefing masthead, class schedule, notices, inbox, and world feed. Screenshots of the portal itself (`nps-login.png`, `nps-after-login.png`) live in the repo root.

## The backend variant

The `backend/` directory is a Node service (`backend/package.json`, version 2.0) for hosting the briefing on your own machine or a server rather than relying on the Pages pipeline:

- Same connector approach (Playwright scraper, Gmail OAuth, news fetch, Groq summarization).
- Adds a Google Calendar connector and routes that read calendar data for the month and upcoming views, with school events marked out from personal ones.
- Adds `AakashSync` (`src/aakash.js`), a midnight cron that processes timetable material from WhatsApp into the school calendar.
- Cron driven syncs at 7:00 and 13:00, plus the midnight Aakash pass.
- Serves its own front end from `backend/public/`, including the dashboard shell, a debug page for timetable uploads, and style sheets.
- Uses Mongoose for storage and credentials in `.env` (see `backend/.env.example`).

## Options and credentials

Both halves read their configuration from environment variables. Locally, copy `backend/.env.example`:

```bash
git clone https://github.com/nithilan-manivannan/nps-ai-assistant
cd nps-ai-assistant

# pipeline part (GitHub Pages variant)
npm install
npx playwright install chromium
node scripts/scrape-nps.mjs

# server variant
cd backend
npm install
npx playwright install chromium
node server.js
```

| Variable | Used by | Purpose |
|---|---|---|
| `MONGODB_URI` | both | Data store |
| `GROQ_API_KEY` | both | Briefing generation |
| `NPS_PORTAL_URL` `NPS_USERNAME` `NPS_PASSWORD` | both | Portal login |
| `GMAIL_CLIENT_ID` `GMAIL_CLIENT_SECRET` `GMAIL_REFRESH_TOKEN` | both | Gmail sync |
| `GCAL_CLIENT_ID` `GCAL_CLIENT_SECRET` `GCAL_REFRESH_TOKEN` `SCHOOL_CALENDAR_ID` | export step, backend | Calendar sync |
| `NEWS_API_KEY` | both | Headlines |

Helper scripts `backend/start-bg.ps1` and `backend/stop.ps1` run and stop the server in the background on Windows. `scripts/clear-nps-summaries.mjs` resets stored summaries when you want a clean re run.

## Repository structure

```
nps-ai-assistant/
├── package.json                  # pipeline scripts and pipeline deps
├── scripts/
│   ├── scrape-nps.mjs            # Playwright portal scraper
│   ├── sync-gmail.mjs
│   ├── sync-news.mjs
│   ├── generate-briefing.mjs
│   ├── export-data.mjs
│   └── lib/mongo.mjs
├── public/                       # the static Pages site
│   ├── index.html · app.js · styles.css
│   └── data/                     # committed JSON snapshots
├── backend/
│   ├── server.js                 # Express bootstrap with cron
│   ├── src/                      # config, db, connectors, services, routes
│   ├── public/                   # back end's own front end
│   ├── .env.example              # portal + mongo + groq credentials
│   └── package.json
├── nps-login.png · nps-after-login.png
└── .github/workflows/
    ├── sync.yml                  # scheduled scrape, sync, brief, export, commit
    └── pages.yml                 # Pages deployment
```

## Related project

[daily-briefing](https://github.com/nithilan-manivannan/daily-briefing) is the successor backend: it keeps the same connectors, folds Calendar into one OAuth flow in a full time server, and deploys to Render instead of relying on a static Pages pipeline.
