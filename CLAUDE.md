# CLAUDE.md — Geliştirici / Claude Context

> Bu dosya: kod üzerinde değişiklik yapacak (Claude veya insan) herkes için. Mimari, dikkat noktaları, edit kuralları.
> Kullanıcı odaklı tanıtım için → [README.md](./README.md)

## Proje Kimliği

- **Repo:** `oguzzkk/refrence-sites-preview-project` (yazım hatası dahil — "refrence" eksik 'e'. Değiştirilirse GitHub Pages URL'i ölür, tüm yer imleri kırılır.)
- **Sahip:** Oğuz Kaan (oguzzkk@gmail.com) — kod, deploy
- **Co-user:** İnci — favori/not katkısı
- **Kapsam:** [inciko.com](https://inciko.com) — Shopify dark/gotik kadın giyim mağazası için rakip & ilham araştırması.
- **Canlı:** https://oguzzkk.github.io/refrence-sites-preview-project/Womens_Wear_Reference_Sites.html

## Repo Yapısı

```
.
├── README.md                         ← kullanıcı için
├── CLAUDE.md                         ← bu dosya, geliştirici için
├── CONTRIBUTING.md                   ← yeni site/etiket ekleme rehberi
├── index.html                        ← hub (4 karta link veriyor ama 3'ü EKSİK, aşağıya bak)
├── Womens_Wear_Reference_Sites.html  ← ASIL UYGULAMA (~149KB, tek dosya, tüm CSS+JS inline)
├── .github/workflows/
│   └── backup-npoint.yml             ← günlük npoint.io yedek workflow'u
└── backups/                          ← Actions tarafından üretilen JSON snapshot'lar
    ├── favorites.json
    ├── trash.json
    └── LAST_BACKUP.txt
```

## Mimari

```
Tarayıcı (HTML+JS)
   ↕  fetch GET/POST her 15 sn
npoint.io  ─── 2 ayrı "bin" ───┐
   │  favorites: 9f430671be24a9b78c70
   │  trash:     8058a715b26b78972bd2
   ↓
GitHub Actions (günlük cron 03:00 UTC)
   ↓
repo: backups/*.json (versionlu yedek)
```

- **Backend yok.** npoint.io ücretsiz JSON storage. Auth yok, CORS açık, GET (oku) + POST (yaz).
- **localStorage fallback:** Offline ise yerel depo, online dönünce remote master kabul.
- **2 kişi senkron:** 15 sn polling ile değişiklikler birbirine ulaşır.

## npoint.io ile çalışırken **DİKKAT**

### 🔴 POST = TÜM bin'i değiştirir (REPLACE, MERGE değil)

```bash
# Mevcut: {"Zara": {...}, "Cos": {...}}
curl -X POST .../9f430671... -d '{"Khaite": {...}}'
# Sonuç: {"Khaite": {...}}  ← Zara ve Cos UÇTU
```

**Kural:** Edit yaparken **önce GET ile mevcut datayı çek**, bellekte merge et, sonra POST.

Sayfa kodundaki `saveFavorites()` bunu güvenle yapıyor çünkü bellekteki `favorites` objesini her zaman master tutuyor ve her POST'ta tüm objeyi gönderiyor.

### npoint.io endpoint'leri

| Veri | URL | Web arayüz |
|---|---|---|
| Favoriler | `https://api.npoint.io/9f430671be24a9b78c70` | https://www.npoint.io/docs/9f430671be24a9b78c70 |
| Çöp | `https://api.npoint.io/8058a715b26b78972bd2` | https://www.npoint.io/docs/8058a715b26b78972bd2 |

### Veri Formatları

```jsonc
// favorites
{
  "BrandName": { "ts": 1771528989579, "note": "kullanıcı notu" }
}

// trash
{
  "BrandName": 1771502455837                    // sadece timestamp (eski format)
  // veya
  "BrandName": { "ts": ..., "note": "..." }     // yeni format (not korunmuş)
}
```

> Trash hem `int` hem `obj` olabilir — `toggleFav` içinde `typeof === 'object'` kontrolü var, koruyun.

## Önemli JS Fonksiyonları (`Womens_Wear_Reference_Sites.html`)

| Fonksiyon | Ne yapar | Dikkat |
|---|---|---|
| `initHearts()` | DOMContentLoaded'da her karta favori, not, çöp butonu enjekte eder | Brand key olarak `.brand` text içeriği kullanılır → aynı isimde iki marka olursa çakışır |
| `toggleFav(brand)` | Favori ekle/çıkar, çöpteki notu geri yükler | İlgili kartı `is-fav` class'ı alır |
| `toggleTrash(brand)` | Çöp ekle/çıkar, favori notunu çöpte korur (restore için) | Favorideyse otomatik favori'den çıkarır |
| `filterCards(tag)` | Kategori filtresi. **`event.target` kullanıyor.** | Programatik çağrıda fail eder → `reapplyFilter()` kullan |
| `filterFavorites()` / `filterTrash()` | Özel filtreler | Aynı `event.target` sorunu |
| `reapplyFilter()` | Mevcut filtreyi yeniden uygular (programatik kullanım için) | toggleTrash sonrası otomatik çağrılır |
| `saveFavorites()` / `saveTrash()` | npoint POST + localStorage backup | Hata olursa `showSync('orange', ...)` |
| `loadFavorites()` / `loadTrash()` | npoint GET, fallback localStorage | Remote boşsa local'i kullanır |
| `showSync(color, text)` | Üstteki yeşil/turuncu sync rozeti | |

### CSS Sınıfları

- `.site-card` — ana kart container
- `.site-card.is-fav` — favori (pembe arka plan)
- `.site-card.is-trashed` — çöpte (soluk)
- `.fav-btn` / `.fav-btn.active` — kalp butonu, sağ üst absolute
- `.fav-note-wrap` / `.fav-note` — auto-grow textarea, favori altında
- `.trash-btn-inline` / `.trash-btn-inline.active` — çöp ikonu, tag-row içinde
- `.tag-XXX` — etiket renkleri (CONTRIBUTING.md'de tam liste)

## Yedek Sistemi

[`.github/workflows/backup-npoint.yml`](./.github/workflows/backup-npoint.yml) her gün 03:00 UTC'de:

1. npoint.io'dan 2 endpoint'i çeker (`curl --fail`)
2. `jq` ile JSON geçerli mi kontrol eder — değilse commit ATLANIR (sağlam yedek bozulmaz)
3. Pretty-print eder (her marka kendi satırında, diff okunur)
4. `LAST_BACKUP.txt` yazar (tarih + count)
5. Değişiklik varsa commit + push (yoksa atlar)

Manuel tetikleme: https://github.com/oguzzkk/refrence-sites-preview-project/actions/workflows/backup-npoint.yml → "Run workflow"

## Bilinen Sorunlar / Borçlar

1. **`index.html` hub'ında 3 kırık link** → `references.html`, `mockup.html`, `design-reference.html`, `build-plan.html` linkleniyor ama dosyalar yok.
   - Drive'da hazır halleri var: `G:\My Drive\Works\inciko\deploy\Site_Deploy_Package\`
   - Çözüm: 4 dosyayı repo'ya kopyalayıp commit, ya da hub'daki linkleri direkt `Womens_Wear_Reference_Sites.html`'e yönlendir.
2. **Repo adında yazım hatası:** "refrence" → "reference" olmalı. Düzeltirseniz GitHub Pages URL'i değişir, tüm yer imleri ve eski paylaşımlar 404. Custom domain (örn. `refs.inciko.com`) bir kez kurulsa kalıcı çözüm olur.
3. **Aynı brand iki kez:** `kfrancestudio.com` ve `fashionnova.com` listede tekrar ediyor. Favoriler/çöp `brand` key'i ile çalıştığı için tek kayıt olur ama liste ham temizlenmeli.
4. **Race condition (nadiren):** İki kişi tam aynı anda yazarsa son POST kazanır. 15 sn polling sayesinde pratikte nadir.
5. **GitHub Pages cache:** Push sonrası 1-10 dk eski sürüm görünebilir. `?v=YYYYMMDD` query ile bypass.
6. **npoint.io rate limit:** Ücretsiz plan dakikada N istek (belgesi flu). Aşırı POST'tan kaçın.

## Editleme Yaparken Checklist

- [ ] `Womens_Wear_Reference_Sites.html` 149KB tek dosya — editör tarafında dikkat
- [ ] Yeni etiket eklerken: filtre butonu (HTML), `.tag-XXX` CSS class'ı, kart `data-tags` — üç yer
- [ ] Yeni JS fonksiyonu eklerken `event.target` kullanma → parametre olarak `btn` al
- [ ] Push öncesi sayfayı lokalde aç, console'da hata yok mu bak
- [ ] Push sonrası canlı linki test et (1-10 dk cache)
- [ ] Veri formatını değiştirdiysen `loadFavorites()` içine eski formatı yeni'ye dönüştüren migration ekle

## İlgili Kaynaklar (Drive)

`G:\My Drive\Works\inciko\` (symlink: `C:\Users\oguzz\Desktop\Working\Inciko.com\`)

- `Referances/ReferanceAnalyzeSite/CONTEXT.md` — Bu CLAUDE.md'nin Şubat 2026 ataşı (eski URL'lerle)
- `Referances/ReferanceAnalyzeSite/backups/` — Manuel snapshot'lar (Actions'tan ayrı)
- `deploy/Site_Deploy_Package/` — Hub'daki 4 eksik HTML burada hazır
- `docs/Shopify_*.pdf` — Shopify build planları (referans aracından ayrı, asıl mağaza için)
- `mockups/Zara_*.html` — Tasarım DNA mockup'ları

## inciko Asıl Tema Repo'su (Ayrı)

Bu repo **sadece araştırma aracı**. Asıl mağaza teması farklı bir repo'da:
- Repo: https://github.com/oguzzkk/inciko (Shopify Horizon teması)
- Lokal: `C:\Users\oguzz\Desktop\İnciko\GitHub\inciko`
- Branch: `dev` (preview), `main` (live)

Bu iki repoyu karıştırma — buradaki değişiklik canlı mağazaya yansımaz.
