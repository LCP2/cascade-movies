# Cascade Movies — Proof of Concept

A working POC for the AU movie release-window tracker: see films that are/were in
cinemas, filter by rating / box office / genre / availability window, tag titles,
and get alerts when a tagged title's availability changes.

It runs **standalone with illustrative sample data** (no keys needed) and switches
to **live AU data** the moment you add three free API keys.

## Files
- `index.html` — the app. **Open it in any real browser** (double-click, or drag into
  Chrome/Edge/Firefox/Safari). Filters, tagging, and a "Simulate next daily poll"
  button that fires alerts on tagged titles. Self-contained; data is embedded so it
  needs no server. (Note: it will look empty in a sanitised inline *preview* that
  strips JavaScript — that's the preview, not the file. A normal browser runs it.)
- `poc_pipeline.py` — the real backend loop: ingest → enrich → poll → derive → diff →
  alert, and it **regenerates index.html** with the latest data on every run.
- `app_template.html` — the app with `__MOVIES_JSON__` / `__TODAY__` placeholders; the
  pipeline fills these in to produce index.html. Edit UI here, not in index.html.
- `sample_data.json` — the illustrative dataset (invented titles, placeholder figures).
- `movies.json` — pipeline output; the source of truth the app is built from.

## End-to-end on a PC (the whole loop)
```
python3 poc_pipeline.py        # builds movies.json + index.html from current data
# then just open index.html in your browser
```
Run it with the three keys set (below) and the same command produces an index.html
full of **real AU films**. Run it once a day (Task Scheduler / cron) to keep data
fresh and to detect window changes → alerts. To rebuild the page from the last
movies.json without re-polling: `python3 poc_pipeline.py --build-html`.

## Try the app
Just open `index.html`. Tap **⚙︎ Filters**, tap the **☆** on a couple of films to
track them, open the **🔔** drawer, and hit **Simulate next daily poll**.

## Run the backend (sample mode)
```
python3 poc_pipeline.py                 # establishes a baseline snapshot
python3 poc_pipeline.py --simulate-day  # advances a couple of titles → prints alerts
```

## Go live (real AU data)
Get three free keys, then set them and run again — no code changes:
```
export TMDB_API_KEY=...        # themoviedb.org/settings/api          (free)
export OMDB_API_KEY=...        # omdbapi.com/apikey.aspx              (free, 1k/day)
export WATCHMODE_API_KEY=...   # api.watchmode.com/requestApiKey      (free, 2.5k/mo)
python3 poc_pipeline.py
```

## What's real vs. simplified in this POC
- **Real:** the data-source choices, the join keys (TMDB↔IMDb↔Watchmode), the
  window-derivation logic, and the diff-and-alert engine — all production-shaped.
- **Simplified:** sample data stands in for live API responses; alerts print to
  console / show in-app rather than going to push (FCM/APNs) yet; state is a JSON
  file rather than a database.

## Known data limits (see project doc for detail)
- AU-specific box office isn't available cheaply → the "Gross" figure is *worldwide*.
- Rotten Tomatoes = **critic** score only (via OMDb); audience score needs scraping.
- The $30-vs-$7 split is a price-threshold heuristic you tune (`PVOD_MIN_PRICE`,
  `RENTAL_MAX_PRICE` in poc_pipeline.py), because prices vary by store and format.
