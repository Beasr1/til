# 10. Migration runners and what they remember

## The problem

A migration runner looks like the simplest tool in the stack: a folder of numbered SQL
files, and a command that runs the ones that haven't run yet. Almost every serious
schema incident involving one comes from misunderstanding a single thing: **what the
runner remembers, and what it therefore can't notice.** It doesn't compare your schema
to your files. It compares a bookkeeping table to a list of filenames, and nothing else.

This chapter is tool-agnostic. The examples name golang-migrate and goose because they
make a useful contrast, but Flyway, Liquibase, Rails, Alembic and Django all face the
same questions.

## What the runner actually stores

| Tool | Bookkeeping | What it can tell |
|---|---|---|
| golang-migrate | `schema_migrations`: **one row**, `(version, dirty)` | The highest version applied, and whether the last attempt failed |
| goose | `goose_db_version`: **one row per applied version** | Exactly which versions have been applied |
| Flyway, Liquibase | One row per migration, with a checksum of the file | Which ran, and whether a file has changed since it ran |

Everything else in this chapter follows from that table.

## Rule 1: an applied migration is frozen

Suppose a migration that created a table was applied everywhere a month ago, and
someone edits the file to add a trigger they forgot. On a fresh database, the trigger
exists. On every existing database, it doesn't: the runner sees the version as applied
and never opens the file again. Nothing errors. The environments have silently diverged,
and the code that relies on the trigger, say to maintain an `updated_at` column, behaves
differently depending on when each database was created.

The symptom often turns up much later and somewhere else entirely, such as a query that
compares `updated_at` and gives wrong answers on production only. Someone "fixes" it by
switching the query to `created_at`, and the real cause is never found.

Tools with checksums (Flyway, Liquibase) refuse to run when an applied file has changed.
Tools without them, including golang-migrate and goose, simply don't notice. The rule
has to be a convention: **an applied migration is never edited; a fix is a new
migration.** It's the same discipline as never renumbering a field in a wire format, for
the same reason. The reader, here the database, already acted on the old version.

## Rule 2: know what "failed" leaves behind

**golang-migrate** sets a dirty flag before running a migration. From its
[FAQ](https://github.com/golang-migrate/migrate/blob/master/FAQ.md): "Execution stops if
a migration fails and the dirty state persists, which prevents attempts to run more
migrations on top of a failed migration." You then "manually fix the error and then
'force' the expected version." That's annoying, and it's honest: it refuses to guess
whether a half-applied migration left the schema in the old state or the new.

**goose** has no dirty state. A failed migration just isn't recorded. Whether that's
safe depends on whether the migration ran in a transaction. The goose
[README](https://github.com/pressly/goose): "By default, all migrations are run within a
transaction." A transactional migration that fails rolls back completely, so "not
recorded" really does mean "not applied". But a migration marked
`-- +goose NO TRANSACTION`, which is required for `CREATE INDEX CONCURRENTLY`, can fail
half-way and leave some statements applied and none recorded. That's exactly the
dirty state, with no flag to warn you.

That's why non-transactional migrations must be **idempotent**, with every statement
guarded by `IF NOT EXISTS` or a catalog check, so that re-running them is safe. And it's
why the `IF NOT EXISTS` trap from [chapter 9](09-changing-a-live-table.md) matters: an
idempotent re-run of an interrupted concurrent build "succeeds" over an INVALID index.

> **Teacher's aside.** "Our tool has no dirty state" sounds like an improvement, and
> often it is. Notice what happened, though: the tool stopped reporting partial failure,
> it didn't stop partial failure from happening. For transactional migrations the two
> are the same. For non-transactional ones, the job of noticing has moved from the tool
> to you.

## Rule 3: migrations belong in the deploy, not in the app's startup

The convenient setup is to have the service run pending migrations on boot. It works for
one instance and small migrations. It fails in ways that get worse with scale:

| Problem | Mechanism |
|---|---|
| Every replica races to migrate | Runners take a lock (golang-migrate uses `pg_advisory_lock` on Postgres) so they serialise, but every replica blocks at startup until the slowest migration finishes |
| A long migration kills the rollout | A forty-minute index build means a forty-minute startup. Health checks fail and the orchestrator restarts the pod, mid-migration |
| Code and schema ship as one unit | You can't apply a schema change ahead of the code that needs it, or roll the code back without the schema |
| The app needs DDL privileges | The runtime database user can drop tables |

Running migrations as a separate deploy step, a job or a container that runs before the
new version rolls out, fixes all four. It also makes the ordering explicit, which is what
safe changes need. The **expand/contract** pattern relies on it: add the new thing,
deploy code that uses both, migrate the data, deploy code that uses only the new thing,
then remove the old thing. Each arrow is a separate deploy.

## Rule 4: numbering is a concurrency problem

Two developers on two branches each add migration `000017`. Both pass review. The
second to merge collides with the first: a duplicate version, or one of them renamed to
`000018` *after* it's already been applied on a shared test database. Renaming an applied
migration is the same as editing it (Rule 1), because the runner keys on the number.

Two schemes deal with this:

| Scheme | Collisions | What goes wrong instead |
|---|---|---|
| Serial (`000017`) | Common with parallel branches | Renumbering after the fact |
| Timestamp (`20260903120000`) | Rare | Migrations arrive **out of order** |

Timestamps move the problem rather than removing it. Branch A creates a migration on
Monday and merges on Friday. Branch B creates one on Wednesday and merges on Thursday.
Production applies B's Wednesday migration on Thursday. On Friday A's *Monday* migration
arrives, older than one already applied.

What happens next depends entirely on the bookkeeping table:

- **golang-migrate** stores only the highest version. A file numbered below it is never
  applied, **silently**, on every database that has already passed it.
- **goose** stores every version, so it can see the gap, and by default it refuses: "By
  default, Goose rejects missing (out-of-order) migrations with an error." The
  `-allow-missing` flag applies them.

goose's README recommends a hybrid for exactly this: timestamps during development, then
its `fix` command to rewrite them as sequential numbers before production. That only
works if `fix` runs before anything is applied anywhere shared. If your pre-production
environments apply migrations straight from feature branches, `fix` would renumber
applied migrations, and plain timestamps plus `-allow-missing` is the honest choice.

Mixing schemes works in goose because a version is just an integer: serial versions in
the low range, timestamps far above them, always sorting after.

## Rule 5: a shared database sees every branch

When several feature branches deploy to one shared environment, its database ends up
with migrations from all of them. Two consequences:

- The database is **ahead** of any single branch's files. golang-migrate errors when
  the file for the current version is missing, and a common workaround, "if the
  current version's file doesn't exist, skip migrating", turns a loud error into silent
  drift.
- A migration that's later deleted or renumbered on its branch stays applied in the
  shared database forever, leaving gaps in the version sequence.

There isn't a tool fix for this. It's an argument for short-lived per-branch databases,
or for treating the shared environment's schema as disposable.

## Rule 6: a down migration isn't an undo

Every runner lets you write a "down" next to each "up", and it's tempting to read the pair
as a reversible operation. For schema-only changes it nearly is. For anything that touches
data or certain types, it isn't, and the down file is a comforting fiction.

| Up | What the down can't restore |
|---|---|
| `DROP TABLE` / `DROP COLUMN` | The data. A down can recreate an *empty* table with the right shape, triggers and indexes, but every row is gone. Only a backup brings it back |
| Copy rows from table A into B, then drop A | Which rows in B came from A. Rows written to B after the up look identical, so a down that copies B back into A takes them too |
| `ALTER TYPE … ADD VALUE` on an enum | Nothing. PostgreSQL's [ALTER TYPE](https://www.postgresql.org/docs/current/sql-altertype.html) has `ADD VALUE` and `RENAME VALUE` but no way to drop a value. A down can only leave it, or recreate the whole type and rewrite every column using it |
| A lossy type change (text → int, truncation) | The values that didn't convert |

Three practical consequences:

- **Destructive steps go last, and alone.** In expand/contract (Rule 3), the contract step
  that drops the old table or column is its own deploy, after the code has stopped using it
  and after you've confirmed a backup. Keep it in a separate migration so it can be held back
  without holding back anything else.
- **Don't add a destructive migration and then delete it.** If a migration that drops a
  table is merged, applied in some environments, and then removed from the repository
  because the drop was premature, those environments have lost the table *and* now record a
  version whose file no longer exists. golang-migrate refuses to run at all in that state,
  because it checks that the file for the database's current version exists. The fix for a
  premature drop is a new migration that recreates the table, plus a restore from backup,
  never a deleted file.
- **Write the down's limits into the down.** If a down is lossy, say exactly what it loses
  in a comment at the top. Whoever runs it during an incident won't have time to work it out.

## Switching tools: baselining

Moving an existing database from one runner to another has one trap: the new tool's
bookkeeping table is empty, so it believes **every** migration is pending, including the
ones that built the schema you're running on. It will try to recreate tables that exist
and fail part-way, at the first statement that isn't idempotent (in PostgreSQL,
`CREATE TRIGGER` has had no `IF NOT EXISTS`), leaving its own table half-populated.

The fix is a **baseline**: before the first run, record the already-applied versions in
the new tool's table, derived from the old tool's table rather than hardcoded, so it's
correct whatever version each environment has reached. Three things make a baseline
safe:

- **Refuse if the old tool says dirty.** Baselining a half-applied migration records a
  lie about the schema.
- **Compare coverage, not existence.** "The new table already has rows" doesn't mean a
  previous baseline finished. A run that failed part-way leaves some rows. Check that
  the recorded versions reach the old tool's version.
- **Run it automatically.** A baseline that's a documented manual step will be skipped
  in one environment. Putting the check in the migration container's entrypoint means it
  happens however the container is started.

## The runner parses your SQL, badly

A last, practical point. Most runners split a file into statements themselves, and their
splitter isn't a SQL parser. Semicolons inside a PL/pgSQL function body (`$$ … $$`), or
inside comments, can be taken as statement ends, and the server receives fragments.
goose asks you to mark such bodies with `-- +goose StatementBegin` and
`-- +goose StatementEnd`. It also reads its annotations out of `--` comments, so the
literal annotation text inside an ordinary comment is parsed as an instruction. Run the
tool's validation command (`goose validate`) in CI, before anything touches a database.

## Check yourself

1. A colleague adds a missing index to an already-applied migration file "because it
   belongs with that table". Describe the state of a fresh database and of production
   afterwards, and how long it might take anyone to notice.
2. With golang-migrate at version 20, a branch merges a migration numbered 18. What
   happens on production? With goose at default settings?
3. goose says it has "no dirty state". Under what conditions does a failed goose
   migration leave the schema partly changed, and what do you have to do about it?
4. List three things that go wrong when a service runs a forty-minute migration on
   startup, in the order you'd hit them during a rolling deploy.
5. Why does goose's recommended hybrid (timestamps in development, `fix` before
   production) fail for a team whose test environment deploys straight from feature
   branches?
6. A migration copies every row from `legacy_orders` into `orders` and drops
   `legacy_orders`. A week later someone runs its down, which recreates `legacy_orders`
   and copies `orders` back. What does `legacy_orders` contain, and what's lost? Separately:
   if the up had instead been deleted from the repository after running on staging, what
   would golang-migrate do on staging's next run?
