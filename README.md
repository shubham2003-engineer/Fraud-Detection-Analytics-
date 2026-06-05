# Fraud Detection Analysis — SQL + Power BI Project

> **End-to-end data analytics project on 6.4 million financial transactions — from raw MySQL data to an interactive Power BI dashboard — detecting fraud patterns, repeat offenders, zero-balance manipulation, and transaction-level risk.**

---

## Project Overview

| Field | Details |
|---|---|
| **Domain** | Financial Fraud Detection |
| **Dataset** | Online Payments Fraud Dataset (`onlinefraud.csv`) |
| **Database** | MySQL |
| **Visualisation** | Power BI Desktop |
| **Records Analysed** | ~6.43 Million Transactions |
| **Tools** | MySQL Workbench · SQL · Power BI Desktop |
| **Skills Demonstrated** | SQL Aggregations · Window Functions · Subqueries · Views · DAX Measures · Power BI Data Modelling · Dashboard Design |

---

## Database Schema

```sql
CREATE DATABASE fraud_detection;
USE fraud_detection;

CREATE TABLE transactions (
    step            INT,              -- 1 step = 1 hour of time
    type            VARCHAR(20),      -- Transaction type (CASH_OUT, TRANSFER, etc.)
    amount          DECIMAL(18,2),    -- Transaction amount
    nameOrig        VARCHAR(50),      -- Originating customer ID
    oldbalanceOrg   DECIMAL(18,2),    -- Sender balance before transaction
    newbalanceOrig  DECIMAL(18,2),    -- Sender balance after transaction
    nameDest        VARCHAR(50),      -- Recipient customer ID
    oldbalanceDest  DECIMAL(18,2),    -- Recipient balance before transaction
    newbalanceDest  DECIMAL(18,2),    -- Recipient balance after transaction
    isFraud         INT,              -- 1 = Fraudulent transaction
    isFlaggedFraud  INT               -- 1 = System-flagged as suspicious
);
```

### Data Import

```sql
LOAD DATA LOCAL INFILE 'C:/Users/shubh/Downloads/onlinefraud.csv'
INTO TABLE transactions
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

---

## Analysis & Queries

### Query 1 — Scale of Fraud: Big Picture Overview

**Business Question:** What proportion of all transactions are fraudulent?

```sql
SELECT
    COUNT(*)                                    AS total_transactions,
    SUM(isFraud)                                AS fraud_transactions,
    ROUND(SUM(isFraud) * 100.0 / COUNT(*), 2)  AS fraud_percentage
FROM transactions;
```

**Findings:**

| Metric | Result |
|---|---|
| Total Transactions | ~6,430,000 |
| Fraud Transactions | 8,318 |
| Fraud Rate | **0.13%** |

> **Insight:** Although fraud is rare (0.13%), even a small percentage across millions of transactions represents significant financial risk. This establishes the baseline for all further analysis.

---

### Query 2 — Fraud by Transaction Type

**Business Question:** Which payment method is most exploited by fraudsters?

```sql
SELECT
    type,
    COUNT(*)      AS total,
    SUM(isFraud)  AS frauds
FROM transactions
GROUP BY type
ORDER BY frauds DESC;
```

**Findings:**
- `CASH_OUT` accounts for the highest number of fraudulent transactions
- `TRANSFER` is the second most exploited type
- Other types (PAYMENT, DEBIT, CASH_IN) show negligible fraud

> **Insight:** Fraud is concentrated in cash-movement transaction types. Risk monitoring should prioritise `CASH_OUT` and `TRANSFER` channels.

---

### Query 3 — High-Value Fraud Transactions

**Business Question:** Which fraudulent transactions pose the greatest financial exposure?

```sql
SELECT *
FROM transactions
WHERE isFraud = 1
  AND amount > 100000
ORDER BY amount DESC;
```

**Findings:**
- Identifies all fraud cases where the transaction exceeded ₹1,00,000 (1 Lakh)
- Results are ranked by amount to surface the most financially damaging cases first

> **Insight:** High-value fraud cases are priority targets for investigation. Flagging and freezing these first minimises total financial loss.

---

### Query 4 — Repeat Fraudsters & Blacklist Candidates

**Business Question:** Are there customers committing fraud more than once?

```sql
SELECT
    nameOrig,
    COUNT(*) AS fraud_count
FROM transactions
WHERE isFraud = 1
GROUP BY nameOrig
ORDER BY fraud_count DESC;
```

**Findings:**
- Identifies all customers linked to more than one fraud transaction
- Enables building an automated blacklist of high-risk accounts

**Fraud Customer Type Breakdown:**

```sql
SELECT
    fraud_type,
    COUNT(*)                                              AS user_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2)  AS percentage
FROM (
    SELECT
        nameOrig,
        CASE
            WHEN COUNT(*) = 1 THEN 'One-time'
            ELSE 'Repeat'
        END AS fraud_type
    FROM transactions
    WHERE isFraud = 1
    GROUP BY nameOrig
) t
GROUP BY fraud_type;
```

> **Insight:** Segmenting fraudsters into one-time vs. repeat offenders helps prioritise investigations and build adaptive fraud prevention rules.

---

### Query 5 — Balance Mismatch Detection

**Business Question:** Are fraudsters manipulating account balances to hide stolen funds?

```sql
SELECT *
FROM transactions
WHERE (oldbalanceOrg - amount) != newbalanceOrig
  AND isFraud = 1;
```

**Findings:**
- Catches transactions where the maths does not add up
- A healthy transaction must satisfy: `oldbalanceOrg - amount = newbalanceOrig`
- Discrepancies reveal deliberate balance tampering

> **Insight:** This query is a precision fraud-signal. A mismatch between expected and actual post-transaction balance is a strong indicator of system manipulation.

---

### Query 6 — High-Risk Transactions View (Reusable)

**Purpose:** Create a persistent, reusable view of all high-risk fraud cases for dashboarding.

```sql
CREATE VIEW high_risk_transactions AS
SELECT *
FROM transactions
WHERE isFraud = 1
  AND amount > 50000;
```

> This view can be plugged directly into Power BI, Tableau, or any BI tool to build live fraud monitoring dashboards without re-running complex queries.

---

### Query 7 — Fraud Summary View for Reporting

**Purpose:** Aggregate fraud statistics by transaction type, ready for BI reporting.

```sql
CREATE VIEW fraud_summary AS
SELECT
    type,
    COUNT(*)      AS total_fraud,
    AVG(amount)   AS avg_fraud_amount,
    MAX(amount)   AS max_fraud_amount
FROM transactions
WHERE isFraud = 1
GROUP BY type;
```

> **Insight:** This summary view gives stakeholders an at-a-glance breakdown of which transaction types are most exploited, average loss per fraud event, and worst-case exposure by category.

---

## Power BI Dashboard

The MySQL database was connected directly to **Power BI Desktop** to build a fully interactive, dark-themed fraud monitoring dashboard. All KPI cards, charts, and tables update dynamically based on the slicer selection at the top.

---

### Dashboard Preview

![Fraud Detection Power BI Dashboard](https://github.com/shubham2003-engineer/Fraud-Detection-Analytics-/blob/main/Screenshot%202026-05-24%20093247.png)



---

###  Interactive Slicer — Transaction Type Filter

A button-style slicer runs across the top of the dashboard. Clicking any button instantly filters **every single visual** on the page — no page changes needed.

| Button | Purpose |
|---|---|
| **Select All** | Resets all visuals to show the complete dataset |
| **CASH_IN** | Drill into cash deposit transactions only |
| **CASH_OUT** | Isolates the #1 highest-fraud channel |
| **DEBIT** | Filters to debit card transactions |
| **PAYMENT** | Filters to payment transactions |
| **TRANSFER** | Isolates the #2 highest-fraud channel |

---

### KPI Cards — 6 Core Metrics

The top row displays 6 KPI cards that give an instant snapshot of the entire fraud landscape:

| # | Card Title | Value | Insight |
|---|---|---|---|
| 1 | **Total Transactions** | **6.43M** | Complete dataset — 6.43 million financial transactions analysed |
| 2 | **Total Fraud** | **8,318** | Total number of confirmed fraudulent transactions detected |
| 3 | **Fraud Rate %** | **0.13%** | Only 0.13% of all transactions are fraud — but the financial impact is massive |
| 4 | **Total Fraud Amount** | **12.09bn** | Total monetary value lost across all fraud transactions |
| 5 | **Zero Balance Cases** | **8,112** | Transactions where the sender's account was completely drained to ₹0 — strongest fraud signal |
| 6 | **Average Fraud Amount** | **1.46M** | Average money stolen per fraudulent transaction — fraudsters target high-value transfers |

>  **Zero Balance Cases (8,112)** is a custom-built metric. It captures cases where `newbalanceOrig = 0` after the transaction, meaning the account was completely emptied. This directly mirrors the SQL balance mismatch query and is one of the most reliable fraud indicators in the dataset.

---

###  Visual 1 — Fraud by Transaction Type (Clustered Bar Chart)

**Location:** Bottom-left of the dashboard

| Detail | Value |
|---|---|
| Chart Type | Horizontal Clustered Bar Chart |
| X-Axis | Transaction count (0K to 5K) |
| Y-Axis | Transaction type |
| Red bars | Fraudulent transactions |
| Green bars | Safe transactions |

**Results:**

| Transaction Type | Fraud Count | Safe Count |
|---|---|---|
| CASH_OUT | ~4,200 | High volume |
| TRANSFER | ~4,100 | High volume |
| CASH_IN | ~0 | High volume |
| DEBIT | ~0 | Moderate |
| PAYMENT | ~0 | High volume |

>  **Finding:** 100% of all fraud is concentrated in just two transaction types — `CASH_OUT` and `TRANSFER`. No fraud occurs in CASH_IN, DEBIT, or PAYMENT. This tells us exactly where to focus fraud prevention resources.

---

###  Visual 2 — Amount by Fraud (Donut Chart)

**Location:** Centre of the dashboard

| Segment | Amount | Percentage |
|---|---|---|
| **Safe transactions** | 1.14 Trillion | 98.95% |
| **Fraud transactions** | 0.01 Trillion | 1.05% |

>  **Finding:** Fraud is only 0.13% of transaction *count* but consumes 1.05% of total transaction *value*. This proves fraudsters deliberately target large, high-value transactions — not random small ones.

---

###  Visual 3 — Fraud by Customer Type (Donut Chart)

**Location:** Top-right of the dashboard

This visual is powered by the SQL window function query that classified fraudsters as One-time or Repeat offenders.

| Customer Type | Count | Percentage |
|---|---|---|
| **One-time fraudsters** | ~8,000 | 98.72% |
| **Repeat fraudsters** | ~100 | 1.28% |

>  **Finding:** 98.72% of fraud accounts commit fraud only once — suggesting mostly opportunistic, account-takeover style fraud. The 1.28% repeat offenders are high-priority blacklist candidates and may indicate organised fraud operations.

---

###  Visual 4 — Fraud Count by Step Interval (Line Chart)

**Location:** Bottom-left of the dashboard

| Detail | Value |
|---|---|
| Chart Type | Line Chart |
| X-Axis | Step (1 step = 1 hour, range: 0 to 744) |
| Y-Axis | Fraud count (range: ~400 to ~650) |
| Line Colour | Red |

**Key observations from the line chart:**
- Fraud starts at a **higher frequency** in the early time steps
- There are **visible spikes** at certain step intervals followed by sharp drops
- Overall fraud frequency shows a **slight declining trend** over the 744-step period
- Several **intermittent surges** appear mid-range, suggesting periodic fraud campaigns

>  **Finding:** Fraud is time-dependent. Certain hours of the day carry significantly higher fraud risk. This insight can drive **real-time alerting rules** — for example, flagging all CASH_OUT transactions during peak-fraud step windows for manual review.

---

###  Visual 5 — Amount by Type (Horizontal Bar Chart)

**Location:** Bottom-centre of the dashboard

Shows the **total money moved** (in Trillions) per transaction type — across both fraud and safe transactions.

| Transaction Type | Total Amount |
|---|---|
| **TRANSFER** | 0.49 Trillion |
| **CASH_OUT** | 0.40 Trillion |
| **CASH_IN** | 0.24 Trillion |
| **PAYMENT** | 0.02838 Trillion |
| **DEBIT** | 0.23 Billion |

>  **Finding:** TRANSFER moves the highest total value (0.49T), making it the most financially dangerous channel. Even though CASH_OUT has more fraud *cases*, a TRANSFER fraud event on average involves more money — requiring stricter controls on high-value transfers.

---

###  Visual 6 — Step × Total Fraud (Table Visual)

**Location:** Bottom-right of the dashboard

A ranked data table showing the exact time steps with the highest fraud volume. Rows highlighted in **red** are peak-fraud periods.

| Step (Hour) | Total Fraud Cases |
|---|---|
| **6** | **44** |
| **212** | **40** |
| **9** | **34** |
| **1** | **32** |
| **523** | **30** |

>  **Finding:** Steps 1, 6, and 9 (the very first few hours of the dataset) show disproportionately high fraud. Step 212 is a notable mid-period spike. This granular view enables **hour-level fraud alerting** — automated systems can apply stricter checks during these specific windows.

---

###  How SQL Views Power the Dashboard

Each dashboard visual is fed by a specific SQL view or table created in this project:

| Dashboard Visual | SQL Source |
|---|---|
| All 6 KPI Cards | `transactions` table with DAX measures |
| Fraud by Transaction Type | `transactions` table grouped by `type` |
| Amount by Fraud (Donut) | `transactions` table — `isFraud` flag |
| Fraud by Customer Type | `transactions` — CASE WHEN window function query |
| Fraud Count by Step | `transactions` grouped by `step` |
| Amount by Type | `transactions` grouped by `type` |
| High-risk table | `high_risk_transactions` view |

```sql
-- Feeds KPI cards and high-risk table
CREATE VIEW high_risk_transactions AS
SELECT * FROM transactions
WHERE isFraud = 1 AND amount > 50000;

-- Feeds fraud summary visuals
CREATE VIEW fraud_summary AS
SELECT type,
       COUNT(*)      AS total_fraud,
       AVG(amount)   AS avg_fraud_amount,
       MAX(amount)   AS max_fraud_amount
FROM transactions
WHERE isFraud = 1
GROUP BY type;
```

**Connection:** Power BI Desktop → Get Data → MySQL Database → loaded `transactions`, `high_risk_transactions`, and `fraud_summary`

---

###  DAX Measures Used in Power BI

```dax
-- Total confirmed fraud transactions
Total Fraud = CALCULATE(COUNT(transactions[isFraud]), transactions[isFraud] = 1)

-- Fraud as % of all transactions
Fraud Rate % = DIVIDE(
    CALCULATE(COUNT(transactions[isFraud]), transactions[isFraud] = 1),
    COUNT(transactions[isFraud])
) * 100

-- Total monetary value of all fraud transactions
Total Fraud Amount = CALCULATE(SUM(transactions[amount]), transactions[isFraud] = 1)

-- Accounts fully drained to zero by fraudsters
Zero Balance Cases = CALCULATE(
    COUNT(transactions[isFraud]),
    transactions[isFraud] = 1,
    transactions[newbalanceOrig] = 0
)

-- Average amount per fraud transaction
Avg Fraud Amount = CALCULATE(AVERAGE(transactions[amount]), transactions[isFraud] = 1)
```

---

##  Key Findings Summary

| # | Query | Key Finding |
|---|---|---|
| 1 | Transaction Overview | 0.13% fraud rate across 6.43M transactions |
| 2 | Fraud by Type | `CASH_OUT` is the highest-risk transaction type |
| 3 | High-Value Fraud | Large frauds (>1 Lakh) identified and ranked |
| 4 | Repeat Fraudsters | Blacklist candidates identified via grouping |
| 5 | Balance Mismatch | Balance manipulation detected mathematically |
| 6 | High-Risk View | Reusable view for BI dashboards (>50K fraud) |
| 7 | Fraud Summary View | Aggregated fraud KPIs by transaction type |

---

##  SQL Skills Demonstrated

- `GROUP BY` with `ORDER BY` for ranked aggregations
- `SUM()`, `COUNT()`, `AVG()`, `MAX()` aggregate functions
- `ROUND()` for formatted percentage calculations
- `CASE WHEN` for conditional classification
- `Window Functions` — `SUM() OVER ()` for percentage distribution
- Subquery-based classification (One-time vs. Repeat fraudsters)
- Arithmetic filtering for anomaly detection (balance mismatch)
- `CREATE VIEW` for reusable, dashboard-ready query objects
- `LOAD DATA LOCAL INFILE` for large dataset ingestion

---

##  Repository Structure

```
fraud-detection-sql/
│
├── README.md                        ← This file
├── schema.sql                       ← Database and table creation
├── queries/
│   ├── 01_fraud_overview.sql
│   ├── 02_fraud_by_type.sql
│   ├── 03_high_value_fraud.sql
│   ├── 04_repeat_fraudsters.sql
│   ├── 05_balance_mismatch.sql
│   ├── 06_view_high_risk.sql
│   └── 07_view_fraud_summary.sql
├── powerbi/
│   └── Fraud_Detection_Dashboard.pbix   ← Power BI report file
└── report/
    ├── dashboard_preview.png            ← Dashboard screenshot
    ├── Fraud_Detection_Report.pdf
    └── Fraud_Detection_Report.docx
```

---

##  How to Run

1. Install **MySQL** and open **MySQL Workbench**
2. Run `schema.sql` to create the database and table
3. Download the dataset and update the file path in the `LOAD DATA` command
4. Execute queries in the `queries/` folder in order
5. Open **Power BI Desktop** → Get Data → MySQL Database
6. Connect to the `fraud_detection` database and load `transactions`, `high_risk_transactions`, and `fraud_summary`
7. Open `powerbi/Fraud_Detection_Dashboard.pbix` to explore the full interactive dashboard

---

## Author

**Shubham**
Data Analytics | SQL | Python | Power BI

---

*This project demonstrates end-to-end data analytics skills — SQL for data extraction and transformation, and Power BI for interactive visualisation — applied to real-world financial fraud detection. Relevant to roles in Data Analytics, Business Intelligence, and Data Engineering.*
