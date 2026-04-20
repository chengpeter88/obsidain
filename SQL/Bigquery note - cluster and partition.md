## Partitioned 

Partitioned  目的幾個 
1. 幫助查詢資料更快速，因為有index 概念不需要整個表去掃秒，可以透過index 方式指引到位子後進行查詢
2. 把資料表的大小壓縮可以把大的東西，根據index 拆分成小的多個表

--- 
## 哪樣可以做Partitioned 

需要做Partitioned 前提有一個條件為東西必須要為`interger` 數值才可以做區分
區分又可以分成幾總類型區分
1. col 有時間日其資訊，如timestamp 的`2025-02-04 17:00:00 UTC`
2. 有一個明確的起頭和終點的區間範圍數值 例如切分編號 1 ,2, ....10 號
3. 切分可以到從每年 -> 月 -> 日 -> 時

## 哪樣可以做cluster
資料需要的精細度比較高，需要幾算比較多
1. 經常需要根據特定欄位篩選
2. 多個篩選的條件組合
3. 高基數欄位（High Cardinality）
	- user_id（百萬種不同值）✅
	- email（百萬種不同值）✅
	- 不是 gender（只有 2-3 種值）❌

```
需要優化查詢性能？
│
├─ 資料量 < 1GB？
│  └─ 不需要 CLUSTER
│
├─ 有時間欄位且常按時間範圍查詢？
│  ├─ 是 → 先用 PARTITION BY date
│  │      └─ 還有其他常用篩選欄位？
│  │         └─ 是 → 加上 CLUSTER BY
│  │
│  └─ 否 → 直接用 CLUSTER BY
│
└─ 選擇 CLUSTER 欄位：
   1. 最常查詢的
   2. 高基數（很多不同值）
   3. 最多 4 個
   4. 依重要性排序

```
## PARTITION vs CLUSTER

| 特性       | PARTITION（分區）                                     | CLUSTER（聚類）                     |
| -------- | ------------------------------------------------- | ------------------------------- |
| **資料組織** | 分成獨立的區塊 （eg: 2025-01-01）                          | 在區塊內排序 （eg : ID: 104 ,103 一團一團） |
| **欄位限制** | 只能 1 個欄位（DATE/TIMESTAMP/INT）                      | 最多 4 個欄位                        |
| **欄位類型** | 只能時間或整數                                           | 任何類型 ✅                          |
| **查詢剪枝** | 嚴格（完全跳過分區），條件之外的區分不會看（eg : 找2025-01-01 只會看個之下的東西） | 彈性（跳過不相關的塊）                     |
| **費用節省** | 高（直接跳過整個分區）                                       | 中高（跳過部分塊）                       |
| **適合情況** | 有明確時間範圍查詢                                         | 多條件查詢                           |
|          |                                                   |                                 |
## 圖片了解兩個差異

![[截圖 2026-04-06 上午11.15.43.png|697]]


## 如何建立partition and cluster 
`create table` 後面partition or cluster 補進去

```
CREATE TABLE `project.dataset.sales_clustered`
PARTITION BY DATE(sale_date)
CLUSTER BY user_id, product_category, country
AS
SELECT 
  user_id,
  product_category,
  country,
  sale_date,
  amount
FROM `project.dataset.sales_raw`;


-- ⚠️ 注意：這會重新寫入整張表
ALTER TABLE `project.dataset.sales`
SET OPTIONS(
  clustering_fields = 'user_id,product_category,country'
);

```


## check table 是否有cluster 
`infomation_schema`

```

-- 方法 1：查看表結構
SELECT 
  table_name,
  clustering_fields
FROM `project.dataset.INFORMATION_SCHEMA.TABLES`
WHERE table_name = 'your_table';

-- 方法 2：查看詳細資訊
SELECT 
  table_catalog,
  table_schema,
  table_name,
  clustering_fields,
  partitioning_type,
  partition_expiration_days
FROM `project.dataset.INFORMATION_SCHEMA.TABLES`
WHERE table_type = 'BASE TABLE';
```


---
