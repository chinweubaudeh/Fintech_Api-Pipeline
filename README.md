 FinTrack Foreign Exchange API ETL Pipeline

An end-to-end **FinTech Data Engineering ETL pipeline** that extracts foreign-exchange rate data from a public REST API, transforms and validates the data using **Python and Pandas**, preserves raw and cleaned datasets, and loads the final data into **PostgreSQL** for downstream analytics.

---

 Project Overview

The **FinTrack Foreign Exchange API Pipeline** is a data engineering project designed to demonstrate how financial data can be collected from an external API and processed through a complete ETL workflow.

The pipeline retrieves foreign-exchange rates using **USD as the base currency**, stores the original API response as raw JSON, converts the exchange rates into a structured Pandas DataFrame, performs data cleaning and validation, saves the processed data as CSV, and finally loads the dataset into PostgreSQL.

The project demonstrates practical skills in:

- REST API integration
- Python programming
- Pandas data transformation
- Data validation
- PostgreSQL
- SQLAlchemy
- Environment-variable management
- Error handling
- ETL pipeline development
- Data quality verification

---

 Project Objectives

The main objectives of this project are to:

- Extract live foreign-exchange data from a REST API.
- Preserve the original API response for traceability.
- Convert semi-structured JSON data into structured tabular data.
- Clean and standardise exchange-rate information.
- Validate the transformed dataset before loading.
- Store processed data as CSV.
- Load validated data into PostgreSQL.
- Protect database credentials using environment variables.
- Verify successful database loading.
- Build a reusable foundation for a production-style financial data pipeline.

---

 Pipeline Architecture

```text
                  Foreign Exchange API
                          │
                          ▼
                  API Data Extraction
                    Python Requests
                          │
                          ▼
                ┌───────────────────┐
                │    Raw Layer      │
                │ Timestamped JSON  │
                └─────────┬─────────┘
                          │
                          ▼
                Pandas Transformation
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
         Cleaning     Validation   Standardisation
             │            │            │
             └────────────┼────────────┘
                          │
                          ▼
                ┌───────────────────┐
                │   Cleaned Layer   │
                │       CSV         │
                └─────────┬─────────┘
                          │
                          ▼
                     SQLAlchemy
                          │
                          ▼
                ┌───────────────────┐
                │    PostgreSQL     │
                │  exchange_rates   │
                └─────────┬─────────┘
                          │
                          ▼
                   Load Verification
```

---

 ETL Workflow

The pipeline follows a traditional **Extract → Transform → Load** architecture.

 1. Extract

The application retrieves current exchange-rate data from:

```text
https://open.er-api.com/v6/latest/USD
```

USD is currently configured as the base currency.

The API response is collected using the Python `requests` library.

The extraction logic also handles common API failures including:

- Request timeout
- Connection errors
- HTTP errors
- Invalid JSON responses
- General request failures
- Unsuccessful API responses

---

 2. Preserve Raw Data

Before performing transformations, the complete API response is saved as a timestamped JSON file.

Example:

```text
data/raw_data/exchange_rates_YYYYMMDD_HHMMSS.json
```

Keeping the raw API response provides:

- Data lineage
- Auditability
- Troubleshooting capability
- Historical records
- Reprocessing capability

This follows an important data engineering principle:

```text
Never modify the original source data.
```

---
 3. Transform

The API exchange-rate dictionary is converted into a Pandas DataFrame.

The resulting dataset contains the following fields:

| Column | Description |
|---|---|
| `base_currency` | Base currency used by the API |
| `currency_code` | Target currency code |
| `exchange_rate` | Exchange rate relative to the base currency |
| `retrieved_at` | Time associated with the API exchange-rate update |
| `extracted_at` | Time the pipeline extracted the data |

---

 Data Cleaning

Several transformations are performed before loading the dataset.

 Currency Standardisation

Currency codes are:

- Converted to string
- Whitespace trimmed
- Converted to uppercase

Example:

```python
exchange_rates["currency_code"] = (
    exchange_rates["currency_code"]
    .astype("string")
    .str.strip()
    .str.upper()
)
```

---

 Numeric Conversion

Exchange rates are explicitly converted to numeric values:

```python
exchange_rates["exchange_rate"] = pd.to_numeric(
    exchange_rates["exchange_rate"],
    errors="coerce"
)
```

Invalid numeric values are converted to missing values so that they can be identified and removed during validation.

---
 Missing Values

Records containing missing values in required columns are removed.

Required fields include:

```text
base_currency
currency_code
exchange_rate
retrieved_at
extracted_at
```

---

 Invalid Exchange Rates

Only positive exchange rates are retained:

```python
exchange_rates = exchange_rates[
    exchange_rates["exchange_rate"] > 0
]
```

---

 Duplicate Handling

Duplicate records are removed using:

```text
base_currency
currency_code
retrieved_at
```

as the uniqueness criteria.

---

 Data Validation

Before the dataset can be loaded into PostgreSQL, it passes several validation checks.

The validation layer checks that:

- All required columns exist.
- The transformed DataFrame is not empty.
- Required values do not contain nulls.
- Currency codes are not duplicated.
- Exchange rates are greater than zero.
- Currency codes contain three characters.

If any validation rule fails, the pipeline raises an exception rather than loading poor-quality data.

This implements a simple **fail-fast data quality strategy**.

---

 Cleaned Data Layer

After successful validation, the cleaned dataset is saved as:

```text
data/cleaned_data/exchange_rates_cleaned.csv
```

This provides an intermediate processed dataset that can be used independently of the database.

The architecture therefore maintains two data layers:

```text
Raw API Data
     │
     ▼
data/raw_data/
     │
     ▼
Transformation
     │
     ▼
data/cleaned_data/
```

---

 PostgreSQL Integration

The pipeline uses:

- SQLAlchemy
- psycopg2
- PostgreSQL

to load the cleaned exchange-rate dataset into a relational database.

The target table is:

```text
exchange_rates
```

The current implementation uses:

```python
dataframe.to_sql(
    name="exchange_rates",
    con=engine,
    if_exists="replace",
    index=False,
    method="multi",
    chunksize=500
)
```

Using `method="multi"` and `chunksize=500` supports more efficient batch insertion compared with inserting every record individually.

---

  Environment Variables

Database credentials are not hard-coded into the Python pipeline.

Instead, they are loaded from a `.env` file using `python-dotenv`.

Create a `.env` file in the project root:

```env
DB_USER=your_postgres_username
DB_PASSWORD=your_postgres_password
DB_HOST=localhost
DB_PORT=5432
DB_NAME=your_database_name
```

The pipeline expects the following variables:

```text
DB_USER
DB_PASSWORD
DB_HOST
DB_PORT
DB_NAME
```

> Never commit your `.env` file to GitHub.

Ensure `.env` is included in your `.gitignore`.

---

  Database Connection Validation

Before loading the data, the pipeline tests the PostgreSQL connection.

It retrieves:

```sql
SELECT
    current_database(),
    current_user,
    version();
```

This confirms that the pipeline can successfully communicate with PostgreSQL before attempting the data load.

---

  Load Verification

After loading the dataset, the pipeline executes:

```sql
SELECT COUNT(*)
FROM exchange_rates;
```

The PostgreSQL record count is compared with the number of records in the transformed Pandas DataFrame.

```text
Pandas DataFrame Row Count
            │
            ▼
        Compare
            ▲
            │
PostgreSQL Table Row Count
```

If the two counts do not match, the pipeline raises an error.

This provides an additional level of **ETL reconciliation and data-qualitye
