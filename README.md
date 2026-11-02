# Leslie Project Dashboard (`lesliedashboard`)

A personal, single-user productivity dashboard. **One static file (`index.html`), no build step, no dependencies, no backend.** It is hosted on GitHub Pages, and all of its data lives in a **separate private GitHub repo** that the page reads and writes directly through the GitHub REST API from the browser.

> This README is written so that another AI (or a new developer) can understand the whole repo immediately. Line numbers below are approximate and refer to the version that added the Biocomputing tab (Nov 2, 2026, `index.html` ≈ 327 lines / 48 KB). Search by the function names instead if they drift.

---

## 0. Quick start for an AI editing this repo

Read this section first. Everything else is reference.

1. **The whole app is `index.html`.** HTML + one `<style>` + one `<script>`. There is nothing else to build, bundle, or install.
2. **State lives in one object `S`.** Collections of items, each with a numeric `id` and a sync timestamp `u`. `S` is saved to `localStorage["lpd"]` and synced to `leslie-data.json` in the private data repo.
3. **The UI is string templates.** Each tab is a function in the object `V` returning an HTML string. `render()` replaces `#view.innerHTML` wholesale. There is no virtual DOM.
4. **All interaction goes through three global listeners:** a `click` listener (reads `data-tab` / `data-a` attributes), a `submit` listener (reads `data-f`), and a `change` listener. Add behavior by adding a `data-a`/`data-f` value and a branch, not by attaching listeners to elements (they are destroyed on every render).
5. **Anything you add to `S` must be registered in `COLS`** (and initialised in the `S` literal) or it will not sync, will not be timestamped, and deletions will not propagate.
6. **Always escape user text with `esc()`** when building HTML.
7. **Do not put `data-a` on a file `<input>`.** The global click handler would call `render()` and destroy the input before the file dialog returns. Use another attribute (this repo uses `data-att`, `data-show`).
8. **Delete items through the `data-a="del" data-k="<collection>"` path** so that the tombstone in `S.dead` is created by `stamp()`.
9. **There is no test suite in the repo.** See [section 14](#14-testing) for how the app was tested (jsdom harness) and what to stub.
10. **Never commit a GitHub token.** It is stored only in the browser's `localStorage["lpd-gh"]`.

---

## 1. Repository layout

```
lesliedashboard/
├── index.html     The entire application (markup, CSS, JS)
├── README.md      This file (architecture + conventions, for AIs/devs)
└── README.txt     Human setup guide (GitHub Pages, private data repo, token, connecting devices)
```

There are **two GitHub repos** involved. Do not confuse them:

| Repo | Visibility | Contains | Role |
|---|---|---|---|
| **Website repo** (this one, e.g. `leslie-dashboard`) | Public is fine | `index.html`, READMEs. **No user data.** | Served by GitHub Pages |
| **Data repo** (per the owner, `napatkitiprasanpong/file`; the name is configurable, README.txt suggests `leslie-data`) | **Private** | `leslie-data.json`, PDFs, pictures | Database + file storage |

The data repo is whatever the user types into *Dashboard → GitHub sync → Repo* on each device (`localStorage["lpd-gh"]`). Nothing in the code hard-codes its name.

### Data repo layout (written by the app)

```
<data repo>/
├── leslie-data.json                         The whole synced state (JSON of S)
├── files/<id>.pdf                           PDFs from the File tab (and Biocomputing PDFs saved by older builds)
└── biocomputing/
    ├── researchers/<id>_<name-slug>.jpg     Researcher photos (240x240 JPEG)
    └── papers/
        ├── <id>_<title-slug>.jpg            Paper pictures (max 480x360 JPEG, aspect preserved)
        └── <id>_<filename-slug>.pdf         Paper PDFs (max 25 MB)
```

`<id>` is the item id with non-word characters replaced by `_` (e.g. `1793620529545_8254`). Slugs are ASCII-only (`[^\w-]+` → `-`, max 40 chars); non-Latin names simply produce an empty slug.

---

## 2. Tech stack and conventions

- **Vanilla JavaScript (ES2020)**, no framework, no modules, no transpiler, no bundler, no npm.
- **No external network requests** except `https://api.github.com/repos/<repo>/...`. No CDNs, no fonts, no analytics. (The GitHub Pages site has no CSP, but keep it dependency-free.)
- **Code style:** very compact, dense one-liners; HTML built with template literals; short names (`S`, `V`, `H`, `GH`, `$`, `esc`, `iso`). Keep that style when editing so diffs stay small. Do not reformat the file.
- **IDs:** `id()` returns `Date.now()+Math.random()` (a float such as `1793620529545.8254`). Ids are rendered into `data-i="${x.id}"` and parsed back with `parseFloat` (round-trips exactly). Ids are compared as numbers in `find`, but used as **strings** for sync maps and file paths (`String(id)`).
- **Dates:** `YYYY-MM-DD` strings (`iso(date)`); `today` is computed **once at load** (`const today=iso(new Date())`), so a tab left open overnight keeps the old "today" until reload.
- **Text escaping:** `esc()` for every user-supplied string put in HTML/attributes.

---

## 3. Application state (`S`)

```js
const S={tasks:[],events:[],money:[],food:[],projects:[],folders:[],files:[],
         researchers:[],papers:[],scholarships:[],dead:{},gu:0,goal:2000};
```

Loaded at startup with `Object.assign(S, JSON.parse(localStorage["lpd"]||"{}"))`, then every item is given `u=1` if it lacks a timestamp.

`COLS` lists the synced collections:

```js
const COLS=["tasks","events","money","food","projects","folders","files","researchers","papers","scholarships"];
```

Non-collection keys: `S.goal` (daily kcal goal) with its own timestamp `S.gu`; `S.dead` (tombstones, `{ "<id>": deletedAtMs }`).

### Item shapes

Every item also has `id` (float) and `u` (last-modified ms, managed by `stamp()`; **never set it by hand**).

| Collection | Fields |
|---|---|
| `tasks` | `t` title, `due` `YYYY-MM-DD` or `""`/undefined, `done` bool, `pid?` project id |
| `events` | `t`, `date`, `time` `"HH:MM"` or `""`, `allday` bool |
| `money` | `type` `"in"`/`"out"`, `n` description, `amt` number, `date` |
| `food` | `n`, `each` kcal per serving, `amt` servings, `kcal` = round(each·amt), `date` |
| `projects` | `title`, `owner`, `name`, `url` (GitHub repo link), `img` data-URL thumbnail (480×270 JPEG, stored **inside** the JSON) |
| `folders` | `name`, `parent` folder id or `null` |
| `files` | `name`, `size`, `folder` id or `null`, `gh` `1` if the PDF is in the data repo (else it is only in this browser's IndexedDB) |
| `researchers` | `name`, `uni`, `contact` (free text; emails/URLs auto-linked), `why` (free text, shown on the card as "Why:"; may be missing on older items), `img` data-URL (only when GitHub not connected / legacy), `imgp` repo path of the photo |
| `papers` | `title`, `img`/`imgp` (as above), `pdf` = `null` or `{id,name,size,gh?,path?}` |
| `scholarships` | `n` name, `org`, `date` deadline or `""`, `req` requirements text |

`papers[].pdf` is a *descriptor* (its own `id`), not a `files` item. PDFs of papers are located with `findPdf(id)` which searches `S.files` and `S.papers[].pdf`.

---

## 4. Persistence and sync (the important part)

### 4.1 Layers

1. **Memory** — `S`.
2. **`localStorage["lpd"]`** — full copy of `S` (written by `locSave()` in a try/catch; quota errors are swallowed).
3. **GitHub data repo** — `leslie-data.json` (JSON of `S`), plus binary files (PDFs, JPEGs). This is the source of truth shared by phone and PC.
4. **IndexedDB `lpd-files` / object store `f`** — blobs: (a) PDFs when GitHub is *not* connected (key `String(id)`), (b) a read-through cache of repo images (key `"img:" + repoPath`).

### 4.2 Change tracking: `stamp()`, `H`, `rebuild()`

- `H` is a map `{ "<id>": JSON-hash-of-item-with-u=0, goal: <number> }`, rebuilt by `rebuild()` after loading/merging.
- `save()` = `stamp()` → `locSave()` → debounce **1500 ms** → `sync()`.
- `stamp()` compares each item to `H`; changed/new items get `u = now`; ids present in `H` but gone from `S` are recorded as tombstones `S.dead[id] = now`; a changed `goal` bumps `S.gu`.

So **any code path that mutates `S` must end with `save()`** (the global click handler does `save();render()` at its end for every `data-a` that does not `return` early; the submit handler does the same).

### 4.3 Merge: `merge(a, b)` (last-writer-wins **per item**)

- Tombstones: union, keeping the larger timestamp per id.
- For each collection in `COLS`: union by `String(id)`, keeping the item with the larger `u`; then drop any item whose tombstone time is `>= u` (a later edit resurrects an item; an older edit does not).
- `goal`: take the remote value if `b.gu > a.gu`.
- Keys not in `COLS` are copied from `a`.
- `norm(o)` produces a cheap fingerprint (`id:u` lists, goal, tombstones) used to decide whether anything changed.

### 4.4 `doSync(manual)` (queued through `sq`, so syncs never overlap)

1. If GitHub is not connected → `SS="GitHub not connected"`, return.
2. `ghRead()` fetches `leslie-data.json` (stores its blob `sha` in `ghSha`; 404 → `null`; falls back to a raw fetch when the Contents API omits `content` for files > 1 MB).
3. `merge(S, remote)`; if it changed local state: `Object.assign(S, merged)`, `rebuild()`, `locSave()`, and `render()` afterwards.
4. If the remote file is missing or differs from the merged result → `ghWrite(JSON.stringify(S))` (PUT with `sha`). **HTTP 409/422 (stale sha) schedules another sync after 1.5 s.**
5. On success: sets the status text (`#ss`), then calls `migrateImgs()` (see §7).
6. Any exception → `SS="Sync failed"` (no retry other than the next trigger).

### 4.5 When sync runs

`boot()` (waits for the first sync up to **8 s**, then removes the loading screen), 1.5 s after any `save()`, every **60 s** while the page is visible and connected, on `visibilitychange` (hidden → `flushNow()`, visible → `sync()`), on `pagehide`, on the "Sync now" button, and right after `ghConnect()`.

### 4.6 Binary files

- `ghPut(path, fileOrBlob)` — `PUT /contents/<path>` with base64 content. **Creates only; it never sends a `sha`, so it cannot overwrite.** This is why replacing a picture always uploads to a **new path** and then deletes the old one.
- `ghDel(path)` — `GET` the file's sha, then `DELETE` it (404 is ignored).
- `ghp(f)` — repo path of a PDF descriptor: `f.path || "files/" + String(f.id).replace(/\W/g,"_") + ".pdf"`. Old items have no `path` (they live under `files/`); new Biocomputing PDFs have `path = biocomputing/papers/...`.
- `getPdf(f)` — if `f.gh` fetch raw from the repo, else read from IndexedDB. `openPdf(id)` shows an in-page viewer (`.pv`, blob URL in an `<iframe>`, buttons *New tab / Download / Close*); `dlPdf(id)` triggers a download.
- Limits: **25 MB per PDF**; images are re-encoded to JPEG (quality 0.75) before upload.

### 4.7 GitHub API details

All calls go through `ghFetch(path, {method, q, body, raw, ok404})` → `https://api.github.com/repos/<repo><path>` with headers `Authorization: Bearer <token>`, `Accept: application/vnd.github+json` (or `application/vnd.github.raw+json` when `raw`), `X-GitHub-Api-Version: 2022-11-28`, `cache: "no-store"`. The branch is `cfg.branch || "main"`. Required token scope: **fine-grained PAT, only the data repo, Repository permission *Contents: Read and write***. `ghConnect()` validates by `GET /repos/<repo>` and gives specific messages for 401/404.

---

## 5. UI architecture

### 5.1 Shell

```
<canvas id="bg">        animated red "lava" background (downscaled 1/5, ~5 radial gradients; static if prefers-reduced-motion)
<div id="load">         full-screen spinner; removed by boot()
<aside id="nav">        sidebar (desktop) / bottom tab bar (<=760px); rebuilt by render()
<main id="view">        the active tab's HTML
<input id="pic" hidden> shared hidden image picker used by "Edit thumbnail / Change photo / Add picture"
```

### 5.2 Routing

`TABS = [["Dashboard","⌂"],["Task","✓"],["Calendar","▦"],["Event","★"],["Money","$"],["Calories","♥"],["Project","❖"],["File","▤"],["Biocomputing","⬡"]]`.
`tab` is the current name; it is mirrored in `location.hash` (`#Biocomputing`) and restored on load and on `hashchange` (only if the name is in `NAMES`). Clicking any element with `data-tab="<Name>"` navigates.

### 5.3 Rendering

`render(enter)` does: rebuild the nav (preserving/centering its horizontal scroll on mobile) → `#view.innerHTML = (tab==="Dashboard" ? "" : "<h2 class=pt>"+tab+"</h2>") + V[tab]()` → re-bind `#imp` / `#fup` inputs → if `enter`, stagger a CSS `up` animation over the cards/rows/forms → `hydrate()` (loads repo images into `<img data-ip>`).

Consequences you must remember:
- **Any `render()` wipes unsaved form input and scroll-local UI state.** A background sync that changes data also calls `render()`.
- Transient UI state lives in module-level `let` variables: `tab`, `cm` (calendar month), `sel` (selected day), `cwd` (File tab folder), `openP` (open project detail), `picFor`/`picK` (target of the image picker), `bsub` (Biocomputing sub-tab).

### 5.4 Global event model

**`click`** (document level, `e.target.closest("[data-tab],[data-a]")`):

| `data-a` | Meaning (`data-i` = id, `data-k` = key/collection) |
|---|---|
| `tg` | toggle task done |
| `del` | delete from collection `k` (folder must be empty; papers confirm and delete their PDF; papers/researchers delete their repo picture) |
| `day`, `mo`, `goto` | calendar day select / month prev-next / jump to a date |
| `ptask`, `pback` | open / close a project's task list |
| `cd` | change File-tab folder |
| `open`, `dl` | open / download a PDF by **pdf id** (File item or paper pdf) |
| `sync`, `ghoff` | sync now / disconnect GitHub |
| `pic` | open image picker; `k` = target collection (`projects` default, `researchers`, `papers`) |
| `nopic` | reset a project thumbnail |
| `exp` | export backup JSON (uses `window.AndroidApp.share` if present, else a download link) |
| `bsub` | switch Biocomputing sub-tab (`k` = `Researcher`/`Paper`/`Scholarship`) |
| `seed` | load the starter researcher list (`seedR()`) |

After handling, unless the branch `return`s early, the handler runs `save(); render();`.

**`submit`** (`data-f` on the `<form>`): `gh` (connect), `task`, `folder`, `event`, `money`, `food`, `goal`, `cal`, `proj`, `researcher`, `paper`, `schol`. `researcher` and `paper` are async (image resize + upload) and `return` their own promise (they call `save(); render()` themselves).

**`change`**: `ad` (all-day disables time), `nd` (no-deadline disables date), `mv` select (move a file to a folder), file inputs with `data-show` (show chosen filename in their label), file inputs with `data-att` (attach a PDF to an existing paper).

### 5.5 Design system (CSS)

- Dark glassmorphism: `--bg:#000; --glass:rgba(22,22,28,.62); --line:rgba(255,255,255,.09); --tx:#f5f5f7; --mut:#9a9aa6; --red:#e11d2e; --red2:#ff4757`.
- Reusable classes: `.card`, `.row` (flex list row), `.pc` (picture card) with `.th` (thumbnail; modifiers `.sq` square, `.pp` 4:3 with `object-fit:contain`), `.pb` (card body), `.pa` (card action row), `.grid` (auto-fill 230 px; `.grid.bio` becomes 2 columns on phones), `.st` (stat tile), `.g2`, `.btn` (primary), `.ghost` (secondary button/link), `.mut`, `.in` (green) / `.out` (red), `.empty`, `.foot`, `.bar`, `.ring`, `.cal`/`.d`, `.sub` (sub-tab pills), `.bf` (Biocomputing form), `.fl` (file-picker label), `.pre` (pre-wrap text), `a.lk` (links in contact text), `.pv`/`.pvh` (PDF viewer overlay).
- **Mobile (`max-width:760px`):** sidebar becomes a fixed bottom bar that scrolls horizontally (9 tabs); `main` has bottom padding for the bar + safe area; inputs are 16 px (prevents iOS focus-zoom); Biocomputing grids are 2-column; Biocomputing forms use full-width fields/buttons. `@media(hover:none)` disables hover lift effects on touch. `viewport-fit=cover` + `env(safe-area-inset-*)` are used for notches/home indicator.

---

## 6. Tabs (what each one does)

| Tab | Function in `V` | Notes |
|---|---|---|
| **Dashboard** | `V.Dashboard` | Greeting, stat tiles (open tasks, upcoming events, balance, projects), "Today" agenda (today's events + due/overdue tasks), calories ring (SVG circumference 327), 3 most recent projects, **GitHub sync card** (`ghCard()`), Export / Import backup. |
| **Task** | `V.Task` | Add (title, deadline or "No deadline"), toggle, delete. Sorted open-first then by due date. |
| **Calendar** | `V.Calendar` | Month grid; red dot = event, white dot = open task; add task/event to the selected day. |
| **Event** | `V.Event` | Sorted list; time/all-day; click date to jump to Calendar. |
| **Money** | `V.Money` | Income/expense ledger with totals. Amounts are plain floats; currency symbol is `$`. |
| **Calories** | `V.Calories` | Food log for **today**, servings × kcal each, daily goal (`S.goal`), progress bar. |
| **Project** | `V.Project` / `projDetail()` | Register GitHub repos (`owner/repo` or URL via `repoInfo()`), custom thumbnail, per-project task list (tasks carry `pid`). |
| **File** | `V.File` | Folder tree (`cwd`), upload PDFs (to `files/` in the data repo, or IndexedDB if not connected), move between folders, open/download/delete. |
| **Biocomputing** | `V.Biocomputing` | See §7. |

Backup: `Export backup` downloads `leslie-dashboard-backup.json` (the full `S`); `Import backup` merges a file into `S` with `merge()` (does not replace).

---

## 7. Biocomputing tab (added Nov 2026)

**Purpose:** the owner (a software engineer studying biomedical engineering in Thailand) is pursuing a career in biocomputing / **organoid intelligence** (living-neuron computing). The tab is a personal tracker for that: people to contact, papers to read, and funding to apply for.

Three sub-tabs (state `bsub`), each with an add form on top and a list below. If GitHub is not connected, a notice with a "Connect GitHub" button appears (files would stay on this device only).

| Sub-tab | Fields | Behaviour |
|---|---|---|
| **Researcher** | name (required), university, contact, why, photo | A **Load starter list (N)** button appears above the form while any of the 74 built-in researchers (`SEED`) are missing; it adds them with fixed ids `SEED_ID+index` (so devices never create duplicates) and skips any the user deleted (`S.dead`). Each seed has name, organization, why, and a source link in `contact`. Photo is cropped square (240×240, biased toward the top so faces are kept). Contact text is auto-linked: `https?://…`, `www.…`, and emails become links (`ct()`, applied **after** `esc()`). "Change photo" / "Add photo". |
| **Paper** | title (required), picture, PDF | Picture keeps aspect ratio (max 480×360, shown `contain`). PDF stored in the repo; **Open / Download** use the shared PDF viewer. If created without a PDF, an **Attach PDF** button appears. Deleting asks for confirmation and removes the PDF and picture from the repo. |
| **Scholarship** | name (required), organization, deadline date, requirements (multi-line) | Sorted: soonest upcoming first, then undated, then past (faded). Countdown text ("40 days left", red ≤ 14 days, "Due today/tomorrow", "Closed N days ago"). |

### Key functions

| Function | Role |
|---|---|
| `shrink(file, {w,h,fy,fit})` | Decode an image, draw to a canvas (cover-crop with vertical focus `fy`, or `fit` = scale down preserving aspect), return a JPEG data-URL. `PIC` holds per-collection sizes. |
| `putImg(dataUrl, kind, label, quiet)` | If GitHub connected: upload JPEG to `biocomputing/<kind>/<id>_<slug>.jpg`, cache it in IndexedDB, return `{img:"", imgp:path}`; else return `{img:dataUrl, imgp:""}`. Returns `null` on failure (after an alert unless `quiet`). |
| `putPdf(file)` | Validate (≤ 25 MB, `application/pdf`), upload to `biocomputing/papers/…pdf` (or IndexedDB if not connected); returns the descriptor `{id,name,size,gh?,path?}` or `null`. |
| `delImg(item)` | Delete an item's repo picture + its IndexedDB cache entry. |
| `imgUrl(path)` / `hydrate()` | Repo images are rendered as `<img data-ip="path">`; after each render `hydrate()` fetches each (IndexedDB cache first, then `ghFetch(...raw)`), creates a blob URL, and sets `src`. Promises are memoised in `IMGC`. Legacy data-URL images render directly via `src`. |
| `addResearcher` / `addPaper` / `attachPdf` | Async form handlers (disable the submit button, upload, push to `S`, `save(); render()`). If an upload fails nothing is added. |
| `migrateImgs()` | Called at the end of every successful sync (guarded by `mig`). Moves (a) data-URL pictures of researchers/papers into the repo, (b) papers' PDFs that exist only in local IndexedDB (added before GitHub was connected) into the repo, then clears the local copy. Silent on failure; retried next sync. Two devices migrating the same item at the same moment can leave an orphan file in the repo (harmless). |
| `SEED`, `seedMiss()`, `seedR()` | Starter researcher list (`[name, org, why, link]`), count of seeds not yet loaded or deleted, and the loader behind `data-a="seed"`. No photos are bundled; add them with "Add photo". |
| `ct`, `dleft`, `fdate`, `pst` | Contact linkifier, days-until, date formatter, "is past" predicate. |

Project thumbnails (Project tab) intentionally remain data-URLs inside the JSON (the original design); researchers/papers use repo files.

---

## 8. Storage rules at a glance

| Thing | GitHub connected | Not connected |
|---|---|---|
| Item data | `leslie-data.json` + `localStorage` | `localStorage` only |
| File-tab PDFs | `files/<id>.pdf` (`file.gh=1`) | IndexedDB (this device only) |
| Biocomputing PDFs | `biocomputing/papers/*.pdf` (`pdf.path`, `pdf.gh=1`) | IndexedDB |
| Biocomputing pictures | `biocomputing/{researchers,papers}/*.jpg` (`imgp`) | data-URL in the item (`img`) |
| Project thumbnails | data-URL in JSON | data-URL in `localStorage` |

---

## 9. Native-wrapper hooks (legacy / optional)

The page can be embedded in an Android WebView wrapper. These are all optional and guarded:

- `window.AndroidApp.share(filename, json)` — used by *Export backup* when present.
- `NAT` object — in the web build it is `{has, read, write}` bound to the GitHub functions. The code also supports optional `NAT.onChange(cb)`, `NAT.onFlush(cb)`, `NAT.flushed()` (called if a wrapper defines them) and has a dead helper `pickSync()` that calls a never-defined `NAT.pick()`; ignore/remove it.
- `window.flushNow()` — clears the debounce and syncs immediately (used on `visibilitychange`/`pagehide` and by wrappers).

---

## 10. Security model

- The page is public; **the data is private** (private repo + token). The site repo contains no user data.
- The token lives in plain text in `localStorage["lpd-gh"]` (`{repo, token, branch}`) on each device. Because of that, **XSS is the main risk**: always use `esc()`; only linkify via `ct()` (which escapes first and only creates `http(s)://`, `www.` and `mailto:` links); never inject unescaped repo/file names.
- Use a **fine-grained token limited to the data repo with Contents read/write** and an expiry (README.txt suggests 1 year). An expired/invalid token shows as "Sync failed".
- GitHub history is the undo/backup mechanism: every sync is a commit (`sync <ISO time>`), every file add/delete is a commit.

---

## 11. How to extend (recipes)

**Add a new synced collection `foo`:**
1. Add `"foo"` to `COLS` and `foo:[]` to the `S` literal.
2. Give items `{id:id(), ...}` — `u` is added by `stamp()`.
3. Create/modify the form (`data-f="foo"`), add a branch in the `submit` listener pushing to `S.foo`, and ensure it reaches `save();render()`.
4. Delete via `data-a="del" data-k="foo" data-i="${x.id}"`.
5. Old backups/remote data without `foo` are fine (`merge`/`norm` use `||[]`; the `S` default supplies `[]`).

**Add a new tab:** append `["Name","glyph"]` to `TABS`, add `V.Name(){ return \`...\`; }`, add any `data-a`/`data-f` branches. The hash route, sidebar button and mobile bar button are automatic. Check the mobile bar still fits (it scrolls).

**Add a field to an existing item:** just write it; `stamp()` detects any JSON change. Render with `esc()`.

**Store a new binary type:** upload with `ghPut(path, blob)` to a **new unique path** (never overwrite), store the *path* in the item, render with `<img data-ip>`/a fetch via `ghFetch(path,{q:br(GH()),raw:true})`, and delete with `ghDel(path)` when the item is deleted.

**Change the data schema:** make readers tolerant of missing fields (both devices may run different versions for a while) and migrate lazily, like `migrateImgs()`.

---

## 12. Known gotchas (learned the hard way)

- File `<input>` elements must not carry `data-a` (see §0.7).
- `render()` destroys DOM; keep drafts in variables or accept the loss. Don't call `render()` while a native file dialog is pending.
- `ghPut` can't overwrite → use fresh paths. `ghWrite` of `leslie-data.json` *does* use `sha` and retries on 409/422.
- Top-level `const`/`let` declarations are **not** properties of `window`; function declarations are. This matters for tests (use `window.eval("S")`).
- `S.dead` tombstones are never pruned; it grows slowly forever. Safe to prune entries older than the oldest device sync if ever needed.
- A very large `leslie-data.json` (> 1 MB) is fetched through the raw endpoint; keep images out of the JSON (that is why Biocomputing pictures moved to repo files).
- iOS Safari: PDFs in the in-page iframe may show only page 1 — use the **New tab** button. Android Chrome often cannot render PDFs in an iframe — use **New tab/Download**.
- Phone camera photos are decoded in memory before downscaling; very large images on low-memory phones may fail with "Could not read that image."
- `today` is fixed at page load.
- GitHub API rate limit (5,000 requests/hour per token) is far above normal use (one sync/minute), but `hydrate()` makes one request per not-yet-cached image per device.

---

## 13. Known limitations / ideas not yet built

- No edit forms (items are delete + re-add; only thumbnails/photos can be changed).
- No search/filter; Researcher list is in insertion order.
- Papers and scholarships have no URL field; scholarship deadlines are not mirrored into the Calendar tab (an "Add to Calendar" button would create an `events` item with `allday:true`).
- Project thumbnails are still base64 in the JSON.
- Merge granularity is per item (last write wins); two devices editing the *same* item offline lose one edit.
- No pagination or virtualisation (fine for hundreds of items).

---

## 14. Testing

There is no test suite in the repo. During development the app was verified with a **jsdom + fake-indexeddb harness** (Node 22). To reproduce:

1. `npm i jsdom fake-indexeddb`.
2. Load `index.html` with `runScripts:"dangerously"`, `url:"https://example.com/"`, `pretendToBeVisual:true`.
3. In `beforeParse` stub what jsdom lacks: `window.indexedDB`, `matchMedia`, `HTMLCanvasElement.prototype.getContext` (return an object with `drawImage`, `fillRect`, `createRadialGradient(){return{addColorStop(){}}}`), `toDataURL`, `URL.createObjectURL/revokeObjectURL`, a fake `Image` that fires `onload` after setting `src`, `alert`, `confirm`, and (for sync tests) a `fetch` that emulates `api.github.com/repos/<repo>/contents/...` (GET/PUT/DELETE, 404 for missing files, raw `blob()` responses). Pre-seed `localStorage["lpd-gh"]` to simulate a connected device.
4. Call app internals with `window.eval("...")` (e.g. `ev("addResearcher")(form, {name:…, photo: new File(...)})`, `ev("S.papers")`, `ev("merge")(a,b)`).
5. Things worth asserting: every tab renders; add/delete round-trips; paths written to the mock repo (`biocomputing/...`); the synced JSON contains paths not image data; delete removes repo files and creates tombstones; `merge` handles old-format data; HTML in user text is escaped; legacy data-URL images and local-only PDFs migrate on sync.
6. Caveats: `fake-indexeddb` cannot round-trip jsdom `File` objects (shim `ghPut` to accept non-Blobs when testing migration); layout/CSS and real phone behaviour are **not** covered by jsdom — check those in a real browser/phone.

---

## 15. Setup and operations (summary; full steps in `README.txt`)

1. Put `index.html` in a public repo; enable GitHub Pages (branch `main`, folder `/`).
2. Create a **private** data repo with a README (so `main` exists).
3. Create a fine-grained token: only the data repo, Contents read/write.
4. On each device: open the site → Dashboard → *GitHub sync* → repo `owner/name`, token, branch → Connect. On a phone, add the site to the home screen.
5. To deploy a change: commit the new `index.html` to the website repo; Pages updates in ~1 minute. Users may need a hard refresh (or reopen the home-screen app) to pick up the new file.

---

## 16. Change log

- **Base app (8 tabs):** Dashboard, Task, Calendar, Event, Money, Calories, Project, File; GitHub sync with per-item merge and tombstones; PDFs in the data repo; animated background; responsive layout.
- **2026-11-02 — Biocomputing tab:** Researcher / Paper / Scholarship sub-tabs; three new synced collections (`researchers`, `papers`, `scholarships`); shared PDF lookup (`findPdf`); generalised image picker (`picK`).
- **2026-11-02 — Pictures and PDFs as repo files:** `biocomputing/` folders in the data repo, `ghp()` honours `path`, image cache + `hydrate()`, deletion cleanup, automatic `migrateImgs()`, "GitHub not connected" notice.
- **2026-11-02 — Researcher "Why" + starter list:** `why` field on researcher form and card; `SEED` list of 74 organoid-computing researchers (incl. Japan and Singapore) loadable with one button.
- **2026-11-02 — Mobile fixes:** scrollable bottom bar that keeps position and shows the active tab, 16 px inputs (no iOS zoom), 2-column Biocomputing cards, full-width forms, notch-safe PDF viewer with New-tab button, safe-area bottom padding, no sticky hover lift on touch.
