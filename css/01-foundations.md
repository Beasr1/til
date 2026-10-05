# 1. Foundations — how CSS decides anything

**Who can skip this:** if you can explain why a rule with an ID loses to an animation, what
`min-height: auto` does to a flex item, and why `display` can't normally be transitioned,
go straight to file 02. Everyone else: every later chapter assumes the vocabulary here.

## 1.1 The problem: CSS isn't a script

A backend engineer usually meets CSS as a pile of rules that mostly work, and then one
that doesn't apply for no visible reason. The reason is almost always that CSS isn't
executed top to bottom like code. It's a **constraint system**: the browser collects every
declaration that could apply to an element, picks a winner per property using a fixed set
of rules, and then runs a layout algorithm over the results. You don't tell it what to do;
you tell it what's true, and it resolves the conflicts.

Three questions come up with every rule you write. The rest of this chapter answers each
one in turn:

| Question | Answered by | Section |
|---|---|---|
| Which value wins for this property? | The cascade | §1.2 |
| How big is this box? | The box model and intrinsic sizing | §1.3 |
| Where does this box go? | The layout mode of its parent | §1.4 |

Then two things this course is really about — change over time (§1.5, §1.6) — and how to
use new features without breaking old browsers (§1.7).

## 1.2 The cascade — which declaration wins

Several declarations can target the same property of the same element. The **cascade** is
the algorithm that picks one. CSS Cascade 5 §6.1 sorts them by these criteria, and only
falls through to the next row when the one above is a tie
([css-cascade-5](https://drafts.csswg.org/css-cascade-5/#cascade-sort)):

| Step | Criterion | What it means in practice |
|---|---|---|
| 1 | **Origin and importance** | Where it came from (browser default, user, your stylesheet) and whether it's `!important` |
| 2 | Context | Shadow DOM encapsulation. Ignore it unless you use web components |
| 3 | Element-attached styles | A `style=""` attribute beats any selector of the same origin and importance |
| 4 | **Cascade layers** | `@layer` order. Later layers win (for normal declarations) |
| 5 | **Specificity** | How precisely the selector targets the element |
| 6 | **Order of appearance** | Last one wins |

Most of your day-to-day battles are steps 5 and 6. But step 1 holds a surprise that matters
for this whole course. The origins, in descending precedence, are:

```
Transition declarations            ← a running transition beats everything
Important user-agent declarations
Important user declarations
Important author declarations      ← your !important
Animation declarations             ← a running @keyframes animation
Normal author declarations         ← your ordinary rules
Normal user declarations
Normal user-agent declarations     ← browser defaults
```

That ordering is quoted from the spec's sort order. Read the consequence slowly: **while a
`@keyframes` animation is running, it overrides every normal rule you've written, whatever
its specificity.** Only `!important` beats it. And a running transition beats even that.
When you see a property "stuck" on a value you didn't expect, ask whether an animation with
`fill-mode: forwards` is still holding it.

### Specificity is a tuple, not a number

**Specificity** is counted per selector as three columns, `(A, B, C)`
([selectors-4 §15](https://drafts.csswg.org/selectors-4/#specificity-rules)):

| Column | Counts | Example |
|---|---|---|
| A | ID selectors | `#log` |
| B | classes, attribute selectors, pseudo-classes | `.open`, `[hidden]`, `:checked` |
| C | type selectors and pseudo-elements | `div`, `::before` |

Compare left to right; the first column that differs decides. `#a` `(1,0,0)` beats
`.x.y.z.w.v.u.t.s.r.q.p` `(0,11,0)`, because 1 > 0 in column A and nothing else is looked at.

Three pseudo-classes take arguments and need a special rule. The specificity of `:is()`,
`:not()` or `:has()` is that of **the most specific selector in its argument**; the
specificity of `:where()` is **always zero**. That makes `:where()` the tool for writing
defaults that are easy to override.

> **Teacher's aside.** People coming from programming expect "later in the file wins". It
> does — but only after specificity has tied. The common bug in this course is a more
> *specific* rule further *up* the file silently overriding a less specific one below it.
> When a rule seems ignored, open DevTools, look at the element's styles pane, and find the
> struck-through declaration. The winner is listed above it, with its selector. You'll
> never have to guess.

### Shorthands set every longhand

`margin`, `transition`, `animation`, `font` and friends are **shorthands**. When you write a
shorthand, every sub-property you leave out is set to its initial value, not left alone —
the spec says "each 'missing' sub-property is assigned its initial value"
([css-cascade-5 §3](https://drafts.csswg.org/css-cascade-5/#shorthand)). So:

```css
.item      { animation-delay: 2s; }
.log .item { animation: fade-in 0.2s; }   /* animation-delay is now 0s */
```

The second rule is more specific, and its shorthand quietly resets the delay. That exact
bug gets a full treatment in file 05 §5.3. Note it here as a rule of the language.

### Inheritance

Some properties **inherit** — a child takes its parent's computed value unless something
sets it. Text properties (`color`, `font-*`, `line-height`) inherit; box properties
(`width`, `margin`, `display`) don't. The spec's property table says which, under
"Inherited". It matters later because `interpolate-size` (file 04) inherits, so setting it
once on `:root` turns it on everywhere.

## 1.3 The box model — how big is a box

Every element generates a box with four nested areas:

```
┌──────────────── margin ────────────────┐
│ ┌────────────── border ──────────────┐ │
│ │ ┌──────────── padding ───────────┐ │ │
│ │ │                                │ │ │
│ │ │            content             │ │ │
│ │ │                                │ │ │
│ │ └────────────────────────────────┘ │ │
│ └────────────────────────────────────┘ │
└────────────────────────────────────────┘
```

`width` and `height` size the **content** box by default. Almost everyone sets
`box-sizing: border-box` globally so that `width` includes padding and border, which is the
mental model people actually have.

The more important idea for this course is **intrinsic size** — the size a box wants to be,
given its content:

| Term | Meaning |
|---|---|
| **min-content** | The narrowest the box can get without overflowing: every line break taken |
| **max-content** | The width if nothing ever wraps |
| **auto** height | In normal flow, "as tall as my content" |

Layout is a negotiation between what the parent offers and what the child's intrinsic size
asks for. Several tricks later in the course work by changing what a box *reports* as its
intrinsic size without changing what it *shows* (file 03 §3.3).

## 1.4 Layout modes — who decides where a box goes

The parent's `display` value picks the algorithm that positions its children. They are
genuinely different algorithms, not variations on one:

| Mode | Turned on by | One-line model | Typical use |
|---|---|---|---|
| **Flow** | default (`display: block` / `inline`) | Blocks stack top to bottom, inline content wraps into lines | Prose, documents |
| **Flex** | `display: flex` | Items laid out along one axis; free space shared by `flex-grow` / `flex-shrink` | Toolbars, rows of controls, a column that must fill a height |
| **Grid** | `display: grid` | Items placed into a 2D grid of rows and columns that size from their content | Page layout, cards, *overlapping things* |
| Positioned | `position: absolute` / `fixed` | Box taken **out of** flow and placed relative to a containing block | Overlays, badges |

Two flex facts you need for later chapters:

- **The main axis can run backwards.** `flex-direction: column-reverse` makes the first
  item sit at the *bottom*. The start of the axis ("main-start") is now the bottom edge.
  File 05 builds a self-scrolling log on this.
- **Flex items have a minimum size of `auto` by default**, which means "don't shrink below
  my content" — *unless* the item is a scroll container (any `overflow` other than
  `visible` or `clip`), in which case the minimum becomes 0
  ([css-flexbox-1 §4.5](https://drafts.csswg.org/css-flexbox-1/#min-size-auto)). So adding
  `overflow: hidden` to a flex item can suddenly let it be squashed. I hit exactly this
  while testing file 05: a box animated to `max-height: 200px` measured 100px because its
  flex parent shrank it. `flex-shrink: 0` fixed it.

> ⚠️ `position: absolute` is the reflex for "put this on top of that", and it's usually the
> wrong one for UI that changes size. An absolutely positioned box no longer contributes to
> its parent's size, so the parent can collapse under it. File 03 shows the grid
> alternative.

## 1.5 Transitions vs animations

Both change a property's value over time. They are triggered differently, and that
difference decides which one you reach for:

| | **Transition** | **Animation** (`@keyframes`) |
|---|---|---|
| Triggered by | A computed value *changing* (a class added, `:checked`, `:hover`) | The `animation-name` property applying to the element |
| Start value | Whatever the value was before the change | The first keyframe |
| End value | The new value | The last keyframe |
| Repeats | No — one A→B per change | `animation-iteration-count`, can be `infinite` |
| Runs on page load? | No (see below) | Yes, as soon as it applies |
| Cascade origin | Transitions — beats everything | Animations — beats normal rules |
| Good for | State changes: open/closed, shown/hidden | Sequences: typing, staggered reveals, loops |

```css
/* transition: reacts to a state change */
.panel        { opacity: 0; transition: opacity 0.3s ease; }
.panel.open   { opacity: 1; }

/* animation: plays a script as soon as it applies */
@keyframes blink { 50% { opacity: 0; } }
.cursor { animation: blink 1s steps(1) infinite; }
```

The "runs on page load?" row is the important one. A transition needs a **before-change
style** to transition *from* — the element's style at the previous style computation
([css-transitions-1 §3](https://drafts.csswg.org/css-transitions-1/#starting)). An element
that is being styled for the first time, or was `display: none` until now, has none, so
nothing animates. File 04 is about the feature that fixes this (`@starting-style`), and
about the bug you get when the browser computes a style earlier than you expected.

Some properties can't be interpolated at all — there's no value halfway between `block` and
`none`. Those are **discrete** properties. By default a transition simply doesn't run for
them and they flip instantly; file 04 §4.3 is how to change that.

## 1.6 `prefers-reduced-motion`

Motion makes some people ill. Operating systems have a "reduce motion" setting, and CSS
exposes it as a media feature. The spec defines `reduce` as: the user "prefers an interface
that removes or replaces the types of motion-based animation that either trigger discomfort
for those with vestibular motion sensitivity, or distraction for those with attention
deficits" ([mediaqueries-5](https://drafts.csswg.org/mediaqueries-5/#prefers-reduced-motion)).

```css
@media (prefers-reduced-motion: reduce) {
  .type   { animation: none; width: auto; }   /* show the final state */
  .panel  { transition-duration: 0.01s; }
}
```

The rule of thumb is **show the end state, not nothing**. A typing effect with its
animation removed must still show the text; an exit transition removed must still remove
the element. A common practice is to keep small fades and drop large movement, zooming
and parallax, since the spec's wording targets motion that causes vestibular discomfort —
but that's a judgement, not a rule. Widely available (Baseline since January 2020, per
[web-features](https://web-platform-dx.github.io/web-features/) data, checked October 2026).

## 1.7 Progressive enhancement — new features without breaking old browsers

Much of this course uses features from 2023–2025, and one (`interpolate-size`) that only
Chromium has. That's workable because CSS fails safe by design:

| Failure | What the browser does | Spec |
|---|---|---|
| Unknown property or invalid value | Drops that one declaration, keeps the rest of the rule | [css-syntax-3](https://drafts.csswg.org/css-syntax-3/#consume-declaration) |
| Unknown selector in a selector list | **Drops the whole rule** | [selectors-4 §3.9](https://drafts.csswg.org/selectors-4/#invalid) |
| Unknown at-rule | Ignores the block | css-syntax-3 |

The first row gives you **fallback by repetition** — write the old value first, the new one
second:

```css
.card {
  transition: opacity 0.25s;                 /* older browsers keep this */
  transition: opacity 0.25s, display 0.25s allow-discrete;  /* newer ones override */
}
```

The second row is a trap: `a:hover, a:some-future-thing { … }` dies completely in a browser
that doesn't know the second selector. Wrap the risky part in `:is()` or `:where()`, whose
argument lists are *forgiving* (invalid entries are skipped), or split the rule.

For explicit tests, there's `@supports`:

```css
@supports (interpolate-size: allow-keywords) {
  :root { interpolate-size: allow-keywords; }
}
@supports selector(:has(a)) { /* … */ }
```

**Progressive enhancement** is the discipline of making the page *correct* with the oldest
behaviour, then making it *nicer* where the new feature exists. For stateful UI the
correctness bar is: the content is reachable, the state can be changed with a keyboard, and
nothing is hidden permanently because an animation never ran. An expanding panel that
snaps open instead of sliding is fine. A panel that never opens is not.

### "Baseline" — reading support claims

Support statements in this course use **Baseline**, the cross-browser status published by
the WebDX community group and shown on MDN:

| Status | Meaning |
|---|---|
| **Limited availability** | Missing from at least one of Chrome, Edge, Firefox, Safari |
| **Newly available** | In the current version of all four, desktop and mobile |
| **Widely available** | Newly available for at least 30 months |

([web.dev/baseline](https://web.dev/baseline)). The raw data is the
[`web-features`](https://github.com/web-platform-dx/web-features) package, which in turn
reads MDN's [`browser-compat-data`](https://github.com/mdn/browser-compat-data). Every
version number in this course was read from those two packages (browser-compat-data 8.1.4,
dated 1 October 2026) — not from memory. Support moves; check the MDN page before you
quote any of it.

---

## Check yourself

1. A `.panel { opacity: 0.5 }` rule and an `#main .panel { opacity: 1 }` rule both match.
   Which wins, and does it matter which comes later in the file?
2. A `@keyframes` animation with `animation-fill-mode: forwards` set `opacity: 0` at its end.
   You now add `.visible { opacity: 1 }` with an ID selector in front. Why does the element
   stay invisible, and what are your two ways out?
3. You put `overflow: hidden` on a flex item to clip a sliding animation, and the item
   suddenly gets shorter than its content. Explain the mechanism.
4. Why does adding a class that changes `opacity` from 0 to 1 animate, but inserting a new
   element that has `opacity: 1` and a `transition` on opacity doesn't?
5. You write `button:focus-visible, button:-moz-focusring { outline: 2px solid }`. In Chrome
   your red outline never appears — only the browser's default ring. What happened, and how do you write it so it degrades
   correctly?

(Answers in [`08-exercises.md`](08-exercises.md).)
