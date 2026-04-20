## 🎯 ARRAY_AGG 完整指南

### 💡 一句話說明

**ARRAY_AGG = 把多行資料變成一個陣列**

---

## 📚 基本概念

### **問題情境**

```sql
-- 原始資料（多行）
user_id | product
--------|----------
101     | iPhone
101     | AirPods
101     | Case
102     | iPad
102     | Pencil
```

**問題：如何把每個用戶買的商品變成一個列表？**

---

### **解法：ARRAY_AGG**

```sql
SELECT 
  user_id,
  ARRAY_AGG(product) AS purchased_products
FROM orders
GROUP BY user_id;
```

**結果：**

```
user_id | purchased_products
--------|-----------------------------------
101     | ['iPhone', 'AirPods', 'Case']
102     | ['iPad', 'Pencil']
```

✅ **多行變成一個陣列！**

---

## 🔧 基本語法

```sql
ARRAY_AGG(欄位名稱 [ORDER BY 排序] [LIMIT 數量])
```

---

## 📊 實際例子

### **範例 1：基本用法**

```sql
WITH orders AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'iPhone' AS product),
    STRUCT(101, 'AirPods'),
    STRUCT(101, 'Case'),
    STRUCT(102, 'iPad'),
    STRUCT(102, 'Pencil')
  ])
)

SELECT 
  user_id,
  ARRAY_AGG(product) AS products
FROM orders
GROUP BY user_id;
```

**結果：**

```
user_id | products
--------|-----------------------------------
101     | ['iPhone', 'AirPods', 'Case']
102     | ['iPad', 'Pencil']
```

---

### **範例 2：加上排序 ORDER BY**

```sql
WITH orders AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'iPhone' AS product, 3 AS order_num),
    STRUCT(101, 'AirPods', 1),
    STRUCT(101, 'Case', 2),
    STRUCT(102, 'iPad', 2),
    STRUCT(102, 'Pencil', 1)
  ])
)

SELECT 
  user_id,
  ARRAY_AGG(product ORDER BY order_num) AS products_in_order
FROM orders
GROUP BY user_id;
```

**結果（按 order_num 排序）：**

```
user_id | products_in_order
--------|-----------------------------------
101     | ['AirPods', 'Case', 'iPhone']  ← 1, 2, 3 順序
102     | ['Pencil', 'iPad']              ← 1, 2 順序
```

---

### **範例 3：限制數量 LIMIT**

```sql
WITH orders AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'iPhone' AS product, DATE('2024-01-15') AS date),
    STRUCT(101, 'AirPods', DATE('2024-01-20')),
    STRUCT(101, 'Case', DATE('2024-02-01')),
    STRUCT(101, 'iPad', DATE('2024-02-10')),
    STRUCT(101, 'Pencil', DATE('2024-03-01'))
  ])
)

SELECT 
  user_id,
  -- 只取最近 3 筆
  ARRAY_AGG(product ORDER BY date DESC LIMIT 3) AS recent_3_products
FROM orders
GROUP BY user_id;
```

**結果（只保留最近 3 個）：**

```
user_id | recent_3_products
--------|-----------------------------------
101     | ['Pencil', 'iPad', 'Case']  ← 最新的 3 個
```

---

## 🔥 進階用法

### **1. ARRAY_AGG + STRUCT（聚合多個欄位）**

```sql
WITH orders AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'iPhone' AS product, 30000 AS price, 2 AS qty),
    STRUCT(101, 'AirPods', 5000, 1),
    STRUCT(101, 'Case', 500, 3)
  ])
)

SELECT 
  user_id,
  ARRAY_AGG(
    STRUCT(product, price, qty)
    ORDER BY price DESC
  ) AS order_details
FROM orders
GROUP BY user_id;
```

**結果：**

```
user_id | order_details
--------|------------------------------------------------
101     | [
        |   {product: 'iPhone', price: 30000, qty: 2},
        |   {product: 'AirPods', price: 5000, qty: 1},
        |   {product: 'Case', price: 500, qty: 3}
        | ]
```

---

### **2. ARRAY_AGG + DISTINCT（去重）**

```sql
WITH visits AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'home' AS page),
    STRUCT(101, 'products'),
    STRUCT(101, 'home'),      -- 重複
    STRUCT(101, 'products'),  -- 重複
    STRUCT(101, 'checkout')
  ])
)

SELECT 
  user_id,
  ARRAY_AGG(DISTINCT page) AS unique_pages
FROM visits
GROUP BY user_id;
```

**結果：**

```
user_id | unique_pages
--------|-----------------------------------
101     | ['home', 'products', 'checkout']  ← 自動去重
```

---

### **3. ARRAY_AGG + IF/CASE（條件聚合）**

```sql
WITH transactions AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'purchase' AS type, 1000 AS amount),
    STRUCT(101, 'refund', -200),
    STRUCT(101, 'purchase', 2000),
    STRUCT(101, 'refund', -500),
    STRUCT(101, 'purchase', 1500)
  ])
)

SELECT 
  user_id,
  -- 只聚合購買記錄
  ARRAY_AGG(amount ORDER BY amount DESC) 
    FILTER (WHERE type = 'purchase') AS purchase_amounts,
  
  -- 只聚合退款記錄
  ARRAY_AGG(amount) 
    FILTER (WHERE type = 'refund') AS refund_amounts
FROM transactions
GROUP BY user_id;
```

**結果：**

```
user_id | purchase_amounts   | refund_amounts
--------|--------------------|-----------------
101     | [2000, 1500, 1000] | [-200, -500]
```

---

### **4. 巢狀聚合（ARRAY_AGG 內再 ARRAY_AGG）**

```sql
WITH order_items AS (
  SELECT * FROM UNNEST([
    STRUCT(1 AS order_id, 101 AS user_id, 'iPhone' AS product),
    STRUCT(1, 101, 'Case'),
    STRUCT(2, 101, 'iPad'),
    STRUCT(3, 102, 'AirPods'),
    STRUCT(3, 102, 'Pencil')
  ])
)

-- 先按訂單聚合商品
, orders_with_items AS (
  SELECT 
    user_id,
    order_id,
    ARRAY_AGG(product) AS items
  FROM order_items
  GROUP BY user_id, order_id
)

-- 再按用戶聚合訂單
SELECT 
  user_id,
  ARRAY_AGG(
    STRUCT(order_id, items)
    ORDER BY order_id
  ) AS all_orders
FROM orders_with_items
GROUP BY user_id;
```

**結果：**

```
user_id | all_orders
--------|--------------------------------------------
101     | [
        |   {order_id: 1, items: ['iPhone', 'Case']},
        |   {order_id: 2, items: ['iPad']}
        | ]
102     | [
        |   {order_id: 3, items: ['AirPods', 'Pencil']}
        | ]
```

---

## 🎯 實戰案例

### **案例 1：用戶購買歷史**

```sql
WITH purchases AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, DATE('2024-01-15') AS date, 'iPhone' AS product, 30000 AS amount),
    STRUCT(101, DATE('2024-02-10'), 'AirPods', 5000),
    STRUCT(101, DATE('2024-03-05'), 'Case', 500),
    STRUCT(102, DATE('2024-01-20'), 'iPad', 25000),
    STRUCT(102, DATE('2024-02-15'), 'Pencil', 3000)
  ])
)

SELECT 
  user_id,
  COUNT(*) AS total_purchases,
  SUM(amount) AS total_spent,
  ARRAY_AGG(
    STRUCT(date, product, amount)
    ORDER BY date DESC
    LIMIT 5
  ) AS recent_purchases
FROM purchases
GROUP BY user_id;
```

**結果：**

```
user_id | total_purchases | total_spent | recent_purchases
--------|-----------------|-------------|------------------
101     | 3               | 35500       | [
        |                 |             |   {date: '2024-03-05', product: 'Case', amount: 500},
        |                 |             |   {date: '2024-02-10', product: 'AirPods', amount: 5000},
        |                 |             |   {date: '2024-01-15', product: 'iPhone', amount: 30000}
        |                 |             | ]
```

---

### **案例 2：網站瀏覽路徑分析**

```sql
WITH page_views AS (
  SELECT * FROM UNNEST([
    STRUCT('session_1' AS session_id, TIMESTAMP('2024-04-04 10:00:00') AS time, '/home' AS page),
    STRUCT('session_1', TIMESTAMP('2024-04-04 10:05:00'), '/products'),
    STRUCT('session_1', TIMESTAMP('2024-04-04 10:10:00'), '/cart'),
    STRUCT('session_1', TIMESTAMP('2024-04-04 10:15:00'), '/checkout'),
    STRUCT('session_2', TIMESTAMP('2024-04-04 11:00:00'), '/home'),
    STRUCT('session_2', TIMESTAMP('2024-04-04 11:03:00'), '/about')
  ])
)

SELECT 
  session_id,
  COUNT(*) AS page_count,
  TIMESTAMP_DIFF(MAX(time), MIN(time), MINUTE) AS session_duration_min,
  
  -- 瀏覽路徑
  ARRAY_AGG(page ORDER BY time) AS browsing_path,
  
  -- 前 3 頁
  ARRAY_AGG(page ORDER BY time LIMIT 3) AS first_3_pages
FROM page_views
GROUP BY session_id;
```

**結果：**

```
session_id | page_count | duration | browsing_path                          | first_3_pages
-----------|------------|----------|----------------------------------------|-------------------------
session_1  | 4          | 15       | ['/home', '/products', '/cart', '/checkout'] | ['/home', '/products', '/cart']
session_2  | 2          | 3        | ['/home', '/about']                    | ['/home', '/about']
```

---

### **案例 3：標籤系統**

```sql
WITH user_tags AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'VIP' AS tag),
    STRUCT(101, 'frequent_buyer'),
    STRUCT(101, 'mobile_user'),
    STRUCT(102, 'new_customer'),
    STRUCT(102, 'email_subscribed'),
    STRUCT(103, 'VIP'),
    STRUCT(103, 'corporate')
  ])
)

SELECT 
  user_id,
  ARRAY_AGG(tag ORDER BY tag) AS all_tags,
  
  -- 檢查是否為 VIP
  'VIP' IN UNNEST(ARRAY_AGG(tag)) AS is_vip,
  
  -- 標籤數量
  COUNT(*) AS tag_count
FROM user_tags
GROUP BY user_id
ORDER BY user_id;
```

**結果：**

```
user_id | all_tags                              | is_vip | tag_count
--------|---------------------------------------|--------|----------
101     | ['VIP', 'frequent_buyer', 'mobile_user'] | true   | 3
102     | ['email_subscribed', 'new_customer']  | false  | 2
103     | ['VIP', 'corporate']                  | true   | 2
```

---

### **案例 4：每月銷售產品列表**

```sql
WITH sales AS (
  SELECT * FROM UNNEST([
    STRUCT(DATE('2024-01-15') AS sale_date, 'iPhone' AS product),
    STRUCT(DATE('2024-01-20'), 'iPad'),
    STRUCT(DATE('2024-01-25'), 'AirPods'),
    STRUCT(DATE('2024-02-05'), 'iPhone'),
    STRUCT(DATE('2024-02-10'), 'MacBook'),
    STRUCT(DATE('2024-03-01'), 'iPad')
  ])
)

SELECT 
  FORMAT_DATE('%Y-%m', sale_date) AS month,
  COUNT(*) AS sale_count,
  
  -- 該月銷售的所有產品（去重）
  ARRAY_AGG(DISTINCT product ORDER BY product) AS products_sold,
  
  -- 該月銷售的所有產品（含重複）
  ARRAY_AGG(product ORDER BY sale_date) AS sales_timeline
FROM sales
GROUP BY month
ORDER BY month;
```

**結果：**

```
month   | sale_count | products_sold                | sales_timeline
--------|------------|------------------------------|---------------------------
2024-01 | 3          | ['AirPods', 'iPad', 'iPhone'] | ['iPhone', 'iPad', 'AirPods']
2024-02 | 2          | ['iPhone', 'MacBook']         | ['iPhone', 'MacBook']
2024-03 | 1          | ['iPad']                      | ['iPad']
```

---

## ⚠️ 常見錯誤

### **錯誤 1：忘記 GROUP BY**

```sql
-- ❌ 錯誤：沒有 GROUP BY
SELECT 
  user_id,
  ARRAY_AGG(product) AS products
FROM orders;
-- 報錯：必須包含 GROUP BY

-- ✅ 正確
SELECT 
  user_id,
  ARRAY_AGG(product) AS products
FROM orders
GROUP BY user_id;
```

---

### **錯誤 2：NULL 值處理**

```sql
WITH data AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'iPhone' AS product),
    STRUCT(101, NULL),
    STRUCT(101, 'iPad')
  ])
)

SELECT 
  user_id,
  ARRAY_AGG(product) AS products
FROM data
GROUP BY user_id;

-- 結果包含 NULL：['iPhone', NULL, 'iPad']

-- ✅ 過濾 NULL
SELECT 
  user_id,
  ARRAY_AGG(product) AS products
FROM data
WHERE product IS NOT NULL  -- 先過濾
GROUP BY user_id;

-- 或使用 FILTER
SELECT 
  user_id,
  ARRAY_AGG(product) FILTER (WHERE product IS NOT NULL) AS products
FROM data
GROUP BY user_id;
```

---

## 💡 與其他函數組合

### **1. ARRAY_AGG + STRING_AGG（轉成字串）**

```sql
WITH tags AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'VIP' AS tag),
    STRUCT(101, 'frequent'),
    STRUCT(101, 'mobile')
  ])
)

SELECT 
  user_id,
  ARRAY_AGG(tag) AS tags_array,
  STRING_AGG(tag, ', ') AS tags_string
FROM tags
GROUP BY user_id;
```

**結果：**

```
user_id | tags_array               | tags_string
--------|--------------------------|--------------------
101     | ['VIP', 'frequent', 'mobile'] | VIP, frequent, mobile
```

---

### **2. ARRAY_AGG + ARRAY_LENGTH（計算數量）**

```sql
WITH purchases AS (
  SELECT * FROM UNNEST([
    STRUCT(101 AS user_id, 'iPhone' AS product),
    STRUCT(101, 'iPad'),
    STRUCT(102, 'AirPods')
  ])
)

SELECT 
  user_id,
  ARRAY_AGG(product) AS products,
  ARRAY_LENGTH(ARRAY_AGG(product)) AS product_count
FROM purchases
GROUP BY user_id;
```

---

## ✅ 總結

|功能|語法|說明|
|---|---|---|
|**基本聚合**|`ARRAY_AGG(欄位)`|把多行變陣列|
|**排序**|`ARRAY_AGG(欄位 ORDER BY 排序欄位)`|指定順序|
|**限制數量**|`ARRAY_AGG(欄位 LIMIT N)`|只取前 N 個|
|**去重**|`ARRAY_AGG(DISTINCT 欄位)`|自動去重|
|**多欄位**|`ARRAY_AGG(STRUCT(欄位1, 欄位2))`|聚合結構|
|**條件過濾**|`ARRAY_AGG(欄位) FILTER (WHERE 條件)`|條件聚合|

**記住：**

- ARRAY_AGG = 多行 → 一個陣列
- 必須搭配 GROUP BY
- 可以排序、限制、去重
- 結合 STRUCT 可以聚合多個欄位

需要針對特定場景的範例嗎？😊