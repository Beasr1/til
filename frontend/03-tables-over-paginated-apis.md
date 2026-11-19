# 3. Tables over paginated APIs

Assumes file 01 and file 02 §2.3 (every input belongs in the query key). The server side of
the same problem — why `OFFSET` drifts and how keyset queries work — is
[postgres/05-pagination.md](../postgres/05-pagination.md). This chapter is the client's half.

## 3.1 The problem

A transactions page has a table, a row of filter dropdowns, and four stats cards: total,
succeeded, failed, today. In testing, with forty rows, everything agrees. In production:

- Filtering by "failed" shows two rows, though the failed card says 312.
- The cards themselves are wrong: they were computed in the browser over a fetch of the first
  1,000 rows, and there are 40,000.
- Elsewhere, a catalogue page that "loads everything" shows 20 items. The API has a default page
  size of 20, and nobody passed a limit.

Nothing errors. Every number is a correct answer about the rows the browser happens to hold,
presented as an answer about all of them.

## 3.2 The browser holds a page, not the table

Once the server paginates, the browser has a window onto the result set:

```
full result set (server)    ████████████████████████████████████████  40,000 rows
                            ▲
the page you hold           ██                                         20 rows
                            └─ sort, filter, count here = sort, filter, count THIS
```

Every operation you run on the client now answers a question about the window. TanStack Table's
guide says it in so many words: "client-side sorting after server-side pagination sorts the
current page, not every matching row. Client-side filtering after server-side pagination can
hide rows on the current page without finding matches on other pages. These results are usually
misleading." And the rule that follows: "when the server owns pagination, it should also own any
filtering, grouping, sorting, or aggregation that must apply to the full result set"
([TanStack Table, Client-Side vs Server-Side](https://github.com/TanStack/table/blob/main/docs/guide/client-side-vs-server-side.md)).
Facet counts — the "(312)" next to a filter option — are aggregates too, and the same guide says
a facet count computed from a page "describe[s] only that page".

| Operation | Done in the browser over one page | Done on the server |
|---|---|---|
| Sort | Sorts 20 rows; row 21 might belong first | Correct order across all rows |
| Filter | Hides non-matches on this page; finds nothing on other pages | Correct matches, re-paginated |
| Count / sum / facet | Counts this page | Correct |
| Formatting, column visibility, row selection on screen | Fine | Pointless round trip |

The last row is the honest exception: anything that only concerns what's on screen can stay in
the browser. The guide allows mixing "when the smaller scope is intentional" and says to make
that scope clear in the UI.

## 3.3 "Manual" mode: the table manages state, you fetch

TanStack Table never fetches. Its `manual*` options tell it the data you pass is "already
processed for that feature", so it stops applying its own row models:

| Feature | Option |
|---|---|
| Filtering (column and global) | `manualFiltering` |
| Sorting | `manualSorting` |
| Pagination | `manualPagination` |
| Grouping | `manualGrouping` |

The guide's checklist for server-side processing, condensed
([same source](https://github.com/TanStack/table/blob/main/docs/guide/client-side-vs-server-side.md)):

1. Own the filter, sort and pagination state yourself, so your fetching code can read it.
2. **Include every server-owned state value in the request or query key.**
3. Turn on the matching `manual*` options and pass the server's rows as `data`.
4. For pagination, pass `rowCount` (or `pageCount`) so the table knows how many pages exist —
   it has "no way of knowing" otherwise
   ([pagination guide](https://github.com/TanStack/table/blob/main/docs/framework/react/guide/pagination.md)).
5. **Reset the page index when filters, sorting or page size change.**
6. Keep previous results deliberately, and don't let a slow old response overwrite a newer one.

Step 2 is file 02 §2.3 again: the filter goes in the query key, so changing it fetches. Step 6
is also handled by the cache: each response lands under its own key, so a late reply for the old
filter fills the old entry and never replaces what the new filter shows.

Step 5 needs care. By default the table resets `pageIndex` to 0 when its client-side row models
recompute, but `manualPagination` turns that off, and in a fully manual table the filter and sort
row models that would trigger it aren't there at all. The guide says to reset the page index
yourself, in the change handlers. Forget, and a user on page 9 who narrows a filter to three
matching rows is shown an empty page 9.

> I checked these against the docs on TanStack Table's `main` branch, which is v9 as of November
> 2026. `manualPagination`, `rowCount`, `pageCount` and `autoResetPageIndex` have the same names
> in v8 (checked in the type declarations of `@tanstack/table-core` 8.21.3); the client-side row
> model options are spelled differently (`getFilteredRowModel()` in v8).

## 3.4 Counts come from the server

The stats cards need numbers about the whole filtered set, so they need an endpoint that
computes them: `GET /transactions/stats?status=…&from=…&to=…`, taking the same filters as the
list and returning the counts. Its query key carries the same filters, so the cards and the
table always describe the same set.

The tempting alternative is "fetch a big page and count it". That isn't an approximation that
gets worse slowly; it's exact up to the limit and then silently wrong. If you ever have to read a
whole list in the browser, treat a full page as a warning:

```ts
const rows = await fetchItems({ limit: 500 });
if (rows.length === 500) {
  // You have no idea whether there are 501 or 50,000. Either page until a short page
  // arrives, or show the result as truncated. Never count it as the total.
}
```

And an endpoint called without a limit is still paginated if the server has a default. Read the
API's contract for its default page size before you trust "fetch all".

## 3.5 Offset or keyset — the one-paragraph version

Offset pagination (`?page=9&size=20`) asks the server to skip 160 rows. Markus Winand's
[No Offset](https://use-the-index-luke.com/no-offset) gives the two costs: the database "must
still fetch these rows" before it can skip them, so deep pages get slower; and when rows are
inserted between two page loads, "you'll get duplicates". Keyset pagination (also called the
seek method, or cursor pagination) instead asks for "rows after the last one I saw", which stays
fast and doesn't drift — but "you cannot directly navigate to arbitrary pages". The database
side, including the tie-breaker column you need, is
[postgres/05 — Keyset pagination](../postgres/05-pagination.md#keyset-pagination).

That last limitation is where the client comes in.

## 3.6 Page numbers on top of cursors

Product wants page numbers. The API returns a page of rows plus an opaque `nextCursor`. Page 4
can only be fetched with the cursor that page 3's response handed you. So the client keeps a map:

```
page → cursor that fetches it
  1  → (start)
  2  → "c_8f1a"     ← from page 1's response
  3  → "c_2b77"     ← from page 2's response
  4  → ?            ← not reached yet: no way to ask for it directly
```

Two rules make this correct:

- **You can go to any page you've reached, plus the next one.** Back to page 2 works; jumping
  from page 3 to page 9 doesn't, because nobody has handed you page 9's cursor. The UI should
  offer Previous, Next and the visited page numbers, and disable Last when there's no total
  (TanStack's pagination guide does the same with `pageCount: -1`).
- **A cursor is a position in one specific ordering of one specific filtered set.** Change the
  filter, the sort, or the page size and every stored cursor points into a list that no longer
  exists. Clear the map and go back to page 1.

```go
// PageCursors maps each reached page to the cursor that fetches it.
// The cursors are only valid for the query (filters + sort + page size) they came from.
type PageCursors struct {
	query   string
	cursors map[int]string
}

// For returns the cursor for page, resetting the map if the query changed.
// ok is false when the page hasn't been reached, so it can't be fetched yet.
func (p *PageCursors) For(query string, page int) (cursor string, ok bool) {
	if p.cursors == nil || query != p.query {
		p.query = query
		p.cursors = map[int]string{1: ""} // page 1 starts from the beginning
	}
	cursor, ok = p.cursors[page]
	return cursor, ok
}

// Record stores the cursor a page's response returned for the page after it.
func (p *PageCursors) Record(page int, next string) {
	p.cursors[page+1] = next
}
```

<details><summary>Rust</summary>

```rust
use std::collections::HashMap;

#[derive(Default)]
struct PageCursors {
    query: String,
    cursors: HashMap<u32, String>,
}

impl PageCursors {
    /// None when the page hasn't been reached and so can't be fetched yet.
    fn cursor_for(&mut self, query: &str, page: u32) -> Option<&String> {
        if self.cursors.is_empty() || self.query != query {
            self.query = query.to_string();
            self.cursors = HashMap::from([(1, String::new())]);
        }
        self.cursors.get(&page)
    }

    fn record(&mut self, page: u32, next: String) {
        self.cursors.insert(page + 1, next);
    }
}
```

</details>

<details><summary>C++</summary>

```cpp
#include <map>
#include <optional>
#include <string>

class PageCursors {
    std::string query_;
    std::map<int, std::string> cursors_;

public:
    // nullopt when the page hasn't been reached and so can't be fetched yet.
    std::optional<std::string> cursorFor(const std::string& query, int page) {
        if (cursors_.empty() || query != query_) {
            query_ = query;
            cursors_ = {{1, ""}};
        }
        auto it = cursors_.find(page);
        if (it == cursors_.end()) return std::nullopt;
        return it->second;
    }

    void record(int page, std::string next) { cursors_[page + 1] = std::move(next); }
};
```

</details>

<details><summary>JavaScript</summary>

```js
class PageCursors {
  #query = null;
  #cursors = new Map();

  // undefined when the page hasn't been reached and so can't be fetched yet.
  cursorFor(query, page) {
    if (query !== this.#query) {
      this.#query = query;
      this.#cursors = new Map([[1, null]]);
    }
    return this.#cursors.get(page);
  }

  record(page, next) {
    this.#cursors.set(page + 1, next);
  }
}
```

</details>

Pass the filters, sort and page size, serialised in a stable order, as `query`. In the Go,
C++ and Rust versions an empty string stands for "start from the beginning"; in the JavaScript
one it's `null`, so `undefined` can mean "not reached".

In React with TanStack Query you often don't write this class at all.
`useInfiniteQuery` stores the fetched pages in an array, each fetched with the previous page's
`nextCursor`, and the pagination guide's cursor example puts the filters, sort and page size in
the query key. A filter change is then a different key, with its own empty page list: the reset
happens because the key changed. The one thing you still do by hand is set `pageIndex` back to 0,
which the guide calls out.

> **Teacher's aside.** People think of a cursor as a page number the server encoded. It isn't.
> It's a bookmark in a particular book: "after the row with `created_at = …, id = …`, in this
> order, among rows matching these filters". Hand the same bookmark to a different filter and
> you get a page that starts at an arbitrary point in the new list — no error, just the wrong
> rows, and duplicates or gaps as the user pages on. That's why the map is keyed by query, and
> why the reset isn't optional.

## Check yourself

1. A table shows 20 rows per page from the server and uses TanStack Table's default client-side
   filtering. A user filters for a customer whose orders are all on page 7 while viewing page 1.
   What do they see, and why does nothing indicate a problem?
2. The stats cards call `fetchTransactions({ limit: 1000 })` and count by status. Describe the
   numbers they show at 400 rows, at 1,000 rows, and at 40,000. What single check would at least
   have made the problem visible?
3. You switch a table to `manualPagination` and `manualFiltering`, put the filters in the query
   key, and pass `rowCount`. A user on page 9 narrows the filter to three results. What do they
   see, and which line of code was missing?
4. Using the page→cursor map, a user visits pages 1, 2, 3, goes back to 1, and changes the sort.
   Which entries does the map hold after each step? What would page 2 show if the map had not
   been cleared?
5. Why can a keyset-paginated UI offer "page 3" after you've seen page 3, but not "page 9" before
   you've reached it? What would the server need to provide to make "Last" work, and what does
   that cost on the database side?
