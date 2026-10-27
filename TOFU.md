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
cp LICENSE site/LICENSE.txt
rm site/css/fonts/icomoon.eot site/css/fonts/piskel.eot
```

- `--ignore-scripts` skips Puppeteer's Chrome download, nw-builder's postinstall and Husky's git hook setup. The web build uses none of them.
- `npm run build` is upstream's own build: `node scripts/sprite.js` (spritesmith icon sheets), `vite build` with the plugins in `vite-plugins/` (concatenates and minifies the scripts, inlines the templates and `piskel-boot.js` into `index.html`), then `node scripts/partials.js`.
- The built script and stylesheet names carry the build time (`-2026-10-23-01-01` here), so a rebuild produces new names. Replace `site/` as a whole (`rsync --delete` above) rather than copying over it.
- The last line removes the two Internet Explorer font files that Tofu's Free static lane refuses (see below).

## What Tofu changed and why

- Added `site/`: upstream's build output for the pinned commit, with no file in it modified, so Tofu's static lane can serve the editor without running a build it cannot run. Two of its font files are removed (below).
- Left out `dest/prod/piskelapp-partials/`. Those three files are Liquid and front-matter page templates for upstream's website, piskelapp.com, not pages of the app; served as they are, they would be broken pages.
- Removed `site/css/fonts/icomoon.eot` and `site/css/fonts/piskel.eot`. EOT is a font format only Internet Explorer reads, and Tofu's Free static lane serves no `.eot` file: a single one anywhere in the folder keeps the whole site off that lane. The stylesheet is left as built. Its one `@font-face` rule lists `icomoon.eot` first, then `icomoon.woff`, `icomoon.ttf` and `icomoon.svg`; browsers other than Internet Explorer skip the EOT entry by its `embedded-opentype` format hint and load `icomoon.woff`, so nothing requests the removed files. Nothing refers to `piskel.eot`: the stylesheet uses none of the `piskel.*` font files.
- Kept both packaged scripts and their source maps. `index.html` loads the minified script; the unminified one keeps the bundled libraries' licence headers in full.
- Added `site/LICENSE.txt`, a byte-identical copy of the root `LICENSE`, so the licence travels with the site. The name ends in `.txt` because Tofu's Free static lane publishes a repository document such as `LICENSE` only when its name ends in a type the host serves as it is. Until 2026-10-27 the copy was named `site/LICENSE`, which that lane skips (Verification, step 8).
- No upstream source file is modified.
- Earlier versions of this file called the mirror branch `main`; it is `master`.

## Upstream and licence

- Upstream: <https://github.com/piskelapp/piskel>, branch `master`, commit `d828941fa9fe76464d69d91d5fd862067c98c02c` (2026-10-21, "Delete RELEASE.md, it's completely outdated"), package version `0.15.2-SNAPSHOT`.
- Fork: <https://github.com/tofu-gallery/piskel>. `master` mirrors upstream and Tofu never commits to it. `tofu` carries Tofu's commits. `pin/2026-10-23` marks the provenance notes alone (no site); `pin/2026-10-23.1` marks the first version with the built site, which still held the two `.eot` fonts; `pin/2026-10-23.2` marks the version without them, which still named the licence copy `site/LICENSE`; the next pin marks the version that renames it to `site/LICENSE.txt` (Verification, 2026-10-27).
- A visitor who chooses this app gets a private snapshot of the pinned version in their own Tofu account. Their copy does not follow upstream updates.
- Licence: Apache-2.0. The root `LICENSE` is unchanged and copied to `site/LICENSE.txt`. Upstream ships no `NOTICE` file.
- Caveats:
  - `site/js` bundles third-party code that upstream vendors in `src/js/lib`, each under its own licence: jQuery 1.8.0 and jQuery UI 1.10.3 (MIT), bootstrap-tooltip 2.1.1 (Apache-2.0), JSZip (MIT or GPLv3, dual) with the zlib.js it carries (MIT), Spectrum colour picker 1.1.2 (MIT) and Ben Alman's pubsub (MIT or GPL, dual). gif.js 0.1.6 and libgif.js are bundled too; their vendored files carry no licence header.
  - The icon fonts in `site/css/fonts` (an IcoMoon set and Piskel's own) and the artwork in `site/img` come from upstream's repository.
  - "Piskel" and its logo are upstream's name and artwork, kept unchanged. This copy is not made or endorsed by the Piskel authors.
  - The site makes no network requests beyond its own files: no analytics, no web fonts, no API. The GitHub and browser download links are plain links.
  - Drawings stay in the visitor's browser or in their downloaded files; Tofu stores nothing. Another browser or device does not see them.

## Verification

Done on 2026-10-23 on macOS (arm64) with Node v22.22.3 and npm 10.9.8. No app was deployed.

Building `site/` (`pin/2026-10-23.1`):

1. `npm ci --ignore-scripts --no-audit --no-fund`: added 501 packages from the upstream lockfile.
2. `npm run build`: icon sheets generated, `vite build` finished ("built in 1.07s"), three partials written; exit 0. `site/` then held 107 files, 5.3 MB.
3. Import filter: copied `site/` with every path Tofu's GitHub import drops removed (`node_modules`, `dist`, `build`, `target`, `.cache` and the rest of the list, `.DS_Store`, `._*`, credential files). `diff -rq` found the copy identical to `site/`: nothing is dropped.
4. `python3 -I -m http.server 43880 --bind 127.0.0.1 --directory <filtered copy>`, then `curl`: HTTP 200 for `/` (59,653 bytes) and for the packaged script, stylesheet, both sprite sheets, the logo, an icon font and `LICENSE`.
5. Browser, in its own tab against 127.0.0.1 only: the editor loaded (32x32 sprite, pen tool, black primary colour). Once the boot screen's loading mask was gone (it covers the page for 0.6 s after start), a five-pixel diagonal stroke and a single click drew 6 pixels in the frame. Export, then PNG, then Download handed the browser `New Piskel.png`: a 189-byte `image/png` file with the PNG signature, 32x32, with exactly 6 opaque pixels. The page's own save was intercepted, so no file was written to the Downloads folder. Console errors: none. Failed requests: none from the app; the browser's automatic `/favicon.ico` request got 404 because upstream ships no favicon, which is harmless.
6. After `tofu` was pushed: `python3 -I simulate_import.py --one tofu-gallery/piskel tofu site yes` reported `root=site container=False kept=107 dropped=0` and no problems.

Removing the two `.eot` fonts (`pin/2026-10-23.2`):

7. Import filter and browser: copied the new `site/` with every path Tofu's GitHub import drops removed. `diff -rq` against a copy of the old `site/` made the same way listed only the two `.eot` files; the new copy holds 105 files. Each copy was served with `python3 -I -m http.server` on 127.0.0.1 and opened in its own browser tab at 1440x900. In both, the page requested one font file, `css/fonts/icomoon.woff`, with HTTP 200, the `piskel` font face reported `loaded`, and nothing asked for an `.eot`. The layers panel and the preview's onion-skin bar, where the icon font is drawn, are pixel-identical in the two screenshots; the full screenshots differ only in a strip under 3 pixels wide at the window's right edge, over the right-hand toolbar's image buttons, which do not use the font. A diagonal pen stroke and a single click drew pixels in the 32x32 frame in both copies (8 and 10 opaque pixels: the editor throttles pointer moves by time, so a scripted stroke's exact pixels vary between runs). Console errors: none. Failed requests: only the browser's automatic `/favicon.ico` (404), as before.
8. After `tofu` was pushed, Tofu's own static-lane and safety predicates, run on the branch as Tofu's GitHub import fetches it with root `site` (`lane_check.mts --one tofu-gallery/piskel tofu site`): `files=105 published=104 -> static (finished site) | safety ok (1 findings)`. On `pin/2026-10-23.1` the same check gave `-> not on the static lane | refused: type×2`, the two `.eot` files. The one file not published is `site/LICENSE`: the lane keeps it in the stored code and skips it as a repository document. The one safety finding is below the blocking severities and was reported for `pin/2026-10-23.1` too.
9. `python3 -I simulate_import.py --one tofu-gallery/piskel tofu site yes`: `root=site container=False kept=105 dropped=0`, no problems.

Not run: upstream's Karma unit tests and Playwright end-to-end tests. They need browser downloads that this preparation does not install; steps 5 and 7 cover the editor's core action.

Done on 2026-10-27 on macOS (arm64) with Python 3.11.4 and curl 8.1.1. No app was deployed in these steps.

Publishing the licence as `.txt`:

10. Why: in step 8 `site/LICENSE` was the one file the Free static lane did not publish, so the site that lane publishes lacked Piskel's Apache-2.0 licence text. The lane skips a repository document such as `README.md`, `LICENSE` or `CHANGELOG` unless its name ends in a type the host serves as it is, such as `.txt`. Tofu's rule for this (`staticPathVerdict` in Tofu's source) lists `txt` among the published types; it was read, not run.
11. `git mv site/LICENSE site/LICENSE.txt`: git reports a 100% rename, and `cmp LICENSE site/LICENSE.txt` finds the two files identical (11,358 bytes each). `grep` over `site/` finds no link or path to a file named `LICENSE`: `index.html` does not mention a licence, and the only mentions elsewhere are the Apache licence URL and JSZip's licence URL, in comments of the bundled stylesheet and the unminified script and in the two source maps.
12. `python3 -I -m http.server <port> --bind 127.0.0.1 --directory site`, then `curl`: HTTP 200 for `/` (59,653 bytes) and for `/LICENSE.txt` (11,358 bytes, `text/plain`). The body of `/LICENSE.txt` is byte-identical to the root `LICENSE` (`cmp`), and the old name, `/LICENSE`, answers 404. The server was stopped afterwards.

Not run for this change: Tofu's own static-lane check on `site/LICENSE.txt` and a copy of the app made through Tofu (steps 8 and 9 were run before the rename), and a browser check, since the change moves one text file that no page loads.
