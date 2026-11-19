# 4. Queries that leak

## The problem

A service looks a person up by whichever identifier the caller has: passport, id card or
national id, within the caller's tenant. A refactor moves the tenant condition into the same
`$or` as the identifiers. Now the query means "has this passport, or this id card, or belongs
to this tenant", and when no identifier matches, it returns some other person from the
tenant. Nothing errors. The other way a filter leaks is through its input: a request field
that arrives as an object, like `{"$ne": null}`, turns "equals this value" into "matches
anything".

Both are cases of the query saying something other than what the developer meant. This
chapter is about writing filters whose meaning can't drift that way. It assumes
[chapter 1](01-what-mongodb-does-with-a-query.md), mainly "Reading a filter".

## Scopes go at the top level

Top-level conditions are ANDed. "MongoDB provides an implicit `AND` operation when you
specify a comma separated list of expressions"
([$and](https://www.mongodb.com/docs/manual/reference/operator/query/and/)). `$or` "selects
documents that satisfy at least one of the expressions"
([$or](https://www.mongodb.com/docs/manual/reference/operator/query/or/)). So the position of
the tenant condition decides the logic:

```js
// Meant: tenant AND not deleted AND (passport OR idCard)
db.people.find({ tenant: "t1", deleted: false,
                 $or: [{ passport: p }, { idCard: c }] })

// Wrote: passport OR idCard OR tenant
db.people.find({ deleted: false,
                 $or: [{ passport: p }, { idCard: c }, { tenant: "t1" }] })
```

Reproduced with three people, two in tenant `t1`, and identifiers that match nobody: the first
form returned `[]`, the second returned both `t1` people. A `findOne` on the second form
returns whichever of them the plan meets first, which is how one person's record gets shown
to the wrong caller.

A **scope** is any condition that must hold for every result whatever else the query says:
tenant, owner, soft-delete flag, visibility. The rule is simple and absolute: **scopes are
top-level conditions, never inside an `$or`, `$nor` or user-supplied sub-filter.**

Two ways it goes wrong in practice:

- **Building the `$or` by pushing conditions into an array.** A helper collects "the
  conditions" into a list and wraps the list in `$or`, and the tenant gets pushed in with
  them. Keep the scope in a separate argument that the builder places at the top level.
- **Two `$or`s in one object.** A filter object can't hold the same key twice. The `$and`
  page notes that most languages and drivers, mongosh included, "do not allow the
  construction of objects with duplicate keys at the same object level", and in a JavaScript
  object the later key simply replaces the earlier one. If a query needs two alternative
  groups, combine them with an explicit `$and: [ {$or: …}, {$or: …} ]`.

The correct form is also the fast one. With indexes `{tenant: 1, passport: 1}` and
`{tenant: 1, idCard: 1}` on 5,000 documents, the scoped query planned as an `OR` of two
`IXSCAN`s, examining 2 keys and 2 documents. The leaking form was a `COLLSCAN` of all 5,000.
The planner carried the top-level `tenant` into each branch, and the manual's condition for
using indexes with `$or` is that "all the clauses in the `$or` expression must be supported
by indexes".

> **Teacher's aside.** The test that catches this is the one people don't write: a lookup
> whose identifiers match **nothing**, against a tenant that has other records. Tests
> usually look up something that exists and check they got it, and the leaking query passes
> those. Assert that a miss returns no result.

## Operator injection

A filter is data. If part of it comes from a request, the request can supply more than a
value. In JavaScript the path is short: `JSON.parse` of a body gives objects where the
client sent objects, and some query-string parsers build nested objects from brackets. The
[qs](https://github.com/ljharb/qs) parser's first example turns `foo[bar]=baz` into
`{ foo: { bar: "baz" } }`. Then:

```js
const { email, password } = req.body;           // client sent {"$ne": null} for both
db.users.findOne({ email, password })            // { email: {$ne: null}, password: {$ne: null} }
```

Reproduced: that filter matched both users in the collection. The same input wrapped in
`$eq` matched none, because `$eq` compares against the object literally:

```
raw input as filter:  ["a@x.io", "b@x.io"]
wrapped in $eq:       []
```

The manual's own advice is on the `$eq` page, under "Security Implications": "Always use the
explicit form `{ field: { $eq: <value> } }` with user-supplied input to avoid problems with
maliciously formed queries" ([$eq](https://www.mongodb.com/docs/manual/reference/operator/query/eq/)).
Mongoose has the same idea built in: `sanitizeFilter` works by "wrapping any nested objects
that have a property whose name starts with `$` in a `$eq`", off by default, with `trusted()`
to exempt a selector you built yourself
([Mongoose API](https://mongoosejs.com/docs/api/mongoose.html)).

The defences, in order of strength:

| Defence | What it stops | Note |
|---|---|---|
| **Enforce the type at the boundary** | Any object where a string was expected | A request schema, or decoding into typed fields. Go's `encoding/json` refuses an object for a `string` field |
| **`$eq` around every user value** | An operator object being read as an operator | Cheap; also a string starting with `$` is just a string under `$eq` |
| **Never pass request objects as filters, updates or sort specs** | Whole-document injection, including in updates (`$set` of fields the user shouldn't touch) | Copy named fields into a filter you build |
| **Allow-list field names** | User-chosen sort or projection fields reaching internal fields | |

> ⚠️ **Blocking characters is not a defence.** A sanitiser that rejects any input containing
> `.` or `$` breaks ordinary data (emails, decimals, names like "St. John", URLs) and still
> misses the real problem, which is the input's *type*. The dangerous thing is an object with
> a `$`-prefixed key reaching the filter, not a dot in a string. Reject wrong types, then pass
> strings as values.

On sources: OWASP's testing guide page,
[Testing for NoSQL Injection](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05.6-Testing_for_NoSQL_Injection),
concentrates on server-side JavaScript through `$where`. It doesn't show the operator-object
pattern above. The evidence for that pattern here is the MongoDB `$eq` page, Mongoose's
`sanitizeFilter` documentation and the reproduction. Server-side JavaScript (`$where`,
`$function`) is a separate hazard: keep user input out of it entirely.

Order matters too. Validate the whole request **before** writing anything, so a request
rejected half-way doesn't leave a record created for it.

## The shape in code

Reproduced against MongoDB 8.0 with the Go driver v2.5.0: `{"email":"a@x.io"}` counted 1,
and `{"email":{"$ne":null}}` was refused at decoding with
`json: cannot unmarshal object into Go struct field lookupRequest.email of type string`.

```go
type lookupRequest struct {
	Email string `json:"email"`
}

// findByEmail counts users with this email in the caller's tenant.
// The tenant comes from the authenticated caller, never from the body.
func findByEmail(ctx context.Context, users *mongo.Collection, tenant string, body []byte) (int64, error) {
	var req lookupRequest
	if err := json.Unmarshal(body, &req); err != nil { // an object here is a type error
		return 0, fmt.Errorf("bad request: %w", err)
	}
	filter := bson.D{
		{Key: "tenant", Value: tenant},                                   // scope, top level
		{Key: "email", Value: bson.D{{Key: "$eq", Value: req.Email}}},    // value, never operator
	}
	return users.CountDocuments(ctx, filter)
}
```

Go's typed decoding does most of the work, which is why the bug is rarer in Go than in Node.
It comes back if you decode into `map[string]any` or `bson.M` and pass that along.

<details><summary>Rust</summary>

Written against `serde`/`serde_json` and the `mongodb` crate 3.x; not compiled here. As in
Go, deserialising into a `String` field rejects an object.

```rust
use mongodb::bson::{doc, Document};
use mongodb::Collection;
use serde::Deserialize;

#[derive(Deserialize)]
struct LookupRequest {
    email: String,
}

pub async fn find_by_email(
    users: &Collection<Document>,
    tenant: &str,
    body: &[u8],
) -> Result<u64, Box<dyn std::error::Error>> {
    let req: LookupRequest = serde_json::from_slice(body)?; // object for email -> error
    let filter = doc! { "tenant": tenant, "email": { "$eq": req.email } };
    Ok(users.count_documents(filter).await?)
}
```

</details>

<details><summary>C++</summary>

Written against the mongocxx/bsoncxx 4.x API reference; not compiled here. The body is parsed
with `bsoncxx::from_json`, and the field's BSON type is checked before it's used.

```cpp
#include <cstdint>
#include <stdexcept>
#include <string>

#include <bsoncxx/builder/basic/document.hpp>
#include <bsoncxx/builder/basic/kvp.hpp>
#include <bsoncxx/json.hpp>
#include <mongocxx/collection.hpp>

using bsoncxx::builder::basic::kvp;
using bsoncxx::builder::basic::make_document;

std::int64_t find_by_email(mongocxx::collection& users, const std::string& tenant,
                           const std::string& body) {
  auto req = bsoncxx::from_json(body);
  auto email = req.view()["email"];
  if (!email || email.type() != bsoncxx::type::k_string)
    throw std::invalid_argument("bad request: email must be a string");
  auto filter = make_document(kvp("tenant", tenant),
                              kvp("email", make_document(kvp("$eq", email.get_string().value))));
  return users.count_documents(filter.view());
}
```

</details>

<details><summary>JavaScript</summary>

An ES module, run with Node 26 and the `mongodb` driver 6.22: a string counted 1, an object
was refused, and the same object passed raw as the filter value counted 1.

```js
export async function findByEmail(users, tenant, body) {
  const { email } = body;
  if (typeof email !== "string") throw new Error("bad request: email must be a string");
  return users.countDocuments({ tenant, email: { $eq: email } });
}
```

With Mongoose, `mongoose.set("sanitizeFilter", true)` adds the `$eq` wrapping to every query
filter; the type check still belongs at the boundary.

</details>

## Check yourself

1. Rewrite `{ $or: [{ passport: p }, { idCard: c }, { tenant: t }], deleted: false }` so it
   means what was intended, and describe the one test that would have caught the original.
2. A query builder in JavaScript does `filter.$or = idConditions;` and, further down for a
   new feature, `filter.$or = statusConditions;`. What does the final query mean, and how
   do you write it correctly?
3. Why is the scoped `$or` query fast with `{tenant, passport}` and `{tenant, idCard}`
   indexes, while the leaking one is a collection scan even though the same indexes exist?
4. A login handler does `users.findOne({ email: req.body.email, passwordHash: hash(req.body.password) })`.
   Is it injectable? Which field, if any, can an attacker turn into an operator, and what
   could they achieve with it?
5. A sanitiser rejects any input containing `.` or `$`. Name two kinds of legitimate input it
   breaks, and an injection it still doesn't stop if the request is a JSON body handled by
   `findOne({ email: body.email })` after the check runs on `String(body.email)`.
