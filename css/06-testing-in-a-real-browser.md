# 6. Testing CSS behaviour in a real browser

## 6.1 The problem: these bugs are geometry over time

Every bug in this course has the same shape: a box is in the wrong place, or the wrong size,
**at some moment**, under **some condition** — a viewport width nobody tried, a cold cache, a
point halfway through a transition. A unit test can't see any of it, because no layout
happens. Eyeballing can't either, because you can't eyeball 350 viewport widths or the 30ms
before a stylesheet arrives.

What works is driving a real browser and **measuring**. Three tools cover nearly everything:

| Need | Tool |
|---|---|
| Where is this box / line, to the pixel? | `getBoundingClientRect()`, `Range.getClientRects()` |
| What does it look like at t = 400ms, repeatably? | Pause and seek animations (Web Animations API, or the DevTools protocol) |
| What happens on a first visit? | Disable the cache and slow down the stylesheet |

## 6.2 Screenshots are the wrong primary assertion

The reflex is screenshot-and-compare. For static layout that's fine. For anything animated,
a screenshot is one frame captured at a moment you don't control. It's sometimes said that
headless screenshots fast-forward or skip transitions. I couldn't reproduce that: in my test
(Chrome 154, puppeteer-core, default headless mode) they captured an in-progress frame.
But they weren't repeatable: a 4s opacity transition read 0.125 by `getComputedStyle` just
before `page.screenshot()`, the screenshot's pixels implied 0.137, and the computed value
read 0.137 just after. The capture itself took about 60ms of animation time. Run it again and
you get a different frame.

So: assert on **numbers** read from the page, and when you do need a picture, freeze time
first (§6.4) so the picture is deterministic.

## 6.3 Setup, and measuring geometry

**puppeteer-core** is the Puppeteer library without a bundled browser; you point it at a
Chrome you already have installed ([pptr.dev](https://pptr.dev/)).

```sh
mkdir css-tests && cd css-tests && npm init -y && npm install puppeteer-core
```

```js
// measure.cjs
const puppeteer = require('puppeteer-core');

(async () => {
  const browser = await puppeteer.launch({
    executablePath: '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome', // adjust per OS
    headless: true,
  });
  const page = await browser.newPage();
  await page.setViewport({ width: 640, height: 800 });
  await page.goto('file://' + __dirname + '/page.html');

  const r = await page.evaluate(() => {
    const box  = document.querySelector('.log').getBoundingClientRect();
    const last = document.querySelector('.log .lines > :last-child').getBoundingClientRect();
    return { gap: box.bottom - last.bottom };
  });
  console.log(r.gap === 0 ? 'newest line pinned' : `off by ${r.gap}px`);
  await browser.close();
})();
```

`getBoundingClientRect()` gives an element's border box in viewport coordinates, after
transforms. That answers "is the newest line at the bottom edge?" to the pixel.

### Counting rendered lines: `Range.getClientRects()`

"Does this text wrap?" is a question about **line boxes**, which elements don't expose. A DOM
`Range` does. CSSOM View defines `Range.getClientRects()` to include, for each text node in the
range, a rectangle per line fragment, "computed using font metrics", plus the border boxes of
any whole elements selected ([cssom-view-1 §9](https://drafts.csswg.org/cssom-view-1/#dom-range-getclientrects)).
So for an element containing only text, one rectangle per rendered line:

```js
function renderedLines(el) {
  const range = document.createRange();
  range.selectNodeContents(el);
  const tops = new Set([...range.getClientRects()].map(r => Math.round(r.top)));
  return tops.size;
}
```

Measured on a 120px-wide paragraph of 43 characters in 16px monospace: four rectangles, at
tops 8, 28, 48, 68 — one per line, 20px apart. Note each rectangle was 19px tall, not the 20px
line height: the height comes from the font's metrics, not from `line-height`. Compare tops,
not heights. If the element has child elements, their border boxes are in the list too;
dedupe by `top` as above, or walk the text nodes yourself.

### Sweep the variable instead of sampling it

The wrapping bug in file 03 §3.4 lived in bands 40–80px wide. Testing 375, 768 and 1280 misses
them. Sweep:

```js
const bad = [];
for (let w = 300; w <= 1000; w += 2) {
  await page.setViewport({ width: w, height: 800 });
  const n = await page.evaluate(() => renderedLines(document.querySelector('.box')));
  if (n > 1) bad.push(w);
}
```

(`renderedLines` must be defined in the page, e.g. via `page.evaluateOnNewDocument` or a
`<script>` in the fixture.) That sweep is what found the
three bands in file 03.

## 6.4 Controlling time

### From inside the page: the Web Animations API

Every CSS transition and `@keyframes` animation is exposed as an `Animation` object
([web-animations-1 §6.10](https://drafts.csswg.org/web-animations-1/#extensions-to-the-documentorshadowroot-interface-mixin)).
You can pause them and set their time:

```js
await page.evaluate((ms) => {
  for (const a of document.getAnimations()) { a.pause(); a.currentTime = ms; }
}, 450);
// now measure, or screenshot — the frame is exactly t = 450ms
```

Every timed measurement in this course was taken this way. It's standard API, works in all
engines, and needs nothing beyond `page.evaluate`.

One caveat: per the spec, calling `getAnimations()` "triggers a style change event for the
target element". That's usually what you want. It's *not* what you want when you're trying to
observe what the browser does **on its own** — a style change you forced might create the bug
you're looking for, or hide it.

### From outside: the DevTools protocol's Animation domain

Puppeteer can open a raw **Chrome DevTools Protocol** (CDP) session. Its **Animation** domain
reports animations without touching the page, and can pause and seek them
([CDP Animation domain](https://chromedevtools.github.io/devtools-protocol/tot/Animation/)).
The whole domain is marked **experimental** in the protocol definition, so it can change
between Chrome versions.

| Command / event | Use |
|---|---|
| `Animation.enable` | Start receiving events |
| `Animation.animationStarted` (event) | Passive detector: an animation or transition started — with its type and timing |
| `Animation.setPaused { animations, paused }` | Freeze specific animations |
| `Animation.seekAnimations { animations, currentTime }` | Jump them to a time |
| `Animation.setPlaybackRate { playbackRate }` | Slow everything down (e.g. 0.1) for frame capture |

```js
const cdp = await page.createCDPSession();
await cdp.send('Animation.enable');
const ids = [];
cdp.on('Animation.animationStarted', e => ids.push(e.animation.id));

// … trigger the transition …
await cdp.send('Animation.setPaused', { animations: ids, paused: true });
for (const t of [1000, 2000, 3000]) {
  await cdp.send('Animation.seekAnimations', { animations: ids, currentTime: t });
  await page.screenshot({ path: `frame-${t}.png` });
}
```

Measured on a 4s linear opacity transition from 0 to 1, black on white: seeking to 1000, 2000,
3000ms gave screenshot pixels implying opacity 0.247, 0.498, 0.749, matching the computed
0.25, 0.5, 0.75. Deterministic frames.

## 6.5 Cold loads

The cold-load bug in file 04 §4.6 only appears when the stylesheet arrives *after* the first
style computation. Two things reproduce it reliably:

1. **Disable the cache** for the session, so every load is a first visit:
   `Network.setCacheDisabled { cacheDisabled: true }` (CDP Network domain, not experimental).
2. **Slow the stylesheet down.** A tiny local server that delays CSS responses by a few hundred
   milliseconds is more predictable than network throttling.

```js
const http = require('http');
const server = http.createServer((req, res) => {
  if (req.url.endsWith('.css')) {
    setTimeout(() => { res.writeHead(200, { 'content-type': 'text/css' }); res.end(css); }, 600);
  } else {
    res.writeHead(200, { 'content-type': 'text/html' }); res.end(html);
  }
}).listen(0);

const cdp = await page.createCDPSession();
await cdp.send('Network.enable');
await cdp.send('Network.setCacheDisabled', { cacheDisabled: true });
await cdp.send('Animation.enable');
const started = [];
cdp.on('Animation.animationStarted', e => started.push(e.animation));

await page.goto(`http://127.0.0.1:${server.address().port}/`, { waitUntil: 'load' });
await new Promise(r => setTimeout(r, 400));
console.log(started.length ? 'transition fired on load' : 'clean load');
```

The Animation domain is the right detector here precisely because it doesn't force a style
computation in the page (§6.4). This harness produced the table in file 04 §4.6, including the
"warm cache: no transition" row, by flipping `cacheDisabled` back to `false` and loading twice.

> ⚠️ Run cold-load tests in a fresh page or browser context. A stylesheet cached by an earlier
> test in the same run turns a cold-load test into a warm one, and the test passes for the
> wrong reason.

## 6.6 Media features

Emulate user preferences rather than changing OS settings:

```js
await page.emulateMediaFeatures([{ name: 'prefers-reduced-motion', value: 'reduce' }]);
const running = await page.evaluate(() => document.getAnimations().length);
// expect 0, and the final state visible
```

Measured on the typing example from file 05 §5.2: under `reduce`, zero animations and
the text at full width (192.7px for 20 Menlo characters at 16px).

## 6.7 What a headless run doesn't tell you

- **Fonts are the machine's fonts.** Advance widths and line counts depend on what's installed.
  The numbers in this course come from macOS. A CI machine with different fonts will measure
  differently — pin a web font if the test depends on glyph metrics.
- **One engine.** Puppeteer drives Chrome. The `safe` alignment behaviour in file 05 §5.5 and
  `interpolate-size` support both differ between engines. Playwright can drive Firefox and
  WebKit builds if you need cross-engine measurements; I haven't used it for this course.
- **No real user input timing.** `keyboard.press` and `click` are synthetic. They are good
  enough for `:focus-visible` (file 02 §2.3 was measured this way) but not for anything that
  depends on gesture velocity, like the user's own smooth scrolling.

---

## Check yourself

1. Why is "take a screenshot 400ms after clicking and compare it to a reference image" a flaky
   test for a 1s transition, even if screenshots capture mid-transition frames correctly?
2. You want to assert that a 60-character line never wraps. Why measure with
   `Range.getClientRects()` rather than the element's height divided by line height?
3. You're investigating whether a transition fires on first load. Why might calling
   `document.getAnimations()` from a polling script make the investigation unreliable, and
   what do you use instead?
4. A cold-load test passes locally every time and the bug still reaches users. Name two ways
   the test could be accidentally warm.
5. Design a test that proves the log in file 05 §5.5 keeps the newest line visible while
   lines are being added *and* doesn't move the view when the user has scrolled up. Which
   numbers would you assert on?

(Answers in [`08-exercises.md`](08-exercises.md).)
