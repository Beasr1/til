# 6. Rendering traps — two trees, loading boundaries, and refs

Assumes file 01 §1.3 (state lives at a position in the tree). Belongs conceptually right after
file 01; it's numbered 06 because it was added later.

## 6.1 The problem

Two bugs from the same kind of screen, a table of records:

- Each row is clickable and opens the record. Each row also has a "…" menu with Download. Clicking
  Download downloads the file *and* opens the record. The author had reasoned that the menu
  renders in a portal, outside the row in the DOM, so a click inside it couldn't reach the row.
- Above the table sit filter inputs. Typing one character starts a fetch, and the input loses
  focus. Typing a word means clicking back into the box after every letter.

The first is about which tree an event travels up. The second is about which tree state lives
in. Both trees are React's, and neither is the DOM you see in the inspector.

## 6.2 The React tree is not the DOM tree

A **portal** renders a component's output into a different DOM node — usually one attached to
`<body>` — so that menus, tooltips and dialogs escape a parent's `overflow: hidden` and stacking
context. Many menu and dialog libraries render their content through one.

```
React tree (what you wrote)              DOM tree (what the inspector shows)

<Table>                                  <body>
  <Row onClick={open}>                     <div id="root">
    <Cell>…</Cell>                           <table>
    <Menu>                                     <tr>  ← Row
      <MenuTrigger/>                             <td>…</td>
      <Portal>                                   <td><button/></td>  ← MenuTrigger
        <MenuItem onClick={download}/>         </tr>
      </Portal>                              </table>
    </Menu>                                </div>
  </Row>                                   <div data-portal>
</Table>                                     <div role="menuitem"/>  ← MenuItem
                                           </div>
                                         </body>
```

In the DOM, the menu item isn't inside the `<tr>`. In React, it is — and React's documentation is
explicit about which one events follow: "Events from portals propagate according to the React tree
rather than the DOM tree. For example, if you click inside a portal, and the portal is wrapped in
`<div onClick>`, that `onClick` handler will fire"
([react.dev, createPortal — Caveats](https://react.dev/reference/react-dom/createPortal#caveats)).
The same page says a portal "only changes the physical placement of the DOM node. In every other
way, the JSX you render into a portal acts as a child node of the React component that renders
it" — context flows through it the same way.

So the click on Download bubbles: `MenuItem` → `Portal` → `Menu` → `Row`, and the row's `onClick`
runs.

> **Teacher's aside.** The inspector shows you the DOM, so that's the tree people reason about.
> But React handlers are attached to *components*, and React walks the component tree when it
> dispatches. Anything React-specific — events, context, state — follows the tree you wrote in
> JSX. Only layout, CSS and native DOM listeners follow the DOM. When a portal is involved, ask
> "which tree does this follow?" before you predict anything.

## 6.3 Three fixes, in order of preference

**1. Don't make the whole row a click target.** A row-wide `onClick` is also invisible to keyboard
users and screen readers unless you add a role and key handling. Put a real link on the primary
cell (the record's name) and the problem disappears, because nothing up the tree is listening.

**2. Let the row ignore clicks that didn't happen inside it in the DOM.** The event's `target` is
the DOM node that was clicked; `currentTarget` is the row's own element. For a click in the portal,
the target isn't a DOM descendant of the row:

```tsx
<tr onClick={(e) => {
  if (!e.currentTarget.contains(e.target as Node)) return;   // came through a portal
  open(record.id);
}}>
```

This keeps the fix in one place instead of spreading it over every item in every menu. (This is my
suggestion, following from DOM `contains` semantics; React's docs don't prescribe it.)

**3. Stop propagation inside the portal**, which is what React's caveat suggests, or move the
portal up the React tree so it isn't inside the row at all:

```tsx
<MenuItem onClick={(e) => { e.stopPropagation(); download(record.id); }} />
```

Know what `stopPropagation` does and doesn't do. MDN: it "prevents further propagation of the
current event in the capturing and bubbling phases. It does not, however, prevent any default
behaviors", and "it also does not prevent propagation to other event-handlers of the current
element" ([MDN, stopPropagation](https://developer.mozilla.org/en-US/docs/Web/API/Event/stopPropagation)).
It also stops *every* ancestor's handler, including ones you wanted, and it has to be repeated on
each item. The menu's trigger button needs it too — but for an ordinary reason: the trigger is
inside the row in both trees.

> ⚠️ When a fix like this "doesn't work", the temptation is to change something unrelated and see
> if the bug moves. Confirm the propagation path first: put a `console.log(e.target,
> e.currentTarget)` in the row's handler and click the menu item. If it logs, the event came
> through the React tree.

## 6.4 Put loading states around data, not around controls

The second bug is file 01 §1.3's rule in its most common form:

```tsx
function OrdersPage() {
  const [filters, setFilters] = useState(initial);
  const { data, isLoading } = useQuery({ queryKey: ['orders', filters], queryFn: … });

  if (isLoading) return <PageSkeleton />;          // replaces EVERYTHING, inputs included
  return <><Filters value={filters} onChange={setFilters} /><Table rows={data} /></>;
}
```

Each filter change is a new query key (file 02 §2.3) with no data, so `isLoading` is true, and the
component returns a different tree. The `<Filters>` element and everything inside it — the focused
`<input>`, its cursor position, any local state — is unmounted. When data arrives, a fresh
`<Filters>` mounts. React discards state when "a different component gets rendered at the same
position" ([react.dev, Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)),
and the DOM input goes with it, so focus does too. (Here the filter *values* survive because they
live in `OrdersPage`, above the early return; anything held inside `<Filters>` wouldn't.)

The fix is to draw the boundary around the part that depends on the data:

```tsx
return (
  <>
    <Filters value={filters} onChange={setFilters} />   {/* always mounted */}
    {isLoading ? <TableSkeleton /> : <Table rows={data} dimmed={isPlaceholderData} />}
  </>
);
```

Combined with `placeholderData` (file 02 §2.7), most filter changes won't show the skeleton at all:
the old rows stay, dimmed, until the new ones arrive.

| Placeholder wraps… | Effect of each fetch |
|---|---|
| The whole page (early return) | Inputs unmount, focus lost, page flashes |
| The data region only | Inputs untouched; table shows a skeleton |
| Nothing, with placeholder data | Inputs untouched; old rows shown dimmed until new ones arrive |

## 6.5 Refs for values that never render

A "refresh from source" button is rate-limited on the client: it remembers when it was last clicked
and refuses again within 30 seconds. The timestamp was kept in `useState`, so every click caused a
re-render that changed nothing on screen.

If a value doesn't affect what's drawn, it isn't state. That's what a **ref** is for. react.dev:
"Unlike state, setting the ref's `current` value does not trigger a re-render", and "if your
component needs to store some value, but it doesn't impact the rendering logic, choose refs"
([react.dev, Referencing Values with Refs](https://react.dev/learn/referencing-values-with-refs)).

```tsx
const lastRefresh = useRef(0);

function onRefresh() {
  const now = Date.now();
  if (now - lastRefresh.current < 30_000) return;   // read and write in a handler: fine
  lastRefresh.current = now;
  refresh();
}
```

| Use state when… | Use a ref when… |
|---|---|
| The value is shown, or decides what's shown | The value only matters inside handlers or effects |
| Changing it should update the screen | Changing it shouldn't re-render anything |
| Example: the filter text, an open/closed flag | Example: a timer ID, a last-click timestamp, a DOM node |

The rule that comes with it: "Don't read or write `ref.current` during rendering", because React
doesn't know when it changes. If the button should *look* disabled during the cooldown, the
cooldown is affecting what's drawn, and it's state again.

## Check yourself

1. A table row has `onClick={open}`; its "…" menu renders items through a portal. Clicking
   Download fires both handlers. Explain why, using both trees, and why the inspector misled the
   author.
2. You add `e.stopPropagation()` to the Download item. Clicking it still opens the record, but only
   when you click the "…" button first, never the item. What's happening?
3. With fix 2 from §6.3 in place (`currentTarget.contains(target)`), a teammate moves the menu out of
   the portal so it renders inline in the row. Does the row still ignore menu clicks? What would
   you do about it?
4. In §6.4's broken version, the filter *values* survive each fetch but focus doesn't. Explain both
   halves. Then: what would a user lose if `<Filters>` kept a "draft" of the text in its own
   `useState` and only called `onChange` on Enter?
5. A component stores `lastFetchedAt` in a ref and renders `Updated {ago(lastFetchedAt.current)}`.
   It shows the right time on first render and then stops updating. Why, and what should change?
