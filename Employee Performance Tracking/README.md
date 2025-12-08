# **Employee Performance Tracking – Full Breakdown (Dept-Wise Ranking + SQL)**


## 🔹 Business Use Case: Employee Performance Tracking

**Goal:**
Use **measurable performance data** to:

* Track employee **productivity** over time
* Compare performance **within departments** (fair comparison)
* Identify **top performers** for bonuses, promotions, recognition
* Spot **low performers** who may need training or support

**Real-World Example (HR Team):**
HR collects monthly performance scores from managers or systems (e.g., sales closed, tickets resolved, code quality, OKR completion).
They want to know:

* “Who are the **top 5 performers in each department** this month?”
* “Which employees in a department are **consistently top-ranked**?”
* “Who needs **performance improvement plans**?”

---

## 🧱 Sample Database Schema (HR / Performance)

A simple realistic design:

### `employees`

| column        | type    | description                  |
| ------------- | ------- | ---------------------------- |
| emp_id        | INT     | Unique employee ID           |
| emp_name      | VARCHAR | Employee full name           |
| department_id | INT     | Foreign key to departments   |
| join_date     | DATE    | Date of joining              |
| status        | VARCHAR | active / resigned / on_leave |



### `departments`

| column        | type    | description                 |
| ------------- | ------- | --------------------------- |
| department_id | INT     | Unique department ID        |
| dept_name     | VARCHAR | e.g. Sales, HR, Engineering |



### `performance_scores`

| column       | type    | description                                |
| ------------ | ------- | ------------------------------------------ |
| perf_id      | INT     | Unique row ID                              |
| emp_id       | INT     | Which employee                             |
| period_start | DATE    | Start of eval period (e.g., 2025-11-01)    |
| period_end   | DATE    | End of eval period (e.g., 2025-11-30)      |
| score        | DECIMAL | Performance score (e.g., 0–100, or rating) |
| reviewer     | VARCHAR | Manager / system name (optional)           |

> You can also have just one `month` or `period` column like `'2025-11'` instead of `period_start` and `period_end`.

---

## 🎯 Core SQL Task

> **Rank employees by department-wise performance score.**

We’ll assume we want **rankings for a specific month or period** (e.g., November 2025).

### ✅ Step 1 – Filter for Specific Period (Example: November 2025)

Let’s say `period_start` / `period_end` represent a month. For simplicity:

```sql
SELECT
    e.emp_id,
    e.emp_name,
    d.dept_name,
    p.score
FROM
    performance_scores p
JOIN
    employees e ON p.emp_id = e.emp_id
JOIN
    departments d ON e.department_id = d.department_id
WHERE
    p.period_start = DATE '2025-11-01'
    AND p.period_end   = DATE '2025-11-30';
```

Now we add **ranking logic**.



### ✅ Step 2 – Department-wise Ranking (Window Function)

We want to rank each employee **within their department** based on `score`, highest first.

```sql
SELECT
    e.emp_id,
    e.emp_name,
    d.dept_name,
    p.score,
    RANK() OVER (
        PARTITION BY d.department_id
        ORDER BY p.score DESC
    ) AS dept_rank
FROM
    performance_scores p
JOIN
    employees e ON p.emp_id = e.emp_id
JOIN
    departments d ON e.department_id = d.department_id
WHERE
    p.period_start = DATE '2025-11-01'
    AND p.period_end   = DATE '2025-11-30'
ORDER BY
    d.dept_name,
    dept_rank;
```

**What this does:**

* `PARTITION BY d.department_id` → ranking restarts **per department**
* `ORDER BY p.score DESC` → highest score = rank 1
* `RANK()` → ties get the same rank (1,1,3 pattern)

You could use:

* `DENSE_RANK()` → 1,1,2 pattern
* `ROW_NUMBER()` → always unique 1,2,3 (even if scores equal)



### ✅ Step 3 – Show Only Top N Per Department (e.g., Top 3)

Wrap it in a subquery and filter:

```sql
SELECT *
FROM (
    SELECT
        e.emp_id,
        e.emp_name,
        d.dept_name,
        p.score,
        RANK() OVER (
            PARTITION BY d.department_id
            ORDER BY p.score DESC
        ) AS dept_rank
    FROM
        performance_scores p
    JOIN
        employees e ON p.emp_id = e.emp_id
    JOIN
        departments d ON e.department_id = d.department_id
    WHERE
        p.period_start = DATE '2025-11-01'
        AND p.period_end   = DATE '2025-11-30'
) ranked
WHERE
    dept_rank <= 3
ORDER BY
    dept_name,
    dept_rank;
```

This gives the **Top 3 performers per department** for that month.

---

## 🧠 More Realistic Scenarios & Variations

### 1️⃣ Use a `period` Column Instead of Date Range

If your `performance_scores` table has:

| emp_id | period     | score |
| ------ | ---------- | ----- |
| 101    | 2025-11-01 | 88    |

Then:

```sql
SELECT
    e.emp_id,
    e.emp_name,
    d.dept_name,
    p.score,
    RANK() OVER (
        PARTITION BY d.department_id
        ORDER BY p.score DESC
    ) AS dept_rank
FROM
    performance_scores p
JOIN
    employees e ON p.emp_id = e.emp_id
JOIN
    departments d ON e.department_id = d.department_id
WHERE
    p.period = DATE '2025-11-01';
```



### 2️⃣ Average Score Across Multiple Months & Then Rank

Sometimes HR wants to see **average performance over last 6 months** instead of just 1 month.

```sql
WITH avg_perf AS (
    SELECT
        p.emp_id,
        AVG(p.score) AS avg_score
    FROM
        performance_scores p
    WHERE
        p.period_start >= CURRENT_DATE - INTERVAL '6 months'
    GROUP BY
        p.emp_id
)
SELECT
    e.emp_id,
    e.emp_name,
    d.dept_name,
    ap.avg_score,
    RANK() OVER (
        PARTITION BY d.department_id
        ORDER BY ap.avg_score DESC
    ) AS dept_rank_6m
FROM
    avg_perf ap
JOIN
    employees e ON ap.emp_id = e.emp_id
JOIN
    departments d ON e.department_id = d.department_id
ORDER BY
    d.dept_name,
    dept_rank_6m;
```

This identifies **consistently strong employees** per department.



### 3️⃣ Add Performance Bands (Rating Labels)

You can add labels like:

* 90+ → “Outstanding”
* 80–89 → “Exceeds Expectations”
* 70–79 → “Meets Expectations”
* < 70 → “Needs Improvement”

```sql
SELECT
    e.emp_id,
    e.emp_name,
    d.dept_name,
    p.score,
    RANK() OVER (
        PARTITION BY d.department_id
        ORDER BY p.score DESC
    ) AS dept_rank,
    CASE
        WHEN p.score >= 90 THEN 'Outstanding'
        WHEN p.score >= 80 THEN 'Exceeds Expectations'
        WHEN p.score >= 70 THEN 'Meets Expectations'
        ELSE 'Needs Improvement'
    END AS performance_band
FROM
    performance_scores p
JOIN
    employees e ON p.emp_id = e.emp_id
JOIN
    departments d ON e.department_id = d.department_id
WHERE
    p.period_start = DATE '2025-11-01'
    AND p.period_end   = DATE '2025-11-30'
ORDER BY
    d.dept_name,
    dept_rank;
```

Now HR can quickly see **who’s in which band** in each department.

---

## 🎓 Interview-Level Questions Based on This Use Case

You can convert this into strong SQL interview problems:

### Q1: “Top 2 Employees per Department over Last Quarter”

> For the last 3 months combined, find the **top 2 employees per department** by average performance score.

(Hint: avg per emp in 3 months → rank per dept → filter rank <= 2)



### Q2: “Most Improved Employee per Department”

> For each department, find the employee whose score improved the most from previous month to current month.

General steps:

* Self-join performance_scores or use window function `LAG(score)` over (emp_id ORDER BY period)
* Compute `score_diff = current_score - prev_score`
* Rank by `score_diff` within department



### Q3: “Employees Below Department Average”

> In a given month, list employees whose score is **below their department’s average score**.

```sql
WITH dept_avg AS (
    SELECT
        d.department_id,
        AVG(p.score) AS dept_avg_score
    FROM
        performance_scores p
    JOIN
        employees e ON p.emp_id = e.emp_id
    JOIN
        departments d ON e.department_id = d.department_id
    WHERE
        p.period_start = DATE '2025-11-01'
        AND p.period_end   = DATE '2025-11-30'
    GROUP BY
        d.department_id
)
SELECT
    e.emp_id,
    e.emp_name,
    d.dept_name,
    p.score,
    da.dept_avg_score
FROM
    performance_scores p
JOIN
    employees e ON p.emp_id = e.emp_id
JOIN
    departments d ON e.department_id = d.department_id
JOIN
    dept_avg da ON d.department_id = da.department_id
WHERE
    p.period_start = DATE '2025-11-01'
    AND p.period_end   = DATE '2025-11-30'
    AND p.score < da.dept_avg_score
ORDER BY
    d.dept_name,
    p.score;
```

Great to show understanding of **aggregates + joins**.

---

## 🧪 Hands-On Practice Tasks (Great for Your Audience)

You can use these as practice exercises:

### 🔹 Task 1 – Department-Wise Top Performer

> For a given month, find the **top 1 employee per department** (if multiple employees tie, list them all).



### 🔹 Task 2 – Performance Trend

> For each employee, show their **last 3 months’ scores** and a column indicating whether they are **improving** (latest score > previous score).

(Hint: use `LAG(score)` window function.)



### 🔹 Task 3 – New Joiners vs Old Employees

> Compare **average performance score of employees who joined in last 6 months** vs those who joined before that.



### 🔹 Task 4 – Performance Distribution per Department

> For each department, categorize employees into bands (“Outstanding”, “Exceeds”, etc.) and count how many fall into each band.



### 🔹 Task 5 – Underperformers for 3 Consecutive Months

> Find employees whose score was **< 70 for 3 months in a row**.

(Hint: use window functions and pattern over time per emp.)

