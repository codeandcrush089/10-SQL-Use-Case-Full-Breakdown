# **Website Traffic Analysis – Full Breakdown (SQL + Product Analytics View)**



## 🔹 Business Use Case: Website Traffic Analysis

**Goal:**
Analyze how users visit and interact with a website to understand:

* Which **pages get the most traffic**
* Where users **spend most of their time**
* Which pages cause **drop-offs (high bounce / exit rate)**
* How traffic changes **daily, weekly, monthly**

**Real-World Example:**
A startup runs a content website / SaaS platform and wants to know:

* Which pages are bringing the **most visitors**
* Which blog posts drive the **highest engagement**
* Which landing pages should be **optimized for conversions**

This directly helps:

* **Marketing team** → optimize campaigns
* **Product team** → improve UX of important pages
* **SEO team** → focus on high-traffic & high-potential pages

---

## 🧱 Sample Database Schema (Realistic Web Analytics Model)

Assume these core tables:

### `users`

| column      | type    | description               |
| ----------- | ------- | ------------------------- |
| user_id     | INT     | Unique user ID            |
| signup_date | DATE    | When the user registered  |
| country     | VARCHAR | User location (optional)  |
| device      | VARCHAR | mobile / desktop / tablet |


### `page_views`

| column     | type      | description                             |
| ---------- | --------- | --------------------------------------- |
| view_id    | INT       | Unique page view ID                     |
| user_id    | INT       | Who visited the page                    |
| page_url   | VARCHAR   | Page URL or slug (e.g. /home, /pricing) |
| view_time  | TIMESTAMP | When the page was viewed                |
| session_id | VARCHAR   | Session identifier                      |
| time_spent | INT       | Time on page in seconds (optional)      |

Each row = **one page visit (one hit/view)**.

---

## 🎯 Core SQL Task

> **Display the top 5 most visited web pages.**

This simply means:

* Count number of rows per `page_url`
* Sort by count
* Take top 5

### ✅ Basic Query (Most Common Interview Version)

```sql
SELECT
    page_url,
    COUNT(*) AS total_visits
FROM
    page_views
GROUP BY
    page_url
ORDER BY
    total_visits DESC
LIMIT 5;
```

**Output Example:**

| page_url | total_visits |
| -------- | ------------ |
| /home    | 125,430      |
| /pricing | 87,210       |
| /blog    | 70,115       |
| /login   | 65,003       |
| /contact | 41,229       |

This directly answers:

> “Which pages are getting the most traffic?”


## 🧠 Slightly Advanced Versions (Real-World Variations)

### 1️⃣ Top 5 Most Visited Pages in the Last 30 Days

```sql
SELECT
    page_url,
    COUNT(*) AS total_visits
FROM
    page_views
WHERE
    view_time >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY
    page_url
ORDER BY
    total_visits DESC
LIMIT 5;
```

Used in **monthly traffic reports**.


### 2️⃣ Top Pages by Unique Users (Not Total Hits)

Sometimes one user refreshes many times. So instead of counting hits, we count **distinct users**.

```sql
SELECT
    page_url,
    COUNT(DISTINCT user_id) AS unique_visitors
FROM
    page_views
GROUP BY
    page_url
ORDER BY
    unique_visitors DESC
LIMIT 5;
```

This answers:

> “Which pages attract the most **unique users**?”


### 3️⃣ Top Pages by Engagement (Average Time Spent)

```sql
SELECT
    page_url,
    AVG(time_spent) AS avg_time_spent_seconds
FROM
    page_views
GROUP BY
    page_url
ORDER BY
    avg_time_spent_seconds DESC
LIMIT 5;
```

This shows **which pages hold user attention the longest**, not just traffic volume.


## 📊 Traffic Trend Analysis (Daily, Weekly, Monthly)

### ✅ Daily Page Views

```sql
SELECT
    DATE(view_time) AS visit_date,
    COUNT(*) AS total_page_views
FROM
    page_views
GROUP BY
    DATE(view_time)
ORDER BY
    visit_date;
```


### ✅ Monthly Page Views

```sql
SELECT
    DATE_TRUNC('month', view_time) AS month,
    COUNT(*) AS total_page_views
FROM
    page_views
GROUP BY
    DATE_TRUNC('month', view_time)
ORDER BY
    month;
```

Used for **growth tracking & reporting**.

---

## 🧠 Advanced Analytical Queries (Interview-Level)

### A) Rank Pages by Traffic (Using Window Function)

```sql
SELECT
    page_url,
    COUNT(*) AS total_visits,
    RANK() OVER (ORDER BY COUNT(*) DESC) AS traffic_rank
FROM
    page_views
GROUP BY
    page_url
ORDER BY
    traffic_rank;
```

This is often asked in **window function interviews**.



### B) Bounce Detection (Single-Page Sessions)

> Bounce = user visits only **1 page in a session** and leaves.

```sql
WITH session_counts AS (
    SELECT
        session_id,
        COUNT(*) AS pages_in_session
    FROM
        page_views
    GROUP BY
        session_id
)

SELECT
    pv.page_url,
    COUNT(*) AS bounce_count
FROM
    page_views pv
JOIN
    session_counts sc
    ON pv.session_id = sc.session_id
WHERE
    sc.pages_in_session = 1
GROUP BY
    pv.page_url
ORDER BY
    bounce_count DESC;
```

This helps identify pages that lose users quickly.


### C) Entry Pages (First Page in a Session)

```sql
WITH first_page AS (
    SELECT
        session_id,
        MIN(view_time) AS first_view_time
    FROM
        page_views
    GROUP BY
        session_id
)

SELECT
    pv.page_url,
    COUNT(*) AS entry_count
FROM
    page_views pv
JOIN
    first_page fp
    ON pv.session_id = fp.session_id
   AND pv.view_time = fp.first_view_time
GROUP BY
    pv.page_url
ORDER BY
    entry_count DESC;
```

Used to analyze **landing pages**.


### D) Exit Pages (Last Page in a Session)

```sql
WITH last_page AS (
    SELECT
        session_id,
        MAX(view_time) AS last_view_time
    FROM
        page_views
    GROUP BY
        session_id
)

SELECT
    pv.page_url,
    COUNT(*) AS exit_count
FROM
    page_views pv
JOIN
    last_page lp
    ON pv.session_id = lp.session_id
   AND pv.view_time = lp.last_view_time
GROUP BY
    pv.page_url
ORDER BY
    exit_count DESC;
```

Shows where users **most often leave the website**.

---

## 🎓 Interview-Level Questions From This Use Case

You can easily frame questions like:

1️⃣ **Top Traffic Page Per Day**

> For each day, find the page with the highest traffic.

(Hint: GROUP BY + RANK() OVER (PARTITION BY date ORDER BY COUNT(*) DESC))


2️⃣ **Returning Visitor Pages**

> Which pages are most visited by users who visited the site more than once?

(Hint: users with COUNT(DISTINCT visit day) > 1)


3️⃣ **Page Conversion Funnel**

> How many users moved from `/home` → `/pricing` → `/checkout`?

(Hint: session-level filtering + ordered page sequences)


4️⃣ **Traffic Source Analysis (If Source Exists)**

> Which traffic source (Google, Ads, Direct) brings the most page views?

(Hint: group by traffic_source)

---

## 🧪 Hands-On Practice Tasks (For Learners / Audience)

You can give these as SQL challenges:

### 🔹 Task 1 – Top 3 Pages by Unique Visitors Last Month

> Find top 3 pages by **distinct user count** in the last 30 days.


### 🔹 Task 2 – Average Pages per Session

> For each session, calculate number of pages visited and then find overall average.

**Hint:**

* First group by `session_id`
* Then compute `AVG(pages_per_session)`


### 🔹 Task 3 – Mobile vs Desktop Traffic

> Compare total page views for **mobile vs desktop users**.

(Hint: join `page_views` with `users` on `user_id`.)


### 🔹 Task 4 – Identify Low-Performing Pages

> Find pages with **< 100 visits in the last 30 days**.


### 🔹 Task 5 – Peak Traffic Hour

> Find the hour of the day with the **highest number of page views**.

**Hint:**

```sql
SELECT
    EXTRACT(HOUR FROM view_time) AS hour_of_day,
    COUNT(*) AS total_views
FROM
    page_views
GROUP BY
    hour_of_day
ORDER BY
    total_views DESC;
```

---

## 🔗 How This Use Case Is Applied in Industry

* **Startups** → find which landing pages convert best
* **Content platforms** → identify viral blog posts
* **E-commerce** → analyze category and product page traffic
* **SaaS companies** → see how users navigate pricing, docs, and signup flows

This exact analysis feeds into tools like:

* Google Analytics
* Adobe Analytics
* Mixpanel
* Amplitude
* Power BI / Tableau dashboards backed by SQL

