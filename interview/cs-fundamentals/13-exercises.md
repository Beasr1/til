# 13. Exercises & answers

Every "Check yourself" question, with worked answers. Commit to an answer before reading
one. In an interview, the reasoning is what's being marked, so the answers show it.

---

## File 1 — What happens when a program runs

**1. Program vs process; process vs thread.**
A program is an executable file on disk; a process is that program running, with its own
address space, open descriptors and identity. A thread is one stream of execution inside a
process; a process can have many, and they share its memory.

**2. How often does the kernel run during a billion-iteration arithmetic loop?**
Only when something forces it in. The loop makes no system calls and (once its pages are
mapped) causes no faults, so the kernel runs only on timer interrupts, typically every few
milliseconds, plus any device interrupts handled on that CPU. On each one it may let the
loop continue or switch to another thread. Between those, the loop runs on the bare CPU
with no kernel involvement.

**3. One-byte reads vs 128 KiB reads of a 1 GiB file.**
Each `read` is a system call: a trap into kernel mode and back, with registers saved and
caches and predictors disturbed. One byte at a time is about a billion system calls;
128 KiB at a time is about 8,200. The arithmetic is identical; the difference is almost
entirely boundary crossings. That's why buffered I/O exists.

**4. Two processes print the same address for a global; one changes it.**
The other doesn't see the change. The addresses are *virtual*. Each process has its own
page table, so the same virtual address translates to different physical memory. (After a
`fork` they may briefly share the physical page, but it's copy-on-write: the first write
gives the writer its own copy.)

**5. The three ways into the kernel, during a Go request.**
A system call: `read` on the client's socket, or `write` of the response. An interrupt:
the network card signalling that the request's packets arrived, or the timer interrupt
preempting the goroutine's thread. A fault: the first touch of a freshly allocated heap
page (a minor page fault), or a nil-pointer dereference (an invalid fault, which Go turns
into a panic).

**6. "The OS is always running."**
Wrong: most of the time your code is running on the CPU, and the kernel isn't running at
all. What's always true is that the kernel *can* take control: the timer interrupt is
armed, and any system call, interrupt or fault traps into it. Control is guaranteed;
continuous execution isn't.

---

## File 2 — Processes and threads

**1. Why a segfault in one thread kills its siblings.**
The threads share one address space and one set of resources, so the kernel can't know
whether the faulting thread corrupted memory the others depend on. The signal is delivered
to the process, whose default action is to terminate it, all threads included. Separate
processes have separate address spaces, so one's crash can't have corrupted the other.

**2. A process's disk read finishes.**
It moves from Waiting to **Ready**, not Running. It joins the run queue and waits for the
scheduler to choose it. If the CPUs are busy with other ready threads, that can take a
while, which is latency that shows up even though the I/O itself was fast.

**3. Why fork and exec are separate.**
So that between them, the child can configure its own environment with ordinary code
before the new program starts. For `./server > out.txt`, the child closes descriptor 1,
opens `out.txt` (which gets descriptor 1, the lowest free one), then `exec`s; the server
inherits descriptor 1 and writes to the file without knowing. A single "spawn" call needs
a parameter for every such adjustment (redirections, working directory, environment,
privileges, which descriptors to close). That's what `posix_spawn`'s long argument lists
are.

**4. N:1 threading and a blocking read.**
All 99 others stop. The kernel sees one thread; when it blocks in `read`, the kernel puts
that thread to sleep, and with it every user thread the library was multiplexing on it.
Go avoids this in two ways: network I/O goes through the netpoller, so a goroutine waiting
on a socket parks without any thread blocking; and when a goroutine makes a blocking
system call, the runtime detaches its P from the blocked M and gives it to another M, so
other goroutines keep running.

**5. 50,000 goroutines, `GOMAXPROCS=8`.**
At most 8 goroutines execute Go code at any instant, one per P. Thread count is usually
somewhat above 8: one M per P, plus a few for the runtime's own work (my 100,000-goroutine
test created 14 threads with 12 Ps). It climbs well above 8 when many goroutines are in
blocking system calls at once, such as file I/O or cgo calls, because each blocked call
holds an M and the runtime starts new Ms to keep the Ps busy.

**6. Node: heavy JSON transformation makes everything slow.**
All JavaScript runs on one thread. While a handler spends 50 ms transforming JSON, no
other callback runs, so even a trivial request waits until the event loop is free.
Latency for everything rises with the CPU work of anything. Fixes: move the CPU-heavy work
to `worker_threads` (or a separate service), stream or reduce the payloads, or run several
Node processes behind a balancer. `async`/`await` doesn't help: it gives up the thread only
at an `await`, and pure computation has none.

---

## File 3 — CPU scheduling

**1. A (8), B (4), C (1), all at time 0.**
FCFS runs A, B, C: waits 0, 8, 12, average 20/3 ≈ 6.67 ms. SJF runs C, B, A: waits 0, 1, 5,
average 2 ms. The difference is large because under FCFS the longest job's whole burst is
added to *both* other waits; SJF makes the long burst the one that delays nobody.

**2. RR with a 30 ms quantum.**
Every job's burst (24, 3, 3) is shorter than the quantum, so each runs to completion in
its first slice, in arrival order. That's FCFS exactly: average wait 17 ms. In general RR
behaves exactly like FCFS whenever the quantum is at least as long as every burst.

**3. "SJF is optimal, so why doesn't Linux use it?"**
Because it needs each job's next CPU burst *in advance*, and the kernel can't know it.
SJF is a yardstick. Approximations: predicting the burst from history (exponential
averaging), and designs that achieve SJF-like behaviour without predictions: MLFQ
(start high, demote jobs that use their whole allotment) and fair schedulers like CFS and
EEVDF (a task that slept has used less than its share, so it runs promptly when it wakes).

**4. Gaming MLFQ without Rule 4's accounting.**
If demotion happens only when a job uses a *whole* time slice, a program can run for 99%
of its slice, then make a trivial blocking call (a tiny I/O) just before the slice ends.
It never uses a full slice, so it's never demoted, and keeps top priority while being
CPU-bound. The rewritten Rule 4 counts the *total* CPU time used at a level, however it's
split across slices; once that allotment is used, the job moves down.

**5. A batch job hasn't run for an hour.**
Starvation. The fix within priority scheduling is aging: raise its priority the longer it
waits. Fair schedulers don't have the problem by construction: every runnable task in the
fair class accumulates a claim to CPU time proportional to its weight, and the one furthest
behind its share is picked. A low weight (high nice value) means a *smaller share*, never
zero.

**6. Container with a 2-CPU limit, `GOMAXPROCS=32`.**
The limit is enforced as a quota per 100 ms period: 200 ms of CPU time per period. With 32
threads running Go code in parallel, that quota can be used up in about 6 ms of wall-clock
time. Then every thread in the container is throttled until the next period. A request
that arrives during the throttled stretch waits up to ~94 ms before any of its code runs.
Average CPU looks fine; tail latency has regular spikes at the period boundaries. Matching
`GOMAXPROCS` to the limit (Go 1.25+ does this by default, for modules whose `go.mod`
declares `go 1.25` or later) spreads the same work evenly across the period.

---

## File 4 — Synchronisation and deadlock

**1. Lost increments, and why an atomic counter isn't enough for withdrawals.**
`counter++` is load, add, store. If two threads both load 41 before either stores, both
store 42 and one increment is lost. An atomic fetch-and-add makes the three steps one
indivisible instruction, so it fixes the counter. "Check the balance, then withdraw" is two
*separate* operations on shared state: both threads can check (balance 100, withdrawing 80
looks fine), then both withdraw. Each step is atomic, the sequence isn't. You need a lock
around the whole check-and-act, or a single atomic compare-and-swap loop, or a
conditional update in the database (`UPDATE … SET bal = bal - 80 WHERE bal >= 80`).

**2. Lock ordering.**
It breaks **circular wait**. If every thread acquires A before B, a thread holding B can
never be waiting for A (it would have taken A first). So the "holds B, wants A" edge can't
exist, the cycle can't form, and because all four Coffman conditions are necessary,
deadlock is impossible, not just unlikely.

**3. `if` instead of `while` around `Wait`.**
The queue is empty; consumers C1 and C2 are both in `Wait`. The producer pushes one item
and signals. C1 wakes, but before it re-acquires the lock, C2 (or a third consumer that
just arrived and never waited) takes the lock, sees the item, pops it and releases. C1 now
gets the lock, and because it used `if`, it doesn't recheck: it pops from an empty queue.
Go's `Wait` doesn't wake spuriously, but that doesn't help: C1 *was* legitimately woken. The
condition was true when it was signalled and false by the time it ran. That's Mesa
semantics, and the loop is the only defence.

**4. P3 requests (0,0,1).**
Pretend to grant it: free goes from (1,1,1) to (1,1,0); P3's allocation becomes (0,1,2) and
its need (1,1,0). Safety check: P1 needs (1,1,0) ≤ (1,1,0), finishes, free = (2,1,1). P3
needs (1,1,0) ≤ (2,1,1), finishes, free = (2,2,3). P2 needs (2,2,0), finishes, free =
(3,3,4). P0 needs (3,1,2), finishes. All four can finish, so the state is safe: **grant**.

**5. Endpoints hang; no deadlock reported.**
A deadlock between goroutines on mutexes (or channels), most likely two code paths taking
the same locks in different orders. Evidence: a goroutine dump from
`/debug/pprof/goroutine?debug=2` showing goroutines stuck in `sync.Mutex.Lock` for many
minutes, with stack traces showing which lock each holds and which it wants; their number
growing over time. The runtime didn't detect it because its check fires only when *every*
goroutine is blocked, and a server always has goroutines waiting on the network.

**6. Priority inheritance needs an owner.**
To boost "the thread holding the lock", the system must know which thread that is.
A mutex with ownership records the owner when it's locked. A semaphore has no owner: any
thread may signal it, so there's nobody to boost. Go's `sync.Mutex` is explicitly "not
associated with a particular goroutine", and Go has no goroutine priorities anyway, so
there's nothing to inherit and no owner to inherit it.

---

## File 5 — Memory

**1. Translate `0x5123`.**
With 4 KiB pages the offset is the low 12 bits: `0x123`; the VPN is `0x5`. VPN 5 maps to
frame `0x2C`, so the physical address is `(0x2C << 12) | 0x123 = 0x2C123`. If the present
bit were clear, the MMU would raise a page fault; the kernel would decide whether it's a
minor fault (map a page), a major one (read it from disk), or invalid (`SIGSEGV`).

**2. Why no flat page table on 64-bit.**
48-bit addresses with 4 KiB pages leave 36 bits of page number: 2³⁶ entries × 8 bytes =
2³⁹ bytes = 512 GiB per process, mostly describing unused memory. A multi-level table only
allocates the lower-level tables for regions actually in use; an unused 512 GiB region
costs one empty slot in a higher-level table.

**3. Array walk vs scattered linked list.**
The array is contiguous: consecutive elements share cache lines and pages, so each TLB
miss is followed by about a thousand hits (4 KiB of 4-byte integers), and the hardware
prefetcher can predict the next lines. The linked list's nodes are scattered: each step can
land on a different page (a TLB miss, perhaps a page-table walk) and a different cache line
(a cache miss), and the next address isn't known until the current node is loaded, so
nothing can be fetched ahead. Same arithmetic; the list spends its time waiting on memory.

**4. `1 2 3 1 4 1 2 5`, 3 frames.**

```
FIFO: *1[1] *2[12] *3[123]  1[123] *4[234] *1[341] *2[412] *5[125]    → 7 faults
LRU:  *1[1] *2[12] *3[123]  1[231] *4[314]  1[341] *2[412] *5[125]    → 6 faults
```

(Frames listed oldest first for FIFO, least recently used first for LRU.) The difference
is the reference to 4: FIFO evicts 1 because it arrived first, though it was just used;
LRU evicts 2, so the next reference to 1 is a hit.

**5. Why Belady's anomaly can't happen under LRU.**
LRU is a stack algorithm: with *n* frames it holds the *n* most recently used pages, and
with *n + 1* it holds the *n + 1* most recently used. The first set is always a subset of
the second. So any reference that hits with *n* frames also hits with *n + 1*, and faults
can only go down. FIFO's contents depend on arrival order, and the set held with *n + 1*
frames isn't necessarily a superset of the set with *n*, as the §5.6 trace shows.

**6. Returning `&local` in Go and C++.**
Go is correct. Escape analysis sees the address outlives the call and allocates the value
on the heap (`moved to heap: p`). The cost is a heap allocation and garbage-collection work
later. C++ is undefined behaviour: the local lives in the function's stack frame, which is
reused as soon as the function returns, so the pointer dangles. Clang only warns
(`-Wreturn-stack-address`); the program compiles and may even appear to work until the
stack is reused.

---

## File 6 — The relational model, keys and normalisation

**1. Superkeys that aren't candidate keys.**
`{id, name}` and `{email, dept_id}` (or `{id, email}`): each is unique because it contains
a unique column. `{id, email}` isn't a candidate key because it isn't minimal: remove
`email` and `{id}` is still unique.

**2. `order_line(order_id, product_id, quantity, product_name, unit_price)`.**
It violates **2NF**: `product_id → product_name` (and `→ unit_price`, if that means the
catalogue price) depends on only part of the composite key. Fix: move `product_name` into a
`product` table keyed by `product_id`. But keeping `unit_price` on the order line can be
right: the price *at the time of the order* is a fact about the order line, not about the
product, and must not change when the catalogue price does. Then it's not a dependency on
`product_id` at all, and it isn't a violation. Asking "what is this a fact about?" settles
it.

**3. Single-column keys and 2NF vs 3NF.**
2NF forbids depending on *part* of a key, and a one-column key has no proper part. 3NF
forbids depending on a non-key column, which a single-column key can't prevent: in
`employee(id, dept_id, dept_name)`, `id → dept_id → dept_name` is transitive, so it violates
3NF.

**4. The tutoring example.**
Dependencies: `{student, subject} → tutor` and `tutor → subject`. Candidate keys are
`{student, subject}` and `{student, tutor}`, so every attribute is prime and 3NF's loophole
(a dependency whose right side is prime) admits `tutor → subject`. BCNF has no loophole:
`tutor` isn't a superkey, so it's violated. After decomposing to `(tutor, subject)` and
`(student, tutor)`, the rule "a student has at most one tutor per subject" spans both
tables; no key on either can enforce it. You'd need a trigger, or accept 3NF.

**5. `department LEFT JOIN employee`.**
6 rows: Eng × 3, Sales × 2, and Legal once with NULLs on the employee side. (Dev isn't
included: he has no department, and the left side is `department`.) With a second
`id = 1` row in `department`, each Eng employee matches twice: 3 more rows, 9 in total.
Both counts were checked in PostgreSQL.

**6. LEFT JOIN returning fewer rows than the left table.**
The join itself never can: every left row appears at least once. Something after it is
removing rows. Usually a `WHERE` condition on a right-table column, which is false or
unknown for unmatched rows (NULLs), turning the join into an inner join (§6.6's trap; the
reproduced example drops from 6 rows to 3). Other causes: a `WHERE` on left-table columns,
or a `DISTINCT`/`GROUP BY` collapsing rows.

---

## File 7 — Transactions, isolation and indexes

**1. Consistency.**
The database can enforce only what it's told: keys, foreign keys, `CHECK` and unique
constraints. Business invariants it can't see are the application's job. Example: "the sum
of all account balances changes only through deposits and withdrawals", or "a doctor's
shifts don't overlap". Atomicity and isolation give your code the tools to preserve them;
they don't know what the invariants are.

**2. Non-repeatable read vs phantom; PostgreSQL's Repeatable Read.**
Non-repeatable: you read order 7's status as `pending`, someone commits a change, you read
order 7 again and see `paid`. Phantom: you count `WHERE status = 'pending'` (5 rows),
someone inserts a pending order and commits, you count again and get 6. PostgreSQL's
Repeatable Read gives the whole transaction one snapshot, taken at its first statement, so
*every* read, of rows or of sets, sees the database as of that moment. A newly committed
row isn't in the snapshot, so phantoms can't appear either. The standard's locking-based
definition only had to keep read rows stable; a snapshot naturally does more.

**3. The two deposits.**
`r1[bal] … r2[bal] … w2[bal] … c2 … w1[bal] … c1`: T1 reads, T2 reads and writes and
commits, T1 writes (based on its stale read) and commits. That contains P4's
`r1[x]…w2[x]…w1[x]…c1`. For a wallet, the **atomic update** (`SET bal = bal + $1`, with a
`WHERE bal >= $amount` for debits and a check on the affected-row count) is simplest and
holds no locks across application code. If the logic can't be expressed in one statement,
`SELECT … FOR UPDATE` on the wallet row, locking rows in a consistent order (by ID) to avoid
deadlocks.

**4. Snapshot isolation, lost updates and write skew.**
First-committer-wins checks whether two concurrent transactions *wrote the same item*. A
lost update is exactly that (both write `bal`), so the second committer is aborted. Write
skew involves writes to *different* items (Alice's row, Bob's row) based on reads of
overlapping data. There's no write-write conflict, so nothing is checked and both commit.
SI checks write sets against write sets; it never checks one transaction's writes against
another's reads.

**5. "We're on Repeatable Read, so no anomalies."**
The on-call example (§7.5) was reproduced on PostgreSQL 17 at Repeatable Read: both doctors
went off call, both transactions committed, and nobody was left. Making it safe: run those
transactions at Serializable and retry on serialisation failures; or make the conflict
visible to the database, e.g. by locking the rows you read (`SELECT … FOR UPDATE` on both
doctors), or by having both transactions update a shared row (a per-shift counter), which
turns write skew into a write-write conflict.

**6. Index on `(merchant_id, created_at)`.**
(a) Yes, ideally: equality on the leading column, then the index is already in
`created_at` order within merchant 7, so it reads the last 20 entries and stops. No sort.
(b) Generally no: `created_at` isn't the leading column, so matching entries are spread
across every merchant's section. The planner may scan the whole index or the table. A
separate index on `created_at` would serve it. (c) Partly: the index finds merchant 7's
rows by the leading column; `status` isn't in it, so each is fetched and checked. Fine if
the merchant has few rows; if it has millions and few are failed, an index that includes
`status` (or a partial index on failed rows) is better.

---

## File 8 — The SQL interviews ask

**1. Alias in `WHERE` vs `ORDER BY`.**
`WHERE` is evaluated at step 2, before `SELECT` (step 5) has computed `annual`, so the name
doesn't exist yet: `ERROR: column "annual" does not exist`. `ORDER BY` runs at step 7,
after `SELECT`, so the alias is available. Repeat the expression in `WHERE`, or compute it
in a subquery.

**2. `ORDER BY salary DESC LIMIT 1 OFFSET 1`.**
First, NULLs: in PostgreSQL they sort first under `DESC`, so on this data it returns 200,
the highest salary (Dev's NULL is position 1). Second, ties: it counts positions, not
distinct salaries, so if two people earn the top salary it returns that top salary again.
Fix: `WHERE salary IS NOT NULL` and `DISTINCT`, or `dense_rank()`.

**3. Zoya's rank, and "the three highest salaries".**
`rank()` gives Zoya 5 (Asha 1, Ravi and Meera 2, Kabir 4); `dense_rank()` gives 4. For
"people with the three highest salaries", use `dense_rank() <= 3`: the distinct salaries
200, 150, 120, which returns **four** people (Asha, Ravi, Meera, Kabir). Checked in
PostgreSQL. If the question meant "the top three people", ask how to break the tie between
Ravi and Meera.

**4. `NOT IN` vs `NOT EXISTS` for Legal.**
`3 NOT IN (1, 1, 1, 2, 2, NULL)` expands to `3<>1 AND 3<>1 AND 3<>1 AND 3<>2 AND 3<>2 AND
3<>NULL` = `true AND … AND true AND unknown` = **unknown**. `WHERE` keeps only true rows, so
Legal is dropped, and so is every other department. `NOT EXISTS` asks whether any employee
row has `dept_id = 3`. None does, NULL rows simply don't match, the subquery is empty, and
`NOT EXISTS` is true.

**5. Department with the highest average salary.**

```sql
SELECT dept_id, avg_salary FROM (
  SELECT dept_id, avg(salary) AS avg_salary,
         rank() OVER (ORDER BY avg(salary) DESC) AS r
  FROM employee WHERE dept_id IS NOT NULL
  GROUP BY dept_id
) t WHERE r = 1;
-- 1 | 166.6666666666666667
```

The window function runs after `GROUP BY`, so it can rank the aggregates. With `rank()`,
tied departments all come back; `ORDER BY avg(salary) DESC LIMIT 1` would silently pick
one.

**6. Salary with department average, two ways.**

```sql
-- window
SELECT name, salary, avg(salary) OVER (PARTITION BY dept_id) AS dept_avg FROM employee;
-- group + join
SELECT e.name, e.salary, a.dept_avg
FROM employee e
JOIN (SELECT dept_id, avg(salary) AS dept_avg FROM employee GROUP BY dept_id) a
  ON a.dept_id = e.dept_id;
```

The window version reads better: one pass, no self-join. They differ on Dev: the window
version returns 6 rows, with Dev in a partition of his own (NULL `dept_id`) and a NULL
average; the join version returns 5, because `a.dept_id = e.dept_id` is unknown when both
are NULL, so Dev has no match. Both outputs were checked in PostgreSQL.

---

## File 9 — Networks: layers, TCP and UDP

**1. Which addresses change at each hop.**
The link-layer (MAC) addresses change: each hop re-wraps the packet in a new frame
addressed from that router to the next device. The IP addresses stay the same end to end,
because they identify the original source and final destination. (NAT is the exception:
a NAT router rewrites the source IP and port.)

**2. TLS in the OSI model.**
Conventionally the presentation layer (6), since it handles encryption, or "between 4 and
7". Neither fits: TLS runs on top of TCP, is invoked by the application, handles both
encryption and authentication, and carries its own record framing and handshake. OSI's
layers were designed for OSI's protocols, and TLS isn't one of them.

**3. Two writes arrive as one read.**
Not a bug. TCP delivers a byte stream with no message boundaries: the sender's writes may
be coalesced or split in any way. The application protocol must frame its own messages,
with a length prefix, a delimiter (like a newline), or a header such as `Content-Length`,
and the reader must loop until it has a whole message.

**4. Why three messages.**
Each side's initial sequence number has to be acknowledged by the other. SYN announces the
client's; SYN-ACK acknowledges it and announces the server's; only the final ACK confirms
the server's. With two messages, the server would treat any SYN as a connection, including
an old duplicate SYN delayed in the network from a previous connection, and would allocate
state and send data to a client that never asked. The third message proves the client is
live and agrees on both sequence numbers.

**5. "Cannot assign requested address" at 2,000 new connections per second.**
Each closed connection leaves the client's socket in TIME-WAIT for 60 seconds on Linux,
holding that (source IP, source port, destination IP, destination port) combination. To one
upstream, only the source port varies, so the client can have at most as many connections
as it has ephemeral ports (tens of thousands by default). At 2,000 per second, 60 seconds of
TIME-WAIT is 120,000 combinations in use, more than there are ports. The fix is to reuse
connections: HTTP keep-alive with a pooled client (in Go, one shared `http.Client`), not
shortening TIME-WAIT.

**6. Slow phone vs congested link.**
Slow phone, clear network: **flow control**. The phone's receive buffer fills, it
advertises a small `rwnd`, and `rwnd` is the smaller of the two windows. Fast client,
congested link: **congestion control**. Losses shrink `cwnd`, which becomes the binding
limit. Either way the sender is limited by min(`cwnd`, `rwnd`).

---

## File 10 — What happens when you type a URL

**1. A cold lookup of `shop.example.in`.**
The browser checks its cache, then asks the OS, which asks the configured recursive
resolver. The resolver, with an empty cache, asks a root server, which refers it to the
`.in` servers; they refer it to the authoritative servers for `example.in`; those either
answer for `shop.example.in` or (if it's delegated further) refer again. Only the
authoritative server knows the answer. The root and TLD servers only know who to ask next.

**2. SNI.**
One IP address often serves many sites with different certificates, and the server must
pick the right certificate before encryption can begin, because the certificate is part of
setting it up. So the name has to travel in the ClientHello, before any keys exist. A
network observer can therefore see the hostname you're connecting to, though not the path,
headers or content.

**3. Round trips before the first byte.**
Cold connection, cached DNS, TLS 1.3: TCP (1) + TLS (1) + request/response (1) = 3. Existing
HTTP/2 connection: 1. Connection setup is two-thirds of the cost, so connection reuse
(keep-alive, pools, one connection per host with HTTP/2) and fewer distinct hosts usually
buy more than shaving server processing time, particularly over long round trips.

**4. Duplicate charges on a retried POST.**
POST isn't idempotent under RFC 9110: sending it twice may have two effects. A timeout
means the client doesn't know whether the first attempt was processed; it may have
succeeded and only the response been lost. Retrying blindly charges twice. The fix is an
**idempotency key**: the client generates a unique key per logical operation and sends it
with every attempt; the server stores the key with the result (in the same transaction as
the charge) and, on seeing a key again, returns the stored result instead of charging
again.

**5. Disabled certificate verification.**
Lost: authentication. The client still encrypts, but to whoever answers. An attacker on
the path (a malicious Wi-Fi hotspot, a compromised router, DNS spoofing) answers the
connection with their own certificate. The client accepts it, completes the handshake with
the attacker, and the attacker opens a second connection to the real server, relaying and
reading everything, credentials included. Both legs are "encrypted".

**6. L7 balancer terminating TLS.**
It can route by HTTP content (host, path, header), and it can do HTTP-level work: retries
of idempotent requests, header injection, rate limiting per route, caching, compression.
An L4 balancer sees only TCP connections. What your service loses: the client's real IP
address as the TCP peer (the peer is now the balancer; the original arrives in a header
such as `X-Forwarded-For`, which you must trust only from the balancer), and the TLS
details of the client connection.

---

## File 11 — Object-oriented programming

**1. Encapsulation vs abstraction with `BankAccount`.**
A private `balance` with only `Deposit` and `Withdraw` exposed is **encapsulation**: no
code outside can set the balance to a negative number, because every change goes through
methods that check. An `AccountStore` interface with `Load` and `Save` is **abstraction**:
callers use it without knowing whether it's Postgres or memory. One protects invariants,
the other hides implementation.

**2. Embedding and substitutability.**
You can't pass an `OrderService` where a `Logger` is required; the compiler rejects it,
because embedding doesn't make `OrderService` a `Logger`. You can pass it where an
*interface* with `Log(string)` is required, because the promoted `Log` method is in
`OrderService`'s method set and interfaces are satisfied by method sets. Concrete types
are never interchangeable in Go; interfaces are how you get substitutability.

**3. `Square extends Rectangle`.**
Test: create a rectangle, set width 5, set height 4, assert area 20. For a `Rectangle` it
passes. For a `Square`, setting the height also sets the width (or the reverse), so the
area is 16 (or 25), and the test fails. The subtype broke a promise the supertype makes:
setting one dimension leaves the other alone. Model them instead as separate immutable
types implementing a common `Shape` interface with `Area()`, with no mutable setters to
disagree about.

**4. Non-virtual method through a base pointer.**
In C++ the *base* version runs: non-virtual calls are bound at compile time from the
pointer's static type, and the subclass's function merely hides it. In Go there's no
subclassing: a method called through an interface always runs the concrete type's method;
a method promoted from an embedded type can be shadowed by the outer type, but code calling
the embedded value directly runs the inner one. In Rust, trait methods called through
`dyn Trait` or a generic run the implementing type's version. In JavaScript, every call is
looked up at run time along the prototype chain, so the subclass's method runs.

**5. Business logic calls `postgres.NewClient()`.**
**Dependency inversion**: high-level logic depends on a concrete low-level module.
Minimal change: define a small interface where the logic uses it (say
`type OrderStore interface { Save(ctx, Order) error }`), have the logic take an
`OrderStore` as a parameter or field, and construct the Postgres implementation in
`main`. The Postgres client satisfies the interface implicitly; the test passes an
in-memory implementation.

**6. Unlocked nil check before the lock.**
Reading `instance` without the lock while another goroutine writes it is a data race: the
Go memory model gives no guarantee that a non-nil pointer seen without synchronisation
points to a fully initialised value. That's the double-checked locking bug. And two
goroutines can both see nil and both take the slow path, so the second must check again
under the lock. `sync.OnceValue` does all of this correctly: the function runs exactly
once, concurrent callers wait for it to finish, and every caller sees the fully
initialised result.

---

## Things to try

1. **Watch a race and its fix.** Run §4.1's Go program with and without `-race`, then
   with `sync/atomic`. Run the C++ version with `-fsanitize=thread`. Try to write the Rust
   version without `Mutex` and read the compiler's errors until they make sense.
2. **Trace system calls.** In a Linux container, `strace -c` on `cat`, on a Go hello-world
   and on `curl https://example.com`. Compare counts, and find `socket`, `connect` and the
   reads and writes in the curl trace.
3. **Count goroutines and threads.** Repeat §2.6's 100,000-goroutine measurement, then
   make each goroutine do a blocking file read instead of sleeping, and watch the thread
   count change.
4. **Reproduce every isolation result.** Two `psql` sessions against a throwaway
   PostgreSQL. Do the lost update at Read Committed, the write skew at Repeatable Read, then
   the same at Serializable. Add `SELECT … FOR UPDATE` and watch the second session block.
5. **Simulate page replacement.** Write a 30-line FIFO/LRU/OPT simulator and find your own
   string that shows Belady's anomaly.
6. **Walk DNS yourself.** `dig +trace` a domain you use at work, and note which TTLs are
   long and which are short, and why.
7. **Break a handshake.** Run a TCP server, connect with `nc`, then watch the states with
   `ss -tan` (Linux) or `netstat -an` as you close each side, and find the TIME-WAIT entry.

## Questions worth asking me

- "Interview me on processes and threads for ten minutes, increasing the depth until I
  get one wrong."
- "Here's my answer to 'what happens when you type a URL'. Where would an interviewer push
  next?"
- "Give me five page-replacement and scheduling numericals of the kind asked in Indian
  campus and lateral interviews, and check my working."
- "How does PostgreSQL's Serializable actually detect write skew? Walk me through SSI's
  rw-dependencies."
- "Compare Go's scheduler with Tokio's. Where do they make different trade-offs?"
- "What's the C++ memory model, and what do `memory_order_acquire` and `release` buy you
  over sequential consistency?"
- "Design a schema for a ride-sharing app and normalise it to 3NF. Then tell me where
  you'd denormalise and why."
- "What did this course leave out that interviewers at product companies still ask?
  (File systems, I/O models like epoll, virtual memory in containers?)"
