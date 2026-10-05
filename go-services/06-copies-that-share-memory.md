# 6. Copies that share memory

## The problem

A function needs a modified version of a request. Say it watermarks the photos before
rendering a document, and the original must still be stored untouched. So it copies the
request struct, changes the copy, and renders it. Later someone notices the stored record
has watermarked photos too. The copy wasn't a copy. In Go, assigning a struct copies its
fields, and some kinds of field are themselves only references to memory that both copies
now share.

## What `b := a` copies

Assigning a struct copies each field's value. For some types the value *is* the data. For
others the value is a small header that points at the data:

| Field type | What the copy gets | Mutating through the copy… |
|---|---|---|
| `int`, `string`, `bool`, arrays, nested structs of these | Its own data | Doesn't affect the original |
| `string` | A pointer to immutable bytes | Can't mutate it; reassigning is safe |
| Slice (`[]T`, including `[]byte` and `json.RawMessage`) | A header: pointer, length, capacity | **Element writes show up in the original** |
| Map | A pointer to the same hash table | **Every write shows up in the original** |
| Pointer, channel, function | The same reference | Shared, by definition |
| `interface{}` holding any of the above | The same dynamic value | Shared if the value is a reference type |

Run on Go 1.26, with a struct holding all three kinds:

```go
type Request struct {
    ID       string
    Snapshot json.RawMessage
    Meta     map[string]any
    Tags     []string
}

cp := orig
cp.ID = "b"                       // independent
cp.Meta["photo"] = "watermarked"  // shared map
cp.Tags[0] = "y"                  // shared backing array
copy(cp.Snapshot[2:7], "PHOTO")   // shared bytes
cp.Snapshot = json.RawMessage(`{"photo":"watermarked"}`) // new slice: safe
```

```
orig.ID=a orig.Meta=map[photo:watermarked] orig.Tags=[y]
orig.Snapshot={"PHOTO":"raw"}
```

The ID survived. The map, the slice element and the in-place byte write all reached the
original. The final line is the subtle one. *Reassigning* `cp.Snapshot` to a new slice was
harmless. Only writing *into* the shared bytes leaked. Whether a "copy" is safe depends on
whether the code downstream assigns new values or mutates in place, and that's usually
several function calls away from the copy.

## JSON documents are the common case

The type that causes this most is the generic JSON document: `map[string]any`, possibly
nested, from `json.Unmarshal` into an `any`. Every object inside it is another map. So
even a correct clone of the *top* level shares everything below it:

```go
c2 := maps.Clone(o2)                                   // Go 1.21+
c2["doc"].(map[string]any)["photo"] = "watermarked"
// o2 is now map[doc:map[photo:watermarked]]
```

`maps.Clone` and `slices.Clone` are **shallow**. They copy one level.

## Ways to get a real copy

| Approach | Deep? | Cost and caveats |
|---|---|---|
| `maps.Clone`, `slices.Clone` | One level | Fine when the elements are values |
| Marshal to JSON and unmarshal into a fresh value | Yes | Slow for big documents. Loses types JSON can't represent, so numbers become `float64` in an `any` |
| A reflection-based deep-copy library | Usually | Read its limits. For example `mohae/deepcopy`'s README says "unexported field values are not copied", so a struct with private state comes back partly zeroed |
| A hand-written `Clone()` method on the type | Yes, by construction | The most code, and the only one that's obviously correct for your type. It needs updating when fields are added |
| Don't mutate: build a new value from the old | n/a | Often the cleanest. The watermarking step returns a new document instead of editing one in place |

> **Teacher's aside.** People reach for a deep-copy library when the real problem is that
> a function mutates its input. "Takes a document, returns a watermarked document" can't
> leak into the caller. "Takes a document and watermarks it" always can, however carefully
> the caller copies first. If you control the function, change its contract. Copy only
> when you're calling code you can't change.

## Data races are the same bug, concurrently

When the two "copies" are used from different goroutines, a shared map isn't just a
logic bug. Concurrent map writes crash the program ("fatal error: concurrent map
writes"), and concurrent slice writes are a data race. The Go race detector
(`go test -race`) finds these, but only on code paths the tests actually run concurrently.

## Check yourself

1. A struct has fields `Name string`, `Photo []byte` and `Attrs map[string]string`. After
   `b := a`, which of these affect `a`: `b.Name = "x"`, `b.Photo = nil`,
   `b.Photo[0] = 0`, `b.Attrs["k"] = "v"`, `b.Attrs = nil`?
2. Why is reassigning `cp.Snapshot` safe while `copy(cp.Snapshot[…], …)` isn't, given that
   both "change the snapshot"?
3. You `maps.Clone` a decoded JSON document and then set a key two levels down. What
   happens to the original, and what would you use instead?
4. A deep-copy library is applied to a struct holding a `sync.Mutex` and an unexported
   cache field. Predict the result, using what the library documents.
5. Rewrite the contract of `watermark(doc map[string]any)` so that no caller can be
   affected by it, without copying anything at the call site.
