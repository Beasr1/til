# 8. The SQL interviews ask

## The problem

Interview SQL questions are a small, recurring set: the *n*th highest salary, the top
earner per department, employees who earn more than their manager, duplicates. Each is
easy to answer in a way that works on the interviewer's sample data and is wrong on
real data: ties, NULLs, empty results. The questions are really probing three things:
whether you know the order SQL evaluates a query in, whether you know what NULL does,
and whether you reach for window functions. This chapter teaches those three and then
works the classics.

Assumes [file 06](06-relational-model-and-normalisation.md) for keys and joins. Every
query and output below was run on PostgreSQL 17.11; the output is pasted, not typed.
SQL is the same language across the four languages this course uses, so there are no
per-language versions here.

## 8.1 The data

```sql
CREATE TABLE department (id int PRIMARY KEY, name text NOT NULL);
CREATE TABLE employee (
  id         int PRIMARY KEY,
  name       text NOT NULL,
  salary     int,
  dept_id    int REFERENCES department(id),
  manager_id int REFERENCES employee(id)
);
INSERT INTO department VALUES (1,'Eng'),(2,'Sales'),(3,'Legal');
INSERT INTO employee VALUES
  (1,'Asha', 200,1,NULL),(2,'Ravi', 150,1,4),(3,'Meera',150,1,1),
  (4,'Kabir',120,2,1),(5,'Zoya', 100,2,4),(6,'Dev', NULL,NULL,1);
```

It's small on purpose and has the three things that break naive answers: a **tie**
(Ravi and Meera both earn 150), a **NULL** salary and department (Dev), and an **empty
group** (Legal has nobody).

## 8.2 The order SQL actually runs in

You write `SELECT … FROM … WHERE … GROUP BY … HAVING … ORDER BY … LIMIT`. The database
evaluates it in a different order. The PostgreSQL `SELECT` reference describes the
processing steps ([sql-select](https://www.postgresql.org/docs/current/sql-select.html));
logically:

```mermaid
flowchart LR
    F["1 FROM / JOIN<br/>build the rows"] --> W["2 WHERE<br/>filter rows"]
    W --> G["3 GROUP BY<br/>form groups"]
    G --> H["4 HAVING<br/>filter groups"]
    H --> S["5 SELECT<br/>compute columns<br/>+ window functions"]
    S --> D["6 DISTINCT"]
    D --> O["7 ORDER BY"]
    O --> L["8 LIMIT / OFFSET"]
    style W fill:#264653,stroke:#8ecae6,color:#fff
    style S fill:#2d6a4f,stroke:#95d5b2,color:#fff
```

(The planner may physically do things in another order, but only when the result is
identical. This is the order the *meaning* is defined in.)

Almost every "why is this an error?" question follows from that picture:

```
SELECT dept_id, count(*) FROM employee WHERE count(*) > 1 GROUP BY dept_id;
ERROR:  aggregate functions are not allowed in WHERE
```

`WHERE` runs at step 2, before any groups exist, so there's nothing to count yet. Filtering
on an aggregate is `HAVING`'s job, at step 4.

```
SELECT name FROM employee WHERE rank() OVER (ORDER BY salary DESC) = 1;
ERROR:  window functions are not allowed in WHERE
```

Window functions are computed at step 5. The PostgreSQL tutorial says it directly:
"Window functions are permitted only in the SELECT list and the ORDER BY clause of the
query" ([tutorial §3.5](https://www.postgresql.org/docs/current/tutorial-window.html)).
To filter on one, compute it in a subquery or CTE and filter outside.

```
SELECT dept_id, name FROM employee GROUP BY dept_id;
ERROR:  column "employee.name" must appear in the GROUP BY clause or be used in an aggregate function
```

After step 3 each output row is a *group*. A group of three Eng employees has no single
`name`, so the database refuses to pick one. (MySQL has historically been more permissive
here, depending on its `ONLY_FULL_GROUP_BY` mode; PostgreSQL never is.)

It also explains why a column alias defined in `SELECT` can be used in `ORDER BY` (step
7 comes after 5) but not in `WHERE` (step 2 comes before).

## 8.3 NULL: the third truth value

NULL means "unknown", and SQL's logic has three values: true, false and unknown. Any
comparison with NULL, including `NULL = NULL`, is unknown, and `WHERE` keeps only rows
where the condition is *true*. Most NULL bugs are one of these:

| Behaviour | Consequence |
|---|---|
| `salary = NULL` is never true | Use `salary IS NULL` |
| Aggregates skip NULLs: `count(salary)`, `sum`, `avg` ignore them. `count(*)` counts rows | `avg(salary)` is 144.00 (720 ÷ 5), while `sum(salary)/count(*)` is 120 (720 ÷ 6). Decide which you mean |
| `ORDER BY salary DESC` puts NULLs **first** in PostgreSQL | "null values sort as if larger than any non-null value; that is, NULLS FIRST is the default for DESC order, and NULLS LAST otherwise" ([§7.5](https://www.postgresql.org/docs/current/queries-order.html)). Other databases differ |
| `NOT IN (subquery)` returns nothing if the subquery yields a NULL | See below |

Measured on our data:

```
 count_star | count_salary | sum |  avg   | sum_over_count_star
------------+--------------+-----+--------+---------------------
          6 |            5 | 720 | 144.00 |                 120
```

### The NOT IN trap

"Departments with no employees" looks like this:

```sql
SELECT name FROM department WHERE id NOT IN (SELECT dept_id FROM employee);
-- (0 rows)
```

Zero rows, though Legal clearly has nobody. Dev's `dept_id` is NULL, so for Legal the
test becomes `3 NOT IN (1, 1, 1, 2, 2, NULL)`, which means `3 <> 1 AND … AND 3 <> NULL`.
The last term is unknown, so the whole thing is unknown, never true. One NULL in the
subquery empties the result. `NOT EXISTS` asks a different question, "is there any
matching row?", which NULLs can't confuse:

```sql
SELECT name FROM department d
WHERE NOT EXISTS (SELECT 1 FROM employee e WHERE e.dept_id = d.id);
-- Legal
```

> ⚠️ Prefer `NOT EXISTS` (or a `LEFT JOIN … WHERE right.id IS NULL`) to `NOT IN` with a
> subquery, always. The `NOT IN` version is correct right up until someone inserts a row
> with a NULL in that column, then silently returns nothing.

## 8.4 Window functions

`GROUP BY` collapses each group into one row. Often you want a per-group calculation
*without* losing the rows: each employee's rank in their department, each salary's share
of its department's total. That's a **window function**: "a calculation across a set of
table rows that are somehow related to the current row" ([tutorial §3.5](https://www.postgresql.org/docs/current/tutorial-window.html)).

```sql
function(...) OVER (PARTITION BY <groups> ORDER BY <order within group>)
```

`PARTITION BY` is like `GROUP BY` that doesn't collapse. `ORDER BY` inside `OVER` orders
rows within each partition, which ranking and running totals need.

The three ranking functions differ only in how they treat ties, and that's the whole
point of most interview questions. Definitions from the docs
([§9.22](https://www.postgresql.org/docs/current/functions-window.html)):

| Function | Definition | On a tie |
|---|---|---|
| `row_number()` | "the number of the current row within its partition, counting from 1" | Arbitrary but distinct numbers |
| `rank()` | "the rank of the current row, with gaps" | Same rank, then skips |
| `dense_rank()` | "the rank of the current row, without gaps; this function effectively counts peer groups" | Same rank, no skip |

```
 name  | salary | row_number | rank | dense_rank
-------+--------+------------+------+------------
 Asha  |    200 |          1 |    1 |          1
 Ravi  |    150 |          2 |    2 |          2
 Meera |    150 |          3 |    2 |          2
 Kabir |    120 |          4 |    4 |          3
 Zoya  |    100 |          5 |    5 |          4
```

Is Kabir third? By `rank` he's fourth (two people are ahead at 150, and one at 200). By
`dense_rank` he has the third-highest *salary*. By `row_number` he's fourth, but Ravi and
Meera's order is arbitrary. Which one is right depends on the question, and saying so out
loud is most of the answer.

Other window functions that come up: `lag()`/`lead()` (the previous/next row's value),
`sum() OVER (ORDER BY …)` (running total), and any aggregate over a partition:

```sql
SELECT name, salary,
       sum(salary) OVER (ORDER BY id)          AS running_total,
       salary - lag(salary) OVER (ORDER BY id) AS change_from_prev
FROM employee WHERE salary IS NOT NULL ORDER BY id;

 name  | salary | running_total | change_from_prev
-------+--------+---------------+------------------
 Asha  |    200 |           200 |
 Ravi  |    150 |           350 |              -50
 Meera |    150 |           500 |                0
 Kabir |    120 |           620 |              -30
 Zoya  |    100 |           720 |              -20

SELECT name, dept_id, salary,
       round(100.0 * salary / sum(salary) OVER (PARTITION BY dept_id), 1) AS pct_of_dept
FROM employee WHERE salary IS NOT NULL ORDER BY dept_id, salary DESC;

 name  | dept_id | salary | pct_of_dept
-------+---------+--------+-------------
 Asha  |       1 |    200 |        40.0
 Ravi  |       1 |    150 |        30.0
 Meera |       1 |    150 |        30.0
 Kabir |       2 |    120 |        54.5
 Zoya  |       2 |    100 |        45.5
```

## 8.5 The nth highest salary, four ways

The most-asked SQL question. Before writing anything, ask the interviewer two things:
**do ties count once or separately** (is the second-highest 150, or is it whoever's in
position 2?) and **what should happen if there's no *n*th** (no row, or a NULL)?

**The naive answer, which is wrong here:**

```sql
SELECT salary FROM employee ORDER BY salary DESC LIMIT 1 OFFSET 1;
-- 200
```

It returns 200, the *highest*. Dev's NULL sorts first under `DESC` (§8.3), so position 2
is Asha. Even without the NULL, it ignores ties: with two people on 200, it would call 200
the second-highest.

**1. DISTINCT plus OFFSET.** Ties count once, NULLs excluded:

```sql
SELECT DISTINCT salary FROM employee
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 1 OFFSET 1;            -- OFFSET n-1
-- 150
```

**2. Max below the max.** Only works for *n* = 2, but needs no window functions:

```sql
SELECT max(salary) FROM employee
WHERE salary < (SELECT max(salary) FROM employee);
-- 150
```

**3. `dense_rank()`.** General *n*, ties counted once, and it returns the *people* too:

```sql
SELECT name, salary FROM (
  SELECT name, salary, dense_rank() OVER (ORDER BY salary DESC) AS r
  FROM employee WHERE salary IS NOT NULL
) ranked
WHERE r = 3;
-- Kabir | 120
```

Swap in `rank()` or `row_number()` if the question means positions rather than distinct
salaries (§8.4).

**4. Correlated subquery.** The pre-window-function classic: a salary is the *n*th highest
if exactly *n − 1* distinct salaries are greater than it.

```sql
SELECT DISTINCT salary FROM employee e1
WHERE 1 = (SELECT count(DISTINCT e2.salary)      -- n - 1
           FROM employee e2 WHERE e2.salary > e1.salary);
-- 150
```

It's correct but runs the inner query per row, so it's quadratic without help from an
index. Know it because older interviewers expect it; prefer `dense_rank`.

**When there's no nth.** Wrapping the query as a scalar subquery turns "no row" into a
NULL, which some online-judge versions of the question ask for:

```sql
SELECT (SELECT DISTINCT salary FROM employee WHERE salary IS NOT NULL
        ORDER BY salary DESC LIMIT 1 OFFSET 9) AS tenth_highest;
-- tenth_highest = NULL (one row)
```

> **Teacher's aside.** Interviewers who ask this aren't checking whether you've memorised
> a query. They're checking whether you notice that "the second-highest salary" is
> ambiguous. Ask about ties and the empty case before typing, and you've shown the thing
> they wanted to see. Then write the `dense_rank` version, and mention `rank` and
> `row_number` as the alternatives for the other readings.

## 8.6 Self joins and hierarchies

A table that refers to itself (`manager_id → employee.id`) is joined to itself under two
aliases, one per role.

**Each employee with their manager.** `LEFT JOIN`, so Asha (no manager) stays:

```sql
SELECT e.name AS employee, m.name AS manager
FROM employee e LEFT JOIN employee m ON m.id = e.manager_id
ORDER BY e.id;

 employee | manager
----------+---------
 Asha     |
 Ravi     | Kabir
 Meera    | Asha
 Kabir    | Asha
 Zoya     | Kabir
 Dev      | Asha
```

**Employees who earn more than their manager:**

```sql
SELECT e.name, e.salary, m.name AS manager, m.salary AS manager_salary
FROM employee e JOIN employee m ON m.id = e.manager_id
WHERE e.salary > m.salary;

 name | salary | manager | manager_salary
------+--------+---------+----------------
 Ravi |    150 | Kabir   |            120
```

**The whole chain of command** needs a recursive CTE: a starting row, plus a step that
joins from the rows found so far to their managers, repeated until nothing new appears.

```sql
WITH RECURSIVE chain AS (
  SELECT id, name, manager_id, 0 AS depth FROM employee WHERE name = 'Zoya'
  UNION ALL
  SELECT m.id, m.name, m.manager_id, c.depth + 1
  FROM employee m JOIN chain c ON m.id = c.manager_id
)
SELECT name, depth FROM chain;

 name  | depth
-------+-------
 Zoya  |     0
 Kabir |     1
 Asha  |     2
```

## 8.7 Grouping, top-N per group, and duplicates

**Headcount and average salary per department, only departments with at least two
people.** `HAVING` filters groups; note `count(e.id)`, not `count(*)`:

```sql
SELECT d.name, count(e.id) AS headcount, round(avg(e.salary), 2) AS avg_salary
FROM department d LEFT JOIN employee e ON e.dept_id = d.id
GROUP BY d.name
HAVING count(e.id) >= 2
ORDER BY d.name;

 name  | headcount | avg_salary
-------+-----------+------------
 Eng   |         3 |     166.67
 Sales |         2 |     110.00
```

Why `count(e.id)`? After a `LEFT JOIN`, Legal has one row with NULLs on the employee
side. `count(*)` counts that row:

```
 name  | count_star | count_id
-------+------------+----------
 Eng   |          3 |        3
 Legal |          1 |        0     ← count(*) says Legal has one employee
 Sales |          2 |        2
```

**Highest earner in each department (top-N per group).** Rank within a partition, then
filter outside:

```sql
SELECT d.name AS dept, r.name, r.salary
FROM (
  SELECT e.*, dense_rank() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS r
  FROM employee e WHERE salary IS NOT NULL
) r
JOIN department d ON d.id = r.dept_id
WHERE r.r = 1
ORDER BY dept;

 dept  | name  | salary
-------+-------+--------
 Eng   | Asha  |    200
 Sales | Kabir |    120
```

`dense_rank` returns every tied top earner; `row_number` would pick one arbitrarily.
PostgreSQL also has `DISTINCT ON` for "first row per group", covered with its own trap in
[postgres file 04](../../postgres/04-the-latest-row-per-group.md#distinct-on).

**Find duplicates:**

```sql
SELECT salary, count(*) FROM employee GROUP BY salary HAVING count(*) > 1;
-- 150 | 2
```

**Delete duplicates, keeping the lowest id:**

```sql
-- emails: (1,'a@x.in'), (2,'b@x.in'), (3,'a@x.in'), (4,'a@x.in')
DELETE FROM emails e USING emails keep
WHERE e.email = keep.email AND e.id > keep.id;
-- DELETE 2; remaining: (1,'a@x.in'), (2,'b@x.in')
```

Every row that has a same-email row with a smaller id is deleted, so only the smallest
survives. (`DELETE … USING` is PostgreSQL syntax; MySQL writes a multi-table `DELETE`
with a join, and a `row_number()` CTE works in both.)

## 8.8 The phrasebook

| Question shape | Reach for |
|---|---|
| "nth highest / top N" | `dense_rank()` (or `rank`/`row_number`, after asking about ties) |
| "…per department / per user" | `PARTITION BY` in the window, filter in an outer query |
| "rows with no match" | `NOT EXISTS` or `LEFT JOIN … IS NULL`, never `NOT IN (subquery)` |
| "filter on a count/sum" | `HAVING` |
| "compare a row to the previous one" | `lag()` / `lead()` |
| "running total / moving average" | `sum()` / `avg()` with `OVER (ORDER BY …)` |
| "employee vs manager" | Self join |
| "the whole hierarchy" | `WITH RECURSIVE` |
| "duplicates" | `GROUP BY … HAVING count(*) > 1`; delete with a self join or `row_number()` |
| "page through results" | Keyset pagination rather than large `OFFSET`s ([postgres file 05](../../postgres/05-pagination.md#keyset-pagination)) |

## Asked in interviews

- "Find the second / nth highest salary." → §8.5
- "Rank vs dense_rank vs row_number?" → §8.4
- "WHERE vs HAVING?" → §8.2
- "Why can't I use an aggregate or a window function in WHERE?" → §8.2
- "Top earner in each department." → §8.7
- "Employees who earn more than their manager." → §8.6
- "Find and delete duplicate rows." → §8.7
- "count(*) vs count(column)? How do aggregates treat NULL?" → §8.3
- "Why does NOT IN return nothing?" → §8.3

## Check yourself

1. Explain, using the evaluation order, why `SELECT salary * 12 AS annual FROM employee
   WHERE annual > 1000` fails but `… ORDER BY annual` works.
2. A colleague's "second-highest salary" query is `SELECT salary FROM employee ORDER BY
   salary DESC LIMIT 1 OFFSET 1`. Give two separate reasons it can return the wrong
   answer, using this chapter's data for one of them.
3. On this chapter's data, what does `rank()` assign to Zoya, and what does
   `dense_rank()`? Which would you use for "the people with the three highest salaries",
   and how many people does it return?
4. Why does `NOT IN` return zero rows in §8.3, while `NOT EXISTS` returns Legal? Write out
   the three-valued logic for Legal's row.
5. Write a query for "the department with the highest average salary", and say what your
   query does if two departments tie.
6. "For each employee, show their salary and the average salary of their department." Write
   it once with a window function and once with `GROUP BY` and a join. Which reads
   better, and is there a case where the results differ?

(Answers in [13-exercises.md](13-exercises.md).)
