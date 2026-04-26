# MyBookmarks

A keyboard-first personal bookmark homepage that runs as a single local HTML file. No server, no account, no extension required. Your bookmarks are stored in `localStorage` and stay on your machine.

---

## Purpose

Most browser bookmark managers are designed for pointing and clicking. MyBookmarks is designed for people who prefer to keep their hands on the keyboard — particularly developers who maintain separate bookmarks for DEV, QA, and PROD environments and want to navigate between them quickly.

The page is meant to be set as your browser's homepage or new tab page. You open a tab, type a few characters, and hit Enter. That's it.

---

## How to Use

### Installation

1. Download `index.html` and save it somewhere permanent (e.g. `Documents/MyBookmarks/index.html`).
2. In Chrome: **Settings → On startup → Open a specific page** → enter the full file path, e.g.:
   ```
   file:///C:/Users/yourname/Documents/MyBookmarks/index.html
   ```
3. Open a new tab — you should see the bookmark homepage.

> **Note:** Because the file runs from `file://`, some browser security restrictions apply. Clipboard paste requires a user gesture (the "paste from clipboard" button). Favicons are loaded from Google and DuckDuckGo's public APIs, so an internet connection is needed for icons to appear.

---

### Adding Bookmarks

Press **Ctrl+Shift+A** (or click the `+` button) to open the Add Bookmark modal.

| Field | Description |
|---|---|
| Title | Display name shown in the list |
| URL | Full or bare domain (e.g. `github.com` — `https://` is added automatically) |
| Category | Freeform label shown as a badge (e.g. `Dev`, `Work`) |
| Environment | `PROD`, `QA`, or `DEV` — colors the left border of the row |
| Tags | Comma-separated, used by search |
| Icon | Auto-detected, manually picked from a grid, or disabled |

**Shortcuts for faster entry:**
- **Drag a browser tab** onto the drop zone to auto-fill the URL and guess the title from the hostname.
- **Paste from clipboard** button reads a URL from your clipboard and does the same.

### Editing & Deleting Bookmarks

Press **Ctrl+Shift+E** while a bookmark is highlighted (use ↑↓ to navigate) to open the edit modal. The same modal is used for editing and for new bookmarks. A red **Delete** button appears when editing an existing entry.

### Searching

Start typing anywhere on the page — the search bar focuses automatically.

- Each space-separated word must appear as an exact substring somewhere in the bookmark's title, URL, tags, category, or environment field.
- Words can match in any order: `"stack over"` matches `"Stack Overflow"`, `"over stack"` also matches.
- Typos do **not** match — `"stck"` does not match `"Stack"`.
- Matching characters are highlighted in the results.
- Results are ranked: matches at word boundaries (e.g. the start of a word) score higher than mid-word matches.

### Keyboard Reference

| Key | Action |
|---|---|
| `↑` / `↓` | Navigate through results |
| `↵` | Open highlighted bookmark |
| `Shift+↵` | Open in a new tab |
| `Esc` | Clear search |
| `Ctrl+Shift+A` | Add a new bookmark |
| `Ctrl+Shift+E` | Edit the highlighted bookmark |
| `PgDn` | Switch to Google search mode |
| `PgUp` | Switch back to bookmark search mode |

### Google Search Mode

Press **PgDn** to switch the search bar into Google mode. The placeholder changes and the mode indicator in the top bar updates. Pressing Enter sends your query to Google in the current tab. Press **PgUp** or **Esc** to return to bookmark search.

### Environment Color Coding

Bookmarks with an environment set display a colored left border and a badge:

| Environment | Color |
|---|---|
| PROD | Amber |
| QA | Purple |
| DEV | Blue |

This makes it easy to scan a list of same-named services across environments without reading closely.

---

## Backup & Restore

Your bookmarks are stored in `localStorage` under the key `mybookmarks_v1`. This survives browser cache clears but would be lost if:

- You delete your Chrome profile
- You reinstall Chrome without exporting your profile
- You clear "site data" for `file://` in Chrome's privacy settings

**To export a backup:** Click the ⚙ button → Export → "all bookmarks ↓". This downloads a dated `.json` file.

**To export only the current search results:** Run a search first, then click "current filter ↓".

**To restore from a backup:** Click ⚙ → Import → paste the JSON array → choose:
- **Merge** — adds new entries, skips any URLs that already exist
- **Replace all** — wipes existing bookmarks and loads the import

**Recommendation:** Export a backup weekly and save it to a cloud folder, USB drive, or private GitHub repository.

---

## Zoom

The page zoom level is saved per machine in localStorage under the key `mybookmarks_zoom`. Open ⚙ → Zoom to adjust it with a slider. Each device keeps its own independent zoom preference.

---

## How It Works — Technical Reference

### Architecture

MyBookmarks is a single self-contained HTML file. There is no build step, no dependencies, no server, and no network requests except for favicon loading. The entire application is:

```
index.html
  ├── <style>        — all CSS, organized by component
  ├── <body>         — static HTML structure (modals, search bar, hint bar)
  └── <script>       — all application logic
```

### Data Model

A bookmark is a plain JavaScript object with the following shape:

```json
{
  "title":    "My App",
  "url":      "https://myapp.company.com",
  "category": "App",
  "env":      "prod",
  "tags":     ["app", "admin"],
  "icon":     ""
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `title` | string | ✅ | Display name |
| `url` | string | ✅ | Must be a valid `https://` URL |
| `category` | string | — | Shown as a grey badge |
| `env` | `"prod"` \| `"qa"` \| `"dev"` | — | Controls row color |
| `tags` | string[] | — | Used by search |
| `icon` | string | — | Omit = auto favicon, `""` = no icon, URL string = explicit icon |

### Storage

| Key | Value | Purpose |
|---|---|---|
| `mybookmarks_v1` | JSON array of bookmark objects | All bookmark data |
| `mybookmarks_zoom` | Number (70–150) | Zoom percentage for this machine |

Both keys live in `localStorage` scoped to the `file://` origin. Because `localStorage` is per-origin and `file://` treats every file path as the same origin in Chrome, the data is accessible from any `file://` page on the same machine — but not shared across machines or browser profiles.

### Search Algorithm

Search is performed by `scoreSearch(haystack, query)`:

1. The query is split on whitespace into individual terms.
2. For each term, every occurrence in the haystack string is located using `String.indexOf()`.
3. A match at a word boundary (start of string, or preceded by space, dash, underscore, dot, or slash) scores double (`term.length * 2`) versus a mid-word match (`term.length`).
4. If any term is not found at all, the entire query returns `null` (no match).
5. The final score is the sum of best-match scores across all terms.

Results are sorted descending by score before rendering. This means boundary matches (e.g. searching `"git"` matching the start of `"GitHub"`) rank above mid-word matches.

The search runs across a combined string of: `title + tags + category + env + url`. Title matches are re-scored separately so that highlight indices correspond only to the title text shown in the row.

### URL Normalization

`normalizeUrl(raw)` is called everywhere a URL is used:

1. Trims whitespace and strips trailing slashes.
2. If the string already starts with `http://` or `https://`, validates it with `new URL()` and returns it unchanged, or returns `""` if invalid.
3. Otherwise, prepends `https://` and validates. Returns `""` if still invalid.

This means users can type `github.com` and the stored URL will be `https://github.com`.

### Favicon Loading

Each bookmark row renders a favicon `<img>` with:
- `src` set to `https://www.google.com/s2/favicons?domain={host}&sz=32`
- `data-url` set to the bookmark's full URL

After each render, `patchFavicons()` attaches an `error` event listener to every favicon image. If the Google service fails, the listener replaces `src` with DuckDuckGo's favicon API (`https://icons.duckduckgo.com/ip3/{host}.ico`). If that also fails, the image is hidden with `visibility: hidden` (preserving layout space).

### Rendering

`renderBookmarkList(items, query)` builds the results list:

1. Sets `filtered` and `activeIdx` state.
2. Maps each item to an HTML string via `buildBookmarkRow()`.
3. Sets `container.innerHTML` to the joined strings.
4. Calls `attachRowEventListeners()` to wire up click handlers and favicon patching.

The first row (`i === 0`) always starts with the `active` class, keeping the keyboard focus on the top result.

### Modal System

Both modals (add/edit bookmark and settings) share:
- A `.modal-overlay` div that covers the full viewport
- An `.open` class toggled by `openX()` / `closeX()` functions
- A mousedown-origin check to prevent accidental closes when the user selects text inside the modal and releases the mouse outside of it

The mousedown check works by recording whether `mousedown` fired on the backdrop itself (not the modal inner div). The `click` handler only closes the modal if that flag is true.

### Search Mode

A `searchMode` variable tracks whether the page is in `'bookmarks'` or `'google'` mode.

- In **bookmarks** mode: typing runs `runSearch()` live and Enter opens the highlighted bookmark.
- In **google** mode: typing does not affect the bookmark list. Enter opens `https://www.google.com/search?q={query}` in the current tab.

`PgDn` switches to Google mode. `PgUp` or `Esc` switches back.

### Zoom

`applyZoom(percent)` sets the CSS custom property `--zoom` on `document.documentElement`, which is picked up by `html { zoom: var(--zoom); }`. The value is also saved to `localStorage` immediately.

On boot, `initZoom()` reads the saved value (defaulting to 100%) and applies it before the first render, preventing a zoom flash.

---

## Sharing & Privacy

This file can be shared publicly on GitHub — it contains no credentials, no personal data, and no hardcoded bookmarks (those are stored only in your local browser). Anyone who downloads the file starts with the seed bookmarks and builds their own list from scratch.

The only network requests made by the page are:
- Google Fonts (for typography, via `@import`)
- Google Favicon API (`www.google.com/s2/favicons`)
- DuckDuckGo Favicon API (`icons.duckduckgo.com`)

None of these receive any information about your bookmarks or search queries.

---

## Seed Bookmarks

On first launch (empty localStorage), the page populates with a small set of example bookmarks defined in `SEED_BOOKMARKS` at the top of the script block. Edit this array directly in the HTML file to change what new users see. After first launch, the seed is ignored — all data comes from localStorage.
