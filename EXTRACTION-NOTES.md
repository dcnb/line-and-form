# Gutenberg Extraction — Work Notes

Working notes on `gutenberg-extraction.py`, covering a debugging and feature
session on 2026-08-21. Started from a failed extraction of book 25290 (*Line and
Form*) and ended with table support and a two-mode illustration pipeline.

---

## 1. Why book 25290 failed

The reported error was:

```
[4/5] Parsing content and converting to Markdown...
  Found 0 front matter + 0 chapters
  No chapters detected — extracting as single document...
ERROR: Could not extract any content
```

**Cause: gzip.** `HEADERS` advertises `Accept-Encoding: gzip, deflate`, but
`urllib` does not decompress automatically. `make_request` was calling
`.decode('utf-8', errors='replace')` on raw gzip bytes, so the "HTML" handed to
the parsers was binary garbage:

```
"\x1f\x8b\x08\x00\x00\x00\x00\x00\x00\x03\xd4\xfd\xdb\x92..."
```

With no tags to see, the chapter parser found nothing, and `WholeBookParser` —
which only emits on `</p>`, `</pre>`, `</h1-4>` — produced an empty string.

This was never specific to 25290. Gutenberg's response carries
`vary: Accept-Encoding`, so whether a given request came back compressed
depended on which backend served it. Earlier extractions happened to get
uncompressed responses.

**Fix:** `read_response()` inflates per the `Content-Encoding` header and decodes
using the response's declared charset.

---

## 2. What changed

### Fetching

| Change | Why |
|---|---|
| `read_response()` | Decompress gzip/deflate; use the declared charset |
| `looks_like_html()` guard in `download_html()` | An unreadable response now falls through to the next source with a message, instead of failing three steps later |
| Size floor of 2000 chars in that guard | A Gutenberg 504 page is valid HTML and passed the tag check |
| `curl -sfL` in the fallback chain | Without `-f`, curl returned HTTP error bodies with exit 0 |
| `max_retries` / `use_fallbacks` params on `make_request` | Bulk image fetches need a lighter policy — a throttled miss cost ~25s through the full retry-plus-wget-plus-curl chain |

### Structure detection

**`find_toc_region()`** replaced two brittle regexes. The old heading pattern
(`<h[1-4]>.*?contents.*?</h[1-4]>` with DOTALL) started at the *first* heading in
the document and ran until it found the word "contents" anywhere, capturing a
huge slice. For *Pride and Prejudice* that pulled 101 `page_*` links out of the
illustration table and turned each into a section boundary — **156 "chapters"**
including files named `49-page_118.md`. Now:

1. An explicit `toc`/`contents` container, extended from `div|nav|section` to
   also cover `p|ul|ol|table` (P&P's real TOC is a `<p class="toc">`).
2. Failing that, the block after a heading whose **own text** is "Contents".
3. A candidate with fewer than 2 internal links is rejected and the search
   continues — Moby Dick's `<p class="toc">` holds only the word "CONTENTS", and
   accepting it silently dropped its Etymology and Extracts sections.
4. With no TOC at all, page anchors are excluded from the whole-document scan.

Note these books genuinely conflict: *Line and Form*'s chapter anchors **are**
`Page_1`, `Page_23`…, so the page-anchor filter is scoped to the no-TOC path
only.

Also: figure anchors (`#f016`) no longer become chapters; sections with no body
text (`has_body_text()`) are dropped rather than written as empty essay files;
and the `<a id>` section path now classifies via `is_section_id()` instead of a
second, divergent set of rules.

### Titles

Illustrated editions put `<span class="pagenum">` and `<span class="caption">`
inside headings, which produced titles like `{ix}` and
`"I hope Mr. Bingley will like it. CHAPTER II."`. Both are now excluded from
title text.

### Tables

`render_markdown_table()` + `TableCollector`, wired into both parsers. Handles
`colspan`, `<br>` and multi-paragraph cells folded onto one line, `|` escaping,
`&nbsp;` cells.

Three judgment calls worth knowing:

- **Headerless output by default.** No book in the test corpus uses `<th>` at
  all, and kramdown accepts a table with no header row. A `<th>` in the first row
  still produces a header + separator.
- **Single-column tables render as paragraphs.** Gutenberg uses them for
  centering — Frankenstein has a 28-row one, Grimms 63.
- **TOC tables are still dropped.** Alice's and Frankenstein's entire tables of
  contents are tables; they sit before the first section so they never enter a
  section's content. CB-Essay generates its own contents nav.

Recovered content includes Gatsby's daily-schedule table and Moby Dick's
whale-etymology table, both previously lost outright. One subtlety: collapsing a
cell onto one line stranded emphasis markers (`* Greek*` renders as a literal
asterisk), so markers are pulled back against their text.

Illustration lists were never sections before — with tables dropped they were
always empty, so nothing claimed them. `list of illustrations`, `illustrations`
and `plates` are now front-matter keywords.

### Illustrations

Before this work, **images effectively did not reach the essays at all**.
*Line and Form* had 168 `<img>` tags and produced **zero** image references. The
parser appended `![alt](src)` to `current_content`, which is only flushed on
`</p>`, `</pre>`, `</blockquote>`, `</li>` — and all 168 of its images sit inside
a `<div>` or a bare `<a>`, so the buffer was discarded at the next `<p>`.

P&P showed the failure sharply: 164 images in, 60 out — and the 60 survivors
were all decorative drop-cap initials inside `<p class="nind">`. Every one of the
37 Hugh Thomson plates was dropped.

Three further bugs behind that:

- `--all-images` renamed downloads to `img-001-<stem>.ext` in `objects/` while
  the Markdown pointed at Gutenberg's relative `images/image002.png` — neither
  path resolved.
- `--all-images` + `--local-html` hung: `base_url` was `file://…`, so every image
  resolved to a nonexistent local path and burned the full retry chain. A run was
  killed after 10 minutes with zero images downloaded.
- Alt text was silently lost. The regex in `extract_image_urls` put an optional
  `alt` group after a greedy `[^>]*`, so it never matched: **168 images found, 0
  alt texts**, though every image in the book has one.

New pipeline:

- `FigureParser` reads attributes through `HTMLParser`, so alt survives any
  attribute order. It captures the stable anchor (`f002`), the hi-res variant
  from a wrapping `<a href>`, and captions from a sibling `<span class="caption">`.
- `extract_figure_captions()` maps anchors to captions via the book's list of
  illustrations — which only works now that tables parse.
- `is_decorative()` skips drop caps (alt of 1–2 characters). Empty alt is *not*
  decorative — that's an uncaptioned plate.
- `gutenberg_image_urls()` resolves relative sources against Gutenberg rather
  than the page they came from, so `--local-html` runs can still fetch images.
- `download_figures()` names files by objectid so the CSV, the object file, and
  the Markdown reference all agree.
- `write_collection_csv()` + `update_config_metadata()` produce
  `_data/<slug>-metadata.csv` and repoint `_config.yml`.
- `build_figure_references()` decides how each figure appears in the essay.

For *Line and Form* this yields 168 figures, all with alt text, 151 with captions
from the illustration list, 167 with stable anchors, 46 with hi-res variants.
P&P yields 105 real figures with 59 drop caps filtered out.

---

## 3. The two illustration modes

`--illustrations {none,link,download}`, default `none` (existing behavior
unchanged). Both non-`none` modes place an `image-gallery.html` include at the
point in the text where the figure appeared. Neither writes raw `<img>` markup.

**link** — nothing downloaded, nothing committed:

```liquid
{% include essay/feature/image-gallery.html
   objectid="https://www.gutenberg.org/cache/epub/25290/images/image002.png"
   alt="The Origin of Outline." caption="The Origin of Outline" %}
```

**download** — files into `objects/<objectid>.<ext>`, a row per figure in
`_data/<slug>-metadata.csv`, `_config.yml` repointed:

```liquid
{% include essay/feature/image-gallery.html objectid="25290_f002" %}
```

`--collection-images` remains as an alias for `--illustrations download`.
`--all-images` still downloads without collection integration, now emitting
plain Markdown references to working `/objects/` paths.

In Actions, the same choice appears as an `illustrations` dropdown below the
*Clear existing* and *Generate about page* checkboxes.

Full user-facing documentation is in
`docs/cb-essay/gutenberg-extraction.md`, section **"Illustrations in CB-Essay"**.

---

## 4. Verification

Nine-book corpus, chosen for structural variety:

| Book | Sections | Expected | Text kept |
|---|---|---|---|
| Line and Form (25290) | preface + illustrations + 10 ch + index | ✅ | 97.0% |
| Pride and Prejudice (1342) | preface + illustrations + 61 ch | ✅ 61 | 99.5% |
| Moby Dick (2701) | 138 (etymology, extracts, 135 ch, epilogue) | ✅ | 99.5% |
| A Christmas Carol (46) | preface + illustrations + 5 staves | ✅ | 99.7% |
| The Yellow Wallpaper (1952) | 1 (whole-document fallback) | ✅ no chapters | 100% |
| Grimms' Fairy Tales (2591) | 62 tales | ✅ | 100% |
| Frankenstein (84) | 4 letters + 24 ch | ✅ | 99.9% |
| Alice (11) | 12 ch | ✅ | 99.4% |
| The Great Gatsby (64317) | 9 ch | ✅ | 99.9% |

"Text kept" = words in the output vs. words in the de-boilerplated source. Line
and Form's remaining 3% is entirely editorial apparatus — page numbers (stripped
by design), the contents list (replaced by CB-Essay's nav), title page,
transcriber's notes. No book prose is missing.

Beyond counts:

- **Tables through the real engine.** All emitted tables rendered with this
  project's kramdown 2.5.2: 5 tables, 286 rows, every Markdown row mapping 1:1 to
  a `<tr>`, no pipe leakage.
- **link mode, full Jekyll build.** 167 includes placed, build exit 0, 167
  `<img>` rendered with alt text, zero leftover Liquid.
- **download mode, full Jekyll build** (reusing 24 already-downloaded images): 23
  figures rendered from `/objects/`, 24 item pages generated, 24 browse
  thumbnails, each figure caption linking to its item page.

That last build caught a real bug: `image-gallery.html` only emits an `<img>`
when `image_small` or `image_thumb` is set — otherwise it draws a placeholder
icon. The CSV had neither, so download mode rendered 24 grey boxes. Both columns
now point at the object file.

Default-mode regression check across all nine books: byte-identical output,
except P&P losing 60 broken drop-cap links that pointed at paths which never
existed.

---

## 4a. The extraction report

Every run writes `extraction-results.md` to the project root, overwriting the
previous one — including runs that fail, which is when it earns its keep. The
workflow prints it to the Actions log and commits it alongside the content.

Two lines carry most of the diagnostic weight:

- **How the table of contents was found.** `container` and `heading` are the
  reliable paths; `NOT FOUND — fell back to scanning the whole document` means
  chapter boundaries came from heading text alone and deserve a look.
- **Text kept**, as a percentage of the de-boilerplated source. Below ~95% on a
  book without heavy apparatus means content is going missing somewhere. This is
  the number that exposed the dropped tables (Line and Form sat at 92.8%) and it
  reads 0% on a run that failed outright.

The report also lists figures catalogued but never placed, images that failed to
download, sections that had a heading but no body text, and whether a cover was
found — each as an actionable line under **Needs a human**, or a single line
saying nothing was flagged.

Word counts ignore Liquid tags and image references, so the includes the script
generates don't inflate retention.

---

## 5. Next steps

### Blocking

1. **Commit and push to `main`.** `workflow_dispatch` inputs only appear on
   github.com once the workflow file is on the default branch, so the
   `illustrations` dropdown won't show until then.

2. **One full `download` run against live Gutenberg has never completed.** Every
   piece is verified — figure detection, CSV, config pointer, includes, item
   pages, Jekyll build — but the 168-image download was interrupted at 24 and the
   rest of that mode was validated with those files rather than a fresh fetch.
   Test with book 46 (10 images) before Line and Form (168).

### Watch for

3. **Rate limiting at scale.** Downloading 168 images was visibly throttled
   (503/504, ~25s per miss before the retry policy was lightened). Untested from
   an Actions runner, which has a different IP and a cold rate-limit budget.
   `download_figures()` uses a 0.2s delay and skips images it cannot fetch rather
   than failing the run — worth watching the first real run's warning count.

4. **`objects/` in `.gitignore`** was removed by hand this session. The workflow
   commits `objects/` in download mode and needs it to stay removed.

5. **Thumbnails.** `image_small` and `image_thumb` point at the full object file.
   Fine for Gutenberg's ~45 KB illustrations, but
   `bundle exec rake generate_derivatives` should replace them for a real
   publication. Note it needs ImageMagick, which an Actions runner would have to
   install.

### Possible later

6. **Hi-res variants go unused.** Line and Form links 46 higher-resolution
   images (`image002h.png`, ~450 KB vs ~130 KB). These are captured in the figure
   model but nothing consumes them; they would make the Spotlight full-screen
   viewer worth opening.

7. **Tables in cells and TOC tables.** Nested tables fold into the parent cell as
   text, and TOC tables are dropped by design. Neither has bitten yet.

8. **`AGENTS.md`** doesn't mention `--illustrations` or the generated metadata
   CSV. Worth a line if this becomes the standard way to bring a book in, since
   the generated CSV changes what `site.metadata` points at.

### Working practice

Cache book HTML locally and test with `--local-html` rather than re-fetching —
this session leaned on gutenberg.org hard enough to get throttled. For
image-pipeline work, validate against a book with a handful of images rather than
a heavily illustrated one.

---

## 6. File map

| Path | Role |
|---|---|
| `gutenberg-extraction.py` | The extraction script |
| `.github/workflows/extract-gutenberg.yml` | Actions workflow, `illustrations` dropdown |
| `docs/cb-essay/gutenberg-extraction.md` | Gutenberg patterns reference + illustration modes |
| `_data/<slug>-metadata.csv` | Generated in download mode; `_config.yml` `metadata:` points here |
| `objects/<objectid>.<ext>` | Downloaded figures, named by objectid |
| `extraction-results.md` | Per-run report; regenerated (overwritten) every run |
