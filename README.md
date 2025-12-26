# Optimizing Performance for Laravel's Upsert Feature

This article shares experiences in improving performance when handling large datasets using the `upsert` feature in Laravel (v8+).

## 1. How Laravel Upsert Works
Laravel executes the `upsert` command using the following SQL statement:
```sql
INSERT INTO ... ON DUPLICATE KEY UPDATE ...
```
**Advantage:** Combines `INSERT` and `UPDATE` into a single call based on the primary key or unique key, making the code much cleaner.

---

## 2. Issues & Temporary Solutions

### Common Problem
When processing large amounts of data (e.g., 10,000 records), if the execution is not well-controlled, the number of queries can skyrocket, severely affecting system performance.

### "Batching" Solution (Temporary)
Instead of using the default `upsert` if it feels slow, we can split it into two large queries:
1. **Batch Insert:** Filter IDs that don't exist in the DB and group them to insert at once.
2. **Batch Update:** Use the `UPDATE...CASE...WHEN` structure to update existing records in bulk.

**Update Logic Comparison:**
*   **Method 1 (Multiple single queries):** `UPDATE table SET field = val WHERE id = x`. The system must check conditions $N$ times for $N$ records ($N^2$ checks).
*   **Method 2 (One grouped query):** Using `CASE WHEN`. The system only needs to traverse the table once ($N$ checks).

---

## 3. Performance Analysis & Reality (New Update)

Through actual testing and comparison, we've found some surprising results:

> [!IMPORTANT]
> **Benchmark Results:**
> 1. Laravel's default `upsert` runs **10 times faster** than using `UPDATE CASE WHEN`.
> 2. Default `upsert` runs **twice as fast** as using "Dynamic Temporary Table" (Joining with a VALUES block).

### When should you use Laravel Upsert?
*   **Small datasets (5-10 records):** Performance difference is negligible; use the built-in feature for convenience.
*   **Large datasets (Bulk Import):** Laravel's `upsert` performs exceptionally well because it supports bulk insert/update in a single query (if passed an array of data).
*   **Multi-key support:** Works well with both Primary Keys and composite Unique Keys.

---

## 4. Notes & Additional Resources

### Technical Notes
*   Currently, the support traits have only been stably tested on **MySQL**.
*   Custom solutions (`SqlBulkUpdatable`, `wantsUpsertQuery`) currently support optimization for a single field.

### Supporting Tools
If you're interested in deeper optimization for Batch Updates, I've packaged a library here:
👉 **[quanggpv/fast-batch-update](https://github.com/quanggpv/fast-batch-update)**

---
*Thanks for reading, hope this share helps your project!*
