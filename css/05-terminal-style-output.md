# 5. Terminal-style output — typing, line reveals, and a log that sticks to the bottom

## 5.1 The problem

A log view, a build output, a command-line demo: text that appears over time, line by line,
in a box that keeps the newest line visible. Three behaviours are wanted at once:

1. Text reveals progressively (a character or a line at a time).
2. While content is short, it starts at the **top** of the box, like a terminal.
3. Once it's longer than the box, the box scrolls and the **newest line stays visible at the
   bottom** — but the user can still scroll up to read older lines.

Each one has an obvious CSS answer that turns out to be wrong. This chapter works through
them in order, then covers two traps that show up around them.

## 5.2 Typing: `steps()` and `ch`

A smooth animation interpolates continuously. A typing effect needs discrete jumps — one
whole character per step. **`steps(n)`** is an easing function that divides the duration
into *n* equal jumps ([css-easing-1 §2.3](https://drafts.csswg.org/css-easing-1/#step-easing-functions));
with no second argument it's `steps(n, end)`, which jumps at the *end* of each interval, so
the first frame shows zero steps.

Combine it with the **`ch`** unit, the advance width of the "0" glyph. In a monospace font
every glyph has that advance, so `20ch` is exactly 20 characters wide.

```html
<!doctype html>
<style>
  .type {
    font-family: Menlo, "Courier New", monospace;
    display: inline-block;
    width: 0;
    overflow: hidden;
    white-space: nowrap;
    vertical-align: bottom;
    animation: type 2s steps(20) forwards;
  }
  @keyframes type { to { width: 20ch; } }

  @media (prefers-reduced-motion: reduce) {
    .type { animation: none; width: auto; }
  }
</style>
<span class="type">git log --oneline -5</span>
```

The string is 20 characters, so 20 steps and `20ch`. Change the text and you must change
both numbers — that coupling is the price of the trick. Measured on this
example (pausing the animation and seeking it): 4.00 characters visible at
450ms, 9.99 at 1s, 19.98 at 2.1s. Whole characters, never a half glyph.

Two consequences of how this works:

- The text is all in the DOM from the start. Screen readers read it, find-in-page finds it,
  copy-paste copies it. The effect is purely visual, which is what you want.
- It only works on **one line** and only in a **monospace** font. In a proportional font,
  `ch` is the width of "0", and an "i" and an "m" are not.

## 5.3 Line-by-line reveal, and the shorthand that ate the delays

For multi-line output, reveal whole lines. The direct approach is a per-line animation with a
staggered delay, using a custom property for the index:

```html
<style>
  .log > p {
    margin: 0;
    opacity: 0;
    animation: show 0s forwards;
    animation-delay: calc(var(--i) * 300ms);
  }
  @keyframes show { to { opacity: 1; } }
</style>
<div class="log">
  <p style="--i:0">$ make</p>
  <p style="--i:1">cc -O2 -c main.c</p>
  <p style="--i:2">cc -o app main.o</p>
</div>
```

Measured, seeking the page's animations: at 450ms the first two lines were visible, at 1s
all three.

> ⚠️ **The `animation` shorthand resets `animation-delay`.** Later you add a variant:
>
> ```css
> .log.fast > p { animation: show 0s forwards; }   /* "same thing, just for fast logs" */
> ```
>
> This rule is more specific than `.log > p`, so its shorthand wins — and a shorthand sets
> every longhand it doesn't mention to its initial value (file 01 §1.2). The per-line
> `animation-delay` becomes `0s`. Every line appears at t = 0, and nothing errors. I
> reproduced it: computed delays for three items were `0s, 1s, 2s` with longhands and
> `0s, 0s, 0s` after a more specific shorthand rule. Override **longhands** in variants
> (`animation-duration`, `animation-name`), or repeat the delay expression after the
> shorthand.

## 5.4 Revealing a block: why `clip-path` fails and `max-height` in `lh` works

The alternative to many per-line animations is one animation on the whole block, revealing
it a line at a time. Two candidates:

| | `clip-path` reveal | `max-height` reveal |
|---|---|---|
| Animate | `clip-path: inset(0 0 100% 0)` → `inset(0)` | `max-height: 0` → `max-height: 10lh` with `overflow: hidden` |
| Effect on layout | **None.** Clipping is paint-only; the box keeps its full height | The box's height grows step by step |
| GPU-friendly | Yes | No — it triggers layout each step |

The `clip-path` row looks better until you put the block in a scroll container that keeps the
bottom in view (§5.5). The container sees the *full* height from the first frame, scrolls to
the bottom of it — and the bottom is the part that's still clipped. The user stares at
blank space while the visible lines are revealed above the scroll position, out of sight.

Measured: a 10-line block (200px) in a 100px bottom-pinned box. With `clip-path`, the block
was 200px tall and sat at −100px from the first frame at every sample time — the box was
showing lines 6–10, all still clipped. With `max-height`, the block was 20px at 0.3s, 80px
at 0.9s, 140px at 1.5s (top at −40px, i.e. scrolled so that the newest revealed line sat at
the bottom edge), 200px at the end.

**`lh`** is the computed line height of the element, so `10lh` is ten lines whatever the
font size, and `steps(10)` makes each step one whole line:

```css
.out {
  max-height: 0;
  overflow: hidden;
  flex-shrink: 0;                 /* see the warning */
  animation: reveal 2s steps(10) forwards;
}
@keyframes reveal { to { max-height: 10lh; } }
```

`lh` is Baseline widely available (since May 2026; Chrome 109, Firefox 120, Safari 16.4).

> ⚠️ My first `max-height` test measured a block stuck at 100px, the height of the box. The
> block was a flex item, and `overflow: hidden` had changed its automatic minimum size from
> "my content" to zero (file 01 §1.4), so the flex container shrank it to fit. `flex-shrink:
> 0` restored the expected 140px. If a revealed block refuses to overflow its scroller, check
> this first.

The general principle: **if something must affect scrolling, it must affect layout.** Paint
effects — `clip-path`, `transform`, `opacity` — are invisible to scroll geometry.

## 5.5 A log that sticks to the bottom

Now the scroll container. Wanted: short content at the top; long content scrolls, newest
line at the bottom, older lines reachable by scrolling up.

### The tempting answers, measured

Each variant below is a 100px box, `overflow-y: auto`, 20px lines, tested with 2 lines
(short) and 20 lines (long) in Chrome 154. "Reachable" means the first line could be scrolled
into view.

| Variant | Short content | Long: newest visible? | Long: oldest reachable? |
|---|---|---|---|
| `column`, `justify-content: flex-end` | At the bottom ✗ | ✓ | ✗ — overflow off the top, unscrollable |
| `column`, `justify-content: safe flex-end` | At the bottom ✗ | ✗ — falls back to top | ✓ |
| `column-reverse`, no alignment | At the bottom ✗ | ✓ | ✓ |
| `column-reverse`, `justify-content: flex-end` | At the top ✓ | ✗ — overflow off the bottom, unscrollable | ✓ |
| `column-reverse`, `justify-content: safe flex-end` | At the top ✓ | ✗ — falls back to the top | ✓ |
| **`column-reverse` + spacer** | **At the top ✓** | **✓** | **✓** |

### Why `column-reverse` is the right base

With `flex-direction: column-reverse`, main-start is the **bottom** edge. The scroll origin of
a scroll container is, per CSS Overflow 3, "the block-start inline-start corner of the
scrollable overflow rectangle … (For example, in a flex container it is the main-start
cross-start corner.)" ([css-overflow-3 §2.3](https://drafts.csswg.org/css-overflow-3/#scroll-origin)).
So the scroll origin is the bottom, and **the default scroll position is "at the bottom"**.
Overflow extends upward.

That has a visible consequence in the DOM API. CSSOM View measures `scrollTop` from the
origin with y increasing downwards, and for a box with *upward overflow direction* clamps
it to `min(0, …)` ([cssom-view-1 §6](https://drafts.csswg.org/cssom-view-1/#dom-element-scrolltop)).
So:

```
scrollTop =    0   → at the bottom (the origin)
scrollTop = −300   → scrolled all the way up
```

Measured on the 20-line box: initial `scrollTop` 0 with the last line's bottom at the box's
bottom; minimum `scrollTop` −300, where the first line's top was at the box's top.

> ⚠️ Any script that assumes `scrollTop` is non-negative, or that "at bottom" means
> `scrollTop + clientHeight === scrollHeight`, is wrong for this container. "At the bottom"
> is `scrollTop === 0` (or within a pixel of it).

Because the origin is the bottom, new content added at the bottom doesn't need any script to
stay in view. I appended lines in a loop: `scrollTop` stayed 0 every time. And if the user has
scrolled up to read (I set `scrollTop` to −60, then appended two 20px lines), `scrollTop`
became −100 — the view stayed on the text they were reading rather than jumping. Both are the
behaviour you'd otherwise write code for.

The one problem with plain `column-reverse` is short content: it sits at main-start, the
bottom.

### The spacer

Put an empty flex item **first in the DOM** (so it lays out at the bottom, main-start) and
let it absorb all free space:

```html
<!doctype html>
<style>
  .log {
    height: 12rem;
    overflow-y: auto;
    display: flex;
    flex-direction: column-reverse;
    font: 14px/1.4 Menlo, monospace;
  }
  .log > .spacer { flex: 1 1 0; }
  .log > .lines  { flex-shrink: 0; }
</style>
<div class="log">
  <div class="spacer"></div>
  <div class="lines">
    <div>line 1</div>
    <div>line 2</div>
    <!-- append more lines here -->
  </div>
</div>
```

- **Short content:** there's free space; the spacer grows into it (`flex-grow: 1`), sits below
  the lines, and pushes them to the top.
- **Long content:** there's no free space; the spacer's basis is 0 so it shrinks to nothing,
  the lines overflow upward from the bottom origin, and the newest line is at the bottom.

```
  short content              long content
 ┌──────────────┐          ┌──────────────┐ ▲ older lines
 │ line 1       │          │ line 14      │ │ (scroll up)
 │ line 2       │          │ line 15      │
 │░░ spacer ░░░░│          │ line 16      │
 │░░░░░░░░░░░░░░│          │ line 17      │ ← newest, at origin
 └──────────────┘          └──────────────┘   spacer = 0px
```

Measured: short content had its first line at the top of the box; long content had its last
line at the bottom, `scrollTop` 0, and could be scrolled to −300.

### Why `safe flex-end` didn't do it — and where the spec is going

`justify-content: flex-end` in `column-reverse` pushes content to the top for the short case,
which looks like the answer. With long content, unsafe alignment pushes the overflow out of
the *bottom*, which is behind the scroll origin and so unreachable. `safe` exists to prevent
exactly that data loss, so `safe flex-end` should be it.

It isn't, in Chrome 154: in my test, the long content's first line sat at the top of the box
and the newest lines hung off the bottom, unreachable. It fell back to the **block-start**
side (the top) rather than toward the scroll origin.

This is a case where the specs and browsers have been genuinely unsettled, and the
disagreement is the lesson:

| Source | What `safe` falls back to on overflow |
|---|---|
| css-align-3 Editor's Draft text, checked October 2026 | "as if the alignment mode were `flex-start`" ([§4.4](https://drafts.csswg.org/css-align-3/#overflow-values)) — that would be the bottom here |
| CSSWG resolution, 25 June 2025, [issue #11937](https://github.com/w3c/csswg-drafts/issues/11937) | "Change `safe` alignment on scroll containers to align towards the scroll origin side (rather than the `start` side)" — also the bottom here |
| Chrome 154, measured | The top (block start) |

The issue was raised because browsers didn't match the spec in reversed scroll containers and
an attempt to align with it caused breakage. If browsers implement the 2025 resolution,
`safe flex-end` should give this behaviour without a spacer. I haven't checked Firefox or
Safari. Until it's the same everywhere, the spacer works in every browser that has flexbox.

> **Teacher's aside.** The spacer trick works because it doesn't use *alignment* at all. It
> uses *free-space distribution*, the oldest and best-specified part of flexbox. Alignment
> only matters when there's leftover space, and the spacer makes sure there never is: it eats
> all of it when content is short and is zero when content is long. When a newer feature is
> unsettled, look for a way to express the same thing with an older one.

## 5.6 Trap: `scroll-behavior: smooth` on the log

Smooth scrolling seems like a free improvement for a log that moves. Set it on the container
and see what it actually controls. CSSOM View applies `scroll-behavior` to **programmatic**
scrolls whose behaviour is `auto`: `scrollTo()`, `scrollIntoView()`, setting `scrollTop`, and
so on ([cssom-view-1 §3.1](https://drafts.csswg.org/cssom-view-1/#scrolling)). It
doesn't affect the user's own scrolling.

So it affects every scroll *your code* makes on that box. A common pattern for a growing log
in a normal `column` container is to pin it on every frame:

```js
box.scrollTop = box.scrollHeight;   // run on every new line / every frame
```

With `scroll-behavior: auto` that's an instant jump each time. With `smooth`, each assignment
*starts a new smooth scroll* toward a target that's out of date by the next frame, so the box
chases the content. Measured with a block growing one 20px line every 100ms and the line
above run every frame: with `auto`, 0px from the bottom at 10 of 11 samples; with `smooth`,
behind at several samples by 4, 14 and 20px — a whole line hidden. It caught up only after
growth stopped.

What I did **not** reproduce: the browser's own scroll anchoring being smoothed. In Chrome 154,
with `smooth` set on the container, inserting 100px above the visible content moved
`scrollTop` from 200 to 300 within one frame — the same as with `auto`. The CSS Scroll
Anchoring spec doesn't mention `scroll-behavior` either. So the trap, as far as I could
reproduce it, is about your own scripted scrolls.

The fix:

- Leave the container at `scroll-behavior: auto`.
- With the `column-reverse` design, you usually don't need follow-scrolling at all (§5.5).
- When you do want a glide (say, a "jump to latest" button), make **one** scripted scroll with
  a fixed duration, rather than many smooth ones:

```js
function glideTo(box, target, ms = 300) {
  const start = box.scrollTop, t0 = performance.now();
  const step = (now) => {
    const t = Math.min(1, (now - t0) / ms);
    box.scrollTop = start + (target - start) * (1 - (1 - t) ** 3);  // ease-out
    if (t < 1) requestAnimationFrame(step);
  };
  requestAnimationFrame(step);
}
glideTo(log, 0);   // column-reverse: 0 is the bottom
```

Respect `prefers-reduced-motion` here too: check
`matchMedia('(prefers-reduced-motion: reduce)').matches` and jump instead.

## 5.7 Trap: copying text from a hidden element loses its line breaks

A "copy" button for a log often reads the text from a hidden element — the full output,
kept out of view while the visible one animates. The natural call is `innerText`, because,
unlike `textContent`, it turns `<br>` into newlines.

Except when the element isn't rendered. The HTML spec's `innerText` getter says: "If element
is not being rendered … return element's descendant text content"
([HTML §3.2.7](https://html.spec.whatwg.org/multipage/dom.html#the-innertext-idl-attribute)).
That's `textContent`, and a `<br>` contains no text. And elements with `visibility` other than
`visible` contribute nothing in the rendered-text steps. Measured on `one<br>two`:

| How the source element is hidden | `innerText` |
|---|---|
| Not hidden | `"one\ntwo"` |
| `display: none` | `"onetwo"` — line breaks lost |
| `visibility: hidden` | `""` — empty |
| Visually hidden (`clip-path: inset(50%)`, 1px box) | `"one\ntwo"` |

The robust fix doesn't depend on rendering at all: clone the node, replace each `<br>` with a
newline, then read `textContent`.

```js
function textWithBreaks(el) {
  const copy = el.cloneNode(true);
  copy.querySelectorAll('br').forEach(br => br.replaceWith('\n'));
  return copy.textContent;
}
navigator.clipboard.writeText(textWithBreaks(hiddenSource));
```

Measured: `"one\ntwo"` from a `display: none` source. (If the source uses block elements per
line instead of `<br>`, append `'\n'` to each block in the clone the same way.)

---

## Check yourself

1. A typing animation uses `steps(30)` and `width: 30ch` and looks right. You switch the font
   to the site's sans-serif and it looks wrong. Explain precisely what's wrong.
2. You add `.log.compact > p { animation: show 0s forwards }` and every line now appears at
   once. Walk through the cascade step by step to explain it.
3. A teammate replaces your `max-height` reveal with `clip-path` "for performance", in a
   bottom-pinned log. Describe what the user sees on a 30-line output, and why.
4. In a `column-reverse` log, someone writes `atBottom = box.scrollTop + box.clientHeight >=
   box.scrollHeight`. When is this true, and what's the correct test?
5. Explain why the spacer gets the short case and the long case right, without using any
   alignment property. What would happen if the spacer came *after* the lines in the DOM
   instead of before them?
6. A copy button works in testing and returns one long line in production. The only
   difference is that production hides the source with `display: none` instead of a
   visually-hidden class. Explain using the `innerText` algorithm, and give a fix that
   doesn't care how it's hidden.

(Answers in [`08-exercises.md`](08-exercises.md).)
