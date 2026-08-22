# Movie Calendar — NYC

This folder builds a static movie showtimes site suitable for GitHub Pages,
covering three AMC theaters in New York City. It's a copy of the
[bham-movies](https://github.com/radurbin/bham-movies) project adapted for
NYC — same code, different theater IDs, no Sidewalk-equivalent source.

Overview
- The Python backend fetches showtimes from the AMC API (`fetchers/amc.py`), enriches metadata using OMDb (`fetchers/omdb.py`) and Letterboxd ratings (`fetchers/letterboxd.py`), downloads poster images into `docs/posters/`, and writes `docs/movies.json` consumed by the frontend (`docs/index.html`).

Quick local preview

1. Install dependencies (recommended in a virtualenv):

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

2. Export API keys locally (example):

```bash
export AMC_API_KEY=your_amc_key
export OMDB_API_KEY=your_omdb_key
```

3. Run the pipeline to generate `docs/movies.json` and download posters:

```bash
python fetch_movies.py
```

4. Preview the generated site:

```bash
python3 -m http.server --directory docs 8000
# Open http://localhost:8000 in your browser
```

GitHub Pages setup (repo-level)

- This project expects a `docs/` folder at the repository root containing `index.html`, `movies.json`, and `posters/`.
- Push this folder as its own repository, then enable GitHub Pages in the repository Settings → Pages and select `main` (or the branch you use) and the `/docs` folder as the source.

Secrets (required for Actions)

Add the following repository secrets under Settings → Secrets & variables → Actions:

- `AMC_API_KEY` — your AMC API key (X-AMC-Vendor-Key header). Same key as any other AMC integration; it's not theater-specific, so the Birmingham project's key works here too if you're using one account.
- `OMDB_API_KEY` — your OMDb API key

CI / GitHub Actions

- A workflow file `.github/workflows/update.yml` is included. It runs daily (and can be triggered manually) to:
  1. Install dependencies from `requirements.txt`.
  2. Run `python fetch_movies.py` which regenerates `docs/movies.json` and downloads missing posters into `docs/posters/`.
  3. Commit any changed files under `docs/` back to the repo so Pages serves the newest data.

Data sources and theaters

This project currently includes showtimes for three theaters:

- AMC Lincoln Square 13 (theater id 2116)
- AMC 84th Street 6 (theater id 2102)
- AMC 34th Street 14 (theater id 2120)

Unlike the Birmingham version, there's no second (non-AMC) source merged
in — `fetchers/sidewalk.py` and its Cloudflare Worker proxy were removed
entirely since they were specific to Sidewalk Film Center in Birmingham.
`fetch_movies.py`'s pipeline is just AMC → OMDb enrichment → write
`movies.json`, with no title-matching/merge step needed.

How far in the future is fetched

- The AMC fetcher (`fetchers/amc.py`) requests showtimes from AMC's `/theatres/{id}/showtimes` endpoint and paginates results. The API determines how many days ahead are returned. Practically, the generated `movies.json` contains whatever upcoming showtimes the AMC API returns at fetch time.

Poster and movie data retention

- OMDb responses are cached in `cache/omdb_cache.json` by `fetchers/omdb.py` to avoid re-querying OMDb for unchanged titles.
- Letterboxd ratings are cached in `cache/letterboxd_cache.json` by `fetchers/letterboxd.py`. Letterboxd has no public API, so each movie's page is found via its IMDb ID (`letterboxd.com/imdb/{imdb_id}/`, which redirects to the film page) and the rating is read out of that page's embedded JSON-LD. This only works for movies OMDb already resolved an `imdb_id` for; Letterboxd's own search page 403s scripted requests, so there's no title-based fallback for movies OMDb missed.
- Posters are downloaded into `docs/posters/`. The pipeline avoids re-downloading posters that already exist (it checks file presence by filename).
- After each run the pipeline removes stale poster files: any files in `docs/posters/` not referenced by the newly generated `movies.json` are deleted. This keeps the poster directory trimmed to only the artwork currently referenced by the frontend.

Scheduling and frequency

- The default workflow runs daily (see `.github/workflows/update.yml`). You can change the cron schedule in that file or trigger the workflow manually from the Actions tab.

Security and secrets

- Never commit API keys. Use GitHub repository secrets for Actions and local environment variables for local testing.

Troubleshooting

- If Actions fails due to missing keys, confirm `AMC_API_KEY` and `OMDB_API_KEY` are set in the repository secrets.
- If posters are failing to download due to remote URL changes, inspect the `docs/movies.json` poster URLs and check network access.
