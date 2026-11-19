# 10. Authorisation: whose object is this?

Assumes [file 01](01-foundations.md) for `401` and `403`, and
[file 09](09-keeping-a-browser-signed-in.md) for how a request comes to carry a verified
identity. This chapter starts where file 09 stops: the server knows who is asking. Now it
has to decide what they may touch.

## The problem

A multi-tenant dashboard checks every route: "is this caller's role high enough?" A manager
at one customer company passes the check, opens `/invoices/8812`, changes the number to
`8813`, and reads an invoice that belongs to a different company. Every check ran and every
check passed. None of them asked whose invoice `8813` was.

It's the most common access-control flaw on the web. OWASP's Top 10 for 2025 keeps Broken
Access Control "at #1", and its list of examples includes "Permitting viewing or editing
someone else's account by providing its unique identifier (insecure direct object
references)" ([A01:2025](https://top10.owasp.org/2025/A01_2025-Broken_Access_Control)).
OWASP's API list puts the same flaw first, as **Broken Object Level Authorization (BOLA)**
([API1:2023](https://api-security.owasp.org/editions/2023/en/0xa1-broken-object-level-authorization/)).

## Two questions, not one

An access check answers one of two questions, and they fail independently.

| | **Function level** (vertical) | **Object level** (horizontal) |
|---|---|---|
| The question | May this *kind* of caller do this at all? | May this caller do it to *this* object? |
| Decided by | The caller's role or group, and the route | The caller's identity or tenant, and the row |
| Typical check | Middleware on the route: `role >= manager` | A condition in the query: `tenant_id = $1` |
| Failure | A regular user calls an admin endpoint | A user changes an ID and gets someone else's record |
| OWASP API Top 10 | [API5:2023](https://api-security.owasp.org/editions/2023/en/0xa5-broken-function-level-authorization/) BFLA | [API1:2023](https://api-security.owasp.org/editions/2023/en/0xa1-broken-object-level-authorization/) BOLA |

The words "vertical" and "horizontal" come from the
[OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html),
which says least privilege "must be applied both horizontally and vertically" and calls
horizontal privilege escalation, "being able to access another user's resources", "an
especially common weakness".

> **Teacher's aside.** A role check is about the *route*; an ID is about the *row*. Once a
> route is open to managers, every manager can reach it, so the role check has nothing more
> to say about which invoice comes back. OWASP draws the line exactly there: "In the case of
> BOLA, it's by design that the user will have access to the vulnerable API
> endpoint/function. The violation happens at the object level, by manipulating the ID."
> Adding more roles never fixes BOLA. Only a check that mentions the object does.

OWASP's rule for the object check is blunt: check "if the logged-in user has access to
perform the requested action on the record in every function that uses an input from the
client to access a record in the database" (API1:2023, How To Prevent). The cheat sheet
puts it as "Just because a user has access to an object of a particular type does not mean
they should have access to every object of that particular type."

Hard-to-guess IDs help less than they look. Random UUIDs stop someone enumerating `8812`,
`8813`, `8814`, but IDs leak through links, logs and shared screenshots, and the cheat sheet
calls masking them "generally not sufficient by itself".

## Where the tenant comes from

In a multi-tenant system most object checks reduce to one: is this row in the caller's
**tenant**, the customer organisation the caller belongs to? That makes the source of the
tenant ID the whole game.

| Source of the tenant ID | Who controls it | Verdict |
|---|---|---|
| A path or query parameter, `?tenant=…` | The caller | Never trust it on its own |
| A field in the request body | The caller | Never trust it on its own |
| A header the client sets | The caller | Never trust it on its own |
| A claim in a token the server verified, or a server-side session | The server that issued it | The default |

A tenant taken from the request is a tenant the caller chose. The cheat sheet's first
mitigation is to avoid exposing the identifier at all where you can: retrieve objects "based
solely on currently authenticated user's identity and attributes (e.g. through information
contained in a securely implemented JSON Web Token (JWT) or server-side session)."

There's one legitimate reason to accept a tenant from the request: a platform operator who
acts on behalf of a customer. The safe shape is to resolve the **effective tenant** once, in
middleware, and to honour a requested tenant only when the caller's role allows it. Every
handler then reads one value and none of them re-implements the rule.

```go
type Claims struct { // from a token the auth middleware has already verified
    UserID   string
    TenantID string
    Role     string // "member", "tenant_admin" or "platform_admin"
}

var errForbidden = errors.New("forbidden")

// effectiveTenant decides which tenant this request acts on. Only a platform admin
// may name another tenant; anyone else naming one is refused.
func effectiveTenant(c Claims, requested string) (string, error) {
    if requested == "" || requested == c.TenantID {
        return c.TenantID, nil
    }
    if c.Role == "platform_admin" {
        return requested, nil
    }
    return "", errForbidden
}

func WithTenant(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        t, err := effectiveTenant(ClaimsFrom(r.Context()), r.URL.Query().Get("tenant"))
        if err != nil {
            http.Error(w, "forbidden", http.StatusForbidden)
            return
        }
        next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), tenantKey, t)))
    })
}

// In a handler: the tenant is a bound parameter, ANDed at the top level.
const findCustomer = `
    SELECT id, name, status FROM customers
    WHERE tenant_id = $1 AND deleted_at IS NULL
      AND (email = $2 OR phone = $3)`

rows, err := db.QueryContext(ctx, findCustomer, TenantFrom(ctx), email, phone)
```

<details><summary>Rust</summary>

```rust
#[derive(PartialEq)]
enum Role { Member, TenantAdmin, PlatformAdmin }

struct Claims { user_id: String, tenant_id: String, role: Role }

struct Forbidden;

fn effective_tenant(c: &Claims, requested: Option<&str>) -> Result<String, Forbidden> {
    match requested {
        None => Ok(c.tenant_id.clone()),
        Some(t) if t == c.tenant_id => Ok(t.to_string()),
        Some(t) if c.role == Role::PlatformAdmin => Ok(t.to_string()),
        Some(_) => Err(Forbidden), // answer 403
    }
}

// With sqlx: the tenant is bound, and ANDed at the top level.
let customers = sqlx::query_as::<_, Customer>(
    "SELECT id, name, status FROM customers
     WHERE tenant_id = $1 AND deleted_at IS NULL
       AND (email = $2 OR phone = $3)",
)
.bind(&tenant)
.bind(email)
.bind(phone)
.fetch_all(&pool)
.await?;
```

</details>

<details><summary>C++</summary>

```cpp
enum class Role { Member, TenantAdmin, PlatformAdmin };
struct Claims { std::string user_id, tenant_id; Role role; };

// std::nullopt means "refuse with 403".
std::optional<std::string> effective_tenant(const Claims& c,
                                            const std::optional<std::string>& requested) {
    if (!requested || *requested == c.tenant_id) return c.tenant_id;
    if (c.role == Role::PlatformAdmin) return *requested;
    return std::nullopt;
}

// Bind the tenant as a query parameter with your driver, exactly as in the Go
// version: WHERE tenant_id = $1 AND deleted_at IS NULL AND (email = $2 OR phone = $3)
```

</details>

<details><summary>JavaScript</summary>

```js
// Express middleware; req.claims was set by the auth middleware after verifying the token.
function withTenant(req, res, next) {
  const { tenantId, role } = req.claims;
  const requested = req.query.tenant;
  if (!requested || requested === tenantId) {
    req.tenant = tenantId;
  } else if (role === 'platform_admin') {
    req.tenant = requested;
  } else {
    return res.status(403).json({ error: 'forbidden' });
  }
  next();
}

// node-postgres: the tenant is bound, and ANDed at the top level.
const { rows } = await pool.query(
  `SELECT id, name, status FROM customers
   WHERE tenant_id = $1 AND deleted_at IS NULL
     AND (email = $2 OR phone = $3)`,
  [req.tenant, email, phone],
);
```

</details>

Refusing a non-admin who names another tenant, rather than quietly giving them their own,
is a choice. Quietly substituting is also safe, but it hides a client bug or a probe that a
`403` and a log line would show you.

### The tenant goes in an AND, at the top

Look again at the `WHERE` clause. A lookup "by any of several keys" is where tenant scoping
most often breaks, because the tenant condition ends up beside the keys instead of above
them.

```sql
-- Wrong: AND binds tighter than OR, so this is (tenant AND email) OR phone.
-- A phone number from any tenant matches.
WHERE tenant_id = $1 AND email = $2 OR phone = $3

-- Wrong in a different way: the tenant is one of the alternatives,
-- so every row in the tenant matches.
WHERE tenant_id = $1 OR email = $2 OR phone = $3

-- Right: the scope is ANDed with the whole group of alternatives.
WHERE tenant_id = $1 AND deleted_at IS NULL AND (email = $2 OR phone = $3)
```

The first is operator precedence: in PostgreSQL's table, `AND` sits above `OR`
([§4.1.6](https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-PRECEDENCE)).
The second is the same mistake a document database makes easy. MongoDB "provides an
implicit `AND` operation when you specify a comma separated list of expressions"
([`$and`](https://www.mongodb.com/docs/manual/reference/operator/query/and/)), and `$or`
"selects documents that satisfy at least one of the expressions"
([`$or`](https://www.mongodb.com/docs/manual/reference/operator/query/or/)). So the tenant
belongs beside the `$or`, not inside it:

```js
// Wrong: "any of these keys matches, OR the row is in this tenant".
{ deleted: false, $or: [{ email }, { phone }, { tenantId }] }

// Right: tenant and soft-delete are top-level, implicitly ANDed with the $or.
{ tenantId, deleted: false, $or: [{ email }, { phone }] }
```

> ⚠️ The wrong versions return rows, so nothing errors and a test that looks up a real
> customer passes. The bug shows only when the keys match nothing in the caller's tenant and
> the query returns some other row instead. Test object lookups with keys that belong to
> another tenant, and with keys that match nothing.

### `403` or `404` for someone else's object

When a caller asks for a row that exists in another tenant, you can say "forbidden" or "no
such thing". RFC 9110 allows either: "An origin server that wishes to "hide" the current
existence of a forbidden target resource MAY instead respond with a status code of 404 (Not
Found)" ([§15.5.4](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.4)). A tenant
filter in the query gives you the `404` for free, since the row simply isn't found. That's
usually what you want: a `403` confirms the ID exists, which is a small leak of its own.

## Permission and entitlement are separate checks

Roles say what a *person* may do. Multi-tenant products usually have a second axis: what
the *tenant* has bought or switched on. A customer without the reporting module shouldn't
reach the reporting routes, whatever roles its users hold.

| Check | Question | Comes from |
|---|---|---|
| **Permission** | Does this caller's role grant `reports:read`, in this tenant? | Role assignments |
| **Entitlement** | Has this tenant got the reporting product at all? | The tenant's plan or feature flags |
| Allowed | Both | — |

Two mistakes recur. The first is letting high privilege skip the entitlement check: "admins
can do anything" lets an operator acting for a tenant run a product the tenant never had,
with whatever billing or legal consequences follow. The role decides who the caller is; the
entitlement belongs to the *target tenant*, and applies whoever is acting on it.

The second is answering the question in two places. A dashboard asks the server "what may I
do?" to decide which buttons to show, and the server's request gate decides what actually
runs. If the listing applies the entitlement and the gate doesn't, the button is hidden and
the API still answers anyone who calls it directly. If the gate applies it and the listing
doesn't, users see buttons that fail. Put both checks in one function and call it from both
places.

```go
// Allowed is the only place that answers "may this caller do this, in this tenant?".
// The request gate calls it, and so does GET /me/permissions.
func (a *Authz) Allowed(c Claims, tenant, resource, action string) bool {
    if !a.roleGrants(c, tenant, resource, action) { // permission, per tenant
        return false
    }
    product := a.productOf(resource) // "" for core features every tenant has
    return product == "" || a.tenantHas(tenant, product) // entitlement: no admin bypass
}

func (a *Authz) Permissions(c Claims, tenant string) []string {
    var out []string
    for _, p := range a.allPermissions { // e.g. {"reports", "read"}
        if a.Allowed(c, tenant, p.Resource, p.Action) {
            out = append(out, p.Resource+":"+p.Action)
        }
    }
    return out
}
```

<details><summary>Rust</summary>

```rust
impl Authz {
    /// The only place that answers "may this caller do this, in this tenant?".
    pub fn allowed(&self, c: &Claims, tenant: &str, resource: &str, action: &str) -> bool {
        if !self.role_grants(c, tenant, resource, action) {
            return false;
        }
        match self.product_of(resource) {
            None => true,                                   // core feature
            Some(product) => self.tenant_has(tenant, product), // no admin bypass
        }
    }

    pub fn permissions(&self, c: &Claims, tenant: &str) -> Vec<String> {
        self.all_permissions
            .iter()
            .filter(|p| self.allowed(c, tenant, &p.resource, &p.action))
            .map(|p| format!("{}:{}", p.resource, p.action))
            .collect()
    }
}
```

</details>

<details><summary>C++</summary>

```cpp
// The only place that answers "may this caller do this, in this tenant?".
bool Authz::allowed(const Claims& c, const std::string& tenant,
                    const std::string& resource, const std::string& action) const {
    if (!role_grants(c, tenant, resource, action)) return false;
    std::optional<std::string> product = product_of(resource);  // nullopt: core feature
    return !product || tenant_has(tenant, *product);            // no admin bypass
}

std::vector<std::string> Authz::permissions(const Claims& c, const std::string& tenant) const {
    std::vector<std::string> out;
    for (const auto& p : all_permissions_)
        if (allowed(c, tenant, p.resource, p.action)) out.push_back(p.resource + ":" + p.action);
    return out;
}
```

</details>

<details><summary>JavaScript</summary>

```js
// The only place that answers "may this caller do this, in this tenant?".
function allowed(claims, tenant, resource, action) {
  if (!roleGrants(claims, tenant, resource, action)) return false;
  const product = productOf(resource); // null for core features
  return product === null || tenantHas(tenant, product); // no admin bypass
}

function permissions(claims, tenant) {
  return ALL_PERMISSIONS
    .filter((p) => allowed(claims, tenant, p.resource, p.action))
    .map((p) => `${p.resource}:${p.action}`);
}
```

</details>

Policy engines model the permission half directly. Casbin's "RBAC with domains" adds the
tenant as a third field of every role assignment, so a user can be an admin in one tenant
and a plain user in another, and its matcher requires the request's tenant to equal the
policy's ([Casbin docs](https://casbin.apache.org/docs/rbac-with-domains/)):

```
m = g(r.sub, p.sub, r.dom) && r.dom == p.dom && r.obj == p.obj && r.act == p.act
```

That covers "may this role do this in this tenant". It doesn't know what the tenant bought,
so the entitlement still needs its own check, inside the same function.

The cheat sheet goes further than roles-per-tenant. It recommends that "ABAC and ReBAC
should typically be preferred for application development": **attribute-based** access
control, which decides from attributes of the caller, the object and the environment, and
**relationship-based**, which decides from relationships such as "created this post". Its
argument is that pure RBAC "does a poor job of supporting object-level or horizontal access
control decisions". Per-tenant roles plus a tenant condition in every query is, in effect, a
small attribute rule bolted onto RBAC, and for most dashboards that's enough. When rules
start depending on the object itself (only the author may edit, only the assigned reviewer
may approve), that's the point the cheat sheet is making.

A role carried inside a token has one more cost: it doesn't change when the role does. A
user demoted at noon keeps the old role until the token expires, for the same reason a
signed session can't be ended early ([file 09, "How a session ends"](09-keeping-a-browser-signed-in.md#how-a-session-ends)).
Short-lived access tokens, or a server-side lookup of the current role, close that gap.

## What a response may contain

Object-level checks decide which *rows* a caller gets. A second flaw lives inside the row:
the caller is allowed this customer, but not every field of it. OWASP calls it **Broken
Object Property Level Authorization**, merging two older entries: an endpoint that "exposes
properties of an object that are considered sensitive and should not be read by the user",
once called Excessive Data Exposure, and one that lets a user change a property they
shouldn't, once called Mass Assignment
([API3:2023](https://api-security.owasp.org/editions/2023/en/0xa3-broken-object-property-level-authorization/)).

The read side usually looks like this. A handler loads a record and returns it, because
that's the least code. Someone notices an internal storage path in the JSON and adds a line
that deletes it. Two weeks later the same path turns up in a different endpoint that returns
the same record.

| | **Denylist**: remove what's secret | **Allowlist**: build what's allowed |
|---|---|---|
| Code | `delete record.storagePath` | `{ id, name, email }` |
| A new column is added | Exposed until someone notices | Absent until someone adds it |
| A new endpoint returns the record | Must repeat every delete | Reuses the view, or exposes nothing |
| Failure mode | A leak | A missing field, which someone reports |

OWASP's advice is the right-hand column: "Avoid using generic methods such as `to_json()`
and `to_string()`. Instead, cherry-pick specific object properties you specifically want to
return." In Go that means a response type separate from the stored one, so adding a column
to the record adds nothing to any response.

```go
// What the service stores.
type Customer struct {
    ID, TenantID, Name, Email string
    DocumentKey               string // object-storage path: internal
    RiskScore                 int    // internal
}

// What a caller may see. Only these fields can ever be serialised.
type CustomerView struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`
}

func viewOf(c Customer, mayReadContact bool) CustomerView {
    v := CustomerView{ID: c.ID, Name: c.Name}
    if mayReadContact {
        v.Email = c.Email
    }
    return v
}
```

<details><summary>Rust</summary>

```rust
use serde::Serialize;

#[derive(Serialize)]
struct CustomerView {
    id: String,
    name: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    email: Option<String>,
}

fn view_of(c: &Customer, may_read_contact: bool) -> CustomerView {
    CustomerView {
        id: c.id.clone(),
        name: c.name.clone(),
        email: may_read_contact.then(|| c.email.clone()),
    }
}
```

</details>

<details><summary>C++</summary>

```cpp
#include <nlohmann/json.hpp>

// Build the JSON field by field; nothing reaches it unless it is named here.
nlohmann::json view_of(const Customer& c, bool may_read_contact) {
    nlohmann::json v = {{"id", c.id}, {"name", c.name}};
    if (may_read_contact) v["email"] = c.email;
    return v;
}
```

</details>

<details><summary>JavaScript</summary>

```js
// Fastify serialises a response with its schema; properties the schema doesn't list are
// dropped. The handler still decides who may see email: the schema is the safety net.
fastify.get('/customers/:id', {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          name: { type: 'string' },
          email: { type: 'string' },
        },
      },
    },
  },
}, async (req) => viewOf(await loadCustomer(req.tenant, req.params.id), mayReadContact(req)));
```

</details>

A response schema is OWASP's "extra layer": "Implement a schema-based response validation
mechanism as an extra layer of security." Fastify's own docs say an output schema can "help
prevent accidental disclosure of sensitive information"
([Validation and Serialization](https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/)),
and the serialiser underneath it ignores "every property that is not explicitly listed"
unless the schema sets `additionalProperties`
([fast-json-stringify](https://github.com/fastify/fast-json-stringify#additionalProperties)).
It's a net, not the decision: a field the schema lists goes to everyone the handler returns
it to.

Three variations on the same mistake:

- **The caller chooses the projection.** A `fields=` parameter, a GraphQL selection, or a
  "response template" sent in the request body all decide which properties come back. That
  makes them an authorisation decision. Resolve the allowed set on the server, from the
  caller's identity and tenant, and intersect any requested fields with it.
- **A conversion fails and the raw value stays.** A handler that turns internal storage
  keys into signed URLs or inline images should drop a field whose conversion failed. Leaving
  the original value in place ships the internal path, which is exactly what the conversion
  was there to hide. Build the output from what succeeded, rather than editing the input.
- **Large blobs ride along.** Embedded images or documents in a list endpoint aren't only a
  size problem; they're properties too. Serve them from their own endpoint, with their own
  check, when someone asks.

The write side is the mirror image. Bind the request body to a type that holds only the
fields a client may change, rather than decoding it straight into the stored record, so a
`"role": "admin"` in a profile update has nowhere to land. OWASP: "Allow changes only to the
object's properties that should be updated by the client."

## What the browser can enforce: nothing

Single-page apps make authorisation look like a front-end problem, because the front end is
where users see it. The cheat sheet settles who enforces: "Developers must never rely on
client-side access control checks. While such checks may be permissible for improving the
user experience, they should never be the decisive factor in granting or denying access to a
resource." OWASP's Top 10 has a scenario for it: an application that "puts all of their
access control in their front-end", bypassed with `curl` (A01:2025, scenario #3).

Client-side gating still matters for the experience, and it has traps of its own.

**A hidden tab still fetches.** Hiding a component doesn't stop the request it triggers.
If the data loads on page entry, it's in the Network tab for anyone who looks. Tie the fetch
to the same condition as the display. In TanStack Query that's the `enabled` option, whose
documentation is about exactly this, a query that should wait for something:
"The query will not execute until the userId exists"
([Dependent Queries](https://tanstack.com/query/latest/docs/framework/react/guides/dependent-queries)).
This is browser code, so there's no Go version:

```js
const { data: documents } = useQuery({
  queryKey: ['documents', customerId],
  queryFn: () => fetchDocuments(customerId),
  enabled: user?.isAdmin === true, // never sent for anyone else
});
```

**"Not known yet" isn't "no".** On the first render the identity may not be resolved: the
token hasn't been decoded, the permissions request hasn't returned. A guard that treats a
missing user as unauthorised sends an admin to the "access denied" page before it has found
out they're an admin. A guard needs three states, not two.

| Identity | Permissions | The guard shows |
|---|---|---|
| Not resolved | — | Loading |
| Resolved | Grant present | The page |
| Resolved | Grant absent | Denied |

TanStack Query reports the first row itself: a query that isn't enabled yet starts with
`status: 'pending'` and `fetchStatus: 'idle'`, so "pending" covers both "fetching" and
"waiting to be allowed to fetch".

**Cached permissions go stale.** A grant revoked by an administrator stays in the client's
cache until it refetches. The server's next answer is a `403`, which is the server doing its
job. The client's part is to treat a `403` as a reason to refetch its permissions, so the UI
catches up with the decision that was already made.

## Check yourself

1. A route requires `role >= manager` and then runs `SELECT * FROM invoices WHERE id = $1`.
   A manager at tenant A requests an invoice ID that belongs to tenant B. What comes back,
   which of the two questions was never asked, and why would comparing the token's user ID
   with a user ID in the URL not have fixed it?
2. A lookup reads `WHERE tenant_id = $1 AND email = $2 OR phone = $3`. A caller in tenant A
   passes an email that matches nothing and a phone number that belongs to a customer in
   tenant B. Which rows come back? Then answer for the MongoDB filter
   `{ $or: [{ email }, { phone }, { tenantId }] }` with an email and a phone that match
   nothing.
3. A platform operator, acting for tenant B through a `?tenant=B` parameter, presses
   "retry" on a failed job. The retry handler reads the tenant from the operator's token.
   What happens, and what design stops each new handler from making the same mistake?
4. The permissions listing hides the reports menu from users whose tenant hasn't bought
   reporting, but the request gate checks roles only. What can a user in such a tenant do,
   and how? What goes wrong if you fix it by adding the entitlement check to the gate alone,
   in its own code?
5. A customer endpoint returns the stored record after deleting two internal fields. A
   migration adds a column of internal notes, and a new export endpoint reuses the same
   loader. Which responses now contain the notes? Redesign it so the next migration can't
   leak anything, and say what you'd use as a second layer.
6. A dashboard hides the Documents tab for non-admins, and the tab's query runs when the
   page loads. Separately, its route guard redirects to "access denied" whenever `user` is
   `null`. What can a non-admin see, what happens to an admin on a hard refresh, and what does
   each fix change on the server?

(Answers in [`11-exercises.md`](11-exercises.md).)
