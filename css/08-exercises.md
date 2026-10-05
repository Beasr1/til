# 8. Exercises and answers

Try each one before reading the answer. Most can be checked in a browser in under a minute —
that's better than reading the answer.

---

## File 01 — Foundations

**1. `.panel` vs `#main .panel`. Which wins; does order matter?**

`#main .panel` is `(1,1,0)`; `.panel` is `(0,1,0)`. Column A decides: 1 > 0, so `#main .panel`
wins, opacity 1. Order of appearance is only consulted when specificity ties, so it doesn't
matter which comes later. (Both are normal author declarations in no layer, so steps 1–4 of
the cascade tie and specificity is the first step that separates them.)

**2. A finished `forwards` animation holds `opacity: 0` against an ID selector. Why, and two ways out?**

A running or filling animation's values belong to the **animation origin**, which outranks
every normal author declaration, however specific. Specificity is only compared *within* an
origin, so the ID never gets a say. Ways out: (a) stop the animation applying —
`.visible { animation: none; opacity: 1 }` — so there's no animation-origin value; or
(b) `opacity: 1 !important`, because important author declarations outrank the animation
origin. (a) is the clean one; (b) works but makes later overrides harder. A third option is
to change the animation so its end state is the one you want and drop `forwards`.

**3. `overflow: hidden` on a flex item makes it shorter than its content. Mechanism?**

A flex item's default `min-height: auto` resolves to its content-based minimum size — *unless*
the item is a scroll container, when it's zero (css-flexbox-1 §4.5). `overflow: hidden` makes it
a scroll container. With the floor gone, the default `flex-shrink: 1` lets the container shrink
it to fit. Fix with `flex-shrink: 0`, or an explicit `min-height`.

**4. A class change animates; inserting a new element doesn't. Why?**

A transition interpolates from the **before-change style** to the after-change style. When a
class is added, the element was styled a moment ago with opacity 0 — that's the before-change
style, and the change to 1 animates. A new element has never been styled, so there's no
before-change style; its first computed style simply *is* opacity 1. `@starting-style` (file
04) supplies a style to start from in exactly that case.

**5. `button:focus-visible, button:-moz-focusring { … }` never applies in Chrome. Why; how to fix?**

Chrome doesn't recognise `:-moz-focusring`, so that selector is invalid, and one invalid
selector in a plain selector list invalidates the **whole rule**. I checked: the stylesheet had
zero rules after parsing, and the button showed only the browser's default ring. Fix by
splitting into two rules, or use the forgiving list `button:is(:focus-visible,
:-moz-focusring)`, which skips the entry it doesn't understand — that version applied the
outline in Chrome.

---

## File 02 — State without JavaScript

**1. The checkbox moves inside its `<label>`. What breaks, and why silently?**

`#t:checked ~ .panel` needs `.panel` to be a later *sibling* of the checkbox. Inside the label,
the checkbox's siblings are the label's other children; the panel is now a sibling of the
*label*. The selector matches nothing. Clicking still toggles the checkbox (a label wrapping an
input works without `for`), so the state changes — it just isn't seen. A selector that matches
nothing is valid CSS, so nothing reports an error. `.card:has(#t:checked) .panel` would survive
the move.

**2. `display: none` checkbox: who is locked out, and how?**

A keyboard user. An element with `display: none` isn't rendered, so it isn't focusable: Tab
goes straight past it (I measured Tab landing on the next control). The label isn't focusable
either, so there's no keyboard route to the state. Screen-reader users lose it too, as the
input is out of the accessibility tree. Mouse testing passes because clicking the label still
toggles the hidden input.

**3. Why `:focus-visible` rather than `:focus`?**

`:focus` matches whenever the element has focus, including after a mouse click on its label —
measured: clicking the label focused the checkbox. With `:focus + label`, every mouse user
gets an outline after clicking, which designers then remove globally, which breaks it for
keyboard users. `:focus-visible` matches only when the browser judges a focus indicator is
useful (keyboard focus), so you can keep it visible without the click-outline complaint.

**4. Specificity of `.card:has(#t:checked) .panel`, and the bite?**

`.card` `(0,1,0)` + `:has(#t:checked)` takes its argument's `(1,1,0)` + `.panel` `(0,1,0)` =
**`(1,3,0)`**. Any later plain rule — `.panel.is-hidden { display: none }` at `(0,2,0)` —
loses to it, so you can't hide the panel without matching an ID. Use a class in the `:has()`
argument (`.card:has(.toggle:checked)`), or wrap it in `:where()` to drop its weight.

**5. Exclusive FAQ: radios vs `<details name>`?**

Radios: one `name` gives mutual exclusion, and `:checked ~` or `:has()` shows the matching
panel. But a radio group can't return to "none selected" by clicking — once one answer is
open, one stays open unless you add a "close" radio — arrow keys move *and select* within the
group, and each is announced as a radio button. `<details name="faq">`: exclusive by spec,
each can be closed, disclosure semantics and keyboard handling for free, find-in-page opens
it. Choose `<details name>`. You give up layout freedom (the summary is the toggle and the
answer must be inside the element), and in a browser older than the 2024 Baseline, entries
just won't close each other — a graceful failure.

**6. One reason to prefer a `<button>` and a few lines of script.**

Correct semantics: a `<button aria-expanded>` is announced as "button, collapsed/expanded",
which is what a disclosure is; a checkbox is announced as a checkbox. Other good answers: the
state must be shared with something outside the DOM subtree (a URL, storage, another
component), or the control and the panel are too far apart in the tree for any selector.

---

## File 03 — Stacking and sizing

**1. `position: absolute` vs grid stack, when the detail is taller.**

The absolutely positioned detail is out of flow, so its parent is sized only by the summary.
The detail spills past the parent's bottom edge and is painted **over** the content below,
which doesn't move because, as far as layout knows, nothing grew. In the grid stack both
children are in flow in one cell, the cell sizes to the taller one, and the following content
is placed after the cell. No overlap, and no jump as long as the cell was already that size.

**2. A log in a stack grows the cell. Fix, and the two mechanisms?**

`contain: size` on the log (with `overflow: auto` so its content scrolls). Mechanism one:
size containment makes the log's intrinsic size that of an empty box, so the grid track sizes
to the *other* children. Mechanism two: grid items stretch by default, so the log is then made
exactly the cell's size. Containment says "don't ask me", stretch says "take this".

**3. `visibility: hidden` reserves space; `display: none` doesn't. In terms of boxes?**

`visibility: hidden` still generates a box that takes part in layout with its full size; only
painting (and hit-testing) is suppressed. `display: none` generates no box at all, so there's
nothing for the grid track to measure.

**4. `container-type: inline-size` on an inline-block shrinks it to nothing.**

The declaration applies inline-size containment: the element's inline size is computed as if it
had no content. An inline-block is shrink-to-fit — its width comes *from* its content. With
content excluded from the calculation, the width is zero (plus padding and border). MDN's
warning: containers need their size "set by context … or explicitly defined", otherwise they
collapse. Give it a width or make it a block that stretches.

**5. `container-type` and `font-size: 4cqi` on the same element sizes against the page.**

An element can't query itself: `cq*` units resolve against the nearest **ancestor** query
container. If there's none, they fall back to the small viewport size — hence "the page". I
measured an element that was itself a container resolving `10cqi` against its ancestor
(40px from a 400px ancestor, not 20px from its own 200px). Put `container-type` on a wrapper.

**6. 0.6 wraps by one character; 0.62 never does. Two causes the margin covers.**

(a) The real advance is larger than 0.6em: Menlo measured 0.602. Over 40 characters that's
0.08em too little — about 1.3px at 16px — so a box sized exactly by the formula is short by a
pixel. (b) Rounding: font sizes and glyph positions snap to fractional or device pixels, and
the line can come out a fraction wider than the arithmetic. Other valid answers: a vertical
scrollbar appearing inside the box takes width the formula didn't subtract; a fallback font
with a different advance is used for some characters.

---

## File 04 — Enter and exit

**1. No animation in either direction. Name each mechanism.**

Enter: the element was `display: none`, so at the previous style computation it wasn't
rendered and has no before-change style. Nothing to start from, no transition. Exit: `display`
is discrete and not in the transition list (and `allow-discrete` isn't set), so it changes to
`none` immediately. The box disappears in the same frame the opacity transition would have
started.

**2. `@starting-style` at the top of the file; enter never runs.**

`@starting-style` adds no specificity. Its `.panel { opacity: 0 }` ties with the base rule's
`.panel { opacity: 1 }` on specificity, and the base rule comes later, so it wins *even within
the starting style*. Starting style = after-change style = opacity 1: no difference, no
transition. Move the block after the base rule or nest it inside it. Measured: 0.07 at 100ms
when after, 1.00 when before.

**3. Double-clicking a button inside a panel that's fading out.**

Nothing happens to the button. CSS Display 4 makes the element **inert** while its display
would compute to `none` ignoring transition and animation origins — i.e. for the whole exit.
Inert content isn't hit-tested or focusable; I measured `elementFromPoint` over the button not
returning it, and `focus()` having no effect. The clicks go to whatever is beneath the panel in
hit-testing terms, which is worth knowing if something clickable is underneath.

**4. Why can a delayed exit animation play on first load but never on reload?**

A transition fires whenever an element's computed value changes between two style
computations, whatever the cause. On a first visit, if anything makes the browser style the
element before the stylesheet arrives (async CSS, a late `<link>`, a script reading layout),
the first style uses browser defaults, e.g. opacity 1. When the sheet arrives, opacity becomes
0: a change, so the transition runs, delay and all. On reload, the stylesheet comes from cache
and is there for the first style computation. The first value is already 0, nothing changes,
nothing animates. Measured: transition fired on cold load with the async pattern, not with a
warm cache.

**5. Safari: "the accordion doesn't animate". Bug?**

Not as such. As of October 2026 Safari doesn't support `interpolate-size`; the declaration is
ignored and the section snaps open and closed. The end state is correct, which is the bar
progressive enhancement sets. It *would* be a bug if anything depended on the intermediate
frames or on a `transitionend` event that never fires there — e.g. content only made visible
in a `transitionend` handler, which would never run.

**6. `.panel.compact { transition: opacity 0.1s }` stops the fade-out.**

The `transition` shorthand resets every longhand it omits: `transition-property` becomes just
`opacity` (`display` is dropped from the list) and `transition-behavior` goes back to `normal`
(measured). On exit, `display` is no longer transitioned, flips to `none` at once, and the box
is gone before the 0.1s fade can show. Override `transition-duration` only, or repeat the full
list including `display 0.1s allow-discrete`.

---

## File 05 — Terminal-style output

**1. `steps(30)` + `30ch` breaks in a sans-serif font.**

`ch` is the advance width of "0" in that font. In a proportional font, glyph widths vary, so
`30ch` isn't 30 characters — narrow letters mean more than 30 fit, wide ones fewer. Each step
reveals a fixed `1ch` slice, which no longer lines up with glyph boundaries, so steps show
half-letters, and the final width either clips the end of the string or leaves a gap.

**2. `.log.compact > p { animation: show 0s forwards }` makes every line appear at once.**

Both rules match each line. For the property `animation-delay`, the candidates are the
longhand in `.log > p` `(0,1,1)` and the shorthand's implied value in `.log.compact > p`
`(0,2,1)`. The shorthand sets `animation-delay` to its initial value, `0s`, because the delay
was omitted. Same origin, no layers, so specificity decides: `(0,2,1)` wins. Every line gets
delay 0s, and with a 0s duration they all show at t = 0. Measured: delays `0s, 0s, 0s` instead
of `0s, 1s, 2s`.

**3. `clip-path` reveal in a bottom-pinned log, 30 lines.**

`clip-path` only affects painting, so the block takes its full 30-line height from the first
frame. The scroll container pins its bottom: the user is looking at lines ~25–30, which are
still clipped. The lines being revealed — 1, 2, 3… — are above the visible area. The user sees
an empty box until the reveal reaches the bottom lines, then everything at once. With
`max-height` the block's height grows, so the newest revealed line is the one at the bottom
edge.

**4. `scrollTop + clientHeight >= scrollHeight` in a `column-reverse` log.**

In a column-reverse scroll container, `scrollTop` is 0 at the bottom and negative going up. At
the bottom, the expression is `0 + clientHeight >= scrollHeight`, which is only true when the
content doesn't overflow. Once the log has more lines than fit, it's false even when the user
*is* at the bottom, and false further up too. It's really testing "is there no overflow". The
correct test is `Math.abs(box.scrollTop) < 1`.

**5. Why the spacer works without alignment; spacer after the lines?**

It uses free-space distribution. Short content: there's free space, and the spacer is the only
item with `flex-grow`, so it takes all of it. Being first in the DOM, it sits at main-start —
the bottom — and the lines sit above it, at the top. Long content: no free space; the spacer's
basis is 0 and the lines are `flex-shrink: 0`, so the spacer is 0px and the lines overflow
upward from the bottom origin, newest at the bottom. If the spacer came **after** the lines, it
would sit at main-end — the top — and push the lines to the bottom in the short case: back to
plain `column-reverse` behaviour. Long content would look the same, since the spacer is 0px.

**6. Copy button returns one long line once the source is `display: none`.**

`innerText` returns rendered text, turning `<br>` into newlines — but the HTML spec's first step
is "if element is not being rendered … return element's descendant text content". A
`display: none` element isn't rendered, so you get `textContent`, and `<br>` has no text: the
lines run together (measured `"onetwo"`). The visually-hidden version is rendered, so it worked
(`"one\ntwo"`). The robust fix: clone the node, replace each `<br>` with a `"\n"` text node, read
`textContent`. It doesn't depend on rendering at all.

---

## File 06 — Testing in a real browser

**1. Why is "screenshot 400ms after the click" flaky for a 1s transition?**

The 400ms is measured by the test's clock, not the animation's. Event dispatch, frame
scheduling and the screenshot itself — about 60ms of animation time in my measurement — all
vary, so each run captures a slightly different frame and the pixels differ from the reference.
Pause and seek to an exact time, or assert on measured values rather than images.

**2. Why `Range.getClientRects()` instead of height ÷ line height?**

Because the element's height may not depend on its line count. In this course's layouts it often
doesn't: a grid-stretched or `contain: size` box has the height of its cell, and a `max-height`
box is clipped. The text can wrap with no change in height. Padding and `line-height` also make
the division fragile. `Range.getClientRects()` returns the line fragments themselves, so the
number of distinct tops *is* the number of rendered lines.

**3. Why can polling `document.getAnimations()` corrupt a first-load investigation?**

The spec says calling it "triggers a style change event for the target element". Polling forces
style computations at moments the browser wouldn't otherwise choose — including, possibly,
before the stylesheet has arrived. That can create the very transition you're investigating, or
shift timing so it disappears. Observe passively instead with the CDP Animation domain's
`animationStarted` event, which doesn't run anything in the page.

**4. A cold-load test passes but the bug reaches users. Two ways it was secretly warm.**

Any two of: the HTTP cache wasn't disabled, or an earlier test in the same browser context
already cached the stylesheet; the local server returns the CSS so quickly that it arrives before
the first style computation (users on real networks are slower — add a deliberate delay).
Two more worth ruling out, though I haven't tested them: loading the fixture from
`file://`, whose load timing is nothing like a network; and a `<link rel=preload>` or service
worker in the test environment that production doesn't have.

**5. Design a test for the bottom-sticking log.**

Set up the `column-reverse` + spacer log in a fixture.

- **Short content:** with two lines, assert the first line's `top` equals the box's `top`
  (allowing for padding).
- **Pinned while growing:** append a line every frame or two; after each, assert
  `Math.abs(box.scrollTop) < 1` and that the last line's `bottom` equals the box's `bottom`, to the
  pixel.
- **Reader not yanked:** set `scrollTop = -60`. Record the `top` of a specific line that's visible.
  Append several lines. Assert that line's `top` is unchanged, and `scrollTop` decreased by the
  appended height (measured: −60 → −100 after 40px).

Also assert the oldest line is reachable: set `scrollTop` to a large negative value and check the
first line's `top` equals the box's `top`.

---

## Things to try

1. **Break the cascade on purpose.** Write an animation with `forwards` and try to override its end
   value with increasingly specific selectors. Then try `!important`. Then try a transition. Check
   each against the origin list in file 01 §1.2.
2. **Rebuild the measured tables.** Take the six log variants in file 05 §5.5 and reproduce the
   table with the harness in file 06 — in Chrome, then in Firefox and Safari if you can drive
   them. The `safe` rows are the ones most likely to differ.
3. **Find your font's advance.** Measure `ch` against `em` for the monospace font you actually use,
   and recompute the fitting formula in file 03 §3.4 for 80 characters.
4. **Animate a `<details>` both ways** with the file 04 §4.5 recipe, then remove one piece at a time
   (`interpolate-size`, `content-visibility … allow-discrete`, the `[open]` rule) and predict what
   breaks before you look.
5. **Reproduce the cold-load bug** with the file 06 §6.5 harness, then fix it with the `.ready` gate
   and confirm the harness reports a clean load.

---

## Questions worth asking me

- "Walk me through the cascade for this exact element — here's the DevTools styles pane."
- "Why is `:has()` slow in this case and fast in that one?"
- "Show me the same component with `<details>`, the checkbox pattern, and a `<button>`, and what a
  screen reader announces for each."
- "What does `contain: layout` / `contain: paint` add on top of `size`, and when do I want them?"
- "How do container *style* queries differ from size queries?"
- "What is `calc-size()` and when would I write it by hand instead of using `interpolate-size`?"
- "Has any browser shipped the 2025 `safe` alignment resolution yet?" (Check the current status;
  file 05 §5.5 reflects October 2026.)
- "How does the top layer change enter/exit animation for `<dialog>` and popovers?"
- "Is my mental model right? Here's what I think happens: …"
