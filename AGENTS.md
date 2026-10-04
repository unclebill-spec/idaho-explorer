# Idaho Explorer — agent handoff

Static Leaflet map for a nurse household, the same shared app as the Kentucky, Tennessee, Massachusetts, Maine, Vermont, Montana and Wyoming Explorers (`app.js`, `areas.js`, `perm.js`, `profiles.js`, `extras.js`, `style.css` are byte-identical copies of `/workspace/kentucky/explorer/*`; another worker edits them there, so run `scripts/sync_shared.sh` right before every build/publish and log any shared-file edit in KY `explorer/AGENTS.md`). `sw.js` differs only in its cache prefix (`idx-`). Idaho-specific settings live in `explorer/build.py` `STATE` (ported from VT by `scripts/port_build.py`, idempotent, marker PORT_ST).

Live: https://unclebill-spec.github.io/idaho-explorer/ · repo unclebill-spec/idaho-explorer · progress log: `/workspace/idaho/STATUS.md` (newest first, ET).
Idaho was copied from the Montana/Wyoming code base (Oct 3 2026). Every script in `scripts/` picks the state from the folder it runs in (`scripts/common.py` ST); the Idaho copies add ID branches and read the price cap from `common.CAP` (MT/WY 550000, ID 600000). The MT/WY folders still have their own (older, hardcoded $550k) copies.

## Caps (Bill, Oct 3 2026: "Add idaho next and increase the house budget to 600k"; Idaho only, MT + WY stay at $550k)
5+ acres $300k–$600k; 1+ acre 3bd/2ba < $600k; near-hospital 1,600+ sqft 3bd/2ba < $600k (townhomes/condos OK, good condition, ≤ 10 min of a hospital with a 10+ bed ER). No cabin category.

## Blocks
44 counties (`scripts/common.py` -> `data/statewide/raw/id_blocks_500k.zip`).

## Data pipeline (run from /workspace/idaho; pandas scripts use /workspace/kentucky/.venv/bin/python, the rest /usr/bin/python3)
1. `scripts/hospitals_research.py` (CMS + Idaho Time Sensitive Emergency trauma designations + border Level I/II: Sacred Heart Spokane, St. Patrick Missoula, U of Utah, Intermountain Medical Center, McKay-Dee) -> `data/hospitals.json`; `scripts/layers.py` -> block CSVs, schools (`scripts/fetch_nces.py`: NCES ArcGIS CCD + postsecondary), SEDA, RN wages (O*NET/BLS May 2025).
2. `scripts/appeal_build.py` (OSRM drives, cached) -> appeal shading; `scripts/er_beds.py` -> `data/hospital_er_beds.json` (10+ bed ERs for the near-hospital rule).
3. Climate normals (`climate/raw` extracted from the national NOAA tarballs in /workspace/kentucky/climate/raw; `climate/parse.py`, `climate/build_clim.py`) -> `data/clim.json`; activities `data/osm/wd_act.py` (Wikidata) + `data/osm/wp_cat_act.py` (waterfalls + hand trail list); colleges `data/osm/nces_post.json`.
4. Compare areas: `compare/areas_build.py` + `land.py` -> `data/areas.json` (Boise, Nampa, Idaho Falls, Coeur d'Alene, Twin Falls).
5. Homes: `scripts/zsearch.py` (Zillow county searches at the $600k caps) -> `data/zsearch/`; `scripts/listings_build.py` (PER_COUNTY 12; MAX_NEW_DETAIL env, default 500 new detail pages per run, cache `data/zsearch/detail_cache.json` so reruns fetch only missing pages; Idaho copy treats a whole-number bath total with no full/half split as full baths unless the description mentions a half bath) -> `listings.json` + `listing-photos/` (log `data/lb3.log`); `scripts/bargains.py`; `scripts/top_lists.py`.
6. Permanent RN jobs: `scripts/perm_jobs.py` -> `data/perm_jobs.json` (reuses the KY readers). Idaho sources: St. Luke's (Avature: public sitemap + job pages), Saint Alphonsus (Trinity Health Workday "Jobs", Boise/Nampa queries), Kootenai Health (hctsportals job cards), Portneuf (Ardent Jibe API, Pocatello), St. Joseph Regional Lewiston (ScionHealth Radancy, Idaho facet), Cassia Regional (Intermountain Workday), Bonner General (UKG), Madison Memorial (idhrp board); HCA last (Eastern Idaho Regional + West Valley): careers.hcahealthcare.com is Cloudflare-blocked, so `data/hca_websearch.json` (web-search list) is used.
7. Travel RN jobs: helpers in `/workspace/tj_id` (`run.sh`: Vivian + Advantis), then `scripts/travel_jobs.py` -> `data/travel_jobs.json`.
8. Phase 4: `data/airports/airports.py`; `data/attractions/wd2.py` + `make_attractions.py`; Crexi `data/forsale/crexi_list.py`, `st_forsale.py` (hand review: HAND_DROP / MOVE_TO_* / RELABEL_*), `make_forsale.py`; thumbnails `explorer/fetch_thumbs.py` (after a build).
9. Border items: `/workspace/border` (`scripts/static.py ID MT WY`, `scripts/make_border.py ID MT WY`), copy `out/<ST>.json` -> each `explorer/border.json`. MT and WY are covered maps (their homes/jobs come in both ways); UT/NV/OR/WA come from the static layers; north side is Canada (BC: Wikidata hospitals/colleges/parks only).
10. Ski areas + peaks: `/workspace/mtn/scripts/make_state.py ID` -> `explorer/mtn.json` + `img/mtn/` (see /workspace/mtn/PROGRESS.md). Ticket prices come only from skiresort.com.
11. Publish: `sh scripts/sync_shared.sh && cd publish && PATH=/usr/bin:$PATH ./publish.sh -m "msg"` (flock /tmp/idx_publish.lock, pull, build, minify, secscan of dist + full history, push, waits for Pages).
12. Tests (state from the folder): `perf/smoke.py BASE TAG`, `perf/test_homes.py`, `perf/test_perm.py`, `perf/test_p4.py`, `perf/loadtime.py URL`, `perf/sw_check.py URL...`; screenshots in `perf/shots/`.

## Known gaps (Oct 3 2026)
- Pay: only 4 of 325 permanent postings list pay (Idaho has no pay-transparency law), so the Perm ICU and Perm Step-down/Med-surg Top 10s have no ranked rows; their jobs sit under "See more: N that don't list pay". The shared app.js perm panel still says "about 19% list pay" (KY figure; shared file, not changed).
- Near-hospital homes: 97 (the per-county limit of 12 and the 10-minute / 10+ bed ER rule leave most counties with none); 1+ acre homes hit the 160 category cap.
- Zillow detail pages for many Idaho listings give only a whole-number bath total; those count as full baths unless the description mentions a half bath / powder room.
- Border jobs: Idaho gets 3 permanent + 1 travel RN job from Wyoming; none from Montana within ~15 mi of the line. MT/WY get no Idaho jobs (no Idaho hospital with postings within ~15 mi of their lines).
- RN employment counts are null (BLS limits). County history layer is empty (no wiki_history.json).
- St. Luke's Boise has no adult trauma level in the Idaho TSE list (pediatric Level II only): shown without a level, with a note.
- Not collected (perm jobs): Bingham Memorial (ADP Workforce Now, browser-only), Gritman (careers page answered 202 bot check), Mountain View / Idaho Falls Community (no public board found), Syringa, Steele Memorial, Minidoka, North Canyon, Teton Valley, Valor and other small critical-access hospitals. Pay is rarely listed (St. Luke's, Saint Alphonsus, Kootenai post none).
- HCA jobs come from web search (not every opening; links go to the syndicated posting found).
- Ada and Canyon counties have no 5+ acre homes under $600k (5+ acre land medians $1.55M / $1.10M).

### 50+ acre lots under $250k (Oct 4, 2026 ~10:31 AM ET, big-land worker)
- Black-star layer `big-land` (50+ ac, < $250k, land or home), "50+ ac" button, Map key row, card; shared code from the KY explorer (see KY explorer/AGENTS.md, same date). build.py (marker BIGLAND) merges `/workspace/idaho/bigland.json`.
- Refresh: `/usr/bin/python3 /workspace/bigland/bigland.py ID --refresh` before build/publish (keeps the old file if Zillow blocks). Notes: /workspace/bigland/PROGRESS.md.
