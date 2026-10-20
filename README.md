# Who Gets a Bluebike?

**Spatial and demographic equity in Boston's Bluebikes network, 2024**

CP 101 final project · George Huang, Oliver Kirk, Handson Li, Shaan Patel

Across the City of Boston in 2024, do Bluebikes stations and trips concentrate in higher-income neighborhoods, and who lives in the areas with the least bikeshare activity?

This repository holds the code and the **cleaned** data. The raw trip files are too large for GitHub, so a notebook downloads them from Bluebikes. Anyone can rebuild every cleaned file from scratch with the steps below.

---

## Reproduce the cleaned data

You need **Python 3.11 or newer**, about **2 GB of free disk space**, and about 2 GB of memory. The whole process takes about five minutes.

**1. Get the code**

```bash
git clone <this repository's URL>
cd bluebikes-equity
```

**2. Create an environment and install the packages**

```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

`requirements.txt` pins the exact versions the results were made with.

**3. Run the two notebooks, in order**

Open Jupyter (`jupyter lab`), then open each notebook and choose **Run All**:

1. `notebooks/00_download_trips.ipynb` downloads the twelve 2024 trip files (about 1.1 GB) into `data/raw/bluebikes/`. It takes about a minute, and it skips files you already have.
2. `notebooks/01_clean_data.ipynb` cleans them and writes the tables in `data/clean/`. It takes one to two minutes.

**Check it worked:** the last line of `data/clean/cleaning_log.csv` should read

```
all files,4751790,1435,17473,3702,0,123,4729057
```

That is 4,751,790 raw trips in and 4,729,057 clean trips out. This was tested by starting from an empty folder holding only this repository, in a brand-new Python environment. All six cleaned files came out byte-for-byte identical.

---

## What is in this repository

```
bluebikes-equity/
├── README.md
├── requirements.txt              exact package versions
├── .gitignore                    keeps the large files off GitHub
├── notebooks/
│   ├── 00_download_trips.ipynb   downloads the 2024 trip files
│   └── 01_clean_data.ipynb       raw files in, clean tables out
└── data/
    ├── raw/
    │   ├── bluebikes/            NOT on GitHub: 12 monthly trip CSVs (made by notebook 00)
    │   ├── acs/                  census data, saved as downloaded
    │   └── boundaries/           census tract and city boundary shapefiles
    ├── interim/                  NOT on GitHub: clean trips, one Parquet file per month (made by notebook 01)
    └── clean/                    the cleaned tables the analysis uses
```

The raw trip files are 37 to 134 MB each, and GitHub refuses files over 100 MB. The clean trip files in `data/interim/` are also large. Both are listed in `.gitignore` and rebuilt by the notebooks. The census and boundary files are small (about 1 MB together), so they are stored in the repository and never need downloading. Everything in `data/clean/` adds up to under 1 MB.

---

## How the cleaning notebook works

`01_clean_data.ipynb` works **one month at a time**. Each monthly file is cleaned, summarized and saved before the next is read, so the computer never holds all 4.7 million trips at once. It has seven sections.

### 1. Clean and summarize each month

The loop reads each file, turns the start and end times into datetimes, and works out each trip's length and month. It then removes rows by five rules, one at a time, recording how many each rule removed in `cleaning_log.csv`:

| Rule | Removed in 2024 | Why |
|---|---:|---|
| 1. No start station | 1,435 | The trip cannot be placed on the map. |
| 2. Shorter than 1 minute | 17,473 | January to May include these trips and June to December do not, because Bluebikes changed how it exports data in June 2024. Removing them from every month keeps the months comparable. This also removes trips that seem to end before they start. |
| 3. Longer than 24 hours | 3,702 | Almost always a lost or unreturned bike, not a ride. |
| 4. Started outside 2024 | 0 | A file can include trips from just before midnight at the end of the previous month. None fell outside 2024, but the rule stays in case the study period changes. |
| 5. Already in the previous month's file | 123 | Rides that crossed midnight on May 31 appear in **both** the May and the June file. They are the same trips, so the first copy is kept, as in Lab 6. A ride lasts at most 24 hours, so a copy can only be in the neighboring file. |

In total, 22,733 of 4,751,790 trips (0.48%) are removed. Trips with a start station but no end station (about 0.1%) are **kept**, because trips are counted by where they start.

After cleaning, the loop saves the month's trips to `data/interim/` and makes three small tables for that month: trips started at each station, trips ended at each station, and each name a station was recorded under with its typical location.

A sanity check then confirms that no ride is left in two files.

### 2. One row per station

* **Name:** the name used for the most trips. Twelve stations were recorded under more than one name.
* **Location:** the median of the station's monthly median locations. Recorded coordinates shift slightly from trip to trip, and the median ignores the odd outlier.

### 3. Trips per station per month

The monthly tables are combined. A file can hold a few trips that started in the previous month, so one station and month can appear twice; adding them with `groupby().sum()` merges them.

### 4. Census data for each tract

The census numbers are the **ACS 2020–2024 5-year estimates** for every Suffolk County census tract.

The team first downloaded **ACS 1-year** tables. Those describe the whole city in one row, because the 1-year survey is only published for areas of 65,000 people or more, so it has no tracts and cannot compare neighborhoods.

A tract the Census could not estimate income for stays **missing** (`NaN`), never zero, so it cannot fall into the lowest income group by mistake.

### 5. Boston's tracts, and which tract each station is in

* Suffolk County also contains Chelsea, Revere and Winthrop. A tract counts as **Boston** if its centroid is inside the City of Boston boundary. That gives **204 Boston tracts**.
* Two tracts are shaped so irregularly that their centroid falls outside the city line: 9801.01 (the Logan Airport area) and 9815.01 (the Harbor Islands). They are left out. Neither has residents or a station.
* A tract counts as **residential** if at least 500 people live there (**190 tracts**).
* Each station is matched to the tract it sits in with a spatial join. **281 of the 529 stations** are in Boston; the rest are in Cambridge, Somerville, Brookline and other towns the system serves.

Every layer is converted to `EPSG:2249` (NAD83 / Massachusetts Mainland, in feet) before any spatial test. It is Boston's equivalent of the `EPSG:2227` used for Berkeley in class.

### 6. Trips per tract

A tract's trips are the trips started at the stations inside it. Tracts with no station are kept, with zeros: **65 of the 204 Boston tracts have no station**.

### 7. Save

The notebook writes the tables below to `data/clean/`.

---

## The cleaned data

All counts are **trips started** in 2024, unless a column says otherwise. `GEOID` is the 11-character census tract code (state, county, tract), stored as text so it keeps its full form.

**`boston_tracts.csv`** and **`boston_tracts.geojson`**: one row per Boston tract (204 rows). The GeoJSON has the same columns plus each tract's shape, for mapping.

| Column | Meaning |
|---|---|
| `GEOID`, `NAMELSAD` | tract code and name |
| `population` | total population (ACS table B01003) |
| `median_hh_income` | median household income in dollars (B19013); blank when not estimated |
| `income_top_coded` | `True` when income shows as $250,001, the ACS's way of writing "$250,000 or more" |
| `white_nh`, `black_nh`, `asian_nh`, `hispanic`, `race_total` | people by race and ethnicity (B03002); `_nh` means not Hispanic or Latino |
| `pct_white_nh`, `pct_black_nh`, `pct_asian_nh`, `pct_hispanic` | the same, as percentages of `race_total` |
| `households`, `hh_no_vehicle`, `pct_hh_no_vehicle` | households, and households with no vehicle (B08201) |
| `residential` | `True` if at least 500 people live in the tract |
| `stations` | stations in the tract with at least one trip started in 2024 |
| `trips_started` | trips started at those stations in 2024 |
| `trips_per_1k_residents` | trips started per 1,000 residents; blank for non-residential tracts |

**`tract_trips_by_month.csv`**: one row per Boston tract (204 rows), one column per month (`2024-01` to `2024-12`) holding the trips started there.

**`stations.csv`**: one row per station (529 rows)

| Column | Meaning |
|---|---|
| `station_id`, `station_name` | Bluebikes' station code and its most-used name |
| `lat`, `lon` | typical location, in degrees |
| `names_used` | number of different names recorded for the station |
| `first_month`, `last_month`, `months_active` | when trips were started from the station |
| `in_boston` | `True` if the station is inside the City of Boston |
| `GEOID` | the tract the station is in (blank outside Boston) |

**`station_month.csv`**: one row per station per month (5,736 rows)

| Column | Meaning |
|---|---|
| `station_id`, `month` | the station, and the month as `YYYY-MM` |
| `trips_started`, `trips_ended` | trips starting and ending at the station |
| `member_trips`, `ebike_trips`, `weekend_trips` | trips started by members, on e-bikes, and on Saturdays or Sundays |
| `in_boston`, `GEOID` | as in `stations.csv` |

**`cleaning_log.csv`**: for each monthly file, the rows in, the rows each rule removed, and the rows out, plus a total row.

---

## Data sources

| Data | Source | Version |
|---|---|---|
| Bluebikes trips | [Bluebikes system data](https://bluebikes.com/system-data); files at `https://s3.amazonaws.com/hubway-data/` | January to December 2024 |
| Census data | American Community Survey, downloaded from the [Census Reporter API](https://censusreporter.org): `https://api.censusreporter.org/1.0/data/show/acs2024_5yr?table_ids=B19013,B01003,B03002,B08201&geo_ids=140|05000US25025` | ACS 2020–2024 5-year |
| Tract and city boundaries | [Census cartographic boundary files](https://www.census.gov/geographies/mapping-files/time-series/geo/cartographic-boundary.html) | 2023, Massachusetts, 1:500,000 |

---

## Known limitations

* **Trips measure use, not access.** Trips started in a tract count everyone who rides from there, not only the people who live there. Downtown tracts such as those around South Station and Government Center look busy because people who work there ride from them.
* **Student tracts.** Tracts near Northeastern, Wentworth and the Longwood colleges show low household incomes but heavy bikeshare use, because students have low household incomes. Income alone is a rough measure of need there.
* **Neighborhoods, not riders.** The trip data says nothing about who the rider is. Linking a trip to its tract's income describes the neighborhood, not the person, and concluding something about individuals from it is the ecological fallacy.
* **Daylight saving time.** The timestamps have no time zone. Trips that cross the November 3 clock change can seem to end before they start (37 trips, removed by rule 2), and trips that cross the March 10 change look one hour longer than they were.
* **Different years of data.** The census estimates cover 2020–2024, the boundaries are from 2023, and the trips are from 2024. All three use the 2020 census tracts, so the tracts line up.

---

## Extending to other years

Change `STUDY_YEAR` at the top of **both** notebooks, then run them again.

**Before 2024, check the file format first.** Bluebikes changed its file format in April 2023. Files up to March 2023 use different column names (`tripduration`, `starttime`, `usertype`, `bikeid`, `postal code`), and their station IDs are numbers such as `386` instead of codes such as `A32011`. The notebook does not handle the older format yet.
