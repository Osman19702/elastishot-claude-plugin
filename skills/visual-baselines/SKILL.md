---
name: visual-baselines
description: Sets up and runs a visual regression suite over a set of page URLs and viewports, using the elastishot CLI. Use when the user asks to add visual regression tests to a project, capture a baseline screenshot of a page, sweep every page after bumping a design system, Tailwind, a font or a UI dependency, gate CI on visual changes with JUnit output, or accept an intentional redesign as the new reference. Covers the config file, the run / snapshot / approve / report commands, and the traps around baselines. Runs a command-line tool, so it needs a terminal with Node.js 20+ (Claude Code).
when_to_use: Trigger phrases include "set up visual regression", "add screenshot tests", "create a baseline", "check all the pages after this upgrade", "run the visual check in CI", "accept the new design as the baseline", "approve the screenshots", "update the baselines". Also use when an elastishot command fails or surprises — `no config file found`, `E_CONFIG`, an unknown config key, exit 2, the visual check failing in CI, every target reported as new, `approve` finding nothing to approve, or baselines that need updating. To check what one branch or pull request changed without keeping baselines, use /elastishot:pr-visual-check; for a one-off comparison of two images or two URLs, use /elastishot:compare-screenshots.
---

# Visual baselines and a CI gate

Config-driven runs with the `elastishot` CLI from npm. Nothing is vendored here; run it with `npx`.

If the user only wants to know what a branch changed, `/elastishot:pr-visual-check` needs no baselines at all. Baselines are for tracking deployed pages over time.

## Prerequisites

Unlike a one-off image comparison, every command here captures pages, so **all** of these apply:

- Node.js 20 or newer (`node --version`). Without a shell, say plainly that this needs Claude Code.
- Playwright and a browser, always:
  `npm install -D playwright && npx playwright install chromium`
  Without it: `E_CAPTURE`, exit 2.
- **An `elastishot.config.mjs`** (or `.js`, `.cjs`, `.json`) **in the project.** The nearest one from the current folder up is used. `--out`, `outDir` and `baselineDir` resolve against the config file's folder.
  - Prefer `.mjs`. A `.js` config with `export default` fails with `E_CONFIG` in a CommonJS project (see below).
  - `.json` works anywhere but cannot carry custom reporters, functions or setup hooks.
- **Every target URL must be reachable while the run executes.**
  - For a local project, start the dev or preview server first, and point the targets (or `baseUrl`, or `run --base-url`) at it.
  - In CI, start the server, or serve a built `dist/`, and wait for the port before `run`.
  - An unreachable URL fails as `E_CAPTURE` with `cannot open <url>` and `net::ERR_CONNECTION_REFUSED`. That means a server is not listening, not that a browser is missing, so do not reinstall Playwright for it.

## The config

```js
// elastishot.config.mjs
export default {
  baseUrl: 'https://example.com/',     // relative target URLs resolve against it
  baselineDir: 'baselines',
  outDir: '.elastishot/runs',
  threshold: 0.98,
  failOn: ['added', 'removed', 'changed', 'moved'],
  viewports: [
    { name: 'desktop', width: 1280, height: 800 },
    { name: 'mobile', width: 390, height: 844, deviceScaleFactor: 2 },
  ],
  capture: { fullPage: true, hide: ['.cookie-banner'] },
  ignore: ['[data-testid="clock"]'],   // left out on both sides, wherever it lands
  report: { junit: true },
  targets: [
    { name: 'home', url: '/', expect: '[data-testid="hero"]' },
    { name: 'pricing', url: '/pricing', viewports: ['desktop'] },
  ],
}
```

**How the config is loaded.** It goes through dynamic `import()`, with no CommonJS fallback.
- `export default` in a `.js` file therefore works only when the nearest package.json has `"type": "module"`.
- Otherwise `run` exits 2 with `elastishot: cannot load <file>: Unexpected token 'export'` (`E_CONFIG`).
- Use `.mjs`, which is ESM whatever the host package says, or a `.cjs` with `module.exports = { ... }`.

**Keys are validated strictly**, and an unknown key or a wrong value is `E_CONFIG`, exit 2:
- **Top level:** `baseUrl`, `pr`, `baselineDir`, `outDir`, `threshold`, `failOn`, `minRegionScore`, `viewports`, `capture`, `compare`, `locators`, `report`, `targets`, `ignore`, `reporters`.
- **A target:** `name`, `url`, `viewports`, `capture`, `compare`, `ignoreRegions`, `ignore`, `expect`.
- **`capture` keys** are checked too.

**Capture options that make captures reproducible:**
- `locale`, `timezoneId`.
- `clock: '2026-01-15T09:41:00Z'`: pins `Date.now()` before the page loads.
- `storageState`: a file path relative to the config, or the state object itself.
- `setup: async (page, context) => {}`: JS configs only, runs before navigation.
- `reducedMotion`: `'reduce'` by default.
- `animations: 'finish'`: runs animations to their end instead of freezing them.
- `waitFor`: a selector, a delay (`2000` or `'2000ms'`), or a function.

A target's `expect` is a selector the ready page must show. Without it, the capture fails instead of quietly baselining a sign-in page.

## Commands

| Goal | Command |
| --- | --- |
| Bootstrap, first run | `npx elastishot@0.3.0 run --update` |
| Regular check | `npx elastishot@0.3.0 run` |
| Only some targets | `npx elastishot@0.3.0 run home pricing/desktop` |
| Same targets against a local build | `npx elastishot@0.3.0 run --base-url http://127.0.0.1:4173/` |
| CI | `npx elastishot@0.3.0 run --junit` |
| Accept an intentional redesign of one target | `npx elastishot@0.3.0 approve <target>` |
| Accept every difference of the last run | `npx elastishot@0.3.0 approve --all` |
| Re-baseline everything that differs, in one go | `npx elastishot@0.3.0 run --update --force` |
| One-off baseline for a single page | `npx elastishot@0.3.0 snapshot <url> --name landing --viewport mobile` |
| Re-render reports from a finished run | `npx elastishot@0.3.0 report <runDir>` |

Keep the exact version. This skill documents the 0.3.0 flags, exit codes and defaults. Bare `npx elastishot` resolves whatever is newest on npm, and a newer engine can move scores enough to make stored baselines fail against a contract you did not choose.

`run --update` writes only the **missing** baselines. Adding `--force` also replaces the ones that differ: those pairs keep their diff in the report with the reason "baseline replaced", and the run exits 0. Prefer `approve <target>` when only some changes are intended. An unknown target name exits 2 and lists the configured names.

`run` applies `--full-page`, `--wait-for`, `--hide` and `--no-map` to every target, over the config. Viewports always come from the config, so `--viewport` and `--name` on `run` are usage errors (exit 2), as is any other flag a command does not use.

## Approving

- Only pairs produced by `run` from config targets are approvable: they carry both a target and a viewport name. `compare` pairs and `pr` runs are not.
- With no run folder, `approve` reads `<outDir>/latest`.
  - When the latest run was a `compare`, it uses the newest run made by `run` instead.
  - It prints a note on stderr saying which run it used.
  - It also notes when that run compared against a different baseline folder.
- `approve <what>` treats `<what>` as a run folder only when that folder holds a `report.json`; otherwise it is a target name.
- Pairs with status `error` are never promoted.
- Runs made with `--single-file` can be approved like any other.
- `approve` with neither a target nor `--all` exits 2 and lists the approvable candidates.

## Exit codes

- **0**: every pair passed. Also a `run --update` whose only non-passing pairs were baselines it just wrote, and a `run --update --force`.
- **1**: at least one pair failed, or a target had no baseline and `--update` was not passed. All reports are still written.
- **2**: usage or runtime error, **including a run where some pair errored** (`totals.errors > 0`) while the others succeeded.

## Output

A run folder at `<outDir>/<runId>` holds:
- `index.html`, one card per pair;
- `pairs/<pairId>/report.html`, the slider / flip / blink / overlay / diff viewer;
- `report.json` and `pairs/<pairId>/result.json`;
- with `--junit`, `junit.xml`;
- with `report.markdown: true`, `summary.md` (in GitHub Actions the same table goes to the job summary).

`<outDir>/latest` holds the newest run's path.

**In CI:**
1. Start the app, or serve the built output, and wait for its port.
2. Run `npx elastishot@0.3.0 run --junit`.
3. Let the reporter pick up `junit.xml`.
4. Upload the run folder as an artifact so the HTML report survives. `index.html` carries inline thumbnails but links into `pairs/`, so upload the whole folder, or add `--single-file` if a single attachable page matters more than size.

**Do not `cat` report.json or pipe `--json` into the conversation**: it embeds base64 thumbnails for every side of every pair.
- Read `pairs/<pairId>/result.json`, or pull only `.totals`, `.pairs[].id`, `.pairs[].status`, `.pairs[].summary.counts` and `.pairs[].locators.changedLocators`.
- The console prints the target name, not the pair id. For a `run` pair the id is `<target>--<viewport>`, with both parts lowercased and non-alphanumerics collapsed to `-`; `.pairs[].id` in report.json is the reliable source.

Check `summary.warnings` before trusting a pass:
- `CAPTURE_ERROR_PAGE`: the capture was an error page.
- `CAPTURE_REDIRECTED`: the page ended at another URL.
- `IGNORE_UNRESOLVED`: an `ignore` locator matched nothing in either element map. Use test ids, ids or role locators, or `capture.mask` for other CSS selectors.

## Do not

- Run `run` in a repo with no baselines without `--update` or asking first: it exits 1 by design, which reads like a failure.
- Use `run --update --force` to make a red check green without reading the report first.
- Expect `networkidle` to settle on a page holding a websocket or long-poll: the 30s capture timeout turns it into an error pair and exit 2. Give that target a `capture: { waitFor: '<selector>' }`.
- Reach for this for a one-off "what is different between these two images". That is **`/elastishot:compare-screenshots`**, and "what did my branch change" is **`/elastishot:pr-visual-check`**.
- Use it for responsive reflow (desktop against mobile *layout*), canvas/WebGL/video pages, or anything a DOM diff answers better.
