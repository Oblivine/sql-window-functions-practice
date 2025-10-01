# Window Functions

Think of window functions as something similar to a `GROUP BY` clause. Rather than returning just one row per group (as a regular aggregation would), they return a result **for every row**, based on the window (partition) you define.

For the stupefied (me xD): **`GROUP BY` without losing rows** — you keep multi-row visibility inside each group.

---

## Aggregation

Using the [example from Machine Learning Plus](https://www.machinelearningplus.com/sql/sql-window-functions/), we want:

| student_id | subject | score | avg_score         |
|------------|---------|-------|-------------------|
| 1          | Math    | 85    | 88                |
| 1          | English | 92    | 88                |
| 1          | Science | 87    | 88                |
| 2          | Math    | 91    | 90.66666666666667 |
| 2          | English | 88    | 90.66666666666667 |
| 2          | Science | 93    | 90.66666666666667 |
| 3          | Math    | 78    | 84.33333333333333 |
| 3          | English | 85    | 84.33333333333333 |
| 3          | Science | 90    | 84.33333333333333 |
| 4          | Math    | 92    | 89                |
| 4          | English | 86    | 89                |
| 4          | Science | 89    | 89                |
| 5          | Math    | 90    | 87.66666666666667 |
| 5          | English | 88    | 87.66666666666667 |
| 5          | Science | 85    | 87.66666666666667 |
| 6          | Math    | 93    | 88.33333333333333 |
| 6          | English | 82    | 88.33333333333333 |
| 6          | Science | 90    | 88.33333333333333 |

For someone unfamiliar with window functions, a typical approach is a subquery + join:

```sql
SELECT
  s.student_id,
  s.subject,
  s.score,
  a.avg_score
FROM students s
JOIN (
  SELECT student_id, AVG(score) AS avg_score
  FROM students
  GROUP BY student_id
) a
  ON a.student_id = s.student_id;
```

It works, but a window function is simpler:

```sql
SELECT
  *,
  AVG(score) OVER (PARTITION BY student_id) AS avg_score
FROM students;
```

If you’re feeling that “I’m incompetent” spiral — hello there! I once bragged that as a SWE with SQL I could replace a data analyst. That aged like milk once I met window functions. The “window” part (sliding scope) will show up soon.

---

## Ranking

This shows up more in analytics, but it’s good to know.

Given:

| student_id | subject | score | average_score     | total_score |
|------------|---------|-------|-------------------|-------------|
| 1          | Math    | 85    | 88                | 264         |
| 1          | English | 92    | 88                | 264         |
| 1          | Science | 87    | 88                | 264         |
| 2          | Math    | 91    | 90.66666666666667 | 272         |
| 2          | English | 88    | 90.66666666666667 | 272         |

“Who’s number one overall?”

```sql
SELECT
  *,
  RANK() OVER (ORDER BY total_score DESC) AS sRank
FROM temp;
```

Problem: each student_id gets the **same** rank across all their subjects. If you want deterministic ordering you need a tiebreaker (subsequent `ORDER BY` expressions are tie-breakers):

```sql
SELECT
  *,
  RANK() OVER (ORDER BY total_score DESC, score DESC) AS iRank
FROM temp;
```

Reference:

| Function       | Do ties share rank? | Do ranks skip after ties? | Always unique? | Example (100,100,90,80) |
|----------------|---------------------|---------------------------|----------------|--------------------------|
| `ROW_NUMBER()` | No                  | n/a                       | Yes            | 1,2,3,4                  |
| `RANK()`       | Yes                 | Yes                       | No             | 1,1,3,4                  |
| `DENSE_RANK()` | Yes                 | No                        | No             | 1,1,2,3                  |

---

## Offsets

Offsets compare one row to its neighbors within the same partition — very common in analytics.

- `LAG()` — *Laggard*: value from the **previous** row  
- `LEAD()` — *Leader*: value from the **next** row  
- `FIRST_VALUE()` — value from the **first row** of the frame  
- `LAST_VALUE()` — value from the **last row** of the frame  
- `NTH_VALUE(n)` — value from the **n-th row** of the frame  

> Note: `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE` depend on the **frame** you specify. `LAG/LEAD` don’t require frames.

**Neighbor analogy:** your left neighbor = `LAG`, right neighbor = `LEAD`. You compare your “car” (value) against theirs.

### Example: previous salary and % difference

```sql
SELECT
  *,
  LAG(salary) OVER (ORDER BY id) AS prev_salary
FROM salaries;
```

`prev_salary` is `NULL` for the first row — there’s no previous row.

Now compute the % change (use a CTE for clarity and safe division):

```sql
WITH temp_sal AS (
  SELECT
    *,
    LAG(salary) OVER (ORDER BY id) AS prev_salary
  FROM salaries
)
SELECT
  *,
  ROUND( (salary / NULLIF(prev_salary, 0) - 1) * 100, 4 ) AS perc_diff
FROM temp_sal;
```

(You can also use `LEAD()` and invert the diff if comparing against the next row.)

---

## Why some min/max looked “wrong”

We tried:

```sql
SELECT
  *,
  MIN(salary) OVER (PARTITION BY dept ORDER BY id) AS min_salary,
  MAX(salary) OVER (PARTITION BY dept ORDER BY id) AS max_salary
FROM salaries;
```

Because `ORDER BY` is present, many engines default the frame to something like
**`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`** → a **running** (cumulative) min/max, not the whole-partition min/max. Hence the mismatch.

---

## Introduction to Window Frames (the SWE analogy)

A **frame** is a **sliding scope inside a loop**.

Python-esque analogy:

```python
min_var = None
max_var = None
for row in dept_rows_sorted_by_salary:
    current_var = row.salary
    min_var = current_var if min_var is None else min(min_var, current_var)
    max_var = current_var if max_var is None else max(max_var, current_var)
    row.min_sal = min_var
    row.max_sal = max_var
```

- i=0 (id 21, salary 75): `min=max=75` → `row.min_sal=75`, `row.max_sal=75`  
- i=1 (id 22, salary 71): `min=71`, `max=75` → `row.min_sal=71`, `row.max_sal=75`

By default, a frame **does not consider the full partition** unless you say so. It behaves like a cumulative scope as you iterate.

---

## `ROWS BETWEEN` (defining the frame width)

Syntax:

```
ROWS BETWEEN <lower bound> AND <upper bound>
```

Common bounds:
- `CURRENT ROW` — the current row
- `UNBOUNDED PRECEDING` — all rows before current
- `UNBOUNDED FOLLOWING` — all rows after current
- `n PRECEDING` — n rows before current
- `n FOLLOWING` — n rows after current

> **Rule of thumb:** if you need a value for the **whole partition**, either **omit `ORDER BY`** or set  
> `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

### Whole-partition min/max (explicit frame)

```sql
SELECT
  *,
  MIN(salary) OVER (
    PARTITION BY dept
    ORDER BY id
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS min_salary,
  MAX(salary) OVER (
    PARTITION BY dept
    ORDER BY id
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS max_salary
FROM salaries;
```

> Simpler alternative (since min/max don’t need ordering if you want the whole partition):
```sql
SELECT
  *,
  MIN(salary) OVER (PARTITION BY dept) AS min_salary,
  MAX(salary) OVER (PARTITION BY dept) AS max_salary
FROM salaries;
```

---

## Bonus: other “sliding” patterns

**Running total per dept**
```sql
SELECT
  dept, id, name, salary,
  SUM(salary) OVER (
    PARTITION BY dept
    ORDER BY id
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_salary
FROM salaries
ORDER BY dept, id;
```

**3-row moving average (centered)**
```sql
SELECT
  dept, id, salary,
  AVG(salary) OVER (
    PARTITION BY dept
    ORDER BY id
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
  ) AS movavg_3
FROM salaries
ORDER BY dept, id;
```

**Top-N per group (Top 3 subjects per student)**
```sql
SELECT *
FROM (
  SELECT
    student_id, subject, score,
    ROW_NUMBER() OVER (
      PARTITION BY student_id
      ORDER BY score DESC, subject ASC
    ) AS rn
  FROM students
) t
WHERE rn <= 3
ORDER BY student_id, rn;
```

---

Wow — exploring this through my own SWE lens made the ideas *click*. Window functions are just “group-aware calculations” with a **sliding scope** that you control. End of story.

# Reference

https://www.machinelearningplus.com/sql/sql-window-functions/