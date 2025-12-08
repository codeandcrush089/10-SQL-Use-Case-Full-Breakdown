# **Churn Analysis – Full Breakdown (Customer Inactive / No Transactions)**

## 🔹 Business Use Case: What Is Churn Here?

**Goal:** Detect customers who **stopped using a service** so the business can:

* Run **win-back campaigns** (discounts, calls, reminders)
* Improve **user experience** (why did they leave?)
* Forecast **revenue loss**

**Real-World Example (Telecom):**
A telecom company wants to find all users who **haven’t recharged in the last 90 days**.
In most telecoms, if a user doesn’t recharge for 90+ days, they’re treated as:

* **Churned** (lost customer), or
* **At high risk** of churn

We’ll generalize this as:

> “Find customers with **no transactions in the last 3 months**.”

(You can replace 3 months with 30 days, 60 days, 180 days, etc., depending on business rules.)

---

## 🧱 Sample Database Schema (Telecom / Subscription Style)

Assume you have:

### `customers`

| column      | type    | description                            |
| ----------- | ------- | -------------------------------------- |
| customer_id | INT     | Unique customer ID                     |
| name        | VARCHAR | Customer name                          |
| join_date   | DATE    | When they started using the service    |
| status      | VARCHAR | active / inactive / blocked (optional) |

### `transactions` (or `recharges`, `payments`, etc.)

| column      | type    | description                           |
| ----------- | ------- | ------------------------------------- |
| txn_id      | INT     | Unique transaction ID                 |
| customer_id | INT     | Who made the transaction              |
| txn_date    | DATE    | When recharge/payment was made        |
| amount      | DECIMAL | Amount paid / recharged               |
| txn_type    | VARCHAR | recharge / bill_payment / etc. (opt.) |

We define **activity** as having at least one transaction in the last 3 months.

So **churn candidate** = customer **with no transaction in last 3 months**.

---

## 🎯 Core SQL Task

> **Find customers with no transactions in the last 3 months.**

### ✅ Approach 1 – LEFT JOIN + NULL Filter (Most Common)

**Idea:**

* Left join customers to transactions that happened in the last 3 months.
* If no such transaction exists, that customer is “inactive in last 3 months”.

**PostgreSQL / MySQL style:**

```sql
SELECT
    c.customer_id,
    c.name
FROM
    customers c
LEFT JOIN
    transactions t
    ON c.customer_id = t.customer_id
    AND t.txn_date >= CURRENT_DATE - INTERVAL '3 months'
WHERE
    t.customer_id IS NULL;
```

**Explanation:**

* `t.txn_date >= CURRENT_DATE - INTERVAL '3 months'` → Only join **recent** transactions.
* If `t.customer_id IS NULL` after the join → this customer has **no transaction** in that window.

> ⚠️ This includes customers who **never transacted even once**.
> If you want to exclude “never activated” and only look at once-active customers, we can modify it (see below).



### ✅ Approach 2 – Using MAX(txn_date) (Very Clear in Interviews)

**Idea:**

1. Compute each customer’s **last transaction date**.
2. Then mark those whose last transaction is older than 3 months (or NULL).

```sql
WITH last_txn AS (
    SELECT
        customer_id,
        MAX(txn_date) AS last_txn_date
    FROM
        transactions
    GROUP BY
        customer_id
)
SELECT
    c.customer_id,
    c.name,
    lt.last_txn_date
FROM
    customers c
LEFT JOIN
    last_txn lt
    ON c.customer_id = lt.customer_id
WHERE
    lt.last_txn_date IS NULL
    OR lt.last_txn_date < CURRENT_DATE - INTERVAL '3 months'
ORDER BY
    lt.last_txn_date NULLS FIRST;
```

**Interpretation:**

* `lt.last_txn_date IS NULL` → customer has **never transacted**.
* `lt.last_txn_date < CURRENT_DATE - INTERVAL '3 months'` → last activity was **more than 3 months ago**.

You can:

* Keep both as churn, or
* Treat “never transacted” as a separate segment (**never activated / dormant**).



### ✅ Approach 3 – MySQL Syntax (DATE Interval)

If you’re on MySQL and prefer `INTERVAL 3 MONTH`:

```sql
SELECT
    c.customer_id,
    c.name
FROM
    customers c
LEFT JOIN
    transactions t
    ON c.customer_id = t.customer_id
    AND t.txn_date >= (CURRENT_DATE - INTERVAL 3 MONTH)
WHERE
    t.customer_id IS NULL;
```

or using `MAX`:

```sql
SELECT
    c.customer_id,
    c.name,
    MAX(t.txn_date) AS last_txn_date
FROM
    customers c
LEFT JOIN
    transactions t
    ON c.customer_id = t.customer_id
GROUP BY
    c.customer_id,
    c.name
HAVING
    last_txn_date IS NULL
    OR last_txn_date < (CURRENT_DATE - INTERVAL 3 MONTH);
```

---

## 🧠 Analytics Layer on Top of Churn Detection

### 1️⃣ Add “days since last transaction” (for scoring risk)

```sql
WITH last_txn AS (
    SELECT
        customer_id,
        MAX(txn_date) AS last_txn_date
    FROM
        transactions
    GROUP BY
        customer_id
)
SELECT
    c.customer_id,
    c.name,
    last_txn_date,
    DATE_PART('day', CURRENT_DATE - last_txn_date) AS days_since_last_txn
FROM
    customers c
LEFT JOIN
    last_txn lt ON c.customer_id = lt.customer_id
ORDER BY
    days_since_last_txn DESC NULLS LAST;
```

You can now define:

* **0–30 days** → Active
* **31–90 days** → At risk
* **>90 days** → Churned



### 2️⃣ Create a “churn_flag” (Yes/No)

```sql
WITH last_txn AS (
    SELECT
        customer_id,
        MAX(txn_date) AS last_txn_date
    FROM
        transactions
    GROUP BY
        customer_id
)
SELECT
    c.customer_id,
    c.name,
    last_txn_date,
    CASE
        WHEN last_txn_date IS NULL THEN 'never_active'
        WHEN last_txn_date < CURRENT_DATE - INTERVAL '3 months' THEN 'churned'
        ELSE 'active'
    END AS churn_status
FROM
    customers c
LEFT JOIN
    last_txn lt ON c.customer_id = lt.customer_id;
```

This is fantastic for dashboards & modeling.



### 3️⃣ Monthly Churn Rate (Interview-Level)

> “For each month, find what percentage of customers became churned that month.”

Simple version (high-level idea):

* For each month, count customers whose **last_txn_date** falls more than 3 months before the end of that month and who were previously active.
* Divide by total active customers.

Even a simpler interview SQL: **count churned customers per month**:

```sql
WITH last_txn AS (
    SELECT
        customer_id,
        MAX(txn_date) AS last_txn_date
    FROM
        transactions
    GROUP BY
        customer_id
),
churned AS (
    SELECT
        customer_id,
        last_txn_date,
        DATE_TRUNC('month', last_txn_date + INTERVAL '3 months') AS churn_month
    FROM
        last_txn
    WHERE
        last_txn_date < CURRENT_DATE - INTERVAL '3 months'
)
SELECT
    churn_month,
    COUNT(*) AS churned_customers
FROM
    churned
GROUP BY
    churn_month
ORDER BY
    churn_month;
```

Interpretation:

* Approx assumption: customer is considered “churned” **3 months after their last transaction**.
* Above query groups churn by that “churn month”.

---

## 🎓 Interview-Level Questions Based on This Use Case

You can easily turn this into **interview-style SQL questions**:

1️⃣ **Question:**

> Find all customers who have **never done a transaction** since joining.

You’d left join `transactions` and filter `WHERE t.customer_id IS NULL` (no join match at all, not only last 3 months).

2️⃣ **Question:**

> For each customer, find the **total amount spent in the last 6 months** and whether they are currently active (at least 1 txn in last 30 days).

You’ll need:

* Conditional filters on dates
* Maybe CASE for active_flag

3️⃣ **Question:**

> For each month, compute the number of **new customers** and the number of **churned customers**.

This mixes:

* `MIN(txn_date)` or `join_date` for new
* `MAX(txn_date)` + churn definition for churned

These are exactly the kinds of problems asked in **product analytics / data analyst / data scientist** interviews.

---

## 🧪 Hands-On Practice Tasks (For You / Your Audience)

You can give these as tasks in reels, carousels, or blogs:

### 🔹 Task 1 – Active vs Inactive in Last 30 Days

> Write a query to label each customer as `active_30d` or `inactive_30d` based on whether they have at least one transaction in the last 30 days.

(Hint: use LEFT JOIN with date filter or MAX(txn_date) and a CASE.)



### 🔹 Task 2 – Find “Revived Customers”

> Customers who were inactive for at least 3 months but **came back** in the last month.

(Hint: need two time windows:

* Last txn before `CURRENT_DATE - INTERVAL '3 months'`
* And a recent txn within last 30 days.)



### 🔹 Task 3 – Churn by Segment

> Using a `plan_type` column (e.g., prepaid / postpaid / basic / premium), find the **churn rate by plan**.

(Hint: count churned customers per plan / total customers per plan.)



### 🔹 Task 4 – Days to Churn

> For each churned customer, compute how many days passed between `join_date` and their churn date.

(Hint: churn date = last_txn_date + 3 months, or just last_txn_date if defined that way.)



### 🔹 Task 5 – Cohort View: % Still Active After 90 Days

> Group customers by **join month**, then compute what percentage of that cohort is **still active** after 90 days.

(Hint:

* Cohort = DATE_TRUNC('month', join_date)
* Compare last_txn_date to join_date + 90 days.)

---
## Datasets

### **1. Telecom Customer Churn Dataset (IBM / Kaggle)**

**Best for:** Telecom churn, 90-day inactivity logic, plan-wise churn
**Contains:** Customer info, tenure, monthly charges, churn flag, services, payment method.

🔗 [https://www.kaggle.com/datasets/blastchar/telco-customer-churn](https://www.kaggle.com/datasets/blastchar/telco-customer-churn)

**Use it for:**

* Customers inactive > 90 days
* Churn by plan type
* Churn prediction features
* Active vs churned customers


###  **2. Online Retail II Dataset (Transactional – Perfect for SQL Churn)**

**Best for:** Transaction-based churn using “last purchase date” logic
**Contains:** Invoice date, customer ID, quantity, price, product.

🔗 [https://www.kaggle.com/datasets/mashlyn/online-retail-ii-uci](https://www.kaggle.com/datasets/mashlyn/online-retail-ii-uci)

**Use it for:**

* Customers with no purchases in last 3 months
* RFM analysis
* Revenue-based churn
* Active vs inactive users


### **3. SaaS / Subscription Churn Dataset**

**Best for:** SaaS, app users, subscription churn
**Contains:** Signup date, last activity, monthly usage, churn flag.

🔗 [https://www.kaggle.com/datasets/radmirzosimov/telecom-users-dataset](https://www.kaggle.com/datasets/radmirzosimov/telecom-users-dataset)

**Use it for:**

* 30/60/90-day inactivity churn
* Active vs dormant users
* Retention analysis
* Subscription lifecycle analysis


### **4. Customer Transactions Dataset (E-commerce – Multiple Purchases)**

**Best for:** SQL-based churn using transactions table
**Contains:** Customer ID, transaction date, amount, frequency.

🔗 [https://www.kaggle.com/datasets/priyankdl/customer-churn-dataset](https://www.kaggle.com/datasets/priyankdl/customer-churn-dataset)

**Use it for:**

* “No transactions in last N days” logic
* Repeat vs one-time customers
* Purchase frequency decay
* Revenue churn


### **5. Banking / Credit Card Customer Churn Dataset**

**Best for:** Financial churn use cases
**Contains:** Account activity, balance, tenure, churn flag.

🔗 [https://www.kaggle.com/datasets/sakshigoyal7/credit-card-customers](https://www.kaggle.com/datasets/sakshigoyal7/credit-card-customers)

**Use it for:**

* Inactive accounts
* High-value churned customers
* Product-wise churn
* Risk-based churn segmentation

