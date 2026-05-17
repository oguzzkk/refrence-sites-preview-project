# Katkı Rehberi — Yeni Site Ekleme

Yeni bir kadın giyim sitesi eklemek istiyorsanız bu rehberi takip edin. Liste `Womens_Wear_Reference_Sites.html` içinde, ham HTML olarak duruyor — başka build adımı yok.

## Hızlı Adımlar

1. `Womens_Wear_Reference_Sites.html` dosyasını aç
2. Uygun kategori bölümüne git (ör. `<!-- DARK / VICTORIAN -->` yorumunun altı)
3. Aşağıdaki template'i kopyala, doldur
4. Commit + push → 1-2 dk sonra canlı

## Kart Template

```html
<a href="https://markaadi.com" target="_blank" class="site-card" data-tags="ETIKET1 ETIKET2">
  <div class="brand">Marka Adı</div><div class="cat">Kategori — Ülke</div>
  <div class="desc">2-3 cümle açıklama. Tasarım gücü, hedef kitle, neden referans olduğu.</div>
  <div class="url">markaadi.com</div>
  <div class="tag-row"><span class="tag tag-ETIKET1">Etiket1</span><span class="tag tag-ETIKET2">Etiket2</span></div>
</a>
```

### Örnek

```html
<a href="https://www.khaite.com" target="_blank" class="site-card" data-tags="luxury minimal">
  <div class="brand">Khaite</div><div class="cat">Premium — USA</div>
  <div class="desc">Modern American luxury. Editorial photography, sparse layout, strong typography.</div>
  <div class="url">khaite.com</div>
  <div class="tag-row"><span class="tag tag-luxury">Luxury</span><span class="tag tag-minimal">Minimal</span></div>
</a>
```

## Etiket Listesi (14 adet)

`data-tags` attribute'unda **boşlukla ayrılmış**, küçük harf. Tek karta birden fazla etiket atayabilirsin.

| Etiket (data-tags) | Görsel etiket | Renk şeması | Anlam |
|---|---|---|---|
| `minimal` | Minimal | gri | Editorial, sade tasarım |
| `luxury` | Luxury | siyah | Lüks, üst segment |
| `dtc` | DTC | yeşil | Direct-to-consumer, kendi markası |
| `shopify` | Shopify | mavi | Shopify üzerinde kurulu |
| `fast` | Fast Fashion | turuncu | Hızlı moda, yüksek hacim |
| `indie` | Indie | mor | Bağımsız, küçük ölçekli |
| `sustainable` | Sustainable | yeşil | Sürdürülebilir/etik üretim |
| `dark` | Dark / Gothic | koyu gri / gümüş | Karanlık estetik, gotik |
| `victorian` | Victorian | mor-siyah / lila | Viktoryan / korse / dantel |
| `avant` | Avant-Garde | koyu gri / açık | Avangart, deneysel |
| `turkish` | Turkish | kırmızı | Türk markası |
| `butik` | Butik | bej | Türk butik tarzı |
| `igbutik` | IG Butik | pembe (Instagram) | Instagram üzerinden satış |

## Etiket Seçim Rehberi (inciko perspektifi)

- Dark wear markası mı? → `dark` (zorunlu) + uygun ek (`victorian`, `avant`, `indie`)
- Türk rakip mi? → `turkish` (zorunlu) + `butik` veya `igbutik` veya `fast`
- Tasarım referansı mı? → `minimal` veya `luxury` (bizim DNA için bunlar yıldız)
- Shopify üzerindeyse not düşmek faydalı → `shopify`

Birden fazla seç → filtre çakışmasında daha fazla bulunabilir olur.

## Yeni Etiket Eklemek (büyük iş)

Mevcut 14 etiket dışında yeni biri lazımsa 3 yerde değişiklik gerekir:

1. **Filtre butonu** — `<div class="filter-bar">` içinde (`onclick="filterCards('YENI_ETIKET')"`)
2. **Tag rengi CSS** — `<style>` içinde `.tag-YENI_ETIKET { background: #...; color: #...; }`
3. **Kart `data-tags`** — kullanmak istediğin kartlarda

Sonra: README.md ve CONTRIBUTING.md'deki etiket tablosunu güncelle.

## Yazım Kuralları

- **Brand:** Markanın resmi yazımı (Title Case). _İlk harfler büyük._
- **Cat (Kategori):** `Tür — Ülke` formatı (em-dash `—`, hyphen değil). Örn: `Premium — France`
- **Desc:** 2-3 cümle, İngilizce (mevcut liste tutarlılığı için). 100-200 karakter ideal.
- **URL (görsel):** Sadece domain (https:// yok, www. yok). Örn: `khaite.com`
- **href:** Tam URL (https:// dahil). Tıklanınca açılan link.

## Brand Adı = Anahtar

Favoriler ve çöp sistemi marka adını (`.brand` text'i) **unique anahtar** olarak kullanıyor. Aynı isimde iki kart eklersen veri çakışır — biri favoriye eklenince diğeri de görünür.

> Mevcut listede tespit edilen duplicate'ler: `kfrancestudio.com` (iki kez), `fashionnova.com` (iki kez). Temizleneceğine düşülen borç.

## Lokalde Test

Push'lamadan önce dosyayı yerel olarak aç:

```
file:///C:/Users/oguzz/Desktop/İnciko/GitHub/refrence-sites-preview-project/Womens_Wear_Reference_Sites.html
```

Tarayıcının Developer Console'unda (F12) hata var mı kontrol et. Yeni kart görünmeli, filtre butonları sayıyı doğru göstermeli.

> Lokal'de açtığında favoriler senkron çalışır (npoint.io public). Test favorisi eklediysen sonra silmeyi unutma.

## Commit Mesajı Örnekleri

```
add: Khaite reference card (luxury, minimal)
add: 5 Turkish IG boutique brands
fix: Sezane URL typo
docs: update tag list in CONTRIBUTING
```

## Sorularınız İçin

- Teknik mimari için: [CLAUDE.md](./CLAUDE.md)
- Genel kullanım için: [README.md](./README.md)
- Direkt Oğuz'a: oguzzkk@gmail.com
