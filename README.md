# Fraud Detection Analysis — SQL + Power BI Project

End-to-end data analytics project on 6.4 million financial transactions — from raw MySQL data to an interactive Power BI dashboard — detecting fraud patterns, repeat offenders, zero-balance manipulation, and transaction-level risk.

---

## Project Overview

| Field | Details |
|---|---|
| Domain | Financial Fraud Detection |
| Dataset | Online Payments Fraud Dataset (onlinefraud.csv / PaySim) |
| Database | MySQL |
| Visualisation | Power BI Desktop |
| Records Analysed | ~6.43 Million Transactions |
| Tools | MySQL Workbench · SQL · Power BI Desktop |
| Skills Demonstrated | SQL Aggregations · Window Functions · Subqueries · Views · DAX Measures · Power BI Data Modelling · Dashboard Design |

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
    isFlaggedFraud  INT               -- 1 = System-flagged as suspicious (rule-based)
);
```

## Data Import

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
| Fraud Rate | 0.13% |

**Insight:** Although fraud is rare (0.13%), even a small percentage across millions of transactions represents significant financial risk. This establishes the baseline for all further analysis.

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
- CASH_OUT accounts for the highest number of fraudulent transactions
- TRANSFER is the second most exploited type
- Other types (PAYMENT, DEBIT, CASH_IN) show negligible fraud

**Insight:** Fraud is concentrated in cash-movement transaction types. Risk monitoring should prioritise CASH_OUT and TRANSFER channels.

### Query 3 — High-Value Fraud Transactions
**Business Question:** Which fraudulent transactions pose the greatest financial exposure?

```sql
SELECT *
FROM transactions
WHERE isFraud = 1
  AND amount > 100000
ORDER BY amount DESC;
```

**Insight:** High-value fraud cases are priority targets for investigation. Flagging and freezing these first minimises total financial loss.

### Query 4 — Repeat Fraudsters & Blacklist Candidates
**Business Question:** Are there customers committing fraud more than once?

```sql
SELECT
    nameDest,
    COUNT(*) AS fraud_count
FROM transactions
WHERE isFraud = 1
GROUP BY nameDest
ORDER BY fraud_count DESC;
```

**Fraud Customer Type Breakdown:**

```sql
SELECT
    fraud_type,
    COUNT(*)                                              AS user_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2)  AS percentage
FROM (
    SELECT
        nameDest,
        CASE
            WHEN COUNT(*) = 1 THEN 'One-time'
            ELSE 'Repeat'
        END AS fraud_type
    FROM transactions
    WHERE isFraud = 1
    GROUP BY nameDest
) t
GROUP BY fraud_type;
```

**Insight:** Segmenting fraudsters into one-time vs. repeat offenders helps prioritise investigations and build adaptive fraud prevention rules.

### Query 5 — Balance Mismatch Detection
**Business Question:** Are fraudsters manipulating account balances to hide stolen funds?

```sql
SELECT *
FROM transactions
WHERE (oldbalanceOrg - amount) != newbalanceOrig
  AND isFraud = 1;
```

**Insight:** A healthy transaction must satisfy `oldbalanceOrg - amount = newbalanceOrig`. Discrepancies reveal deliberate balance tampering — a precision fraud signal.

### Query 6 — High-Risk Transactions View (Reusable)

```sql
CREATE VIEW high_risk_transactions AS
SELECT *
FROM transactions
WHERE isFraud = 1
  AND amount > 50000;
```

### Query 7 — Fraud Summary View for Reporting

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

**Insight:** This summary view gives stakeholders an at-a-glance breakdown of which transaction types are most exploited, average loss per fraud event, and worst-case exposure by category.

---

## Power BI Dashboard

The MySQL database was connected directly to Power BI Desktop to build a fully interactive, dark-themed fraud monitoring dashboard. All KPI cards, charts, and tables update dynamically based on the slicer selection at the top.

### Interactive Slicer — Transaction Type Filter

| Button | Purpose |
|---|---|
| Select All | Resets all visuals to show the complete dataset |
| CASH_IN | Drill into cash deposit transactions only |
| CASH_OUT | Isolates the #1 highest-fraud channel |
| DEBIT | Filters to debit card transactions |
| PAYMENT | Filters to payment transactions |
| TRANSFER | Isolates the #2 highest-fraud channel |

### KPI Cards — 6 Core Metrics

| # | Card Title | Value | Insight |
|---|---|---|---|
| 1 | Total Transactions | 6.43M | Complete dataset — 6.43 million financial transactions analysed |
| 2 | Total Fraud | 8,318 | Total number of confirmed fraudulent transactions detected |
| 3 | Fraud Rate % | 0.13% | Only 0.13% of all transactions are fraud — but the financial impact is massive |
| 4 | Total Fraud Amount | 0.0121T | Total monetary value lost across all fraud transactions — now shown in the same unit (Trillions) as the Amount-by-Fraud chart, so the two figures reconcile |
| 5 | Accounts Drained to Zero | 8,112 | Renamed from "Zero Balance Cases" for clarity — transactions where the sender's account was completely drained to ₹0 (`newbalanceOrig = 0`). One of the strongest fraud signals in the dataset |
| 6 | Fraud_Detected | 16 | **New KPI.** Count of transactions the system's own rule (`isFlaggedFraud`) proactively flagged as suspicious — separate from `isFraud` (the confirmed ground-truth label) |

> **Why Fraud_Detected matters:** `isFlaggedFraud` and `isFraud` are two different signals — one is what the rule *caught in advance*, the other is what was *confirmed after the fact*. Comparing the two (Fraud_Detected: 16 vs. Total Fraud: 8,318) already tells a story on its own: the existing rule catches a tiny fraction of true fraud. See [Suggested Next Enhancements](#suggested-next-enhancements) for how to turn this into a full precision/recall metric.
>
> <img width="1154" height="643" alt="Screenshot 2026-09-20 000120" src="https://github.com/user-attachments/assets/6ca0ae4d-0b12-44db-882e-68fa5319f495" />


### Visual 1 — Fraud by Transaction Type (Clustered Bar Chart)
*Location: Bottom-left of the dashboard*

| Transaction Type | Fraud Count |
|---|---|
| CASH_OUT | ~4,200 |
| TRANSFER | ~4,100 |
| CASH_IN | ~0 |
| DEBIT | ~0 |
| PAYMENT | ~0 |

**Finding:** 100% of all fraud is concentrated in just two transaction types — CASH_OUT and TRANSFER. This tells us exactly where to focus fraud prevention resources.

### Visual 2 — Amount by Fraud (Donut Chart)
*Location: Centre of the dashboard*

| Segment | Amount | Percentage |
|---|---|---|
| Safe transactions | 1.14 Trillion | 98.95% |
| Fraud transactions | 0.01 Trillion | 1.05% |

**Finding:** Fraud is only 0.13% of transaction count but consumes 1.05% of total transaction value — fraudsters deliberately target large, high-value transactions.

### Visual 3 — Fraud by Customer Type (Line Chart)
*Location: Top-right of the dashboard*

> **Changed:** this was previously a donut chart showing percentages (One-time 98.72% / Repeat 1.28%). It is now a **line chart plotting raw counts**, which makes the size gap between the two groups far more visually dramatic.

| Customer Type | Count |
|---|---|
| One-time fraudsters | ~8.1K |
| Repeat fraudsters | ~0.1K |

**Finding:** The overwhelming majority of fraud accounts commit fraud only once, suggesting mostly opportunistic, account-takeover style fraud. The small repeat-offender group is a high-priority blacklist candidate.

### Visual 4 — Fraud Count by Day (Line Chart)
*Location: Bottom-left of the dashboard*

> **Changed:** this was previously "Fraud Count by Step Interval," plotting raw hourly `step` values (0–744) on the x-axis. It has been rebuilt to bucket every step into a **calendar day (1–31)**, giving a much more readable trend line for a business audience than raw hour numbers.

**DAX used to build the Day column:**
```dax
Day = QUOTIENT('transactions'[step] - 1, 24) + 1
```

**Finding:** Fraud count shows a declining trend across the days in scope, with the sharpest volume in the earliest days of the dataset window.

### Visual 5 — Amount by Type (Horizontal Bar Chart)
*Location: Bottom-centre of the dashboard*

| Transaction Type | Total Amount |
|---|---|
| TRANSFER | 0.49 Trillion |
| CASH_OUT | 0.40 Trillion |
| CASH_IN | 0.24 Trillion |
| PAYMENT | 0.028 Trillion |
| DEBIT | 0.23 Billion |

**Finding:** TRANSFER moves the highest total value, making it the most financially dangerous channel even though CASH_OUT has slightly more fraud cases.

### Visual 6 — Hour No × Total Fraud (Table Visual)
*Location: Bottom-right of the dashboard*

> **Changed:** the first column was previously labeled "Step." It is now labeled **"Hour No"** — same underlying `step` field, but a clearer, business-friendly name since each step is one hour.

| Hour No | Total Fraud Cases |
|---|---|
| 523 | 30 |
| 1 | 32 |
| 9 | 34 |
| 212 | 40 |
| 6 | 44 |

**Finding:** The very first few hours of the dataset show disproportionately high fraud, with a notable mid-period spike at hour 212. This granular view enables hour-level fraud alerting.

---

## How SQL Views Power the Dashboard

| Dashboard Visual | SQL Source |
|---|---|
| All 6 KPI Cards | `transactions` table with DAX measures |
| Fraud by Transaction Type | `transactions` grouped by `type` |
| Amount by Fraud (Donut) | `transactions` — `isFraud` flag |
| Fraud by Customer Type | `transactions` — CASE WHEN window function query |
| Fraud Count by Day | `transactions` grouped by derived `Day` column |
| Amount by Type | `transactions` grouped by `type` |
| Hour No × Total Fraud table | `high_risk_transactions` view |

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

Connection: **Power BI Desktop → Get Data → MySQL Database** → loaded `transactions`, `high_risk_transactions`, and `fraud_summary`.

---

## DAX Measures Used in Power BI

```dax
-- Total confirmed fraud transactions
Total Fraud = CALCULATE(COUNT(transactions[isFraud]), transactions[isFraud] = 1)

-- Fraud as % of all transactions
Fraud Rate % = DIVIDE(
    CALCULATE(COUNT(transactions[isFraud]), transactions[isFraud] = 1),
    COUNT(transactions[isFraud])
) * 100

-- Total monetary value of all fraud transactions (unit standardised to Trillions to match the donut chart)
Total Fraud Amount = CALCULATE(SUM(transactions[amount]), transactions[isFraud] = 1)

-- Accounts fully drained to zero by fraudsters (displayed on card as "Accounts Drained to Zero")
Zero Balance Cases = CALCULATE(
    COUNT(transactions[isFraud]),
    transactions[isFraud] = 1,
    transactions[newbalanceOrig] = 0
)

-- NEW: count of transactions the system's own rule flagged in advance
Fraud_Detected = CALCULATE(
    COUNT(transactions[isFlaggedFraud]),
    transactions[isFlaggedFraud] = 1
)

-- NEW: calculated column bucketing hourly step into calendar day
Day = QUOTIENT('transactions'[step] - 1, 24) + 1
```

---

## What Changed in This Version

| Area | Before | Now |
|---|---|---|
| KPI #5 | "Zero Balance Cases" | Renamed to **"Accounts Drained to Zero"** — same value (8,112), clearer business meaning |
| KPI #6 | "Average Fraud Amount" (1.46M) | Replaced with **"Fraud_Detected"** (16) — a rule-based flag count, distinct from confirmed fraud |
| Total Fraud Amount | Shown as `12.09bn` on the KPI card vs. `0.01T` on the donut chart (unit mismatch) | Standardised to **0.0121T** in both places — the earlier inconsistency is resolved |
| Fraud by Customer Type | Donut chart, percentages (98.72% / 1.28%) | **Line chart**, raw counts (8.1K / 0.1K) |
| Bottom-left trend chart | "Fraud Count by Step Interval," x-axis = raw hour `step` (0–744) | Renamed **"Fraud Count by Day"**, x-axis = derived `Day` column (1–31) |
| Table column | "Step" | Renamed **"Hour No"** for clarity |

---

## Suggested Next Enhancements

1. **Turn `Fraud_Detected` into a full rule-performance panel.** You already have both `isFlaggedFraud` (the rule's guess) and `isFraud` (ground truth) — build a confusion matrix (True Positive / False Positive / False Negative / True Negative) and derive Precision and Recall. This will likely show high precision but very low recall (16 flagged vs. 8,318 actual), which is a genuinely useful finding to surface on the dashboard rather than leave implicit.
   ```dax
   True Positive = CALCULATE(COUNTROWS(transactions), transactions[isFlaggedFraud]=1, transactions[isFraud]=1)
   False Positive = CALCULATE(COUNTROWS(transactions), transactions[isFlaggedFraud]=1, transactions[isFraud]=0)
   False Negative = CALCULATE(COUNTROWS(transactions), transactions[isFlaggedFraud]=0, transactions[isFraud]=1)
   Precision % = DIVIDE([True Positive], [True Positive] + [False Positive]) * 100
   Recall % = DIVIDE([True Positive], [True Positive] + [False Negative]) * 100
   ```
2. **Add a one-line insight banner** near the top of the dashboard summarizing the Fraud_Detected vs. Total Fraud gap for anyone who doesn't dig into the numbers themselves.
3. **Rename the underlying DAX measure**, not just the card title, from `Zero Balance Cases` to `Accounts Drained to Zero` so the model and the visual stay in sync for future maintainers.
4. **Document the Day-bucketing logic** (`QUOTIENT` formula above) in-line as a comment in the `.pbix` measure pane, since it's easy for a future editor to reintroduce the raw-`step` version by mistake.
5. **Add a legend/tooltip** clarifying that `Fraud_Detected` is a *rule flag count*, not a count of confirmed fraud — the two KPI cards sitting side by side (8,318 vs. 16) could otherwise be misread as contradictory.

---

## Key Findings Summary

| # | Query | Key Finding |
|---|---|---|
| 1 | Transaction Overview | 0.13% fraud rate across 6.43M transactions |
| 2 | Fraud by Type | CASH_OUT and TRANSFER are the only channels carrying fraud |
| 3 | High-Value Fraud | Large frauds (>1 Lakh) identified and ranked |
| 4 | Repeat Fraudsters | Blacklist candidates identified via grouping |
| 5 | Balance Mismatch | Balance manipulation detected mathematically |
| 6 | High-Risk View | Reusable view for BI dashboards (>50K fraud) |
| 7 | Fraud Summary View | Aggregated fraud KPIs by transaction type |
| 8 | Rule vs. Ground Truth | Existing `isFlaggedFraud` rule catches only 16 of 8,318 confirmed fraud cases — signals a recall gap worth addressing |

## SQL Skills Demonstrated
- `GROUP BY` with `ORDER BY` for ranked aggregations
- `SUM()`, `COUNT()`, `AVG()`, `MAX()` aggregate functions
- `ROUND()` for formatted percentage calculations
- `CASE WHEN` for conditional classification
- Window Functions — `SUM() OVER ()` for percentage distribution
- Subquery-based classification (One-time vs. Repeat fraudsters)
- Arithmetic filtering for anomaly detection (balance mismatch)
- `CREATE VIEW` for reusable, dashboard-ready query objects
- `LOAD DATA LOCAL INFILE` for large dataset ingestion

## Repository Structure

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

## How to Run
1. Install MySQL and open MySQL Workbench
2. Run `schema.sql` to create the database and table
3. Download the dataset and update the file path in the `LOAD DATA` command
4. Execute queries in the `queries/` folder in order
5. Open Power BI Desktop → Get Data → MySQL Database
6. Connect to the `fraud_detection` database and load `transactions`, `high_risk_transactions`, and `fraud_summary`
7. Open `powerbi/Fraud_Detection_Dashboard.pbix` to explore the full interactive dashboard

## Author
**Shubham** — Data Analytics | SQL | Python | Power BI

This project demonstrates end-to-end data analytics skills — SQL for data extraction and transformation, and Power BI for interactive visualisation — applied to real-world financial fraud detection. Relevant to roles in Data Analytics, Business Intelligence, and Data Engineering.
