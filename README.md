# Beer Sheva Apartment Rental Price Analysis

## Dataset: `appartments_dedup_equal_query.csv`

A curated dataset of **458 apartment listings** in Beer Sheva, Israel, with rental prices, apartment attributes, and walking distances to key points of interest for students. The dataset was constructed by aggregating listings from multiple online real estate platforms and enriching them with geospatial features.

### Columns

| Column | Type | Description |
|--------|------|-------------|
| `id` | string | Unique listing identifier |
| `description` | string | Free-text Hebrew listing description |
| `price` | integer | Monthly rent in NIS |
| `street` | string | Street address |
| `neighborhood` | string | Neighborhood (Alef, Bet, Gimel, Shchuna D) |
| `bedrooms` | float | Number of bedrooms |
| `squareFeet` | float | Apartment size in square meters |
| `hasBalcony` | integer | Whether the apartment has a balcony (0/1) |
| `hasShelter` | integer | Whether the building has a shared shelter (0/1) |
| `hasMamad` | integer | Whether the apartment has a protected room (0/1) |
| `listingLink` | string | URL to the original listing |
| `source` | string | Data source (Dorin, Yad2, Komo, Facebook, Homeless, Madlan, Homendy, Emmaleh) |
| `propertyType` | string | Type of property (e.g., apartment, garden apartment) |
| `latitude` | float | Latitude (WGS84) |
| `longitude` | float | Longitude (WGS84) |
| `group` | string | Listing metadata (e.g., Yad2 price history) |
| `geometry` | string | GeoJSON Point representation of coordinates |
| `walking_distance_bgu_gate_m` | float | Walking distance to the nearest BGU gate (meters) |
| `walking_distance_block_center_m` | float | Walking distance to "The Block" student entertainment center (meters) |
| `walking_distance_ringelblum_m` | float | Walking distance to Ringelblum street — a major bar and nightlife strip (meters) |
| `walking_distance_sophrim_park_m` | float | Walking distance to Sofrim Park (meters) |
| `walking_distance_metzadah_m` | float | Walking distance to Metzadah street (meters) |
| `walking_distance_nearest_bar_m` | float | Walking distance to the nearest bar (meters) |
| `walking_distance_nearest_cafe_m` | float | Walking distance to the nearest cafe (meters) |
| `walking_distance_nearest_restaurant_m` | float | Walking distance to the nearest restaurant (meters) |
| `walking_distance_student_hub_m` | float | Walking distance to the nearest student hub — the minimum of Ringelblum, Block Center, and Sofrim Park distances (meters) |

---

## Data Aggregation & Preparation Methodology

### 1. Scraping (`scraper.py`)

Listings were collected from **five sources** using purpose-built scrapers:

- **Dorin API** (`DorinScraper`): REST API client fetching paginated listings from `dorin.app` with full structured data (price, bedrooms, square footage, coordinates, amenities).
- **Facebook Marketplace** (`FacebookScraper`): Playwright browser automation to scrape Facebook group posts, with Gemini 2.5 Flash (`gemini-2.5-flash`) used to parse unstructured Hebrew post text into structured fields via JSON-schema-enforced LLM extraction.
- **Emmalah** (`EmmalehScraper`): BeautifulSoup-based scraper that discovers listing pages via pagination and extracts prices from raw text.
- **Homendy** (`HomendyScraper`): BeautifulSoup scraper querying `/apts/{id}/modal` endpoints to extract listing details.
- **Yad2** (`Yad2Scraper`): REST API client targeting Yad2's internal feed API (`gw.yad2.co.il/feed-search-legacy`) to retrieve structured listing data.

All scrapers normalize their output to a shared 41-column schema and write to `Dorin_dump.csv`.

### 2. Data Preparation (`dataprep.py`)

The raw scraped data undergoes:

- **Type normalization**: Price, latitude, and longitude are coerced to numeric types.
- **Boolean standardization**: Boolean fields (`hasBalcony`, `hasShelter`, `hasMamad`, etc.) default to `False` where missing.
- **Spatial neighborhood enrichment**: Apartment coordinates are converted to Shapely Point geometries and spatially joined (via GeoPandas `sjoin` with `predicate='within'`) against OSM neighborhood polygons (Alef: 12164851, Bet: 12164854, Gimel: 12164853, Dalet: 961302720) loaded from `Neighborhoods.json`. Where a point falls within a polygon, the neighborhood name is overridden with the OSM-standardized name.

Output: `appartments.csv`

### 3. Walking Distance Computation (`distance_calc.py`)

Walking distances from each apartment to points of interest are computed in two stages:

**Stage A — Fetch OSM geometries and amenities:**

- **BGU Gates**: Four precise OSM node IDs (1667827765, 1486123013, 3677540945, 1667756897) representing the main entrances to Ben-Gurion University.
- **The Block**: OSM way ID 1009305349 — a popular student entertainment complex. Converted to a Polygon (closed way) or convex hull (open way).
- **Ringelblum**: OSM way ID 361973550 — the central bar and nightlife street. Geometry computed as a convex hull of its node coordinates.
- **Sofrim Park**: OSM relation ID 12684979 — a large urban park. Geometry computed as the convex hull of all member way nodes.
- **Metzadah**: OSM way ID 368168562 — a commercial street. Handled as a LineString.
- **North Train Station**: OSM node ID 621348196 — Beer Sheva's north railway station.
- **Amenities (bars, cafes, restaurants)**: Queried via the Overpass API for all nodes and ways with `amenity=bar`, `amenity=cafe`, and `amenity=restaurant` within Beer Sheva.

**Stage B — Distance calculation:**

For each apartment with valid coordinates:

1. **Distance to BGU gates**: Walking distance computed via OSRM foot profile to each of the four gate nodes; the minimum distance is retained.
2. **Distance to polygon/line geometries**: The nearest point on the target geometry is found using Shapely's `nearest_points`, then walking distance is computed from the apartment to that nearest point via OSRM.
3. **Distance to nearest amenities**: Haversine distance is used to find the closest bar, cafe, and restaurant, then OSRM walking distance is computed to each.
4. **Fallback**: If OSRM routing fails (timeout or error), the distance is estimated as straight-line (haversine) distance × 1.3 (a standard urban detour factor).

OSRM is queried via `http://router.project-osrm.org/route/v1/foot/{lon1},{lat1};{lon2},{lat2}?overview=false`.

Output: `appartments_w_improved_distances.csv`

### 4. Feature Verification (`verify_features.py`)

To ensure data quality, each listing's extracted features are verified against the raw Hebrew description text using Gemini 2.5 Flash with JSON-structured output. For each row, the LLM receives the description and the extracted values, and is asked to correct any discrepancies. Features verified include: price, bedrooms, squareFeet, hasParking, hasElevator, hasBalcony, isPetFriendly, hasShelter, hasMamad, isFlatshare, isSublet. Processing is done concurrently (5 workers via ThreadPoolExecutor).

Output: `appartments_verified.csv`

### 5. Unification, Deduplication & Final Processing

Within `explore.ipynb`:

- Two data sources are unified: cleaned Excel data (`cleanDataPt2.xlsx`) and updated CSV data (`updated_data.csv`), which includes the distance-enriched verified data with geocoded locations.
- Duplicate listings are removed by ID.
- Boolean columns are converted to 0/1 integers.
- **`walking_distance_student_hub_m`** is computed as the element-wise minimum of `walking_distance_ringelblum_m`, `walking_distance_block_center_m`, and `walking_distance_sophrim_park_m`, representing the distance to the nearest major student congregation point.
- Hebrew neighborhood names are mapped to English equivalents via a hardcoded dictionary (`update_names.py`).

Output: `appartments_dedup_equal_query.csv`

---

## Feature Engineering for Analysis (in `explore_analysis_final_query.Rmd`)

The following derived features are computed during analysis:

| Feature | Formula |
|---------|---------|
| `roommates` | `max(bedrooms - 1, 1)` |
| `price_per_roommate` | `price / roommates` |
| `log_price` | `log(price)` |
| `sqft_per_bedroom` | `squareFeet / max(bedrooms, 0.5)` |
| `spaciousness_zscore` | `z-score(sqft_per_bedroom)` within each neighborhood |
| `log_walking_distance_*` | `log(distance_in_meters + 1)` for each distance column |

A gravity/pull transformation (`1000 / (distance + 1)`) was also explored in earlier versions but replaced by the log transform in the final analysis.
