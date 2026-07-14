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

## Catalogue scope — backwards AND forwards from cinema
The pipeline makes two TMDB `discover` passes, both filtered to **AU theatrical**
(`with_release_type=2|3`, `region=AU`):

| Pass | Window | Cap | Cost |
| --- | --- | --- | --- |
| **Released** (`ingest_tmdb`) | `release_date` in the last `LOOKBACK_DAYS` (~3 yrs) | `MAX_TITLES` (60) | TMDB + OMDb + **Watchmode** per title |
| **Upcoming** (`ingest_tmdb_upcoming`) | `release_date` from tomorrow to `+UPCOMING_LOOKAHEAD_DAYS` (~4 mths) | `MAX_UPCOMING` (12) | TMDB + OMDb only — **no Watchmode** |

The upcoming pass costs **zero Watchmode calls**: a film that hasn't opened has no AU
home offers to poll, so the free-tier budget stays entirely with the released catalogue.
Those titles carry no offers and therefore derive to the **`upcoming`** window, which is
what fills the cinema slot of the app's cascade stepper ("Upcoming → Premium → Standard →
Streaming") and feeds the **Blockbuster radar** Cascade. Sort by **Most anticipated** to put
them in order — upcoming films have no box office and usually no budget either, so TMDB
`popularity` is the only field that's actually populated for all of them.

Tune the scope with `LOOKBACK_DAYS` / `MAX_TITLES` / `UPCOMING_LOOKAHEAD_DAYS` /
`MAX_UPCOMING` at the top of `poc_pipeline.py`.

## Scale & origin fields (what the editor's Budget / Tentpole / Origin controls read)

| Field in `movies.json` | Source | Notes |
| --- | --- | --- |
| `popularity` | TMDB `popularity` | Present for **every** title — the only scale signal that is. |
| `budget` | TMDB `budget` | **Missing for 11 of 72** titles, and missing for most *upcoming* ones. |
| `language` | TMDB `original_language` | ISO code (`en`, `ja`, …). |
| `culture` | derived: `original_language` + `production_countries` | Bucket: Western / European / Japanese / Chinese / Korean / Indian / Southeast Asian / Spanish-Latin / Other. See `_culture()`. |

**The tentpole rule, stated in full** (front-end, `tentpoleOf()` — and shown to the user
inside the editor rather than hidden in here):

> A title is a tentpole if its TMDB popularity is in the **top 25% of its own cohort**, *or*
> in the **top 50%** of it *and* carries a **$120M+ budget**. The cohort is **upcoming** titles
> (→ **Anticipated**) or **released** titles (→ **Blockbuster**).

Two deliberate choices:

- **Popularity leads; budget only ever *promotes*, never demotes.** Budget is the field we're
  missing (11 of 72, and most upcoming titles). Leading with it would have blanked exactly the
  films the tag exists to catch.
- **Each film is ranked against its own cohort, not the whole catalogue.** A film that opened
  two years ago has had two years to accumulate popularity; measuring an unreleased title
  against it is a rigged race. On a whole-catalogue bar, *Spider-Man: Brand New Day* — the #2
  most-anticipated title in the set, and one with **no budget figure at all** — did not clear
  it. Against the upcoming cohort it does, which is the honest answer.

Both thresholds are **percentiles recomputed at build** over the current catalogue, so they
can't drift into meaning nothing.

Titles with **no budget figure** behave exactly like titles with no IMDb rating: included at
"Any", dropped as soon as a band is chosen, with a one-tap escape hatch that names how many
are being left out. We never impute a number we don't have.

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
