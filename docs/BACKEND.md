# Backend Architecture Documentation
Hello homeboys! Welcome to my documentation for the backend of my code if you have any questions hit me up as though I am sigeon pex 7326889157 ;)
## Overview

The backend folder (`backend/`) contains all server-side logic—essentially all the processing that happens behind the scenes. This includes three main components:

### 1. **Data Processing Pipelines**
Think of a pipeline like an assembly line for data. When you upload a CSV file with traffic violations:
- **Step 1 (Cleaning):** The raw CSV data gets validated and standardized. Column names are normalized, missing values are handled, and bad data is removed
- **Step 2 (Ingestion):** The cleaned data gets loaded into DuckDB (our database). This converts thousands of spreadsheet rows into organized database tables
- **Step 3 (Detection):** The system queries the database to identify super speeders and warning drivers based on violation thresholds

Example: Raw CSV (100 rows) → Cleaned Data (95 valid rows) → Database Tables → Detection Results (2 super speeders found)

### 2. **Database Schema**
A schema is the blueprint for how data is organized. It defines:
- **What tables exist** (e.g., `fct_violations` stores all violation records, `dim_driver` stores driver info)
- **What columns each table has** (e.g., violation date, license plate, points assessed)
- **How tables connect** (e.g., a violation record points to a driver record)
- **What indexes exist** (for fast searching, like indexing by driver_id)

In our system, we have 6 main tables with 145,000+ violation records across 77,000+ unique drivers.

### 3. **Business Logic**
The system automatically detects:
- **Super Speeders:** Drivers with ≥16 camera tickets in 12 months OR ≥11 violation points in 18 months
- **Warning Drivers:** Drivers approaching these thresholds (12-15 tickets OR 8-10 points)

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## File Structure

```
backend/
├── app.py                     # FastAPI web application
├── src/
│   ├── __init__.py           # Package initialization
│   ├── cleaning.py           # Data validation & standardization
│   ├── ingestion.py          # DuckDB warehouse loading
│   └── super_speeder_detector.py  # Detection logic
├── sql/
│   └── 01_schema.sql         # Database schema definition
└── tests/
    └── test_super_speeder_detector.py  # Unit tests
```

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Core Modules

### `app.py` - FastAPI Web Server 

**Purpose:** Main web application entry point so users can upload CSVs and view results as well as other resources.

**Key Responsibilities:**
- Handle HTTP requests (upload, display, resources) - Make the web app web
- Manage file uploads and validation - This is where the magic happens this essentially is the controller that ties everything together for the backend
- Orchestrate data pipeline (clean → ingest → detect) - Call other modules to process data (obvious what is does)
- Render Jinja2 templates for HTML pages - What Jinja2 is used for is basicaly templating engine for python that allow us to generate web pages dynamically or make them look pretty
- Error handling and logging - Log errors and important events so if there is any issues we as a group can debug them 

**Main Routes:**
| Route | Method | Purpose |
|-------|--------|---------|
| `/` | GET | Upload interface |
| `/upload` | POST | Process CSV upload |
| `/results` | GET | Display detection results |
| `/driver/{id}` | GET | Individual driver details |
| `/resources` | GET | Program information |
| `/health` | GET | Health check |

**Dependencies:**
- FastAPI, Uvicorn (server side framework FastAPI is used to build APIs with Python and Uvicorn is an ASGI server to run FastAPI apps and ASGI is a specification for Python that allows for internal communication between the web server and web applications)
- Jinja2Templates (This helps render the templates and make them look pretty)
- Pathlib (path management)

**Running:**
```bash
cd /path/to/project/backend
python app.py
# Server runs on http://localhost:8000
```

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### `src/cleaning.py` - Data Cleaning Module

**Purpose:** Validate and standardize raw CSV data before database loading

**Key Classes:**
- **DataCleaner:** Handles both speed camera and traffic violation datasets

**Key Functions:**
```python
DataCleaner.clean_speed_cameras(df: pd.DataFrame) → pd.DataFrame
  - Normalizes column names
  - Validates critical fields (plate, violation_date)
  - Handles missing values
  - Returns cleaned DataFrame

DataCleaner.clean_traffic_violations(df: pd.DataFrame) → pd.DataFrame
  - Similar process for violation-specific fields
  - Validates license IDs and point values

clean_and_export(input_dir, output_dir, file_patterns, strict_mode) → Tuple[pd.DataFrame, pd.DataFrame]
  - Batch processes multiple files
  - Exports to Parquet format which is efficient for storage and querying which is useful for us because we are using duckdb as our database and duckdb works really well with parquet files
  - Returns (speed_cameras_df, violations_df)
```

**Data Validation:**
- **Speed Cameras:** plate, violation_date, violation, fine_amount
- **Violations:** license, violation_date, violation_description, points

**Output:** Parquet files in `../data/cleaned/`

**CLI Usage:**
```bash
python src/cleaning.py \
  --input-dir ../data/raw \
  --output-dir ../data/cleaned \
  --pattern "test*" \
  --strict
```

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### `src/ingestion.py` - DuckDB Ingestion Pipeline

**Purpose:** Load cleaned data into DuckDB analytics warehouse

**Key Classes:**
- **DuckDBIngester:** Manages database connections and data loading

**Key Functions:**
```python
DuckDBIngester.connect()
  - Establish DuckDB connection
  - Install httpfs extension

DuckDBIngester.initialize_schema(schema_file: str) → bool
  - Execute all SQL statements from 01_schema.sql
  - Create tables and indexes

DuckDBIngester.load_speed_cameras(parquet_file: str) → int
  - Load speed camera violations into fct_violations

DuckDBIngester.load_traffic_violations(parquet_file: str) → int
  - Load traffic violations into fct_violations

ingest_pipeline(duckdb_path, schema_file, cleaned_dir, fresh_start) → bool
  - Complete pipeline: connect → schema → load → compute
```

**Database Output:**
- `fct_violations` (145,000+ rows)
- `dim_driver`, `dim_violation_type`, `dim_time`
- `agg_repeat_offenders` (aggregate table)

**CLI Usage:**
```bash
python src/ingestion.py \
  --duckdb-path ../data/duckdb/test.duckdb \
  --cleaned-dir ../data/cleaned \
  --schema-file ./sql/01_schema.sql \
  --fresh
```

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### `src/super_speeder_detector.py` - Detection Engine

**Purpose:** Identify drivers meeting super speeder and warning thresholds

**Key Classes:**
- **SuperSpeederDetector:** Queries DuckDB and applies detection logic

**Key Methods:**
```python
SuperSpeederDetector.detect_super_speeders() → Tuple[List[Dict], List[Dict], Dict]
  Returns:
    - super_speeders: List of 1,332+ drivers meeting thresholds
    - warning_drivers: List of 227+ drivers approaching thresholds
    - stats: Summary statistics

SuperSpeederDetector.get_ingestion_stats() → Dict
  Returns:
    - total_violations, unique_drivers, unique_plates
    - date_range, violation breakdown by source
```

**Detection Logic:**

**Super Speeder:** Driver qualifies if:
$$\text{Speed Camera Tickets (12mo)} \geq 16 \quad \text{OR} \quad \text{Violation Points (18mo)} \geq 11$$

**Warning Driver:** Driver approaches if:
$$12 \leq \text{Speed Camera Tickets} < 16 \quad \text{OR} \quad 8 \leq \text{Points} < 11$$

**SQL Queries:** Dynamic temporal queries using date subtraction

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### `sql/01_schema.sql` - Database Schema

**Purpose:** Define complete DuckDB warehouse structure

**Tables Created:**

1. **fct_violations** (Fact Table)
   - Central table with 145,000+ violation records
   - Columns: summons_number, driver_id, violation_code, points_assessed, violation_date, data_source, etc.

2. **dim_driver** (Driver Dimension)
   - 77,475 unique drivers
   - Columns: driver_id, registration_state

3. **dim_violation_type** (Violation Dimension)
   - Categorical violations (speeding, unsafe speed, etc.)

4. **dim_time** (Time Dimension)
   - Date hierarchy for temporal analysis

5. **agg_repeat_offenders** (Aggregate)
   - Pre-computed violation counts per driver
   - Optimizes detection queries

6. **agg_risk_scores_by_location** (Aggregate)
   - Geographic violation patterns

**Key Indexes:** driver_id, violation_date, data_source

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### `tests/test_super_speeder_detector.py` - Unit Tests

**Purpose:** Validate detection logic and data integrity

**Test Coverage:**
```python
test_detect_super_speeders()
  - Verify count matches expected threshold

test_license_plate_not_null_super_speeders()
  - All drivers have valid license plates

test_license_plate_matches_driver_id()
  - Data model integrity check

test_warning_drivers_count()
  - Warning threshold logic validation

test_unique_plates_count()
  - Distinct plate count accuracy

test_get_ingestion_stats()
  - Statistics accuracy

test_driver_details_has_license_plate()
  - Individual driver view has data
```

**Run Tests:**
```bash
cd /path/to/project/backend
python -m pytest tests/ -v
```

**Results:** All 7 tests passing on sample data (1,332 super speeders, 227 warning drivers)

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Data Flow Diagram

```
[User CSV Upload]
         ↓
[app.py POST /upload]
         ↓
[cleaning.py: validate & standardize]
         ↓
[Parquet files: data/cleaned/]
         ↓
[ingestion.py: load into DuckDB]
         ↓
[DuckDB database: data/duckdb/test.duckdb]
         ↓
[super_speeder_detector.py: query & detect]
         ↓
[results.html: display findings]
```

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Configuration & Paths

**Relative to backend root (`backend/`):**

```python
# In app.py
BASE_DIR = Path(__file__).parent  # backend/
ROOT_DIR = BASE_DIR.parent       # project/

TEMPLATES_DIR = ROOT_DIR / "frontend" / "templates"
DATA_DIR = ROOT_DIR / "data"
RAW_DATA_DIR = DATA_DIR / "raw"
CLEANED_DIR = DATA_DIR / "cleaned"
DUCKDB_PATH = DATA_DIR / "duckdb" / "test.duckdb"
SCHEMA_FILE = BASE_DIR / "sql" / "01_schema.sql"
```

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Performance Notes

- **Query time:** ~0.5-2s per 200-record upload
- **DuckDB:** Columnar storage optimizes aggregations
- **Parquet caching:** Avoids repeated CSV parsing
- **Indexes:** On driver_id and violation_date for fast filtering

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Dependencies

```
fastapi==0.104.1
uvicorn==0.24.0
duckdb==0.9.2
pandas==2.1.3
jinja2==3.1.2
python-multipart==0.0.6
```

Install with:
```bash
pip install -r requirements.txt
# or
uv sync
```

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Future Enhancements

1. **Real-time Monitoring:** WebSocket updates for live violations
2. **ML Models:** Predict recidivism and intervention success
3. **API Integration:** Connect to DMV systems
4. **Batch Processing:** Handle datasets 100,000+ rows
5. **Audit Trail:** Log all operations for accountability

