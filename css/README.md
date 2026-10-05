# Modern CSS for Stateful, Animated UI — A Course

A course on building UI that has state and motion — panels that open, views that overlap, text
that types itself out, logs that stay pinned to the newest line — with little or no
JavaScript. Written for someone who knows HTML and basic CSS but learned them before the
layout and animation model changed underneath.

**This is reference learning material.** It's about the CSS specifications and how browsers
implement them, not about any particular site. The examples are neutral on purpose: a panel
that expands over a block, a log view that keeps the newest line visible.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Every chapter opens with the problem the feature exists to solve.
- Every behavioural claim was either checked against a spec or measured in a real browser
  (Chrome 154, October 2026), and the text says which. Where I couldn't confirm something,
  the text says that too.
- Every file ends with **Check yourself** questions. Answers are in `08-exercises.md`.

## The primary sources

There's no reference *implementation* for a language; the equivalents are the specs, the
compatibility data, and the browser's own tooling. These are what the chapters cite.

| Source | What it's for | Link |
|---|---|---|
| **CSS Working Group Editor's Drafts** | The specs themselves: cascade, selectors, flexbox, alignment, containment, transitions, overflow | <https://drafts.csswg.org/> |
| **CSSWG issue tracker** | Where spec changes are argued and resolved — e.g. the `safe` alignment issue in file 05 | <https://github.com/w3c/csswg-drafts/issues> |
| **WHATWG HTML** | `<details>`, `innerText`, find-in-page, the rendering rules | <https://html.spec.whatwg.org/multipage/> |
| **CSSOM View** | `scrollTop`, `getClientRects()`, `scroll-behavior` | <https://drafts.csswg.org/cssom-view-1/> |
| **Web Animations** | `getAnimations()`, `Animation.currentTime` | <https://drafts.csswg.org/web-animations-1/> |
| **MDN Web Docs** | Readable reference, with compatibility tables | <https://developer.mozilla.org/en-US/docs/Web/CSS> |
| **MDN browser-compat-data** | The raw per-browser version data behind MDN's tables | <https://github.com/mdn/browser-compat-data> |
| **web-features / Baseline** | Cross-browser status and dates ("newly" / "widely available") | <https://github.com/web-platform-dx/web-features>, <https://web.dev/baseline> |
| **Chrome DevTools Protocol** | Driving and inspecting Chrome from tests | <https://chromedevtools.github.io/devtools-protocol/> |
| **Puppeteer** | The Node library used in file 06 | <https://pptr.dev/> |

Support statements are dated and come from browser-compat-data 8.1.4 (1 October 2026) and the
matching web-features release. They will go stale; recheck MDN before quoting one.

## Reading order

### Part 1 — Foundations (prerequisite for everything else)

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Foundations](01-foundations.md) | Predict which declaration wins, why a box is the size it is, which layout mode places it, and whether a change will animate |

### Part 2 — State and layout

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [State without JavaScript](02-state-without-javascript.md) | Build toggles with `<details>`, the checkbox pattern and `:has()` — keyboard-accessible |
| 3 | [Stacking and sizing](03-stacking-and-sizing.md) | Overlap elements without layout jumps; size text so N characters always fit, at every width |

### Part 3 — Motion

| # | File | After this you can… |
|---|------|---------------------|
| 4 | [Enter and exit](04-enter-and-exit.md) | Animate elements in and out of `display: none` and to `height: auto`, and avoid the cold-load trap |
| 5 | [**Terminal-style output**](05-terminal-style-output.md) | ⭐ Typing and line reveals, a log that sticks to the bottom, and four traps that cost real time |

### Part 4 — Proving it

| # | File | After this you can… |
|---|------|---------------------|
| 6 | [Testing in a real browser](06-testing-in-a-real-browser.md) | Assert layout to the pixel, freeze animations at exact times, and reproduce first-visit bugs |

### Reference

| # | File | |
|---|------|--|
| 7 | [Glossary](07-glossary.md) | Look things up |
| 8 | [Exercises & answers](08-exercises.md) | Worked answers to every Check yourself, plus things to try |

## If you're short on time

- **30 minutes:** file 01 §1.2 and §1.5, then file 05.
- **You need a toggle by this afternoon:** file 02, then file 04 §4.3.
- **Something animates wrong and you don't know why:** file 01 §1.2 (origins, shorthands), file 04
  §4.6, file 05 §5.3.
- **You need to prove a layout holds:** file 03 §3.4, then file 06.

## The one-paragraph summary of everything

CSS doesn't run; it resolves. The cascade picks one value per property — and running animations
and transitions outrank your ordinary rules, while every shorthand silently resets the longhands
it omits. State can live in HTML that CSS can already read: `<details open>` with real
semantics, a visually hidden (never `display: none`) checkbox read by `:checked`, and `:has()`
to read it from anywhere in an ancestor. A one-cell grid stacks elements in flow, sized by the
largest; `contain: size` makes one of them stop counting; container query units size text to
its box rather than the window, which closes the gaps breakpoints leave. Transitions fire on any
computed-value change between style passes: `@starting-style` gives new elements something to
start from, `allow-discrete` keeps `display` visible through an exit, `interpolate-size` reaches
`height: auto` in Chromium only — and the same mechanism makes delayed exits fire on a cold page
load unless gated. Paint effects like `clip-path` don't move scroll geometry; layout effects like
`max-height` in `lh` do. A `column-reverse` scroller has its origin at the bottom, so it pins the
newest line by itself, and an empty `flex: 1 1 0` spacer puts short content at the top — where
`safe` alignment, in today's Chrome, does not. Prove all of it by measuring geometry in a real
browser with time frozen, not by comparing screenshots.

## How to use me

Ask anything, including:

- "Explain the cascade again, but with this element's DevTools styles pane"
- "Why did my `@starting-style` not apply here?"
- "Draw the flex free-space calculation for the spacer, with numbers"
- "Show me the same component three ways and what a screen reader announces for each"
- "Has the `safe` alignment change shipped anywhere yet?"
- "Write the puppeteer test for this component"
- "Is my mental model right? Here's what I think happens…"

I'll add chapters if a topic earns one — scroll-driven animations and view transitions are the
obvious next candidates.
