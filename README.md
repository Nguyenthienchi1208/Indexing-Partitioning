# PostgreSQL Query Performance Optimization

Optimize analytical SQL queries on large-scale e-commerce datasets using **Partitioning**, **Indexing**, and **Execution Plan Analysis**.

---

# Demo

<p align="center">
```sql
	EXPLAIN(ANALYZE, BUFFERS)
SELECT 
    oi.order_id, 
    o.order_date, 
    SUM(oi.quantity * oi.unit_price) AS Total_Revenue
FROM orders o 
JOIN order_items oi 
    ON o.order_id = oi.order_id 
    AND o.order_date = oi.order_date
WHERE o.order_date >= '2025-08-01' 
	AND o.order_date < '2025-09-01'
	AND oi.order_date >= '2025-08-01' 
	AND oi.order_date < '2025-09-01'
GROUP BY oi.order_id, o.order_date
ORDER BY Total_Revenue DESC;
```
</p>

---

# Overview

This project investigates how PostgreSQL query performance changes after applying:

- Range Partitioning
- Indexing
- Query Plan Optimization

The database simulates a large-scale e-commerce system containing millions of transaction records.

Performance is evaluated using

```sql
EXPLAIN (ANALYZE, BUFFERS)
```

to compare execution time, disk I/O, and query plans before and after optimization.

---

# Dataset

Generated using Python **faker**

| Table | Rows |
|-------|------:|
| orders | ~2.5M |
| order_items | ~10M |


---

# Database Design

## Main Tables

### orders

| Column | Type |
|---------|------|
| order_id | PK |
| order_date | TIMESTAMP |
| seller_id | INT |
| status | VARCHAR |
| total_amount | DECIMAL |

### order_items

| Column | Type |
|---------|------|
| order_item_id | PK |
| order_id | FK |
| product_id | INT |
| quantity | INT |
| unit_price | DECIMAL |

---

# Optimization

## Before

```
orders_backup
orders_items_backup
```

Large heap tables

↓

Sequential Scan

↓

Millions of rows scanned

---

## After

```
orders
├── orders_2025_08
├── orders_2025_09
├── orders_2025_10
└── orders_2025_11

order_items
├── order_items_2025_08
├── order_items_2025_09
├── order_items_2025_10
└── order_items_2025_11
```

### Partition Strategy

Range Partitioning

```
order_date
```

### Indexes

```
PRIMARY KEY(order_id)

INDEX(product_id)

INDEX(order_date)

INDEX(order_id, order_date)
```

---

# Benchmark Results

## Monthly Revenue

| Metric | Before | After | Improvement |
|---------|--------:|-------:|------------:|
| Execution Time | 13,060 ms | 6,479 ms | **50.4% faster** |
| Buffers Read |111,328|51,699|**53.6% fewer**|
| Temp Read |144,656|44,229|**69.4% fewer**|

---

## Orders Filtered by Seller

| Metric | Before | After | Improvement |
|---------|--------:|-------:|------------:|
| Execution Time |1548 ms|455 ms|**70.6% faster**|
| Buffers Read |111,136|36,981|**66.7% fewer**|
| Scan Target |Full Table|Partition|Partition Pruning|

---

## Filter Products

| Metric | Before | After | Improvement |
|---------|--------:|-------:|------------:|
| Execution Time |371 ms|70 ms|**81.1% faster**|
| Buffers Read |84,280|12,259|**85.5% fewer**|
| Scan Method |Seq Scan|Bitmap Index Scan|Improved|

---

## Highest Order Value

| Metric | Before | After | Improvement |
|---------|--------:|-------:|------------:|
| Execution Time |8528 ms|4850 ms|**43.1% faster**|
| Buffers Read |112,000|25,561|**77.2% fewer**|

---

## Top Selling Products

| Metric | Before | After | Improvement |
|---------|--------:|-------:|------------:|
| Execution Time |499 ms|58 ms|**88.4% faster**|
| Buffers Read |84,056|1|≈100% fewer|

---

## Orders by Seller (October)

| Metric | Before | After | Improvement |
|---------|--------:|-------:|------------:|
| Execution Time |291 ms|135 ms|**53.4% faster**|
| Buffers Read |27,687|8,955|**67.7% fewer**|

---

## Revenue per Product

| Metric | Before | After |
|---------|--------:|-------:|
| Execution Time |21.06 s|6.68 s|
| Improvement | |**3.16× faster**|

---

## Products Sold per Seller

| Metric | Before | After | Improvement |
|---------|--------:|-------:|------------:|
| Execution Time |967 ms|402 ms|**2.4× faster**|
| Rows Scanned |2,723,133|916,787|**66% fewer**|
| Buffers Read |111,711|27,779|**75% fewer**|

---

# Project Structure

```
.
├── data_generator/
├── sql/
│   ├── schema.sql
│   ├── indexes.sql
│   ├── partitions.sql
│   ├── benchmark.sql
│   └── explain_queries.sql
├── notebooks/
├── images/
└── README.md
```

---

# Key Takeaways

Applying **Partitioning** and **Indexing** significantly improved analytical query performance by:

- Reducing unnecessary table scans
- Lowering disk I/O
- Enabling partition pruning
- Improving execution plans
- Accelerating large-scale aggregation queries

Performance gains reached up to **88% faster execution** depending on query patterns.
