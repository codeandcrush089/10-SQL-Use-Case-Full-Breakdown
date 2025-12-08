# **Inventory Management – Full Breakdown (Stock Levels + SQL)**


## 🔹 Business Use Case: Monitor Stock Levels

**Goal:**
Track **current inventory levels** to avoid:

* **Stock-outs** → lost sales because items are unavailable
* **Over-stocking** → cash stuck in slow-moving inventory

**Real-World Example (Retail Store):**
A retail store (offline + online) wants to know:

* Which products are **about to go out of stock**
* Which items should be **reordered now**
* Which items are **overstocked** and need discount/promotion

The store defines for each product a **restock_level** (minimum safe quantity).
If `current_quantity < restock_level`, the product is considered **“needs restock”**.

---

## 🧱 Sample Database Schema (Inventory-Focused)

A simple and realistic schema:

### `products`

| column        | type    | description                          |
| ------------- | ------- | ------------------------------------ |
| product_id    | INT     | Unique product ID                    |
| product_name  | VARCHAR | Product name                         |
| category      | VARCHAR | Category (e.g. Electronics, Grocery) |
| restock_level | INT     | Minimum quantity before reordering   |


### `inventory`

(If you store stock separately per location/warehouse)

| column       | type | description                            |
| ------------ | ---- | -------------------------------------- |
| product_id   | INT  | Which product                          |
| warehouse_id | INT  | Which warehouse / store                |
| quantity     | INT  | Current on-hand stock in that location |

Or, in a simpler setup you might store `current_quantity` directly in the `products` table.

We’ll assume `inventory` + `products` for flexibility.

---

## 🎯 Core SQL Task

> **List products with quantity less than restock level.**

We want products where total quantity (or per warehouse) is below `restock_level`.

### ✅ Case 1 – Quantity Stored in `products` Table

If your `products` table already has `current_quantity`:

```sql
SELECT
    product_id,
    product_name,
    current_quantity,
    restock_level
FROM
    products
WHERE
    current_quantity < restock_level
ORDER BY
    current_quantity ASC;
```

This gives all **low-stock products**.


### ✅ Case 2 – Quantity Stored in Separate `inventory` Table (More Realistic)

Here, we may have multiple warehouses per product.

#### Option A: Check per Product (Total Stock Across Warehouses)

```sql
SELECT
    p.product_id,
    p.product_name,
    p.category,
    SUM(i.quantity) AS total_quantity,
    p.restock_level
FROM
    products p
JOIN
    inventory i ON p.product_id = i.product_id
GROUP BY
    p.product_id,
    p.product_name,
    p.category,
    p.restock_level
HAVING
    SUM(i.quantity) < p.restock_level
ORDER BY
    total_quantity ASC;
```

**What this shows:**

* All products where **total stock across all warehouses** is below the threshold.


#### Option B: Check Per Warehouse (Store-Wise Low Stock)

```sql
SELECT
    p.product_id,
    p.product_name,
    i.warehouse_id,
    i.quantity,
    p.restock_level
FROM
    products p
JOIN
    inventory i ON p.product_id = i.product_id
WHERE
    i.quantity < p.restock_level
ORDER BY
    i.warehouse_id,
    i.quantity ASC;
```

This answers:

> “In which specific **warehouse / store** is each product under-stocked?”

---

## 🧠 Extended Analytics (Inventory-Focused Thinking)

Inventory management usually involves:

* **Reorder quantity**: how much to order once threshold is hit
* **Fast-moving vs slow-moving items**
* **Stock coverage**: how many days stock can last at current demand

We can add this logic using SQL when we have more columns.

Assume also a `sales` (or `order_items`) table with average daily demand.

### 1️⃣ Reorder Quantity Suggestion

Assume you have an estimated **target_stock_level** and know current total quantity:

```sql
WITH stock AS (
    SELECT
        p.product_id,
        p.product_name,
        p.category,
        SUM(i.quantity) AS total_quantity,
        p.restock_level,
        p.target_stock_level  -- ideal stock you want to maintain
    FROM
        products p
    JOIN
        inventory i ON p.product_id = i.product_id
    GROUP BY
        p.product_id,
        p.product_name,
        p.category,
        p.restock_level,
        p.target_stock_level
)
SELECT
    product_id,
    product_name,
    total_quantity,
    restock_level,
    target_stock_level,
    CASE
        WHEN total_quantity < restock_level
        THEN (target_stock_level - total_quantity)
        ELSE 0
    END AS reorder_quantity
FROM
    stock
WHERE
    total_quantity < restock_level
ORDER BY
    total_quantity ASC;
```

This tells you:

> “How many units should I reorder **for each low-stock product** to reach the desired level?”


### 2️⃣ Stock Coverage in Days (Using Average Daily Sales)

Assume you have `daily_sales` table with average daily units sold per product:

`daily_sales`
| product_id | avg_daily_sales |

Then:

```sql
WITH stock AS (
    SELECT
        product_id,
        SUM(quantity) AS total_quantity
    FROM
        inventory
    GROUP BY
        product_id
)
SELECT
    s.product_id,
    p.product_name,
    s.total_quantity,
    ds.avg_daily_sales,
    CASE
        WHEN ds.avg_daily_sales > 0
        THEN ROUND(s.total_quantity::NUMERIC / ds.avg_daily_sales, 2)
        ELSE NULL
    END AS days_of_stock_left
FROM
    stock s
JOIN
    products p ON s.product_id = p.product_id
LEFT JOIN
    daily_sales ds ON s.product_id = ds.product_id
ORDER BY
    days_of_stock_left;
```

This answers:

> “At current sales rate, how many **days** before each product goes out of stock?”

You can then define:

* `< 7 days` → urgent restock
* `7–30 days` → normal
* `> 30 days` → safe (or maybe overstock)


### 3️⃣ Identify Overstocked Products

You might have a column `max_stock_capacity` or want to flag anything way above target.

```sql
SELECT
    p.product_id,
    p.product_name,
    SUM(i.quantity) AS total_quantity,
    p.target_stock_level
FROM
    products p
JOIN
    inventory i ON p.product_id = i.product_id
GROUP BY
    p.product_id,
    p.product_name,
    p.target_stock_level
HAVING
    SUM(i.quantity) > p.target_stock_level * 1.5  -- 50% above target
ORDER BY
    total_quantity DESC;
```

These are candidates for:

* Discounts
* Bundles
* Special offers

---

## 🎓 Interview-Level Questions from This Use Case

You can easily turn this into SQL interview problems:

1️⃣ **Question:**

> Write a query to find all products that are **low stock** (below restock level) **and have had at least 10 sales in the last 7 days**.

(Hint: join inventory + products + sales, use HAVING filters on both stock and recent sales.)


2️⃣ **Question:**

> For each category, compute the **number of low-stock products** and sort categories by this count (descending).

(Hint: subquery for low-stock products, then group by category.)


3️⃣ **Question:**

> For each warehouse, find the **total value of inventory** (quantity × cost_price) and flag warehouses where total value is above a threshold.

(Hint: add `cost_price` in products and aggregate.)


4️⃣ **Question:**

> Identify products that have **zero stock** but **had sales in the last 30 days**.

These are **stock-out risk** items that clearly have demand.


5️⃣ **Question:**

> Compute, for each product, how many times it has **dropped below restock_level** in the last 6 months (using daily snapshots or transaction logs).

---

## 🧪 Hands-On Practice Tasks

You (or your audience) can practise with these:

### 🔹 Task 1 – Simple Low Stock List

> List all products where stock < restock level, sorted by **lowest quantity first**.


### 🔹 Task 2 – Category-Wise Low Stock Count

> Find, per category, how many products are below restock level.


### 🔹 Task 3 – Restock Priority Score

> Give each product a **priority score**:

* **High** → stock < 50% of restock_level
* **Medium** → stock between 50–100% of restock_level
* **Low** → stock ≥ restock_level

Use a CASE expression.


### 🔹 Task 4 – Warehouse Health Check

> For each warehouse, compute:

* total products
* number of low-stock products
* percentage low-stock


### 🔹 Task 5 – Merge Inventory & Sales

> Join inventory with sales data to find **high-demand + low-stock** items. These are “critical” restock products.

