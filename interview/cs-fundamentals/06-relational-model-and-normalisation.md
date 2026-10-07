# 6. The relational model, keys and normalisation

## The problem

Store data the obvious way, one big spreadsheet with a row per enrolment holding the
student's name, the course title and the instructor's phone number, and three things go
wrong. Change the instructor's phone and you must find every row that repeats it; miss
one and the data contradicts itself. You can't record a new course until someone enrols
in it. Delete the last enrolment and the course vanishes with it. Normalisation is the
discipline of designing tables so those three failures can't happen, and keys are what
make it possible to talk about "the same thing" across tables at all.

Assumes nothing beyond [file 01](01-what-happens-when-a-program-runs.md). This chapter
is about modelling; [file 07](07-transactions-and-isolation.md) is about what happens
when many users change the data at once.

## 6.1 The relational model in five terms

E. F. Codd proposed it in 1970 ("A relational model of data for large shared data
banks", *CACM* 13(6)). Its central idea was **data independence**: programs say *what*
data they want, in terms of values, and never depend on how it's stored, sorted or
indexed. That's why you can add an index without changing a query, and why SQL is
declarative.

| Formal term | Everyday term | Meaning |
|---|---|---|
| **Relation** | Table | A set of tuples with the same attributes |
| **Tuple** | Row | One fact: these values belong together |
| **Attribute** | Column | A named property with a **domain** (its allowed values) |
| **Schema** | Table definition | The relation's name and attributes |
| **Instance** | The data | The tuples present right now |

"Set" matters: in the pure model a relation has no duplicate tuples and no row order. SQL
relaxes the first (a table without a key can hold duplicate rows) and never had the
second, which is why `ORDER BY` is required if you care about order.

## 6.2 Before tables: the ER model

Design often starts a level up, with an **entity–relationship (ER) model**: entities
(Student, Course), their attributes, and relationships between them with
**cardinalities**. Each cardinality becomes a table shape:

| Relationship | Example | Becomes |
|---|---|---|
| One-to-one | User — Profile | A foreign key on either side, with a unique constraint |
| One-to-many | Department — Employee | A foreign key on the "many" side: `employee.dept_id` |
| Many-to-many | Student — Course | A **junction table**, `enrolment(student_id, course_id)`, holding a foreign key to each |

A **weak entity** has no identity of its own: an order line exists only within an order,
so its key includes the order's key.

## 6.3 Keys

A key is a set of columns that identifies a row. The vocabulary distinguishes *how
minimal* a key is and *which one you chose*:

| Term | Definition | Example in `employee(id, email, pan, name, dept_id)` |
|---|---|---|
| **Superkey** | Any set of columns whose values are unique per row | `{id}`, `{id, name}`, `{email, dept_id}` |
| **Candidate key** | A *minimal* superkey: remove any column and it's no longer unique | `{id}`, `{email}`, `{pan}` |
| **Primary key** | The candidate key you chose as the main identifier. Not null | `{id}` |
| **Alternate key** | A candidate key you didn't choose; usually enforced with a unique constraint | `{email}`, `{pan}` |
| **Composite key** | A key of more than one column | `{student_id, course_id}` in `enrolment` |
| **Foreign key** | Columns that must match a key in another (or the same) table | `dept_id` → `department(id)` |
| **Surrogate key** | An artificial identifier with no business meaning | An auto-increment `id` or a UUID |
| **Natural key** | A key made of real-world data | `pan`, `email` |

Surrogate vs natural is a real design choice. Natural keys change (people change
emails), can be reused, and spread personal data into every referencing table. Surrogates
are stable and compact but say nothing, so you still need a unique constraint on the
natural key or duplicates creep in.

The rules that keys enforce are **integrity constraints**: **entity integrity** (a
primary key is never null) and **referential integrity** (a foreign key always points at
a row that exists, or is null).

## 6.4 Functional dependencies: the idea underneath normalisation

Normalisation is defined in terms of one idea. A **functional dependency** `X → Y` means:
any two rows that agree on `X` must agree on `Y`. "Knowing X determines Y."

- `student_id → student_name`: a student ID always goes with the same name.
- `course_id → instructor`: each course has one instructor.
- `{student_id, course_id} → grade`: a grade belongs to a student *in* a course.

A **key** is then just a set of columns that determines every other column. And every
normal form is a rule of the form "the only dependencies allowed are ones whose
left-hand side is (or contains) a key". The anomalies in the opening paragraph come from
dependencies on something *other* than a key: a fact about the instructor stored in a row
that's about an enrolment.

William Kent's summary of 2NF and 3NF is the one to remember ("A Simple Guide to Five
Normal Forms in Relational Database Theory", *CACM*, 1983,
[text](https://www.bkent.net/Doc/simple5.htm)): "a non-key field must provide a fact about
the key, the whole key, and nothing but the key."

## 6.5 Normalisation, worked through

One example carries all the way from unnormalised to 3NF. A college records enrolments:

**Unnormalised.** One row per student, with a repeating group:

| student_id | student_name | courses |
|---|---|---|
| S1 | Asha | CS101 (Dr Rao, 98450-11111, A); CS102 (Dr Iyer, 98450-22222, B) |
| S2 | Ravi | CS101 (Dr Rao, 98450-11111, B) |

### First normal form: one value per cell

**1NF**: every attribute holds a single, atomic value, with no repeating groups or lists.
Split the list into rows:

| student_id | student_name | course_id | course_name | instructor | instructor_phone | grade |
|---|---|---|---|---|---|---|
| S1 | Asha | CS101 | Data Structures | Dr Rao | 98450-11111 | A |
| S1 | Asha | CS102 | Operating Systems | Dr Iyer | 98450-22222 | B |
| S2 | Ravi | CS101 | Data Structures | Dr Rao | 98450-11111 | B |

Key: `{student_id, course_id}`. Dependencies:

```
student_id            → student_name
course_id             → course_name, instructor
instructor            → instructor_phone
student_id, course_id → grade
```

All three anomalies are here. Dr Rao's phone is stored twice (**update anomaly**). You
can't add a course nobody's taking (**insertion anomaly**). If Ravi drops CS101 and Asha
does too, the fact that Dr Rao teaches it disappears (**deletion anomaly**).

### Second normal form: the whole key

**2NF**: 1NF, and no non-key attribute depends on only *part* of a composite key (no
**partial dependency**). `student_name` depends on `student_id` alone, and `course_name`
and `instructor` on `course_id` alone. Move each to a table keyed by what it depends on:

```
student(student_id, student_name)
course(course_id, course_name, instructor, instructor_phone)
enrolment(student_id, course_id, grade)
```

A table with a single-column key is automatically in 2NF: there's no "part" of the key to
depend on.

### Third normal form: nothing but the key

**3NF**: 2NF, and no non-key attribute depends on another non-key attribute (no
**transitive dependency**). In `course`, `course_id → instructor → instructor_phone`: the
phone is a fact about the instructor, not the course. Split it out:

```
student(student_id, student_name)
instructor(instructor, instructor_phone)
course(course_id, course_name, instructor)        -- instructor is now a foreign key
enrolment(student_id, course_id, grade)
```

Every anomaly is gone. The phone is stored once. A course can exist with no enrolments.
Deleting enrolments deletes only enrolments.

```mermaid
flowchart LR
    U["unnormalised<br/>repeating groups"] -->|"atomic values"| N1["1NF"]
    N1 -->|"remove partial<br/>dependencies"| N2["2NF"]
    N2 -->|"remove transitive<br/>dependencies"| N3["3NF"]
    N3 -->|"every determinant<br/>is a superkey"| BC["BCNF"]
    style N3 fill:#2d6a4f,stroke:#95d5b2,color:#fff
```

### BCNF: when 3NF isn't quite enough

**Boyce–Codd normal form**: for every non-trivial dependency `X → Y`, `X` is a superkey.
It's stricter than 3NF because 3NF has a loophole: it allows `X → Y` where `Y` is part of
some candidate key (a **prime attribute**). That loophole matters only when a table has
several overlapping candidate keys, which is rare, so the classic example is a little
contrived:

`tutoring(student, subject, tutor)`, where each tutor teaches exactly one subject, and a
student has one tutor per subject.

```
student, subject → tutor
tutor            → subject
```

Candidate keys: `{student, subject}` and `{student, tutor}`. Every attribute is in some
candidate key, so the table is in 3NF. But `tutor → subject` has a left side that isn't a
superkey, so it violates BCNF, and the anomaly is real: "Dr Rao teaches Physics" is
repeated for every student Dr Rao tutors.

Decompose into `tutor_subject(tutor, subject)` and `student_tutor(student, tutor)`. The
redundancy is gone, but look at what's lost: the rule "a student has only one tutor per
subject" (`student, subject → tutor`) no longer lives inside any one table. Two rows in
`student_tutor` could give a student two Physics tutors, and no single-table constraint
can stop it.

That trade-off is the textbook result worth knowing (Silberschatz, Korth & Sudarshan,
*Database System Concepts*, 7th ed., 2019): a **lossless** decomposition into **3NF**
that **preserves every dependency** always exists. For **BCNF**, lossless is always
possible but dependency preservation isn't. So 3NF is the practical target, and BCNF is
what you get for free in the common case where there's only one candidate key.

> **Teacher's aside.** Normalisation is often taught as a ladder of rules to memorise.
> It's better understood as one rule applied with increasing strictness: *every fact
> should be stored exactly once, in the table whose key it's a fact about.* 2NF catches
> facts about part of the key, 3NF catches facts about a non-key column, BCNF catches the
> leftover case with overlapping keys. When you look at a table, ask of each column
> "what is this a fact about?" If the answer isn't "the key", it's in the wrong table.

### How many normal forms are there, really?

The sources disagree on how many matter, and it's worth knowing both sides:

| Normal form | Origin | What it removes | Who bothers |
|---|---|---|---|
| 1NF | Codd, 1970 | Repeating groups, non-atomic values | Everyone |
| 2NF, 3NF | Codd, 1971 ("Further Normalization of the Data Base Relational Model") | Partial and transitive dependencies | Everyone; "normalised" usually means 3NF |
| BCNF | Boyce and Codd, 1974 | Dependencies on non-superkeys | Taught everywhere; rarely differs from 3NF in practice |
| 4NF | Fagin, 1977 | Independent multi-valued facts in one table (a person's skills and languages in one table, producing every combination) | Textbooks; occasionally real |
| 5NF (PJNF) | Fagin, 1979 | Join dependencies not implied by keys | Mostly theory |
| DKNF | Fagin, 1981 | Everything, by defining all constraints via domains and keys | Theory |
| 6NF | Date, Darwen and Lorentzos, 2003 | Every non-trivial join dependency; one non-key attribute per table | Temporal databases, some warehouse designs |

Interview questions stop at BCNF almost without exception. Silberschatz covers 4NF and
mentions the rest. Date and Darwen argue 6NF is the natural form for temporal data. In
practice 4NF violations are worth recognising (the "every combination" smell), and
beyond that, the honest answer is "I know they exist and what they target".

### Denormalisation

Normalised designs make writes safe and reads more expensive, because answering a
question needs joins. Kent said it in 1983: normalisation rules "tend to penalize
retrieval", and "There is no obligation to fully normalize all records when actual
performance requirements are taken into account."

Denormalising (storing a derived or duplicated value on purpose, such as an order total
or a `latest_status` column) is legitimate when you know who keeps the copy correct. The
failure mode is the update anomaly you designed out, coming back. The
[postgres course, file 07](../../postgres/07-history-rows-and-derived-state.md) works
through exactly this trade-off for derived state.

## 6.6 Joins

A join combines rows from two tables where a condition holds. The kinds differ in what
they do with rows that have **no match**.

The two tables used here (and in [file 08](08-sql-interviews-ask.md)) were loaded into
PostgreSQL 17 and every count below is from running the query:

```
department                employee
id | name                 id | name  | salary | dept_id
---+------                ---+-------+--------+--------
 1 | Eng                   1 | Asha  |   200  |   1
 2 | Sales                 2 | Ravi  |   150  |   1
 3 | Legal                 3 | Meera |   150  |   1
                           4 | Kabir |   120  |   2
                           5 | Zoya  |   100  |   2
                           6 | Dev   |  NULL  |  NULL
```

Legal has no employees. Dev has no department.

| Join | Keeps | `employee e … department d ON d.id = e.dept_id` rows |
|---|---|---|
| **INNER JOIN** | Only rows with a match on both sides | **5** (Dev and Legal both dropped) |
| **LEFT JOIN** | Every row of the left table; NULLs where the right has no match | **6** (Dev kept, with NULL department) |
| **RIGHT JOIN** | Every row of the right table | 6 (Legal kept, with NULL employee) |
| **FULL OUTER JOIN** | Every row of both | **7** (Dev *and* Legal) |
| **CROSS JOIN** | Every combination; no condition | **18** (6 × 3) |
| **Self join** | A table joined to itself, under two aliases | Employee → manager ([file 08 §8.6](08-sql-interviews-ask.md#86-self-joins-and-hierarchies)) |

```
         INNER               LEFT                FULL
      ┌───┬───┬───┐      ┌───┬───┬───┐      ┌───┬───┬───┐
      │   │███│   │      │███│███│   │      │███│███│███│
      └───┴───┴───┘      └───┴───┴───┘      └───┴───┴───┘
       emp  both dept     emp  both dept     emp  both dept
```

Venn diagrams like these are the standard teaching picture and they're slightly
misleading: a join isn't a set intersection, because one row can match *many* rows. If
`department` had two rows with `id = 1`, every Eng employee would appear twice. A join's
output size is governed by how many matches each row has, which is why a join on a
non-unique column can multiply rows unexpectedly.

> ⚠️ **A filter on the right table in `WHERE` turns a LEFT JOIN into an INNER JOIN.**
> `LEFT JOIN department d ON d.id = e.dept_id WHERE d.name <> 'Sales'` drops Dev, because
> Dev's `d.name` is NULL and `NULL <> 'Sales'` isn't true. To keep unmatched rows, put the
> condition in the `ON` clause instead.

How the database *executes* a join (nested loop, hash join, merge join) is a separate
question; the [postgres course, file 01 "Reading a plan"](../../postgres/01-what-postgres-does-with-a-query.md#reading-a-plan)
introduces the three methods.

## 6.7 Relational algebra, briefly

SQL is built on a small algebra, and interviewers at some companies ask for it by name:

| Operator | Symbol | SQL equivalent |
|---|---|---|
| Selection | σ (sigma) | `WHERE`: keep rows that satisfy a condition |
| Projection | π (pi) | The `SELECT` column list (with `DISTINCT`, since relations are sets) |
| Join | ⋈ | `JOIN … ON` |
| Union, difference, intersection | ∪, −, ∩ | `UNION`, `EXCEPT`, `INTERSECT` |
| Cartesian product | × | `CROSS JOIN` |
| Rename | ρ (rho) | `AS` |

The confusing pair is selection and projection, because SQL's `SELECT` keyword does
*projection*, and selection happens in `WHERE`.

## Asked in interviews

- "What's normalisation? Explain 1NF, 2NF, 3NF and BCNF with an example." → §6.5
- "What are update, insertion and deletion anomalies?" → §6.5, the 1NF table
- "Difference between 3NF and BCNF?" → §6.5, "BCNF"
- "Super key vs candidate key vs primary key?" → §6.3
- "What's a functional dependency?" → §6.4
- "When would you denormalise?" → §6.5, "Denormalisation"
- "Explain the types of joins. How many rows does this join return?" → §6.6
- "Selection vs projection?" → §6.7

## Check yourself

1. In `employee(id, email, name, dept_id)` where `id` and `email` are both unique, list
   two superkeys that aren't candidate keys, and say why `{id, email}` isn't a candidate
   key.
2. A table `order_line(order_id, product_id, quantity, product_name, unit_price)` has key
   `{order_id, product_id}`. Which normal form does it violate, which dependency causes
   it, and what's the fix? Is there an argument for keeping `unit_price` on the order line
   anyway?
3. Explain why a table whose primary key is a single column can't violate 2NF, but can
   violate 3NF. Give an example.
4. In the tutoring example, explain why the table is in 3NF, why it's not in BCNF, and
   exactly which business rule becomes impossible to enforce with simple keys after the
   BCNF decomposition.
5. With this chapter's tables, how many rows does
   `department d LEFT JOIN employee e ON e.dept_id = d.id` return, and which department
   rows have NULLs? Then predict the count if `department` accidentally had two rows with
   `id = 1`.
6. A `LEFT JOIN` returns fewer rows than the left table has. Name two different causes.

(Answers in [13-exercises.md](13-exercises.md).)
