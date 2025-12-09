# **Recommendation Systems – Full Breakdown (Products Bought Together + SQL)**



## 🔹 Business Use Case: Recommendation Systems

**Goal:**
Suggest **relevant products** to users to:

* Increase **average order value (AOV)**
* Improve **user experience** (“this feels made for me”)
* Boost **cross-sell** and **up-sell**
* Power sections like: “Frequently Bought Together”, “Customers Also Bought”, “You May Also Like”

**Real-World Example (Amazon-style):**
When a user adds a **laptop** to cart, Amazon might suggest:

* Laptop bag
* Mouse
* Keyboard
* Screen protector

These recommendations usually come from **order history** of many users, using patterns like:

> “Products that are **frequently bought together** in the same order.”

This is basically **Market Basket Analysis / Association Rules**.

---

## 🧱 Sample Database Schema (E-Commerce Orders)

We’ll use a classic structure:

### `orders`

| column     | type | description                   |
| ---------- | ---- | ----------------------------- |
| order_id   | INT  | Unique order identifier       |
| user_id    | INT  | Customer who placed the order |
| order_date | DATE | When the order was created    |

### `order_items`

| column        | type    | description                      |
| ------------- | ------- | -------------------------------- |
| order_item_id | INT     | Unique row                       |
| order_id      | INT     | Which order this line belongs to |
| product_id    | INT     | Which product was purchased      |
| quantity      | INT     | Units of this product            |
| price         | DECIMAL | Price per unit at purchase time  |

### `products` (optional, for product names)

| column       | type    | description         |
| ------------ | ------- | ------------------- |
| product_id   | INT     | Unique product ID   |
| product_name | VARCHAR | Name of product     |
| category     | VARCHAR | Category (optional) |

We assume:

* Each order has **one or more** `order_items` rows
* Products bought together = **products in the same `order_id`**

---

## 🎯 Core SQL Task

> **Find products frequently bought together using order data.**

In SQL terms:

* For each `order_id`, find **pairs of products**
* Count how many times each pair appears across orders
* Sort by highest count



## ✅ Step 1 – Generate Product Pairs from the Same Order (Self Join)

We self-join `order_items` on `order_id` and ensure we don’t duplicate or pair a product with itself.

```sql
SELECT
    oi1.product_id AS product_a,
    oi2.product_id AS product_b
FROM
    order_items oi1
JOIN
    order_items oi2
    ON oi1.order_id = oi2.order_id
   AND oi1.product_id < oi2.product_id;  -- avoid duplicates and self-pairs
```

* `oi1.product_id < oi2.product_id`

  * ensures we don’t get both (A,B) and (B,A)
  * also avoids (A,A)

This gives all product pairs in all orders.



## ✅ Step 2 – Count How Often Each Pair Appears (Frequency)

Wrap that in an aggregation:

```sql
SELECT
    oi1.product_id AS product_a,
    oi2.product_id AS product_b,
    COUNT(*) AS times_bought_together
FROM
    order_items oi1
JOIN
    order_items oi2
    ON oi1.order_id = oi2.order_id
   AND oi1.product_id < oi2.product_id
GROUP BY
    oi1.product_id,
    oi2.product_id
ORDER BY
    times_bought_together DESC;
```

This gives you a ranking of product pairs by **how many orders contain both products**.

You can add a `LIMIT` to get top N pairs.



## ✅ Step 3 – Add Product Names (More Readable Output)

```sql
SELECT
    p1.product_name AS product_a,
    p2.product_name AS product_b,
    pair.times_bought_together
FROM (
    SELECT
        oi1.product_id AS product_a,
        oi2.product_id AS product_b,
        COUNT(*) AS times_bought_together
    FROM
        order_items oi1
    JOIN
        order_items oi2
        ON oi1.order_id = oi2.order_id
       AND oi1.product_id < oi2.product_id
    GROUP BY
        oi1.product_id,
        oi2.product_id
) pair
JOIN
    products p1 ON pair.product_a = p1.product_id
JOIN
    products p2 ON pair.product_b = p2.product_id
ORDER BY
    times_bought_together DESC;
```

Now you get results like:

| product_a | product_b        | times_bought_together |
| --------- | ---------------- | --------------------- |
| Laptop    | Laptop Bag       | 421                   |
| Phone     | Phone Case       | 390                   |
| Phone     | Screen Protector | 355                   |

---

## 🧠 Going Deeper: Support, Confidence, Lift (Association Rules)

In recommendation systems, for pair (A → B), we often compute:

* **Support(A,B)**:
  Fraction of orders that contain **both A and B**
* **Confidence(A → B)**:
  Among orders that contain **A**, how many also contain **B**
* **Lift(A → B)**:
  How much more often A and B occur together **than if they were independent**

Let’s focus on useful ones for SQL:



### 1️⃣ Compute Support and Confidence for A → B

Steps:

1. Count **orders per product**
2. Count **orders per product pair**
3. Compute `confidence = pair_count / product_A_count`

#### Step A – Orders per Product

```sql
WITH product_orders AS (
    SELECT
        product_id,
        COUNT(DISTINCT order_id) AS orders_with_product
    FROM
        order_items
    GROUP BY
        product_id
),
product_pairs AS (
    SELECT
        oi1.product_id AS product_a,
        oi2.product_id AS product_b,
        COUNT(DISTINCT oi1.order_id) AS orders_with_both
    FROM
        order_items oi1
    JOIN
        order_items oi2
        ON oi1.order_id = oi2.order_id
       AND oi1.product_id < oi2.product_id
    GROUP BY
        oi1.product_id,
        oi2.product_id
)

SELECT
    pp.product_a,
    pp.product_b,
    pp.orders_with_both,
    po_a.orders_with_product AS orders_with_a,
    ROUND(
        pp.orders_with_both::DECIMAL / po_a.orders_with_product,
        4
    ) AS confidence_a_to_b
FROM
    product_pairs pp
JOIN
    product_orders po_a ON pp.product_a = po_a.product_id
ORDER BY
    confidence_a_to_b DESC;
```

Interpretation:

> “If a user buys **product A**, what’s the probability they also buy **product B**?”

This is much better than raw counts for recommendations.



### 2️⃣ Generate Recommendations for a Specific Product

Suppose you want recommendations for `product_id = 101` (e.g., a Laptop):

```sql
WITH product_pairs AS (
    SELECT
        oi1.product_id AS product_a,
        oi2.product_id AS product_b,
        COUNT(DISTINCT oi1.order_id) AS orders_with_both
    FROM
        order_items oi1
    JOIN
        order_items oi2
        ON oi1.order_id = oi2.order_id
       AND oi1.product_id < oi2.product_id
    GROUP BY
        oi1.product_id,
        oi2.product_id
),
product_orders AS (
    SELECT
        product_id,
        COUNT(DISTINCT order_id) AS orders_with_product
    FROM
        order_items
    GROUP BY
        product_id
)
SELECT
    CASE
        WHEN pp.product_a = 101 THEN pp.product_b
        ELSE pp.product_a
    END AS recommended_product_id,
    pp.orders_with_both,
    ROUND(
        pp.orders_with_both::DECIMAL / po.orders_with_product,
        4
    ) AS confidence
FROM
    product_pairs pp
JOIN
    product_orders po
    ON (po.product_id = pp.product_a AND pp.product_a = 101)
    OR (po.product_id = pp.product_b AND pp.product_b = 101)
ORDER BY
    confidence DESC,
    orders_with_both DESC
LIMIT 5;
```

This gives “top 5 products to recommend when a user buys product 101”.

(You can join `products` table to show names.)

---

## 🎓 Interview-Level Questions from This Use Case

You can easily turn this into SQL interview challenges:

### Q1: “Top 3 Co-Purchased Products for Every Product”

> For each product, find its **top 3 most frequently co-purchased products**.

(Hint:

* Build pairs
* For each `product_a`, rank `product_b` by `times_bought_together` using `ROW_NUMBER()` or `RANK()`
* Filter `rank <= 3`.)



### Q2: “Category-Level Co-Purchase Patterns”

> Instead of product-to-product, find **category-to-category** pairs that are frequently bought together.

(Hint: join `order_items` → `products` twice and group by `category_a`, `category_b`.)



### Q3: “Time-Window Based Recommendations”

> Only use orders from the last 6 months when calculating “frequently bought together”.

(Hint: filter `orders.order_date >= CURRENT_DATE - INTERVAL '6 months'` before building pairs.)



### Q4: “Avoid Obvious Bundles”

Sometimes items are **always sold together** as a bundle (e.g., phone + charger in same box).
You might want to **ignore those** and only show *discovery* cross-sells.

> Write a query that ignores product pairs where **co-purchase rate > 90%** of all orders containing product A.

(Hint: use `confidence` and filter.)



### Q5: “Lift-Based Ranking”

If you also compute **overall popularity** of B, you can compute **lift**:

`lift(A → B) = confidence(A → B) / (orders_with_B / total_orders)`

Pairs with high lift are **truly associated**, not just both popular individually.

---

## 🧪 Hands-On Practice Tasks (Great for Content)

You can give these tasks to your audience as **SQL challenges**:

### 🔹 Task 1 – Most Common Product Pairs

> List top 10 product pairs by `times_bought_together`.



### 🔹 Task 2 – Recommendations for a Single Product

> Given a `product_id`, return top 5 **“frequently bought together”** products.



### 🔹 Task 3 – Co-Purchase by Category

> For each category, find another category that is **most frequently co-purchased** with it.



### 🔹 Task 4 – Confidence Threshold

> Only show product pairs where:

* `times_bought_together >= 20`
* `confidence >= 0.3`

These are stronger candidates for recommendations.



### 🔹 Task 5 – Build a Simple “Also Bought” Table

> Create a materialized view or table like:

| base_product_id | recommended_product_id | score |

Where `score` = combination of frequency + confidence.
Fill this using SQL from your product-pair stats.


