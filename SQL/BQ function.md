---
tags:
  - SQL
  - BigQuery
  - 函數
---

# 📦 BigQuery Aggregate Functions

> 針對「一組資料」進行計算，搭配 `GROUP BY` 使用。

---

## ANY_VALUE

### 一句話說明

**ANY_VALUE = 從這組資料中，任意回傳一筆值**

不保證順序，但搭配 `HAVING MAX/MIN` 可以抓最大/最小時對應的值。

---

### 基本語法

```sql
ANY_VALUE(expression [HAVING { MAX | MIN } having_expression])
[OVER over_clause]
```

---

### 使用場景 1：搭配 HAVING MAX 取最高銷量的水果

```sql
WITH Store AS (
  SELECT 20 AS sold, 'apples'  AS fruit UNION ALL
  SELECT 30,         'pears'            UNION ALL
  SELECT 30,         'bananas'          UNION ALL
  SELECT 10,         'oranges'
)
SELECT ANY_VALUE(fruit HAVING MAX sold) AS highest_selling_fruit
FROM Store;

-- 結果：pears（或 bananas，sold 同為 30，任意回傳其中一個）
```

---

### 使用場景 2：Window Function（視窗函數）

```sql
SELECT
  fruit,
  ANY_VALUE(fruit) OVER (
    ORDER BY LENGTH(fruit)
    ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
  ) AS any_value
FROM UNNEST(['apple', 'banana', 'pear']) AS fruit;

/*
 fruit  | any_value
--------+-----------
 pear   | pear       ← 只有自己
 apple  | pear       ← 前一行是 pear
 banana | apple      ← 前一行是 apple
*/
```

---

### 使用場景 3：搭配 GROUP BY 取任一值

```sql
-- 場景：每個訂單有多筆商品，只想取訂單日期（同一訂單日期相同）
SELECT
  order_id,
  ANY_VALUE(order_date) AS order_date,   -- 不用 MAX/MIN，因為都一樣
  SUM(amount) AS total_amount
FROM order_items
GROUP BY order_id;
```

---

## SQL 查詢標準架構（BigQuery）

```sql
WITH
  cte_name AS (
    SELECT ...
    FROM table
  )

SELECT
  column1,
  SUM(column2)       AS total,
  ANY_VALUE(column3) AS any_col3
FROM cte_name
WHERE condition
GROUP BY 1
HAVING SUM(column2) > 100
ORDER BY total DESC
LIMIT 10;
```

> 📌 **執行順序：** `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`

---

## 常用 Aggregate Functions 速查

| 函數 | 說明 | 範例 |
|------|------|------|
| `COUNT(*)` | 計算行數 | `COUNT(*)` |
| `COUNT(DISTINCT col)` | 計算不重複值數量 | `COUNT(DISTINCT user_id)` |
| `SUM(col)` | 加總 | `SUM(amount)` |
| `AVG(col)` | 平均 | `AVG(score)` |
| `MAX(col)` | 最大值 | `MAX(date)` |
| `MIN(col)` | 最小值 | `MIN(price)` |
| `ANY_VALUE(col)` | 任意一筆值 | `ANY_VALUE(name)` |
| `ARRAY_AGG(col)` | 多行變陣列 | 見 [[ARRAY_AGG]] |
| `STRING_AGG(col, sep)` | 多行合併成字串 | `STRING_AGG(tag, ', ')` |
