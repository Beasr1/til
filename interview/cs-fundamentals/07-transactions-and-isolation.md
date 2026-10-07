# 7. Transactions, isolation and indexes

## The problem

A transfer debits one account and credits another. If the server crashes between the
two writes, money vanishes. If two transfers run at once, each can read a balance the
other is about to change, and the totals stop adding up. A database has to make a group
of writes behave as one indivisible step, survive crashes, and let hundreds of clients
work at once without seeing each other's half-finished work. Transactions are how. The
interview questions about isolation levels are where most candidates (and much prep
material) go wrong, because the textbook definitions are themselves known to be
inadequate, and real databases don't follow them literally.

Assumes [file 04](04-synchronisation-and-deadlock.md) (races, locks, deadlock) and
[file 06](06-relational-model-and-normalisation.md) (tables and keys). The
[postgres course, file 01](../../postgres/01-what-postgres-does-with-a-query.md) covers
the storage side of what's here, MVCC row versions and the WAL, in more depth.

## 7.1 What a transaction promises: ACID

A **transaction** is a sequence of reads and writes that the database treats as one
unit, between `BEGIN` and `COMMIT` (or `ROLLBACK`). The four promises got their name in
Haerder and Reuter's "Principles of transaction-oriented database recovery" (*ACM
Computing Surveys*, 1983): "These four properties, atomicity, consistency, isolation, and
durability (ACID), describe the major highlights of the transaction paradigm".

| Letter | Promise | Breaks without it | Provided by |
|---|---|---|---|
| **Atomicity** | All of the transaction's writes happen, or none do | Debit without credit after a crash | Logging, so incomplete work can be undone or ignored on recovery |
| **Consistency** | A transaction takes the database from one valid state to another | Negative balances, orphaned foreign keys | **Partly you.** The database enforces declared constraints; any other invariant is the application's job |
| **Isolation** | Concurrent transactions don't see each other's partial work, to a degree set by the isolation level | Lost updates, reports that don't add up | Locks and/or multi-versioning (§7.5) |
| **Durability** | Once `COMMIT` returns, the change survives a crash | "Your payment succeeded", then it's gone after a restart | Writing the log to stable storage before acknowledging the commit |

"Consistency" is the odd one out. It's the only property the database can't deliver on
its own: it can't know that your two account rows should sum to the same total before
and after a transfer unless you tell it. The other three are tools that make it possible
for *your code* to preserve consistency.

Atomicity and durability share a mechanism: the **write-ahead log**. Every change is
recorded in the log before the data files change, and a commit is durable once its log
record is flushed to disk. After a crash, the database replays the log to redo committed
work and discards what was never committed. The postgres course covers this in
[file 01 "Every change is logged first: the WAL"](../../postgres/01-what-postgres-does-with-a-query.md#every-change-is-logged-first-the-wal).

## 7.2 Isolation: why not just run one at a time?

Perfect isolation is easy: run transactions one after another, **serially**. Nothing can
interfere. It's also hopelessly slow, because a transaction waiting on a disk read or a
slow client would hold up everyone.

So databases run transactions concurrently and promise that the result looks like *some*
serial order. That's **serialisability**, the gold standard. The SQL standard defines it
the same way (SQL-92, §4.28): "A serializable execution is defined to be an execution of
the operations of concurrently executing SQL-transactions that produces the same effect
as some serial execution of those same SQL-transactions."

Serialisability costs throughput, so the standard also defines weaker **isolation
levels** that allow some anomalies in exchange for more concurrency. The trouble starts
with how it defined them.

## 7.3 The ANSI levels, as the standard wrote them

SQL-92 defines the levels by which **phenomena** each one rules out. The definitions,
from the standard's text ([SQL-92 draft, §4.28](https://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)):

| Phenomenon | SQL-92's definition |
|---|---|
| **P1 Dirty read** | "SQL-transaction T1 modifies a row. SQL-transaction T2 then reads that row before T1 performs a COMMIT. If T1 then performs a ROLLBACK, T2 will have read a row that was never committed" |
| **P2 Non-repeatable read** | "SQL-transaction T1 reads a row. SQL-transaction T2 then modifies or deletes that row and performs a COMMIT. If T1 then attempts to reread the row, it may receive the modified value or discover that the row has been deleted." |
| **P3 Phantom** | "T1 reads the set of rows N that satisfy some <search condition>. SQL-transaction T2 then executes SQL-statements that generate one or more rows that satisfy the <search condition> used by SQL-transaction T1. If SQL-transaction T1 then repeats the initial read with the same <search condition>, it obtains a different collection of rows." |

And the levels:

| Level | Dirty read | Non-repeatable read | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Not possible | Possible | Possible |
| REPEATABLE READ | Not possible | Not possible | Possible |
| SERIALIZABLE | Not possible | Not possible | Not possible |

One detail most prep material gets wrong: in the standard, "The isolation level of a
SQL-transaction is SERIALIZABLE by default." Real databases choose otherwise:
PostgreSQL's default is Read Committed, InnoDB's is Repeatable Read.

The difference between P2 and P3 is *one row changed* versus *the set of rows matching a
condition changed*. Rereading row 7 and getting a different balance is P2. Counting
`WHERE status = 'pending'` twice and getting a different count because someone inserted
a pending row is P3.

## 7.4 Where the definitions break

In 1995, Berenson, Bernstein, Gray, Melton, O'Neil and O'Neil published "A Critique of
ANSI SQL Isolation Levels" (SIGMOD '95; [arXiv cs/0701157](https://arxiv.org/abs/cs/0701157)).
Their opening finding: "The three ANSI phenomena are ambiguous. Even their broadest
interpretations do not exclude anomalous behavior." Four problems, each now standard
vocabulary:

**1. Dirty writes aren't mentioned at all.** They define **P0 (Dirty Write)**: "Transaction
T1 modifies a data item. Another transaction T2 then further modifies that data item
before T1 performs a COMMIT or ROLLBACK. If T1 or T2 then performs a ROLLBACK, it is
unclear what the correct data value should be." Without forbidding this, even rollback
can't work properly. Their Remark 3: "ANSI SQL isolation should be modified to require P0
for all isolation levels."

**2. Each phenomenon has a strict and a broad reading.** Written as histories (`w1[x]` =
T1 writes x, `r2[x]` = T2 reads x, `c1`/`a1` = T1 commits/aborts):

```
A1 (strict dirty read):  w1[x]...r2[x]...(a1 and c2 in any order)
P1 (broad dirty read):   w1[x]...r2[x]...((c1 or a1) and (c2 or a2) in any order)
```

The strict reading forbids a dirty read only if it *actually* turns out wrong (T1 aborts).
The broad reading forbids reading uncommitted data at all, whatever happens next. They
show the strict reading lets through real anomalies and conclude "we should take the
interpretation P1 for what was intended by ANSI rather than A1." The same split exists for
P2/A2 and P3/A3.

**3. Important anomalies don't appear in the table.** They named the ones the standard
missed:

| Anomaly | History | In words |
|---|---|---|
| **P4 Lost update** | `r1[x]...w2[x]...w1[x]...c1` | "T1 reads a data item and then T2 updates the data item ... then T1 (based on its earlier read value) updates the data item and commits." T2's update is overwritten |
| **A5A Read skew** | `r1[x]...w2[x]...w2[y]...c2...r1[y]...(c1 or a1)` | T1 reads x, T2 changes x and y together and commits, T1 reads y. T1 sees old x with new y: a state that never existed |
| **A5B Write skew** | `r1[x]...r2[y]...w1[y]...w2[x]...(c1 and c2 occur)` | Each transaction reads what the other will write, then writes something different. Each was valid alone; together they break an invariant |

**4. Snapshot isolation doesn't fit on the ladder.** Many databases implement isolation
by giving each transaction a consistent **snapshot** of committed data as of when it
started, plus a rule that if two concurrent transactions write the same item, the first
to commit wins and the other aborts ("first-committer-wins"). The paper shows this
**snapshot isolation** prevents every ANSI phenomenon under the strict readings, and is
still "not serializable", because it allows write skew. It also allows less than their
locking-based REPEATABLE READ in one respect and more in another: "Snapshot Isolation
histories prohibit histories with anomaly A3, but allow A5B, while REPEATABLE READ does
the opposite." Their Remark 9 writes it as "REPEATABLE READ »« Snapshot Isolation": the
two are **incomparable**, neither stronger than the other.

The paper's Table 4, abridged (their REPEATABLE READ is the locking-based definition):

| Level | P0 dirty write | P1 dirty read | P4 lost update | P2 fuzzy read | P3 phantom | A5A read skew | A5B write skew |
|---|---|---|---|---|---|---|---|
| READ UNCOMMITTED | Not possible | Possible | Possible | Possible | Possible | Possible | Possible |
| READ COMMITTED | Not possible | Not possible | Possible | Possible | Possible | Possible | Possible |
| REPEATABLE READ | Not possible | Not possible | Not possible | Not possible | Possible | Not possible | Not possible |
| Snapshot | Not possible | Not possible | Not possible | Not possible | Sometimes possible | Not possible | **Possible** |
| SERIALIZABLE | Not possible | Not possible | Not possible | Not possible | Not possible | Not possible | Not possible |

> **Teacher's aside.** The table most people memorise, three phenomena and four levels,
> is the one the field's own experts showed was ambiguous in 1995. That doesn't make it
> useless: it's still the shared vocabulary, and interviewers will use it. But it means
> "which level prevents phantoms?" has a different answer per database, because each one
> implements its levels with different machinery and maps them onto the standard's names
> as best it can. The better question is always "what does *this* database do at *this*
> level?", and the answer comes from its documentation, not from the standard.

## 7.5 What PostgreSQL actually does, reproduced

PostgreSQL's own table ([docs §13.2, Table 13.1](https://www.postgresql.org/docs/current/transaction-iso.html))
adds a fourth column for exactly the gap Berenson et al. found:

| Level | Dirty read | Non-repeatable read | Phantom read | Serialization anomaly |
|---|---|---|---|---|
| Read uncommitted | Allowed, but not in PG | Possible | Possible | Possible |
| Read committed | Not possible | Possible | Possible | Possible |
| Repeatable read | Not possible | Not possible | Allowed, but not in PG | Possible |
| Serializable | Not possible | Not possible | Not possible | Not possible |

What sits behind it, from the same page:

- Read Uncommitted isn't really a level: "PostgreSQL's Read Uncommitted mode behaves like
  Read Committed."
- "Read Committed is the default isolation level in PostgreSQL." Each *statement* sees a
  fresh snapshot.
- Repeatable Read "is implemented using a technique known in academic database literature
  and in some other database products as Snapshot Isolation", and is "a stronger guarantee
  than is required by the SQL standard for this isolation level". The whole *transaction*
  sees one snapshot.
- Serializable "is implemented using a technique known in academic database literature as
  Serializable Snapshot Isolation, which builds on Snapshot Isolation by adding checks for
  serialization anomalies", and "applications using this level must be prepared to retry
  transactions due to serialization failures."

I ran each case on PostgreSQL 17.11 in a container, with two `psql` sessions overlapping
in time:

| Experiment | Level | What happened |
|---|---|---|
| A reads balance (100); B updates it to 50 and commits; A reads again | Read Committed | A's second read: **50**. Non-repeatable read |
| Same | Repeatable Read | A's second read: **100**. A also didn't see a row B inserted (count stayed 2): no phantom either |
| …then A tries to update that same row | Repeatable Read | `ERROR: could not serialize access due to concurrent update`. First-committer-wins |
| B updates a row to 0 without committing; A reads it | Read Uncommitted | A sees **100**. No dirty read, as documented |
| Two transactions each read balance 100, then write `100 + 10` and `100 + 20` | Read Committed | Final balance **110**, not 130. **Lost update** |
| Write skew (below) | Repeatable Read | Both commit. Invariant broken |
| Write skew (below) | Serializable | Second commit fails: `could not serialize access due to read/write dependencies among transactions`, `HINT: The transaction might succeed if retried.` |

The write-skew case is the standard example, and worth walking through. A hospital rule:
at least one doctor must be on call. Alice and Bob both are.

```
T1 (Alice)                                  T2 (Bob)
BEGIN ISOLATION LEVEL REPEATABLE READ;      BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM oncall WHERE on_call;  SELECT count(*) FROM oncall WHERE on_call;
  → 2, so it's safe for me to go off          → 2, so it's safe for me to go off
UPDATE oncall SET on_call = false           UPDATE oncall SET on_call = false
  WHERE doc = 'alice';                        WHERE doc = 'bob';
COMMIT;                                     COMMIT;

→ nobody on call
```

They wrote *different* rows, so first-committer-wins sees no conflict. Each transaction
was correct given its snapshot. Only the combination is wrong, and no serial order could
have produced it: whichever ran second would have seen a count of 1 and stayed. Under
Serializable, PostgreSQL tracks what each transaction *read* (the docs say these show up
"in the pg_locks system view with a mode of SIReadLock"), notices the dangerous pattern,
and aborts one.

MySQL's InnoDB makes different choices for the same names. Repeatable Read is "the
default isolation level for InnoDB", and for locking reads it "locks the index range
scanned, using gap locks or next-key locks to block insertions by other sessions into the
gaps covered by the range" ([MySQL 8.4 manual](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)).
Same level name, different mechanism, different edge cases. Hence the aside.

> ⚠️ **Read Committed is the default, and it allows lost updates.** The 110-instead-of-130
> result above is what an application gets by default if it reads a value, computes in
> application code, and writes the result back. It doesn't error. It just loses money.

## 7.6 Fixing the lost update, four ways

| Fix | How | Trade-off |
|---|---|---|
| **1. Atomic update** | Do the arithmetic in the database: `UPDATE acct SET bal = bal + 10 WHERE id = 1` | Simplest, when the logic fits in SQL. Reproduced: two concurrent deposits gave **130** under Read Committed |
| **2. Pessimistic lock** | `SELECT bal FROM acct WHERE id = 1 FOR UPDATE`, compute, then `UPDATE` | The second transaction blocks at its `SELECT … FOR UPDATE` until the first commits, then reads the new value. Reproduced: **130**. Holds a row lock across application work; deadlock-prone if rows are locked in different orders ([file 04 §4.10](04-synchronisation-and-deadlock.md#410-four-ways-to-handle-deadlock)) |
| **3. Optimistic check** | Add a `version` column. `UPDATE acct SET bal = $new, version = version + 1 WHERE id = 1 AND version = $read` and check the affected-row count | No locks held while you think. If someone else got there first, the update matches 0 rows and you retry. Good when conflicts are rare |
| **4. Stronger isolation + retry** | Run at Repeatable Read or Serializable; catch serialisation failures (SQLSTATE `40001`) and retry the whole transaction | Covers anomalies you didn't think of, including write skew at Serializable. Every transaction needs a retry loop |

Why does fix 1 work at Read Committed, when the read-then-write version doesn't? The docs
explain it: if the row was updated by a concurrent transaction, "the would-be updater will
wait for the first updating transaction to commit or roll back", and then "will attempt
to apply its operation to the updated version of the row", re-evaluating the `WHERE`
clause. So `bal = bal + 10` is computed from the *committed* value, not a stale one.

## 7.7 Two ways to implement isolation: locks and versions

| | Locking (two-phase locking) | Multi-versioning (MVCC) |
|---|---|---|
| Core idea | Readers take shared locks, writers take exclusive locks, and conflicting requests wait | Writers create new row versions; readers read the version their snapshot says is visible |
| Readers vs writers | Block each other | "reading never blocks writing and writing never blocks reading" ([PostgreSQL §13.1](https://www.postgresql.org/docs/current/mvcc-intro.html)) |
| Writers vs writers | Block each other | Still block each other on the same row |
| Cost | Lock contention, deadlocks | Old versions pile up and need cleaning (`VACUUM` in PostgreSQL) |
| Gives serialisability by | Strict 2PL: holding all locks until commit, plus range or predicate locks for phantoms | An extra layer, such as SSI's read tracking, because plain snapshot isolation allows write skew |

**Two-phase locking (2PL)** means a transaction acquires locks in a growing phase and
releases them in a shrinking phase, never acquiring after it has released. That rule is
enough to guarantee serialisability. **Strict 2PL** holds write locks until commit, so
nobody reads uncommitted data and aborts don't cascade. The lock modes are the familiar
readers–writers pattern from [file 04 §4.7](04-synchronisation-and-deadlock.md#47-the-classic-problems):

| Held \ Requested | Shared (S) | Exclusive (X) |
|---|---|---|
| **Shared (S)** | Compatible | Wait |
| **Exclusive (X)** | Wait | Wait |

Real databases mix both. PostgreSQL uses MVCC for row data and still takes locks for
writes to the same row and for schema changes. The
[postgres course, file 01 "An update writes a new row: MVCC"](../../postgres/01-what-postgres-does-with-a-query.md#an-update-writes-a-new-row-mvcc)
shows row versions on disk, and [file 09 "Locks, briefly"](../../postgres/09-changing-a-live-table.md#locks-briefly)
covers table lock modes.

## 7.8 Indexes, at interview depth

Without an index, finding `WHERE id = 42` means reading every row (a **full table
scan**). An index is a separate structure that maps key values to row locations, kept
sorted so a lookup touches a few pages instead of all of them.

### Why a B+ tree, not a binary search tree

Disks and the OS read in pages (PostgreSQL's are 8 kB), and fetching a page costs the
same whether you use one byte or all of it. A binary search tree uses one key per node,
so finding one key among a billion takes about 30 node visits, potentially 30 page reads.
A **B-tree** packs hundreds of keys into each node, one node per page, so the tree is
short and fat.

A **B+ tree** is the variant databases use:

```
                      ┌──────────────┐
                      │   30 │ 60    │                    internal nodes: keys only,
                      └──┬────┬───┬──┘                    for routing
            ┌────────────┘    │   └────────────┐
      ┌─────▼─────┐     ┌─────▼─────┐     ┌────▼──────┐
      │ 10 │ 20   │     │ 40 │ 50   │     │ 70 │ 80   │
      └─┬────┬──┬─┘     └─┬────┬──┬─┘     └─┬────┬──┬─┘
        ▼    ▼  ▼         ▼    ▼  ▼         ▼    ▼  ▼
  leaves: [5,10] ⇄ [20,25] ⇄ [30,35] ⇄ [40,45] ⇄ ...         leaves: every key, with a
                                                            pointer to its row, linked
                                                            to their neighbours
```

| Property | Why it matters |
|---|---|
| High fan-out (hundreds of children per node) | Very few levels. As an illustration, with a fan-out of 200, three levels index 200³ = 8 million keys and four levels 1.6 billion |
| All keys and row pointers live in the leaves | Every lookup costs the same number of levels |
| Leaves are linked in key order | Range scans (`BETWEEN`, `>`, `ORDER BY`) find the start, then walk sideways |
| Internal nodes hold only routing keys | More keys per page, so even more fan-out |

PostgreSQL's btree is a concurrency-oriented variant; its source README describes "a
correct implementation of Lehman and Yao's high-concurrency B-tree management algorithm"
([nbtree README](https://github.com/postgres/postgres/blob/master/src/backend/access/nbtree/README)),
which adds a right-link from each page to its sibling. Neither the PostgreSQL docs nor the
README calls it a "B+ tree"; that label is the textbook's, and it fits because the leaves
hold the row pointers and are linked.

### Clustered vs secondary indexes

| | Clustered (InnoDB) | Heap + indexes (PostgreSQL) |
|---|---|---|
| Where rows live | Inside the primary key's B+ tree, in key order. "Each InnoDB table has a special index called the clustered index that stores row data" ([MySQL manual](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)) | In an unordered heap. Every index, including the primary key's, points at a row location |
| Secondary index stores | "the primary key columns for the row, as well as the columns specified for the secondary index" | A pointer to the row's location in the heap |
| Consequence | A secondary lookup is two tree searches; a wide primary key bloats every secondary index; range scans on the PK are cheap | Every index lookup goes index → heap (unless it's an index-only scan) |

### The rules interviewers probe

- **Composite indexes follow the leftmost prefix.** An index on `(country, city)` is
  sorted by country, then city within each country. It serves `WHERE country = 'IN'` and
  `WHERE country = 'IN' AND city = 'Pune'`, but not `WHERE city = 'Pune'` alone: Pune
  rows are scattered across every country. Like a phone book sorted by surname then first
  name.
- **An index serves only what its order can answer.** `LIKE 'abc%'` can use a btree;
  `LIKE '%abc'` can't. Wrapping the column in a function (`WHERE lower(email) = …`) hides
  it from a plain index.
- **Indexes cost writes.** Every insert updates every index on the table.
- **Low selectivity defeats them.** If a condition matches most of the table, reading the
  whole table sequentially is cheaper than jumping between index and table.
- **Hash indexes** answer equality only, no ranges and no ordering.

The [postgres course, file 02](../../postgres/02-how-an-index-answers-a-query.md#a-btree-is-a-sorted-list)
works through when a btree does and doesn't answer a query, with real plans, including
the collation trap that stops `LIKE 'x%'` using one.

## Asked in interviews

- "Explain ACID." → §7.1
- "What are the isolation levels and which anomalies does each prevent?" → §7.3, then §7.5 for what PostgreSQL actually does
- "Dirty read vs non-repeatable read vs phantom?" → §7.3
- "What's a lost update / write skew? How do you prevent it?" → §7.4, §7.6
- "Optimistic vs pessimistic locking?" → §7.6
- "What's MVCC? How does it differ from locking? What's two-phase locking?" → §7.7
- "Why do databases use B+ trees?" → §7.8
- "Clustered vs non-clustered index? When won't an index be used?" → §7.8

## Check yourself

1. Which ACID property can't the database guarantee on its own, and why? Give an
   invariant the database couldn't know about unless you declared it.
2. Explain the difference between a non-repeatable read and a phantom using one table of
   orders. Then explain why PostgreSQL's Repeatable Read prevents both, even though the
   standard only requires it to prevent the first.
3. Write the history for the lost update in §7.5 (two deposits) in Berenson et al.'s
   notation. Which of the four fixes in §7.6 would you choose for a wallet balance, and
   why?
4. Explain why snapshot isolation allows write skew but prevents lost updates. Your answer
   should say what first-committer-wins checks and what it doesn't.
5. A teammate says "we're on Repeatable Read, so we can't get anomalies." Give a concrete
   counterexample for PostgreSQL, and say what would make it safe.
6. A table has an index on `(merchant_id, created_at)`. For each query, say whether the
   index helps and why: (a) `WHERE merchant_id = 7 ORDER BY created_at DESC LIMIT 20`;
   (b) `WHERE created_at > now() - interval '1 day'`; (c) `WHERE merchant_id = 7 AND
   status = 'failed'`.

(Answers in [13-exercises.md](13-exercises.md).)
