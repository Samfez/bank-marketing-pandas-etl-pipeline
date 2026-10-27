# Bank Marketing — Pandas ETL Pipeline

![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-3.0-150458?logo=pandas&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?logo=jupyter&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

An ETL pipeline that turns a flat, inconsistently formatted bank marketing export into three clean tables with consistent types, ready to load into a relational database for downstream analysis.

## Overview

A Portuguese bank ran a telemarketing campaign to sell term deposits. The raw export mixes three kinds of data in one table: client demographics, campaign interactions and macroeconomic indicators. It also stores booleans as `"yes"`/`"no"`/`"unknown"` strings, splits dates across separate columns and uses inconsistent category labels.

This project **extracts** the raw CSV, **transforms** it into normalized and correctly typed DataFrames, and **loads** them as three CSV files whose schemas line up with a PostgreSQL database design.

| | |
|---|---|
| **Input** | 1 CSV · 41,188 rows · 16 columns |
| **Output** | 3 CSVs (`client`, `campaign`, `economics`) joined on `client_id` |
| **Stack** | Python 3.12, pandas, NumPy, python-dotenv, Jupyter |

## Pipeline

```
data/raw/bank_marketing.csv
        │
        ▼  Extract ── read the CSV, with its path set in .env
        │
        ▼  Transform
        │    ├─ split into 3 subsets by domain (client / campaign / economics)
        │    ├─ normalize category labels ("." → "_")
        │    ├─ map "unknown" education to NaN
        │    ├─ convert yes/no/unknown and success/failure columns to bool
        │    └─ build last_contact_date (datetime) from day + month + year
        │
        ▼  Load
data/processed/
    ├── client.csv
    ├── campaign.csv
    └── economics.csv
```

## Transformations

| Table | Column | Raw value | Clean value |
|---|---|---|---|
| `client` | `job` | `admin.` | `admin_` |
| `client` | `education` | `basic.4y` · `unknown` | `basic_4y` · `NaN` |
| `client` | `credit_default`, `mortgage` | `yes` · `no` · `unknown` | `True` · `False` · `False` |
| `campaign` | `campaign_outcome` | `yes` · `no` | `True` · `False` |
| `campaign` | `previous_outcome` | `success` · `failure` · `nonexistent` | `True` · `False` · `False` |
| `campaign` | `last_contact_date` | `day=13`, `month=may` | `2022-05-13` |

> **Design decision:** `"unknown"` is mapped to `False` for `credit_default` and `mortgage`, following the target schema, which requires non-null booleans. Note that this is a business assumption: 8,597 rows have an unknown default status, while only 3 rows report an actual default.

## Output schema

**`client.csv`**

| Column | Type | Description |
|---|---|---|
| `client_id` | int | Primary key |
| `age` | int | Client age |
| `job` | str | Job category |
| `marital` | str | Marital status |
| `education` | str (nullable) | Education level |
| `credit_default` | bool | Whether the client has credit in default |
| `mortgage` | bool | Whether the client has a mortgage |

**`campaign.csv`**

| Column | Type | Description |
|---|---|---|
| `client_id` | int | Foreign key → `client` |
| `number_contacts` | int | Contacts made during this campaign |
| `contact_duration` | int | Last contact duration (seconds) |
| `previous_campaign_contacts` | int | Contacts made before this campaign |
| `previous_outcome` | bool | Whether the previous campaign succeeded |
| `campaign_outcome` | bool | Whether the client subscribed to a term deposit |
| `last_contact_date` | date | Date of last contact (`YYYY-MM-DD`) |

**`economics.csv`**

| Column | Type | Description |
|---|---|---|
| `client_id` | int | Foreign key → `client` |
| `cons_price_idx` | float | Consumer price index (monthly) |
| `euribor_three_months` | float | 3-month Euribor rate (daily) |

## Project structure

```
bank-marketing-pandas-etl-pipeline/
├── data/
│   ├── raw/                 # Source CSV (not tracked, see "Getting the data")
│   └── processed/           # Pipeline output (generated)
├── notebooks/
│   └── data_cleaning.ipynb  # ETL pipeline
├── .env.example             # Environment variable template
├── requirements.txt         # Pinned dependencies
├── LICENSE
└── README.md
```

## Getting started

### Prerequisites

- Python **3.11+** (developed and tested on 3.12)
- Jupyter, or VS Code with the Jupyter extension

### 1. Clone and install

```bash
git clone https://github.com/Samfez/bank-marketing-pandas-etl-pipeline.git
cd bank-marketing-pandas-etl-pipeline

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure the environment

```bash
cp .env.example .env            # Windows: copy .env.example .env
```

`.env` holds the name of the raw file inside `data/raw/`:

```
RAW_FILE_NAME=bank_marketing.csv
```

### 3. Getting the data

The dataset is not tracked in this repository. Download `bank_marketing.csv` from the DataCamp project *"Cleaning Bank Marketing Data"* and place it in `data/raw/`. That file is a preprocessed version of the [UCI Bank Marketing dataset](https://archive.ics.uci.edu/dataset/222/bank+marketing).

Expected format: 41,188 rows, comma-separated, with these columns:

```
client_id, age, job, marital, education, credit_default, mortgage, month, day,
contact_duration, number_contacts, previous_campaign_contacts, previous_outcome,
cons_price_idx, euribor_three_months, campaign_outcome
```

### 4. Run the pipeline

Open `notebooks/data_cleaning.ipynb` and select **Run All**. To use Jupyter from the terminal, run:

```bash
pip install jupyterlab
jupyter lab notebooks/data_cleaning.ipynb
```

The three clean files are written to `data/processed/`.

The notebook is committed with its outputs, so you can review every step and its results directly on GitHub without running anything.

> Run the notebook from the `notebooks/` directory. Paths are resolved relative to it (`Path().resolve().parent`).

## Key takeaways

- **11.3%** of contacted clients subscribed to a term deposit, so the target is heavily imbalanced, which matters for any later modeling.
- **1,731** clients have unknown education, kept as `NaN` instead of being imputed.
- Every last contact falls between **March and December 2022**.

## Skills demonstrated

- Data wrangling with pandas: column subsetting, vectorized string operations, mapping, type casting
- Date construction and parsing with `pd.to_datetime`
- Schema-driven data modeling: splitting a flat table into related entities
- Reproducible setup: pinned dependencies, environment-based configuration, separation of raw and processed data

## Roadmap

- [ ] Refactor the notebook into a modular `src/` package (`extract.py`, `transform.py`, `load.py`) with a CLI entry point
- [ ] Add data validation tests (pytest / pandera)
- [ ] Load the output directly into PostgreSQL with SQLAlchemy
- [ ] Add CI with GitHub Actions

## License

Released under the [MIT License](LICENSE).

## Author

[@Samfez](https://github.com/Samfez)
