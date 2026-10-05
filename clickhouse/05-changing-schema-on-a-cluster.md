# 5. Changing schema on a cluster

## The problem

On one server, a schema change is a statement. On a cluster, it's a statement plus three
questions: which servers does it run on, which servers does its effect reach, and what does
the migration runner believe happened. Get one wrong and you have a column on one replica
and not the other, a "replicated" table whose copies never talk to each other, or a runner
that refuses to start after someone tidied up the migration folder. Most of this course's
other surprises come from reading. This one comes from deploying.

Assumes [chapter 1](01-what-clickhouse-does-with-a-table.md) (replicas, shards, Keeper,
`ON CLUSTER`). For migration runners in general, what they store and why an applied file is
frozen, read [postgres/10](../postgres/10-migration-runners.md) first; this chapter adds what's
different on ClickHouse. Measured on a two-replica, one-shard ClickHouse 26.9 cluster with
embedded Keeper, in throwaway containers. Names are invented.

## Two documents, two mechanisms

The documentation describes this in two places that read differently.

The replication page
([Data replication](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replication)):
"Compressed data for `INSERT` and `ALTER` queries is replicated", while "`CREATE`, `DROP`,
`ATTACH`, `DETACH` and `RENAME` queries are executed on a single server and are not
replicated."

The distributed DDL page
([ON CLUSTER](https://clickhouse.com/docs/sql-reference/distributed-ddl)): "By default, the
`CREATE`, `DROP`, `ALTER`, and `RENAME` queries affect only the current server where they are
executed."

They disagree about `ALTER`, and the disagreement is resolved by the table engine. Measured
(a dash means that combination wasn't tested):

| Statement, run on replica 1 only | Replicated table | Plain MergeTree |
|---|---|---|
| `ALTER TABLE … ADD INDEX` | appeared on replica 2 | — |
| `ALTER TABLE … ADD COLUMN` | — | replica 2 unchanged |
| `INSERT` | rows on replica 2 | replica 2 still has 0 rows |
| `RENAME TABLE` | replica 2 kept the old name | — |

So: on a replicated table, `ALTER` travels to the other replicas of the shard through
Keeper, with or without `ON CLUSTER`. `CREATE`, `DROP` and `RENAME` never travel; they need
`ON CLUSTER`, or a run on every server. On a plain MergeTree, nothing travels, data included.
And `ON CLUSTER` is how a statement reaches *other shards*, because replication stops at the
shard boundary.

> ⚠️ A migration that renames a replicated table without `ON CLUSTER` succeeds, and leaves
> the servers disagreeing about the table's name. The next migration that refers to the new
> name fails on half the cluster.

## The Keeper path is the table's identity

Two copies of a replicated table are replicas of each other if, and only if, they register
under the same Keeper path. Two measured consequences:

**The default path needs `ON CLUSTER`.** The server default, from `system.server_settings`,
is `default_replica_path = /clickhouse/tables/{uuid}/{shard}`. Creating a replicated table
with no path arguments, without `ON CLUSTER`:

```
Macro 'uuid' in engine arguments is only supported when the UUID is explicitly
specified, used within an ON CLUSTER query, or when using the Replicated database engine.
```

Each server would otherwise invent its own table UUID, and the "replicas" would be strangers.
With `ON CLUSTER`, all servers get the same UUID and the copies replicate: a row inserted on
replica 1 was on replica 2 two seconds later.

**An explicit path must be unique per table, database included.** Explicit paths
(`'/clickhouse/tables/{shard}/<db>/<table>', '{replica}'`) work without `ON CLUSTER`, if
every server uses the same string. Leave the database out of the path and two databases on
the same cluster collide:

```
Replica /clickhouse/tables/01/t/replicas/r1 already exists. (REPLICA_ALREADY_EXISTS)
```

## Converting a table to a replicated engine

Engines can't be altered in place. There are two documented conversions, a flag file
(`convert_to_replicated`) in the table's data directory and `ATTACH TABLE … AS REPLICATED`
for a detached table, and one undocumented habit: create a replicated twin, `INSERT INTO
twin SELECT * FROM old`, then rename both.

The habit has three problems the documented paths avoid:

- **Writes during the copy are lost.** Rows inserted into the old table after the `SELECT`
  started aren't in the twin, and the rename then hides them.
- **Each server copies only its own data.** On a cluster of unreplicated tables, the servers
  hold *different* data. Running the copy on one server copies one server's rows.
- **The rename isn't atomic.** The `RENAME` documentation: "If you rename multiple tables in
  one query, the operation is not atomic. It may be partially executed, and queries in other
  sessions may get `Table ... does not exist ...` error." Splitting the two renames into two
  migrations widens that window from milliseconds to however long the runner takes between
  files.

For an atomic swap of one pair, the documentation points to `EXCHANGE TABLES`, which is
atomic for a single pair on `Atomic` databases. But: "When exchanging multiple table pairs,
the exchanges are performed sequentially, not atomically."

## DDL that depends on the server version

Two schema features that were rejected on 24.8 are accepted on 26.9, measured on the official
images:

| DDL | 24.8 | 26.9 |
|---|---|---|
| `Nullable(Tuple(…))` column | `Nested type Tuple(…) cannot be inside Nullable type` | accepted (`enable_nullable_tuple_type` = 1) |
| `ORDER BY (a, ts DESC)` | syntax error at `DESC` | accepted; `allow_experimental_reverse_key` is now "Obsolete setting, does nothing" |

A migration written and tested against a newer development server can fail on an older
production cluster, and the other way round a workaround for an old limitation can outlive
it. Pin the server version in development to production's, and note in the migration which
version it assumes.

## Migration runners on ClickHouse

Everything in [postgres/10](../postgres/10-migration-runners.md) still applies. What's
specific to ClickHouse, read from golang-migrate v4.18.3's ClickHouse driver and file
source:

| Behaviour | Source | Consequence |
|---|---|---|
| The lock is an in-process boolean (`isLocked.CAS(false, true)`) | `database/clickhouse/clickhouse.go` | Two runner processes against one cluster don't exclude each other |
| The version table is `version Int64, dirty UInt8, sequence UInt64`, and the current version is `ORDER BY sequence DESC LIMIT 1` | same | A hand-made version table without `sequence` breaks every run |
| Its default engine is `TinyLog`, created with `ON CLUSTER` only if `x-cluster-name` is set | same | Without it, each server keeps its own idea of the version |
| A file is sent as one statement unless `x-multi-statement=true`; with it, statements run one by one with no transaction around them | same, `Run` | With multi-statement on, a failure can leave the file half-applied; `dirty` then means "find out by hand what ran" |
| A recorded version missing from the folder fails with `no migration found for version N` | `migrate.go`, `versionExists` | Renumbering or collapsing applied files stops the runner until someone uses `force` |
| Two files with the same version number fail with `duplicate migration file` | `source/iofs/iofs.go` | A merge that brings back an old branch's files breaks every run |
| Files that don't match `^([0-9]+)_(.*)\.(down\|up)\.(.*)$` are silently skipped | `source/parse.go`, `iofs.go` | A `NOTES.txt` in the folder is ignored, not an error |

The renumbering row is the ClickHouse cousin of postgres/10's rule 1. Collapsing twenty
applied migrations into five tidy files changes nothing on databases that ran the twenty,
except that the runner now can't find the version it recorded, and stops. Fresh environments
get the five; old ones need a baseline (`force` to the new number after checking by hand that
the schema matches). For a while, two environments built from the same folder can have
different schemas.

> **Teacher's aside.** On one server, "where does this statement run?" has one answer, so
> nobody asks it. On a cluster it has three: the server you're connected to, every server
> `ON CLUSTER` names, and every replica the engine's Keeper path connects. Most cluster
> schema bugs come from answering one of those and assuming the others. Before running any
> DDL on a cluster, say all three out loud.

## Check yourself

1. You run `ALTER TABLE t ADD COLUMN c UInt8` without `ON CLUSTER` on one server of a
   two-shard, two-replica cluster where `t` is replicated. Which of the four servers have the
   column afterwards?
2. Two engineers each create a replicated table on their own server, without `ON CLUSTER`,
   with the same explicit Keeper path. Are the tables replicas of each other? What if one
   path omits the database name and both servers host two databases with a table `t`?
3. A conversion migration creates a replicated twin, copies with `INSERT … SELECT`, and
   renames in the next migration file. List what can be lost or broken, and when.
4. Someone collapses migrations 10–29 into new files 10–14. What does the runner do on a
   database at version 29, and on a fresh database?
5. Two deploy pods start at once and both run golang-migrate against the same ClickHouse
   cluster. What stops them both applying migration 15?
6. A migration uses `ORDER BY (k, ts DESC)` and was tested on a recent development server.
   What can happen in production, and how do you prevent it?
