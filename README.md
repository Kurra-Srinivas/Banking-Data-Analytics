# BankScope — Enterprise Banking BI, Analytical SQL & Performance Optimization

BankScope is an enterprise-grade banking data analytics platform, relational PostgreSQL data warehouse, and interactive business intelligence suite built on a 1.26-million-row financial ledger dataset. The project encompasses relational schema architecture, automated streaming ETL migration, a 30-query production analytical SQL catalog, 7-tier RFM customer segmentation, empirical index performance audits via `EXPLAIN (ANALYZE, BUFFERS)`, and a full-stack interactive Streamlit dashboard.

---

## 1. Project Overview & Business Problem

Modern retail and commercial banks face critical data challenges:
* **Siloed Ledger Data**: Disconnected deposit accounts, card instruments, customer profiles, and commercial merchant networks create fragmented customer views and inaccurate risk evaluations.
* **Analytical Query Latency**: High-volume transaction ledgers (millions of rows) degrade to multi-second full-table scans without targeted composite indexing strategies.
* **Reporting Integrity**: Business decisions frequently rely on unvalidated metrics, artificial Cartesian join inflations, and misleading micro-benchmarks.

**BankScope solves these challenges by:**
1. Establishing a normalized **PostgreSQL 18 relational data warehouse** enforcing 100% referential integrity across 1.26 million financial records.
2. Implementing a modular **30-query analytical SQL catalog** covering customer wealth ranking, loan credit risk, merchant volume, and behavioral segmentation.
3. Conducting rigorous, **20-run alternating performance audits** utilizing `EXPLAIN (ANALYZE, BUFFERS)` to measure real I/O page reductions and query latency improvements.
4. Delivering a production-ready **Streamlit executive dashboard** with parameter-driven database queries, dynamic Plotly visual analytics, and dark-theme accessible styling.

---

## 2. Verified Dataset Metrics

Forensic auditing of the canonical banking dataset verified exact record counts, numeric totals, and date boundaries:

| Entity / Metric | Verified Value | Forensic Notes & Business Role |
| :--- | :---: | :--- |
| **Customers** | `50,000` | Retail customer profiles across 50 US states; FICO credit scores range 300–850 |
| **Deposit Accounts** | `75,000` | Checking (25,080), Savings (25,102), and Business (24,818) account holdings |
| **Payment Cards** | `100,000` | Debit (50,051) and Credit (49,949) cards linked to deposit accounts |
| **Loan Commitments** | `30,000` | Personal, auto, and commercial loans across 4 interest rate tiers |
| **Commercial Merchants** | `5,000` | Payment processing merchant directory across retail, dining, and travel |
| **Bank Branches** | `500` | Facility directory *(unlinked standalone entity; zero FK relationships)* |
| **Transaction Ledger** | `1,000,000` | Individual debits and credits timestamped from Jan 1, 2019 to Dec 31, 2025 |
| **Total Warehouse Rows** | **1,260,500** | **100% verified row count with zero orphaned foreign key records** |
| **Total Deposit Holdings** | **$7,494,138,742.77** | Average account balance: $99,921.85 across checking, savings, business |
| **Total Gross Payments** | **$5,001,164,534.00** | Transaction ticket size range: $1.02 to $9,999.98 (Mean: $5,001.16) |
| **Total Loan Principal** | **$4,513,099,925.48** | Total portfolio commitment (Disbursed principal range: $5,000 to $250,000) |

---

## 3. Architecture & How It Works

BankScope strictly decouples presentation from analytical data processing. All calculations, groupings, window aggregations, and joins execute directly within the PostgreSQL engine:

```
+---------------------------------------------------------------------------------------------------+
| ARCHITECTURAL DATA PIPELINE                                                                       |
+-------------------+      +----------------------+      +-------------------+      +---------------+
|     Streamlit     | ---> |  Parameterized SQL   | ---> |    PostgreSQL     | ---> |   Plotly / UI |
|  Interactive Web  |      |   Prepared Queries   |      |   (bankscope_db)  |      | Visual Charts |
|  Filters & Inputs |      |  (dashboard/queries) |      | 1.26M Ledger Rows |      |  & Dashboards |
+-------------------+      +----------------------+      +-------------------+      +---------------+
```

1. **User Interaction (Streamlit)**: Users select date intervals, portfolio dimensions, or query views via sidebar controls.
2. **Parameterized Query Layer (`dashboard/queries.py`)**: User inputs are sanitized and bound to parameterized SQL statements, preventing SQL injection and avoiding duplicate in-memory data munging.
3. **Relational Processing (PostgreSQL 18)**: PostgreSQL evaluates aggregations, window functions, and multi-table joins utilizing composite B-tree indexes directly in the database engine.
4. **Data Delivery & Visualization (Plotly)**: Query results return as compact Pandas DataFrames and render into interactive Plotly time series, segment heatmaps, and financial metrics.

---

## 4. Dashboard Preview

### 1. Executive Banking Overview
High-level portfolio snapshot of retail customers, deposit accounts, lending commitments, and gross transaction volume.
![Executive Overview](docs/screenshots/01_executive_overview.png)

### 2. Customer Analytics & RFM Segmentation
Customer demographic concentration, deposit account product mix, and 7-tier RFM value contribution matrix across 38,849 transacting customers.
![Customer Analytics & RFM](docs/screenshots/02_customer_rfm.png)

### 3. Payment Transaction Analytics
Interactive date-filtered time series, commercial merchant processing volume, and high-value transaction outliers across 1,000,000 ledger records.
![Transaction Analytics](docs/screenshots/03_transaction_analytics.png)

### 4. Lending Portfolio & Credit Risk Analytics
Capital exposure across APR interest rate tiers, annual loan origination trends, and credit score vs. interest rate pricing breakdown across $4.51B in lending assets.
![Loans & Lending](docs/screenshots/04_loans_lending.png)

### 5. SQL Performance & Index Benchmarking
Controlled `EXPLAIN (ANALYZE, BUFFERS)` benchmarks measuring execution latency and physical block reduction across 20 alternating iterations per test state.
![SQL Performance Benchmarks](docs/screenshots/05_sql_performance.png)

---

## 5. SQL Showcase & Query Engineering

The repository includes a production-ready showcase of 7 validated SQL queries in [`sql/sql_showcase.sql`](sql/sql_showcase.sql):

1. **Safe Multi-Table Pre-Aggregation**: Pre-aggregates `accounts` and `loans` in isolated CTEs before joining to `customers`, eliminating Cartesian row multiplication on multi-product banking clients.
2. **Time-Series Analysis with `CTE + LAG()`**: Tracks monthly gross payment volume and calculates period-over-period dollar and percentage growth velocity across 7 years of ledger history.
3. **Window Function Rankings (`DENSE_RANK`)**: Ranks commercial merchants by gross processed dollar volume while avoiding gaps in ranking sequences.
4. **Behavioral RFM Segmentation (`NTILE`)**: Scores 38,849 transacting customers across quintiles (1–5) for Recency, Frequency, and Monetary spend to categorize them into 7 marketing segments.
5. **Cumulative & Pareto Concentration Analysis**: Computes running spend totals via `SUM(...) OVER (ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` to quantify portfolio concentration among top depositors.
6. **Portfolio Risk Tiering (`CASE` Expressions)**: Evaluates loan principal exposure and computes capital-weighted APRs across Prime, Standard, and Subprime borrowing tiers.
7. **Execution Plan & Buffer Investigation**: Profiles plan shifts between unindexed sequential heap scans and composite index lookups via `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)`.

---

## 6. RFM Customer Segmentation Methodology

To assess customer engagement without relying on external credit bureau data, BankScope implements behavioral **Recency, Frequency, and Monetary (RFM)** segmentation on `2025-12-31`:

* **Recency (R)**: Days elapsed between `2025-12-31` and the customer's most recent transaction.
* **Frequency (F)**: Total count of completed transactions across all linked customer accounts.
* **Monetary (M)**: Cumulative dollar volume transacted through the account ledger.

Using windowed quintile scoring (`NTILE(5)`), customers are classified into 7 distinct segments:

| RFM Segment Name | Customer Count | Base Share (%) | Total Spend (USD) | Spend Share (%) | Average Ticket Size |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **1. Champions** | 8,365 | 21.53% | $1,825,765,726.78 | **36.51%** | $5,000.74 |
| **2. Loyal Customers** | 8,137 | 20.95% | $1,235,425,625.28 | **24.70%** | $5,000.67 |
| **3. Recent Promising** | 3,651 | 9.40% | $240,523,662.00 | 4.81% | $5,000.49 |
| **4. At Risk** | 5,791 | 14.91% | $875,450,958.46 | 17.50% | $5,001.43 |
| **5. Need Attention** | 549 | 1.41% | $53,536,652.24 | 1.07% | $5,003.43 |
| **6. Hibernating / Lost** | 8,759 | 22.55% | $516,886,779.00 | 10.34% | $5,001.32 |
| **7. Average / Steady** | 3,597 | 9.26% | $253,575,130.24 | 5.07% | $5,003.46 |
| **Total Transacting Base** | **38,849** | **100.00%** | **$5,001,164,534.00** | **100.00%** | — |

---

## 7. Performance Benchmark Methodology & Audit

Performance benchmarks were executed using PostgreSQL's `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` over **20 alternating runs per state** (Unindexed ➔ Indexed ➔ Unindexed) after warm-up runs to eliminate sequential caching bias:

| Benchmark ID | Analytical Query Workload | Target Index Under Test | Unindexed Median | Indexed Median | Median Latency Gain | Buffer Page Reads | Audit Status |
| :---: | :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| **BENCH_01** | Date-Range Slicing (Q4 2025 Aggregation) | `idx_transactions_date` | 79.8 ms | **33.7 ms** | **57.78% (2.4x speedup)** | 12,452 ➔ 11,796 (-5.3%) | **ROBUST** |
| **BENCH_02** | Account Ledger Statement & Chronological Sort | `idx_transactions_account_date` | 70.1 ms | **0.060 ms** | **99.91% (1,168x speedup)** | 12,376 ➔ 18 (**-99.85%**) | **CAUTION\*** |
| **BENCH_03** | Merchant Monthly Time-Series Aggregation | `idx_transactions_merchant_date` | 71.4 ms | **0.595 ms** | **99.17% (120x speedup)** | 12,599 ➔ 258 (**-97.95%**) | **ROBUST** |

### Critical Engineering Note on BENCH_02 (CAUTION Classification)
* **Physical Reality**: The composite index `idx_transactions_account_date` eliminates full table scans and memory quicksorts, reducing shared buffer page reads by **99.85% (12,376 down to 18 blocks)**.
* **Cache & Timing Caveat**: At 60 microseconds (0.060 ms), execution is 100% resident in PostgreSQL's shared buffer RAM pool, and OS timer jitter accounts for a large fraction of measured variance (StdDev: 0.030 ms).
* **Production Context**: In real-world multi-tier applications, client-to-database network round-trip time (1–5 ms) completely overshadows sub-0.1ms execution. The defensible engineering metric is the **99.85% reduction in I/O buffer reads**, rather than an unhedged latency multiple.

---

## 8. Validation & Testing Results

BankScope includes automated test suites covering data migration, referential integrity, and UI reactivity:

```bash
# 1. Database Parity & Foreign Key Audit
python scripts/validate_postgres.py
# Result: [ALL PASSED] 100% row count parity, PK uniqueness, zero FK orphans across all 7 tables.

# 2. Streamlit Dashboard End-to-End Test Suite
python scratch/test_dashboard_e2e.py
# Result: [ALL PASSED] 5 out of 5 sections rendered with 0 exceptions and verified KPIs.
```

---

## 9. Limitations & Forensic Boundaries

* **Branches Disconnection**: The raw source dataset contains 500 branch records, but provides zero foreign keys linking branches to accounts, customers, or transactions. Furthermore, `branches.city` is 100% NULL. Branches is maintained as an independent catalog.
* **Credit Score vs. APR Independence**: In this synthetic benchmark dataset, loan interest rates do not exhibit risk-based correlation with customer credit scores (average APR remains ~8.5% across all credit score bands).
* **Micro-Benchmark Generalization**: In-engine execution latencies under 0.1 ms reflect buffer-cache residency; network RTT must be factored in for distributed client deployments.

---

## 10. Key Business Insights

1. **Deposit Mix Equivalence**: Deposit balances are evenly split across Checking (33.50%), Savings (33.48%), and Business (33.02%) accounts, showing balanced retail and commercial deposits.
2. **Pareto Spend Concentration**: In the RFM behavioral segmentation, **Champions drive 36.51% of total transaction spend** despite representing only 21.53% of the customer base.
3. **High-Value Outliers**: High-value transactions cluster just beneath the $10,000 threshold ($9,999.98), reflecting standard retail banking currency transaction reporting (CTR) limits.

---

## 11. Project Structure

```
Banking Data Analytics/
├── dashboard/
│   ├── app.py                     # Interactive Streamlit application
│   ├── db.py                      # SQLAlchemy connection pool & query executor
│   ├── queries.py                 # Parameterized SQL query repository
│   └── README.md                  # Dashboard usage & architecture guide
├── docs/
│   ├── database_schema.md         # Complete ER diagram and table DDL specs
│   ├── database_design.md         # Database design decisions and constraints
│   ├── dataset_forensics.md       # Forensic audit of raw SQLite source data
│   ├── migration.md               # Streaming migration protocol documentation
│   ├── sql_analytics_catalog.md   # Catalog of all 30 analytical business queries
│   ├── sql_performance_benchmark.md # Baseline SQL performance benchmark report
│   ├── sql_performance_benchmark_audit.md # Rigorous 20-run benchmark audit report
│   ├── sql_quality_audit.md       # Quality audit of SQL queries & edge cases
│   └── screenshots/               # GitHub-ready dashboard previews
│       ├── 01_executive_overview.png
│       ├── 02_customer_rfm.png
│       ├── 03_transaction_analytics.png
│       ├── 04_loans_lending.png
│       └── 05_sql_performance.png
├── scripts/
│   ├── benchmark_sql_performance.py # SQL benchmark runner engine
│   ├── migrate_sqlite_to_postgres.py# High-speed streaming ETL engine
│   └── validate_postgres.py       # Post-migration database parity validation suite
├── sql/
│   ├── schema.sql                 # DDL: Base table definitions
│   ├── constraints.sql            # DDL: Foreign keys and domain CHECK constraints
│   ├── indexes.sql                # DDL: B-tree performance index architecture
│   ├── customer_analytics.sql     # Queries: Customer profiling & wealth tiering
│   ├── account_analytics.sql      # Queries: Deposit product mix & liquidity
│   ├── transaction_analytics.sql  # Queries: Payment velocity & volume trends
│   ├── loan_analytics.sql         # Queries: Loan portfolio exposure & APR pricing
│   ├── merchant_analytics.sql     # Queries: Commercial merchant settlement
│   ├── branch_analytics.sql       # Queries: Branch facility geographic distribution
│   ├── rfm_analysis.sql           # Queries: 7-tier RFM behavioral segmentation
│   ├── advanced_analytics.sql     # Queries: Window analytics, Pareto & MoM growth
│   └── sql_showcase.sql           # Curated showcase of top 7 validated queries
├── .env.example                   # Safe template for environment variables
├── .gitignore                     # Git exclusion rules for credentials, caches, & data
├── README.md                      # Comprehensive project documentation
└── requirements.txt               # Pinned Python package dependencies
```

---

## 12. Setup & Installation

### Prerequisites
* Python 3.11+
* PostgreSQL 15+ installed and active on `localhost:5432`

### Step 1: Environment Setup
```bash
git clone <repo-url>
cd "Banking Data Analytics"
python -m venv .venv
.\.venv\Scripts\activate      # Windows PowerShell
pip install -r requirements.txt
```

### Step 2: Configure Database Credentials
Create a `.env` file in the project root based on `.env.example`:
```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=bankscope_db
DB_USER=postgres
DB_PASSWORD=your_password_here
```

### Step 3: Run Streaming ETL Migration & Validation
```bash
python scripts/migrate_sqlite_to_postgres.py
python scripts/validate_postgres.py
```

### Step 4: Launch the Dashboard
```bash
streamlit run dashboard/app.py
```
Open `http://localhost:8501` in your browser.
