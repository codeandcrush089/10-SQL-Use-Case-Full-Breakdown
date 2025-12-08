# **Fraud Detection – Full Breakdown (Suspicious Transactions + SQL)**


## 🔹 Business Use Case: Identify Suspicious Transactions

**Goal:**
Detect **unusual / risky / suspicious transactions** so the system can:

* Flag them for **manual review**
* Trigger **OTP / step-up authentication**
* Temporarily **hold or delay** the transaction
* Feed them into a **fraud detection model** (ML) as signals

**Real-World Example (Bank):**
A bank wants to flag cases like:

* A user whose typical transaction size is ₹1,000 suddenly makes a transfer of ₹50,000
* A dormant account suddenly starts sending large amounts
* Many high-value transactions in a very short time

One simple rule-based logic is:

> “Flag transactions whose amount is **greater than 3× the user’s average transaction value**.”

This is not a full fraud system, but a **good first filter**.

---

## 🧱 Sample Database Schema (Banking / Wallet Style)

Assume a simple schema:

### `users`

| column     | type    | description                    |
| ---------- | ------- | ------------------------------ |
| user_id    | INT     | Unique user identifier         |
| name       | VARCHAR | User name                      |
| join_date  | DATE    | When account was created       |
| risk_level | VARCHAR | low / medium / high (optional) |


### `transactions`

| column      | type      | description                                   |
| ----------- | --------- | --------------------------------------------- |
| txn_id      | BIGINT    | Unique transaction ID                         |
| user_id     | INT       | Which user made the transaction               |
| txn_time    | TIMESTAMP | When it occurred                              |
| amount      | DECIMAL   | Transaction amount (₹)                        |
| txn_type    | VARCHAR   | debit / credit / transfer / card_payment etc. |
| merchant_id | INT       | Where it was spent (optional)                 |
| status      | VARCHAR   | success / failed / reversed / pending         |

We want to compare **each transaction’s amount** with that **user’s average amount**.

---

## 🎯 Core SQL Task

> **Find transactions greater than 3× the user’s average transaction value.**

There are two main ways to do this:


### ✅ Approach 1 – Using a Per-User Aggregation (Subquery / CTE)

1. First, compute each user’s **average transaction value**
2. Then join it back to individual transactions and filter

```sql
WITH user_avg AS (
    SELECT
        user_id,
        AVG(amount) AS avg_txn_amount
    FROM
        transactions
    GROUP BY
        user_id
)

SELECT
    t.txn_id,
    t.user_id,
    t.txn_time,
    t.amount,
    ua.avg_txn_amount
FROM
    transactions t
JOIN
    user_avg ua
    ON t.user_id = ua.user_id
WHERE
    t.amount > 3 * ua.avg_txn_amount
ORDER BY
    t.txn_time DESC;
```

**What you get:**

* All transactions where `amount > 3 × user’s overall average amount`
* Includes their average so reviewers see **how abnormal** it is

> 🔎 This uses the **average over all history** for the user.
> In some setups, you might prefer **average over last N days** (we’ll show that next).


### ✅ Approach 2 – Using Window Functions (No Separate CTE Needed)

```sql
SELECT
    txn_id,
    user_id,
    txn_time,
    amount,
    AVG(amount) OVER (PARTITION BY user_id) AS avg_txn_amount
FROM
    transactions
QUALIFY
    amount > 3 * AVG(amount) OVER (PARTITION BY user_id);
```

> `QUALIFY` is supported in some warehouses (e.g., BigQuery, Snowflake).
> For systems without `QUALIFY`, wrap in a subquery:

```sql
SELECT *
FROM (
    SELECT
        txn_id,
        user_id,
        txn_time,
        amount,
        AVG(amount) OVER (PARTITION BY user_id) AS avg_txn_amount
    FROM
        transactions
) t
WHERE
    amount > 3 * avg_txn_amount;
```

Same logic, slightly different style.

---

## 🧠 More Realistic Enhancements

In production, you often want to:

* Use **recent behavior** (last 30–90 days) instead of entire history
* Exclude obvious outliers from the avg (e.g., first filter very high values)
* Add **other rules** along with “3× average”

Here are common patterns.


### 1️⃣ Use Average Over Last 90 Days Only

```sql
WITH recent_txn AS (
    SELECT
        *
    FROM
        transactions
    WHERE
        txn_time >= CURRENT_DATE - INTERVAL '90 days'
),
user_avg_90d AS (
    SELECT
        user_id,
        AVG(amount) AS avg_txn_amount_90d
    FROM
        recent_txn
    GROUP BY
        user_id
)

SELECT
    t.txn_id,
    t.user_id,
    t.txn_time,
    t.amount,
    ua.avg_txn_amount_90d
FROM
    transactions t
JOIN
    user_avg_90d ua
    ON t.user_id = ua.user_id
WHERE
    t.amount > 3 * ua.avg_txn_amount_90d
ORDER BY
    t.txn_time DESC;
```

**Why this is better:**

* User behavior can change over years
* Most fraud rules care about **recent normal behavior**


### 2️⃣ Add a Minimum Amount Threshold

Sometimes small amounts (₹50, ₹100) should not be flagged even if they are 3× the user’s avg (if user’s avg is tiny).

Add a **minimum absolute value**, e.g., only flag if transaction is above ₹10,000:

```sql
...
WHERE
    t.amount > 3 * ua.avg_txn_amount_90d
    AND t.amount >= 10000;
```

So you get only **high-impact suspicious** transactions.


### 3️⃣ Separate by Transaction Type

You might want different thresholds for:

* **Bank transfers**
* **Card swipes**
* **ATM withdrawals**

Example: only apply rule to `txn_type = 'transfer'`:

```sql
WHERE
    t.txn_type = 'transfer'
    AND t.amount > 3 * ua.avg_txn_amount
```


## 📊 Fraud Detection Context: This Rule is One Signal

Real fraud systems don’t rely on a **single SQL rule**; they use multiple rules:

* Amount > 3× user average
* Many transactions in short time (e.g., 5+- high-value tx in <10 minutes)
* New device / new location / new IP
* Odd time (e.g., 3–4 AM for that user)
* Merchant categories unusual for that user

But the **3× avg rule** is a **great simple example** and very common in interviews.

---

## 🎓 Interview-Level SQL Questions You Can Build from This

You can easily create interview-style questions like:

### Q1: “Top N Most Suspicious Transactions Per User”

> For each user, find the **top 3 most suspicious transactions** by ratio = `amount / avg_amount`.

```sql
WITH txn_with_avg AS (
    SELECT
        txn_id,
        user_id,
        txn_time,
        amount,
        AVG(amount) OVER (PARTITION BY user_id) AS avg_amount,
        amount / NULLIF(AVG(amount) OVER (PARTITION BY user_id), 0) AS ratio
    FROM
        transactions
)
SELECT *
FROM (
    SELECT
        txn_id,
        user_id,
        txn_time,
        amount,
        avg_amount,
        ratio,
        RANK() OVER (PARTITION BY user_id ORDER BY ratio DESC) AS rnk
    FROM
        txn_with_avg
) t
WHERE
    rnk <= 3
ORDER BY
    user_id,
    rnk;
```


### Q2: “Flag Suspicious Bursts of Activity”

> Identify users who have **more than 5 transactions > 3× average in a single day.**

```sql
WITH suspicious AS (
    WITH user_avg AS (
        SELECT user_id, AVG(amount) AS avg_amount
        FROM transactions
        GROUP BY user_id
    )
    SELECT
        t.user_id,
        DATE(t.txn_time) AS txn_date,
        t.amount,
        ua.avg_amount
    FROM
        transactions t
    JOIN
        user_avg ua
        ON t.user_id = ua.user_id
    WHERE
        t.amount > 3 * ua.avg_amount
)
SELECT
    user_id,
    txn_date,
    COUNT(*) AS suspicious_count
FROM
    suspicious
GROUP BY
    user_id,
    txn_date
HAVING
    COUNT(*) > 5
ORDER BY
    suspicious_count DESC;
```

These customers might be in the middle of a **fraud spree**.


### Q3: “Z-Score Style Outlier Detection”

Instead of 3× average, use **(value – mean) / stddev > k**:

```sql
SELECT
    txn_id,
    user_id,
    txn_time,
    amount,
    AVG(amount) OVER (PARTITION BY user_id) AS mean_amt,
    STDDEV_POP(amount) OVER (PARTITION BY user_id) AS std_amt
FROM
    transactions
QUALIFY
    std_amt > 0
    AND (amount - mean_amt) / std_amt > 3;   -- 3 standard deviations
```

This is more statistically grounded and shows maturity in interviews.

---

## 🧪 Hands-On Practice Tasks (Great for Content / Reels / Blog)

You can derive these tasks for your followers/students:

### 🔹 Task 1 – Basic Suspicious Transactions

> Find all transactions that are more than **3× the user’s average amount**, show `txn_id`, `user_id`, `amount`, `avg_amount`, and `ratio`.


### 🔹 Task 2 – Daily Suspicious Count per User

> For each user and day, count how many suspicious transactions they had (using 3× avg rule). Filter only where count ≥ 2.


### 🔹 Task 3 – Suspicious Merchant Detection

> For each merchant, find how many **suspicious transactions** (3× rule) they have and sort from highest to lowest.


### 🔹 Task 4 – Risk Score

> For each transaction, create a **“risk_score”** column:

* score 3 if amount > 5× avg
* score 2 if amount between 3× and 5× avg
* score 1 otherwise

Use a `CASE` expression.


### 🔹 Task 5 – New-User Fraud Check

> Consider only users with `join_date >= CURRENT_DATE - INTERVAL '30 days'`. Among those, find all suspicious transactions (3× avg) and return the **top 10** by amount.

