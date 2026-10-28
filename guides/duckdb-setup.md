# 🦆 DuckDB Setup Guide

A step-by-step guide to installing DuckDB on Ubuntu, loading data, and running analytical queries — all without a separate database server.

> Tested on **Ubuntu 22.04 LTS** and **Ubuntu 24.04 LTS**.

---

## What Is DuckDB?

DuckDB is an **in-process analytical database** — it runs inside your application or CLI session rather than as a separate background server. It is designed for **OLAP** (Online Analytical Processing) workloads: aggregations, joins, and scans over large datasets.

Key characteristics:
- **Serverless** — no daemon, no ports, no configuration files required to get started.
- **Columnar storage** — stores data column-by-column for fast analytical scans.
- **Vectorised execution** — processes batches of rows at a time using SIMD instructions.
- **File-based** — a database is a single `.duckdb` file you can copy, move, or delete like any other file.
- **SQL-first** — full SQL support including window functions, CTEs, `PIVOT`, `UNPIVOT`, and more.
- **Zero-copy reads** — can query Parquet, CSV, JSON, and Arrow files directly without importing data first.

> Think of it as SQLite, but built for analytics instead of transactions.

---

## Prerequisites

- A VPS running Ubuntu 22.04 or 24.04 (ideally set up with the [VPS security guide](../README.md))
- A non-root sudo user (the guide uses `admin`)
- Internet access on the server

---

## Table of Contents

1. [Install the DuckDB CLI](#1-install-the-duckdb-cli)
2. [Install the Python Package](#2-install-the-python-package)
3. [Basic CLI Usage](#3-basic-cli-usage)
4. [Loading Data](#4-loading-data)
5. [Querying Files Directly](#5-querying-files-directly)
6. [Exporting Data](#6-exporting-data)
7. [Using DuckDB in Python](#7-using-duckdb-in-python)
8. [Extensions](#8-extensions)
9. [Performance Tips](#9-performance-tips)
10. [Security Notes](#10-security-notes)

---

## 1. Install the DuckDB CLI

The DuckDB CLI is a self-contained binary — no package manager or runtime dependency needed.

### Option A — Official install script (recommended)

```bash
curl https://install.duckdb.org | bash
```

> The official DuckDB installer script detects your OS and architecture, downloads the latest stable release, and installs it to `~/.duckdb/cli/latest/duckdb`. It also adds the binary to your `$PATH` by appending a line to `~/.bashrc`.

Re-load your shell config to pick up the updated `$PATH`:

```bash
source ~/.bashrc
```

### Option B — Manual download

Use this if you prefer not to pipe a script to bash, or need to install system-wide for all users.

**Step 1 — Download and extract the binary:**

```bash
curl -fsSL https://github.com/duckdb/duckdb/releases/latest/download/duckdb-cli-linux-amd64.tar.gz -o /tmp/duckdb.tar.gz
```

> - `-f` — Fails silently on HTTP errors instead of printing an HTML error page.
> - `-s` — Silent mode (suppresses the progress meter).
> - `-S` — Shows errors even in silent mode.
> - `-L` — Follows HTTP redirects (GitHub releases redirect to the actual asset).
> - The archive is a `.tar.gz` file containing a single `duckdb` binary.

> **ARM server?** Replace `amd64` with `arm64` in the URL for 64-bit ARM VPS instances.

```bash
tar -xzf /tmp/duckdb.tar.gz -C /tmp/
```

> - `-x` — Extract files from the archive.
> - `-z` — Decompress using gzip.
> - `-f` — Specifies the archive filename.
> - `-C /tmp/` — Extracts into `/tmp/` rather than the current directory.

**Step 2 — Install the binary system-wide:**

```bash
sudo mv /tmp/duckdb /usr/local/bin/duckdb
```

> Moves the binary to `/usr/local/bin`, which is on the system `$PATH` — making `duckdb` available as a command for all users.

```bash
sudo chmod +x /usr/local/bin/duckdb
```

> Marks the binary as executable. Without this flag, the OS will refuse to run it.

**Step 3 — Clean up:**

```bash
rm /tmp/duckdb.tar.gz
```

> Removes the downloaded archive now that the binary is installed.

### Verify the installation

```bash
duckdb --version
```

> Prints the installed DuckDB version (e.g., `v2.0.0`). If this returns a version number, the installation succeeded.

---

## 2. Install the Python Package

If you plan to use DuckDB from Python scripts or Jupyter notebooks, install the `duckdb` package via pip.

```bash
pip install duckdb
```

> Installs the `duckdb` Python package from PyPI. This is the most common way to use DuckDB in production Python applications. The package includes its own embedded DuckDB engine — you don't need the CLI binary for this to work.

> **Using a virtual environment?** Always activate it first:
> ```bash
> source .venv/bin/activate
> pip install duckdb
> ```

Verify the installation:

```bash
python3 -c "import duckdb; print(duckdb.__version__)"
```

> Imports the package and prints its version number. Confirms the package is importable from your current Python environment.

---

## 3. Basic CLI Usage

### Start an in-memory session

```bash
duckdb
```

> Opens an interactive DuckDB session with a **temporary in-memory database**. All data is lost when you exit. Useful for quick explorations and one-off queries.

You'll see a prompt like:

```
v1.2.0 ...
Enter ".help" for usage hints.
Connected to a transient in-memory database.
D
```

### Open (or create) a persistent database file

```bash
duckdb ~/mydata.duckdb
```

> Opens the file `mydata.duckdb` as a DuckDB database. If the file doesn't exist, DuckDB creates it automatically. All tables and data are persisted to this file.

### Useful shell commands inside the CLI

```sql
-- Show all tables in the current database
SHOW TABLES;

-- Describe the columns and types of a table
DESCRIBE my_table;

-- Show current DuckDB settings
SELECT * FROM duckdb_settings();

-- Exit the CLI
.quit
```

> - `.quit` — Exits the DuckDB CLI. You can also press `Ctrl+D`.
> - SQL statements must end with a semicolon (`;`).
> - DuckDB is case-insensitive for SQL keywords but preserves case in identifiers.

---

## 4. Loading Data

### From a CSV file

```sql
-- Create a table from a CSV file
CREATE TABLE orders AS
    SELECT * FROM read_csv_auto('orders.csv');
```

> - `read_csv_auto` — Reads a CSV file and **automatically detects** the delimiter, header row, column names, and data types. Handles quoted fields, escape characters, and encoding.
> - `CREATE TABLE ... AS SELECT` — Creates a new table populated with the result of the query. The schema is inferred from the CSV columns.

You can also import into an existing table with explicit types:

```sql
-- Create a table with explicit column types first
CREATE TABLE orders (
    order_id   INTEGER,
    customer   VARCHAR,
    amount     DOUBLE,
    order_date DATE
);

-- Then load data into it
COPY orders FROM 'orders.csv' (HEADER TRUE, DELIMITER ',');
```

> - `COPY ... FROM` — Bulk-loads data from a file into an existing table. Significantly faster than row-by-row `INSERT` statements for large files.
> - `HEADER TRUE` — Tells DuckDB to skip the first row (the header).
> - `DELIMITER ','` — Specifies the field separator character.

### From a Parquet file

```sql
CREATE TABLE events AS
    SELECT * FROM read_parquet('events.parquet');
```

> DuckDB has native Parquet support — no extra configuration needed. Parquet's columnar format means DuckDB can skip columns it doesn't need, making this extremely fast even for very large files.

### From JSON

```sql
CREATE TABLE logs AS
    SELECT * FROM read_json_auto('app.log');
```

> `read_json_auto` handles newline-delimited JSON (NDJSON), JSON arrays, and nested structures. Nested objects and arrays are stored as DuckDB's `STRUCT` and `LIST` types.

---

## 5. Querying Files Directly

One of DuckDB's most powerful features is the ability to **query files without importing them** — no `CREATE TABLE` step needed.

### Query a CSV file directly

```sql
SELECT
    customer,
    SUM(amount) AS total_spent
FROM read_csv_auto('orders.csv')
GROUP BY customer
ORDER BY total_spent DESC
LIMIT 10;
```

> DuckDB reads the CSV on the fly, executes the query, and returns results — without ever creating a table or writing anything to disk. This is ideal for one-off investigations.

### Query multiple Parquet files with a glob pattern

```sql
SELECT COUNT(*) FROM read_parquet('logs/2025/*.parquet');
```

> The `*` wildcard matches all `.parquet` files in the `logs/2025/` directory. DuckDB reads all matching files as if they were one large table — perfect for partitioned datasets.

### Query a remote file over HTTP

```sql
SELECT * FROM read_csv_auto('https://example.com/data.csv') LIMIT 5;
```

> DuckDB can stream remote files directly using HTTP/HTTPS (requires the `httpfs` extension — see [Extensions](#8-extensions)). This means you can query S3 buckets, GitHub raw files, or any public URL without downloading first.

---

## 6. Exporting Data

### Export to CSV

```sql
COPY (SELECT * FROM orders WHERE amount > 100) TO 'big_orders.csv' (HEADER TRUE);
```

> `COPY ... TO` exports the result of any query to a file. `HEADER TRUE` writes column names as the first row. The file is written to the working directory from which you launched `duckdb`.

### Export to Parquet

```sql
COPY orders TO 'orders.parquet' (FORMAT PARQUET);
```

> Exports the entire `orders` table to a Parquet file. Parquet is strongly recommended for large datasets — it's compressed, columnar, and can be read by Spark, Pandas, Polars, BigQuery, and more.

### Export to JSON

```sql
COPY orders TO 'orders.json' (FORMAT JSON);
```

> Writes each row as a JSON object on its own line (NDJSON format). Useful for interoperability with log pipelines or APIs.

---

## 7. Using DuckDB in Python

### In-memory analytics

```python
import duckdb

# Connect to a transient in-memory database
con = duckdb.connect()

# Run a query and get results as a Python list of tuples
result = con.execute("SELECT 42 AS answer, 'hello' AS greeting").fetchall()
print(result)  # [(42, 'hello')]
```

> - `duckdb.connect()` — Opens an in-memory database. No file is created.
> - `.execute()` — Runs a SQL query.
> - `.fetchall()` — Returns all result rows as a list of tuples.

### Persistent database file

```python
import duckdb

# Open (or create) a file-backed database
con = duckdb.connect("mydata.duckdb")

con.execute("""
    CREATE TABLE IF NOT EXISTS products (
        id    INTEGER PRIMARY KEY,
        name  VARCHAR,
        price DOUBLE
    )
""")

con.execute("INSERT INTO products VALUES (1, 'Widget', 9.99)")
con.execute("INSERT INTO products VALUES (2, 'Gadget', 24.99)")

rows = con.execute("SELECT * FROM products ORDER BY price DESC").fetchall()
for row in rows:
    print(row)

con.close()
```

> - `CREATE TABLE IF NOT EXISTS` — Creates the table only if it doesn't already exist. Safe to run on every startup.
> - Always call `con.close()` when you're done, or use a context manager (`with duckdb.connect(...) as con:`).

### Query a Pandas DataFrame directly

```python
import duckdb
import pandas as pd

df = pd.read_csv("sales.csv")

# DuckDB can query the DataFrame directly by name — no import step needed
result = duckdb.query("""
    SELECT
        region,
        SUM(revenue) AS total_revenue
    FROM df
    GROUP BY region
    ORDER BY total_revenue DESC
""").df()

print(result)
```

> - `duckdb.query()` — A shorthand that opens a temporary in-memory connection and runs the query.
> - `FROM df` — References the Python variable `df` (a Pandas DataFrame) directly in SQL. DuckDB scans it using zero-copy Arrow memory sharing where possible.
> - `.df()` — Returns the result as a new Pandas DataFrame.

### Return results as an Arrow table or Polars DataFrame

```python
# Return as a PyArrow Table
arrow_table = con.execute("SELECT * FROM orders").arrow()

# Return as a Polars DataFrame
polars_df = con.execute("SELECT * FROM orders").pl()
```

> DuckDB integrates natively with PyArrow and Polars for zero-copy data transfers — no serialisation overhead when passing data between libraries.

---

## 8. Extensions

DuckDB has a growing ecosystem of extensions for additional data formats and remote storage. Install them from inside a DuckDB session.

### Install and load an extension

```sql
-- Install once (downloads from the DuckDB extension repository)
INSTALL httpfs;

-- Load for this session (must be loaded each session, or set autoload)
LOAD httpfs;
```

> - `INSTALL` — Downloads and stores the extension locally. Only needs to be run once.
> - `LOAD` — Activates the extension for the current session.

### Enable autoloading (recommended)

```sql
SET autoload_known_extensions = true;
SET autoinstall_known_extensions = true;
```

> With these settings, DuckDB automatically installs and loads a known extension the first time you use a function that requires it — no manual `INSTALL`/`LOAD` needed.

### Useful extensions

| Extension | What it adds |
|-----------|-------------|
| `httpfs` | Read/write files over HTTP, HTTPS, and S3 |
| `parquet` | Parquet read/write support (built-in since v0.10) |
| `json` | Advanced JSON functions (built-in since v0.10) |
| `postgres` | Attach a live PostgreSQL database and query it with DuckDB |
| `sqlite` | Attach a SQLite `.db` file and query it directly |
| `spatial` | Geospatial types and functions (PostGIS-style) |
| `excel` | Read `.xlsx` files directly |
| `iceberg` | Read Apache Iceberg tables |
| `delta` | Read Delta Lake tables |

### Example: query an S3 file

```sql
INSTALL httpfs;
LOAD httpfs;

SET s3_region = 'us-east-1';
SET s3_access_key_id = 'YOUR_ACCESS_KEY';
SET s3_secret_access_key = 'YOUR_SECRET_KEY';

SELECT COUNT(*) FROM read_parquet('s3://my-bucket/data/events.parquet');
```

> DuckDB streams the Parquet file from S3 using column and row-group pruning — only the data needed for the query is downloaded.

---

## 9. Performance Tips

### Use Parquet instead of CSV for large datasets

Parquet is a columnar, compressed format. DuckDB can skip entire column groups and row groups it doesn't need, making queries dramatically faster than scanning CSVs.

```sql
-- Convert a CSV to Parquet once
COPY (SELECT * FROM read_csv_auto('data.csv')) TO 'data.parquet' (FORMAT PARQUET);

-- Then query the Parquet file going forward
SELECT * FROM 'data.parquet' WHERE date > '2025-01-01';
```

### Tune memory usage

By default DuckDB uses up to 80% of available RAM. Adjust this for a shared VPS:

```sql
-- Limit DuckDB to 4 GB of RAM
SET memory_limit = '4GB';

-- Set a directory for spilling to disk when memory is exceeded
SET temp_directory = '/tmp/duckdb-spill';
```

> If DuckDB exceeds the memory limit during a query, it spills intermediate results to `temp_directory` instead of crashing. Make sure the directory has enough free disk space.

### Control parallelism

```sql
-- Limit to 4 CPU threads (default: all available cores)
SET threads = 4;
```

> On a shared VPS, limiting threads prevents DuckDB from monopolising all CPU cores during a heavy query.

### Use `EXPLAIN ANALYZE` to profile queries

```sql
EXPLAIN ANALYZE
SELECT customer, SUM(amount)
FROM orders
GROUP BY customer;
```

> Runs the query and prints the execution plan with actual row counts and timing for each operator. Use this to identify bottlenecks — e.g., a sequential scan where a filter could have been pushed down.

---

## 10. Security Notes

### DuckDB has no built-in access control

DuckDB is a file-based database with no concept of users, passwords, or network authentication. Security is enforced entirely at the **operating system and filesystem level**.

- **Set strict file permissions** on your `.duckdb` files:

```bash
chmod 600 ~/mydata.duckdb
```

> Only the owner can read or write the file. No other user or process on the system can access the database.

- **Never store credentials in a `.duckdb` file** that is readable by other users or committed to version control.

### Secrets management for cloud storage

When connecting to S3 or other cloud storage, avoid setting credentials with `SET s3_access_key_id` directly in SQL — those values appear in query logs and shell history. Use DuckDB's **Secrets Manager** instead:

```sql
CREATE SECRET my_s3_secret (
    TYPE S3,
    KEY_ID     'YOUR_ACCESS_KEY',
    SECRET     'YOUR_SECRET_KEY',
    REGION     'us-east-1'
);
```

> Secrets are stored in DuckDB's internal secret store and used automatically for matching queries. They do not appear in `duckdb_settings()` output.

For production use, prefer **IAM roles** (on AWS EC2) or **environment variables** over hardcoded credentials:

```bash
export AWS_ACCESS_KEY_ID=YOUR_KEY
export AWS_SECRET_ACCESS_KEY=YOUR_SECRET
```

> DuckDB's `httpfs` extension automatically picks up standard AWS environment variables.

### Be careful with `read_csv_auto` on untrusted input

`read_csv_auto` will attempt to execute any file path you pass it, including paths on mounted network shares or symbolic link targets. Always validate or sanitise file paths when building dynamic queries from user input.

---

## 📚 Further Reading

- [DuckDB Documentation](https://duckdb.org/docs/)
- [DuckDB SQL Reference](https://duckdb.org/docs/sql/introduction)
- [DuckDB Python API](https://duckdb.org/docs/api/python/overview)
- [DuckDB Extensions](https://duckdb.org/docs/extensions/overview)
- [DuckDB GitHub Repository](https://github.com/duckdb/duckdb)
