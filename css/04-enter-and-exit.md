# 4. Enter and exit — animating things that appear and disappear

## 4.1 The problem: `display: none` can't be animated

You have a panel that toggles between `display: none` and `display: block`. You add
`transition: opacity 0.3s` and give the hidden state `opacity: 0`. Nothing animates in
either direction. It pops in and pops out.

Two separate mechanisms cause this, one for each direction:

| Direction | Why it doesn't animate |
|---|---|
| **Enter** (`none` → `block`) | A transition needs a *before-change style* to start from. An element that wasn't rendered at the previous style computation has none, so there's nothing to transition from (file 01 §1.5) |
| **Exit** (`block` → `none`) | `display` is a *discrete* property. By default transitions don't run on discrete properties at all, so `display` flips to `none` immediately — and the box, with its opacity animation, is gone in the same frame |

Height has a similar problem from a different cause. `height: 0` → `height: auto` doesn't
animate because `auto` is a keyword, not a number, and you can't compute a halfway point
between a number and a keyword.

Until 2023–24 the fixes were JavaScript: measure the element, set explicit pixel heights,
listen for `transitionend`, then apply `display: none`. This chapter covers the three CSS
features that replace that, and one trap that comes with them.

| Feature | Fixes | Support (as of Oct 2026, MDN BCD / web-features) |
|---|---|---|
| `@starting-style` | Enter | Baseline newly available Aug 2024 — Chrome 117, Firefox 129, Safari 17.5 |
| `transition-behavior: allow-discrete` | Exit (and `display` generally) | Baseline newly available Aug 2024 — Chrome 117, Firefox 129, Safari 17.4 |
| `interpolate-size: allow-keywords` | Height to/from `auto` | **Limited availability** — Chrome/Edge 129 only; not in Firefox or Safari |

## 4.2 Enter: `@starting-style`

**`@starting-style`** lets you declare the style an element should transition *from* when it
has no before-change style. The spec defines the **starting style** as "the after-change
style with `@starting-style` rules applied in addition", used "instead of the before-change
style" when there isn't one
([css-transitions-2 §3.3](https://drafts.csswg.org/css-transitions-2/#defining-before-change-style)).

```css
.panel {
  opacity: 1;
  transition: opacity 0.4s;
}
@starting-style {
  .panel { opacity: 0; }        /* "if you're appearing, start from here" */
}
```

When `.panel` goes from not rendered to rendered — inserted into the DOM, or leaving
`display: none` — the browser computes its style twice: once with the `@starting-style` rules
(opacity 0), once without (opacity 1), and transitions between them.

Measured in Chrome 154 with a 1s transition: opacity 0.02 at 30ms, 0.80 at 500ms, 1.00 at
1.2s. The same page without the `@starting-style` block: opacity 1.00 at 30ms — no
animation.

> ⚠️ **Order matters.** `@starting-style` adds no specificity. Its rules cascade against the
> others normally, so if the block comes *before* the base rule, the base rule's
> `opacity: 1` wins the tie on order and the starting style is opacity 1 — nothing animates.
> I checked: with the block after the base rule, opacity was 0.07 at 100ms; moved before
> it, 1.00. Put `@starting-style` after the rule it modifies, or nest it inside that rule.

## 4.3 Exit: `transition-behavior: allow-discrete`

To keep the element on screen while it fades out, `display` itself has to participate in
the transition. **`transition-behavior: allow-discrete`** permits transitions on discrete
properties ([css-transitions-2](https://drafts.csswg.org/css-transitions-2/#transition-behavior-property)),
and `display` has a special rule for it. CSS Display 4 §2.9 says that "during interpolation
between `none` and any other display value, *p* values between 0 and 1 map to the non-none
value" ([css-display-4](https://drafts.csswg.org/css-display-4/#display-animation)). In plain
terms:

```
 entering  none → block    display:  block ███████████████████  (block for the whole run)
 exiting   block → none    display:  block ███████████████████▏none  (flips at the very end)
```

So the element is visible for the entire transition in both directions — exactly what the
opacity fade needs.

```css
.panel {
  opacity: 1;
  transition: opacity 0.4s, display 0.4s allow-discrete;
}
.panel.hidden {
  display: none;
  opacity: 0;
}
@starting-style {
  .panel { opacity: 0; }
}
```

`display 0.4s allow-discrete` inside the `transition` shorthand is the same as listing
`display` in `transition-property` and setting `transition-behavior: allow-discrete`.

Measured, adding `.hidden` with a 1s transition: `display: block`, opacity 0.98 at 30ms;
`block`, 0.20 at 500ms; `none`, 0 at 1.2s. Without `display … allow-discrete` in the list:
`display: none` at 30ms. The element was gone before the fade started.

The same spec paragraph adds: "the element is inert as long as its display value would
compute to none when ignoring the Transitions and Animations cascade origins". So a panel
that is fading out can't be clicked or focused, even though you can still see it. I checked:
200ms into a 1s exit, `elementFromPoint` over a button inside the panel did not return the
button, and `focus()` didn't move focus to it. That's what you want — no clicks on a
disappearing dialog.

> ⚠️ The `transition` shorthand resets `transition-behavior` like any other longhand (file 01
> §1.2). A rule like `.panel.fast { transition: opacity 0.1s }` written to speed up one case
> also turns `allow-discrete` off, and that case loses its exit animation. I confirmed the
> computed `transition-behavior` goes back to `normal`. Repeat the whole list in every
> override, or override only `transition-duration`.

## 4.4 Height to and from `auto`: `interpolate-size`

A collapsible section wants `height: 0` closed and `height: auto` open. **`interpolate-size:
allow-keywords`** allows "a `<size-keyword>` and a `<length-percentage>`" to be interpolated,
by treating the keyword as though it were `calc-size(keyword, size)`
([css-values-5 §11.3](https://drafts.csswg.org/css-values-5/#interpolate-size)). The property
inherits, so the usual move is to set it once:

```css
:root { interpolate-size: allow-keywords; }

.section        { height: 0; overflow: hidden; transition: height 0.3s; }
.section.open   { height: auto; }
```

Measured with a 1s linear transition to a 90px `auto` height: with the default
(`numeric-only`), height was already 90px at 500ms; with `allow-keywords`, 43.5px at 500ms
and 90px at 1.2s.

Two limits from the spec: it's keyword ↔ length only — two *different* keywords
(`min-content` ↔ `max-content`) still don't interpolate.

**Support is the real constraint.** As of October 2026, MDN lists `interpolate-size` (and the
related `calc-size()` function) in Chrome and Edge 129+ only; Firefox and Safari have no
support. That's acceptable *because* it fails safe: in other browsers the property is
ignored and the section snaps open, which is the correct end state. This is progressive
enhancement working as intended — don't build anything that depends on the intermediate
frames.

## 4.5 All three together: an animated `<details>`

`<details>` has an extra wrinkle. When closed, its content isn't hidden with `display: none`
but with `content-visibility: hidden` on the `::details-content` slot — the HTML rendering
rules set "display: block; content-visibility: hidden" on it while `open` is absent
([HTML §15.5.5](https://html.spec.whatwg.org/multipage/rendering.html#the-details-and-summary-elements)).
So `content-visibility` is the discrete property to transition:

```html
<!doctype html>
<style>
  :root { interpolate-size: allow-keywords; }

  details::details-content {
    block-size: 0;
    overflow: hidden;
    transition: block-size 0.3s, content-visibility 0.3s allow-discrete;
  }
  details[open]::details-content {
    block-size: auto;
  }
</style>
<details>
  <summary>Build log</summary>
  <p>step 1 … ok<br>step 2 … ok<br>step 3 … ok</p>
</details>
```

Measured on a five-line body with a 0.6s close: 108px open, 63px at 300ms, 18px (just the
summary) at the end. Remove `content-visibility … allow-discrete` from the list and it was
18px at 300ms — the content vanished at once and only the opening direction animated.

`::details-content` is Baseline newly available since September 2025 (Chrome 131, Firefox
143, Safari 18.4). In Firefox and Safari the height still won't animate (no
`interpolate-size`), but the page works.

## 4.6 The cold-load trap

Exit transitions are often written with a delay — "stay visible for two seconds, then fade
out":

```css
.toast { opacity: 0; transition: opacity 0.3s 1.5s; }
.toast.show { opacity: 1; transition-delay: 0s; }
```

That's fine when it starts from a known state. Now consider what the transition machinery
sees on page load. A transition starts whenever an element's computed value changes between
two style computations. On a warm load, the stylesheet is present the first time the
element is styled, so the first computed opacity is already 0 — no change, no transition.

But if the browser computes styles **before your stylesheet arrives**, the element is first
styled with browser defaults (`opacity: 1`). When the sheet lands, opacity changes 1 → 0,
and that is a perfectly valid style change: the exit transition fires. The user sees the
element sitting there at full opacity for 1.5s, then fading away, on a page where nothing
was ever shown.

When can styles be computed before the sheet? Anything that lets rendering or a script run
ahead of it:

| Situation | Transition fired on load? (measured, 600ms-slow stylesheet, cache disabled) |
|---|---|
| `<link rel=stylesheet>` in `<head>` (render-blocking) | No |
| Async CSS pattern: `media="print" onload="this.media='all'"` | **Yes** — `opacity`, delay 1500ms |
| `<link>` placed after the element, with a script reading layout before it | **Yes** |
| Async pattern, **warm cache** | No |

The last row is why this bug survives testing: it only fires on a **cold** load. Reload the
page and the stylesheet comes from cache before first style, and everything looks right.

The fix is to not let the transition exist until the page has settled. Gate it behind a
class added after `load`:

```css
.toast { opacity: 0; }
.ready .toast { transition: opacity 0.3s 1.5s; }
```

```html
<script>
  addEventListener('load', () =>
    requestAnimationFrame(() => document.documentElement.classList.add('ready')));
</script>
```

Same test harness with this gate and the async pattern: no transition fired on cold load.

> **Teacher's aside.** People think of transitions as "an animation I attach to a state".
> The mechanism is narrower and dumber than that: *any* change in a computed value between
> two style passes animates, whatever caused it — a class, a media query flipping, a font
> loading, a stylesheet arriving late. Once you hold that model, the cold-load bug stops
> being mysterious, and so does the "my element animates when I resize the window" bug.

(The `.ready` gate is JavaScript. It's the kind this course accepts: if the script never
runs, the toast simply doesn't animate, and the page is still correct.)

## 4.7 Reduced motion for enter and exit

Removing the transitions entirely under `prefers-reduced-motion` is correct here, because
`display` then flips instantly and the end state is right:

```css
@media (prefers-reduced-motion: reduce) {
  .panel, details::details-content { transition: none; }
}
```

---

## Check yourself

1. Your panel has `transition: opacity 0.4s` and toggles `display: none`. Explain why it
   doesn't animate in **either** direction, naming a different mechanism for each.
2. A colleague puts `@starting-style { .panel { opacity: 0 } }` at the top of the stylesheet
   and the enter animation never runs. What's wrong?
3. During an exit transition with `allow-discrete`, the panel is visible for 400ms. A user
   double-clicks a button inside it. What happens and why?
4. Explain to someone who's never seen it why an exit animation with a delay can play on
   first page load but never on reload.
5. You ship `interpolate-size: allow-keywords` for an accordion. A Safari user reports "it
   doesn't animate". Is that a bug? What would make it one?
6. A designer writes `.panel.compact { transition: opacity 0.1s; }` to speed up one variant.
   Compact panels stop fading out and just vanish. Explain mechanically.

(Answers in [`08-exercises.md`](08-exercises.md).)
