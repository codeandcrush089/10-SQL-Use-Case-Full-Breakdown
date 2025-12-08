# **Sales Performance Analysis – Full Breakdown (Data Analyst + SQL View)**


## 🔹 Business Use Case

**Goal:** Track **daily, monthly, and yearly sales** and understand

* Which **product categories** make the most money
* How **revenue is trending** over time
* Which months/products need more marketing or discount push

**Real-World Example:**
An **e-commerce company** (like Amazon, Flipkart, Meesho) wants to know:

* “Which product categories generate the **highest revenue every month**?”
* “Is **Electronics** growing faster than **Fashion**?”
* “Which month had **highest overall revenue** this year?”

This directly helps:

* **Marketing team** → where to run campaigns
* **Inventory team** → which products to stock
* **Leadership** → which categories to invest more in

---

## 🧱 Sample Database Schema (Simple + Realistic)

Assume we have these tables:

### `orders`

| column      | type | description                |
| ----------- | ---- | -------------------------- |
| order_id    | INT  | Unique order ID            |
| order_date  | DATE | Date when order was placed |
| customer_id | INT  | Who placed the order       |

### `order_items`

| column        | type    | description                        |
| ------------- | ------- | ---------------------------------- |
| order_item_id | INT     | Unique item row                    |
| order_id      | INT     | Link to orders table               |
| product_id    | INT     | Which product was sold             |
| quantity      | INT     | How many units were sold           |
| price         | DECIMAL | Price per unit at time of purchase |

### `products`

| column       | type    | description                |
| ------------ | ------- | -------------------------- |
| product_id   | INT     | Unique product ID          |
| product_name | VARCHAR | Product name               |
| category     | VARCHAR | Category (Electronics etc) |

> **Revenue formula:** `revenue = quantity * price`

---

## 🎯 Core SQL Task

> **Task:** Find **monthly revenue by product category** from a sales table.

We’ll calculate revenue from `order_items` and join with `products` to get the category.

### ✅ Query: Monthly Revenue by Category

**PostgreSQL / MySQL 8+ style using `DATE_TRUNC`:**

```sql
SELECT
    DATE_TRUNC('month', o.order_date) AS month,
    p.category,
    SUM(oi.quantity * oi.price) AS monthly_revenue
FROM
    orders o
JOIN
    order_items oi ON o.order_id = oi.order_id
JOIN
    products p ON oi.product_id = p.product_id
GROUP BY
    DATE_TRUNC('month', o.order_date),
    p.category
ORDER BY
    month,
    monthly_revenue DESC;
```

### If your SQL doesn’t support `DATE_TRUNC` (like some MySQL versions):

```sql
SELECT
    DATE_FORMAT(o.order_date, '%Y-%m-01') AS month_start,
    p.category,
    SUM(oi.quantity * oi.price) AS monthly_revenue
FROM
    orders o
JOIN
    order_items oi ON o.order_id = oi.order_id
JOIN
    products p ON oi.product_id = p.product_id
GROUP BY
    DATE_FORMAT(o.order_date, '%Y-%m-01'),
    p.category
ORDER BY
    month_start,
    monthly_revenue DESC;
```

This gives you:
➡ One row per **month + category**
➡ Total **revenue** for that combination

---

## 📊 Expanding the Use Case: Daily, Monthly, Yearly Views

### 1) Daily Revenue (All Categories Combined)

```sql
SELECT
    o.order_date AS day,
    SUM(oi.quantity * oi.price) AS daily_revenue
FROM
    orders o
JOIN
    order_items oi ON o.order_id = oi.order_id
GROUP BY
    o.order_date
ORDER BY
    day;
```

### 2) Monthly Revenue (All Categories Combined)

```sql
SELECT
    DATE_TRUNC('month', o.order_date) AS month,
    SUM(oi.quantity * oi.price) AS monthly_revenue
FROM
    orders o
JOIN
    order_items oi ON o.order_id = oi.order_id
GROUP BY
    DATE_TRUNC('month', o.order_date)
ORDER BY
    month;
```

### 3) Yearly Revenue (All Categories Combined)

```sql
SELECT
    EXTRACT(YEAR FROM o.order_date) AS year,
    SUM(oi.quantity * oi.price) AS yearly_revenue
FROM
    orders o
JOIN
    order_items oi ON o.order_id = oi.order_id
GROUP BY
    EXTRACT(YEAR FROM o.order_date)
ORDER BY
    year;
```

---

## 🧠 Advanced Analytics Layer (Interview-Level)

These are the types of queries you get in **data analyst / BI interviews.**

### A) Top 3 Categories by Revenue for Each Month

```sql
WITH monthly_category_revenue AS (
    SELECT
        DATE_TRUNC('month', o.order_date) AS month,
        p.category,
        SUM(oi.quantity * oi.price) AS revenue
    FROM
        orders o
    JOIN
        order_items oi ON o.order_id = oi.order_id
    JOIN
        products p ON oi.product_id = p.product_id
    GROUP BY
        DATE_TRUNC('month', o.order_date),
        p.category
)

SELECT *
FROM (
    SELECT
        month,
        category,
        revenue,
        RANK() OVER (
            PARTITION BY month
            ORDER BY revenue DESC
        ) AS category_rank
    FROM
        monthly_category_revenue
) ranked
WHERE
    category_rank <= 3
ORDER BY
    month,
    category_rank;
```

**What this shows:**
➡ For every month, **top 3 categories** by revenue, with ranking.

---

### B) Month-over-Month Revenue Growth

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', o.order_date) AS month,
        SUM(oi.quantity * oi.price) AS revenue
    FROM
        orders o
    JOIN
        order_items oi ON o.order_id = oi.order_id
    GROUP BY
        DATE_TRUNC('month', o.order_date)
)

SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100,
        2
    ) AS mom_growth_percent
FROM
    monthly_revenue
ORDER BY
    month;
```

**Use:** Helps answer:

> “Did we grow or drop compared to last month, and by how much (%)?”

---

### C) Running Total of Revenue (Cumulative)

```sql
WITH daily_revenue AS (
    SELECT
        o.order_date AS day,
        SUM(oi.quantity * oi.price) AS revenue
    FROM
        orders o
    JOIN
        order_items oi ON o.order_id = oi.order_id
    GROUP BY
        o.order_date
)

SELECT
    day,
    revenue,
    SUM(revenue) OVER (ORDER BY day) AS cumulative_revenue
FROM
    daily_revenue
ORDER BY
    day;
```

**Use:** See how revenue is **accumulating over the year**, like a running meter.

---

## 🎓 Interview-Level Problem Ideas (Based on This Use Case)

You can turn this into **interview-style SQL questions** like:

1️⃣ **Question:**

> Find the **top 5 products by revenue** in the last 3 months.

* Filter date (last 3 months)
* Group by product
* Sort and limit

2️⃣ **Question:**

> For each category, find the **month with the highest revenue** and the value.

* First compute monthly revenue by category
* Use window functions (ROW_NUMBER / RANK) to pick max per category

3️⃣ **Question:**

> Find customers whose **monthly spending is increasing** for at least 3 consecutive months.

* Compute customer’s monthly spend
* Use window + pattern (advanced)

These can be used as **Instagram carousels, YouTube shorts, or blog posts**.

---

## 🧪 Hands-On Practice Tasks (Based on Same Schema)

You can give your audience these challenges:

### Task 1 – Best Month Overall

> Write a query to find the **month with the highest total revenue** in the entire dataset.

**Hint:**

* Use `DATE_TRUNC('month', order_date)` + `SUM(quantity * price)`
* Sort by revenue, pick top 1

---

### Task 2 – Category Share (%) of Revenue

> For each month, calculate **what percentage of monthly revenue** comes from each category.

**Hint:**

* Compute monthly revenue per category
* Compute total monthly revenue
* Join and divide: `category_revenue / total_month_revenue * 100`

---

### Task 3 – Average Order Value (AOV) by Month

> For each month, find the **average order value** (total revenue per order).

**Hint:**

1. First get revenue per order
2. Then group by month and take `AVG(order_revenue)`

```sql
WITH order_revenue AS (
    SELECT
        o.order_id,
        o.order_date,
        SUM(oi.quantity * oi.price) AS order_total
    FROM
        orders o
    JOIN
        order_items oi ON o.order_id = oi.order_id
    GROUP BY
        o.order_id,
        o.order_date
)

SELECT
    DATE_TRUNC('month', order_date) AS month,
    AVG(order_total) AS avg_order_value
FROM
    order_revenue
GROUP BY
    DATE_TRUNC('month', order_date)
ORDER BY
    month;
```

---
## Datasets
### 1. E‑Commerce Data: 
> Real online-retail transaction data (orders, items) from a UK-based non-store online retailer. Great for practicing revenue by date, category, etc.
<a href="https://www.kaggle.com/datasets/carrie1/ecommerce-data" target="_blank">
Download
</a>

### 2. Online Sales Dataset – Popular Marketplace Data
> Structured online-sales data with product & category information — suitable for category-based revenue / sales-performance analysis.
<a href="https://www.kaggle.com/datasets/shreyanshverma27/online-sales-dataset-popular-marketplace-data" target="_blank">
Download
</a>

### 3. Sample Sales Data bundle
> A synthetic but realistic set of sales/retail datasets (orders, online-store orders, POS, inventory, etc.) — useful for experimentation, pivot-table work, or building dashboards.
<a href="https://excelx.com/practice-data/sales-retail" target="_blank">
Download
</a>

### 4. Warehouse and Retail Sales
> Monthly sales and item/department-level data from a retail context — good for time-series revenue analysis (monthly, quarterly, yearly).
<a href="https://catalog.data.gov/dataset/warehouse-and-retail-sales" target="_blank">
Download
</a>

### 5. ga4_obfuscated_sample_ecommerce
> Demo ecommerce dataset with structured transactional data — useful if you want to test SQL work in a cloud data-warehouse or BigQuery context.
<a href="https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset" target="_blank">
Download
</a>
