# 3. Stacking and sizing — two things in one place, text that always fits

## 3.1 The problem: overlap without collapse

You want two views of the same thing in the same spot — a compact summary and an expanded
panel that appears over it, or two states that crossfade. Whichever is visible, the
surrounding layout shouldn't jump.

The reflex is `position: absolute` on one of them. As file 01 §1.4 warned, an absolutely
positioned box leaves the flow: its parent stops sizing to it. If the overlay is taller than
the thing underneath, it spills out over whatever comes next on the page, and you start
hard-coding heights. The rest of this chapter is about letting the layout engine do that
sizing for you.

## 3.2 The grid stack

Make the parent a one-cell grid and put **every** child in that same cell:

```css
.stack            { display: grid; }
.stack > *        { grid-area: 1 / 1; }   /* row 1, column 1 — all of them */
```

`grid-area: 1 / 1` is shorthand for row-start 1, column-start 1; the ends default to
spanning one track. Grid explicitly allows several items in one cell — they simply overlap,
painted in DOM order (later on top, adjustable with `z-index`).

Now the important property: **the cell is sized by the largest child**, because a grid
track sizes to fit its contents. And grid items stretch to fill their cell by default
(`align-self: normal` "sizes as either stretch (typical non-replaced elements) or start
(typical replaced elements)", [css-align-3 §6.2.5](https://drafts.csswg.org/css-align-3/#align-self-property);
an `<img>` is replaced, a `<div>` isn't). So every
child ends up the size of the largest one.

```
 DOM                         rendered (one grid cell)
 ┌──────────────┐            ┌──────────────────────────┐
 │ .summary     │  ──┐       │ .summary   (stretched)   │
 │  (2 lines)   │    ├────►  │ .expanded  (on top)      │ ← cell height =
 ├──────────────┤    │       │                          │   tallest child
 │ .expanded    │  ──┘       │                          │
 │  (9 lines)   │            └──────────────────────────┘
 └──────────────┘
```

I measured it in Chrome 154: a 300px-wide stack with a one-word child and a long paragraph
child produced a 180px cell, and **both** children reported a height of 180px at the same
`top`. Neither child is out of flow, so the stack's parent sizes to the cell like any normal
block.

A runnable crossfade:

```html
<!doctype html>
<style>
  .stack       { display: grid; width: 20rem; border: 1px solid; }
  .stack > *   { grid-area: 1 / 1; padding: 1rem; transition: opacity 0.3s; }
  .expanded    { opacity: 0; background: Canvas; }
  .stack:hover .expanded,
  .stack:focus-within .expanded { opacity: 1; }
</style>
<div class="stack" tabindex="0">
  <div class="summary">3 warnings</div>
  <div class="expanded">disk 91% full<br>backup 3 hours late<br>2 retries on job 7</div>
</div>
<p>This paragraph never moves.</p>
```

Hover it. Nothing below jumps, because the cell was always as tall as the expanded view.

## 3.3 When the largest child shouldn't win: `contain: size`

Sometimes the opposite is wanted: a log or a long panel that should fill *whatever space
its sibling defines* and scroll internally, rather than making the cell grow to fit all of
its content.

**Size containment** does exactly this. With `contain: size`, "the intrinsic sizes of the
size containment box are determined as if the element had no content"
([css-contain-2 §3.1](https://drafts.csswg.org/css-contain-2/#size-containment)). The spec
calls out the grid case directly: it affects "sizing grid tracks into which a size contained
item is placed".

So in a grid stack:

1. The contained child reports a size of zero (plus its own padding and border).
2. The cell sizes to the other children.
3. Grid's default stretch then makes the contained child fill that cell.
4. Its real content is laid out inside, and overflows — so give it `overflow: auto`.

```css
.log { contain: size; overflow: auto; }
```

Measured, same test page: a stack whose first child was three lines (60px) and whose second
child was a long paragraph with `contain: size; overflow: auto` produced a **60px** cell.
Without containment, the same paragraph made it 180px. The contained child was 60px tall and
scrolled.

> **Teacher's aside.** It's easy to read `contain: size` as "this element has a fixed size".
> It doesn't fix anything. It changes what the element **reports** to its parent during
> sizing — "count me as empty" — and leaves the element free to be stretched to whatever the
> parent then decides. That's why it composes so well with grid stretch: one property says
> "don't ask me", the other says "then take what you're given".

### The sizer: reserving space from content you don't show

The mirror-image problem: the visible content is short, but you want the box sized as if it
held something longer — say, the longest message a status line can ever show, so it doesn't
reflow as messages change. Put that longest content in the stack as an invisible child:

```html
<div class="stack">
  <div class="sizer" aria-hidden="true">Reconnecting to upstream (attempt 10 of 10)…</div>
  <div class="live">Connected</div>
</div>
```

```css
.sizer { visibility: hidden; }
```

`visibility: hidden` keeps the box in layout — it takes its full size — but paints nothing
and receives no pointer events. (Contrast `display: none`, which removes the box entirely and
so reserves nothing.) Measured: a five-line hidden sizer made a 100px cell for a one-line
visible child. `visibility: hidden` content is also left out of the accessibility tree, and
`aria-hidden="true"` says so explicitly for anyone reading the markup.

| Tool | Child's contribution to cell size | Child's own size | Use for |
|---|---|---|---|
| Plain grid stack | Its content size | Stretched to the largest | Crossfades, overlays without layout jump |
| `contain: size` | Zero | Stretched to the others; scrolls its content | A panel that fills space it didn't define |
| `visibility: hidden` sizer | Its content size | Invisible | A minimum size taken from content |

## 3.4 Text that always fits: container queries

Different problem, same theme — sizing from context. You have a box of monospace text (a
log line, a table of hashes) that must show **N characters per line without wrapping**,
and the box's width depends on where it's placed.

### Why viewport breakpoints leave gaps

The classic tool is a media query on the viewport:

```css
.box { font-size: 1rem; }
@media (max-width: 600px) { .box { font-size: 0.7rem; } }
@media (max-width: 400px) { .box { font-size: 0.55rem; } }
```

This answers "how wide is the window?" when the real question is "how wide is the box?". And
it's a step function trying to approximate a continuous one. Between steps, the font is too
big for some widths.

I measured it: a 60-character Menlo line in a padded box (max 720px, 16px page gutters),
viewport swept from 300px to 1000px in 2px steps. With the three breakpoints above, the line
**wrapped at 300–382px, 402–470px and 602–642px.** Each band sits just above a breakpoint,
where the font size has jumped back up but the box hasn't grown enough to hold it. You can
keep adding breakpoints; you're tuning a staircase against a slope.

### Container queries and `cqi`

**Container queries** let an element respond to the size of an *ancestor box* instead of
the viewport. You opt an ancestor in:

```css
.wrap { container-type: inline-size; }
```

That makes `.wrap` a **query container** for its inline size (width, in horizontal text).
Descendants can then use `@container (width > 30rem) { … }` rules, and — more useful here —
**container query units**:

| Unit | Meaning ([css-conditional-5 §7](https://drafts.csswg.org/css-conditional-5/#container-lengths)) |
|---|---|
| `cqi` | 1% of the query container's inline size |
| `cqb` | 1% of its block size (needs `container-type: size`) |
| `cqw` / `cqh` | 1% of its width / height |
| `cqmin` / `cqmax` | The smaller / larger of `cqi` and `cqb` |

If no eligible container exists, the units fall back to the **small viewport** size, so
they degrade to roughly `svw`.

Two things the declaration does that you might not expect:

- **It applies inline-size containment** to the container (plus style containment). Its
  width can no longer come from its content. In a context that gives it a width (a normal
  block, which stretches) that's invisible. In a shrink-to-fit context (a float, an inline
  block, a flex item with no basis), MDN warns it **collapses**.
- **An element can't query itself.** `cqi` resolves against the nearest *ancestor*
  container. I tested `font-size: 10cqi` on a 200px-wide element that was itself a container
  inside a 400px container: it resolved to 40px — 10% of the ancestor, not of itself. So the
  container and the text that uses `cqi` must be different elements.

### The fitting formula

For a monospace font, each character advances the same distance — a fixed fraction of the
font size. Call it *a* (in em). N characters need `N × a × font-size` of width. Set that
equal to the available width and solve:

```
font-size = available / (N × a)
available = 100cqi − horizontal padding − border
```

```css
.wrap { container-type: inline-size; }
.box  {
  font-family: Menlo, "Courier New", monospace;
  padding: 1rem;
  border: 1px solid;
  font-size: min(1rem, calc((100cqi - 2rem - 2px) / (0.61 * 60)));
}
```

`min(1rem, …)` caps it so wide boxes don't get comically large text. Same sweep as above: 
**no wrapping at any width from 300 to 1000px.**

Where does 0.61 come from? I measured the advance width per em in Chrome on macOS:

| Font | Advance (em) |
|---|---|
| Menlo | 0.602 |
| Courier New | 0.600 |
| Monaco | 0.600 |

0.6 is a common value but not universal — other monospace fonts differ, and the generic
`monospace` maps to a different font per platform. Use a value slightly above your font's
real advance (0.61 here) so rounding never pushes the last character over, and measure
your actual font if it isn't one of these. (The `ch` unit *is* the advance of "0", but you
can't use it here: `ch` inside `font-size` refers to the parent's font, and the formula is
solving for the font size itself.)

> ⚠️ The breakpoint version had passed review at the common widths people test — 375, 768,
> 1280. The failures were in bands nobody had a device for. If something must hold at
> *every* width, sweep every width (file 06 §6.3 shows how), and prefer a formula over a
> table of cases.

Support: container size queries and the `cq*` units are Baseline **widely available**
(newly available February 2023: Chrome 105, Firefox 110, Safari 16), per web-features data
checked October 2026.

---

## Check yourself

1. You stack a summary and a detail view with `position: absolute` on the detail. What does
   the page below the component do when the detail is taller, and why doesn't the grid
   stack have this problem?
2. In a grid stack, a scrolling log grows the cell to the full height of every log line
   instead of matching its sibling. Which one property fixes it, and what are the two
   mechanisms that together make the log the right size?
3. Why does a `visibility: hidden` sizer reserve space while a `display: none` one doesn't?
   Answer in terms of boxes.
4. You put `container-type: inline-size` on a `display: inline-block` badge and it shrinks to
   nothing. Explain.
5. You write `.box { container-type: inline-size; font-size: 4cqi }` and the text sizes
   against the page, not the box. Why?
6. Your fitting formula uses 0.6 and a 40-character line wraps by one character at some
   widths, but never with 0.62. Give two distinct causes the margin is covering.

(Answers in [`08-exercises.md`](08-exercises.md).)
