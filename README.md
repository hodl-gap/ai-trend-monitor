# ai-trend-monitor

An unattended **AI-trend monitor** across X / LinkedIn / YouTube — it reads, filters
to what matters (what AI leaders are doing/thinking), reports a digest, discovers new
people/channels, and (optionally) likes/follows to tune your feeds. Driven by Claude
Code + the chrome-devtools MCP on **your own** logged-in browser sessions.

This umbrella bundles 4 repos as git submodules:

| Submodule | Role |
|-----------|------|
| `people-db` | **shared core** — registries (`people.json` = humans + X/LinkedIn watchlist; `channels.json` = YouTube watchlist), the significance rubric (`judge_prompt.md`), shared tools, and the design docs (`PRD.md`, `YOUTUBE.md`, `pipeline-flowcharts.html`) |
| `twitter-scraper-chrome-devtools` | X pipeline (`scan-twitter.sh` watchlist, `scan-twitter-home.sh` home feed) |
| `linkedin-scraper` | LinkedIn pipeline (`scan-linkedin.sh`, `scan-linkedin-home.sh`) |
| `youtube-scraper` | YouTube pipeline (`discover-youtube.sh`, `scan-youtube.sh`) |

> **Layout matters:** the 3 scrapers read the shared core via the relative path
> `../people-db/`. This umbrella keeps them as **flat siblings**, so those paths
> resolve. Don't move the submodules around.

For the full picture open **`people-db/pipeline-flowcharts.html`** in a browser.

---

## 🔐 No secrets in this repo
There are **no credentials and no cookies** here. Login sessions live only in your
local Chrome profile dir (e.g. `~/.cache/chrome-devtools-mcp/chrome-profile`), which
is never committed. You create your own sessions during setup (below). Each scraper's
`store/` and `digests/` (scraped data + outputs) are git-ignored — keep them local.

## Clone
You need read access to **all 4 submodule repos** (they're private — submodules don't
bypass that).
```bash
git clone --recurse-submodules https://github.com/hodl-gap/ai-trend-monitor.git
# already cloned without --recurse-submodules?
git submodule update --init --recursive
```

## Prerequisites
- **WSL2 + Linux Google Chrome** at `/usr/bin/google-chrome-stable`, rendered via WSLg.
  (The chrome-devtools MCP drives a real headed Chrome; `--ozone-platform=wayland` is
  mandatory under WSLg — headed Chrome crashes on the X11 path.)
- **Claude Code** (`claude`) on PATH and logged in — the scrapers run headless
  `claude -p` agents.
- **`yt-dlp`** (YouTube transcripts) and **python3**.

## One-time setup

### 1. Register the chrome-devtools MCP (adjust paths to YOUR machine)
```bash
claude mcp add chrome-devtools -s user -- \
  npx -y chrome-devtools-mcp@latest \
  --executablePath /usr/bin/google-chrome-stable \
  --userDataDir "$HOME/.cache/chrome-devtools-mcp/chrome-profile" \
  --chromeArg=--ozone-platform=wayland \
  --chromeArg=--window-size=1440,900
```
Then **restart Claude Code** so the `mcp__chrome-devtools__*` tools load.
Also edit each scraper's **`.mcp.json`** — set `--executablePath` and `--userDataDir`
to your machine's Chrome binary and a profile dir you control.

### 2. Log in once per platform (this seeds *your* session cookies into the profile)
Open the chrome-devtools Chrome (it launches when an MCP tool first runs, or launch it
yourself with the same flags) and log in:
- **LinkedIn** and **YouTube** — log in normally in that Chrome window.
- **X / Twitter — important caveat:** X **blocks login under automation**
  (`navigator.webdriver=true`). So log into X with a **clean, non-automated** Chrome
  pointed at the **same profile dir**, then let automation reuse the session:
  ```bash
  google-chrome-stable --user-data-dir="$HOME/.cache/chrome-devtools-mcp/chrome-profile" \
    --ozone-platform=wayland https://x.com/login
  # log in, then close that window
  ```
  X only challenges at *login*; it allows automated browsing/like/follow on an existing
  session. Sessions persist across runs — re-do this if one expires.

> Only one Chrome may hold the profile dir at a time — close the clean login window
> before running a scan, and don't run two scans at once.

### 3. Seed the registries (in `people-db/`)
- `people.json` — the people to watch (X/LinkedIn). Watchlist == people-db.
- `channels.json` — YouTube channels to watch.

## Run
```bash
cd twitter-scraper-chrome-devtools
./scan-twitter.sh              # watchlist (tracked people)
./scan-twitter-home.sh         # your home feed (discovers new AI people)

cd ../linkedin-scraper
./scan-linkedin.sh             # watchlist
./scan-linkedin-home.sh        # home feed

cd ../youtube-scraper
./scan-youtube.sh              # tracked channels -> new videos + discovery
./discover-youtube.sh          # recommended-feed -> channel discovery
```
**Engagement is OFF by default.** Add `--engage --dry-run` to log intended
likes/follows without clicking, or `--engage` to do them for real. Caps
(`--max-likes`, `--max-follows`, `-n`, `--max-new`) are **ceilings, not targets** —
a quiet run does little or nothing, by design.

Outputs land in each scraper's `digests/` (the human report) and `store/` (raw data,
dedup, action logs) — all local and git-ignored.

## Notes
- Respect each platform's Terms of Service; keep engagement human-paced.
- `people.json`/`channels.json` are real watchlists (public handles/roles, no secrets) —
  treat per your own sharing norms.
- Two criteria are intentionally loose placeholders (see `people-db/YOUTUBE.md` and
  `PRD.md`): the video-"valuable" bar and the home-feed "AI-person" bar.
