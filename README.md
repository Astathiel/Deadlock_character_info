# Deadlock Heroes, Lane Roles & Player Meta Infographic

A responsive, standalone web infographic and telemetry hub for Valve's hero shooter, **Deadlock**. Displays all 44 playable heroes sorted by played lane roles, map zones (*Yellow/Green Side*, *Blue Mid*, *Jungle/Roam*), meta tiers (`S+` through `D`), hero characteristics, and live Ranked player telemetry.

---

## Features
- **44 Playable Heroes Indexed:** Complete roster with official character cards, role tags, and traits.
- **Dynamic Grouping & Filtering:** Group by Lane Role, Map Zone, Meta Tier, or Hero Class; filter by character traits or search by name.
- **Direct Build Links:** Quick links to hero builds and guides on [Deadlocked.gg](https://deadlocked.gg) and [Tracklock.gg](https://tracklock.gg).
- **Multi-Tier Portrait Resolution:** Seamless portrait loading from Deadlock Wiki with automatic fallbacks to Tracklock, minimap icons, and inline SVG avatars.
- **Zero-Database Daily Telemetry Auto-Sync:** Optional daily scraping using a single Vercel Serverless Function and Edge caching.

---

## Deployment on Vercel

This repository is optimized for one-click deployment on [Vercel](https://vercel.com):

1. Push this repository to GitHub.
2. In the Vercel dashboard, click **Add New Project** and import the repository.
3. Deploy with default settings. The `vercel.json` and `index.html` files at the root will handle static routing automatically.

---

## Vercel Backend & Daily Auto-Sync Setup

The frontend is wired to fetch `/api/meta` on load. By adding a single Serverless Function to your repository, Vercel's Edge network will scrape and cache daily match telemetry from Tracklock.gg and Deadlocked.gg for 24 hours (`s-maxage=86400`), serving fresh stats to all visitors with zero database requirements.

### 1. Folder Structure

```text
Deadlock_character_info/
├── api/
│   └── meta.js          # (Optional) Vercel Serverless Function
├── index.html           # Main infographic web application
├── vercel.json          # Routing & daily cron configuration
└── README.md
```

### 2. File: `api/meta.js` (Serverless Scraper)

Create `api/meta.js` with the following code (Node.js 20+):

```javascript
// api/meta.js — Vercel Serverless Function (Node.js 20+)
// Fetches Tracklock.gg & Deadlocked.gg daily and caches at Vercel Edge for 24 hours.

export default async function handler(req, res) {
  try {
    const [tracklockRes, deadlockedRes] = await Promise.allSettled([
      fetch("https://tracklock.gg/heroes-tier-list", {
        headers: { "User-Agent": "Mozilla/5.0 (compatible; DeadlockInfographicBot/1.0)" }
      }),
      fetch("https://deadlocked.gg/", {
        headers: { "User-Agent": "Mozilla/5.0 (compatible; DeadlockInfographicBot/1.0)" }
      })
    ]);

    const heroes = {};

    // 1. Parse Tracklock.gg Ranked Table rows (Hero, Tier, Win Rate, WR Delta, Pick Rate)
    if (tracklockRes.status === "fulfilled" && tracklockRes.value.ok) {
      const html = await tracklockRes.value.text();
      const text = html.replace(/<[^>]+>/g, " ").replace(/\s+/g, " ");
      // Matches patterns like: "Graves S+ 55.13% +0.07% +0.07% 39.26%"
      const rowRegex = /([A-Z][a-zA-Z0-9 &'-]+?)\s+(S\+|S|A|B|C|D)\s+(\d{2}\.\d{2})%\s+([+-]\d+\.\d{2}%)\s+[+-]\d+\.\d{2}%\s+(\d{1,2}\.\d{2})%/g;
      let match;
      while ((match = rowRegex.exec(text)) !== null) {
        const name = match[1].trim();
        heroes[name] = {
          tier: match[2],
          winRate: parseFloat(match[3]),
          wrDelta: match[4],
          pickRate: parseFloat(match[5])
        };
      }
    }

    // 2. Parse Deadlocked.gg 30d Win Rate Trends
    if (deadlockedRes.status === "fulfilled" && deadlockedRes.value.ok) {
      const dlHtml = await deadlockedRes.value.text();
      const dlText = dlHtml.replace(/<[^>]+>/g, " ").replace(/\s+/g, " ");
      const dlRegex = /([A-Z][a-zA-Z0-9 &'-]+?)\s+(S\+|S|A|B|C|D)\s*(\d{2}\.\d)%\s*([▲▼]\s*[+-]?\d+\.\d%)/g;
      let m;
      while ((m = dlRegex.exec(dlText)) !== null) {
        const name = m[1].trim();
        if (heroes[name]) {
          heroes[name].deadlockedWR = parseFloat(m[3]);
          heroes[name].note = `Deadlocked 30d: ${m[3]}% (${m[4]})`;
        }
      }
    }

    // Cache on Vercel CDN Edge for 24 hours (86400s), revalidate in background
    res.setHeader("Cache-Control", "public, s-maxage=86400, stale-while-revalidate=3600");
    return res.status(200).json({
      updatedAt: new Date().toISOString(),
      source: "Tracklock.gg + Deadlocked.gg (Vercel 24h Edge Cache)",
      heroes
    });
  } catch (err) {
    return res.status(500).json({ error: err.message });
  }
}
```

### 3. File: `vercel.json` (Crons & Routing)

Update `vercel.json` to include daily automated execution at 06:00 UTC:

```json
{
  "rewrites": [
    { "source": "/", "destination": "/index.html" },
    { "source": "/deadlock_heroes_lane_roles_infographic", "destination": "/index.html" },
    { "source": "/deadlock_heroes_lane_roles_infographic.html", "destination": "/index.html" }
  ],
  "crons": [
    {
      "path": "/api/meta",
      "schedule": "0 6 * * *"
    }
  ]
}
```

---

## Data Sources
- **Hero characteristics & traits:** [deadlock.one](https://deadlock.one)
- **Ranked win rates, pick rates, & tiers:** [Tracklock.gg](https://tracklock.gg)
- **30-day trending analytics & guides:** [Deadlocked.gg](https://deadlocked.gg)
- **Hero portraits & cards:** [Deadlock Wiki](https://deadlock.wiki)
