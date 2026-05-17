# Referans Siteler — Kadın Giyim

[inciko.com](https://inciko.com) için kadın giyim e-ticaret sitelerini araştırma, filtreleme, favorileme ve birlikte not alma aracı. **Tek HTML dosyası**, backend yok, iki kişi (Oğuz + İnci) arasında canlı senkron.

## 🔗 Canlı Link

**Tıklayın — tarayıcıda açılır:**

→ https://oguzzkk.github.io/refrence-sites-preview-project/Womens_Wear_Reference_Sites.html

> Tarayıcınızın yer imlerine ekleyin. Telefondan da çalışır.

## Ne yapar?

- **263+ kadın giyim sitesi** tek sayfada (Zara'dan Killstar'a, Toteme'den Trendyol'a)
- 14 kategori ile filtreleme: Minimal · Luxury · DTC · Shopify · Fast Fashion · Indie · Sustainable · Dark/Gothic · Victorian · Avant-Garde · Turkish · Butik · IG Butik
- ♥ **Favori ekle** → siz ve İnci aynı listeyi anında görürsünüz (15 sn senkron)
- ✍️ **Her favoriye not yaz** ("Hero bölümü çok temiz", "Footer iyi referans" vb.)
- 🗑 **Çöp sistemi** → ilgisiz siteleri "Çöp"e at, listeden kalksın (geri alınabilir)

## Nasıl Kullanılır?

| İşlem | Nasıl |
|---|---|
| Site ziyaret | Karta tıkla → yeni sekmede açılır |
| Favori ekle | Kartın **sağ üst köşesindeki ♥** ikonuna tıkla |
| Not yaz | Favorilediğin kartın altında textarea belirir → yaz, 1 sn sonra otomatik kaydeder |
| Çöpe at | Kartın altındaki etiket satırında **🗑** ikonu |
| Sadece favoriler | Üstte kırmızı **♥ Favoriler** butonu |
| Sadece çöp | Üstte **🗑 Çöp** butonu |
| Kategori filtre | Üstteki kategori butonları |

## Veriniz Güvende mi?

Evet. Üç katmanlı koruma:

1. **Bulutta (npoint.io)** — Sizin ve İnci'nin gördüğü ortak liste.
2. **Tarayıcınızda (localStorage)** — Çevrimdışıyken bile çalışır.
3. **GitHub'da otomatik yedek** — Her gün (TR 06:00) GitHub Actions çalışıp [`backups/`](./backups) klasörüne snapshot atıyor. Tüm geçmiş [commit geçmişinde](https://github.com/oguzzkk/refrence-sites-preview-project/commits/main/backups) görünür.

**Veri kaybolursa:** [`backups/favorites.json`](./backups/favorites.json) ve [`backups/trash.json`](./backups/trash.json) en son sağlam halinizi tutar. Bir önceki haftaya dönmek için ilgili commit'i açıp dosyayı bilgisayarınıza indirin.

## Yeni Site Eklemek

Yeni referans site eklemek için: [CONTRIBUTING.md](./CONTRIBUTING.md)

## Sorun Bildirimi

Bir şey bozuksa Oğuz'a (oguzzkk@gmail.com) yaz veya repo'ya issue aç.

## Teknik Detay

Geliştirme yapacaksanız (HTML/JS değişikliği): [CLAUDE.md](./CLAUDE.md)
