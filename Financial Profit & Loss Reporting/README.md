# **Financial Profit & Loss (P&L) Reporting – Full Breakdown (with SQL)**


## 🔹 Business Use Case: Track Company Profitability

**Goal:**
Track **how much profit (or loss)** the company makes over time so finance can:

* Compare **monthly revenue vs expenses**
* See which months were **most profitable**
* Plan **budget, hiring, investments, and cost-cutting**
* Report to **management, investors, and auditors**

**Real-World Example (Finance Team):**
Each month, the finance team wants a report like:

| Month   | Total Revenue | Total Expenses | Profit (Revenue − Expenses) |
| ------- | ------------- | -------------- | --------------------------- |
| 2025-01 | ₹10,00,000    | ₹7,50,000      | ₹2,50,000                   |
| 2025-02 | ₹8,50,000     | ₹9,20,000      | -₹70,000 (loss)             |

Use this to answer:

* “Which months were **loss-making**?”
* “Is profit **growing or shrinking** over time?”
* “Are expenses becoming **too high vs revenue**?”

---

## 🧱 Sample Database Schema (Simple Finance Model)

There are 2 common designs for finance data. I’ll show both:


### 🅰️ Design 1 – Separate Revenue and Expense Tables

#### `revenue`

| column     | type    | description                            |
| ---------- | ------- | -------------------------------------- |
| revenue_id | INT     | Unique revenue record                  |
| txn_date   | DATE    | Date of sale/earning                   |
| amount     | DECIMAL | Revenue amount (positive)              |
| source     | VARCHAR | e.g. product_sales, services, interest |

#### `expenses`

| column     | type    | description                         |
| ---------- | ------- | ----------------------------------- |
| expense_id | INT     | Unique expense record               |
| txn_date   | DATE    | When expense occurred               |
| amount     | DECIMAL | Expense amount (positive)           |
| category   | VARCHAR | e.g. salary, rent, marketing, infra |


### 🅱️ Design 2 – Single Ledger Table (More “Accounting-Style”)

#### `transactions`

| column   | type    | description                                         |
| -------- | ------- | --------------------------------------------------- |
| txn_id   | INT     | Unique transaction                                  |
| txn_date | DATE    | Date                                                |
| amount   | DECIMAL | Amount (positive for revenue, negative for expense) |
| type     | VARCHAR | 'revenue' / 'expense'                               |
| account  | VARCHAR | account name/category                               |

We’ll write queries for **both** patterns.

---

## 🎯 Core SQL Task

> **Calculate monthly profit = total revenue − total expenses.**


## ✅ Approach 1 – Using Separate `revenue` and `expenses` Tables

### Step 1: Monthly Revenue

```sql
SELECT
    DATE_TRUNC('month', txn_date) AS month,
    SUM(amount) AS total_revenue
FROM
    revenue
GROUP BY
    DATE_TRUNC('month', txn_date)
ORDER BY
    month;
```

### Step 2: Monthly Expenses

```sql
SELECT
    DATE_TRUNC('month', txn_date) AS month,
    SUM(amount) AS total_expenses
FROM
    expenses
GROUP BY
    DATE_TRUNC('month', txn_date)
ORDER BY
    month;
```

### Step 3: Combine Revenue & Expenses → Monthly Profit

Use the two aggregates above in a join:

```sql
WITH rev AS (
    SELECT
        DATE_TRUNC('month', txn_date) AS month,
        SUM(amount) AS total_revenue
    FROM
        revenue
    GROUP BY
        DATE_TRUNC('month', txn_date)
),
exp AS (
    SELECT
        DATE_TRUNC('month', txn_date) AS month,
        SUM(amount) AS total_expenses
    FROM
        expenses
    GROUP BY
        DATE_TRUNC('month', txn_date)
)

SELECT
    COALESCE(r.month, e.month) AS month,
    COALESCE(r.total_revenue, 0) AS total_revenue,
    COALESCE(e.total_expenses, 0) AS total_expenses,
    COALESCE(r.total_revenue, 0) - COALESCE(e.total_expenses, 0) AS profit
FROM
    rev r
FULL OUTER JOIN
    exp e
    ON r.month = e.month
ORDER BY
    month;
```

**Why `COALESCE` + `FULL OUTER JOIN`?**

* Some months may have **revenue but no expense**, or vice versa.
* We still want those months to appear with 0 for the missing side.

> On MySQL (no `FULL OUTER JOIN`), you can emulate with `UNION` or join via a calendar table, or use an inner join if your data always has both.



## ✅ Approach 2 – Single Ledger (`transactions` Table)

If `transactions.amount` is **positive for revenue and negative for expenses**, then:

```sql
SELECT
    DATE_TRUNC('month', txn_date) AS month,
    SUM(CASE WHEN type = 'revenue' THEN amount ELSE 0 END) AS total_revenue,
    SUM(CASE WHEN type = 'expense' THEN amount ELSE 0 END) AS total_expenses,
    SUM(
        CASE 
            WHEN type = 'revenue' THEN amount
            WHEN type = 'expense' THEN -amount  -- if expenses stored as positive
            ELSE 0
        END
    ) AS profit
FROM
    transactions
GROUP BY
    DATE_TRUNC('month', txn_date)
ORDER BY
    month;
```

Two possibilities based on how you store expenses:

1. **Expenses stored as positive numbers** → use `-amount` while computing profit.
2. **Expenses stored as negative numbers** → profit is just `SUM(amount)` for the month.

Example if revenue = +ve, expenses = **-ve**:

```sql
SELECT
    DATE_TRUNC('month', txn_date) AS month,
    SUM(CASE WHEN type = 'revenue' THEN amount ELSE 0 END) AS total_revenue,
    SUM(CASE WHEN type = 'expense' THEN -amount ELSE 0 END) AS total_expenses,
    SUM(amount) AS profit  -- revenue - expenses because expenses are negative
FROM
    transactions
GROUP BY
    DATE_TRUNC('month', txn_date)
ORDER BY
    month;
```

---

## 📊 Extra Metrics: Profit Margin, YTD Profit, etc.

### 1️⃣ Profit Margin (%)

Add `profit_margin = profit / total_revenue * 100`.

```sql
WITH monthly_pl AS (
    SELECT
        DATE_TRUNC('month', txn_date) AS month,
        SUM(CASE WHEN type = 'revenue' THEN amount ELSE 0 END) AS total_revenue,
        SUM(CASE WHEN type = 'expense' THEN -amount ELSE 0 END) AS total_expenses
    FROM
        transactions
    GROUP BY
        DATE_TRUNC('month', txn_date)
)
SELECT
    month,
    total_revenue,
    total_expenses,
    total_revenue - total_expenses AS profit,
    CASE
        WHEN total_revenue > 0
        THEN ROUND((total_revenue - total_expenses) / total_revenue * 100, 2)
        ELSE NULL
    END AS profit_margin_percent
FROM
    monthly_pl
ORDER BY
    month;
```


### 2️⃣ Year-to-Date (YTD) Cumulative Profit

Helps finance see how the **year is progressing**.

```sql
WITH monthly_pl AS (
    SELECT
        DATE_TRUNC('month', txn_date) AS month,
        SUM(CASE WHEN type = 'revenue' THEN amount ELSE 0 END) AS total_revenue,
        SUM(CASE WHEN type = 'expense' THEN -amount ELSE 0 END) AS total_expenses
    FROM
        transactions
    GROUP BY
        DATE_TRUNC('month', txn_date)
)
SELECT
    month,
    total_revenue,
    total_expenses,
    (total_revenue - total_expenses) AS profit,
    SUM(total_revenue - total_expenses)
        OVER (ORDER BY month) AS cumulative_profit_ytd
FROM
    monthly_pl
ORDER BY
    month;
```


### 3️⃣ Department / Cost Center–Wise Profit & Loss

If you track **department** or **cost_center** per transaction:

`transactions` add: `cost_center` / `department_id`.

```sql
SELECT
    DATE_TRUNC('month', txn_date) AS month,
    cost_center,
    SUM(CASE WHEN type = 'revenue' THEN amount ELSE 0 END) AS total_revenue,
    SUM(CASE WHEN type = 'expense' THEN -amount ELSE 0 END) AS total_expenses,
    SUM(amount) AS profit  -- assuming expense is negative
FROM
    transactions
GROUP BY
    DATE_TRUNC('month', txn_date),
    cost_center
ORDER BY
    month,
    cost_center;
```

Now you can answer:

> “Which departments are most profitable or burning money?”

---

## 🎓 Interview-Level Questions Based on This Use Case

You can convert this into excellent SQL interview problems:

### Q1: “Top 3 Most Profitable Months”

Using a P&L query, find the **top 3 months with highest profit.**

> (Hint: compute monthly profit then `ORDER BY profit DESC LIMIT 3`.)


### Q2: “Months with Negative Profit (Loss Months)”

List all months where profit was **negative**, along with profit margin.

> (Hint: filter `WHERE profit < 0`.)


### Q3: “Compare This Year vs Last Year”

For each month number (1–12), compare **this year’s profit vs last year’s profit**.

> (Hint: extract `YEAR` and `MONTH`, group, then self-join or pivot.)


### Q4: “Expense Breakdown by Category per Month”

For each month, show total expenses split by category (salary, rent, marketing, etc.).

> (Hint: group by `DATE_TRUNC('month', txn_date), category`.)


### Q5: “Revenue Concentration”

Find what **% of total yearly revenue** comes from the **top 2 months**.

> (Hint: monthly revenue, order, sum top 2 / total.)

---

## 🧪 Hands-On Practice Tasks (Perfect for Your Audience)

Here are practice tasks you can put as **Instagram carousel / YouTube short / blog exercises**:

### 🔹 Task 1 – Basic Monthly P&L

> Query to show per month: `total_revenue`, `total_expenses`, `profit`.


### 🔹 Task 2 – Highlight Loss Months

> Modify Task 1 to **only show months where profit < 0**.



### 🔹 Task 3 – Profit Trend Over Last 6 Months

> Show profit for the **last 6 months**, ordered chronologically, and compute **cumulative profit**.


### 🔹 Task 4 – Department P&L

> Per department and month: show revenue, expenses, and profit. Sort by lowest profit first.


### 🔹 Task 5 – Profit Margin Banding

> For each month, categorize profit margin into:

* `High` → margin ≥ 30%
* `Medium` → 15%–30%
* `Low` → 0–15%
* `Negative` → < 0

Use a `CASE` expression.

---
## Datasets
### 1. Accounting Data for Financial Management: 
> A dataset modeling real-world financial transactions: revenue, expenses, cash flow, profitability etc. — ideal for building P&L statements, monthly profit queries.
<a href="https://www.kaggle.com/datasets/ziya07/accounting-data-for-financial-management" target="_blank">
Download
</a>

### 2. Profit & Loss Example Project:
> Sample P&L data for several fiscal years — good to practise monthly/quarterly profit calculations and trend analysis.
<a href="https://www.kaggle.com/datasets/aleckszhukovsky/profit-loss-example-project" target="_blank">
Download
</a>

### 3. Financial Sheets Dataset:
> Dataset containing financial statement-style data for companies — useful for analyzing company performance, revenue/expense breakdown, profitability.
<a href="https://www.kaggle.com/datasets/pacificrm/financial-sheets" target="_blank">
Download
</a>

### 4. Free Finance & Accounting Sample Data:
> A collection of synthetic but realistic datasets (general ledger, AR/AP, expense claims etc.) — useful to simulate real-company accounting, generate P&L, cash flow etc.
<a href="https://excelx.com/practice-data/finance-accounting/" target="_blank">
Download
</a>

### 5. Companies ranked by earnings - CompaniesMarketCap.com.csv:
> Contains earnings/financial metrics of ~6,000+ companies globally — useful for high-level profitability analysis or as mock data for revenue/expense modelling.
<a href="https://www.opendatabay.com/data/financial/0ac70e7d-7f08-4112-a300-8add4434ade8" target="_blank">
Download
</a>
