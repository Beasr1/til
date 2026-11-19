# 2. Uniqueness

## The problem

A people collection holds optional identifiers: a national id, a passport, an id card. Any of
them may be absent, but when one is present no two people may share it. The first attempt is
a unique index marked "sparse, so it ignores nulls". The second person created without a
national id fails with a duplicate-key error on `null`. The second attempt, a compound unique
index on `{ passport, idCard }`, lets person A's passport number appear as person B's id
card, because it only forbids a repeated *pair*. And the guard against a second in-progress
job, "check none exists, then insert", lets two through when two requests arrive together.

Three different constraints, three different tools. This chapter is about picking the right
one, and about the null and missing values that break the obvious choices. It assumes
[chapter 1](01-what-mongodb-does-with-a-query.md): the three states of a field, and that an
ordinary index holds a key for every document.

## What a unique index actually enforces

A unique index says: **no two documents in this index have the same key.** The catch is in
"in this index". An ordinary index contains every document, and a document without the field
gets the key `null`. The manual
([Unique Indexes](https://www.mongodb.com/docs/manual/core/index-unique/)):

> If a document has a `null` or missing value for the indexed field in a unique single-field
> index, the index stores a `null` value for that document. Because of the unique
> constraint, a single-field unique index can only contain one document that contains a
> `null` value in its index entry.

So a plain unique index on an optional field allows exactly one document without it. "Unique
when present" means **leaving the other documents out of the index**, and MongoDB has two
ways to do that: sparse indexes and partial indexes. They leave out different things.

## Sparse, `$exists` and `$type`: what each leaves out

A **sparse index** skips documents that don't have the field. A **partial index** skips
documents that don't match a `partialFilterExpression`, a filter written in a restricted
subset of the query language ([Partial Indexes](https://www.mongodb.com/docs/manual/core/index-partial/)
lists equality, `$exists: true`, `$gt`/`$gte`/`$lt`/`$lte`, `$type`, `$and`, `$or`, `$in`
and the two geo operators). With `unique: true`, "the unique constraint only applies to the
documents that meet the filter expression".

The question is what each does with a stored `null`. Reproduced on MongoDB 8.0, inserting
two documents with the field missing, then two with `nid: null`:

| Index on `nid` | Two missing | Two `null` | Why |
|---|---|---|---|
| `unique` | second fails | second fails | Both states are key `null` |
| `unique, sparse: true` | ✅ | second fails | Sparse skips *missing* only. The manual: sparse indexes "only contain entries for documents that have the indexed field, even if the index field contains a null value" |
| `unique, partialFilterExpression: { nid: { $exists: true } }` | ✅ | second fails | `$exists: true` matches a field holding `null` ([$exists](https://www.mongodb.com/docs/manual/reference/operator/query/exists/)) |
| `unique, partialFilterExpression: { nid: { $type: "string" } }` | ✅ | ✅ | `null` is not a string, so the document isn't in the index |

```
sparse: null #2: E11000 duplicate key error ... index: nid_1 dup key: { nid: null }
partial exists: null #2: E11000 duplicate key error ... index: nid_1 dup key: { nid: null }
partial type: null x2, missing x2: OK
partial type: 'A' #2: E11000 duplicate key error ... index: nid_1 dup key: { nid: "A" }
```

The obvious alternative, `{ nid: { $ne: null } }`, isn't allowed: `$ne` isn't on the list,
and the server refuses the index with `CannotCreateIndex`. `$type` is the operator that says
"present and meaningful".

> **Teacher's aside.** "Sparse ignores nulls" is the belief behind most of these bugs, and
> it's half right. Sparse ignores *missing*. Whether your nulls are missing or stored is
> decided by whichever code last wrote the document ([chapter 1](01-what-mongodb-does-with-a-query.md)),
> so a sparse unique index works until the first writer that stores `null` instead of
> leaving the field out. The `$type` filter doesn't care which writer you have.

### A compound sparse index includes almost everything

Sparse has a second trap. For a compound index the manual's table says a sparse index
"only indexes documents that contain a value for at least one of the keys"
([Sparse Indexes](https://www.mongodb.com/docs/manual/core/index-sparse/)). Put a field every
document has, like a tenant, next to the optional one:

```js
db.people.createIndex({ tenant: 1, nid: 1 }, { unique: true, sparse: true })
db.people.insertOne({ tenant: "t1" })   // ok
db.people.insertOne({ tenant: "t1" })   // E11000 ... dup key: { tenant: "t1", nid: null }
```

Every document has `tenant`, so every document is in the index, and two people in one
tenant without a national id collide. The partial version scopes properly, because the filter
names the optional field itself:

```js
db.people.createIndex(
  { tenant: 1, nid: 1 },
  { unique: true, partialFilterExpression: { nid: { $type: "string" } } })
```

The filter can also name fields outside the key, which is how soft-deleted documents are
kept out of a constraint: `partialFilterExpression: { deleted: false, nid: { $type: "string" } }`.

### What `$type` doesn't cover

Two consequences of filtering on type, both reproduced:

- **Other types escape the constraint.** Two documents with `nid: 5` both inserted into the
  `$type: "string"` index, because a number isn't a string. The index enforces uniqueness of
  *string* ids. Make the type a rule, with a validator (`$jsonSchema` with `bsonType`) or a
  single typed write path, or the constraint has a hole.
- **Queries must say `$type` to use the index.** A partial index is used only when "a query
  [contains] the filter expression (or a modified filter expression that specifies a subset
  of the filter expression)" ([Partial Indexes](https://www.mongodb.com/docs/manual/core/index-partial/)).
  The planner doesn't infer that `{ nid: "A" }` implies a string: that query was a
  `COLLSCAN`. `{ nid: { $eq: "A", $type: "string" } }` used the index. The constraint works
  either way; it's lookups that silently go slow.

### Changing the filter means a new index

You can't edit a partial filter in place. Asking `createIndex` for the same key with a new
filter, under the default generated name, fails with `IndexKeySpecsConflict` ("An existing
index has the same name as the requested index"). With an explicit new name, the server
accepts a second index on the same key with a different filter, so the safe change is:
create the new one under a new name, then drop the old one. The new build has to succeed on
the existing data first, which is a [chapter 3](03-changing-indexes-on-live-collections.md)
problem.

## Unique across several fields

The second constraint: a document number must be unique *whichever slot it sits in*.
Person A's passport must not reappear as person B's id card.

A **compound unique index** doesn't do this. The manual: it "ensures that any given
combination of the index key values appears at most once". `(P1, I1)` and `(P1, I2)` are
different combinations, so both inserts succeed, and the same passport is on two people.
Two separate unique indexes, one per slot, catch a repeat within a slot but not across them,
because they're different indexes over different paths.

MongoDB has no expression indexes, so you can't index "all the document numbers of this
person" directly. What you can do is **store** that set:

```js
{ _id: ..., passport: { country: "GBR", number: "P1" },
            idCard:   { country: "FRA", number: "X9" },
  docKeys: ["GBR::P1", "FRA::X9"] }              // derived, one per occupied slot

db.people.createIndex(
  { docKeys: 1 },
  { name: "docKeys_unique", unique: true,
    partialFilterExpression: { docKeys: { $type: "string" } } })
```

An index on an array field is **multikey**: one key per distinct element. And the unique
constraint is between documents, not inside one. The manual: "a document may have array
elements that result in repeating index key values as long as the index key values for that
document do not duplicate those of another document" ([Unique Indexes](https://www.mongodb.com/docs/manual/core/index-unique/)).
So one index now enforces "no document number appears on two people", whichever slot.
Reproduced: a second document holding `"g::Y::2"` failed against a first holding
`["p::X::1", "g::Y::2"]`, while a single document holding `["z::1", "z::1"]` inserted fine.

The key format is part of the design. Include everything that makes two numbers the same
document (here the issuing country), normalise case and whitespace, and pick a separator that
can't occur in either part.

### Leave the field out, and filter it anyway

What does a person with no documents store? Not an empty array, and not nothing under a plain
unique index. Reproduced against `{ docKeys: 1 }, { unique: true }` with no filter:

```
multikey: [] #2:       E11000 ... dup key: { keys: undefined }
multikey: missing #2:  E11000 ... dup key: { keys: null }
```

An empty array is indexed as a single `undefined` key, so two people with `docKeys: []`
collide. Missing is indexed as `null`, so two people without the field collide too.
(The manual's multikey page doesn't say how an empty array is indexed; the `undefined` key is
what the server reported.) Two habits fix both:

- **The partial filter `{ docKeys: { $type: "string" } }`.** For an array, `$type` "returns
  documents in which at least one array element matches"
  ([$type](https://www.mongodb.com/docs/manual/reference/operator/query/type/)). An empty
  array has no elements, so it's out, and so is a missing field. Reproduced: two `[]`
  documents inserted fine under this filter.
- **Unset the field when there are no keys**, rather than writing `[]`. The filter already
  protects the index, but an absent field keeps the data honest for every other reader, and
  it means an index built with `$exists: true` (or a sparse one) by someone later doesn't
  reintroduce the collision. Watch ODM defaults here. Mongoose: "Arrays are special because
  they implicitly have a default value of `[]`", and you override it with
  `default: undefined` ([Mongoose SchemaTypes](https://mongoosejs.com/docs/schematypes.html)).

### Keeping a derived field true

A derived field is only as correct as the one path that writes it. In Go there's no
middleware, so it's a repository function every write goes through. In Mongoose it's tempting
to compute it in a `pre('save')` hook, and the docs are explicit about the limit: "Pre and
post `save()` hooks are **not** executed on `update()`, `findOneAndUpdate()`, etc."
([Mongoose middleware](https://mongoosejs.com/docs/middleware.html)). Any route that writes
identifiers with `updateOne` bypasses the hook and leaves `docKeys` stale, so either route
everything through `save()` or recompute the keys in the update itself.

Existing documents need a **backfill** before the index can be built. The backfill usually
computes the same key in an aggregation (`$concat`, `$toUpper`, `$trim`) because application
code can't run inside the server. That's a second implementation of the key builder, and the
two must agree exactly, or the backfilled keys and the live keys won't collide when they
should. Keep them next to each other and test them on the same inputs.

## "At most one active" and the check-then-insert race

The third constraint: at most one in-progress job per order. The natural code reads, then
writes:

```
request 1: find({order: "o1", state: "in_progress"}) -> none
request 2: find({order: "o1", state: "in_progress"}) -> none
request 1: insertOne({order: "o1", state: "in_progress"})
request 2: insertOne({order: "o1", state: "in_progress"})   // two active jobs
```

Nothing in the application can close that gap, because the read and the write are separate
operations ([chapter 1](01-what-mongodb-does-with-a-query.md): atomicity is per document,
per operation). The index can, because the server checks it inside the insert:

```js
db.jobs.createIndex(
  { order: 1 },
  { unique: true, partialFilterExpression: { state: "in_progress" } })
```

Reproduced: two `done` jobs and one `in_progress` job for the same order inserted together;
a second `in_progress` job failed with `E11000`. Finished jobs stay out of the index, so
history accumulates freely. The application now just inserts and treats a duplicate-key
error (code `11000`) as "already running".

The same reasoning applies to **upserts**. An upsert is a query then a write, and without a
unique index two concurrent upserts can both insert. The manual
([db.collection.update(), "Upsert with Duplicate Values"](https://www.mongodb.com/docs/manual/reference/method/db.collection.update/)):
"Upserts can create duplicate documents, unless there is a unique index to prevent
duplicates." With the index, exactly one inserts and the others "either update the
newly-inserted document or fail due to a unique key collision". They update rather than fail
only when the filter is equality on exactly the unique index's fields and the update doesn't
change those fields. So keep upsert filters to the unique key, and still handle `11000`.

## The shape in code

The key builder is pure logic; the write sets the keys or unsets the field, and a duplicate
key means the document number is taken. Reproduced against MongoDB 8.0 with the Go driver
v2.5.0: a second person with no documents saved fine, and saving an id card equal to another
person's passport (different case) returned `ErrDocTaken`.

```go
type TravelDoc struct{ Country, Number string }

// docKeys returns one normalised key per occupied document slot,
// or nil when there are none, so the caller can leave the field unset.
func docKeys(slots ...TravelDoc) []string {
	var keys []string
	seen := map[string]bool{}
	for _, d := range slots {
		country := strings.ToUpper(strings.TrimSpace(d.Country))
		number := strings.ToUpper(strings.Join(strings.Fields(d.Number), ""))
		if country == "" || number == "" {
			continue
		}
		k := country + "::" + number
		if !seen[k] {
			seen[k] = true
			keys = append(keys, k)
		}
	}
	return keys
}

var ErrDocTaken = errors.New("travel document already registered")

func ensureDocKeyIndex(ctx context.Context, people *mongo.Collection) error {
	_, err := people.Indexes().CreateOne(ctx, mongo.IndexModel{
		Keys: bson.D{{Key: "docKeys", Value: 1}},
		Options: options.Index().
			SetName("docKeys_unique").
			SetUnique(true).
			SetPartialFilterExpression(bson.D{
				{Key: "docKeys", Value: bson.D{{Key: "$type", Value: "string"}}},
			}),
	})
	return err
}

// savePerson is the only function that writes identifiers, so docKeys can't drift.
func savePerson(ctx context.Context, people *mongo.Collection, id bson.ObjectID,
	passport, idCard TravelDoc) error {
	set := bson.D{{Key: "passport", Value: passport}, {Key: "idCard", Value: idCard}}
	update := bson.D{{Key: "$set", Value: set}}
	if keys := docKeys(passport, idCard); len(keys) > 0 {
		update[0].Value = append(set, bson.E{Key: "docKeys", Value: keys})
	} else {
		update = append(update, bson.E{Key: "$unset", Value: bson.D{{Key: "docKeys", Value: ""}}})
	}
	_, err := people.UpdateOne(ctx, bson.D{{Key: "_id", Value: id}}, update,
		options.UpdateOne().SetUpsert(true))
	if mongo.IsDuplicateKeyError(err) {
		return ErrDocTaken
	}
	return err
}
```

<details><summary>Rust</summary>

The key builder was compiled and run with rustc 1.96. The driver calls are written against
the `mongodb` crate 3.x API docs ([`IndexOptions`](https://docs.rs/mongodb/latest/mongodb/options/struct.IndexOptions.html),
[`Update`](https://docs.rs/mongodb/latest/mongodb/action/struct.Update.html)) and were not
compiled here.

```rust
use mongodb::bson::{doc, oid::ObjectId, Document};
use mongodb::error::{Error, ErrorKind, WriteFailure};
use mongodb::{options::IndexOptions, Collection, IndexModel};

pub struct TravelDoc<'a> {
    pub country: &'a str,
    pub number: &'a str,
}

/// One normalised key per occupied slot, or None when there are none,
/// so the caller can leave the field unset.
pub fn doc_keys(slots: &[TravelDoc]) -> Option<Vec<String>> {
    let mut keys: Vec<String> = Vec::new();
    for d in slots {
        let country = d.country.trim().to_uppercase();
        let number = d.number.split_whitespace().collect::<String>().to_uppercase();
        if country.is_empty() || number.is_empty() {
            continue;
        }
        let k = format!("{country}::{number}");
        if !keys.contains(&k) {
            keys.push(k);
        }
    }
    if keys.is_empty() { None } else { Some(keys) }
}

pub async fn ensure_doc_key_index(people: &Collection<Document>) -> Result<(), Error> {
    let opts = IndexOptions::builder()
        .name("docKeys_unique".to_string())
        .unique(true)
        .partial_filter_expression(doc! { "docKeys": { "$type": "string" } })
        .build();
    let model = IndexModel::builder().keys(doc! { "docKeys": 1 }).options(opts).build();
    people.create_index(model).await?;
    Ok(())
}

pub enum SaveError {
    DocTaken,
    Db(Error),
}

pub async fn save_person(
    people: &Collection<Document>,
    id: ObjectId,
    mut fields: Document, // passport and idCard, already shaped
    keys: Option<Vec<String>>,
) -> Result<(), SaveError> {
    let update = match keys {
        Some(k) => {
            fields.insert("docKeys", k);
            doc! { "$set": fields }
        }
        None => doc! { "$set": fields, "$unset": { "docKeys": "" } },
    };
    match people.update_one(doc! { "_id": id }, update).upsert(true).await {
        Ok(_) => Ok(()),
        Err(e) => match *e.kind {
            ErrorKind::Write(WriteFailure::WriteError(ref we)) if we.code == 11000 => {
                Err(SaveError::DocTaken)
            }
            _ => Err(SaveError::Db(e)),
        },
    }
}
```

</details>

<details><summary>C++</summary>

The key builder was compiled and run with Apple clang 17 (`-std=c++20 -Wall -Wextra
-Wpedantic`). The driver calls are written against the
[mongocxx 4.x API reference](https://mongocxx.org/api/mongocxx-4.3.0) and were not compiled
here (no C++ driver on the machine). Index options are passed as a document, which is what the
`createIndexes` command takes.

```cpp
#include <algorithm>
#include <cctype>
#include <optional>
#include <string>
#include <vector>

#include <bsoncxx/builder/basic/array.hpp>
#include <bsoncxx/builder/basic/document.hpp>
#include <bsoncxx/builder/basic/kvp.hpp>
#include <bsoncxx/oid.hpp>
#include <mongocxx/collection.hpp>
#include <mongocxx/exception/operation_exception.hpp>
#include <mongocxx/options/update.hpp>

using bsoncxx::builder::basic::kvp;
using bsoncxx::builder::basic::make_document;

struct TravelDoc {
  std::string country, number;
};

static std::string upper_without_spaces(const std::string& s) {
  std::string out;
  for (unsigned char c : s)
    if (!std::isspace(c)) out.push_back(static_cast<char>(std::toupper(c)));
  return out;
}

// One normalised key per occupied slot, or nullopt when there are none,
// so the caller can leave the field unset.
std::optional<std::vector<std::string>> doc_keys(const std::vector<TravelDoc>& slots) {
  std::vector<std::string> keys;
  for (const auto& d : slots) {
    auto country = upper_without_spaces(d.country);
    auto number = upper_without_spaces(d.number);
    if (country.empty() || number.empty()) continue;
    auto k = country + "::" + number;
    if (std::find(keys.begin(), keys.end(), k) == keys.end()) keys.push_back(k);
  }
  if (keys.empty()) return std::nullopt;
  return keys;
}

void ensure_doc_key_index(mongocxx::collection& people) {
  people.create_index(
      make_document(kvp("docKeys", 1)),
      make_document(kvp("name", "docKeys_unique"), kvp("unique", true),
                    kvp("partialFilterExpression",
                        make_document(kvp("docKeys", make_document(kvp("$type", "string")))))));
}

// Returns false when a document number is already registered to someone else.
bool save_person(mongocxx::collection& people, const bsoncxx::oid& id,
                 bsoncxx::document::view fields,  // passport and idCard, already shaped
                 const std::optional<std::vector<std::string>>& keys) {
  bsoncxx::builder::basic::document set;
  for (auto el : fields) set.append(kvp(el.key(), el.get_value()));
  bsoncxx::builder::basic::document update;
  if (keys) {
    bsoncxx::builder::basic::array arr;
    for (const auto& k : *keys) arr.append(k);
    set.append(kvp("docKeys", arr.extract()));
    update.append(kvp("$set", set.extract()));
  } else {
    update.append(kvp("$set", set.extract()));
    update.append(kvp("$unset", make_document(kvp("docKeys", ""))));
  }
  mongocxx::options::update opts;
  opts.upsert(true);
  try {
    people.update_one(make_document(kvp("_id", id)), update.extract(), opts);
  } catch (const mongocxx::operation_exception& e) {
    if (e.code().value() == 11000) return false;  // duplicate key
    throw;
  }
  return true;
}
```

</details>

<details><summary>JavaScript</summary>

An ES module, run with Node 26 and the `mongodb` driver 6.22 against MongoDB 8.0; same
results as the Go version.

```js
// One normalised key per occupied slot; undefined (not []) when there are none.
export function docKeys(...slots) {
  const keys = new Set();
  for (const d of slots) {
    const country = (d?.country ?? "").trim().toUpperCase();
    const number = (d?.number ?? "").replace(/\s+/g, "").toUpperCase();
    if (country && number) keys.add(`${country}::${number}`);
  }
  return keys.size ? [...keys] : undefined;
}

export async function ensureDocKeyIndex(people) {
  await people.createIndex(
    { docKeys: 1 },
    { name: "docKeys_unique", unique: true,
      partialFilterExpression: { docKeys: { $type: "string" } } });
}

export async function savePerson(people, id, passport, idCard) {
  const keys = docKeys(passport, idCard);
  const update = keys
    ? { $set: { passport, idCard, docKeys: keys } }
    : { $set: { passport, idCard }, $unset: { docKeys: "" } };
  try {
    await people.updateOne({ _id: id }, update, { upsert: true });
  } catch (err) {
    if (err?.code === 11000) throw new Error("travel document already registered");
    throw err;
  }
}
```

</details>

## Check yourself

1. A unique index `{ nid: 1 }` with `sparse: true` has worked for a year. A new service
   starts writing people, and the second one it creates without a national id fails. What
   is that service doing differently, and which index definition would have been immune?
2. Why does `{ tenant: 1, nid: 1 }` with `unique: true, sparse: true` reject the second
   person in a tenant who has no `nid`, when the single-field sparse version wouldn't
   (provided the field is missing)?
3. With the partial filter `{ nid: { $type: "string" } }`, one client stores national ids as
   numbers. What does the constraint guarantee now, and what would you add?
4. Explain why `{ passport: 1, idCard: 1 }` with `unique: true` doesn't stop one person's
   passport from being another person's id card, and why one unique index on a derived
   `docKeys` array does.
5. You drop the partial filter from the `docKeys` index "because it's unique anyway" and
   deploy. Which two kinds of document start colliding, and on which keys?
6. A Mongoose model computes `docKeys` in `pre('save')`. A new admin endpoint corrects a
   passport number with `findOneAndUpdate`. Walk through what the index enforces afterwards,
   and describe a concrete duplicate it now lets in.
