# 12. Glossary

Reference, not reading. Each entry points at the chapter that explains it.

| Term | Meaning | Where |
|---|---|---|
| **1:1 / N:1 / M:N threading** | How many kernel threads carry a language's threads: one each, all on one, or many multiplexed onto fewer | [02](02-processes-and-threads.md) |
| **1NF, 2NF, 3NF** | Normal forms: atomic values; no partial dependency on a composite key; no transitive dependency through a non-key column | [06](06-relational-model-and-normalisation.md) |
| **ACID** | Atomicity, consistency, isolation, durability: what a transaction promises | [07](07-transactions-and-isolation.md) |
| **Aging** | Raising a waiting job's priority over time so it can't starve | [03](03-cpu-scheduling.md) |
| **AIMD** | Additive increase, multiplicative decrease: TCP's congestion-control sawtooth | [09](09-networks-layers-and-tcp.md) |
| **ALPN** | TLS extension in which client and server agree an application protocol (h2, http/1.1) during the handshake | [10](10-what-happens-when-you-type-a-url.md) |
| **Anomaly (update, insertion, deletion)** | Inconsistencies caused by storing a fact in more than one place, or in a table about something else | [06](06-relational-model-and-normalisation.md) |
| **ARP** | Finds the MAC address for an IP address on the local network | [09](09-networks-layers-and-tcp.md) |
| **B+ tree** | Balanced, high-fan-out tree with all keys in linked leaves; the usual index structure | [07](07-transactions-and-isolation.md) |
| **Banker's algorithm** | Deadlock avoidance: grant a request only if a safe completion order still exists | [04](04-synchronisation-and-deadlock.md) |
| **BCNF** | Boyce–Codd normal form: every determinant is a superkey | [06](06-relational-model-and-normalisation.md) |
| **Belady's anomaly** | More frames causing more page faults, possible under FIFO | [05](05-memory.md) |
| **Candidate key** | A minimal set of columns that uniquely identifies a row | [06](06-relational-model-and-normalisation.md) |
| **CFS** | Completely Fair Scheduler: Linux's fair scheduler from 2.6.23 until EEVDF replaced it in 6.6 | [03](03-cpu-scheduling.md) |
| **Clustered index** | An index whose leaves hold the rows themselves, in key order (InnoDB's primary key) | [07](07-transactions-and-isolation.md) |
| **Composition** | Building behaviour from objects you hold and delegate to, rather than inherit from | [11](11-object-oriented-programming.md) |
| **Condition variable** | Lets a thread sleep until another signals that some state may have changed; always waited on in a loop | [04](04-synchronisation-and-deadlock.md) |
| **Congestion control** | Sender-side limit (`cwnd`) protecting the network | [09](09-networks-layers-and-tcp.md) |
| **Context switch** | Saving one thread's CPU state and restoring another's | [02](02-processes-and-threads.md) |
| **Convoy effect** | Short jobs stuck behind a long one under FCFS | [03](03-cpu-scheduling.md) |
| **Copy-on-write** | Sharing pages read-only and copying one only when it's written; makes `fork` cheap | [05](05-memory.md) |
| **Critical section** | Code touching shared state that must not run concurrently with itself | [04](04-synchronisation-and-deadlock.md) |
| **Data race** | Two unsynchronised accesses to the same memory, at least one a write | [04](04-synchronisation-and-deadlock.md) |
| **Deadlock** | A cycle of threads each waiting for a resource another holds | [04](04-synchronisation-and-deadlock.md) |
| **Demand paging** | Giving a page physical memory only when it's first touched | [05](05-memory.md) |
| **Dense rank** | Ranking where ties share a rank and no numbers are skipped | [08](08-sql-interviews-ask.md) |
| **Dirty read / dirty write** | Reading, or overwriting, another transaction's uncommitted change | [07](07-transactions-and-isolation.md) |
| **DNS** | The distributed, cached system mapping names to addresses | [10](10-what-happens-when-you-type-a-url.md) |
| **Dynamic dispatch** | Choosing which method runs at run time, from the object's actual type | [11](11-object-oriented-programming.md) |
| **EEVDF** | Earliest Eligible Virtual Deadline First: Linux's fair scheduler since 6.6 | [03](03-cpu-scheduling.md) |
| **Embedding (Go)** | Including a type in a struct so its methods are promoted, without creating a subtype | [11](11-object-oriented-programming.md) |
| **Encapsulation** | Hiding an object's data behind operations that keep it valid | [11](11-object-oriented-programming.md) |
| **Escape analysis** | The Go compiler's decision whether a value can live on the stack or must move to the heap | [05](05-memory.md) |
| **Event loop** | A single thread running callbacks one at a time as events arrive (JavaScript) | [02](02-processes-and-threads.md) |
| **FCFS** | First-come, first-served scheduling | [03](03-cpu-scheduling.md) |
| **File descriptor** | Small integer naming a process's open file, socket or pipe | [01](01-what-happens-when-a-program-runs.md) |
| **First-committer-wins** | Snapshot isolation's rule: of two concurrent writers to one item, the first to commit wins and the other aborts | [07](07-transactions-and-isolation.md) |
| **Flow control** | Receiver-advertised limit (`rwnd`) protecting the receiver | [09](09-networks-layers-and-tcp.md) |
| **Foreign key** | Columns that must match a key in another table | [06](06-relational-model-and-normalisation.md) |
| **fork / exec / wait** | Copy a process; replace its program; collect a child's exit status | [02](02-processes-and-threads.md) |
| **Functional dependency** | `X → Y`: rows that agree on X agree on Y | [06](06-relational-model-and-normalisation.md) |
| **G, M, P** | Go scheduler: goroutine, OS thread, processor slot (`GOMAXPROCS` of them) | [02](02-processes-and-threads.md) |
| **Head-of-line blocking** | One lost packet stalling unrelated data queued behind it; TCP's, fixed by QUIC | [09](09-networks-layers-and-tcp.md) |
| **HSTS** | A site's instruction to browsers to use HTTPS only | [10](10-what-happens-when-you-type-a-url.md) |
| **Idempotent** | Same effect whether done once or many times (PUT, DELETE, safe methods) | [10](10-what-happens-when-you-type-a-url.md) |
| **Interrupt** | A hardware signal (timer, device) forcing the CPU into the kernel | [01](01-what-happens-when-a-program-runs.md) |
| **Isolation level** | How much concurrent transactions may see of each other | [07](07-transactions-and-isolation.md) |
| **Kernel mode / user mode** | CPU privilege levels; only kernel mode can touch hardware and page tables | [01](01-what-happens-when-a-program-runs.md) |
| **Liskov substitution** | A subtype must be usable anywhere its supertype is, with the same promised behaviour | [11](11-object-oriented-programming.md) |
| **Livelock** | Threads busily reacting to each other without progressing | [04](04-synchronisation-and-deadlock.md) |
| **Lost update** | A write based on a stale read overwriting another transaction's write | [07](07-transactions-and-isolation.md) |
| **LRU** | Evict the least recently used page | [05](05-memory.md) |
| **MLFQ** | Multi-level feedback queue: scheduling that learns from how jobs used the CPU | [03](03-cpu-scheduling.md) |
| **Monitor** | Lock, condition variables and protected data packaged together | [04](04-synchronisation-and-deadlock.md) |
| **MSL** | Maximum segment lifetime; TIME-WAIT lasts 2×MSL by the spec | [09](09-networks-layers-and-tcp.md) |
| **Mutex** | A lock allowing one holder at a time | [04](04-synchronisation-and-deadlock.md) |
| **MVCC** | Multi-version concurrency control: writers create versions, readers read their snapshot | [07](07-transactions-and-isolation.md) |
| **Non-repeatable read** | Rereading a row inside one transaction and getting a different value | [07](07-transactions-and-isolation.md) |
| **OPT / MIN** | Evict the page used furthest in the future; optimal and unimplementable | [05](05-memory.md) |
| **OSI model** | Seven-layer reference model (ISO/IEC 7498-1); a vocabulary more than a design | [09](09-networks-layers-and-tcp.md) |
| **Page / frame** | Fixed-size unit of virtual / physical memory, typically 4 KiB | [05](05-memory.md) |
| **Page fault** | Trap on access to an unmapped or protected page: minor, major or invalid | [05](05-memory.md) |
| **Page table** | Per-process map from virtual pages to physical frames | [05](05-memory.md) |
| **Phantom** | Repeating a query and getting a different set of rows | [07](07-transactions-and-isolation.md) |
| **Polymorphism** | One interface, many implementations | [11](11-object-oriented-programming.md) |
| **Preemption** | Taking the CPU from a running thread, usually on a timer interrupt | [03](03-cpu-scheduling.md) |
| **Priority inheritance** | A lock holder temporarily runs at the priority of its highest waiter | [04](04-synchronisation-and-deadlock.md) |
| **Priority inversion** | A high-priority thread stalled behind a low-priority lock holder | [04](04-synchronisation-and-deadlock.md) |
| **Process** | A program in execution: address space, threads, open descriptors | [02](02-processes-and-threads.md) |
| **Race condition** | A result that depends on uncontrolled timing | [04](04-synchronisation-and-deadlock.md) |
| **Read skew / write skew** | Seeing parts of two different states; two transactions each writing based on what the other changes | [07](07-transactions-and-isolation.md) |
| **Recursive vs iterative DNS** | The server chases the answer for you, versus refers you onward | [10](10-what-happens-when-you-type-a-url.md) |
| **Round robin** | Preemptive turn-taking with a fixed time quantum | [03](03-cpu-scheduling.md) |
| **Safe state** | A state from which every process can still finish in some order | [04](04-synchronisation-and-deadlock.md) |
| **Semaphore** | A counter with atomic wait (P) and signal (V) | [04](04-synchronisation-and-deadlock.md) |
| **Serialisable** | Concurrent execution equivalent to some serial order | [07](07-transactions-and-isolation.md) |
| **SJF / SRTF** | Shortest job first, and its preemptive version | [03](03-cpu-scheduling.md) |
| **SNI** | TLS extension carrying the hostname, so one IP can serve many certificates | [10](10-what-happens-when-you-type-a-url.md) |
| **Snapshot isolation** | Each transaction reads a consistent snapshot; first-committer-wins on writes. Allows write skew | [07](07-transactions-and-isolation.md) |
| **SOLID** | Five design principles: single responsibility, open–closed, Liskov, interface segregation, dependency inversion | [11](11-object-oriented-programming.md) |
| **Spinlock** | A lock that waits by looping rather than sleeping | [04](04-synchronisation-and-deadlock.md) |
| **SSI** | Serializable snapshot isolation: SI plus detection of dangerous read/write patterns (PostgreSQL's Serializable) | [07](07-transactions-and-isolation.md) |
| **Stack vs heap** | Per-call memory freed on return, versus memory living until freed or collected | [05](05-memory.md) |
| **Starvation** | A thread or job never getting a resource it's waiting for | [04](04-synchronisation-and-deadlock.md) |
| **Superkey** | Any set of columns that's unique per row | [06](06-relational-model-and-normalisation.md) |
| **System call** | Controlled entry into the kernel to request a service | [01](01-what-happens-when-a-program-runs.md) |
| **Thrashing** | Spending nearly all time paging because working sets exceed memory | [05](05-memory.md) |
| **Thread** | A stream of execution within a process, sharing its memory | [02](02-processes-and-threads.md) |
| **Three-way handshake** | SYN, SYN-ACK, ACK: TCP connection setup | [09](09-networks-layers-and-tcp.md) |
| **TIME-WAIT** | State the actively closing side holds after a TCP close | [09](09-networks-layers-and-tcp.md) |
| **TLB** | CPU cache of virtual-to-physical translations | [05](05-memory.md) |
| **Two-phase locking** | Acquire locks, then release, never acquire after releasing; guarantees serialisability | [07](07-transactions-and-isolation.md) |
| **vtable** | Per-type table of function pointers implementing dynamic dispatch | [11](11-object-oriented-programming.md) |
| **Window function** | A calculation over related rows that doesn't collapse them (`OVER (…)`) | [08](08-sql-interviews-ask.md) |
| **Working set** | The pages a process used in a recent window of time | [05](05-memory.md) |
| **Zombie / orphan** | Exited child not yet waited for; child whose parent exited first | [02](02-processes-and-threads.md) |
