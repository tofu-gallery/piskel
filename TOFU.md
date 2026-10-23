# Piskel on Tofu

Piskel is a free, open-source editor for pixel art and animated sprites, by Julian Descottes and contributors. It runs entirely in the browser: pen, shape, fill and selection tools, frames and layers, onion skin, palettes, and export to PNG spritesheets, animated GIF, ZIP and `.piskel` files. There is no account, server or database: a drawing is kept in the visitor's own browser (Save in Browser, and automatic backups in that browser's IndexedDB) or in files the visitor downloads.

## How Tofu runs it

- Framework preset: **static**.
- Root folder: **`site`**. It holds `index.html` and the finished assets; there is no `package.json` in it.
- Install command: none. Build command: none. Output directory: `site` itself, served as it is.
- Environment variables: none. Database and migrations: none.
- `index.html` loads its files by relative paths (`css/`, `js/`, `img/`), so the site works at the address's root.
- Do not import the repository root. Its `package.json` declares `npm run build`, which runs a sprite step, then Vite with upstream's own plugins, then a partials script, and writes to `dest/prod`, not `dist/`. That fits neither of Tofu's static lanes, so the finished site is committed instead.

### How `site/` was made

From the repository root, on the pinned upstream commit:

```sh
npm ci --ignore-scripts --no-audit --no-fund
npm run build
rsync -a --delete --exclude piskelapp-partials dest/prod/ site/
cp LICENSE site/LICENSE
```

- `--ignore-scripts` skips Puppeteer's Chrome download, nw-builder's postinstall and Husky's git hook setup. The web build uses none of them.
- `npm run build` is upstream's own build: `node scripts/sprite.js` (spritesmith icon sheets), `vite build` with the plugins in `vite-plugins/` (concatenates and minifies the scripts, inlines the templates and `piskel-boot.js` into `index.html`), then `node scripts/partials.js`.
- The built script and stylesheet names carry the build time (`-2026-10-23-01-01` here), so a rebuild produces new names. Replace `site/` as a whole (`rsync --delete` above) rather than copying over it.

## What Tofu changed and why

- Added `site/`: upstream's build output for the pinned commit, unmodified, so Tofu's static lane can serve the editor without running a build it cannot run.
- Left out `dest/prod/piskelapp-partials/`. Those three files are Liquid and front-matter page templates for upstream's website, piskelapp.com, not pages of the app; served as they are, they would be broken pages.
- Kept both packaged scripts and their source maps. `index.html` loads the minified script; the unminified one keeps the bundled libraries' licence headers in full.
- Added `site/LICENSE`, a byte-identical copy of the root `LICENSE`, so the licence travels with the site.
- No upstream source file is modified.
- Earlier versions of this file called the mirror branch `main`; it is `master`.

## Upstream and licence

- Upstream: <https://github.com/piskelapp/piskel>, branch `master`, commit `d828941fa9fe76464d69d91d5fd862067c98c02c` (2026-10-21, "Delete RELEASE.md, it's completely outdated"), package version `0.15.2-SNAPSHOT`.
- Fork: <https://github.com/tofu-gallery/piskel>. `master` mirrors upstream and Tofu never commits to it. `tofu` carries Tofu's commits. `pin/2026-10-23` marks the provenance notes alone (no site); `pin/2026-10-23.1` marks this version, with the built site.
- A visitor who chooses this app gets a private snapshot of the pinned version in their own Tofu account. Their copy does not follow upstream updates.
- Licence: Apache-2.0. The root `LICENSE` is unchanged and copied to `site/LICENSE`. Upstream ships no `NOTICE` file.
- Caveats:
  - `site/js` bundles third-party code that upstream vendors in `src/js/lib`, each under its own licence: jQuery 1.8.0 and jQuery UI 1.10.3 (MIT), bootstrap-tooltip 2.1.1 (Apache-2.0), JSZip (MIT or GPLv3, dual) with the zlib.js it carries (MIT), Spectrum colour picker 1.1.2 (MIT) and Ben Alman's pubsub (MIT or GPL, dual). gif.js 0.1.6 and libgif.js are bundled too; their vendored files carry no licence header.
  - The icon fonts in `site/css/fonts` (an IcoMoon set and Piskel's own) and the artwork in `site/img` come from upstream's repository.
  - "Piskel" and its logo are upstream's name and artwork, kept unchanged. This copy is not made or endorsed by the Piskel authors.
  - The site makes no network requests beyond its own files: no analytics, no web fonts, no API. The GitHub and browser download links are plain links.
  - Drawings stay in the visitor's browser or in their downloaded files; Tofu stores nothing. Another browser or device does not see them.

## Verification

Done on 2026-10-23 on macOS (arm64) with Node v22.22.3 and npm 10.9.8. No app was deployed.

1. `npm ci --ignore-scripts --no-audit --no-fund`: added 501 packages from the upstream lockfile.
2. `npm run build`: icon sheets generated, `vite build` finished ("built in 1.07s"), three partials written; exit 0. `site/` then held 107 files, 5.3 MB.
3. Import filter: copied `site/` with every path Tofu's GitHub import drops removed (`node_modules`, `dist`, `build`, `target`, `.cache` and the rest of the list, `.DS_Store`, `._*`, credential files). `diff -rq` found the copy identical to `site/`: nothing is dropped.
4. `python3 -I -m http.server 43880 --bind 127.0.0.1 --directory <filtered copy>`, then `curl`: HTTP 200 for `/` (59,653 bytes) and for the packaged script, stylesheet, both sprite sheets, the logo, an icon font and `LICENSE`.
5. Browser, in its own tab against 127.0.0.1 only: the editor loaded (32x32 sprite, pen tool, black primary colour). Once the boot screen's loading mask was gone (it covers the page for 0.6 s after start), a five-pixel diagonal stroke and a single click drew 6 pixels in the frame. Export, then PNG, then Download handed the browser `New Piskel.png`: a 189-byte `image/png` file with the PNG signature, 32x32, with exactly 6 opaque pixels. The page's own save was intercepted, so no file was written to the Downloads folder. Console errors: none. Failed requests: none from the app; the browser's automatic `/favicon.ico` request got 404 because upstream ships no favicon, which is harmless.
6. After `tofu` was pushed: `python3 -I simulate_import.py --one tofu-gallery/piskel tofu site yes` reported `root=site container=False kept=107 dropped=0` and no problems.

Not run: upstream's Karma unit tests and Playwright end-to-end tests. They need browser downloads that this preparation does not install; step 5 covers the editor's core action.
