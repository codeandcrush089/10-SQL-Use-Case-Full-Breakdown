# Customer Segmentation – SQL & Analytics Use Case


## 🔹 Business Use Case: Customer Segmentation

**Goal:** Group customers based on their purchase behavior, so that, for example, marketing can target “high-value customers” for premium offers — improving retention and revenue per user.

**Real-World Example:**
An e-commerce company wants to identify all customers who have spent more than ₹ 50,000 (or any threshold) in total purchases over time. Those customers might get exclusive discounts, loyalty rewards, upsell campaigns, or outreach for premium products.

Segmentation helps with:

* Customer loyalty / retention programs
* Personalized marketing
* Identifying high-value vs low-value vs churn-risk customers
* Better resource allocation for marketing / offers

---

## 🧱 Sample Database Schema (Realistic for Ecommerce + Customers)

Assume tables similar to:

### `customers`

| column                       | type                 | description                  |
| ---------------------------- | -------------------- | ---------------------------- |
| customer_id                  | INT                  | Unique customer identifier   |
| name                         | VARCHAR              | Customer name                |
| join_date                    | DATE                 | When the customer registered |
| email                        | VARCHAR              | Contact info (optional)      |
| — other demographic fields — | e.g. city, age, etc. |                              |

### `orders`

| column      | type | description                     |
| ----------- | ---- | ------------------------------- |
| order_id    | INT  | Unique order ID                 |
| customer_id | INT  | Which customer placed the order |
| order_date  | DATE | When order was placed           |

### `order_items`

| column        | type    | description                        |
| ------------- | ------- | ---------------------------------- |
| order_item_id | INT     | Row ID                             |
| order_id      | INT     | Which order this item belongs to   |
| product_id    | INT     | Which product was bought           |
| quantity      | INT     | How many units                     |
| price         | DECIMAL | Price per unit at time of purchase |

Optionally, you may have a `products` table for product metadata (category, category_id, etc.), but for basic customer-spend segmentation you may not need to join to products.

**Definition:**
Total purchase per customer = **SUM of (quantity × price)** across all their order items.

With this you can segment customers by their total spend.

---

## 🎯 Core SQL Task: Identify High-Value Customers (spend > ₹50,000)

Assuming you have relational schema as above:

```sql
SELECT
    c.customer_id,
    c.name,
    SUM(oi.quantity * oi.price) AS total_spent
FROM
    customers c
JOIN
    orders o ON c.customer_id = o.customer_id
JOIN
    order_items oi ON o.order_id = oi.order_id
GROUP BY
    c.customer_id,
    c.name
HAVING
    SUM(oi.quantity * oi.price) > 50000
ORDER BY
    total_spent DESC;
```

This query returns all customers whose cumulative spend exceeds ₹ 50,000, sorted by amount spent (highest first).

If you don’t need `name`, you can just group by `customer_id`.

If you want **additional data** (e.g. since when they joined, number of orders, last order date), you can extend:

```sql
SELECT
    c.customer_id,
    c.name,
    COUNT(DISTINCT o.order_id)       AS total_orders,
    SUM(oi.quantity * oi.price)      AS total_spent,
    MIN(o.order_date)                AS first_order_date,
    MAX(o.order_date)                AS last_order_date
FROM
    customers c
JOIN
    orders o ON c.customer_id = o.customer_id
JOIN
    order_items oi ON o.order_id = oi.order_id
GROUP BY
    c.customer_id,
    c.name
HAVING
    SUM(oi.quantity * oi.price) > 50000
ORDER BY
    total_spent DESC;
```

---

## 🧠 Advanced Analytics & Interview-Level Variants

Here are some more complex tasks/questions — useful for interviews, reporting, or deeper segmentation:

### A) Segment Customers into Buckets: high / medium / low spenders

```sql
WITH customer_spend AS (
    SELECT
      c.customer_id,
      SUM(oi.quantity * oi.price) AS total_spent
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY c.customer_id
)

SELECT
    customer_id,
    total_spent,
    CASE
      WHEN total_spent > 100000 THEN 'Platinum'
      WHEN total_spent > 50000 THEN 'Gold'
      WHEN total_spent > 10000 THEN 'Silver'
      ELSE 'Bronze'
    END AS spend_segment
FROM
    customer_spend
ORDER BY
    total_spent DESC;
```

You can adjust spending thresholds as per business. This gives a segmentation for marketing or loyalty programs.

---

### B) Recency–Frequency–Monetary (RFM) Segmentation (basic version)

```sql
WITH customer_orders AS (
  SELECT
    c.customer_id,
    COUNT(DISTINCT o.order_id) AS frequency,
    MAX(o.order_date)         AS last_order_date,
    SUM(oi.quantity * oi.price) AS monetary
  FROM customers c
  JOIN orders o ON c.customer_id = o.customer_id
  JOIN order_items oi ON o.order_id = oi.order_id
  GROUP BY c.customer_id
)

SELECT
  customer_id,
  frequency,
  monetary,
  DATE_PART('day', CURRENT_DATE - last_order_date) AS recency_days
FROM
  customer_orders
ORDER BY
  monetary DESC;
```

Using `frequency`, `monetary` (spend), and `recency` (how many days since last purchase) — you can classify customers (e.g. “recent & high spenders”, “churn risk”, “frequent but low spenders”, etc.).

---

### C) Identify “New High-Value Customers”: customers who have spent > ₹50,000 but only joined in last 6 months

```sql
WITH customer_spend AS (
  SELECT
    c.customer_id,
    c.join_date,
    SUM(oi.quantity * oi.price) AS total_spent
  FROM customers c
  JOIN orders o ON c.customer_id = o.customer_id
  JOIN order_items oi ON o.order_id = oi.order_id
  GROUP BY c.customer_id, c.join_date
)

SELECT
  customer_id,
  total_spent,
  join_date
FROM
  customer_spend
WHERE
  total_spent > 50000
  AND join_date >= (CURRENT_DATE - INTERVAL '6 months')
ORDER BY
  total_spent DESC;
```

This helps to identify new signups who quickly became high-value — useful for “welcome + upsell” campaigns.

---

## 🎓 Hands-On Practice Tasks (for Yourself or For Learners)

You can pose these as exercises to practise SQL with real or sample datasets:

1. **Find Average Spend per Customer** — What is the average total spend per customer across entire dataset.

2. **Identify Top 10% Customers by Spend** — Find customers whose total spend is in the top 10 percentile.

3. **Find Customers Who Purchased in Last 3 Months But Spent Less Than ₹ 5000** — Potential low-value, maybe churn-risk customers.

4. **Compute “Customer Lifetime Value (CLV)”** — For each customer: total spend / number of years since join (or last purchase).

5. **Cohort Analysis** — Group customers by join month, then compute average spend per cohort over time to see how retention or spend evolves.

---

## 🔗 5 Public / Sample Datasets for Practice Data Links

Here are **5 publicly available datasets** (or dataset collections) you can download and use to practise the above customer segmentation tasks:

### 1. Brazilian E-Commerce Public Dataset by Olist 
> ~100,000 orders, with customers, products, order data — good for segmentation and RFM analysis.
<a href="https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce" target="_blank">
Download
</a>

### 2. E‑Commerce Orders Data
> Transactional data across orders, items, and products — useful for spend and customer analysis.
<a href="https://www.kaggle.com/datasets/carrie1/ecommerce-data" target="_blank">
Download
</a>

### 3. Global Online Orders
> E-commerce database containing info about products, categories, customers, and orders. Useful for segmentation with product-category insight.
<a href="https://www.kaggle.com/datasets/javierspdatabase/global-online-orders" target="_blank">
Download
</a>

### 4. E‑commerce Order & Supply Chain Dataset
> Contains orders, items, customers, payments and products — a complete transactional model for deeper analysis.
<a href="https://www.kaggle.com/datasets/bytadit/ecommerce-order-dataset" target="_blank">
Download
</a>

### 5. E‑Commerce Data Playground (from Maven Analytics) – “Toy Store E-Commerce Database”
> A synthetic but realistic dataset to practice segmentation, cohort, and RFM analyses — useful for learning/training.
<a href="https://mavenanalytics.io/data-playground" target="_blank">
Download
</a>

---

> 💡 **Tip:** Download one of these datasets (CSV/Parquet), load into your database 
(MySQL/PostgreSQL/SQLite), and run the SQL queries above to practise.


