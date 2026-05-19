# CLAUDE.md — Developer / AI Context

> For anyone (human or AI) editing this codebase. Architecture, pitfalls, edit rules.
> User-facing intro → [README.md](./README.md)

## Project

- **Repo:** `oguzzkk/refrence-sites-preview-project` (typo "refrence" intentionally preserved — renaming would break the GitHub Pages URL and all bookmarks)
- **Purpose:** Two-user shared research tool for women's fashion e-commerce sites. Filterable card grid + favorites + notes + trash, synced via a public JSON store.
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
├── index.html                        ← hub page (links to 3 files NOT in repo, see "Known debt")
├── Womens_Wear_Reference_Sites.html  ← MAIN APP (~149KB, single file, all CSS+JS inline)
├── .github/workflows/
│   └── backup-npoint.yml             ← daily npoint.io snapshot workflow
└── backups/                          ← workflow output: JSON snapshots
    ├── favorites.json
    ├── trash.json
    └── LAST_BACKUP.txt
```

## Architecture

```
Browser (HTML + JS)
   ↕  fetch GET/POST every 15s
npoint.io  ─── 2 bins ──────────┐
   │  favorites: 9f430671be24a9b78c70
   │  trash:     8058a715b26b78972bd2
   ↓
GitHub Actions (daily cron 03:00 UTC)
   ↓
repo: backups/*.json (versioned snapshots)
```

- **No backend.** npoint.io is a free anonymous JSON storage. GET reads, POST writes. No auth, CORS open.
- **localStorage fallback:** If offline, uses local copy. When online, remote is master.
- **Multi-user sync:** 15s polling pushes changes to the other browser.

## npoint.io — Critical Behavior

### 🔴 POST replaces the ENTIRE bin (it is NOT a merge)

```bash
# Current: {"Zara": {...}, "Cos": {...}}
curl -X POST .../9f430671... -d '{"Khaite": {...}}'
# Result:  {"Khaite": {...}}   ← Zara and Cos DELETED
```

**Rule:** Always `GET` current state, merge in memory, then `POST` the merged whole object.

The page's `saveFavorites()` does this correctly by keeping the in-memory `favorites` object as the master and sending the whole thing on every save. Any external tool/script that POSTs a partial object will wipe data.

### Endpoints

| Bin | URL | Web UI |
|---|---|---|
| Favorites | `https://api.npoint.io/9f430671be24a9b78c70` | https://www.npoint.io/docs/9f430671be24a9b78c70 |
| Trash | `https://api.npoint.io/8058a715b26b78972bd2` | https://www.npoint.io/docs/8058a715b26b78972bd2 |

> These URLs ARE the auth — anyone who knows them can read/write. They are already publicly exposed in the HTML, so do not treat them as secrets.

### Data formats

```jsonc
// favorites
{
  "BrandName": { "ts": 1771528989579, "note": "user note text" }
}

// trash (mixed legacy + new format — handle both)
{
  "BrandName": 1771502455837,                       // legacy: timestamp only
  "AnotherBrand": { "ts": ..., "note": "..." }      // new: with preserved note
}
```

## Key JS Functions (`Womens_Wear_Reference_Sites.html`)

| Function | Role | Caveat |
|---|---|---|
| `initHearts()` | On DOMContentLoaded, injects heart/note/trash buttons into every card | Brand key = `.brand` element's text — duplicate brand names will collide |
| `toggleFav(brand)` | Toggle favorite, restores note from trash if present | Adds/removes `is-fav` class |
| `toggleTrash(brand)` | Toggle trash, preserves favorite note for later restore | Auto-removes from favorites if present |
| `filterCards(tag)` | Category filter. **Uses `event.target`.** | Will fail when called programmatically — use `reapplyFilter()` |
| `filterFavorites()` / `filterTrash()` | Special filters | Same `event.target` caveat |
| `reapplyFilter()` | Re-applies current filter (safe for programmatic calls) | Called automatically after toggleTrash |
| `saveFavorites()` / `saveTrash()` | npoint POST + localStorage backup | On error → `showSync('orange', ...)` |
| `loadFavorites()` / `loadTrash()` | npoint GET with localStorage fallback | If remote empty, uses local |
| `showSync(color, text)` | Updates sync badge (green/orange) | |

### CSS Classes

- `.site-card` — base card container
- `.site-card.is-fav` — favorited (pink background)
- `.site-card.is-trashed` — trashed (faded)
- `.fav-btn` / `.fav-btn.active` — heart button, absolute top-right
- `.fav-note-wrap` / `.fav-note` — auto-grow textarea below favorited card
- `.trash-btn-inline` / `.trash-btn-inline.active` — trash icon inside tag-row
- `.tag-XXX` — per-tag colors (full table in CONTRIBUTING.md)

## Backup Workflow

[`.github/workflows/backup-npoint.yml`](./.github/workflows/backup-npoint.yml) runs daily at 03:00 UTC:

1. `curl --fail` both npoint endpoints
2. Validate JSON with `jq` — invalid → step fails → commit skipped (protects last good snapshot from npoint outages)
3. Pretty-print (one brand per line) for readable diffs
4. Write `LAST_BACKUP.txt` (timestamp + counts)
5. Commit + push if anything changed; skip otherwise

Manual trigger: Actions tab → "Backup npoint data" → "Run workflow".

## Known Debt

1. **Broken hub links** — `index.html` references `references.html`, `mockup.html`, `design-reference.html`, `build-plan.html`. None exist in repo. Either commit them or point the hub directly at `Womens_Wear_Reference_Sites.html`.
2. **Repo name typo** — "refrence" should be "reference". Renaming breaks the GitHub Pages URL (which is not auto-redirected). A custom domain would make future renames painless.
3. **Duplicate brands** — `kfrancestudio.com` and `fashionnova.com` appear twice each. Since favorites use brand text as a unique key, only one of each can be favorited; the dup should be removed.
4. **Race condition (rare)** — Two simultaneous writes lose one update. The 15s polling makes this rare in practice.
5. **GitHub Pages cache** — Up to 10 min stale after push. Use `?v=YYYYMMDD` query to bypass for testing.
6. **npoint.io rate limits** — Free tier has undocumented limits; avoid hammering with POST loops.

## Edit Checklist

- [ ] `Womens_Wear_Reference_Sites.html` is 149KB single file — be careful with editor performance
- [ ] Adding a new tag requires **three changes**: filter button (HTML), `.tag-XXX` CSS class, card `data-tags` attribute
- [ ] Never use `event.target` in new JS — pass the button as a parameter instead
- [ ] Open the file locally before pushing; check browser console for errors
- [ ] After push, test the live URL (1–10 min Pages cache)
- [ ] If you change a data format, add a migration in `loadFavorites()` that converts old → new shape

## Manual Restore from Backup

If npoint.io data is wiped/corrupted:

```bash
# 1. Download the desired snapshot from a past commit in backups/
# 2. POST it back to the bin (replaces current state)
curl -X POST https://api.npoint.io/9f430671be24a9b78c70 \
  -H "Content-Type: application/json" \
  -d @favorites.json

curl -X POST https://api.npoint.io/8058a715b26b78972bd2 \
  -H "Content-Type: application/json" \
  -d @trash.json
```

Then open the live page in a browser — within 15s both users see the restored data.
