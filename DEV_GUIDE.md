# ذخیره خوارزمشاهی — Developer Guide (v2.0)

This document explains how `zakhire_reader_v2.0.html` was built, how its data
pipeline works, and how to regenerate or extend it. It assumes you're picking
this project up cold and need to understand *why* things are built the way
they are, not just what the code does.

## 1. What this actually is

A single self-contained HTML file (~6 MB) that renders a digitized, corrected,
and annotated reading experience for *Zakhireye Khwarazmshahi*, a 12th-century
Persian medical encyclopedia by Esma'il Jorjani. No server, no build step —
open the file in a browser and it works, including offline.

The source material was a 791-page OCR'd/exported PDF with badly broken text
encoding. Roughly 90% of the total effort in this project was **decoding that
PDF correctly**; the reader features on top were comparatively quick once the
text was trustworthy.

## 2. Source data pipeline (Python, offline, one-time)

All of this lives in `/home/claude/*.py` snippets run during development —
there's no single build script, so if you need to redo this from scratch,
follow this order. All intermediate JSON files are in `/home/claude/`.

### 2.1 PDF → raw text (`all_pages.json`)

The PDF's text layer is **not standard Unicode**. Two separate problems had
to be solved:

1. **Broken ToUnicode CMaps.** The embedded CID font's own ToUnicode table
   (meant to map character codes → Unicode) is corrupted — a known bug in
   old (~2008) MS Word Persian PDF exporters. Trusting it produces garbage.
   **Fix:** ignore ToUnicode entirely. Extract the font's *own* internal
   `cmap` table (via `fontTools`) from the embedded `FontFile2`, which is
   intact, and build a GID → Unicode map from that instead.
2. **Visual-order glyph storage.** Even with correct character values, glyphs
   are stored in left-to-right *visual* order, not logical reading order.
   **Fix:** reverse each text-run's character order after decoding. Runs are
   grouped into lines by Y-position, then ordered right-to-left by X-position
   to reconstruct correct RTL line layout.
3. Simple (non-CID) fonts — used for the Latin/footer text and Persian digit
   runs — need a **different** fix: their content-stream bytes are WinAnsi
   character codes, not glyph indices. Map byte → standard Adobe Glyph Name
   → this font's actual intended Unicode via its own cmap (not a naive
   byte-as-GID lookup, which silently corrupts text like "WordPress" →
   "PordPress").

Result: `all_pages.json`, a `[page][paragraph]` array of strings, lines
merged into paragraphs using vertical-gap heuristics between text runs.

### 2.2 Text artifact fixes → `pages_base.json`

Several further corrections, all regex-based, all **empirically verified
against the actual rendered PDF pages** before shipping (never assume a
fix is safe — check real examples first, including the *rare* ones, not
just the top-frequency ones):

- **ط/ظ mid-word space splits** (e.g. "ط بیعی" → "طبیعی"): a PDF-export
  quirk inserts a spurious space after these two letters mid-word. *But*
  some words genuinely end in ط or ظ followed by a real next word
  (توسط, فقط, غلیظ, حفظ, لفظ, وعظ, لحاظ, …) — those are in an explicit
  exclusion list and left alone. Get this list wrong and you silently merge
  real words like "توسط موسسه" → "توسطموسسه".
- **Space before punctuation** (`. ، ؟ :`) — straightforward strip.
- **Arabic yeh → Persian yeh** (ي U+064A → ی U+06CC) — global, safe.
- **Missing space after a period** when glued directly to the next word.

### 2.3 The manuscript-variant apparatus (critical!)

This is the single most important thing to understand about this text. The
critical edition compares **two manuscripts**, and for most of the running
narrative text (Books 1–3), nearly every word is annotated with both
readings inline: `۱WORD،۲WORD` (Persian digit 1 = manuscript A's reading,
digit 2 = manuscript B's, separated by an Arabic comma). Usually identical;
sometimes they genuinely differ.

- `DUAL_PATTERN = r'۱([^\s،]+)\s*،\s*۲([^\s،]+)'` — matches the common
  single-word case safely.
- A **multi-word greedy version was tried and abandoned** — it started
  silently deleting real connector words ("و مردم", "را نه شاید") when the
  variant spans crossed word boundaries in ways the regex couldn't safely
  bound. **Lesson: never let a "clean the text" regex be greedy past a
  single token if you can't prove the boundary.** Safety > completeness.
- After pairing extraction, a **safety-net pass**
  (`(?<![۰-۹])[۱۲](?=[آ-ی])`) strips any leftover lone marker digit still
  glued to a word, guaranteeing zero visible `۱`/`۲` artifacts in "no
  duplicate" mode — even for the messier multi-word/nested cases the pair
  regex can't fully resolve. Content is kept; only the marker digit is
  dropped, so worst case is a rare residual duplicated word, never a
  deleted one.
- Watch for **false-positive digit stripping** on legitimate multi-digit
  numbers (e.g. a footnote list entry "۱۲بسودنی" = "entry 12: بسودنی" had
  its "۲" wrongly stripped by an earlier, less careful version of this
  regex, corrupting "12" into "1"). The current regex requires the marker
  digit not be preceded by another digit.

### 2.4 The book is 3 concatenated volumes, not 1

Discovered by accident while checking why the dedup regex was corrupting
numbers on pages "within" what was assumed to be pure narrative — those
pages turned out to be an **embedded per-volume back-matter block**
(subject index + references + English abstract + numbered glossary),
repeated **three times** through the document, not once at the end as
initially assumed:

```
1–46      front matter
47–251    Book 1 + Book 2 (narrative)
252–281   Volume-1 back matter  ← embedded index/refs/abstract/glossary
282–375   Book 3 part 1 (narrative)
376–405   Volume-2 back matter  ← another one
406–604   Book 3 part 2 (narrative)
605–791   Volume-3 (master) back matter
```

`NON_NARRATIVE_RANGES = [(1,46),(252,281),(376,405),(605,791)]` is used
everywhere text transformations need to *not* apply (manuscript-dedup,
footnote-linking) since those blocks are index/citation-number-dense and
get corrupted by narrative-text regexes otherwise.

### 2.5 Footnote reference numbers → clickable links (`pages_with_fn.json`)

The two embedded "numbered glossary" back-matter blocks (within the
376–405 and 605–791 ranges specifically, at pages ~398–405 and ~782–791)
are **not** the subject index — they're sequentially-numbered footnote
definitions (`.N text`), and the small numbers scattered through the main
narrative text glued to specific words (e.g. `دق۲۰`) are references into
them.

Getting this right took two iterations:
1. First attempt only captured 1–2 digit markers → silently truncated
   3-digit references (entries go up to ~259), causing **wrong-but-
   plausible-looking** resolutions (a marker "27" truncated from "271"
   resolves to glossary entry 27, which is unrelated). This produced
   confidently wrong footnote popups — worse than no feature. **Verify
   resolution quality by manually reading random samples in context, not
   just by checking the "% resolved" number**, which stayed at 100% through
   this whole bug since truncated numbers still happened to match *some*
   entry.
2. Fixed by extending to 3 digits and validating that resolved numbers form
   a **monotonically increasing sequence** as you read through the book —
   this is what confirmed the numbers are genuine sequential footnotes
   and caught the truncation bug.

Order of operations matters: footnote-marker detection must run on text
that has **already had the manuscript-dedup applied**, not before — a
footnote reference to entry "1" or "2" is character-identical to the start
of a `۱X،۲Y` manuscript-variant marker, so running them in the wrong order
causes one system to corrupt the other. (`pages_nodup_clean.json` = dedup
only, no footnote handling yet → then footnote-detection runs on top →
`pages_nodup.json`.)

Resolved footnotes are embedded as `⟦N⟧` tokens (U+27E6/U+27E7, chosen
because they can't collide with anything in the source text) with a
separate `footnotes.json` id→definition lookup. Unresolved markers are
simply deleted (per explicit product decision: link if resolvable,
otherwise remove — never show a bare confusing number).

"Full" (دو نسخه) mode does **not** get footnote links — only a blind strip
of the marker digits — because linking them there would require running
footnote-detection on text that still has manuscript-dual-markers present,
recreating the exact conflict above. This was a deliberate scope cut, not
an oversight.

### 2.6 TOC / chapter titles (`toc_tree_final.json`)

Built by regex-detecting `^باب ... از (جزو ... از )?گفتار ...:` headings
in the (already fully cleaned) `pages_nodup.json`, filtered to exclude
front-matter TOC *listings* (which cram many "باب" mentions together) vs.
real chapter-opening paragraphs — the filter checks for a second "باب"
occurrence only in the **first 100 characters**, not the whole paragraph,
because legitimate chapter text sometimes contains an in-body cross-
reference to another chapter ("چنانکه اندر باب دوم…") later in the same
paragraph, which a wider window would wrongly flag as a crammed listing.

Title extraction (`make_title()`) has to cut the heading off before it
runs into the chapter's body prose. Watch for: this text's style often
**restates the heading's last word as the first word of the following
sentence** (e.g. "…تشریح جگر جگر عضوی است…" — genuinely correct in running
prose, but looks like a duplication bug when isolated as a title). The fix
detects that adjacent-repeat pattern and treats it as the true heading/body
boundary, rather than blindly trimming a trailing repeat (which only
catches the case where the repeat happens to land at the end of an
already-truncated string).

Book/volume boundaries (`BOOK_BOUNDARIES`) are **hand-set**, not detected,
based on the printed volume markers found during the investigation in
§2.4. If a more complete source PDF is ever substituted (see §7), these
almost certainly need re-deriving.

### 2.7 Glossary, bios, callouts, slide emoji (hand-curated)

- `glossary.json` (57 entries): classical medical/anatomical/pharmacological
  terminology with modern Persian equivalents, e.g. `ثفل→رسوب`. Scoped
  deliberately to **domain terminology only** — general archaic grammar
  (اندر→در, وی→او, etc.) was tried first, then explicitly removed per
  product feedback in favor of narrower medical-term focus.
- `bios.json` (5 entries): جالینوس، بقراط، رازی، ابن سینا، دیسقوریدوس —
  chosen by frequency count in the narrative text (only figures appearing
  20+ times got a bio).
- `callouts.json` (6 entries): hand-picked "what does modern medicine say"
  notes at specific (page, paragraph) anchors — four elements, four humors,
  innate heat, Galenic faculties, pulse theory, bloodletting. Kept small and
  curated rather than auto-generated, since getting medical-history
  commentary subtly wrong at scale is a real risk.
- Slide bullet emoji: keyword-regex rules (`EMOJI_RULES` in the slide-
  generation script) scanning each chapter title for topic keywords
  (قلب→🫀, خون→🩸, etc.), falling back to 📌. Purely cosmetic, safe to
  extend freely.

## 3. Runtime architecture (the HTML file itself)

Single file, built by string-substituting `__PLACEHOLDER__` tokens in
`shell_head2.html` (the template) with `json.dumps(...)` output for each
data file, via a small Python assembly script (see §6). All datasets are
embedded as `<script type="application/json" id="...">` blocks and parsed
once at load.

### 3.1 Why pagination is lazy (memory)

Early versions rendered the entire 791-page book into the DOM at once,
which was reported to crash on mobile. The fix, and the invariant to
preserve: **only the currently-viewed chapter's paragraphs exist as DOM
nodes at any time.** `renderPage()` clears `#pageContent` and rebuilds it
from scratch on every navigation. Data (`PAGES_FULL`, `PAGES_NODUP`) lives
as plain JS arrays in memory regardless — that's cheap; DOM nodes are not.

The same lazy principle applies to the TOC tree in the sidebar: branches
are only built into the DOM the first time the user expands into them
(`_ensureBuilt()` closure per node), not eagerly for all 477 entries.

### 3.2 Chapter-based loading (v2.0)

`renderPage(pageNum)` resolves `pageNum` to a **chapter range** via
`getChapterPageRange()` (built from the TOC's flattened باب list) and
renders the whole chapter at once, rather than a fixed N-page chunk. Falls
back to the old fixed-chunk behavior (`CHUNK` setting, 1/3/6 pages) only
for front-matter/back-matter pages that aren't part of any chapter. Chapter
ranges are clipped to never cross into a `NON_NARRATIVE_RANGES` block, even
if the next chapter's heading is on the far side of one (see §2.4 — this
was a real bug caught by checking actual page-length outliers, not just
spot-checking a few chapters).

Median chapter is 1 page; a couple of reference-listing chapters run to
30–40+ pages — those are fine, they only load on demand when the reader
navigates into them specifically.

### 3.3 Text-rendering pipeline per paragraph

`renderParaHtml(text, seenOnPage, pageNum, fnCounterState)`:
1. Split on the footnote token regex (`⟦N⟧`) first.
2. Footnote segments → `<sup class="fn-ref">` (numbered 1, 2, 3… **per
   page**, resetting each render) if `footnoteEnabled` and resolvable;
   silently dropped otherwise.
3. Non-footnote segments → `renderWordsHtml()`, which further splits on
   Persian-letter-run boundaries and checks each whole word against
   `GLOSSARY` (→ inline italic parenthetical, once per page via
   `seenOnPage` dedup, cumulative "learned words" tracked in
   `localStorage`) and `BIOS` (→ clickable dotted-underline span).

All of this only runs when `glossEnabled`/`bioEnabled` are on — when both
are off, it's a cheap `escapeHtml()` passthrough.

### 3.4 Feature inventory & where to find each in the code

| Feature | Toggle setting key | Key functions |
|---|---|---|
| Two text modes (دو نسخه / بدون تکرار) | `textMode` | `applyTextMode()`, swaps `PAGES` between `PAGES_FULL`/`PAGES_NODUP` |
| Glossary annotations | `glossEnabled` | `renderWordsHtml()`, `GLOSSARY` |
| Person bio popovers | `bioEnabled` | `showBio()`, `BIOS` |
| Footnote links | `footnoteEnabled` | `renderParaHtml()`, `showFootnote()`, `FOOTNOTES` |
| "Then vs now" callouts | `calloutEnabled` | `buildCalloutBox()` equivalent, `CALLOUTS_MAP` |
| PPT-style section slides | `slideEnabled` | `buildSlideBox()`, `SLIDES_MAP` |
| Bookmarks | (always on) | `toggleBookmark()`, `renderBookmarksList()`, `localStorage['zkh_bookmarks']` |
| "Words I've learned" | (always on) | `recordLearnedWord()`, `renderWordsList()`, `localStorage['zkh_learnedWords']` |
| Surprise-me random chapter | (button, no toggle) | `ALL_BAB_LEAVES`, `surpriseBtn` handler |
| Search | (always on) | `buildSearchIndex()`, `runSearch()` — rebuilt on text-mode switch |
| TOC (adjustable depth) | `tocDepth` | `buildTocNode()` (lazy), `setTocDepth()` |
| Theme / font / spacing / brightness / contrast | various | straightforward CSS variable toggles |
| Chapter-based pagination | `chunk` (fallback only) | `getChapterPageRange()`, `renderPage()` |

All settings persist via `localStorage`, prefixed `zkh_`. "Reset to
defaults" (`resetSettings`) clears all of these keys by name — **if you add
a new toggle, add its key to that list too**, or reset won't actually reset
it (this has bitten this project before).

## 4. Testing approach

There's no formal test suite — verification throughout was:
1. `node --check` on the extracted `<script>` block after every change,
   to catch syntax errors (this project hit real duplicate-`const`
   crashes from overlapping edits more than once — **always run this
   before shipping**).
2. Playwright headless-browser smoke tests per feature, re-run as a full
   regression suite before every ship: navigation, search, TOC, both text
   modes, bookmarks, glossary, bios, callouts, slides, footnotes, brightness/
   contrast, reset-to-defaults, DOM node count sanity check.
3. Direct data-quality spot checks against the actual rendered PDF pages
   (`pdftoppm`) whenever a text-correctness claim needed verifying — don't
   trust that a regex fix is safe just because it compiles; render the
   before/after and read it, and always sample the *rare* cases, not just
   the top-frequency ones (this is where the ط/ظ exclusion lists and the
   footnote 2-vs-3-digit bug were actually found).

A recurring failure mode worth flagging explicitly: **test scripts clicking
`#menuBtn` or `#settingsBtn` to "open" a panel that a previous step already
opened via a side effect** (e.g. focusing the search box auto-opens the
sidebar) — the click then *closes* it instead, producing a false "element
not found" failure that looks like an app bug but isn't. Always check
`classList.contains('collapsed')`/`'show'` before clicking a toggle button
in a test.

## 5. Known limitations (honest list)

- **Books 4–10 are not in the source PDF at all.** The uploaded file only
  contains Books 1–3 plus back matter. The TOC and reader only cover what
  exists; this can't be fixed without a more complete source document.
- **Footnote-link resolution is not 100% accurate.** Spot-checking suggests
  a clear majority of the ~1000 resolved links are correct or contextually
  plausible, but a minority are likely mismatched due to residual digit-
  extraction imprecision inherited from the underlying PDF decode. There's
  no per-link confidence score.
- **"No duplicate" mode's word-dedup is not perfect** for the rarer multi-
  word/nested manuscript-variant cases — the safety-net digit-strip
  guarantees no visible `۱`/`۲` markers, but a handful of these cases may
  show a word twice with no digit rather than cleanly once.
- **TOC book/volume boundaries are hand-set**, not derived from a reliable
  structural marker — if you re-run the pipeline on a different edition or
  a corrected PDF, re-verify these.
- The glossary/bios/callouts lists are all intentionally small and curated,
  not exhaustive. Expect readers to find domain terms or figures that
  aren't covered.

## 6. Regenerating the file end-to-end

There's no single script; the working order (all files in `/home/claude/`,
referenced by relative name throughout this doc) is:

```
all_pages.json                          (raw PDF decode)
  → pages_base.json                     (§2.2 artifact fixes)
    → fn_glossary_1.json, fn_glossary_2.json   (§2.5, extracted from pages_base)
    → pages_with_fn.json                (§2.5 footnote tokens on FULL text)
      → pages_compact.json              (FULL mode: blind-strip remaining digits)
    → pages_nodup_clean.json            (§2.3 manuscript-dedup, no footnote handling)
      → pages_nodup.json                (§2.5 footnote-detection on clean text)
        → parsed_headings.json          (§2.6)
          → toc_tree_final.json
            → slides.json               (§2.7)
heading_keys.json                       (derived from parsed_headings.json)
glossary.json, bios.json, callouts.json (hand-curated, §2.7)
```

Final assembly: read `shell_head2.html` (the HTML/CSS/JS template with
`__PLACEHOLDER__` tokens), `json.dumps()` each data file with
`ensure_ascii=False, separators=(',',':')` for compactness, string-replace
into the template (escaping any literal `</script>` inside the JSON as
`<\/script>` first), write the result as the final single-file HTML.

Always run `node --check` on the extracted script block and the Playwright
regression suite (§4) before shipping a rebuilt file.

## 7. If a more complete source PDF becomes available

Books 4–10 (see §5) would need the *entire* §2 pipeline re-run against the
new file — none of the page-number-based constants (`NON_NARRATIVE_RANGES`,
`BOOK_BOUNDARIES`, footnote glossary page ranges) can be assumed to carry
over, since they're specific to this exact PDF's pagination. Budget real
time for the font/CMap decoding step (§2.1) to be re-verified too, even if
it's the "same" book — a different scan or export could use a different
embedded font subset.
