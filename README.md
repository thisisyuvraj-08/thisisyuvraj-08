# Animal Video Scraper (Reddit + Pinterest)

Downloads animal videos from Reddit and Pinterest into per-animal folders
(`downloads/cats/`, `downloads/dogs/`, `downloads/elephants/`, etc.), never
downloading the same video twice.

## How it works

Rather than writing custom HTML/JSON parsing (which breaks constantly as
Reddit/Pinterest change their sites), this uses two actively-maintained,
purpose-built tools under the hood:

- **[gallery-dl](https://github.com/mikf/gallery-dl)** — handles the actual
  scraping of Reddit and Pinterest, and has a built-in `--download-archive`
  feature: every downloaded item's unique ID is stored in a small database
  file, so **future runs automatically skip anything already downloaded**.
  This is the "never download the same video twice" guarantee — it's
  permanent, not just per-run.
- **[yt-dlp](https://github.com/yt-dlp/yt-dlp)** — gallery-dl calls this
  internally to pull actual video files (e.g. `v.redd.it` links).

`run_scraper.py` just loops over your animal list and calls gallery-dl once
per source, with a `--filter` that keeps only video files (mp4/webm/mov/gif),
so images are skipped.

## 1. Install

```bash
pip install gallery-dl yt-dlp
# ffmpeg is required for merging some reddit videos+audio
# macOS: brew install ffmpeg | Ubuntu: sudo apt install ffmpeg | Windows: winget install ffmpeg
```

## 2. Set up Reddit API credentials

Reddit scraping via gallery-dl works much better (and follows Reddit's rules)
with an API app:

1. Go to https://www.reddit.com/prefs/apps → "create another app..."
2. Choose type **"installed app"**, give it any name, redirect URI can be
   `http://localhost:8080`
3. Copy the string under the app name (that's your `client-id`)
4. Open `gallery-dl.conf.json` and fill in:
   ```json
   "client-id": "your_client_id_here",
   "user-agent": "python:animal-video-scraper:v1.0 (by u/your_reddit_username)"
   ```

## 3. Set up Pinterest access (optional but recommended)

Pinterest heavily limits what anonymous/logged-out requests can see. For
decent search results:

1. Log into pinterest.com in your browser
2. Export your cookies with a browser extension like "Get cookies.txt LOCALLY"
3. Save the exported file as `pinterest_cookies.txt` in this folder
4. In `gallery-dl.conf.json`, set:
   ```json
   "cookies": "pinterest_cookies.txt"
   ```
   (replace the `"cookies": null` line under `"pinterest"`)

Without this, Pinterest scraping will still run but may return few or no
results — Pinterest actively blocks logged-out scraping.

## 4. Run it

```bash
python run_scraper.py --list                 # see available categories
python run_scraper.py --animal cats           # scrape just cats
python run_scraper.py --all                   # scrape every category
python run_scraper.py --all --limit 15        # cap items per source per run
```

Output structure:

```
downloads/
  cats/
  dogs/
  elephants/
  rhinos/
  deer/
  birds/
archives/
  cats.sqlite3      <- dedup database, don't delete this
  dogs.sqlite3
  ...
```

## 5. Add more animals

Edit `animals.json` and add a new entry, e.g.:

```json
"foxes": {
  "reddit": [
    "https://www.reddit.com/r/foxes/",
    "https://www.reddit.com/search/?q=fox+video&sort=new"
  ],
  "pinterest_queries": ["fox video"]
}
```

No code changes needed — `run_scraper.py` reads this file automatically.

## 6. Automate it (optional)

To keep the folders growing over time, schedule a periodic run, e.g. with
cron (Linux/macOS):

```
0 6 * * * cd /path/to/animal-scraper && /usr/bin/python3 run_scraper.py --all --limit 10
```

Because of the download-archive dedup, running this daily/weekly is safe —
it will only ever add new videos, never re-download old ones.

## 7. Building 10-minute compilation videos (recommended path)

Everything above (`run_scraper.py`) downloads and keeps raw clips — useful for
building a personal archive, but it needs Reddit API credentials and Pinterest
cookies you haven't set up yet (`gallery-dl.conf.json` still has placeholder
values).

For actually producing finished 10-11 minute compilation videos, use
`make_compilation.py` / `orchestrator.py` instead. This path:

- Pulls clips from the **Pixabay and Pexels official video APIs** (keys already
  in `config.py`) — no Reddit/Pinterest setup needed.
- Downloads clips to a temp directory, builds one scaled/padded/concatenated
  1080p compilation with ffmpeg, then **deletes every source clip** — only the
  finished video is ever kept on disk.
- Never reuses the same combination of source clips twice (tracked per-animal
  in `history.json`).
- After building, uploads the finished video to the **Gemini API** and asks it
  to describe the content and compare it against the summaries of every prior
  video for that animal. If Gemini flags it as too similar (similarity ≥ 0.8),
  the video is discarded and rebuilt with a different clip combination (up to
  3 attempts) before being kept.
- Logs every finished video's details (clip IDs, duration, Gemini's content
  summary, tags, and similarity verdict) into `history.json`.

Manual run for one category:

```bash
python3 make_compilation.py cats        # one ~11 min cats video
python3 make_compilation.py dogs 2      # two ~11 min dogs videos
```

Output lands in `~/Downloads/<animal>_compilations/`.

### Automated daily run

`orchestrator.py` builds exactly one video per invocation, rotating through
every category in `animals.json` in order (one category/day if run daily), and
remembers where it left off in `rotation_state.json`. It's installed as a
daily cron job:

```
0 9 * * * cd ~/Downloads/animalyt && /Library/Frameworks/Python.framework/Versions/3.13/bin/python3.13 orchestrator.py >> cron.log 2>&1
```

(cron doesn't see your shell's `python3` alias/PATH, so this points straight at the interpreter that
actually has `requests`/`imageio_ffmpeg` installed — check with `python3 -c "import sys; print(sys.executable)"`
if you ever reinstall Python and need to update this.)

Check it any time with `crontab -l`; remove it with `crontab -e`. Progress and
errors are appended to `cron.log` in this folder.

## Notes on legality / etiquette

- Pixabay and Pexels videos are offered under their own free-to-use stock
  license (attribution not required for either), so the compilation pipeline
  above is on much firmer footing than the Reddit/Pinterest scraper.
- The scraper path (`run_scraper.py`) is intended for **personal use**
  (building your own local video collection), not redistribution — the
  creators of those videos retain copyright.
- Respect Reddit's API terms (don't set an absurdly high `--limit` or run
  extremely frequently) and Pinterest's Terms of Service.
- gallery-dl already rate-limits/retries sensibly by default; avoid
  overriding that to hammer either site.
