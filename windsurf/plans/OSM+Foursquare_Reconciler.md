# OSM + Foursquare Smart Reconciler PoC Plan

## Notes

- Project Scope: Merge and deduplicate POIs from OpenStreetMap (Overpass) and Foursquare OS Places for a chosen bounding box.
- Output: "Gold" GeoJSON layer with provenance and confidence scores.
- Target duration: ~10 developer-hours.
- Embedding model: all-MiniLM-L6-v2 (free, 384-dim).
- Key Libraries: duckdb, pandas, pyarrow, requests, tqdm, sentence-transformers, geopy, shapely, fastapi, uvicorn, pydantic, geopandas, python-dotenv.
- A Python 3.10 virtual environment was created by Poetry. The plan specified 3.11, but this is acceptable.
- Foursquare data will be fetched via their API using credentials stored in a `.env` file.
- Foursquare API integration successful. Key learnings:
  - Base URL: `https://places-api.foursquare.com`
  - Headers: `Authorization: Bearer <API_KEY>`, `X-Places-Api-Version: <date>`, `Accept: application/json`
  - Bbox params: `sw=min_lat,min_lon`, `ne=max_lat,max_lon`
  - Single API key (not client ID/secret).
- OSM data uses `lon` for longitude, while Foursquare uses `lng`. This needs to be handled consistently during feature engineering and matching.
- Feature engineering and matching will explore multiple techniques (e.g., semantic vs. string-based name similarity, different spatial thresholds, varied scoring weights) to compare efficacy.
- Feature tables (`fsq_features`, `osm_features`) will use a hybrid schema: core features as dedicated columns, and additional/experimental features in a JSON column.
- Success Criteria:
  - `make demo` starts FastAPI, `/poi` returns ≥ 500 merged POIs.
  - Precision ≥ 85% on manual 20-sample audit.
  - README covers setup in ≤ 3 commands.
  - Repo passes `pytest` and `pre-commit run --all-files`.
- HuggingFace Hub token is required for downloading models. Resolved 401 errors by creating a "Read" token and loading it from `.env` via `HUGGINGFACE_HUB_READER_TOKEN` variable in the script.
- Debugging of `ModuleNotFoundError` for `pytest` revealed that the test runner was not using the correct Python environment managed by Poetry. The issue was resolved by installing `pytest` as a dev dependency (`poetry add --dev pytest`) and running tests with an explicit `PYTHONPATH` (`PYTHONPATH=$(pwd) poetry run pytest ...`) to ensure project modules are discoverable.
- Trigram Implementation: Implemented as a sorted list of unique trigrams from a name string that has been lowercased and stripped of all non-alphanumeric characters (including spaces). This is a best practice for robust fuzzy matching.
- String-based heuristics implemented: `name_lower`, `name_no_punct`, `name_no_punct_space`, `name_no_legal_suffix`, `name_token_count`, `name_length`. These are stored in the `extra_features` JSON column.
- Haversine distance function implemented in Python and tested successfully.
- Spatial proximity features will be implemented by registering the Haversine function as a DuckDB UDF and using SQL-based candidate generation for efficiency and to demonstrate advanced data engineering skills.
- The Haversine function has been successfully registered as a DuckDB UDF (after fixing a `TypeError` in the `create_function` call arguments), enabling direct use in SQL queries for scalable spatial analysis.
- A `run.py` script orchestrates the pipeline, prompting for a bounding box with error handling for empty results. The data fetching scripts have been significantly debugged: created integration tests, fixed Foursquare API auth/endpoint issues, and resolved a critical argparse bug with negative coordinates in the OSM script.
- `run.py` was updated to robustly handle empty or unreadable Parquet files by deleting and re-triggering the fetch if necessary.
- Successfully debugged and fixed `fetch_fsq.py`. The script now correctly uses the v3 API endpoint (`https://api.foursquare.com/v3/places/search`), handles pagination, maps `bbox` to `sw` and `ne` parameters, and accepts a `--query` argument, returning a valid and complete dataset.
- A `TypeError` was encountered in `feature_engineering.py` when processing the OSM `tags` column, which was sometimes a pre-parsed dictionary instead of a JSON string. The script was updated to handle both data types robustly.
- Implemented a candidate scoring module in `scripts/feature_engineering.py`. The score is a weighted sum of spatial proximity, category match, and phone/website match.
- Initial runs produced no candidate pairs due to a strict 25m spatial join threshold. The threshold was made configurable via a `--distance-threshold` command-line argument to allow for flexible experimentation.
- The `--clean` functionality was correctly moved from `feature_engineering.py` to `run.py`, ensuring data is purged, re-fetched from APIs, and then the feature engineering pipeline is run.
- `run.py` is the main orchestrator for the pipeline. It correctly passes arguments to sub-scripts (e.g., fetch scripts using `f"--bbox={bbox}"`) and ensures the `--clean` workflow correctly re-fetches data before proceeding.
- Successfully orchestrated the end-to-end pipeline using `run.py` with CLI arguments for `--clean`, `--bbox`, `--query`, and `--distance-threshold`.
- Updated `blog_notes.md` with the latest progress on pipeline orchestration.
- Updated `README.md` with comprehensive project information, setup, and usage instructions.
- Implemented a 'fast path' in `fetch_fsq.py` to use a local CSV (`data/ny_places_jan_2022.csv`) if it exists, bypassing the API. This required adding a schema mapping step to transform the CSV columns to match the pipeline's expected schema, resolving a `Binder Error` during DuckDB ingestion.

## Task List

### 1. Project Skeleton (≈ 0.5 h)

- [x] Initialize Poetry project and manage dependencies:
  - [x] Initialize Poetry (`poetry init` for `smart-reconciler` executed, `pyproject.toml` created).
  - [x] Create `requirements.txt` (e.g., using `poetry export`, if explicitly needed alongside Poetry).
  - [x] Install dev dependencies for code quality (Black, isort, flake8) using `poetry add --group dev ...`.
- [x] Set up directory layout: `src/`, `notebooks/`, `data/raw/`, `data/processed/`, `scripts/`, `tests/`.
- [x] Configure pre-commit hooks (e.g., create `.pre-commit-config.yaml` and run `pre-commit install`).

### 2. Environment & Dependencies (≈ 0.5 h)

- [x] Set up Python 3.11 environment (e.g., using `poetry env use 3.11` or ensuring system Python is 3.11).
- [x] Install core project libraries (duckdb, pandas, pyarrow, requests, tqdm, sentence-transformers, geopy, shapely, fastapi, uvicorn, pydantic, geopandas) using `poetry add ...`.
- [x] Install `python-dotenv` to manage environment variables for API keys.
- [x] Confirm embedding model `all-MiniLM-L6-v2` is accessible (typically by ensuring `sentence-transformers` is installed).

### 3. Data Acquisition (≈ 1 h)

- [x] Update `scripts/fetch_fsq.py` to:
  - [x] Load Foursquare API credentials from `.env`.
  - [x] Implement an authenticated API call to fetch POIs for a given bounding box, ensuring headers match the working cURL request (e.g., `Authorization: Bearer <API_KEY>`, `X-Places-Api-Version`).
  - [x] Normalize the JSON response to the required DataFrame structure.
  - [x] Save the result as Parquet in `data/raw/`.
- [x] Verify Foursquare data acquisition script (`scripts/fetch_fsq.py`):
    - [x] Run the script with a test bounding box.
    - [x] Inspect the output Parquet file in `data/raw/` for correctness (columns, data types, expected number of records).
- [x] Configure and run `scripts/fetch_osm.py` to query OpenStreetMap (OSM) using Overpass API for the same bbox, normalize, and save as Parquet in `data/raw/`. (Script scaffolded)
- [x] Debug fetch scripts to ensure they return data.
  - [x] Created integration tests for `fetch_fsq.py` and `fetch_osm.py` that validate against live APIs.
  - [x] Fixed Foursquare script: corrected API endpoint, added `Bearer` prefix to auth header, and improved error handling.
  - [x] Fixed OSM script: resolved `argparse` issue with negative coordinates by using `--bbox=<value>` syntax in tests.
  - [x] Diagnose and fix `fetch_fsq.py` returning an anomalous single record.
    - [x] Add a `--query` argument to `fetch_fsq.py` to allow for more specific searches (e.g., 'food', 'store').
    - [x] Modify the API call to correctly use `sw` and `ne` for the bounding box, and include the new query parameter.
    - [x] Implement pagination to handle multiple pages of results from the Foursquare API, using the `cursor` from the response header.
    - [x] Enhance verbose logging to print the raw Foursquare API JSON response for easier debugging.

### 4. Ingestion into DuckDB (≈ 1 h)

- [x] Create DuckDB database `smart.db`
- [x] Load data from `data/raw/fsq_merged.parquet` (or relevant FSQ parquet) into `fsq_raw` table.
- [x] Load data from `data/raw/osm_merged.parquet` (or relevant OSM parquet) into `osm_raw` table.
- [x] Add Bayer index on `id` for both `fsq_raw` and `osm_raw`.
- [x] Add spatial index on `(lat,lng)` for FSQ table and `(lat,lon)` for OSM table.

### 5. Feature Engineering & Candidate Generation (`scripts/feature_engineering.py`) (≈ 2.0 h)

- [x] Create scaffold for `scripts/feature_engineering.py` with a hybrid schema design.
- [x] Implement and store multiple name similarity features in the feature engineering script:
    - [x] Implement semantic embeddings (`SentenceTransformer.encode(name)` → `FLOAT[384]`)
    - [x] Implement N-gram similarity scores (e.g., trigram).
    - [x] Implement string-based heuristics (e.g., lowercasing, punctuation removal, legal suffix removal).
- [x] Implement and store spatial proximity features:
    - [x] Haversine distance function (Python) implemented and tested.
    - [x] Register Haversine distance as a DuckDB UDF (TypeError in `create_function` call fixed).
    - [x] Create `candidate_pairs` table using SQL-based spatial join (after `fsq_features` and `osm_features` are created):
      - [x] Join `fsq_features` and `osm_features` where `haversine_distance(...) < 0.025`.
      - [x] Store `fsq_id`, `osm_id`, and the calculated distance.
- [x] Implement and store category matching features:
    - [x] Implement category canonicalization (e.g., using a lookup dictionary).
    - [x] Option for loose/fuzzy category matching.
- [x] Implement and store other attribute heuristics (e.g., phone/website exact match flags).
- [x] Ensure all features are stored in the `*_features` tables in DuckDB.

### 6. Matching, Deduplication & Strategy Comparison (≈ 2.5 h)

- [ ] Implement flexible blocking strategies (e.g., based on spatial proximity, category match, or initial name similarity).
- [x] Develop a scoring module that can combine various features with configurable weights (e.g., `score = w1*name_sim_semantic + w2*name_sim_trigram + w3*(1−norm_dist) + ...`).
- [ ] Experiment with different scoring formulas and thresholds to determine optimal matching.
- [ ] Implement conflict resolution logic (e.g., prefer FSQ attribute unless OSM newer, keep both provenance URLs).
- [ ] Implement confidence scoring based on agreement and match strength.
- [ ] Output merged POIs to table `poi_merged`, potentially tagging them with the matching strategy used.
- [x] Refactor the `--clean` functionality:
  - [x] Move the data purging logic from `feature_engineering.py` to `run.py`.
  - [x] In `run.py`, have `--clean` delete all `.parquet` files in `data/raw/` and `data/processed/`, and also delete the `smart.db` file.
  - [x] Ensure `run.py` automatically re-fetches data after a clean, before running feature engineering.
  - [x] Remove the `--clean` argument and associated logic from `feature_engineering.py`.

### 7. API & Output (≈ 2 h)

- [ ] Create FastAPI app
- [ ] Implement endpoint `POST /merge/run {bbox}`: Triggers pipeline, returns job id
- [ ] Implement endpoint `GET /poi`: Returns merged GeoJSON list
- [ ] Implement endpoint `GET /poi/{id}`: Returns single POI with provenance & confidence
- [ ] Ensure async DuckDB queries
- [ ] Serve with `uvicorn --reload`

### 8. Tests & Evaluation (≈ 1.5 h)

- [x] Write unit tests (pytest) for core functions (e.g., distance UDF, specific similarity metrics).
- [ ] Develop a methodology for evaluating different matching strategies:
    - [ ] Perform manual spot-checks (e.g., inspect 20 random matches per strategy).
    - [ ] Log precision/recall in `README.md` or a dedicated evaluation document, comparing different approaches.
- [ ] Enhance `scripts/eval.py` to support evaluation of different matching configurations.
- [x] Write tests for trigram implementation.
- [x] Write tests for string-based heuristics.
- [x] Write tests for Haversine distance function.
- [x] Write integration tests for data fetching scripts (`fetch_fsq.py`, `fetch_osm.py`).
- [x] Write unit tests for category and phone/website normalization functions.

### 9. Docs & Demo (≈ 0.5 h)

- [x] Create `README.md` with problem, approach diagram, setup, `make run`, sample curl
- [ ] Create `why-reprompt.md` linking PoC choices to Reprompt’s needs
- [ ] Record a short Loom video (<2 min walk-through)

### 10. Git LFS Setup (≈ 0.5 h)

- [x] Install `git-lfs` system-wide (command `sudo apt install git-lfs` initiated by user).
- [x] Initialize `git-lfs` for the repository (`git lfs install`).
- [x] Configure `git-lfs` to track large file types (e.g., `*.parquet`, `*.csv` in `.gitattributes`).
- [x] Configure `git-lfs` to track database files (e.g., `*.db` in `.gitattributes`).

### 11. Bonus Objectives (LLM Integration)

- [ ] Implement an LLM-powered interactive data exploration chatbot (CLI or web) to allow natural language queries about the POI data, pipeline, or results.

## Current Goal

Implement FastAPI app
