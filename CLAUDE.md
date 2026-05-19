# CLAUDE.md — Developer / AI Context

> For anyone (human or AI) editing this codebase. Architecture, pitfalls, edit rules.
> User-facing intro → [README.md](./README.md)

## Project

- **Repo:** `oguzzkk/refrence-sites-preview-project` (typo "refrence" intentionally preserved — renaming would break the GitHub Pages URL and all bookmarks)
- **Purpose:** Two-user shared research tool for women's fashion e-commerce sites. Filterable card grid + favorites + notes + trash. **Storage is the repo itself** (`data/` folder) via GitHub Contents API — every edit becomes a git commit.
- **Live:** https://oguzzkk.github.io/refrence-sites-preview-project/Womens_Wear_Reference_Sites.html

## Companion Docs

- **Adding sites, adding tags, card HTML template, tag color table** → [CONTRIBUTING.md](./CONTRIBUTING.md). Read this before editing the card list.
- **Public-facing intro** → [README.md](./README.md) (minimal vitrine, links back here).

## Repo Layout

```
.
├── README.md                         ← user-facing
├── CLAUDE.md                         ← this file, for editors
├── CONTRIBUTING.md                   ← how to add sites / tags
├── index.html                        ← hub page (1 card → main app)
├── Womens_Wear_Reference_Sites.html  ← MAIN APP (~155KB, single file, all CSS+JS inline, token embedded)
├── data/                             ← LIVE STORAGE (every edit = commit)
│   ├── favorites.json
│   └── trash.json
└── backups/                          ← legacy npoint snapshots (kept as historical reference)
    ├── favorites.json
    ├── trash.json
    └── LAST_BACKUP.txt
```

## Architecture

```
Browser (HTML + JS)
   │
   │  GitHub Contents API (token-auth)
   │  GET   /repos/.../contents/data/favorites.json   ← read with SHA
   │  PUT   /repos/.../contents/data/favorites.json   ← write with SHA
   │  poll  every 15s — diff SHA → if changed, refresh
   ↓
data/favorites.json + data/trash.json in this repo
   │
   │  Every PUT = new git commit = automatic versioning
   ↓
Full restore via GitHub UI → "Commits" → pick a commit → "View file" → revert
```

- **Storage is git itself.** No external service, no third party. The repo is the database.
- **Auth via hardcoded fine-grained PAT** baked into the HTML. Scope: **only this repo, only Contents read+write**. Worst-case leak = sham commit (revert via git history).
- **SHA-based concurrent-update protection.** Every PUT requires the file's last known SHA. If two clients race, the second one gets `409 Conflict` → re-fetch latest, merge, retry. No silent overwrite.
- **localStorage fallback** for offline use. When back online, GitHub is master.
- **15s polling** detects partner's edits (SHA change → fetch new content).

## GitHub Contents API — How It Works Here

### Read

```js
const data = await ghGet('data/favorites.json');
// data = { content: "<base64-utf8>", sha: "abc123...", ... }
favorites = JSON.parse(b64decode(data.content));
favoritesSha = data.sha;   // save for next write
```

### Write

```js
const res = await ghPut('data/favorites.json', favorites, favoritesSha);
// PUT body: { message, content: b64encode(json), sha, branch }
// On 200/201: new SHA returned, update favoritesSha
// On 409: someone else wrote in between — re-fetch + merge + retry once
```

### Base64 helpers (UTF-8 safe)

JS's native `btoa`/`atob` are Latin-1 only — Turkish characters break.
```js
function b64encode(str) { return btoa(unescape(encodeURIComponent(str))); }
function b64decode(b64) { return decodeURIComponent(escape(atob(b64))); }
```

## Token

Embedded in `Womens_Wear_Reference_Sites.html` as `const GH_TOKEN`. It is **public** (anyone can view-source). Scope limits the blast radius:

- ✅ Read/write THIS repo's contents
- ❌ Any other repo
- ❌ Gists
- ❌ Account / e-mail / settings
- ❌ Issues, PRs, Actions

**Rotation plan:** when token expires (1 year), generate a new fine-grained PAT with the same scope (Contents read+write on this repo only), search-replace `GH_TOKEN` in the HTML, commit, push. ~2 min job.

**Compromise response:** if you suspect leak, revoke the token at https://github.com/settings/tokens — site goes read-only immediately. Generate new one, replace.

## Data formats (`data/*.json`)

```jsonc
// data/favorites.json
{
  "BrandName": { "ts": 1771528989579, "note": "user note text" }
}

// data/trash.json  (mixed legacy + new formats — handle both)
{
  "BrandName":  1771502455837,                                       // legacy: timestamp only
  "AnotherB":   { "ts": ..., "note": "..." },                        // user-trashed, has note
  "ThirdBrand": { "ts": ..., "note": "...", "preservedOnly": true }  // silent archive (note from an un-favorited card; NOT shown in Çöp list)
}
```

## Key JS Functions (`Womens_Wear_Reference_Sites.html`)

| Function | Role | Caveat |
|---|---|---|
| `ghGet(path)` / `ghPut(path, obj, sha)` | GitHub Contents API helpers | `ghPut` returns the raw Response; caller checks `.ok` and handles 409 conflicts |
| `initHearts()` | On DOMContentLoaded, injects heart/note/save-button/trash buttons into every card | Brand key = `.brand` element's text — duplicate brand names collide. **Notes use a hybrid save model**: typing marks the Kaydet button `.pending` (red outline) and triggers a **3-second idle auto-save fallback** so a closed tab doesn't lose unsaved text. Clicking Kaydet commits immediately and cancels the pending timer. Both paths bump `favorites[brand].ts = Date.now()`. |
| `toggleFav(brand)` | Toggle favorite, restores note from trash if present. **Un-favoriting a card with a note silently archives the note into `trashed[brand]` with `preservedOnly: true`** — protects against accidental clicks wiping the note. | Adds/removes `is-fav` class |
| `isUserTrashed(brand)` | Helper: returns true only if the brand is in `trashed` AND not a silent archive. | Used by `renderTrash`, `filterCards`, `filterTrash`, trash badge count |
| `toggleTrash(brand)` | Toggle trash, preserves favorite note for later restore | Auto-removes from favorites if present |
| `filterCards(tag)` | Category filter. **Uses `event.target`.** | Will fail when called programmatically — use `reapplyFilter()` |
| `filterFavorites()` / `filterTrash()` | Special filters | Same `event.target` caveat |
| `reapplyFilter()` | Re-applies current filter (safe for programmatic calls) | Called automatically after toggleTrash |
| `saveFavorites()` / `saveTrash()` | ghPut with SHA tracking + 409 retry with merge | On error → `showSync('orange', ...)` + localStorage |
| `loadFavorites()` / `loadTrash()` | ghGet + b64decode + save SHA | If remote unreachable, uses localStorage |
| `showSync(color, text)` | Updates sync badge (green/orange) | |

### CSS Classes

- `.site-card` — base card container
- `.site-card.is-fav` — favorited (pink background)
- `.site-card.is-trashed` — trashed (faded)
- `.fav-btn` / `.fav-btn.active` — heart button, absolute top-right
- `.fav-note-wrap` / `.fav-note` — auto-grow textarea below favorited card
- `.fav-note-save` / `.fav-note-save.pending` — Kaydet button (red outline = unsaved)
- `.trash-btn-inline` / `.trash-btn-inline.active` — trash icon inside tag-row
- `.tag-XXX` — per-tag colors (full table in CONTRIBUTING.md)

## Known Debt

1. **Repo name typo** — "refrence" should be "reference". Renaming breaks the GitHub Pages URL (which is not auto-redirected). A custom domain would make future renames painless.
2. **Duplicate brands** — `kfrancestudio.com` and `fashionnova.com` appear twice each. Since favorites use brand text as a unique key, only one of each can be favorited; the dup should be removed.
3. **GitHub Pages cache** — Up to 10 min stale after push. Use `?v=YYYYMMDD` query to bypass for testing.
4. **GitHub API rate limits** — Authenticated requests: 5000/hour per token. Two users polling every 15s = ~480/hour total. Plenty of headroom but be careful with debug loops.
5. **HTML structural integrity** — Any stray `</div>` in the card list closes the grid container early and breaks layout for all cards after it. Diagnostic: count `<div>` vs `</div>` inside `.site-grid`. Should be equal (offset by 2: the grid + the `.note` are still open at the search boundary).

## Edit Checklist

- [ ] `Womens_Wear_Reference_Sites.html` is ~155KB single file — be careful with editor performance
- [ ] Adding a new tag requires **three changes**: filter button (HTML), `.tag-XXX` CSS class, card `data-tags` attribute
- [ ] Never use `event.target` in new JS — pass the button as a parameter instead
- [ ] Open the file locally before pushing; check browser console for errors
- [ ] After push, test the live URL (1–10 min Pages cache)
- [ ] If you change a data format, add a migration in `loadFavorites()` that converts old → new shape
- [ ] Do NOT regenerate the token unnecessarily — rotation breaks both users' sessions briefly

## Manual Restore

Data is git-versioned. To restore an earlier state:

### Option A — UI (easiest)
1. Open https://github.com/oguzzkk/refrence-sites-preview-project/commits/main/data/favorites.json
2. Pick the commit you want to revert to
3. "View file at this commit" → "Raw" → save locally as `favorites.json`
4. Upload back to the repo via UI (replace `data/favorites.json`)
5. Within 15s both browsers re-sync

### Option B — Git CLI
```bash
git log --oneline -- data/favorites.json   # find the commit
git show <commit>:data/favorites.json > data/favorites.json
git add data/favorites.json && git commit -m "restore favorites" && git push
```

### Option C — Specific bad write rollback
```bash
git revert <bad-commit-hash>
git push
```

## Migration history

- **2026-02 to 2026-05:** Storage was npoint.io (free anonymous JSON store). Daily backup workflow snapshot'd it to `backups/`.
- **2026-05-19:** Migrated to GitHub Contents API. Token-auth, repo-scoped, every edit committed. Old `backups/` folder kept as historical reference but no longer updated. npoint.io bins abandoned (still reachable, just unused).
