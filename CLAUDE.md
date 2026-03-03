# SM Citation Generator – Project Memory

## What this project is
A single-file web app (`index.html`) that generates Australian Government Style Manual (AGSM) author–date citations. It has two tabs:
- **Quick lookup** – paste a DOI, ISBN, or URL; the app fetches metadata and generates a citation automatically
- **Manual entry** – select a source type and fill in a form

All logic, styles, and markup live in one file. The app is hosted on GitHub Pages from the `master` branch; pushing to `master` deploys immediately.

---

## Tech stack
- Pure HTML/CSS/JS — no build step, no frameworks, no dependencies
- Single file: `index.html`
- Deploy: `git push origin master` → live on GitHub Pages

---

## File structure (sections within `index.html`)

| Section | What's in it |
|---|---|
| `<style>` | All CSS, including responsive breakpoints |
| HTML body | Header, tabs, source type grid, form container, result area, footer |
| `CONFIGURATION & STATE` | `state` object, `authorComponents`, `formState`, `TYPE_DISPLAY_NAMES` |
| `SOURCE TYPE DEFINITIONS` | `SOURCE_FIELDS` object — field config for each type |
| `UTILITY FUNCTIONS` | Author formatting, date helpers, `toSentenceCase`, `linkTitle`, etc. |
| `AUSTRALIAN GOVERNMENT BODY LOOKUP` | `GOV_AU_ORGS` map + `lookupGovAuOrg()` |
| `API FUNCTIONS` | CrossRef, Open Library, Google Books, URL page-title fetch |
| `CITATION FORMATTERS` | One function per source type, plus `formatCitation()` dispatch |
| `AUTHOR COMPONENT` | `initAuthorComponent`, `renderAuthorComponent`, `attachAuthorComponentEvents`, `getAuthorData` |
| `FORM BUILDING` | `buildForm`, `buildFieldHtml`, `wrapHalfRow` |
| `DATA COLLECTION` | `collectFormData`, `mapLookupToFormData`, `saveFormState` |
| `DISPLAY & COPY` | `displayResult`, `copyPlainText`, `copyRichText` |
| `SMART PASTE HANDLER` | `handleSmartPaste` — routes DOI/ISBN/URL to correct API |
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
| `website` | Webpage | `formatWebsite` | italics | year | accessed date |
| `report` | Govt report | `formatReport` | italics | year | accessed date, repeats agency short name |
| `newspaper` | Newspaper | `formatNewspaper` | single quotes | full date | accessed date only if URL present |
| `dataset` | Data set | `formatDataset` | italics | year | `[data set]` outside link; accessed date |
| `mediarelease` | Media release | `formatMediaRelease` | italics | full date | `[media release]` outside link; accessed date only if URL present |

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
| `required` | boolean | Adds ` *` to label |
| `sentenceCase` | boolean | Shows the "Convert to sentence case" button |
| `half` | boolean | Field takes half the row width (pairs with the next `half` field) |
| `placeholder` | string | Input placeholder |
| `helpText` | string | Small grey text below the input |
| `defaultValue` | string | Pre-fills the input if no saved value exists |

Special marker: `{ id: '_editors', label: '__editors__' }` — tells `buildForm` to insert the editor author component at that position (used in `chapter`).

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
- `showOrgToggle`: `['website', 'report', 'dataset', 'mediarelease']` — these types show the person/org toggle
- `defaultAuthorMode`: `'organisation'` for `report` and `dataset`; `'person'` for everything else

**Organisation author format:**
- With abbreviation: `ABBREV (Full Name)` in the reference list; `ABBREV` in-text
- Without abbreviation: `Full Name` everywhere

**Person author format (AGSM):**
- Family name + initials, no comma, no full stops: `Kelleher T`
- Multiple authors: commas between, `and` before last (never `&`)
- In-text: 1 author → family name; 2 → `Family and Family`; 3+ → `Family et al.`

**Paste & parse:**
- The "Paste author list" `<details>` accepts comma, semicolon, ampersand, or newline-separated names in natural or reversed order
- Strips affiliation numbers, ORCID URLs, and credential suffixes (MD, PhD, etc.)

---

## API lookup system (Quick lookup tab)

Input is classified by `detectInputType()` as `doi`, `isbn`, `url`, or `unknown`.

| Input type | API(s) used | Notes |
|---|---|---|
| DOI | CrossRef (`api.crossref.org`) | Returns structured metadata; maps to source type via `mapCrossRefType()` |
| ISBN | Open Library → Google Books (fallback) | Open Library tried first; Google Books used if not found |
| URL | `fetchPageTitle()` via `allorigins.win` proxy | Grabs H1 or `<title>` from the page; `suggestTypeFromURL()` picks website vs newspaper; `lookupGovAuOrg()` tries to match a gov.au body |

After lookup, `mapLookupToFormData()` maps API response fields to form field IDs, and the "Edit details" button lets the user correct the prefilled form.

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
- `plain` — `html` with all tags stripped (for "Copy plain text")
- `year` — four-digit year string or `'n.d.'`, used for in-text citations
- `intext` — two-item array: `['Author (Year)', '(Author Year)']`

**Key utilities used inside formatters:**

| Function | What it does |
|---|---|
| `linkTitle(titleHtml, data)` | Wraps `titleHtml` in `<a href>` if DOI or URL present; DOI takes priority |
| `formatWebsiteName(name)` | Appends ` website` if name doesn't already end with "website" |
| `formatDOI(doi)` | Returns `doi:xxxxx` formatted as a hyperlink |
| `formatEdition(n)` | `2` → `2nd edn`, `3` → `3rd edn`, etc. |
| `formatPageRange(pages)` | Converts hyphens to en dashes in page ranges |
| `buildIntextOptions(authorStr, year)` | Returns the two in-text forms |
| `getTodayFormatted()` | Returns today as `D Month YYYY` |
| `autoFormatDate(value)` | Parses D/M/YY(YY) and reformats to `D Month YYYY`; returns original if unrecognised |

**`[data set]` and `[media release]` descriptors** sit outside the `<a>` link tag — only the italic title text is hyperlinked:
```js
const linkedTitle = linkTitle(`<em>${data.title}</em>`, data);
parts.push(`${linkedTitle} [data set]`);
```

---

## `toSentenceCase(title)`

- Lowercases all words except the first
- Preserves ALL-CAPS acronyms of 2+ characters (e.g. `UNESCO`, `ABS`)
- Does **not** capitalise after a colon — words following a colon are treated like any other mid-title word and lowercased
- Triggered by the "Convert to sentence case" button (present on fields with `sentenceCase: true`)

---

## Form building & state persistence

- `buildForm(sourceType, prefillData)` — renders the author component(s) and all fields into `#manual-form`
- Field IDs follow the pattern `field-{sourceType}-{fieldId}` (e.g. `field-journal-title`)
- `saveFormState(type)` — called before switching source types; snapshots all field values and author state into `formState[type]`
- `formState` persists in memory for the page session so switching types and back doesn't lose data

---

## Copy system

- **Copy plain text** — `state.lastResult.plain` (tags stripped)
- **Copy with formatting** — uses `ClipboardItem` API to copy `text/html` + `text/plain` simultaneously so italics paste into Word/Google Docs; falls back to plain text if `ClipboardItem` is unavailable

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
- **"Data set"** is always two words in user-facing text (labels, citation output); the internal type key is `dataset` (one word)
- **Date format:** `D Month YYYY` throughout (e.g. `4 January 2020`); no leading zero on day
- **Year fallback:** blank year fields display as `n.d.`
- **Accessed date:** auto-fills with today's date if the field is blank at generation time

---

## Things to watch out for

- `element.dataset` (the DOM API for reading `data-*` attributes) appears throughout the JS event handlers — this is completely unrelated to the `dataset` citation type
- The `fullDate` auto-formatter is a delegated blur listener that checks `id.endsWith('-fullDate')` — it applies automatically to any source type that has a `fullDate` field (currently `newspaper` and `mediarelease`)
- `TYPE_DISPLAY_NAMES` is currently only populated for the original six types; it is used in status messages during lookup — update it if adding new types that can be reached via the lookup tab
- The `mapLookupToFormData()` switch only has cases for the original six types — the two newer types (`dataset`, `mediarelease`) cannot currently be reached via the Quick lookup tab
