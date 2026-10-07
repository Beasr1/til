# Core CS Subjects — A Course

Assumes [`interview/01`](../01-how-the-loop-works.md) — the shape of the interview loop.

A course on the four "core subjects" that software engineering interviews in India probe
alongside data structures and system design: **operating systems**, **databases**,
**computer networks** and **object-oriented programming**. It's written for a working
backend engineer who uses these things every day (goroutines, Postgres transactions,
HTTPS calls) and wants to understand them properly, not just recite definitions.

**This is reference learning material.** Everything here is general computer science,
checked against the standard textbooks, the RFCs, the PostgreSQL, Linux and language
documentation, and, wherever behaviour could be observed, reproduced: every SQL query and
isolation-level anomaly was run on PostgreSQL 17, and every code example was compiled and
run. Interview questions appear as a by-product, in a short section at the end of each
chapter. The teaching is the point.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the problem that forced each idea comes before its name.
- Every chapter ends with **Check yourself** questions. Answers are in
  [13-exercises.md](13-exercises.md).
- Where the sources disagree (mutex vs binary semaphore, what the isolation levels mean,
  how many normal forms matter, whether JavaScript classes are "just sugar"), both sides
  are shown. The disagreement is usually the lesson.

**Code is polyglot.** Examples are in Go first, with Rust, C++ and JavaScript versions in
collapsible blocks. Where the languages genuinely differ, the difference is part of the
lesson: how each one maps threads onto the OS ([file 02](02-processes-and-threads.md)),
when each one catches a data race ([file 04](04-synchronisation-and-deadlock.md)), who
decides stack or heap ([file 05](05-memory.md)), and what each one thinks OOP is
([file 11](11-object-oriented-programming.md)).

**On which questions are "common":** each chapter's *Asked in interviews* list comes from
publicly shared candidate interview reports and the standard prep syllabi. That's
anecdotal, not measured. Treat the lists as a reasonable guess at what gets asked, not as
data.

## The one thing to understand first

> **Operating systems, databases and networks are three views of one event: a program
> runs inside a box the kernel built for it, and every time it needs something outside
> the box (memory, a file, the network, another CPU, shared data) it has to ask, and
> wait, and coordinate with everything else asking at the same time.**

Scheduling is how the box shares a CPU. Virtual memory is how it shares RAM. Locks and
transactions are how programs share data without corrupting it. TCP is how two boxes on
different machines share a conversation. Learn the subjects as one story and the
interview questions stop being separate syllabi.

## Reference sources

| Source | What it settles |
|---|---|
| [OSTEP — *Operating Systems: Three Easy Pieces*](https://pages.cs.wisc.edu/~remzi/OSTEP/), Arpaci-Dusseau, v1.10, free | Limited direct execution, MLFQ's rules, locks, semaphores, condition variables, paging |
| [Silberschatz, Galvin, Gagne — *Operating System Concepts*](https://www.os-book.com/OS10/index.html), 10th ed., 2018 | Process states, scheduling metrics, the standard OS syllabus |
| [Silberschatz, Korth, Sudarshan — *Database System Concepts*](https://www.db-book.com/), 7th ed., 2019 | Normal forms, the 3NF/BCNF trade-off, transactions |
| [Kurose & Ross — *Computer Networking: A Top-Down Approach*](https://gaia.cs.umass.edu/kurose_ross/index.php), 9th ed., 2025 | The five-layer teaching model |
| [Linux kernel docs — scheduler](https://docs.kernel.org/scheduler/sched-eevdf.html), [mutex design](https://docs.kernel.org/locking/mutex-design.html), [sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html) | EEVDF and CFS; mutex ownership; scheduling classes |
| [Go runtime source `proc.go`](https://github.com/golang/go/blob/master/src/runtime/proc.go), [`sync`](https://pkg.go.dev/sync), [Go memory model](https://go.dev/ref/mem) | G/M/P scheduling; Mutex without ownership; data races |
| [Rust book ch16–18](https://doc.rust-lang.org/book/), [cppreference](https://en.cppreference.com/), [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript), [libuv](https://docs.libuv.org/en/v1.x/threadpool.html) | Threading models, data races, OOP features per language |
| [PostgreSQL — Transaction Isolation §13.2](https://www.postgresql.org/docs/current/transaction-iso.html) | What each level actually does in PostgreSQL |
| [Berenson et al. — "A Critique of ANSI SQL Isolation Levels"](https://arxiv.org/abs/cs/0701157), SIGMOD 1995 | Dirty writes, lost updates, read and write skew, snapshot isolation |
| [SQL-92 text, §4.28](https://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt) | The standard's own phenomena and levels |
| [Kent — "A Simple Guide to Five Normal Forms"](https://www.bkent.net/Doc/simple5.htm), CACM 1983 | "The key, the whole key, and nothing but the key" |
| [RFC 9293 (TCP)](https://www.rfc-editor.org/rfc/rfc9293.html), [RFC 5681 (congestion control)](https://www.rfc-editor.org/rfc/rfc5681.html), [RFC 1122](https://www.rfc-editor.org/rfc/rfc1122.html) | Handshake, TIME-WAIT, windows, the four-layer model |
| [RFC 1034](https://www.rfc-editor.org/rfc/rfc1034.html) / [1035 (DNS)](https://www.rfc-editor.org/rfc/rfc1035.html), [RFC 8446 (TLS 1.3)](https://www.rfc-editor.org/rfc/rfc8446.html), [RFC 9110 (HTTP semantics)](https://www.rfc-editor.org/rfc/rfc9110.html) | Resolution modes, the handshake and 0-RTT caveats, safe and idempotent methods |
| [Go FAQ](https://go.dev/doc/faq#is-go-an-object-oriented-language), [Effective Go](https://go.dev/doc/effective_go#embedding) | Go's "yes and no" OOP, embedding |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What happens when a program runs](01-what-happens-when-a-program-runs.md) | ⭐ Follow a program from source to exit and use the vocabulary every later chapter assumes: process, thread, system call, interrupt, virtual address, socket. **Prerequisite for everything else** |

### Part 1 — Operating systems

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Processes and threads](02-processes-and-threads.md) | Explain what a process owns that a thread doesn't, what a context switch costs, fork/exec, zombies, and how goroutines, Rust threads, C++ threads and Node differ |
| 3 | [CPU scheduling](03-cpu-scheduling.md) | Compute waiting times under FCFS, SJF and RR, explain MLFQ, and say what Linux actually runs (EEVDF since 6.6) |
| 4 | [Synchronisation and deadlock](04-synchronisation-and-deadlock.md) | ⭐ Explain races, mutexes, semaphores and condition variables, answer "mutex vs binary semaphore" with the nuance, prevent deadlock, and run the banker's algorithm |
| 5 | [Memory](05-memory.md) | Translate an address, explain the TLB and page faults, count faults for FIFO/LRU/OPT, explain Belady's anomaly and thrashing, and say who decides stack or heap in each language |

### Part 2 — Databases

| # | File | After this you can… |
|---|------|---------------------|
| 6 | [The relational model, keys and normalisation](06-relational-model-and-normalisation.md) | Name every kind of key, normalise a table from 1NF to BCNF and explain the 3NF/BCNF trade-off, and predict a join's row count |
| 7 | [Transactions, isolation and indexes](07-transactions-and-isolation.md) | ⭐ Explain ACID, the isolation levels *and why their standard definitions are inadequate*, what PostgreSQL does at each, four fixes for a lost update, and why indexes are B+ trees |
| 8 | [The SQL interviews ask](08-sql-interviews-ask.md) | Write the nth-highest-salary, top-per-group, self-join and duplicate queries correctly with ties and NULLs, and explain why WHERE can't see aggregates |

### Part 3 — Networks

| # | File | After this you can… |
|---|------|---------------------|
| 9 | [Networks: layers, TCP and UDP](09-networks-layers-and-tcp.md) | Compare OSI and TCP/IP honestly, choose TCP or UDP, explain the handshake, TIME-WAIT, and flow vs congestion control |
| 10 | [What happens when you type a URL](10-what-happens-when-you-type-a-url.md) | ⭐ Tell the whole story from URL to rendered page, go deep on DNS and TLS on request, and count the round trips |

### Part 4 — Object-oriented programming

| # | File | After this you can… |
|---|------|---------------------|
| 11 | [Object-oriented programming](11-object-oriented-programming.md) | Explain the four pillars and how C++, Go, Rust and JavaScript each provide them, argue composition over inheritance, apply SOLID in Go, and write a thread-safe singleton in four languages |

### Reference

| # | File | |
|---|------|---|
| 12 | [Glossary](12-glossary.md) | Terms, each linked to its chapter |
| 13 | [Exercises & answers](13-exercises.md) | Worked answers to every Check yourself question, things to try, and questions to ask |

Read chapter 1 first. After that the four parts are independent of each other, with two
cross-links worth following: chapter 7 builds on chapter 4's locks and deadlocks, and
chapter 10 builds on chapter 9's TCP. Within Part 1, read 2 before 3 and 4.

Where another course in this repo already teaches something in depth, these chapters
teach the interview-level core and link out: PostgreSQL's MVCC, WAL, locks and indexes to
the [postgres course](../../postgres/README.md); HTTP versions, ALPN and certificates to the
[http course](../../http/README.md); goroutine leaks and `GOMAXPROCS` to the
[go-services course](../../go-services/README.md).

Not covered, and asked at some companies: file systems (inodes, journaling), I/O models
(blocking, non-blocking, epoll), and computer architecture (caches, pipelining). The
questions at the end of file 13 are the honest list of gaps.

## If you're short on time

- **Interview tomorrow, one hour:** file 01 §1.10 (vocabulary), then the *Asked in
  interviews* list at the end of every chapter, following the pointers only for the
  questions you can't answer.
- **The OS round:** files 02, 04, 05. If time remains, 03 §3.2–§3.4 for the numericals.
- **The DBMS round:** file 07 §7.3–§7.6, then file 08 §8.5 (nth highest salary) and
  §8.3 (NULLs), then file 06 §6.5 for normalisation.
- **The "what happens when you type a URL" question:** file 10, then file 09 §9.5 for the
  handshake if they push on TCP.

## The one-paragraph summary of everything

A program runs on the bare CPU in user mode, inside an address space the kernel built,
and enters the kernel only through system calls, interrupts and faults. A **process** owns
that address space; its **threads** share it, so they're cheap to create and dangerous to
coordinate. The kernel time-slices threads onto CPUs with a **scheduler** that tries to get
shortest-job-first's results without knowing the future (MLFQ by learning, Linux's EEVDF
by fair shares), while Go multiplexes goroutines onto a few threads of its own, Rust and
C++ use one OS thread per thread, and JavaScript runs one thread per event loop. Shared
memory makes **race conditions**; locks, semaphores and condition variables prevent them,
and their misuse makes **deadlock**, which needs mutual exclusion, hold-and-wait, no
preemption and a circular wait, so a global lock order prevents it. Every address a program
uses is **virtual**, translated through multi-level page tables and cached in the TLB;
pages arrive on first touch, page replacement approximates LRU because the optimum needs
the future, and thrashing is a working set bigger than memory. A relational database
stores each fact once, in the table whose key it's about (**normalisation**, 3NF in
practice), and wraps changes in **transactions** whose isolation levels the SQL standard
defined ambiguously: real databases implement snapshot isolation and its relatives,
PostgreSQL's Repeatable Read prevents phantoms but allows write skew, its default Read
Committed allows lost updates, and only Serializable prevents both. Interview SQL is mostly
about evaluation order, NULLs and window functions. Networks are layered (OSI is the
vocabulary, TCP/IP the reality); TCP builds a reliable, ordered byte stream over
best-effort IP with a three-way handshake, retransmission, flow control for the receiver
and congestion control for the network. Loading a URL is DNS, then TCP, then TLS, then
HTTP, and reusing a connection skips all but the last. OOP is encapsulation, abstraction,
inheritance and polymorphism by convention; Go and Rust keep everything but inheritance,
separating code reuse (embedding) from substitutability (interfaces and traits), which is
the cleanest argument for composition over inheritance.

## How to use me

Ask anything, including:

- "Explain the difference between a process and a thread again, but through what the
  kernel stores"
- "Give me a page-replacement numerical and check my answer"
- "Is my answer to 'mutex vs semaphore' good enough? Here it is…"
- "What does MySQL do differently from PostgreSQL at Repeatable Read?"
- "Walk me through the TLS 1.3 handshake message by message"
- "Write this chapter's Go example in Rust and explain what the borrow checker is
  objecting to"
- "Interview me on file 07 and stop when I get something wrong"

I'll add files here as we go if a topic earns its own page.
