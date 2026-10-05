# 7. Glossary

Reference, not reading. The file in brackets is where the term is taught.

## The cascade and selectors

| Term | Meaning |
|---|---|
| **Cascade** | The algorithm that picks one winning declaration per property per element (01) |
| **Origin** | Where a declaration came from: user agent, user, author, plus the animation and transition origins (01) |
| **Cascade layer** | A named group created with `@layer`; layer order outranks specificity (01) |
| **Specificity** | `(A, B, C)` = IDs, classes/attributes/pseudo-classes, types/pseudo-elements; compared left to right (01) |
| **`:is()` / `:not()` / `:has()` specificity** | That of the most specific argument (01, 02) |
| **`:where()`** | Like `:is()` but always zero specificity (01) |
| **Forgiving selector list** | The argument of `:is()` / `:where()`: invalid entries are skipped instead of killing the rule (01) |
| **Shorthand / longhand** | `animation` is a shorthand for `animation-delay` etc.; a shorthand resets every longhand it omits to its initial value (01, 05) |
| **Inherited property** | One whose value passes from parent to child by default (`color`, `font`, `interpolate-size`) (01) |
| **`:has()`** | Relational pseudo-class: matches an element if a selector relative to it matches. Selects "upwards" (02) |
| **`:checked`** | Matches a checked checkbox or radio button (02) |
| **`:focus-visible`** | Matches when the browser decides focus should be shown — typically keyboard focus, not mouse clicks (02) |
| **Sibling combinators** | `+` next sibling, `~` any later sibling. Forward only (02) |

## Boxes and layout

| Term | Meaning |
|---|---|
| **Box model** | Content, padding, border, margin (01) |
| **`box-sizing: border-box`** | `width` includes padding and border (01) |
| **Intrinsic size** | The size a box wants from its content: min-content, max-content (01) |
| **Flow / flex / grid** | The three main layout modes, chosen by the parent's `display` (01) |
| **Main axis / main-start** | Flexbox's layout direction and its starting edge; the bottom in `column-reverse` (01, 05) |
| **Automatic minimum size** | Flex items' default `min-height`/`min-width: auto` = "not smaller than my content", unless the item is a scroll container (01) |
| **Free space** | Space left in a flex container after items take their basis; shared by `flex-grow` (05) |
| **Grid stack** | One-cell grid with every child at `grid-area: 1 / 1`; the cell sizes to the largest child (03) |
| **Size containment** | `contain: size`: the box's intrinsic size is computed as if it had no content (03) |
| **Sizer** | An invisible (`visibility: hidden`) child that reserves space from content (03) |
| **Query container** | An element with `container-type`, whose size descendants can query (03) |
| **`cqi`** | 1% of the nearest ancestor query container's inline size (03) |
| **`ch`** | Advance width of "0" in the element's font; exact per-character only in monospace (05) |
| **`lh`** | The element's computed line height (05) |
| **Safe / unsafe alignment** | Whether alignment may push content into an unreachable overflow area; `safe` falls back on overflow (05) |

## Scrolling

| Term | Meaning |
|---|---|
| **Scroll container** | A box with `overflow` other than `visible`/`clip` (01, 05) |
| **Scroll origin** | The corner a scroll container's overflow area expands from; main-start cross-start in flexbox (05) |
| **Upward overflow direction** | Overflow that extends above the origin; `scrollTop` is ≤ 0 (05) |
| **Scroll anchoring** | Browser adjusts scroll position so visible content doesn't jump when content above it changes (05) |
| **`scroll-behavior`** | Whether *programmatic* scrolls on a box are instant or smooth. Doesn't affect user scrolling (05) |

## Change over time

| Term | Meaning |
|---|---|
| **Transition** | Interpolation triggered by a computed value changing between style computations (01) |
| **Animation** | `@keyframes` sequence that runs while `animation-name` applies (01) |
| **Before-change style** | An element's style at the previous style computation; what a transition starts from (01, 04) |
| **Discrete property** | One with no intermediate values (`display`, `content-visibility`) (01, 04) |
| **`@starting-style`** | Rules giving the style to transition *from* when there's no before-change style (04) |
| **`transition-behavior: allow-discrete`** | Lets discrete properties transition; `display` then stays visible for the whole run (04) |
| **`interpolate-size: allow-keywords`** | Allows interpolation between a length and a size keyword like `auto` (04) |
| **`::details-content`** | Pseudo-element for the collapsible part of `<details>` (02, 04) |
| **Inert** | Not focusable or clickable; an element exiting via `display` transition is inert (04) |
| **`steps(n)`** | Easing that moves in n discrete jumps (05) |
| **`prefers-reduced-motion`** | Media feature reflecting the user's OS "reduce motion" setting (01) |

## Tooling and support

| Term | Meaning |
|---|---|
| **Progressive enhancement** | Correct with old behaviour, nicer with new (01) |
| **`@supports`** | Conditional rules based on whether a property/value or selector is supported (01) |
| **Baseline** | Cross-browser status: limited / newly available / widely available (30 months after newly) (01) |
| **puppeteer-core** | Puppeteer without a bundled browser; drives an installed Chrome (06) |
| **CDP** | Chrome DevTools Protocol; raw commands like `Animation.seekAnimations` (06) |
| **Web Animations API** | `document.getAnimations()`, `Animation.pause()`, `currentTime` (06) |
| **`Range.getClientRects()`** | Per-line rectangles for text in a range (06) |
| **`innerText`** | Rendered text, with line breaks — but falls back to `textContent` when the element isn't rendered (05) |
