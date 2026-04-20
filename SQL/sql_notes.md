# SQL 語法快速參考筆記

---

## 目錄
- [[#DML 資料操作]]
- [[#DDL 物件建立]]
- [[#View 視圖]]
- [[#暫存 / CTAS]]
- [[#Procedure]]
- [[#比較總覽]]

---

## DML 資料操作

> DML（Data Manipulation Language）— 操作資料內容，不改架構

### INSERT — 新增資料

```sql
INSERT INTO table_name (col1, col2, col3)
VALUES (val1, val2, val3);
```

用於新增一筆或多筆資料進已存在的表。

---

### UPDATE — 更新資料

```sql
UPDATE table_name
SET column1 = value1, column2 = value2
WHERE condition;
```

> ⚠️ `WHERE` 很重要，忘了加會更新整張表。

---

### MERGE — 合併（Upsert）

```sql
MERGE INTO target_table
USING source_table
ON merge_condition
WHEN MATCHED THEN
  UPDATE SET col1 = value1
WHEN NOT MATCHED THEN
  INSERT (col1, col2) VALUES (val1, val2);
```

資料存在就更新、不存在就新增，常用於 ETL pipeline。

---

### ALTER — 改結構

```sql
ALTER TABLE table_name ADD COLUMN new_col INT64;
ALTER TABLE table_name DROP COLUMN old_col;
ALTER TABLE table_name RENAME TO new_name;
```

> 改結構，不改資料內容。

---

## DDL 物件建立

> DDL（Data Definition Language）— 定義結構和物件，不動資料

### CREATE 可建立的物件（BigQuery）

| 類型 | 說明 |
|------|------|
| `TABLE` | 實體表，有真實資料 |
| `VIEW` | 虛擬視圖，每次重跑 |
| `MATERIALIZED VIEW` | 快取視圖，定時更新 |
| `EXTERNAL TABLE` | 外連 GCS / Drive，不落地 |
| `FUNCTION (UDF)` | 自訂函數，封裝邏輯 |
| `PROCEDURE` | 多步驟流程，可 ETL |
| `SNAPSHOT TABLE` | 時間點快照，audit 用 |
| `TEMP TABLE` | 暫存，session 結束消失 |

---

### 常用修飾語法

#### CREATE OR REPLACE

```sql
CREATE OR REPLACE VIEW dataset.view_name AS
SELECT ...;
```

更新邏輯時不用先 DROP，直接覆蓋。

#### CREATE IF NOT EXISTS

```sql
CREATE TABLE IF NOT EXISTS dataset.table_name (...);
```

用於 pipeline / automation，避免重複建立報錯。

#### OPTIONS（設定屬性）

```sql
CREATE TABLE dataset.table_name
OPTIONS (expiration_timestamp = TIMESTAMP("2026-06-01"))
AS SELECT ...;
```

---

### EXTERNAL TABLE — 不落地查詢

```sql
CREATE EXTERNAL TABLE dataset.ext_table
OPTIONS (
  format = 'CSV',
  uris   = ['gs://bucket/file.csv']
);
```

---

### FUNCTION — 封裝可重用邏輯

```sql
CREATE FUNCTION dataset.double_it(x INT64)
RETURNS INT64
AS (x * 2);
```

把常用計算邏輯包成函數，在 `SELECT` 裡直接呼叫。

---

## View 視圖

### View 是什麼

View 是**虛擬的表**，本身不儲存資料。每次查詢 view，SQL 引擎會先展開 view 定義，再去查底層的實體表（兩階段溝通）。

- ✅ 邏輯永久保存
- ✅ 資料不落地
- ⚠️ 每次查詢都重跑

---

### 為什麼用 View

- 封裝複雜 JOIN 邏輯，不用每次重寫
- 抽象分層：`DB → Table → View`（業務層用 view，不直接碰 table）
- 限制使用者只能 READ，沒有 WRITE 權限
- Data Warehouse 架構下，建立 Data Mart 小子集的最佳作法

---

### 建立語法範例

```sql
CREATE OR REPLACE VIEW Sales.order_detail AS (
  SELECT
    o.orderid,
    o.orderdate,
    p.product,
    p.category,
    COALESCE(c.firstname,'') + ' ' + COALESCE(c.lastname,'') AS customer_name,
    c.country AS customer_country,
    o.sales,
    o.quantity
  FROM sales.orders o
  LEFT JOIN sales.products p ON p.productid = o.productid
  LEFT JOIN sales.customers c ON c.customerid = o.customerid
);
```

---

### View vs CTE — 差在哪？

| | CTE（`WITH ...`） | View |
|---|---|---|
| 生命週期 | 單次 query | 永久存在 DB |
| 跨 query 重用 | ❌ | ✅ |
| 需要 CREATE | ❌ 即寫即用 | ✅ 需命名 |
| 適合情境 | 臨時複雜邏輯 | 多人 / 多次共用邏輯 |

---

## 暫存 / CTAS

### CTAS — Create Table As Select

```sql
-- 標準語法（BigQuery / Postgres）
CREATE TABLE new_table AS
SELECT * FROM source_table WHERE ...;

-- SQL Server 獨有語法
SELECT *
INTO new_table
FROM source_table;
```

把查詢結果「拍照」成一張新的實體表。原始表更新後，CTAS 表**不會**自動跟著變。

- 資料靜止快照
- ⚠️ 不自動跟原始表同步

---

### TEMP TABLE — 暫存表

```sql
-- BigQuery
CREATE TEMP TABLE tmp_check AS
SELECT * FROM source WHERE condition;

-- SQL Server（# 前綴）
SELECT * INTO #tmp_check FROM source;
```

Session 結束（重新開連線）就消失。常用於：先拉出資料做驗證（去重、補空值），確認後再寫回正式表。

- ⚠️ session 結束即消失
- 適合中間驗證用

---

### SNAPSHOT TABLE（BigQuery）

```sql
CREATE SNAPSHOT TABLE dataset.snap_20260415
CLONE dataset.source_table
FOR SYSTEM_TIME AS OF TIMESTAMP("2026-04-15 00:00:00");
```

拍下某個時間點的資料版本，用於 audit 或版本比對，報表資料穩定不受後續更新影響。

---

## Procedure

### PROCEDURE 是什麼

把多個 SQL 步驟包在一起，像是一個可重複呼叫的「程式」。

與 FUNCTION 不同的是，Procedure **不回傳值**，但可以執行 `INSERT / UPDATE / DELETE / MERGE`。

- 適合多步驟 ETL
- ⚠️ 不 RETURN 值（要回傳值請用 FUNCTION）
- ✅ 可操作資料

---

### 建立語法（Postgres）

```sql
CREATE OR REPLACE PROCEDURE schema.sync_orders()
LANGUAGE plpgsql
AS $$
BEGIN
  INSERT INTO orders_archive
  SELECT * FROM orders WHERE status = 'done';

  DELETE FROM orders WHERE status = 'done';
END;
$$;

-- 呼叫方式：
CALL schema.sync_orders();
```

---

### 建立語法（BigQuery）

```sql
CREATE OR REPLACE PROCEDURE dataset.sync_data()
BEGIN
  INSERT INTO dataset.table1
  SELECT * FROM dataset.table2
  WHERE created_at > DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY);
END;

-- 呼叫方式：
CALL dataset.sync_data();
```

---

## 比較總覽

### 五種物件完整比較

| | Subquery / CTE | View | Temp Table | CTAS | Snapshot |
|---|---|---|---|---|---|
| **速度** | 最快 | 取決於底層表 | 中等 | 最慢（寫磁碟） | 快（clone） |
| **生命週期** | 單次 query | 永久（邏輯） | session 內 | 永久 | 永久（直到刪除） |
| **資料是否最新** | ✅ 每次重跑 | ✅ 每次重跑 | ❌ 已固定 | ❌ 快照 | ❌ 時間點快照 |
| **可存放邏輯** | ❌ | ✅ 多 query 共用 | session 內可重用 | 有實體表 | 有，但唯讀 |
| **適合情境** | 臨時整理邏輯 | 業務邏輯封裝 / Data Mart | 驗證、中間計算 | 穩定報表、歷史備份 | audit、版本比對 |

---

### DDL vs DML — 快速記憶

| | DDL | DML |
|---|---|---|
| 全名 | Data Definition Language | Data Manipulation Language |
| 操作對象 | 結構 / 物件 | 資料內容 |
| 常用指令 | `CREATE` / `ALTER` / `DROP` | `INSERT` / `UPDATE` / `DELETE` / `MERGE` |
| 一句話 | 不動資料，動架構 | 不動架構，動資料 |
