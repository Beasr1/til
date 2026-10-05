# 2. State without JavaScript

## 2.1 The problem: CSS has no variables you can flip

A "show more" toggle needs one bit of state: open or closed. The JavaScript reflex is a
click handler that adds a class. That works, but it means the toggle is dead until the
script loads, dead if it fails, and you've written code for a boolean.

CSS can't store state. What it *can* do is **read** state that already lives in the DOM.
So the question is never "how do I make CSS remember something", it's **"which HTML element
already holds this bit, and which selector can see it?"**

| Where the bit lives | Selector that reads it | Changed by | Notes |
|---|---|---|---|
| `<details open>` | `details[open]` | Clicking / Enter / Space on `<summary>` | Real semantics. First choice for disclosure |
| A checkbox's checked state | `:checked` | Clicking it or its `<label>`, Space | Any element can be the visible control via `<label>` |
| A radio group | `:checked` | Same; one per `name` | Tabs, mutually exclusive options |
| The URL fragment | `:target` | Following `#id` links | Survives reload, shareable, adds history entries |
| Focus | `:focus-within` | Tabbing / clicking into it | Lost as soon as focus moves — not durable |

This chapter covers the first two in depth, then `:has()`, which removes the main
restriction both of them have.

## 2.2 `<details>` and `<summary>` — the one that comes with semantics

```html
<details>
  <summary>Build log</summary>
  <pre>…long output…</pre>
</details>
```

No CSS, no script, and you already have: a focusable control, keyboard operation, an
expanded/collapsed state that assistive technology announces, and the `open` attribute
reflecting the state so CSS can style it:

```css
details[open] > summary { font-weight: bold; }
details:not([open]) > summary::after { content: " …"; }
```

Three newer features make it more useful than it used to be:

| Feature | What it does | Support (as of Oct 2026, MDN BCD) |
|---|---|---|
| `name` attribute | `<details name="faq">` on several elements makes them **exclusive** — "at most one" in the group can be open, an accordion with no script ([HTML §4.11.1](https://html.spec.whatwg.org/multipage/interactive-elements.html#the-details-element)) | Baseline newly available Sept 2024 (Chrome 120, Firefox 130, Safari 17.2) |
| `::details-content` | A pseudo-element for the collapsible part, so it can be styled and **animated** (file 04 §4.5) | Baseline newly available Sept 2025 (Chrome 131, Firefox 143, Safari 18.4) |
| Find-in-page | Searching the page for text inside a closed `<details>` opens it ([HTML §6.9.2](https://html.spec.whatwg.org/multipage/interaction.html#interaction-with-details-and-hidden=until-found)) | Specified; I didn't check per-browser support |

What it can't do is the reason the next section exists: **the toggle must be the
`<summary>`, and the thing that changes must be inside the `<details>`.** You can't put the
control in a toolbar and the panel somewhere else, and you can't easily restyle the summary
into something that isn't a disclosure.

## 2.3 The checkbox pattern

A checkbox is a boolean the browser already stores, toggles, and exposes to CSS as
`:checked`. A `<label for>` makes any element a click target for it. Put those together:

```html
<!doctype html>
<style>
  .panel { display: none; }
  #more:checked ~ .panel { display: block; }

  /* swap the label text */
  .label-close            { display: none; }
  #more:checked + label .label-open  { display: none; }
  #more:checked + label .label-close { display: inline; }
</style>

<div class="card">
  <input type="checkbox" id="more" class="visually-hidden">
  <label for="more">
    <span class="label-open">Show details</span>
    <span class="label-close">Hide details</span>
  </label>
  <div class="panel">The expanded content.</div>
</div>
```

Read the selectors as a sentence: "a `.panel` that is a **later sibling** (`~`) of a checked
`#more`". `+` means "the immediately following sibling".

> ⚠️ Sibling combinators only look **forward** and only among **siblings**. That's why the
> input has to come *first* and sit at the *same level* as everything it controls. Put it
> inside the label, or after the panel, and the selector silently matches nothing. This
> constraint shapes the whole markup — §2.4 is how to escape it.

### Keep it keyboard-operable

The tempting way to hide the input is `display: none`. Don't. An element that isn't rendered
can't take focus. I checked in Chrome: with `display: none` on the checkbox, pressing Tab
from the previous control skips straight past it. The label still works with a mouse, so
the bug is invisible to anyone who tests by clicking.

Hide it *visually* instead, so it stays focusable and stays in the accessibility tree:

```css
.visually-hidden {
  position: absolute;
  width: 1px; height: 1px;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
}
```

Now Tab lands on the checkbox and Space toggles it. But the user can't *see* where focus
is, because the focused thing is a 1px invisible box. Move the focus ring onto the label:

```css
#more:focus-visible + label {
  outline: 2px solid;
  outline-offset: 2px;
}
```

**`:focus-visible`** matches when the browser decides focus should be shown — in practice,
keyboard focus but not a mouse click. I measured both: after Tab, the label got its outline;
after clicking the label, the checkbox was focused but `:focus-visible` didn't match, so no
ring. That's the behaviour you want.

> **Teacher's aside.** The checkbox pattern is accessible *as a checkbox*. A screen reader
> will announce something like "Show details, checkbox, not checked" (exact wording varies by
> screen reader) — honest, operable, but not what the
> control is. A disclosure is semantically a button with `aria-expanded`, and that's what
> `<details>/<summary>` gives you for free. So the order of preference is: `<details>` if the
> layout allows it; the checkbox if you need the control and panel separated and can live
> with "checkbox"; a real `<button>` plus a few lines of script if neither is acceptable.
> "No JavaScript" is a means, not a virtue.

## 2.4 `:has()` — selecting upwards

Every combinator before 2023 pointed down or forward. **`:has()`** points the other way: an
element matches if something *relative to it* matches
([selectors-4 §4.5](https://drafts.csswg.org/selectors-4/#relational)).

```css
/* the card, when the checkbox inside it is checked */
.card:has(#more:checked) { border-color: currentColor; }

/* now the panel can be anywhere inside the card */
.card:has(#more:checked) .panel { display: block; }

/* an h2 immediately followed by a p */
h2:has(+ p) { margin-bottom: 0.25rem; }
```

The second rule is the important one. With `:has()`, the input no longer has to precede the
panel or be its sibling — it just has to be somewhere inside the same ancestor. The
markup is freed from the selector.

Rules to know, from the spec and MDN:

| Rule | Consequence |
|---|---|
| Specificity is that of its most specific argument | `.card:has(#more:checked)` is `(1,2,0)` — the ID counts |
| Can't be nested: `:has(:has(…))` is invalid | Flatten into one relative selector |
| Pseudo-elements aren't allowed inside it | `:has(::before)` is invalid |
| In a browser without `:has()`, the whole rule is dropped (file 01 §1.7) | Put the essential behaviour in a sibling rule and use `:has()` for extras, or test with `@supports selector(:has(a))` |

Support: Baseline **widely available** since June 2026 — newly available December 2023
(Chrome 105, Firefox 121, Safari 15.4), per web-features data checked October 2026.

On performance: `:has()` makes the browser re-check ancestors when descendants change. MDN's
guidance is to anchor it narrowly (`.card:has(…)`, not `body:has(…)`) and keep the inner
selector tight with `>` or `+`. I haven't measured this; treat it as guidance, not a number.

## 2.5 Putting it together

```html
<!doctype html>
<style>
  .visually-hidden { position:absolute; width:1px; height:1px; overflow:hidden;
                     clip-path:inset(50%); white-space:nowrap; }
  .card            { border: 1px solid #ccc; padding: 1rem; }
  .panel           { display: none; }
  .card:has(.toggle:checked) .panel   { display: block; }
  .card:has(.toggle:checked)          { border-color: #333; }
  .toggle:focus-visible + label       { outline: 2px solid; outline-offset: 2px; }
  .when-open                          { display: none; }
  .card:has(.toggle:checked) .when-open   { display: inline; }
  .card:has(.toggle:checked) .when-closed { display: none; }
</style>

<section class="card">
  <header>
    <input type="checkbox" id="t1" class="toggle visually-hidden">
    <label for="t1"><span class="when-closed">Expand</span><span class="when-open">Collapse</span></label>
  </header>
  <div class="panel">Content that only appears when expanded.</div>
</section>
```

The input lives in a header, the panel is a cousin, not a sibling. Without `:has()` this
markup couldn't work. Save it as a file, open it, and try Tab then Space.

---

## Check yourself

1. Your panel is controlled by `#t:checked ~ .panel`. A designer moves the checkbox inside
   the `<label>` "to keep things tidy". What breaks, and why does it break silently?
2. You hide the checkbox with `display: none` and every click test passes. Describe a user
   for whom the control is now broken, and the exact mechanism.
3. Why style `#t:focus-visible + label` rather than `#t:focus + label`? What would a mouse
   user see with the second?
4. What's the specificity of `.card:has(#t:checked) .panel`, and why might that bite you
   when a later, plainer rule tries to hide the panel?
5. You need four FAQ entries where opening one closes the others, with no script. Compare
   doing it with radio buttons and with `<details name>`. Which would you choose and what
   do you give up?
6. Give one reason to choose a `<button>` and three lines of JavaScript over either pattern
   in this chapter.

(Answers in [`08-exercises.md`](08-exercises.md).)
