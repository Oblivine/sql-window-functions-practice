# Window Functions - Exercise Answers

These are answers containing my rationales and explanations for exercise materials from Machine Learning Plus and other websites for my own use, which may contain answers different from the official ones.

# [Machine Learning Plus](https://www.machinelearningplus.com/sql/sql-window-functions-exercises/)

## Q1. Find the running cumulative total demand

**Task:**

From the `demand2` table, find the cumulative total sum for `qty`.

![](img/q1_a.png)
![](img/q1_b.png)

### Solution:
```sql
SELECT *, SUM(QTY) OVER (ORDER BY day ASC) AS cumQty FROM demand2
```

There is no need to use PARTITION BY clause, as we are not dealing with a grouped dataset. We can simply use ORDER BY on its own.

## Q2. Find the running cumulative total demand by product.

**Task:**

From the `demand` table, find the cumulative total sum for `qty` for each `product` category.

![](img/q2_a.png)
![](img/q2_b.png)

### Solution:
```sql
SELECT *, SUM(qty) OVER (PARTITION BY product ORDER BY day ASC) AS 'CUMSUM' FROM demand
```

In contrast to Q1, we will be using `PARTITION BY` as there is a clear grouping, and the desired solution seeks grouping under the `product` column. Bear in mind that we are looking for cumulative sum, and we can leverage `ORDER BY` as-is in this situation, since the framing boundary for such situations defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which is what we are after.