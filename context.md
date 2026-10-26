# Project context and changelog

A running record of what this project is, what data it uses, what has been decided and why, and what changed when.

**How to use this file**

* Read it before starting work, so you know the current state and the decisions already made.
* After any meaningful change (new data, a cleaning rule, a decision, a new notebook), add an entry at the **top** of the [Changelog](#changelog) and update the sections above it if they are now out of date.
* Keep entries short: date, who, what changed, and why. Link commits by their short hash.

---

## 1. The project

| | |
|---|---|
| **Title** | Who Gets a Bluebike? Spatial and Demographic Equity in Boston's Bluebikes Network |
| **Course** | CP 101 Introduction to Urban Data Analytics, UC Berkeley, Fall 2026 |
| **Team** | Group 23: George Huang, Oliver Kirk, Handson Li, Shaan Patel |
| **Track** | Track B (Data Analytics & Reproducibility): a GitHub repository with a GitHub Pages site and at least 2 interactive, animated HTML maps |
| **Repository** | https://github.com/CYPLAN-101-Fall-2026-Group-23/bluebikes_equity |
| **Study area** | The City of Boston (not the whole Bluebikes system, which also serves Cambridge, Somerville, Brookline and other towns) |
| **Study period** | Calendar year 2024. The plan is to expand to more years later. |

**Research question (current wording):** Across the City of Boston in 2024, do Bluebikes stations and trips concentrate in higher-income neighborhoods, and who lives in the areas with the least bikeshare activity?

**Approach from the proposal:**

1. Rank the busiest stations by hour and day of week.
2. Map the most common origin and destination pairs.
3. Group stations into areas and read their patterns (weekday mornings as commuting, late weekends as nightlife).
4. Overlay tract demographics on each area.
5. Profile the least-active areas, and check them against prior bikeshare research and MBTA service gaps.

**Course deadlines:** midterm checkpoint November 2 (clean data, a working workflow, early visualizations); final submission December 15.

---

## 2. Rules the team works by

* **Python only.** The course does not allow ArcGIS Pro, QGIS or R.
* **Every line must be explainable without AI.** At grading the team must reproduce the outputs and explain the code line by line. Prefer techniques taught in class, and keep outside complexity low. Where a line goes beyond class material, it gets a comment saying what it does.
* **Reproducible from a fresh clone.** `README.md` gives the steps. Pin package versions in `requirements.txt`.
* **Raw data stays off GitHub; cleaned data goes on GitHub.** GitHub refuses files over 100 MB, so the raw trips and the trip-level files are in `.gitignore` and rebuilt by the notebooks.
* **Measure in feet.** Every spatial operation uses `EPSG:2249` (NAD83 / Massachusetts Mainland, US survey feet), Boston's equivalent of the `EPSG:2227` used for Berkeley in class.
* **Missing is not zero.** A missing estimate stays `NaN`, so it cannot be counted as a low value.

---

## 3. Datasets

### In use

| Data | Source | Version | Where it lives | In git? |
|---|---|---|---|---|
| Bluebikes trips | [Bluebikes system data](https://bluebikes.com/system-data), files at `https://s3.amazonaws.com/hubway-data/` | January to December 2024, 12 monthly CSVs, 4,751,790 trips | `data/raw/bluebikes/` (downloaded by `00_download_trips.ipynb`) | No (1.1 GB) |
| Census data | ACS from the [Census Reporter API](https://censusreporter.org), tables B01003, B19013, B03002, B08201, all 235 Suffolk County tracts | **ACS 2020–2024 5-year** (`acs2024_5yr`, pinned) | `data/raw/acs/acs5_suffolk_tracts.json` | Yes |
| Tract and city boundaries | Census cartographic boundary files | 2023, Massachusetts, 1:500,000 | `data/raw/boundaries/` | Yes |

**Bluebikes trip columns** (13 per trip): `ride_id`, `rideable_type`, `started_at`, `ended_at`, `start_station_name`, `start_station_id`, `end_station_name`, `end_station_id`, `start_lat`, `start_lng`, `end_lat`, `end_lng`, `member_casual`.

**Why each ACS table:**

| Table | Gives | Why we need it |
|---|---|---|
| B01003 | population | The denominator for per-resident rates, and the residential flag (500+ people) |
| B19013 | median household income | The core equity variable |
| B03002 | race and ethnicity | Lets us ask whether majority-Black or majority-Hispanic neighborhoods are underserved (the team's first GSI question) |
| B08201 | households with no vehicle | Measures need for alternatives to a car |

### Considered and not used

| Data | Why not |
|---|---|
| ACS **1-year** tables (`2024_ACS_one_year_Boston.csv`, `ACSST1Y2023.S0802`, `ACSST1Y2024.S0802`), downloaded 2026-10-07 | Each holds a single row, "Boston city." The 1-year survey is only published for areas of 65,000 people or more, so it has no tracts and cannot compare neighborhoods. Kept in the team's Drive, not in the repo. |
| Bluebikes August and September 2026 | Outside the study period. |
| Bluebikes 2021–2023 | The proposal's original period was 2021–2025; the scope was cut to 2024. Note for expansion: Bluebikes changed its file format in **April 2023**. Earlier files use different columns (`tripduration`, `starttime`, `usertype`, `bikeid`, `postal code`) and numeric station IDs (`386`) instead of codes (`A32011`). January to March 2023 also include the rider's home ZIP code (93% filled in). |
| Census Bureau API directly | Now requires an API key. Census Reporter serves the same ACS tables without one. |

---

## 4. The pipeline

| Notebook | Does | Reads | Writes |
|---|---|---|---|
| `00_download_trips.ipynb` | Downloads the 12 monthly 2024 trip files, extracting only the CSV from each zip | Bluebikes S3 | `data/raw/bluebikes/` |
| `01_clean_data.ipynb` | Cleans the trips one month at a time, builds station and tract tables, joins the census data | raw trips, ACS, boundaries | `data/clean/`, `data/interim/trips/` |
| `02_visualizing_stations.ipynb` | Maps the stations: on Boston's tracts, over median household income, and colored by trips started and trips ended | `data/clean/` | figures in the notebook |

### Cleaning rules (in `01_clean_data.ipynb`)

| Rule | Removed in 2024 | Why |
|---|---:|---|
| 1. No start station | 1,435 | Cannot be placed on the map |
| 2. Shorter than 1 minute | 17,473 | Only January to May contain these; Bluebikes changed its export in June 2024, so removing them everywhere keeps months comparable |
| 3. Longer than 24 hours | 3,702 | Lost or unreturned bikes |
| 4. Started outside 2024 | 0 | Kept in case the study period changes |
| 5. Already in the previous month's file | 123 | Rides crossing midnight on May 31 appear in both the May and June files |

4,751,790 trips in, **4,729,057** out (0.48% removed). Trips with no end station (about 0.1%) are kept, because trips are counted where they start.

### Clean outputs (`data/clean/`)

| File | One row per | Key columns |
|---|---|---|
| `boston_tracts.csv` / `.geojson` | Boston tract (204) | `GEOID`, `population`, `median_hh_income`, `pct_black_nh`, `pct_hispanic`, `pct_hh_no_vehicle`, `residential`, `stations`, `trips_started`, `trips_per_1k_residents` |
| `tract_trips_by_month.csv` | Boston tract (204) | one column per month of trips started |
| `stations.csv` | station (529) | `station_id`, `station_name`, `lat`, `lon`, `in_boston`, `GEOID` |
| `station_month.csv` | station and month (5,736) | `trips_started`, `trips_ended`, `member_trips`, `ebike_trips`, `weekend_trips` |
| `cleaning_log.csv` | monthly file | rows in, rows removed by each rule, rows out |

The trip-level files in `data/interim/trips/` also keep `hour`, `is_weekend`, `is_member` and `is_ebike` for each trip, for the time-of-day and origin-destination work still to come.

---

## 5. Decisions and why

| Date | Decision | Reason |
|---|---|---|
| 2026-09-21 | Study Boston's Bluebikes rather than the Bay Area | Chosen by the team for the proposal |
| 2026-10-20 | Use ACS 5-year tract data instead of the 1-year city tables | The 1-year tables have no neighborhoods |
| 2026-10-20 | Scope: 2023–2024, then **2024 only** for now | Team decision, to expand later |
| 2026-10-20 | Count trips at the **start** station | The closest available proxy for where riders get on |
| 2026-10-20 | Work at the **census tract** level | The smallest unit with ACS income and race data |
| 2026-10-20 | Keep tracts with **no station** (as zeros) | They are the possible equity gap; dropping them would hide it |
| 2026-10-20 | A tract is **Boston** if its centroid is inside the city boundary | Uses a week 7 technique. It drops two tracts (9801.01 Logan Airport area, 9815.01 Harbor Islands) whose centroids fall outside the city line; neither has residents or a station |
| 2026-10-20 | A tract is **residential** if it has 500+ people | Parks, the airport and the islands would give meaningless per-resident rates |
| 2026-10-20 | Rewrite the notebooks with simple, class-taught techniques | The team must explain every line at grading without AI. The rewrite gives identical results |
| 2026-10-20 | Clean one month at a time and summarize inside the loop | Combining all of 2024 needs about 1.2 GB of memory, risky on DataHub |

---

## 6. Key numbers (2024)

* 4,729,057 clean trips; 529 stations, **281 in the City of Boston**
* 204 Boston tracts, 190 residential; **65 Boston tracts have no station**
* 18 Suffolk County tracts have no income estimate; 3 are top-coded at $250,001
* 12 stations were recorded under more than one name

Early pattern, from a September 2025 test run for the lightning talk (not from this pipeline): the highest-income fifth of Boston tracts started about 1.8 times as many trips per resident as the lowest-income fifth, but the pattern was not a steady climb, because student tracts near Northeastern and Wentworth have low household incomes and heavy use.

---

## 7. Open issues and questions

**Analysis questions (raised for the professor check-in, around 2026-10-21):**

* **Use or access?** Trips per resident measures *use*, inflated downtown by commuters and in student tracts. A cleaner *access* measure would be the share of each tract's residents within a quarter mile (1,320 ft) of a station, using week 7 buffers. Also needed: a definition of "underserved" with a number attached.
* **Student and downtown tracts:** exclude them, flag them (for example with ACS school-enrollment data), or keep them and explain them?
* **ACS margins of error** are not yet included; tract estimates can be uncertain.
* Is one year (2024) enough for the midterm?

**Professor's feedback from the check-in:** *(not yet recorded; add it here)*

**Known technical issues:**

* `02_visualizing_stations.ipynb`, the trips-**ended** map (cell 4): its legend says "Trips started in 2024" and its title matches the trips-started map. Both should say ended.
* Running with `pyproj` 3.7.1 (the current pin) instead of 3.8.0 changes 31 station latitudes in the 15th decimal place. Counts are unaffected, but the files are then not byte-for-byte identical across machines. Rounding `lat` and `lon` to 6 decimals in `01_clean_data.ipynb` would fix this.
* Daylight saving time: timestamps have no time zone. Trips crossing the November 3 change can seem to end before they start (37, removed by rule 2); trips crossing March 10 look an hour longer.
* Time of day (`hour`) and origin-destination pairs are in the trip-level files but not summarized yet.

---

## Changelog

Newest first.

### 2026-10-26
* **Oliver:** maps of trips started and trips ended per station added to `02_visualizing_stations.ipynb`; small update to `00_download_trips.ipynb` ([`6cb6fdb`]).
* **George:** added this `context.md`, and added `.claude/` to `.gitignore` (folders the Claude Code assistant creates locally, not project files).

### 2026-10-21
* **Oliver:** re-ran notebooks 00 and 01 and changed the `pyproj` pin from 3.8.0 to 3.7.1 in `requirements.txt` ([`46bcf7c`]).
* **Oliver:** started `02_visualizing_stations.ipynb`, with maps of the stations on Boston's tracts and over median household income ([`7ad96ed`], [`ca4686f`]).
* Professor check-in. Planned questions: use vs. access, and how to treat student and downtown tracts. *(Outcome to be recorded in section 7.)*

### 2026-10-20
* **George, with Claude (AI) assistance:** built the data pipeline.
  * Found that the team's ACS 1-year files are city-level only, and replaced them with ACS 2020–2024 5-year tract data.
  * Profiled the trip files and found: sub-1-minute trips only in January to May 2024; trips over 24 hours; trips filed in the neighboring month's file; 123 rides duplicated across the May and June files.
  * Scope changed from 2021–2025 to 2023–2024, then to **2024 only**.
  * Made the pipeline reproducible: a download notebook, the ACS release pinned, `requirements.txt`, `.gitignore`. Verified that a fresh clone in a new environment rebuilds byte-for-byte identical outputs.
  * Rewrote both notebooks with simpler, class-taught techniques so every line can be explained; results identical to the earlier version.
* **George:** pushed the notebooks, cleaned data and README to GitHub ([`1ab272d`]), replacing the starter README and adding the project's rules to GitHub's template `.gitignore`.
* **George:** created the repository ([`d167700`]).

### 2026-10-07
* **Team:** downloaded the ACS 1-year tables and the Bluebikes trip files for 2024 and August–September 2026 into the shared Drive.

### 2026-09-21
* **Team:** submitted the proposal and lightning talk ("Who Gets a Bluebike?": Boston, 2021–2025, Bluebikes trips plus census income).
  * The draft's claim that Bluebikes exists only in Boston was wrong: the system spans about 14 municipalities, and the study is limited to the City of Boston.

[`6cb6fdb`]: https://github.com/CYPLAN-101-Fall-2026-Group-23/bluebikes_equity/commit/6cb6fdb
[`ca4686f`]: https://github.com/CYPLAN-101-Fall-2026-Group-23/bluebikes_equity/commit/ca4686f
[`7ad96ed`]: https://github.com/CYPLAN-101-Fall-2026-Group-23/bluebikes_equity/commit/7ad96ed
[`46bcf7c`]: https://github.com/CYPLAN-101-Fall-2026-Group-23/bluebikes_equity/commit/46bcf7c
[`1ab272d`]: https://github.com/CYPLAN-101-Fall-2026-Group-23/bluebikes_equity/commit/1ab272d
[`d167700`]: https://github.com/CYPLAN-101-Fall-2026-Group-23/bluebikes_equity/commit/d167700
