# **Marketing Campaign Analysis – Full Breakdown (Conversion Rate + SQL)**


## 🔹 Business Use Case: Measure Marketing Effectiveness

**Goal:**
Understand **how well each marketing campaign performs**, using data like:

* How many users **clicked** on ads
* How many of those **ended up purchasing**
* Which campaigns bring **high-intent** vs **low-quality** traffic

This helps a digital marketing / growth team:

* Optimize **budget allocation** (scale winning campaigns, pause bad ones)
* Compare **performance across channels** (Google Ads vs Meta vs Email)
* Improve **targeting, creatives, landing pages**

**Real-World Example:**
A digital marketing team runs multiple campaigns:

* `CAMP001` – Diwali Sale (Meta Ads)
* `CAMP002` – New User Offer (Google Ads)
* `CAMP003` – Email Retargeting

They want to know, for each campaign:

* How many users **clicked**
* How many **purchased** after clicking
* What is the **conversion rate**

> **Conversion Rate (CR)** =
> `(# of conversions / # of clicks) × 100`
> or
> `(# of users who purchased / # of users who clicked) × 100`

---

## 🧱 Sample Database Schema (Click + Purchase Attribution)

We’ll assume simple, realistic tables:

### `campaigns`

| column        | type    | description                |
| ------------- | ------- | -------------------------- |
| campaign_id   | INT     | Unique campaign identifier |
| campaign_name | VARCHAR | Campaign name              |
| channel       | VARCHAR | e.g. google, meta, email   |
| start_date    | DATE    | Start date                 |
| end_date      | DATE    | End date                   |



### `user_clicks`

| column      | type      | description                         |
| ----------- | --------- | ----------------------------------- |
| click_id    | BIGINT    | Unique click row                    |
| user_id     | INT       | Who clicked                         |
| campaign_id | INT       | Which campaign the click belongs to |
| click_time  | TIMESTAMP | When the click happened             |

Each row = **one ad click**.



### `orders` (or `purchases`)

| column      | type      | description                                 |
| ----------- | --------- | ------------------------------------------- |
| order_id    | BIGINT    | Unique order/purchase                       |
| user_id     | INT       | Who purchased                               |
| order_time  | TIMESTAMP | When purchase was made                      |
| campaign_id | INT       | Attributed campaign (from tracking / click) |

> In many systems, `campaign_id` is attached to the order through attribution logic (last touch / first touch / multi-touch).
> For simplicity here, we assume order has a direct `campaign_id` that matches `user_clicks`.

---

## 🎯 Core SQL Task

> **Calculate conversion rate per campaign using user click & purchase data.**

We’ll calculate **per campaign**:

* `clicks` → total number of clicks
* `conversions` → number of purchases
* `conversion_rate` → `conversions / clicks * 100 (%)`



## ✅ Version 1 – Based on Total Clicks and Total Orders

This is the simplest version: per campaign, count clicks & purchases.

```sql
SELECT
    c.campaign_id,
    c.campaign_name,
    COUNT(DISTINCT uc.click_id) AS total_clicks,
    COUNT(DISTINCT o.order_id) AS total_conversions,
    CASE
        WHEN COUNT(DISTINCT uc.click_id) = 0 THEN 0
        ELSE ROUND(
            COUNT(DISTINCT o.order_id)::DECIMAL
            / COUNT(DISTINCT uc.click_id) * 100,
            2
        )
    END AS conversion_rate_percent
FROM
    campaigns c
LEFT JOIN
    user_clicks uc
        ON c.campaign_id = uc.campaign_id
LEFT JOIN
    orders o
        ON c.campaign_id = o.campaign_id
GROUP BY
    c.campaign_id,
    c.campaign_name
ORDER BY
    conversion_rate_percent DESC;
```

**What this gives:**
One row per campaign with:

* `total_clicks` – total ad clicks
* `total_conversions` – total orders attributed
* `conversion_rate_percent` – campaign effectiveness



## ✅ Version 2 – Unique User-Based Conversion Rate

Sometimes, marketing wants to know:

> “Out of **users who clicked**, how many **ended up buying**?”

So instead of counting clicks and orders, we count **unique users**:

```sql
SELECT
    c.campaign_id,
    c.campaign_name,
    COUNT(DISTINCT uc.user_id) AS users_clicked,
    COUNT(DISTINCT o.user_id) AS users_converted,
    CASE
        WHEN COUNT(DISTINCT uc.user_id) = 0 THEN 0
        ELSE ROUND(
            COUNT(DISTINCT o.user_id)::DECIMAL
            / COUNT(DISTINCT uc.user_id) * 100,
            2
        )
    END AS user_conversion_rate_percent
FROM
    campaigns c
LEFT JOIN
    user_clicks uc
        ON c.campaign_id = uc.campaign_id
LEFT JOIN
    orders o
        ON c.campaign_id = o.campaign_id
GROUP BY
    c.campaign_id,
    c.campaign_name
ORDER BY
    user_conversion_rate_percent DESC;
```

**When this is useful:**

* When some users click multiple times
* You care about **unique user behavior**, not click volume



## ✅ Version 3 – Add a Date Range (e.g., Last 30 Days)

You rarely compute conversion rate on entire history; usually it’s for:

* Last 7 days
* Last 30 days
* Campaign active period

Example: last 30 days (PostgreSQL-style):

```sql
SELECT
    c.campaign_id,
    c.campaign_name,
    COUNT(DISTINCT uc.click_id) AS total_clicks,
    COUNT(DISTINCT o.order_id) AS total_conversions,
    ROUND(
        CASE
            WHEN COUNT(DISTINCT uc.click_id) = 0 THEN 0
            ELSE COUNT(DISTINCT o.order_id)::DECIMAL
                 / COUNT(DISTINCT uc.click_id) * 100
        END,
        2
    ) AS conversion_rate_percent
FROM
    campaigns c
LEFT JOIN
    user_clicks uc
        ON c.campaign_id = uc.campaign_id
       AND uc.click_time >= CURRENT_DATE - INTERVAL '30 days'
LEFT JOIN
    orders o
        ON c.campaign_id = o.campaign_id
       AND o.order_time >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY
    c.campaign_id,
    c.campaign_name
ORDER BY
    conversion_rate_percent DESC;
```

---

## 🧠 More Analytical Enhancements

Once conversion rate per campaign is ready, you can add **more metrics**:

* Cost per campaign
* Cost per click (CPC)
* Cost per acquisition (CPA)
* Revenue per campaign
* ROAS (Return On Ad Spend)

Assume we add:

### `campaigns` extra column:

* `total_spend` – money spent on campaign in given period

### 1️⃣ Add Revenue + ROAS + CPA

Assume `orders` has `order_amount`:

```sql
SELECT
    c.campaign_id,
    c.campaign_name,
    c.channel,
    SUM(DISTINCT c.total_spend) AS total_spend,  -- or take from another table
    COUNT(DISTINCT uc.click_id) AS total_clicks,
    COUNT(DISTINCT o.order_id) AS total_conversions,
    SUM(o.order_amount) AS total_revenue,
    ROUND(
        CASE
            WHEN COUNT(DISTINCT uc.click_id) = 0 THEN 0
            ELSE COUNT(DISTINCT o.order_id)::DECIMAL
                 / COUNT(DISTINCT uc.click_id) * 100
        END,
        2
    ) AS conversion_rate_percent,
    CASE
        WHEN COUNT(DISTINCT o.order_id) = 0 THEN NULL
        ELSE ROUND(
            SUM(DISTINCT c.total_spend)::DECIMAL
            / COUNT(DISTINCT o.order_id),
            2
        )
    END AS cpa,  -- Cost Per Acquisition
    CASE
        WHEN SUM(DISTINCT c.total_spend) = 0 THEN NULL
        ELSE ROUND(
            SUM(o.order_amount)::DECIMAL
            / SUM(DISTINCT c.total_spend),
            2
        )
    END AS roas  -- Return On Ad Spend
FROM
    campaigns c
LEFT JOIN
    user_clicks uc ON c.campaign_id = uc.campaign_id
LEFT JOIN
    orders o ON c.campaign_id = o.campaign_id
GROUP BY
    c.campaign_id,
    c.campaign_name,
    c.channel
ORDER BY
    roas DESC NULLS LAST;
```

> This kind of query is **exactly what performance marketers + finance teams** want.



### 2️⃣ Channel-Wise Conversion Rate

Sometimes you want **aggregate by channel** (Google, Meta, Email):

```sql
SELECT
    c.channel,
    COUNT(DISTINCT uc.click_id) AS total_clicks,
    COUNT(DISTINCT o.order_id) AS total_conversions,
    ROUND(
        CASE
            WHEN COUNT(DISTINCT uc.click_id) = 0 THEN 0
            ELSE COUNT(DISTINCT o.order_id)::DECIMAL
                 / COUNT(DISTINCT uc.click_id) * 100
        END,
        2
    ) AS conversion_rate_percent
FROM
    campaigns c
LEFT JOIN
    user_clicks uc ON c.campaign_id = uc.campaign_id
LEFT JOIN
    orders o ON c.campaign_id = o.campaign_id
GROUP BY
    c.channel
ORDER BY
    conversion_rate_percent DESC;
```



### 3️⃣ Time-Series: Daily Conversion Rate per Campaign

```sql
SELECT
    c.campaign_id,
    c.campaign_name,
    DATE(uc.click_time) AS day,
    COUNT(DISTINCT uc.click_id) AS clicks,
    COUNT(DISTINCT o.order_id) AS conversions,
    ROUND(
        CASE
            WHEN COUNT(DISTINCT uc.click_id) = 0 THEN 0
            ELSE COUNT(DISTINCT o.order_id)::DECIMAL
                 / COUNT(DISTINCT uc.click_id) * 100
        END,
        2
    ) AS conversion_rate_percent
FROM
    campaigns c
LEFT JOIN
    user_clicks uc
        ON c.campaign_id = uc.campaign_id
LEFT JOIN
    orders o
        ON c.campaign_id = o.campaign_id
       AND DATE(o.order_time) = DATE(uc.click_time)  -- same day assumption
GROUP BY
    c.campaign_id,
    c.campaign_name,
    DATE(uc.click_time)
ORDER BY
    c.campaign_id,
    day;
```

(*Real attribution windows can be longer than 1 day, but this is great for exercises.*)

---

## 🎓 Interview-Level Questions Based on This Use Case

You can convert this into strong SQL interview problems:

### **Question 1:**

For each campaign, calculate: clicks, conversions, conversion rate, and **rank campaigns by conversion rate** (highest to lowest).

> (Hint: window function `RANK() OVER (ORDER BY conversion_rate DESC)`.)



### **Question 2:**

For each campaign, compute conversion rate **for mobile vs desktop users**.

> (Assumes device info is stored in `user_clicks` or `users` table.)


### **Question 3:**

For each campaign, show **week-over-week change** in conversion rate.

> (Hint: compute weekly metrics, then use `LAG()` on conversion rate.)



### **Question 4:**

Find campaigns with **high click volume but low conversion rate** (e.g., clicks > 10,000 and CR < 1%).

> This reveals **inefficient campaigns** wasting spend.



### **Question 5:**

Given an A/B test where campaigns A and B share the same audience, compare their conversion rates and find which one wins statistically (at least at SQL level you can do part of this: group, compute CR & differences).

---

## 🧪 Hands-On Practice Tasks (Perfect for Your Audience)

Here are practice tasks you can give as exercises / reels / carousels:

### 🔹 Task 1 – Basic Conversion Rate

> For each campaign, calculate total clicks, total conversions, and conversion rate (%).



### 🔹 Task 2 – Top 3 Campaigns by Conversion Rate

> Show only campaigns with at least 1,000 clicks and rank the **top 3 by conversion rate**.



### 🔹 Task 3 – Channel vs Campaign Drill-Down

> For each channel, show:

* number of campaigns
* total clicks
* total conversions
* overall conversion rate



### 🔹 Task 4 – New vs Returning Users

> If you have `users.signup_date`, compare conversion rate of **new users** (joined in last 30 days) vs **old users**, per campaign.



### 🔹 Task 5 – Trend for a Single Campaign

> For a given `campaign_id`, generate a **daily time series** of clicks, conversions, and conversion rate for the last 14 days.



---
## Datasets
### 1. Marketing Campaign Results: 
> Sample marketing-campaign data of ~2240 customers (profile, channel performance, campaign success/failure) — ready for conversion / campaign-effectiveness analysis.
<a href="https://mavenanalytics.io/data-playground/marketing-campaign-results" target="_blank">
Download
</a>

### 2. Marketing Campaign Performance Dataset:
> Dataset that captures campaign performance (clicks/conversions) useful to compute conversion rate per campaign.
<a href="https://www.kaggle.com/datasets/manishabhatt22/marketing-campaign-performance-dataset" target="_blank">
Download
</a>

### 3. E-commerce Clickstream and Transaction Dataset:
> Includes user interactions (clicks, page-views) + transaction history — good to simulate click → purchase attribution and conversion funnels.
<a href="https://www.kaggle.com/datasets/waqi786/e-commerce-clickstream-and-transaction-dataset" target="_blank">
Download
</a>

### 4. Clickstream Data for Online Shopping:
> Historic clickstream data from an online shopping store (sessions, page views, purchase events) — useful for funnel analysis / attribution experiments.
<a href="https://archive.ics.uci.edu/dataset/553/clickstream%2Bdata%2Bfor%2Bonline%2Bshopping" target="_blank">
Download
</a>

### 5. Synthetic E‑commerce Data with Marketing Campaign:
> Synthetic but well-structured data representing sales + a “marketing_campaign” flag, useful for experimenting with before/after campaign analysis or A/B style impact analysis.
<a href="https://github.com/andresvourakis/synthetic-e-commerce-data-with-marketing-campaign" target="_blank">
Download
</a>
