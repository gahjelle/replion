# Rendering Markdown and diffs in a Lustre client

Research for [#4](https://github.com/gahjelle/replion/issues/4). Question: how should the Lustre SPA (`client/`, JavaScript target, see ADR 0001) render (a) Markdown in Replies and (b) file diffs for edit-type Tool Calls, including syntax highlighting? A Reply is typed out character by character, so the Markdown gets re-rendered on every tick.

Versions checked 2026-09-17 against hex.pm and the npm registry (latest stable releases).

## Recommendation

- **Markdown:** use pure Gleam, **`mork` 1.12.1** plus **`mork_to_lustre` 1.0.0**, which gives you Lustre `Element`s with no JS FFI and no `innerHTML`. For the typing effect, **parse the whole Reply once and reveal a growing prefix of the parsed document's visible text** instead of re-parsing a growing prefix of the source. This means you never show half-typed syntax like `**bold` that suddenly turns bold. Write your own small renderer over `mork/document` (a few hundred lines, with `mork_to_lustre` as a model) so you control truncation, code-block highlighting and raw HTML.
- **Diffs:** no library is needed for Claude Code. Its Edit/Write tool results already hold `structuredPatch` hunks (jsdiff format) plus `oldString`/`newString`/`originalFile`. Render those hunks as a hand-made Lustre table. Use **`gliff` 1.1.0** (pure Gleam, Myers/Patience) only when a Harness gives just the old and new strings, and ideally run it on the server in the Adapter, so the shared Session type always carries hunks.
- **Syntax highlighting:** **`smalto` 3.0.0** plus **`smalto_lustre` 3.0.0** (pure Gleam, Prism-derived grammars for 36 languages, Lustre elements, 45 themes in `smalto_lustre_themes`). This beats highlight.js/Shiki behind FFI: those return HTML strings, which would force `unsafe_raw_html`.
- JS libraries (marked, markdown-it, micromark, jsdiff, diff2html, highlight.js, Shiki) all work but return HTML strings or need async setup. Keep them as fallbacks if a Gleam package turns out to be buggy or too slow.

## (a) Markdown

### Gleam options

| Package | Version | Notes |
|---|---|---|
| [`mork`](https://hexdocs.pm/mork/) | 1.12.1 (2026-05) | Pure Gleam, "100% spec compliant with Commonmark", most GFM extensions, footnotes. Tested on Node, Deno and Bun. Its public AST lives in `mork/document` (`Code(lang, text)`, `HtmlBlock(raw)`, `RawHtml`, `InlineHtml`, ...). Tables, tasklists and autolinks are off by default (`mork.configure()`, then `mork.tables(True)` etc.). The README notes that commonmark.js is "probably faster" on JS. |
| [`mork_to_lustre`](https://hexdocs.pm/mork_to_lustre/) | 1.0.0 | `to_lustre(Document) -> List(Element(msg))`. The README calls it "experimental". It renders `HtmlBlock` via `element.unsafe_raw_html` (source `src/mork/to_lustre.gleam:170`) and has no hook for highlighting code blocks. |
| [`maud`](https://hexdocs.pm/maud/) | 2.0.0 | MDX-style: `render_markdown(md, mork.configure(), components)` with a setter for each element (`pre`, `code`, `h1`, ...). It also passes `HtmlBlock` through `unsafe_raw_html` (`src/maud/internal/render.gleam:63`) and drops inline `RawHtml`. `gleam.toml` declares `target = "erlang"`, but the package is pure Gleam and compiled for JS in the benchmark below. |
| [`commonmark`](https://hexdocs.pm/commonmark/) | 0.1.8 (last release 2024-06) | Pre-1.0 and inactive. Not recommended. |
| [`jot`](https://hexdocs.pm/jot/) | 12.1.1 | Parses **Djot**, not Markdown. Wrong language for agent output. |
| `kirala_markdown`, `gbr_md_lustre`, `render_md` | | Small, older or niche. See [packages.gleam.run](https://packages.gleam.run/?search=markdown). |

### JS options (via FFI)

| Package | Version | Notes |
|---|---|---|
| [`marked`](https://github.com/markedjs/marked) | 18.0.13 | Fast. `marked.lexer()` returns a token tree. It **does not sanitize** output; the README recommends DOMPurify. |
| [`markdown-it`](https://github.com/markdown-it/markdown-it) | 15.0.2 | CommonMark plus plugins. `html: false` is the default, and [docs/safety.md](https://github.com/markdown-it/markdown-it/blob/master/docs/safety.md) says output is then safe without a sanitizer. Has a `highlight(str, lang)` callback that returns HTML. |
| [`micromark`](https://github.com/micromark/micromark) | 4.0.2 | Small, spec-compliant, gives positional info. HTML string output (or mdast via the unified ecosystem). |
| [`streamdown`](https://github.com/vercel/streamdown) / [`remend`](https://github.com/vercel/streamdown/tree/main/packages/remend) | 2.6.0 / 1.3.1 | Streamdown is React-only. `remend(text)` is framework-free: it "completes" partial Markdown (closes `**`, `` ` ``, links, code fences, ...) before parsing, so it could run in front of any parser. |

Using any of these from Lustre means an HTML string passed to `element.unsafe_raw_html`, which Lustre docs warn about ("may expose your applications to XSS attacks", `lustre/element.gleam`). Lustre also replaces the whole inner HTML on every change, which throws away DOM diffing on each tick. You could turn the JS token tree into Lustre elements yourself, but at that point you are writing the same renderer you would write over `mork`, and you pay for an FFI boundary as well.

### Incremental rendering while typing

There are two approaches.

1. **Re-parse a growing source prefix on every tick** (what streaming chat UIs do). Measured with `mork` + `mork_to_lustre` on Node 26: re-parsing and rendering every prefix of a 1,272-char Reply (headings, list, table, fenced code) took **1,050 ms total, about 0.8 ms per tick** on average. That is cheap enough. The downside is what the audience sees: `mork.parse("Some **bold and \`co")` gives `<p>Some **bold and \`co</p>`, so the markers show as literal text and then "pop" into formatting once they close. An unclosed code fence is fine (it renders as an open `<pre><code>`). `remend` exists only to hide this effect.
2. **Parse once, reveal a prefix of the rendered text** (recommended). Replion has the whole Reply up front, unlike a live stream. Parse the full Reply into a `mork` `Document` once (for example when the Event is first shown). Then on each tick render the tree truncated to the first *N* visible characters: walk blocks and inlines, count the text in `Text`/`CodeSpan`/`Code` leaves, and cut the node that reaches *N*. Formatting shows up the moment its first character does, raw Markdown syntax never appears, and per-tick cost is a tree walk with no parsing. Lustre's virtual DOM then only adds one text change per tick. You also get to decide the reveal unit: characters, words, or whole code blocks at once.

Either way, if long Replies get slow, wrap finished blocks in `element.memo(dependencies, view)` (Lustre 5.7.1, compares by reference via `element.ref`). Lustre's docs say `memo` is usually unnecessary, so measure first.

### Raw HTML safety

Both Gleam renderers pass `HtmlBlock` through `unsafe_raw_html`. Replion replays the presenter's own local sessions, so the risk is low. Even so, a Reply that *talks about* HTML (for example `<script>` in prose outside a code fence) would be injected. A custom renderer should show `HtmlBlock`/`RawHtml` as escaped text.

## (b) Diffs for edit-type Tool Calls

### What the Harness already stores

In Claude Code session JSONL, the `toolUseResult` of an Edit Tool Call has the keys `filePath, oldString, newString, originalFile, replaceAll, structuredPatch, userModified`. A Write has `type, filePath, content, originalFile, structuredPatch, userModified`. `structuredPatch` is a list of hunks `{oldStart, oldLines, newStart, newLines, lines: [" ctx", "-old", "+new", ...]}`. This is exactly the jsdiff `structuredPatch` hunk format ([jsdiff README](https://github.com/kpdecker/jsdiff/blob/master/README.md)). (Observed in local `~/.claude/projects/*/*.jsonl` files. Other Harnesses such as Copilot CLI were not checked here.)

So for Claude Code, the Adapter can map these hunks into a shared `Hunk` type, and the client just renders rows: gutter line numbers, a `+`/`-` class, and optionally smalto-highlighted content. No diff algorithm runs in the browser.

### When only old/new strings exist

| Package | Version | Target | Notes |
|---|---|---|---|
| [`gliff`](https://hexdocs.pm/gliff/) | 1.1.0 | Gleam, both targets, depends only on `gleam_stdlib` | `diff` (lines, Myers), `diff_patience`, `diff_words`, `diff_chars`, `to_unified`, `from_unified`, `inline_highlight` (character-level spans within changed lines). Measured: a 400-line file with 200 changed lines took about 119 ms on JS (cold, one-off). Do this once per Tool Call, preferably on the server. |
| [`diff` (jsdiff)](https://github.com/kpdecker/jsdiff) | 9.0.0 (npm) | JS | `diffLines`, `diffWordsWithSpace`, `structuredPatch`, `createTwoFilesPatch`. The reference implementation behind Claude Code's hunk format. |
| [`diff2html`](https://github.com/rtfpessoa/diff2html) | 3.4.56 (npm) | JS | Turns a unified-diff string into an HTML string (line-by-line or side-by-side). `Diff2HtmlUI` draws into a DOM node and highlights with highlight.js. Complete-looking, but it owns its DOM and CSS, which fights Lustre and the Theme. Not recommended. |
| hex `diff` | 1.1.0 (2018) | Elixir | Not a Gleam package. |

Hand-rolling a Lustre diff view is small: a `<table>` with one row per hunk line, plus optional word-level emphasis from `gliff.diff_words` / `inline_highlight` for paired `-`/`+` lines.

## Syntax highlighting

| Option | Version | Output | Notes |
|---|---|---|---|
| [`smalto`](https://hexdocs.pm/smalto/) + [`smalto_lustre`](https://hexdocs.pm/smalto_lustre/) (+ `smalto_lustre_themes`) | 3.0.0 | Tokens, then Lustre `Element`s (inline styles or your own classes via builder functions) | Pure Gleam, both targets. Regex grammars converted from Prism.js: 36 languages including Gleam, Python, JS/TS, Rust, Bash, JSON, YAML, TOML, SQL, Markdown. Measured 23 ms for 400 lines of Python on JS. Class-based config (`smalto_lustre.keyword(fn(v) { html.span([class("smalto-keyword")], ...) })`) lets the Theme own the colors through CSS variables. |
| Single-language Gleam highlighters: `contour` (Gleam), `just` (JS), `pearl` (Erlang), `tear` (Elixir), `william` (TOML), `swatch` (CSS) | | tokens | Only relevant if smalto is weak for a given language. |
| [`highlight.js`](https://highlightjs.org) | 11.12.0 (npm) | HTML string, synchronous | Auto-detects language. Needs `unsafe_raw_html` (its output is escaped, so that is safe in practice). |
| [`shiki`](https://github.com/shikijs/shiki) | 4.4.3 (npm) | HTML string, HAST, or tokens (`codeToTokens`) | TextMate grammars, the most accurate option. Setup is async (`createHighlighter`), but `createHighlighterCoreSync` with `createJavaScriptRegexEngine()` works synchronously once the grammars are imported ([sync-usage guide](https://github.com/shikijs/shiki/blob/main/docs/guide/sync-usage.md)). Token output could be mapped to Lustre elements via FFI. Heaviest bundle. |

Highlighting a code block that is still being typed means re-tokenizing a growing prefix. With approach 2 above, you can instead highlight the complete block once, then truncate the token list, so colors stay stable as the code types out.

For diffs, highlight the whole old and new files (or at least whole hunks) once, then give each diff line its tokens. Highlighting line by line breaks multi-line strings and comments.

## Benchmark notes

Throwaway Gleam 1.18.1 project, JS target, Node 26.2.0, packages as listed above. The numbers are single runs on a laptop, useful only for order of magnitude. Code: parse and `element.to_string` for every prefix; `gliff.diff` on 400 lines with every other line changed; `smalto.to_tokens` plus `smalto_lustre.to_lustre` for 400 lines of Python.

## Sources

- hex.pm package API and hexdocs READMEs: mork, mork_to_lustre, maud, commonmark, jot, gliff, smalto, smalto_lustre, lustre 5.7.1 (package sources read from `repo.hex.pm` tarballs)
- [packages.gleam.run](https://packages.gleam.run/) searches for "markdown", "diff", "highlight", "syntax"
- npm registry `latest` for marked, markdown-it, micromark, diff, diff2html, highlight.js, shiki, streamdown, remend, dompurify
- Context7 docs for markdown-it (options and safety), marked (security warning, lexer), jsdiff (structuredPatch), diff2html (Diff2HtmlUI), shiki (sync usage, fine-grained bundles), streamdown/remend
- Local Claude Code session JSONL files for the `toolUseResult` shape
