# Contributing — Adding Sites and Tags

The site list lives as raw HTML inside `Womens_Wear_Reference_Sites.html`. No build step.

## Quick Steps

1. Open `Womens_Wear_Reference_Sites.html`
2. Jump to the right category section (e.g. `<!-- DARK / VICTORIAN -->`)
3. Paste the template below, fill it in
4. Commit + push → live in 1–2 min (GitHub Pages cache)

## Card Template

```html
<a href="https://brandsite.com" target="_blank" class="site-card" data-tags="TAG1 TAG2">
  <div class="brand">Brand Name</div><div class="cat">Category — Country</div>
  <div class="desc">2–3 sentences. Design strength, target audience, why it's a reference.</div>
  <div class="url">brandsite.com</div>
  <div class="tag-row"><span class="tag tag-TAG1">Tag1</span><span class="tag tag-TAG2">Tag2</span></div>
</a>
```

### Example

```html
<a href="https://www.khaite.com" target="_blank" class="site-card" data-tags="luxury minimal">
  <div class="brand">Khaite</div><div class="cat">Premium — USA</div>
  <div class="desc">Modern American luxury. Editorial photography, sparse layout, strong typography.</div>
  <div class="url">khaite.com</div>
  <div class="tag-row"><span class="tag tag-luxury">Luxury</span><span class="tag tag-minimal">Minimal</span></div>
</a>
```

## Tag Reference (14 tags)

In `data-tags`, use **space-separated, lowercase**. A card can have multiple tags.

| Tag (data-tags) | Display | Color scheme | Meaning |
|---|---|---|---|
| `minimal` | Minimal | grey | Editorial, clean design |
| `luxury` | Luxury | black | High-end, premium |
| `dtc` | DTC | green | Direct-to-consumer, own brand |
| `shopify` | Shopify | blue | Built on Shopify |
| `fast` | Fast Fashion | orange | Fast fashion, high volume |
| `indie` | Indie | purple | Independent, small scale |
| `sustainable` | Sustainable | green | Sustainable / ethical production |
| `dark` | Dark / Gothic | dark grey / silver | Dark aesthetic, gothic |
| `victorian` | Victorian | purple-black / lilac | Victorian / corset / lace |
| `avant` | Avant-Garde | dark grey / light | Avant-garde, experimental |
| `turkish` | Turkish | red | Turkish brand |
| `butik` | Butik | beige | Turkish boutique style |
| `igbutik` | IG Butik | pink (Instagram) | Instagram-first seller |

## Tag Selection Guide

- **Pick all tags that genuinely apply** — overtagging dilutes filters, undertagging makes the site invisible.
- **Combine for discoverability:** a Turkish IG boutique selling dark gothic items → `turkish butik dark`. It will show up in all three filters.
- **Platform tags matter:** if the site is on Shopify, always add `shopify` — that's a research signal.

## Adding a Brand-New Tag (bigger task)

If none of the 14 existing tags fit, you need to change **three places**:

1. **Filter button** in `<div class="filter-bar">` (`onclick="filterCards('NEW_TAG')"`)
2. **Tag color CSS** in `<style>`: `.tag-NEW_TAG { background: #...; color: #...; }`
3. **Card `data-tags`** in whichever cards use it

Then update the tag table in this file and in `README.md`.

## Writing Rules

- **Brand:** Brand's official spelling, Title Case.
- **Cat (category):** `Type — Country` format with em-dash `—` (not hyphen). E.g. `Premium — France`
- **Desc:** 2–3 sentences. English (for consistency with existing list). 100–200 chars ideal.
- **URL (display):** Domain only — no `https://`, no `www.`. E.g. `khaite.com`
- **href:** Full URL including `https://`. This is what opens on click.

## Brand Name = Unique Key

The favorites and trash systems use the brand text as a **unique identifier**. If you add two cards with the same brand name they will collide — favoriting one favorites both.

> Existing duplicates in the list: `kfrancestudio.com` (×2), `fashionnova.com` (×2). Tech debt to be cleaned up.

## Testing Locally

Open the file directly in a browser before pushing:

```
file://<repo-root>/Womens_Wear_Reference_Sites.html
```

Open DevTools (F12) and check the Console for errors. New card should appear; filter buttons should count correctly.

> Local also syncs with npoint.io (public bin). If you favorite something for testing, remember to un-favorite before pushing.

## Commit Message Examples

```
add: Khaite reference card (luxury, minimal)
add: 5 Turkish IG boutique brands
fix: Sezane URL typo
docs: update tag list in CONTRIBUTING
```

## More

- Architecture details: [CLAUDE.md](./CLAUDE.md)
- General usage: [README.md](./README.md)
