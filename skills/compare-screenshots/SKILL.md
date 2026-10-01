---
name: compare-screenshots
description: Compares two screenshots or two page URLs and names the DOM elements that changed. Use when the user asks whether a CSS, component or refactor change moved anything else on the page, whether staging still looks like production, whether a deploy or PR changed the UI, what is different between two screenshots or image files they point at, or which element moved when a section reflowed (not CLS, the web-vitals layout-shift metric, which this cannot measure). Built for pairs a plain pixel diff cannot handle — different sizes, a zoom, or a section that collapsed or expanded.
when_to_use: Trigger phrases include "did this change the UI", "compare these two screenshots", "does staging look like prod", "what moved on the page", "visual diff", "screenshot diff", "this PR should not change anything visually", "the design mock vs what we built" — including when the two images arrive as chat attachments rather than files. Also use when a comparison fails or surprises — `E_CAPTURE`, `E_DECODE`, a capture that times out, or a page that never settles. Needs two capturable sides — two image files on disk, or two URLs a browser can reach; with neither, a code or DOM diff is the faster answer. For baselines, a config-driven suite or a CI gate, use /elastishot:visual-baselines instead.
---

# Compare two screenshots or two pages

One-shot visual comparison with the `elastishot` CLI from npm. Nothing is vendored here; run it with `npx`.

## Prerequisites

- Node.js 20 or newer. `npx elastishot@^0.2.0` fetches the package on first use (~13 MB, pure JS plus a WASM build, no native compile step).
- **Playwright is needed only when a side is a page URL** — a page you want captured, not an image file or an image URL. Install it in the project:
  `npm install -D playwright && npx playwright install chromium`
  Without it a page capture fails with `E_CAPTURE` and exit 2. Comparing two local images needs no browser at all.
- **`compare` is not config-free.** It discovers `elastishot.config.js`, `.mjs` or `.json` in the directory you run from (cwd only, no upward search). When one is present it overrides the defaults below — threshold, fail-on, outDir, report images, junit even without `--junit`, and **any custom reporters in the config run**, so an ad-hoc comparison inside a project can fire that project's Slack webhook. `cd` to a neutral directory, or pass `--threshold` and `--out` explicitly, for a comparison that must not touch the project's settings.

## Run it

Two local images, the offline path:

```
npx elastishot@^0.2.0 compare before.png after.png
```

Two live pages:

```
npx elastishot@^0.2.0 compare https://staging.example.com/ https://example.com/ --full-page
```

The version range is deliberate: this skill documents the 0.2.x flags, exit codes and defaults, and bare `npx elastishot` would silently pick up a newer contract (0.2.0 already changed scores on ~10% of a 230-pair corpus).

Each side may independently be a PNG or JPEG file, an image URL, a page URL, or a baseline/snapshot folder. **PNG and JPEG only** — SVG, WebP, AVIF and PDF fail with `E_DECODE`, exit 2. Exactly two positionals; a third is a usage error.

If the two images arrived as chat attachments rather than files on disk, write them out first (the scratchpad or a temp directory) and pass those paths. The CLI takes only filesystem paths, image URLs, page URLs or snapshot folders — it cannot read an attachment.

Flags that apply to `compare`:

- `--out <dir>` — write the run folder to a known path instead of `.elastishot/runs/<timestamp>`.
- `--name <name>` — name the pair (default: derived from the candidate). It also fixes the pair id, so pass it when you want to know the id up front.
- `--threshold <0..1>` — minimum similarity to pass (default `0.98`, unless a config in the cwd overrides it).
- `--fail-on <kinds>` — comma list of `added,removed,changed,moved`, or the single word `none`.
- `--config <file>` — point at an `elastishot.config.*` explicitly.
- `--viewport <WxH[@dpr]>` — e.g. `1280x800` or `390x844@2`. Page captures only.
- `--full-page` — capture the whole scrollable page.
- `--wait-for <selector|ms>` — **all digits means milliseconds**; anything else is a CSS selector. `--wait-for 2000` waits 2s; `--wait-for 2000ms` is treated as a selector and times out.
- `--hide <selector>` — hide an element before capture, repeatable. Use it for cookie banners, clocks and carousels.
- `--single-file` — inline every image into both HTML pages, so each page is one attachable file. Expect a few MB per pair.
- `--no-map` — skip the element map. **Do not pass it here**: it removes the changed-locator list, which is the point of this tool.
- `--junit` / `--junit-file <file>`, `--quiet`, `--json`.

## Read the output

Human output is one line per pair, then a totals line, then the report path:

```
FAIL after  score 97.1%  +2 -0 ~4 >0  -> #faq, [data-testid="cta"]
1 pair: 0 passed, 1 failed, 0 new, 0 errors
report: /work/app/.elastishot/runs/2026-09-24T09-31-07Z/index.html
```

That is `compare before.png after.png`: the pair name comes from the candidate file, so it is `after`. The run folder path is always printed absolute, and its directory name is the UTC run id.

The counts are added / removed / changed / moved; the arrow lists the top locators by pixel evidence. That locator list is usually the whole answer — report it to the user instead of describing the picture.

Exit codes:

- **0** — every pair passed. Say plainly that nothing changed visually.
- **1** — differences found. The run folder is still written.
- **2** — usage or runtime error, or a pair that failed to capture or compare. stderr always reads `elastishot: <message>`; read it before guessing.

**Do not `cat` report.json and do not pipe `--json` into the conversation.** report.json always embeds base64 thumbnails for every side, plus every region and alignment band — easily megabytes. Read `<runDir>/pairs/<pairId>/result.json` instead: the same per-pair report with the thumbnails stripped. From report.json, read only `.totals`, `.pairs[].id`, `.pairs[].status`, `.pairs[].summary.counts`, `.pairs[].locators.changedLocators` and `.pairs[].artifacts.report`.

The console line prints the pair **name**, not its id, and `<pairId>` appears nowhere in the output. Get it from `.pairs[].id` in report.json (`.pairs[].artifacts.report` gives the pair page's path relative to `index.html`), or pass `--name <slug>` so you know it up front. The id is the name lowercased with any run of non-alphanumeric characters collapsed to `-`, trimmed and cut at 80 characters — `compare before.png after.png` gives id `after`, and a candidate of `https://example.com/` gives `example-com`.

Before trusting a high score on a captured page, check `summary.warnings` for `CAPTURE_ERROR_PAGE` — an error page compares beautifully against another error page.

To show the user the result, point at `<runDir>/index.html` (summary cards with thumbnails) or `<runDir>/pairs/<pairId>/report.html` (slider / flip / blink / overlay / diff viewer).

`index.html` embeds its thumbnails as data URIs, so it renders on its own, but its links into `pairs/` only work inside the run folder. `pairs/<pairId>/report.html` references `baseline.png`, `candidate.png` and `diff.png` as sibling files, so on its own it renders with no images. To hand someone a single file, re-run with `--single-file`, which inlines every image into both pages (expect a few MB per pair); otherwise send the whole run folder.

In a project that keeps baselines, `compare` rewrites `<outDir>/latest` — the file a bare `approve` reads — and a compare pair is never approvable. Pass `--out` to send an ad-hoc comparison elsewhere if an `approve` is pending.

## When something fails

- `E_CAPTURE` naming a missing Playwright (`capturing pages needs Playwright`) or a failed `browserType.launch` (`cannot launch chromium`): install Playwright and a browser (above), or compare saved images instead.
- `E_CAPTURE` reading `cannot open <url>` with `net::ERR_CONNECTION_REFUSED`, `ERR_NAME_NOT_RESOLVED` or an HTTP status: the URL is wrong or nothing is serving it — a dev server that is not running, or the wrong port. Start the server and retry. **Do not reinstall Playwright for this**; the browser launched fine.
- A capture that times out: the default wait is `networkidle` with a 30s timeout, so a page holding a websocket or long-poll never settles. Pass `--wait-for <selector>` for an element that marks the page ready.
- Noise everywhere: `--hide` the moving parts before you widen the threshold.
- Equal-width pairs up to 1920px compare at full size; wider or unequal-width pairs are downscaled, which makes whole-pixel shifts fractional and slightly noisier. Same-viewport pairs are the most precise.

## Do not use this

- For text or content changes. A DOM/HTML diff or `git diff` answers that faster and exactly.
- To assert an element exists, has a class or shows a value. That is a Playwright or Testing Library assertion.
- To check whether two images are identical or trivially different. A hash or pixelmatch is far cheaper; this tool exists for pairs that are *not* the same size or layout.
- To measure Cumulative Layout Shift. CLS is a runtime web-vitals metric from Lighthouse or the `web-vitals` library, not something two still images can show.
- On desktop-vs-mobile or any responsive reflow where columns become rows. Alignment is one global transform plus a vertical band map, so you get huge regions and a misleading score.
- On rotated, perspective-warped or more-than-2x-rescaled pairs, or photographic and generated imagery with no stable features.
- On pages dominated by canvas, WebGL, video or cross-origin iframes — no locators, and nothing reliable to align on.
- In a watch mode, a per-save check or a unit test suite. Every invocation reloads the OpenCV WASM runtime and relaunches a browser: seconds per call, not milliseconds.
- When the user only wants a screenshot. Drive Playwright directly.

For baselines, a config-driven suite across targets and viewports, approving an intentional redesign, or a CI gate, use **`/elastishot:visual-baselines`** in this plugin.
