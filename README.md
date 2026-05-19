# Womens Wear Reference Sites

Single-page research tool for browsing, filtering, favoriting and annotating women's fashion e-commerce sites. **No backend**, **single HTML file**, **two-user live sync** via a public JSON store.

## 🔗 Live

→ https://oguzzkk.github.io/refrence-sites-preview-project/Womens_Wear_Reference_Sites.html

## What it does

- **263+ women's fashion sites** in a single page (Zara, Toteme, Killstar, COS, Khaite, etc.)
- **14 category filters:** Minimal · Luxury · DTC · Shopify · Fast Fashion · Indie · Sustainable · Dark/Gothic · Victorian · Avant-Garde · Turkish · Butik · IG Butik
- ♥ **Favorite a site** → shared across both users, syncs every 15s
- ✍️ **Write a note** on any favorite (auto-saves after 800ms)
- 🗑 **Trash** sites you've ruled out (reversible, hidden from main view)

## How to use

| Action | How |
|---|---|
| Visit a site | Click the card — opens in new tab |
| Favorite | Click the **♥** in the card's top-right corner |
| Note | Once favorited, a textarea appears under the card — type, auto-saves |
| Trash | Click the **🗑** in the card's tag row |
| Show favorites only | Top bar → **♥ Favoriler** |
| Show trash only | Top bar → **🗑 Çöp** |
| Filter by category | Category buttons in the top bar |

## Is the data safe?

Three-layer protection:

1. **Cloud (npoint.io)** — Primary shared storage.
2. **Browser localStorage** — Offline fallback per browser.
3. **GitHub Actions daily backup** — Snapshot committed to [`backups/`](./backups) every day. All history in [commit log](https://github.com/oguzzkk/refrence-sites-preview-project/commits/main/backups).

**To restore:** open any past commit in `backups/` and download `favorites.json` / `trash.json`. To re-import, POST it back to the npoint endpoint (see [CLAUDE.md](./CLAUDE.md)).

## Contributing

To add a new site or category, see [CONTRIBUTING.md](./CONTRIBUTING.md).
For architecture details and editing rules, see [CLAUDE.md](./CLAUDE.md).
