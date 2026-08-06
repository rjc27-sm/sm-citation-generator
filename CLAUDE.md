# AIHW Citation Generator – Project Memory

## What this project is
A web app that generates **AIHW** author–date citations. The AIHW referencing style is based on the Australian Government Style Manual (AGSM) author–date system, with AIHW-specific variations (see "AIHW-specific rules" below). It has three tabs:
- **Quick look up** – paste a DOI, ISBN, or URL; the app fetches metadata and generates a citation automatically
- **Manual entry** – select a source type and fill in a form
- **Link titles** – for EndNote users: paste a reference list output by EndNote's 'Australian Government author–date' style and the tool hyperlinks each title with its trailing URL or DOI (which EndNote can't do natively)

The app is hosted on GitHub Pages from the `master` branch; pushing to `master` deploys immediately.

> **History:** This was formerly the "Style Manual citation generator". That Style Manual version now lives in the Proof Positive tool (Style Manual Check folder). This version has been rebranded to the **AIHW citation generator** and customised for AIHW referencing style.

## AIHW-specific rules (differ from Style Manual)
- **Reference list authors:** list the first **6 authors**, then `et al.` for the rest. Style Manual lists all authors. Encoded in `formatAuthorList()`. Applies to all source types (including books).
- **METEOR items** are a dedicated source type (`meteor`), always AIHW-authored.
- **Changed agency names:** cite the department/organisation name as it appeared on the source at publication time (both author and publisher positions), not the current name. This is guidance only (not auto-detectable); covered by a help FAQ entry.

---

## Tech stack
- Pure HTML/CSS/JS — no build step, no frameworks, no dependencies
- Two files: `index.html` (the app) and `help.html` (FAQ page)
- Deploy: `git push origin master` → live on GitHub Pages
- `.claude/launch.json` is a local preview dev-server config, gitignored — the app is static and needs no server to run (opening `index.html` via `file://` works fine)

---

## File structure (sections within `index.html`)

| Section | What's in it |
|---|---|
| `<style>` | All CSS, including responsive breakpoints |
| HTML body | Header (with Help link), tabs, source type grid, form container, result area, footer |
| `CONFIGURATION & STATE` | `state` object, `authorComponents`, `formState`, `TYPE_DISPLAY_NAMES` |
| `SOURCE TYPE DEFINITIONS` | `SOURCE_FIELDS` object — field config for each type |
| `UTILITY FUNCTIONS` | Author formatting, date helpers, `toSentenceCase`, `linkTitle`, `resetAll`, etc. |
| `AUSTRALIAN GOVERNMENT BODY LOOKUP` | `GOV_AU_ORGS` map + `lookupGovAuOrg()` |
| `API FUNCTIONS` | CrossRef, Open Library, Google Books, URL page-title fetch |
| `CITATION FORMATTERS` | One function per source type, plus `formatCitation()` dispatch |
| `AUTHOR COMPONENT` | `initAuthorComponent`, `renderAuthorComponent`, `attachAuthorComponentEvents`, `getAuthorData` |
| `FORM BUILDING` | `buildForm`, `buildFieldHtml`, `wrapHalfRow` |
| `DATA COLLECTION` | `collectFormData`, `mapLookupToFormData`, `saveFormState`, `seedFormState` |
| `DISPLAY & COPY` | `displayResult`, `copyPlainText`, `copyRichText` |
| `SMART PASTE HANDLER` | `handleSmartPaste` — routes DOI/ISBN/URL to correct API |
| `LINK TITLES` | Everything behind the Link titles tab — Word HTML cleanup, URL/DOI extraction, title detection and linking (see 'Link titles tab' below) |
| `EVENT LISTENERS` | All wired up in a single `DOMContentLoaded` block at the bottom |

---

## Source types

Each type has:
1. A key string (used everywhere as the internal ID)
2. A `SOURCE_FIELDS` array entry
3. A formatter function
4. A `case` in the `formatCitation()` switch

| Key | Button label | Formatter | Title style | Date in parens | Notes |
|---|---|---|---|---|---|
| `journal` | Journal article | `formatJournalArticle` | single quotes | year | |
| `book` | Book | `formatBook` | italics | year | |
| `chapter` | Book chapter | `formatBookChapter` | single quotes | year | Has editor component |
| `website` | Webpage | `formatWebsite` | italics (standalone) or single quotes (in series) | year | accessed date; optional `series` field switches to 'webpage in larger publication' format; supports **archived sources** (Trove/Wayback) – see below |
| `report` | Govt report | `formatReport` | italics | year | accessed date; repeats agency short name; supports 'Unpublished' |
| `newspaper` | Newspaper | `formatNewspaper` | single quotes | full date | accessed date only if URL present |
| `dataset` | Data set | `formatDataset` | italics | year | `[data set]` outside link; accessed date; supports 'Unpublished' |
| `mediarelease` | Media release | `formatMediaRelease` | italics | full date | `[media release]` outside link; accessed date only if URL present |
| `conferencepaper` | Conference paper | `formatConferencePaper` | single quotes | full date | `[conference presentation]` descriptor; supports 'Unpublished' (no accessed date when unpublished) |
| `thesis` | Thesis | `formatThesis` | italics | year | `[type of thesis]` descriptor; accessed date; supports 'Unpublished' |
| `legislation` | Legislation | `formatLegislation` | roman (ref list), italics (in-text) | year from title | No author component; `(Jurisdiction)` suffix; year parsed from the title |
| `meteor` | METEOR item | `formatMeteor` | italics | year | Always AIHW-authored (org author pre-filled to AIHW); `(METEOR ID NNN)` outside link; fixed 'AIHW METEOR Metadata Online Registry website'; accessed date |

### Adding a new source type — checklist
1. Add a `SOURCE_FIELDS` entry
2. Write a `formatXxx(data)` function returning `{ html, plain, year, intext }`
3. Add a `case 'key': return formatXxx(data);` to the switch in `formatCitation()`
4. Add a `<button class="source-type-btn" data-type="key" ...>` to the source type grid in the HTML
5. If the type uses org authors, add its key to the `showOrgToggle` array in `buildForm()`
6. If it should default to org mode, add its key to the `defaultAuthorMode` check in `buildForm()`
7. If lookup can produce this type, add a case to `mapLookupToFormData()` and to `mapCrossRefType()`

---

## SOURCE_FIELDS field object properties

| Property | Type | Description |
|---|---|---|
| `id` | string | Field key; also becomes the HTML input ID suffix (`field-{type}-{id}`) |
| `label` | string | Label shown in the form |
| `type` | string | Omit for standard text input; `'checkbox'` renders a checkbox control; `'select'` renders a dropdown (requires `options`) |
| `options` | string[] | For `type: 'select'` only — the option values. An empty-string `''` option renders as a 'Select…' placeholder |
| `required` | boolean | Adds ` *` to label |
| `sentenceCase` | boolean | Shows the 'Convert to sentence case' button |
| `half` | boolean | Field takes half the row width (pairs with the next `half` field) |
| `placeholder` | string | Input placeholder |
| `helpText` | string | Small grey text below the input |
| `defaultValue` | string | Pre-fills the input if no saved value exists |

Special marker: `{ id: '_editors', label: '__editors__' }` — tells `buildForm` to insert the editor author component at that position (used in `chapter`).

**Checkbox fields:** `buildFieldHtml` detects `field.type === 'checkbox'` and renders a `<div class="checkbox-field">` containing the input and an inline label. `collectFormData` and `saveFormState` use `el.checked` (boolean) rather than `el.value` for checkbox elements.

**Select fields:** `buildFieldHtml` detects `field.type === 'select'` and renders a `<select>` from `field.options` (the `''` option becomes a 'Select…' placeholder; the option matching the prefill value is `selected`). `collectFormData`/`saveFormState` treat it like a text field (`el.value`). Currently used only by the `archiveName` field on `website`.

---

## Author component system

Two components can exist on a form: `ac-author` (always present) and `ac-editor` (chapters only).

State is stored in `authorComponents[containerId]`:
```
{
  mode: 'person' | 'organisation',
  persons: [{ given, family }, ...],
  orgAbbrev: '',
  orgFull: '',
}
```

**Key settings in `buildForm()`:**
- `showOrgToggle`: `['website', 'report', 'dataset', 'mediarelease', 'meteor']` — these types show the person/org toggle
- `defaultAuthorMode`: `'organisation'` for `report`, `dataset` and `meteor`; `'person'` for everything else
- **METEOR auto-fill:** for `meteor`, `buildForm()` pre-fills the org author to `AIHW` / `Australian Institute of Health and Welfare` when no org value is present. `seedFormState()` also skips seeding any author state into `meteor`, so another type's org (e.g. a report's `DVA`) can't leak in.

**Organisation author format:**
- With abbreviation: `ABBREV (Full Name)` in the reference list; `ABBREV` in-text
- Without abbreviation: `Full Name` everywhere

**Person author format:**
- Family name + initials, no comma, no full stops: `Kelleher T`
- Reference list (`formatAuthorList`): **AIHW rule — list the first 6 authors, then `et al.`** (7+ authors). For ≤6: commas between, `and` before last (never `&`). This differs from Style Manual, which lists all authors.
- In-text (`formatAuthorsInText`): 1 author → family name; 2 → `Family and Family`; 3+ → `Family et al.`

**Paste & parse:**
- The 'Paste author list' `<details>` accepts comma, semicolon, ampersand, or newline-separated names in natural or reversed order
- Strips affiliation numbers, ORCID URLs, and credential suffixes (MD, PhD, etc.)

---

## API lookup system (Quick lookup tab)

Input is classified by `detectInputType()` as `doi`, `isbn`, `url`, or `unknown`.

| Input type | API(s) used | Notes |
|---|---|---|
| DOI | CrossRef (`api.crossref.org`) | Returns structured metadata; maps to source type via `mapCrossRefType()` |
| ISBN | Open Library → Google Books (fallback) | Open Library tried first; Google Books used if not found |
| URL | `fetchPageTitle()` — three strategies (see below) | `suggestTypeFromURL()` picks source type; `lookupGovAuOrg()` tries to match a gov.au body |

After lookup, `mapLookupToFormData()` maps API response fields to form field IDs, and the 'Edit details' button lets the user correct the prefilled form.

### `fetchPageTitle(url)` — three-strategy title fetch

Returns `{ title, isHint }` or `null` if all strategies fail.

1. **allorigins proxy** — fetches raw HTML and checks `og:title`, `twitter:title`, `<h1>`, then `<title>` (strips trailing site-name suffixes from `<title>`). 8-second timeout.
2. **Jina AI reader** (`r.jina.ai`) — fully renders the page including JavaScript and parses `Title:` from the markdown response. Useful for JS-rendered pages blocked by strategy 1. 10-second timeout.
3. **URL slug heuristic** — last resort; infers a title from the URL path segments (skips pure numbers, very short segments, and 4-digit years). Sets `isHint: true` to flag the result for manual review.

When `isHint` is true, a 'Title inferred from URL — please verify.' help-text message is shown under the title field, and the status notice explains the inference. A 'Why didn't this work?' link points to the relevant FAQ anchor in `help.html`.

### URL source type special-casing in `suggestTypeFromURL()`

Certain hostnames are mapped to a specific source type before the generic `.gov.au` → `'website'` fallback:

| Hostname pattern | Mapped type | Reason |
|---|---|---|
| `aihw.gov.au` / `*.aihw.gov.au` | `report` | AIHW pages are Cloudflare-protected; title fetch always falls back to slug heuristic |
| `legislation.*` (any TLD) | `legislation` | Covers all nine state/territory/federal legislation registries (ACT, NSW, NT, QLD, SA, TAS, VIC, WA, federal) |

The legislation rule matches any hostname starting with `legislation.`, which covers all nine registries (`legislation.act.gov.au`, `legislation.nsw.gov.au`, `legislation.gov.au`, etc.).

### `stylemanual.gov.au` special-casing in `handleSmartPaste()`

Beyond `suggestTypeFromURL()` and `lookupGovAuOrg()`, there is an additional post-lookup override for Style Manual pages. After the standard `prefill` object is constructed, `handleSmartPaste()` checks whether the URL hostname is `stylemanual.gov.au` and the path is not the homepage (`/`). If so, it overrides:
- `prefill.websiteName` → `'stylemanual.gov.au'` (instead of the default `orgMatch.abbrev` value `'APSC'`)
- `prefill.series` → `'Australian Government style manual'`

`stylemanual.gov.au` is also in `GOV_AU_ORGS` so the org author (APSC / Australian Public Service Commission) is pre-filled via the normal lookup path.

---

## GOV_AU_ORGS lookup table

A hardcoded map of `hostname → { full: 'Full Name', abbrev: 'ACRONYM' }` sourced from the Australian Government Organisations Register (AGOR), December 2025.

`lookupGovAuOrg(url)` does:
1. Direct hostname match (strips `www.`)
2. Suffix match (e.g. `subdomain.abs.gov.au` → tries `abs.gov.au`)

Used during URL lookup to pre-fill the org author fields.

---

## Citation formatter conventions

All formatters return `{ html, plain, year, intext }`:
- `html` — HTML string with `<em>`, `<a>`, etc.
- `plain` — `html` with all tags stripped (for 'Copy plain text')
- `year` — four-digit year string, `'n.d.'`, or `'unpublished'` (report/dataset only)
- `intext` — two-item array: `['Author (Year)', '(Author Year)']`

**Key utilities used inside formatters:**

| Function | What it does |
|---|---|
| `linkTitle(titleHtml, data)` | Wraps `titleHtml` in `<a href>` if DOI or URL present; DOI takes priority |
| `formatWebsiteName(name)` | Appends ` website` if name doesn't already end with 'website' and isn't URL-style (no spaces + contains a dot — e.g. `aihw.gov.au`) |
| `formatDOI(doi)` | Returns the DOI as plain text in the form `doi:xxxxx` (not a link — the *title* carries the DOI hyperlink, via `linkTitle`) |
| `formatEdition(n)` | `2` → `2nd edn`, `3` → `3rd edn`, etc. |
| `formatPageRange(pages)` | Converts hyphens to en dashes in page ranges |
| `buildIntextOptions(authorStr, year)` | Returns the two in-text forms |
| `getTodayFormatted()` | Returns today as `D Month YYYY` |
| `autoFormatDate(value)` | Parses D/M/YY(YY) and reformats to `D Month YYYY`; returns original if unrecognised |

**`website` type — conditional title format:** `formatWebsite` checks `data.series`. If present, the title is rendered as `'<linked title>'` (single-quoted, linked) followed by `<em>series name</em>` — the AGSM 'webpage as part of a larger publication or series' rule. If absent, the title is `<em><linked title></em>` (italic) as normal.

**`[data set]` and `[media release]` descriptors** sit outside the `<a>` link tag — only the italic title text is hyperlinked:
```js
const linkedTitle = linkTitle(`<em>${data.title}</em>`, data);
parts.push(`${linkedTitle} [data set]`);
```

**Unpublished sources (`report` and `dataset`):** both formatters check `data.unpublished` first:
```js
const year = data.unpublished ? 'unpublished' : (data.year || 'n.d.');
```
This renders as e.g. `White N and Jackson D (unpublished) Report title, ...`

---

## Archived sources (Trove / Wayback Machine)

**Scope:** currently the **`website` type only.** Lets a user cite a page as captured by a web archive when the live original is gone. Output changes from:
`... Site Name, accessed D Month YYYY.`
to:
`... Site Name, archived D Month YYYY, accessed D Month YYYY via <Archive Name>.`

`archived <date>` is the **snapshot capture date** (from the archive URL's timestamp), **not** today. `accessed <date>` keeps its normal meaning (today/when completed).

**Form fields (in `SOURCE_FIELDS.website`, after `url`):**
- `isArchived` (checkbox) – 'This is an archived copy…'. Reveals the sub-fields.
- `archiveName` (`type: 'select'`, options `['', 'Trove', 'Wayback Machine', 'Other']`)
- `archiveNameOther` (text) – shown only when `archiveName === 'Other'`
- `archivedDate` (text) – snapshot date; **not** auto-filled with today (unlike `accessDate`)

**Show/hide:** `syncArchiveFields()` toggles the `formfield-website-*` wrapper divs based on the checkbox and dropdown. It is called at the end of `buildForm` (for `website`) and from the delegated `change` listener on `#manual-form` (which fires on the `isArchived` checkbox and `archiveName` select). This relies on `buildForm` giving every non-half field wrapper an `id="formfield-{type}-{id}"`.

**URL parsing – `parseArchiveUrl(url)`** returns `{ archiveName, archivedDate, originalUrl }` or `null`. Matches `/(awa|web)/YYYYMMDDhhmmss[modifiers]/<original>`:
- `/awa/` → Trove (`webarchive.nla.gov.au`); `/web/` → Wayback (`web.archive.org`), tolerating `id_`/`im_`-style modifiers.
- `archivedDate` via `archiveTimestampToDate()` (parses the 14-digit `YYYYMMDDhhmmss` → `D Month YYYY`).
- `originalUrl` is the live URL embedded after the timestamp.

**Manual entry:** a delegated `input` listener on `#manual-form` calls `handleArchiveUrlDetection()` when the webpage URL field changes — pre-fills `archivedDate`, ticks `isArchived`, and selects the archive name **only if the user hasn't already chosen one**.

**Quick lookup:** the URL branch of `handleSmartPaste()` calls `parseArchiveUrl(input)`. If it's an archive URL it: forces `suggestedType = 'website'`; runs `lookupGovAuOrg()` and the Style Manual override against the **`originalUrl`** (so the org/site resolves to the real site, not `web.archive.org`); adds `isArchived`/`archiveName`/`archivedDate` to the prefill; keeps the **archive URL** as `url` (the citation links to the snapshot); and fetches the title from the archive URL. A `archiveNotice` is appended to the status messages. Non-archive URLs are unaffected (`parseArchiveUrl` returns `null`).

**Output (`formatWebsite`):** only when `data.isArchived` — pushes `archived <archivedDate>` after the website name (guarded on `archivedDate` being present) and appends ` via <archiveName>` to the accessed element (`archiveNameOther` used when `archiveName === 'Other'`). Live-source output is byte-identical to before.

**Extending to other types (agreed, not yet built):** the natural candidates are the URL-bearing online types — **report, dataset, mediarelease, newspaper** (report highest-value). Out of scope: journal/book/chapter (DOI/print), legislation (own point-in-time versioning), meteor (AIHW's own registry). Extending means factoring the archive fields + the `archived/via` output out of `SOURCE_FIELDS.website`/`formatWebsite` into something shared, and relaxing the current 'archive ⇒ website' assumption in `handleSmartPaste` so an archived report URL maps to `report`.

---

## Link titles tab

Takes a reference list pasted from Word (EndNote 'Australian Government author–date' output) and hyperlinks each reference title, matching what the other two tabs produce.

**Pipeline:** `cleanWordHtml` → `splitHtmlBlocks` → `processReferenceHtml` (per reference) → `renderFixResults`.

`processReferenceHtml` does, in order:
1. Normalise italics — `ensureJournalNameItalic`, `ensureBookTitleItalic`, `ensureUnquotedTitleItalic` (these run even for references with no URL or DOI)
2. `extractTrailingUrl(text)` — last non-doi.org URL, or null
3. `extractDoi(text)` — **last** DOI anywhere in the text, not just at the end (EndNote can output `doi:… . https://…`, i.e. URL after DOI)
4. Remove the raw URL (`removeUrlFromBlock`); rewrite a doi.org URL to `doi:10.xxxx/yyy` (`rewriteDoiTextInBlock`)
5. `hyperlinkTitleInBlock` with the DOI href if there is one, else the URL

**URL vs DOI:** a raw URL is *removed* and folded into the title link; a DOI *stays visible* as plain `doi:10.xxxx/yyy` and the title links to `https://doi.org/<doi>`. Where a reference has both, the URL is removed and the DOI supplies the link — matching `linkTitle`'s DOI-over-URL priority in the citation formatters.

**DOI parsing (`DOI_RE` / `extractDoi`):** matches both `doi:10.xxxx/yyy` and `https://doi.org/…` (and `dx.doi.org`). DOIs contain full stops (`10.1017/gmh.2023.3`), so only punctuation at the very end of the match is stripped as the reference's closing full stop.

**Title detection (`findTitleSpan`)** returns `{ titleText, isQuoted, start, end }` where `start`/`end` are character offsets into the block's `textContent`. Callers use the offsets, never `indexOf(titleText)` — the title can appear more than once, and offsets are exact.

Two traps this design exists to avoid:
- **Possessive apostrophes.** In `'…consumers’ perspectives'` the `’` is a quote character followed by a space and looks exactly like a closing quote. The quoted branch therefore collects *all* candidate closing quotes and ranks them: quote-then-comma (the AGSM pattern) beats quote-then-full-stop/end, which beats quote-then-space.
- **Titles spanning multiple text nodes.** A title containing its own italics (`'Prevalence of <em>E. coli</em> in water'`) can never be found by an `indexOf` on a single text node. `offsetToPosition` + `wrapRangeInLink` use a DOM `Range` so the link can wrap across element boundaries.

`offsetToPosition(div, offset, preferNext)` — an offset on a text-node boundary is ambiguous (end of one node, start of the next). `preferNext: true` resolves to the following node, which is what the already-linked guard needs to see that a character actually sits inside an `<a>`. Without it, re-running the tool on its own output nests `<a>` tags.

**Quoted titles never fall through** to the `<em>` branch of `hyperlinkTitleInBlock`: for a quoted title the first `<em>` is the journal or book name, and linking the wrong text is worse than leaving it for the user (the reference is flagged amber instead).

---

## `toSentenceCase(title)`

- Lowercases all words except the first
- Preserves ALL-CAPS acronyms of 2+ characters (e.g. `UNESCO`, `ABS`)
- Does **not** capitalise after a colon — words following a colon are treated like any other mid-title word and lowercased
- Triggered by the 'Convert to sentence case' button (present on fields with `sentenceCase: true`)

---

## Form building & state persistence

- `buildForm(sourceType, prefillData)` — renders the author component(s) and all fields into `#manual-form`
- Field IDs follow the pattern `field-{sourceType}-{fieldId}` (e.g. `field-journal-title`)
- `wrapHalfRow(items)` — always used for half-field pairs (both 1-item and 2-item cases), wrapping each item in `<div class="form-field">` so label and input stay together within the grid cell
- `saveFormState(type)` — called before switching source types; snapshots all field values (using `el.checked` for checkboxes) and author state into `formState[type]`
- `seedFormState(fromType, toType)` — called after switching; if the new type has never been visited (`formState[toType]` is undefined), copies matching field IDs and author state from the previous type's saved state so the user doesn't lose common data (title, URL, access date, etc.) when switching to a new type for the first time
- `formState` persists in memory for the page session so switching types and back doesn't lose data

---

## Help page (`help.html`)

A separate FAQ page linked from the top-right corner of the app header. Organised into four accordion sections:
- **Quick look up** — covers URL title fetch behaviour (including the three-strategy approach and Cloudflare), DOI/ISBN lookup failures (`#url-title` anchor)
- **Manual entry** — author entry, org authors, changed agency names, archived sources (`#archived-sources` anchor), sentence case, n.d., date formats
- **Copying and using citations** — copy modes, in-text citations, Word/Google Docs pasting
- **Link titles** — what the Link titles tab is for (EndNote background) and how to use it (`#link-titles` anchor)

Inline contextual nudges in the app link directly to anchors within `help.html` (e.g. `help.html#url-title`) so users land on the relevant open section. The `<details>` element for the linked section must have the matching `id` attribute so the anchor resolves correctly.

**Not yet covered by the FAQ** (all shipped, none documented): the `unpublished` checkbox, METEOR items, and legislation.

When changing behaviour, check the FAQ for statements it makes false — the help page describes mechanics (how many title-fetch strategies, which types show the org toggle, what the DOI does) and drifts silently. Verify claims against the code rather than trusting them; a review in August 2026 found a stray `</li></ul>` closing a list that didn't exist, and an answer stating the DOI itself is hyperlinked when only the title is.

---

## Result area buttons

All four buttons share the `copy-btn` style (grey pill, consistent appearance):

| Button | ID | Action |
|---|---|---|
| Copy plain text | `#copy-plain` | Copies `state.lastResult.plain` (tags stripped) |
| Copy with formatting | `#copy-rich` | Uses `ClipboardItem` API to copy `text/html` + `text/plain`; falls back to plain if unavailable |
| Edit details | `#edit-details-btn` | Re-populates the manual form from the last lookup result |
| New citation | `#new-citation-btn` | Calls `resetAll()` — clears all state and starts fresh |

### `resetAll()`
Clears the lookup input, `state.lookupData`, `state.lastResult`, all `formState` entries, all `authorComponents` entries, resets the source type selector to `journal`, rebuilds a blank Journal article form, and hides the result area and any status/notice messages. Replaces the old footer instruction 'Refresh your browser to start a new citation.'

---

## Responsive layout

| Breakpoint | Source type grid | Other changes |
|---|---|---|
| Default (desktop) | 4 columns | — |
| ≤ 600px | 2 columns | Half-width form rows become full width; flex layouts stack |
| ≤ 400px | 2 columns (smaller padding/font) | Copy buttons stack |

---

## Style & language conventions

- **Punctuation:** en dashes (–) in all user-facing text; em dashes (—) may appear in code comments only
- **Quotation marks:** single quotes in all user-facing text (labels, help text, FAQ copy); double quotes in HTML attributes and JS strings only
- **'Data set'** is always two words in user-facing text (labels, citation output); the internal type key is `dataset` (one word)
- **'Look up'** is two words as a verb in user-facing text, and the first tab is labelled **Quick look up**. 'Lookup' (one word) is for code identifiers and internal prose only — `handleSmartPaste`, `lookupGovAuOrg`, `mapLookupToFormData`. Easy to get wrong when writing help copy
- **Date format:** `D Month YYYY` throughout (e.g. `4 January 2020`); no leading zero on day
- **Year fallback:** blank year fields display as `n.d.`
- **Accessed date:** auto-fills with today's date if the field is blank at generation time

---

## Things to watch out for

- `element.dataset` (the DOM API for reading `data-*` attributes) appears throughout the JS event handlers — this is completely unrelated to the `dataset` citation type
- The date auto-formatter is a delegated blur listener that checks `id.endsWith('-fullDate' | '-accessDate' | '-archivedDate')` — it applies automatically to any source type that has one of those fields
- `buildForm` gives every **non-half** field wrapper an `id="formfield-{type}-{id}"` (half-row wrappers from `wrapHalfRow` do **not** get ids). `syncArchiveFields` relies on these ids to show/hide the archive sub-fields
- `TYPE_DISPLAY_NAMES` is currently only populated for the original six types; it is used in status messages during lookup — update it if adding new types that can be reached via the lookup tab
- The `mapLookupToFormData()` switch only has cases for the original six types — the newer types (`dataset`, `mediarelease`, `conferencepaper`, `thesis`, `legislation`, `meteor`) cannot currently be reached via the Quick lookup tab
- The `unpublished` checkbox disables the year field via a delegated `change` listener on `#manual-form` — this is wired up in the `DOMContentLoaded` block, not inline in the field HTML
- Half-row field pairs must always use `wrapHalfRow()` — placing raw `buildFieldHtml` output directly into a `form-row` grid without a wrapper div causes the label and input to become separate grid cells
