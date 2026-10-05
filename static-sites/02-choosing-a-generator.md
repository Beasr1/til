# 2. Choosing a static site generator

## 2.1 The problem

File 01 §1.3 said a generator turns content and templates into files. There are dozens
of them, they all do that, and their home pages all claim to be fast and flexible. So the
choice can't be made on features alone — it has to be made on what you'll be living
with: what you install, what you write templates in, and what happens to your site if
the project changes hands or stops.

This chapter compares four generators that between them cover the design space:
**Hugo**, **Zola**, **Eleventy** and **Astro**. Every version and date here was checked
on 5 October 2026; generators move fast, so recheck before quoting.

## 2.2 The axis that matters most: what you install

The biggest practical difference isn't the template syntax. It's whether the generator
is **one binary** or **a dependency tree**.

```
 single binary (Hugo, Zola)                 npm package (Eleventy, Astro)
 ──────────────────────────                 ─────────────────────────────
 download one file, run it                  need Node.js, then npm install
 no runtime on the build machine            node_modules/ with hundreds of packages
 version = one number                       version = a lockfile
 extend it: only what's built in            extend it: anything on npm
 ▲ less to break, less to audit             ▲ more power, more moving parts
```

Neither is better. A single binary means a build from three years ago still runs if you
kept the binary, and that your CI downloads one file you can checksum (file 04 §4.6). An
npm tool means you can do almost anything — any Markdown plugin, any image pipeline, any
UI framework — at the cost of a large dependency tree that changes underneath you unless
you lock it.

## 2.3 The four, side by side

| | **Hugo** | **Zola** | **Eleventy** | **Astro** |
|---|---|---|---|---|
| Written in | Go | Rust | JavaScript (Node.js) | JavaScript/TypeScript (Node.js) |
| Install | Single executable | Single binary, "everything built-in" | `npm` package | `npm create astro@latest`; needs Node ≥ 22.12 |
| Latest (5 Oct 2026) | v0.167.0 (28 Sep 2026) — still 0.x | v0.23.6 (12 Sep 2026) — still 0.x | v3.1.6 (2 Jun 2026); v4 in alpha | v7.3.5 (24 Sep 2026) |
| Templating | Go `html/template` / `text/template` | **Tera** (Jinja2-like) | Your choice: Nunjucks, Liquid, WebC, Markdown, HTML, JS/JSX, Handlebars, and more | `.astro` components — HTML with JSX-like expressions — plus React/Vue/Svelte etc. |
| Client-side JS by default | None added | None added | "zero client-side JavaScript by default" | None; opt in per component with `client:*` (**islands**) |
| Feeds | RSS generated **by default** for home, sections, taxonomies | Atom and RSS templates built in; **off** by default (`generate_feeds = false`) | Official plugin `@11ty/eleventy-plugin-rss` (Atom, RSS, JSON Feed) | Official package `@astrojs/rss` |
| Assets | Hugo Pipes (bundling, minifying, Sass via Dart Sass, images) | Built-in Sass, syntax highlighting, image resizing | Bring your own (plugins) | Vite-based bundling and image handling |
| GitHub stars (5 Oct 2026) | ~90k | ~17.5k | ~20k | ~63k |
| Stewardship | Open-source project | Open-source project | Owned by Font Awesome since Sept 2024; rebranded 2026 (§2.5) | Company joined Cloudflare, Jan 2026 (§2.5) |

Sources: [Hugo intro](https://gohugo.io/about/introduction/), [Hugo templates](https://gohugo.io/templates/introduction/),
[Hugo RSS](https://gohugo.io/templates/rss/), [Hugo Pipes](https://gohugo.io/hugo-pipes/introduction/);
[Zola overview](https://www.getzola.org/documentation/getting-started/overview/), [Zola repo](https://github.com/getzola/zola),
[Zola feeds](https://www.getzola.org/documentation/templates/feeds/);
[11ty.dev](https://www.11ty.dev/), [Eleventy RSS plugin](https://www.11ty.dev/docs/plugins/rss/);
[Astro install](https://docs.astro.build/en/install-and-setup/), [Astro islands](https://docs.astro.build/en/concepts/islands/),
[Astro RSS](https://docs.astro.build/en/guides/rss/). Version numbers and dates are from
the GitHub Releases API and npm registry; star counts from the GitHub API.

Stars measure attention, not quality or longevity. They are a rough proxy for how likely
it is that someone else has already hit your problem and written about it.

Two notes that change older advice you'll find online:

- **Hugo "extended" is no longer needed for Sass.** Extended exists for the embedded
  LibSass, which Hugo deprecated in v0.153.0 (December 2025): "Use the Dart Sass
  transpiler instead, which is compatible with any edition"
  ([Hugo — css.Sass](https://gohugo.io/functions/css/sass/)). Dart Sass is a separate
  program you install alongside Hugo, so "single binary" becomes two if you use Sass.
- **Zola's feed settings were renamed in 0.19.0** (June 2024): `generate_feed` became
  `generate_feeds`, and `feed_filename` became the list `feed_filenames`
  ([Zola CHANGELOG](https://github.com/getzola/zola/blob/master/CHANGELOG.md)). Config
  copied from an older tutorial fails to parse: Zola's `Config` struct is declared with
  `deny_unknown_fields`, and its test suite includes
  `test_backwards_incompatibility_for_feeds`, which asserts that `generate_feed = true`
  is rejected (`components/config/src/config/mod.rs` in the Zola repo).

## 2.4 How to pick

| If you… | Lean towards | Because |
|---|---|---|
| Want the least to install and maintain | Zola or Hugo | One binary, pinned by one version and one checksum |
| Want feeds, Sass and image resizing with no plugins | Zola | All built in; nothing to wire up |
| Have a large site or want the biggest theme ecosystem | Hugo | Fastest builds of the four by reputation (not benchmarked here), most themes and answers |
| Already write JavaScript and want to choose your template language | Eleventy | It supports many languages and imposes little structure |
| Want interactive components on some pages, static HTML elsewhere | Astro | Islands: ship JS only for the components that need it |
| Want to avoid touching Node.js at all | Hugo or Zola | Eleventy and Astro both require it |

> **Teacher's aside.** People choose a generator by its feature list, then discover the
> thing they actually spend time on is the template language. Hugo's Go templates are
> powerful and notoriously odd at first — `{{ range .Pages }}`, the meaning of `.`
> changing inside blocks. Tera feels like Jinja2 or Django templates. Eleventy lets you
> pick something you already know. Write one non-trivial template — a list page with
> pagination — in your two shortlisted options before you commit. An hour there saves a
> week of fighting the one you picked from a table.

## 2.5 Stewardship: who owns the tool you build on

A generator is a long-lived dependency. Your content is portable — it's Markdown — but
your templates are written in one engine's dialect and don't move. So it's worth knowing
who controls the project and how that has changed. Two of the four changed hands in the
last two years.

**Eleventy.** On 12 September 2024 the project announced "11ty is joining Font Awesome",
with its creator Zach Leatherman saying "No action is required by folks using and
building with Eleventy"
([11ty blog](https://www.11ty.dev/blog/eleventy-font-awesome/)). Then:

| Date | Event | Source |
|---|---|---|
| 3 Mar 2026 | "Eleventy is now Build Awesome" — a rebrand, with a Kickstarter for a commercial "Build Awesome Pro" builder; the post says Pro "will not be required to use Build Awesome (Eleventy)" | [11ty blog](https://www.11ty.dev/blog/build-awesome/) |
| 6 Mar 2026 | Font Awesome pauses the Kickstarter. Stated reason: email deliverability — launch emails reached "5-10%" of recipients. Restates that Build Awesome "will always be free and open source" | [Font Awesome blog](https://blog.fontawesome.com/pausing-kickstarter/) |
| 28 Apr 2026 | Kickstarter relaunched; the page later thanks "838 backers" | [11ty blog](https://www.11ty.dev/blog/build-awesome-pro/) |
| Oct 2026 | GitHub `11ty/eleventy` redirects to `11ty/buildawesome`; the npm package is still `@11ty/eleventy` | GitHub, npm (checked 5 Oct 2026) |

The rebrand drew public criticism from long-time users, which you can read in community
posts from March 2026. A third-party blog reported in mid-March that the renaming plans
had been "canned"; I could find **no primary source supporting that**, and the project's
own later posts, its GitHub repository and its new home page all use the new name. As of
October 2026 the evidence says: the Kickstarter was paused and relaunched; the rename was
not reversed. If you read otherwise, check the date and whether it cites Font Awesome or
the project itself.

**Astro.** On 16 January 2026 Cloudflare announced "The Astro Technology Company,
creators of the Astro web framework, is joining Cloudflare", stating Astro "will remain
open source, MIT-licensed, and open to contributions, with a public roadmap and open
governance" ([Cloudflare blog](https://blog.cloudflare.com/astro-joins-cloudflare)).

Neither change has, on the evidence, made either tool worse to use. The point is the
general one: a tool backed by a company follows that company's incentives, and promises
about the future are promises. An MIT- or similarly licensed generator can be forked if
those incentives drift — which is the real protection, and is worth confirming in the
licence file rather than the press release.

## 2.6 The template-engine trap: your HTML is now template source

Once a file goes through a template engine, every character in it is read by that
engine first. Most of the time that's invisible. It stops being invisible when the HTML
you hand-wrote happens to contain the engine's delimiters.

Tera has three, documented as: "`{{` and `}}` for expressions, `{%` and `%}` for
statements, `{#` and `#}` for comments" ([Tera docs](https://keats.github.io/tera/)).
Jinja-family engines (Nunjucks, Jinja2) use the same three; Liquid and Go templates use
`{{ }}` too.

Where they turn up in perfectly ordinary hand-written markup:

| You wrote | Engine sees | Result |
|---|---|---|
| Minified CSS: `@media (max-width:40em){#nav{display:none}}` | `{#` — start of a comment | Everything until the next `#}` vanishes, or a parse error about an unclosed comment |
| Inline JS: `const cfg = {{a: 1}}` — nested object literal | `{{` — start of an expression | A parse error, or `a: 1` evaluated as template code |
| A code sample in a post showing Jinja, Vue, Handlebars or Go template syntax | Real template tags | The engine executes your example instead of displaying it |

> ⚠️ **Errors here often don't point at the cause.** A stray `{#` in CSS may produce a
> complaint about the end of the file, or simply a page with a chunk missing and no error
> at all. When a template "loses" content, search the template for the three delimiters
> before anything else.

The fixes, best first:

1. **Keep CSS and JS out of templates.** Put them in their own files, served as assets.
   The engine never reads them. This also lets the browser cache them.
2. **Escape the region.** Tera's `{% raw %}…{% endraw %}` block: "Tera will consider all
   text inside the raw block as a string and won't try to render what's inside"
   ([Tera docs](https://keats.github.io/tera/)). Nunjucks and Jinja have the same idea.
3. **Break the pattern.** A space — `{ #nav` or `{ {a: 1} }` — is legal CSS and JS and is
   not a delimiter. Fine as a one-off; fragile if a minifier later removes the space.

Whether this reaches your **Markdown content** depends on the generator, and they differ
in a way that surprises people moving between them. Some render Markdown to HTML and
insert the result into a template, so the engine never reads your prose. Eleventy, by
contrast, runs Markdown through a template engine first — "Markdown files run through
this template engine before transforming to HTML", with the default `liquid`
([Eleventy — configuration, `markdownTemplateEngine`](https://www.11ty.dev/docs/config/)).
Zola sits between: it looks for its **shortcode** syntax (`{{ name(...) }}`, `{% name() %}`)
inside Markdown. Test a post containing `{{` and `{%` in a code block before you rely on
either behaviour.

---

## Check yourself

1. Give one concrete situation in which "single binary" is the deciding factor, and one
   in which "it's on npm" is. What property of each are you actually buying?
2. You copy a `config.toml` from a 2023 Zola tutorial containing `generate_feed = true`
   and build with Zola 0.23. What happens, and how would you have predicted it?
3. A layout template contains `<style>@media print{#comments{display:none}}</style>`.
   The built page is missing its footer and there's no error. Explain the mechanism and
   give two fixes, saying which you'd choose and why.
4. A blog post written in Markdown shows readers a Jinja2 snippet. It renders correctly
   on one generator and breaks on another. What's the difference between the two
   generators' pipelines that explains this?
5. A colleague argues: "Eleventy is now owned by a company and was renamed, so it's
   risky; Hugo and Zola are safer." Evaluate the claim. What would you check, in which
   files, to decide how exposed a site is to a generator's stewardship changing?

(Answers in [08-exercises.md](08-exercises.md).)
