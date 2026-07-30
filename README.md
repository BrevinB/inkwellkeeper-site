# Inkwell Keeper — inkwellkeeper.app

Source for the Ink Well Keeper website: the SEO landing page, collecting guides, support and
privacy pages, the web deck viewer, and the Universal Links plumbing that makes a scanned QR
**open the app** when installed and **go to the App Store** when not.

This repo is the single source of truth for the site — the app repo no longer carries a copy.

## Files (all must sit at the REPO ROOT, not in a subfolder)

| File | Purpose |
|------|---------|
| `CNAME` | Binds the custom domain `inkwellkeeper.app`. **Keeping this committed is what stops the site from un-binding ("There isn't a GitHub Pages site here") on redeploys.** |
| `.well-known/apple-app-site-association` | AASA file iOS downloads to verify the app may open links on this domain. No file extension. |
| `.nojekyll` | Forces GitHub Pages to serve dotfolders like `.well-known/`. Without it, Jekyll hides the AASA → 404. |
| `404.html` | Catch-all fallback: any path without the app (e.g. `/card?id=…`) redirects to the App Store. Bulletproofs against a missing per-path page. |
| `index.html` | SEO landing page (features, screenshots, FAQ with JSON-LD schema, smart app banner). |
| `guides/`, `support/`, `privacy/` | Collecting guides (linked from the landing FAQ) and the support/privacy pages App Store Connect points to. |
| `robots.txt`, `sitemap.xml` | Crawl directives + sitemap (update `lastmod` when pages change). |
| `card/`, `set/` | Per-path fallback pages (open-in-app nudge, else App Store). |
| `deck/` | **Web deck viewer**: decodes `?code=IWK2:…` share codes client-side and renders the deck (card art, cost curve, copy-as-text). Falls back to open-in-app / App Store when there's no code or decoding fails. |
| `assets/lzfse.js` | Apple LZFSE decoder — reference [lzfse](https://github.com/lzfse/lzfse) (BSD-3) compiled to single-file WASM, decode-only, plus a small `LZFSE.decode()` wrapper. |
| `assets/card-index.json` | Card id → name/cost/ink/image lookup the viewer resolves codes against. |

AASA `appIDs` = `<TeamID>.<BundleID>` = `YFXZ6WNN53.co.brevinb.Inkwell-Keeper`.

## Regenerating assets

- **`card-index.json`** — rerun after the app gains a new set, then commit here:
  ```sh
  python3 "…/Inkwell Keeper/Scripts/generate_web_card_index.py"
  ```
- **`lzfse.js`** — only if the decoder ever needs rebuilding:
  ```sh
  git clone --depth 1 https://github.com/lzfse/lzfse.git
  docker run --rm -v "$PWD/lzfse:/src" -w /src emscripten/emsdk:3.1.74 emcc -O2 \
    src/lzfse_decode.c src/lzfse_decode_base.c src/lzfse_fse.c src/lzvn_decode_base.c \
    -Isrc --no-entry -sMODULARIZE=1 -sEXPORT_NAME=createLzfseModule -sSINGLE_FILE=1 \
    -sEXPORTED_FUNCTIONS=_lzfse_decode_buffer,_malloc,_free \
    -sEXPORTED_RUNTIME_METHODS=HEAPU8 -sALLOW_MEMORY_GROWTH=1 -sENVIRONMENT=web,node \
    -o lzfse_core.js
  cat lzfse_core.js <wrapper block at the end of assets/lzfse.js> > assets/lzfse.js
  ```

## Deploy

1. Copy the **contents** of this folder to the site repo root (AASA ends up at
   `/.well-known/apple-app-site-association`, and `CNAME` at the root).
2. Commit & push. Settings → Pages → Source = your branch, `/ (root)`.
3. Settings → Pages → Custom domain should already read `inkwellkeeper.app` (from the CNAME file).
   If it's blank, type it and Save once.
4. Tick **Enforce HTTPS** when available (`.app` is HSTS — it won't load over plain HTTP).

Cloudflare DNS (already done): apex A records → `185.199.108–111.153`, **DNS only / grey cloud**.

## Verify

```sh
curl -I https://inkwellkeeper.app/                                            # 200
curl -I https://inkwellkeeper.app/.well-known/apple-app-site-association      # 200 + JSON
curl -I "https://inkwellkeeper.app/card?id=test"                             # 200 (page) or 404→still redirects
```

Apple's cached copy (can lag hours after first deploy):
https://app-site-association.cdn-apple.com/a/v1/inkwellkeeper.app

## Different domain?

Change it in three places to the same host: `AppLinks.universalHost`, the app entitlement
(`applinks:<domain>`), and this `CNAME` file.
