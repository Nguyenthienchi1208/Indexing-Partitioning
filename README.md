
````md
# PostgreSQL Query Performance Optimization

## About

This project focuses on improving PostgreSQL query performance by applying **partitioning** and **indexing** techniques.

### Problem

When working with large-scale transactional data, queries that scan millions of rows can become slow and consume significant I/O resources.

This project simulates an e-commerce database with millions of orders and order items to evaluate:

- How partitioning reduces unnecessary data scanning.
- How indexing improves selective query performance.
- How query execution plans change after optimization.
- How database design affects analytical query performance.

The main optimization techniques:

- Table Partitioning (Range Partitioning by `order_date`)
- Indexing (`product_id`, `order_id`, ...)
- Query plan analysis using:

```sql
EXPLAIN (ANALYZE, BUFFERS)
````

---

# Use

## 1. Generate and Load Data

Using Python `faker` library to generate realistic e-commerce transaction data and load into PostgreSQL.

Dataset size:

* `orders`: ~2.5M rows
* `order_items`: ~7.5M - 10M rows

Relationship:

```
1 order
 |
 └── 2-4 order_items
```

Data validation:

* # `orders.total_amount`

  `SUM(order_items.subtotal)`

---

## 2. Database Schema

### orders

| Column       | Type               | Description             |
| ------------ | ------------------ | ----------------------- |
| order_id     | SERIAL PRIMARY KEY | Unique order identifier |
| order_date   | TIMESTAMP          | Order creation time     |
| seller_id    | INT                | Seller reference        |
| status       | VARCHAR(20)        | Order status            |
| total_amount | DECIMAL(12,2)      | Total order value       |
| created_at   | TIMESTAMP          | Record creation time    |

### order_items

| Column        | Type          | Description            |
| ------------- | ------------- | ---------------------- |
| order_item_id | BIGSERIAL     | Unique item identifier |
| order_id      | INT           | Related order          |
| product_id    | INT           | Purchased product      |
| order_date    | TIMESTAMP     | Order date             |
| quantity      | INT           | Purchased quantity     |
| unit_price    | NUMERIC(10,2) | Price per unit         |
| subtotal      | NUMERIC(12,2) | quantity × unit_price  |
| created_at    | TIMESTAMP     | Record creation time   |

---

## 3. Partition Design

Original tables:

```
orders_backup
orders_items_backup
```

Optimized tables:

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

Partition key:

```sql
order_date
```

Partition strategy:

```
RANGE PARTITIONING
```

---

# Result

## 1. Monthly Revenue Analysis

### Optimization Impact

| Metric         |    Before |    After |           Improvement |
| -------------- | --------: | -------: | --------------------: |
| Execution Time | 13,060 ms | 6,479 ms |      **50.4% faster** |
| Buffers Read   |   111,328 |   51,699 | **53.6% fewer reads** |
| Temp Read      |   144,656 |   44,229 |   **69.4% reduction** |
| Temp Written   |   145,119 |   44,343 |   **69.4% reduction** |

---

## 2. Filter Orders by Seller and Date

| Metric                 |     Before |            After |           Improvement |
| ---------------------- | ---------: | ---------------: | --------------------: |
| Execution Time         |   1,548 ms |           455 ms |      **70.6% faster** |
| Buffers Read           |    111,136 |           36,981 | **66.7% fewer reads** |
| Rows Removed by Filter |    994,459 |          331,981 |   **66.6% reduction** |
| Scan Target            | Full Table | August Partition |     Partition Pruning |

---

## 3. Filter Order Items by Product

| Metric         |   Before |                         After |           Improvement |
| -------------- | -------: | ----------------------------: | --------------------: |
| Execution Time |   371 ms |                         70 ms |      **81.1% faster** |
| Buffers Read   |   84,280 |                        12,259 | **85.5% fewer reads** |
| Returned Rows  |   15,871 |                        15,871 |                  Same |
| Scan Method    | Seq Scan | Bitmap Index Scan + Heap Scan |              Improved |

Index used:

```
order_items_2025_08_product_id_idx
```

---

## 4. Find Orders with Highest Total Amount

| Metric         |    Before |              After |               Improvement |
| -------------- | --------: | -----------------: | ------------------------: |
| Execution Time |  8,528 ms |           4,850 ms |          **43.1% faster** |
| Buffers Read   |   112,000 |             25,561 |     **77.2% fewer reads** |
| Join Strategy  | Hash Join | Parallel Hash Join | Better parallel execution |

---

## 5. Top Products by Quantity Sold

| Metric         |            Before |                         After |              Improvement |
| -------------- | ----------------: | ----------------------------: | -----------------------: |
| Execution Time |            499 ms |                         58 ms | **88.4% faster (~8.6x)** |
| Buffers Read   |            84,056 |                             1 |    **~100% fewer reads** |
| Scan Method    | Parallel Seq Scan | Bitmap Heap Scan + Index Scan |                 Improved |

Index:

```
order_items_2025_08_product_id_idx
```

---

## 6. Orders by Seller in October

| Metric         |     Before |             After |           Improvement |
| -------------- | ---------: | ----------------: | --------------------: |
| Execution Time |     291 ms |            135 ms |      **53.4% faster** |
| Buffers Read   |     27,687 |             8,955 | **67.7% fewer reads** |
| Rows Returned  |     21,118 |            21,118 |                  Same |
| Data Access    | Full Table | October Partition |     Partition Pruning |

---

## 7. Revenue per Product per Month

| Metric             | Before             | After              | Improvement      |
| ------------------ | ------------------ | ------------------ | ---------------- |
| Execution Time     | 21.06s             | 6.68s              | **3.16x faster** |
| Join Strategy      | Nested Loop        | Parallel Hash Join | Improved         |
| Orders Access      | 8.1M Index Lookups | Parallel Seq Scan  | Improved         |
| Order Items Access | Parallel Append    | Parallel Seq Scan  | Improved         |

---

## 8. Products Sold per Seller

| Metric         |             Before |              After |        Improvement |
| -------------- | -----------------: | -----------------: | -----------------: |
| Execution Time |             967 ms |             402 ms |    **2.4x faster** |
| Rows Scanned   |          2,723,133 |            916,787 | **66% fewer rows** |
| Buffers Read   |            111,711 |             27,779 |  **75% reduction** |
| Data Access    |    Full Table Scan |  Partition Pruning |           Improved |
| Join Strategy  | Parallel Hash Join | Parallel Hash Join |               Same |

---

# Conclusion

By applying **partitioning and indexing**, the database achieved:

* Faster query execution time.
* Reduced disk I/O.
* Less unnecessary data scanning.
* More efficient query execution plans.

The biggest performance improvements came from:

1. **Partition pruning** for time-based queries.
2. **Indexing** for highly selective filters.
3. **Better execution plans** after reducing scanned data volume.
