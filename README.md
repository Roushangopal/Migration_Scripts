# Sybase ASE (on-prem) → Amazon DynamoDB migration kit

Client used to read Sybase: **SQuirreL SQL**. Everything else is Python + boto3.

```
sybase_to_dynamodb/
├── sql/                         <- run these in SQuirreL (Steps 2)
│   ├── 01_tables.sql
│   ├── 02_columns.sql
│   ├── 03_indexes.sql
│   └── 04_foreign_keys.sql
├── schema_export/               <- SQuirreL CSV exports go here
├── data/                        <- table data CSVs go here
├── output/                      <- generated: table JSON, catalog, reports
├── config.json                  <- edit this first
├── common.py
├── 01_generate_dynamodb_schema.py
├── 02_create_dynamodb_tables.py
├── 03_extract_data.py
├── 04_load_data.py
└── 05_validate.py
```

## What "preserve the schema" means here

DynamoDB is schemaless apart from its keys, so the schema is preserved in two ways:

1. **Structure**: 1 Sybase table → 1 DynamoDB table; every column becomes an attribute with its **original
   name**; primary key → table key; other indexes → Global Secondary Indexes (GSIs).
2. **Metadata**: the complete Sybase definition (type, length, precision, scale, nullability, IDENTITY,
   defaults, all indexes incl. unique/clustered flags, foreign keys) is saved in `output/catalog.json` and in a
   DynamoDB table called `_sybase_schema_catalog`.

### Type mapping

| Sybase | DynamoDB | Notes |
|---|---|---|
| tinyint, smallint, int, bigint, unsigned *, numeric, decimal, float, real, money | **N** | exact decimals preserved |
| char, varchar, nchar, nvarchar, unichar, univarchar, text, unitext | **S** | trailing spaces trimmed (`trim_char_columns`) |
| datetime, smalldatetime, bigdatetime, date | **S** | ISO-8601 (sortable) |
| time, bigtime | **S** | `HH:MM:SS[.ffffff]` |
| bit | **BOOL** | becomes **N** (0/1) if used in a key |
| binary, varbinary, image, timestamp | **B** | hex in CSV → bytes |
| NULL | attribute omitted | sparse items |

### Key mapping

| Source PK | DynamoDB key |
|---|---|
| 1 column | HASH = column |
| 2 columns | HASH = col1, RANGE = col2 |
| 3+ columns | HASH = col1, RANGE = `_sk` (cols 2..n joined by `#`) |
| none (heap) | first all-NOT-NULL unique index, else synthetic `_row_id` |

Non-PK indexes follow the same pattern (`_gsiN_sk` for composites).

### Things DynamoDB cannot do (documented, then checked by `05_validate.py`)
Foreign keys, unique constraints on non-key columns, CHECK constraints, defaults, IDENTITY auto-increment,
joins, and ad-hoc queries on non-indexed columns. Item size limit is **400 KB** (watch `text`/`image` columns).

---

## Step 0 – Prerequisites

* Python 3.9+, `pip install -r requirements.txt`
* AWS credentials configured (`aws configure` or env vars) with `dynamodb:*` on the target account
* Java 8+ (only for `03_extract_data.py` / `05_validate.py --jdbc`)
* Edit **`config.json`**: `aws_region`, `table_prefix`, `billing_mode`, Sybase `jdbc_url`/`jar_path`/`user`
* Optional – test safely first with DynamoDB Local: set `"dynamodb_endpoint_url": "http://localhost:8000"`

## Step 1 – Connect SQuirreL to Sybase

1. **Drivers → New Driver**
   * *jTDS (recommended)*: class `net.sourceforge.jtds.jdbc.Driver`, URL `jdbc:jtds:sybase://<host>:<port>/<database>`, jar `jtds-1.3.1.jar`
   * *jConnect*: class `com.sybase.jdbc4.jdbc.SybDriver`, URL `jdbc:sybase:Tds:<host>:<port>/<database>`, jar `jconn4.jar`
2. **Aliases → New Alias**: pick the driver, enter the URL (include the database name), user, password → **Test → Connect**.
3. **Remove the 100-row limit** (otherwise exports are truncated): *Session → Session Properties → SQL* →
   untick **"Limit rows"**.
4. Use the same jar path in `config.json → sybase.jar_path`.

## Step 2 – Extract the Sybase schema (in SQuirreL)

Open the **SQL** tab, load each file from `sql/`, put the cursor in the statement and press **Ctrl+Enter**.
For each result: right-click the result grid / use the result toolbar → **Export to CSV**, choose
*complete result set*, **comma** separator, **header row on**, **UTF-8**, and save as:

| Run | Save as |
|---|---|
| `sql/01_tables.sql` | `schema_export/tables.csv` |
| `sql/02_columns.sql` | `schema_export/columns.csv` |
| `sql/03_indexes.sql` | `schema_export/indexes.csv` |
| `sql/04_foreign_keys.sql` | `schema_export/foreign_keys.csv` (skip if no rows) |

> Run them against the correct database (it is part of the alias URL). For several databases, repeat the whole
> flow per database and use a different `table_prefix`.

## Step 3 – Generate the DynamoDB definitions

```bash
python 01_generate_dynamodb_schema.py
```

Review **`output/schema_report.md`** (key choices, tables without a PK, unique indexes, LOB columns,
unknown types). Optionally edit `output/tables/*.json` to tune keys before creating anything.
Also produced: `output/row_counts.sql` (used in Step 7).

## Step 4 – Create the tables and store the schema catalog

```bash
python 02_create_dynamodb_tables.py --dry-run
python 02_create_dynamodb_tables.py
```

(Equivalent AWS CLI for one table: `aws dynamodb create-table --cli-input-json file://output/tables/orders.json`)

## Step 5 – Get the data out of Sybase

**Option A – automated (recommended), uses the same JDBC jar as SQuirreL**
```bash
set SYBASE_PASSWORD=********          # Windows   (export SYBASE_PASSWORD=... on Linux/macOS)
python 03_extract_data.py                       # all tables
python 03_extract_data.py --tables dbo.orders   # or selected tables
```

**Option B – manual from SQuirreL** (fine for a handful of small tables): run `SELECT * FROM owner.table`,
export the complete result to `data/<owner>.<table>.csv` (UTF-8, header row, comma). NULL → empty cell,
binary → hex.

For very large tables (tens of millions of rows) extract in key ranges, or use `bcp`.

## Step 6 – Load the data

```bash
python 04_load_data.py --dry-run          # converts & validates every row, writes nothing
python 04_load_data.py --workers 4        # real load (idempotent, safe to re-run)
```

Rejected rows (NULL key, bad number, item > 400 KB…) are written to `output/errors/<table>.errors.csv`.
With on-demand billing no capacity planning is needed; with provisioned billing raise WCUs for the load.

## Step 7 – Validate

1. In SQuirreL run **`output/row_counts.sql`** (one statement) and export it to `output/source_counts.csv`
   (or skip this and use `--jdbc` below).
2. ```bash
   python 05_validate.py --spot-check 200          # or: python 05_validate.py --jdbc --spot-check 200
   ```
   Checks row counts, random row-by-row comparison, unique-index duplicates and foreign-key orphans.
   Report: `output/validation_report.csv`. Exit code is non-zero on any failure.

## Step 8 – Cut-over (suggested)

* Freeze writes on Sybase (or run a final incremental load: re-run steps 5–6 for changed tables – puts are idempotent).
* For **continuous replication / minimal downtime** consider AWS DMS (SAP ASE source → DynamoDB target) on top of
  the tables created here, using this kit for the schema and initial load.
* Repoint the application to DynamoDB using the key mapping in `output/catalog.json`; rewrite SQL joins as
  key lookups / GSI queries.
* Keep Sybase read-only until validation passes and the app has run on DynamoDB for an agreed period.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Only 100 rows exported from SQuirreL | Untick *Limit rows* (Step 1.3) and export the *complete* result |
| `index_col` / `sysindexes` errors in SQL 03 | Make sure you're connected to the target database, not `master` |
| `Missing file ... schema_export/*.csv` | Export names must match Step 2 exactly |
| `InvalidOperation` in errors file | Number formatted with thousands separators – re-export without locale formatting |
| `ResourceInUseException` / table exists | Step 4 skips existing tables; delete the table to recreate |
| `jaydebeapi` can't find Java | Install a JRE and set `JAVA_HOME` |
