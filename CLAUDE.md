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

## How It Works — End-to-End Flow

Senaryo bazlı tam akış. Yeni bir oturumda kodu okumaya başlamadan önce buradan başla.

### 1. Sayfa ilk açıldığında

```
1. Browser HTML+CSS+JS yükler (~155 KB tek dosya)
2. JS başlar:
   - initHearts() her karta heart/note/save/trash butonları enjekte eder
   - loadTrash()  → GitHub Contents API GET → data/trash.json → b64 decode → trashed objesi
   - loadFavorites() → aynı → favorites objesi
   - SHA'lar saklanır (favoritesSha, trashedSha) — sonraki PUT'larda gerekecek
3. renderHearts() / renderTrash() → DOM'da kalp pembelenir, çöp ikonu aktiflenir
4. Üstte yeşil "Senkronize" rozeti
5. setInterval başlar: her 15 saniyede bir SHA diff kontrolü
```

### 2. Kullanıcı bir karta kalp tıklar

```
1. toggleFav(brand) → favorites[brand] = { ts: Date.now(), note: '' }
   (eğer çöpte preservedOnly:true ile not varsa, onu da restore eder)
2. renderHearts() → kalp anında pembelenir (optimistic UI)
3. saveFavorites():
   - ghPut('data/favorites.json', favorites, favoritesSha)
   - GitHub: { message, content: b64encode(JSON), sha, branch }
   - Yanıt 200/201 → yeni SHA gelir, favoritesSha güncellenir
   - Yanıt 409/422 → conflict: re-fetch + merge + retry once
4. localStorage'a yedek yazılır
5. Üstte yeşil "Kaydedildi" rozeti (2 sn)
6. GitHub'da yeni commit: "update data/favorites.json @ <ISO timestamp>"
```

### 3. Kullanıcı favoride bir not yazar

```
1. Textarea'ya yazmaya başlar
   - oninput: textarea auto-grow + saveBtn.classList.add('pending') (kırmızı vurgu)
   - clearTimeout(noteTimer) + setTimeout(commitNote, 3000) → 3 sn idle fallback
2. İki yoldan biri commit'i tetikler:
   - Kullanıcı Kaydet butonuna tıklar → commitNote() anında
   - VEYA kullanıcı 3 saniye yazmazsa → setTimeout commitNote()'u çağırır
3. commitNote():
   - favorites[brand].note = textarea.value
   - favorites[brand].ts = Date.now()
   - saveBtn'den pending kalkar
   - saveFavorites() → PUT (yukarıdaki akış)
```

### 4. Partner cihazından değişiklik (örn. İnci akşam telefonla)

```
Sizin tarayıcınız her 15 sn'de polling yapar:
1. ghGet('data/favorites.json') → { content, sha }
2. data.sha !== favoritesSha mı? → EVET → değişiklik var
3. b64decode(content) → JSON.parse → favorites = remote
4. favoritesSha = data.sha
5. renderHearts() → İnci'nin eklediği yeni kalpler pembelenir
6. Üstte "Partnerden güncellendi" → 2 sn sonra "Senkronize"
```

### 5. Yarış durumu (siz + İnci aynı anda yazma)

```
T=0    : İnci "Khaite"a not yazar, Kaydet basar → PUT (sha=A)
         GitHub kabul eder, yeni sha=B oluşur
T=0.5  : Siz "Sezane"a not yazıyorsunuz, sizin local sha hâlâ A
T=1    : Siz Kaydet basarsınız → PUT (sha=A, ama remote sha=B) → 409 Conflict
T=1.2  : saveFavorites() conflict handler:
         - ghGet() → remote = {...İnci'nin Khaite notu..., ...mevcut...}
         - favorites = Object.assign({}, remote, favorites)
                        ← bizim Sezane notumuzu remote'un üstüne katlar
         - favoritesSha = yeni sha (B)
         - retry PUT (sha=B) → kabul, yeni sha=C
T=1.5  : Hem İnci'nin Khaite hem sizin Sezane notu GitHub'da
```

> Bu, npoint.io'nun "last-write-wins" davranışından çok daha iyi. Race condition'larda kayıp yerine merge oluyor.

### 6. Kazara favori'den çıkarma (silent archive)

```
1. Kullanıcı notu olan bir kartın kalbine tekrar tıklar (kazara)
2. toggleFav():
   - existingNote = favorites[brand].note (boş değil)
   - trashed[brand] = { ts, note: existingNote, preservedOnly: true }
   - saveTrash() → GitHub'a commit
   - delete favorites[brand]
   - saveFavorites() → GitHub'a commit
3. Kart "All" filtresinde GÖRÜNÜR (preservedOnly olduğu için çöp olarak işaretlenmez)
4. Kullanıcı tekrar kalp tıklarsa: trashed[brand].note → favorites[brand].note restore
```

### 7. Offline → online geçişi

```
Offline iken:
- ghPut() throw eder (fetch fail)
- catch: localStorage.setItem('store-project-favs', JSON.stringify(favorites))
- Üstte turuncu "Yerel olarak kaydedildi"

Online döner:
- Bir sonraki edit ghPut'u tetikler
- GitHub'a önce GET (loadFavorites SHA güncellemesi için), sonra PUT
- localStorage'daki son state otomatik buluta gider
```

### 8. Restore — kazara silinen not

```
Senaryo: İnci "Khaite" notunu yanlışlıkla sildi.

1. https://github.com/oguzzkk/refrence-sites-preview-project/commits/main/data/favorites.json
2. Silmeden öncekine tıkla → commit detayında diff görür
3. "Browse files at this point in history" → data/favorites.json → "Raw" → kopyala
4. data/favorites.json'a UI'dan yapıştır → commit "restore favorites"
5. 15 sn içinde her iki kullanıcının sayfasında not geri görünür
```

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

## Future Improvements — "Belki Sonra" Defteri

Şu an gerek görülmeyen ama ileride faydalı olabilecek iyileştirmeler. Karar verilmedi, sadece belgelendi. Bir gün biri (sen veya yeni bir oturum) "şu yapılsa nasıl olur?" diye sorduğunda buradan başlasın.

### ~~Cloudflare Worker proxy (token gizleme + origin check)~~ ✅ DONE (2026-05-20)

Implemented. Worker URL: `https://refsites-proxy.oguzzkk.workers.dev/`. Token lives as `GITHUB_TOKEN` Secret in Cloudflare. Worker enforces `Origin: https://oguzzkk.github.io`. Worker code lives only in Cloudflare panel (not in repo) — if it ever needs editing, log into Cloudflare → Workers & Pages → refsites-proxy → Edit code.

**If the token needs rotation (1 year):**
1. GitHub → Settings → Personal access tokens → revoke old + create new (same scope: Contents read+write, only this repo)
2. Cloudflare → Worker → Settings → Variables and Secrets → edit `GITHUB_TOKEN` → paste new value → Save and deploy
3. HTML never changes

### Custom domain

**Sorun:** Repo adında yazım hatası var ("refrence" yerine "reference" olmalı). Yeniden adlandırırsak GitHub Pages URL'i bozulur, yer imleri kırılır.

**Çözüm:** Domain bağla → `refs.<senin-domain>.com` gibi. Repo adı değişse bile URL stabil kalır.

**Maliyet:** Yıllık ~$10-15 domain ücreti, DNS ayarı 5-10 dk, GitHub Pages tarafı 5 dk.

**Ne zaman değer:** Repo'nun adını düzeltmek istediğinde, veya URL'i daha sade/hatırlanır yapmak istediğinde.

### Real-time sync (15 sn polling → anında push)

**Sorun:** Partner'in değişikliği 15 saniyeye kadar gecikebilir. Çoğu kullanım senaryosunda fark etmez ama anlık sohbet hissi yok.

**Çözüm seçenekleri:**
- **Firebase Realtime Database** — gerçek anlık WebSocket bağlantısı. Free tier rahat. Ama Google ekosistemine bağlanmak.
- **Supabase Realtime** — açık kaynak Firebase alternatifi. Self-host edilebilir. PostgreSQL tabanlı.
- **Cloudflare Durable Objects** — daha düşük seviye, WebSocket destekli. Worker ekosisteminde.

**Maliyet:** 1-2 saatlik mimari değişikliği. Storage modelinden auth'a kadar her şey değişir.

**Ne zaman değer:** Sayfa "araştırma defteri"nden gerçek "ortak çalışma uygulaması"na dönüşmek istenirse.

### Aktivite akışı (sağ panel — "ne ne zaman olmuş")

**Sorun:** Şu an "kim ne zaman ne ekledi" bilgisi sadece git history'de. Sayfa içinde Slack tarzı bir aktivite akışı yok.

**Çözüm:** Sağ tarafa mini panel:
```
14:32  ♥ Khaite favorilere eklendi
       "Hero bölümü çok temiz"
14:28  ♥ Sezane favorilere eklendi
13:45  🗑 Onze Mode çöpe atıldı
```

Veriyi data/activity.json'a yaz (her aksiyon → append). Veya git commit mesajlarını GitHub API ile çekip parse et (daha temiz, ekstra dosya yok).

**Maliyet:** 1-1.5 saat. UI bileşeni + commit parsing.

**Ne zaman değer:** "Bugün ne yaptık?" sorusu sık sorulmaya başlanırsa.

### Duplicate brand temizliği

**Sorun:** Listede `kfrancestudio.com` ve `fashionnova.com` birer kez fazla geçiyor. Aynı brand text'i = aynı key, favoriler/çöp ikisini birden işaretler.

**Çözüm:** HTML'den fazla geçen kartları sil (5 dakika), commit.

**Maliyet:** ~5 dakika.

**Ne zaman değer:** Bir gün biri "neden iki kart aynı görünüyor" diye sorduğunda. Veya genel temizlikte.

### data.json dosyalarını saatlik snapshot olarak da yedekle

**Sorun:** Şu an sadece data/ klasörü versionlu (her edit commit). Tarihli bir snapshot listesi yok. "Geçen pazartesi ne vardı?" demek için commit history'i tarihe göre filtrelemek gerekir.

**Çözüm:** Eski backup workflow geri eklenebilir — günlük veya saatlik bir cron, `data/` → `backups/snapshots/<tarih>.json` olarak kaydeder. Browse açısından kolay.

**Maliyet:** 10 dk, workflow yazma.

**Ne zaman değer:** Restore senaryoları sıklaşırsa veya tarih-bazlı sorgular önemli olursa.

### npoint.io bin'lerini sil veya boşalt

**Sorun:** Eski npoint.io bin'leri hâlâ canlı (içeride eski data), URL'leri bilen biri yazıp bozabilir (artık bu sayfadan etkilenmiyor ama yine de açık).

**Çözüm:** Bin'lere son bir POST atıp `{}` yap. Veya npoint.io paneline girip silmek (eğer hesap varsa).

**Maliyet:** 2 dakika.

**Ne zaman değer:** Tam temizlik istenirse. Yoksa bırakılabilir, görmezden gelinir.

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
