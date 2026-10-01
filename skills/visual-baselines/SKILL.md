---
name: visual-baselines
description: Sets up and runs a visual regression suite over a set of page URLs and viewports, using the elastishot CLI. Use when the user asks to add visual regression tests to a project, capture a baseline screenshot of a page, sweep every page after bumping a design system, Tailwind, a font or a UI dependency, gate CI on visual changes with JUnit output, or accept an intentional redesign as the new reference. Covers the config file, the run / snapshot / approve / report commands, and the traps around baselines.
when_to_use: Trigger phrases include "set up visual regression", "add screenshot tests", "create a baseline", "check all the pages after this upgrade", "run the visual check in CI", "accept the new design as the baseline", "approve the screenshots". Also use when an elastishot command fails or surprises — `no config file found`, `E_CONFIG`, an unknown config key, exit 2, the visual check failing in CI, every target reported as new, `approve` finding nothing to approve, capture flags on `run` having no effect, or baselines that need updating. For a one-off comparison of two images or two URLs with no baselines, use /elastishot:compare-screenshots instead.
---

# Visual baselines and a CI gate

Config-driven runs with the `elastishot` CLI from npm. Nothing is vendored here; run it with `npx`.

## Prerequisites

Unlike a one-off image comparison, every command here captures pages, so **all** of these apply:

- Node.js 20 or newer.
- Playwright and a browser, always:
  `npm install -D playwright && npx playwright install chromium`
  Without it: `E_CAPTURE`, exit 2.
- An `elastishot.config.js`, `.mjs` or `.json` **in the directory you run from**. There is no upward search. From a subdirectory you silently get all defaults and `run` exits 2 with `no config file found`. Pass `--config <file>` when running from anywhere but the project root — and note that `--out`, `outDir` and `baselineDir` then resolve against the *config file's* directory, not the cwd. Prefer `.mjs`: a `.js` config with `export default` fails with `E_CONFIG` in a CommonJS project (see below). `.json` works anywhere but cannot carry custom reporters.
- **Every `targets[].url` must be reachable while the run executes.** For a local project, start the dev or preview server first and point the targets at its `http://localhost:<port>` URLs; in CI, start the server (or serve a built `dist/`) and wait for the port before `run`. An unreachable URL fails as `E_CAPTURE` with `cannot open <url>` and a `net::ERR_CONNECTION_REFUSED` message — that is a server that is not listening, not a missing browser, so do not reinstall Playwright for it.

## The config

```js
// elastishot.config.mjs
export default {
  baselineDir: 'baselines',
  outDir: '.elastishot/runs',
  threshold: 0.98,
  failOn: ['added', 'removed', 'changed', 'moved'],
  viewports: [
    { name: 'desktop', width: 1280, height: 800 },
    { name: 'mobile', width: 390, height: 844, deviceScaleFactor: 2 },
  ],
  capture: { fullPage: true, hide: ['.cookie-banner'] },
  report: { junit: true },
  targets: [
    { name: 'home', url: 'https://example.com/' },
    { name: 'pricing', url: 'https://example.com/pricing', viewports: ['desktop'] },
  ],
}
```

The config is loaded with dynamic `import()`, with no CommonJS fallback. `export default` in a `.js` file therefore works only when the nearest package.json has `"type": "module"` — otherwise `run` exits 2 with `elastishot: cannot load <file>: Unexpected token 'export'` (`E_CONFIG`), and on Node 20 before 20.19 that is also what a package.json with no `type` field does. Use `elastishot.config.mjs`, which is ESM whatever the host package says; in a CommonJS project a `.js` file with `module.exports = { ... }` also works, because `import()` hands its `module.exports` back as the default. Discovery looks for `elastishot.config.js`, then `.mjs`, then `.json` — `.cjs` is not discovered.

Keys are validated strictly: an unknown key is `E_CONFIG`, exit 2. Top level accepts only `baselineDir`, `outDir`, `threshold`, `failOn`, `minRegionScore`, `viewports`, `capture`, `compare`, `locators`, `report`, `targets`, `reporters`. A target accepts only `name`, `url`, `viewports`, `capture`, `compare`, `ignoreRegions`. Custom reporters need a `.js`/`.mjs` config; a `.json` config cannot carry them.

## Commands

| Goal | Command |
| --- | --- |
| Bootstrap, first run | `npx elastishot@0.2.0 run --update` |
| Regular check | `npx elastishot@0.2.0 run` |
| Only some targets | `npx elastishot@0.2.0 run home pricing/desktop` |
| CI | `npx elastishot@0.2.0 run --junit` |
| Accept an intentional redesign | `npx elastishot@0.2.0 approve <target>` or `approve --all` |
| One-off baseline for a single page | `npx elastishot@0.2.0 snapshot <url> --name landing --full-page` |
| Re-render reports from a finished run | `npx elastishot@0.2.0 report <runDir>` |

Keep the exact version. This skill documents the 0.2.0 flags, exit codes and defaults, and bare `npx elastishot` resolves whatever is newest on npm — 0.2.0 alone moved the score on ~10% of a 230-pair corpus, which is enough to make stored baselines fail against a contract you did not choose.

`run --update` writes only the **missing** baselines and never overwrites an existing one; re-baselining a page that changed on purpose is what `approve` is for. An unknown target name exits 2 and lists the configured names.

## `run` silently ignores the capture flags

`run` forwards only the positional targets, `--update`, `--threshold`, `--fail-on`, `--out`, `--single-file`, `--junit`/`--junit-file`, plus `--config`, `--json` and `--quiet`. **`--viewport`, `--full-page`, `--wait-for`, `--hide`, `--no-map` and `--name` are parsed without complaint and never reach the capture.** They belong in the config, under `viewports[]` and `capture{}`. Likewise `snapshot` ignores everything but `--out`, `--name`, `--viewport`, `--full-page`, `--wait-for`, `--hide`, `--no-map`, plus `--config` and `--json` — and `--config` matters on `snapshot`, because it sets the `baselineDir` root the capture lands in and supplies the default viewport when `--viewport` is omitted. `approve` ignores everything but `--all`, `--config` and `--json`.

## Approving

- Only pairs produced by `run` from config targets are approvable — they carry both a target and a viewport name. A pair made by `compare` is never approvable and `approve` reports nothing to approve.
- With no argument it reads `<outDir>/latest`, a plain text file holding the newest run folder's absolute path.
- **Any `compare` in the same project rewrites `<outDir>/latest`** — `compare` picks up the project config, so it shares the same `outDir`, and it overwrites that file on every invocation. Since a compare pair is not approvable, one ad-hoc `compare` between `run` and `approve` makes a bare `approve` exit 2 with nothing to approve, and the run you meant is then reachable only by name. Pass the run folder explicitly (`npx elastishot@0.2.0 approve --all <outDir>/<runId>`) whenever a compare may have happened since the run.
- `approve <what>` treats `<what>` as a **run folder** if it happens to be an existing directory relative to the cwd, otherwise as a target name. A target named like a directory in the project root is misread silently.
- Pairs with status `error` are never promoted.
- A run made with `--single-file` writes no `baseline.png` / `candidate.png`, so it leaves nothing for `approve` to promote.
- `approve` with neither a target nor `--all` exits 2 and lists the approvable candidates.

## Baseline path trap

`snapshot --viewport 390x844@2` names the folder from the literal viewport string: `baselines/<name>/390x844-2`. A later `run` whose config calls that viewport `mobile` looks in `baselines/<name>/mobile`, finds nothing and reports the target as new. Create baselines a `run` will find with `run --update`, and keep `snapshot` for ad-hoc captures.

## Exit codes

- **0** — every pair passed. Also a `run --update` whose only non-passing pairs were baselines it just wrote.
- **1** — at least one pair failed, or a target had no baseline and `--update` was not passed. All reports are still written.
- **2** — usage or runtime error, **including a run where some pair errored** (`totals.errors > 0`) while the others succeeded.

## Output

A run folder at `<outDir>/<runId>` with `index.html` (one card per pair), `pairs/<pairId>/report.html` (slider / flip / blink / overlay / diff viewer), `report.json`, `pairs/<pairId>/result.json` and, with `--junit`, `junit.xml`. `<outDir>/latest` holds the newest run's path.

In CI: start the app (or serve the built output) and wait for its port, run `npx elastishot@0.2.0 run --junit` from the config's directory, let the reporter pick up `junit.xml`, and upload the run folder as an artifact so the HTML report survives. `index.html` carries inline thumbnails but links into `pairs/`, so upload the whole folder — or add `--single-file` if a single attachable page matters more than size.

**Do not `cat` report.json or pipe `--json` into the conversation** — it embeds base64 thumbnails for every side of every pair. Read `pairs/<pairId>/result.json`, or pull only `.totals`, `.pairs[].id`, `.pairs[].status`, `.pairs[].summary.counts` and `.pairs[].locators.changedLocators`. The console prints the target name, not the pair id; for a `run` pair the id is `<target>--<viewport>` with both parts lowercased and non-alphanumerics collapsed to `-`, and `.pairs[].id` in report.json is the reliable source.

## Do not

- Run `run` in a repo with no baselines without `--update` or asking first — it exits 1 by design and reads like a failure.
- Run it from anywhere but the config's directory without `--config`.
- Expect `networkidle` to settle on a page holding a websocket or long-poll: the 30s capture timeout turns it into an error pair and exit 2. Give that target a `capture: { waitFor: '<selector>' }`.
- Reach for this for a one-off "what is different between these two images" — that is **`/elastishot:compare-screenshots`** in this plugin.
- Use it for responsive reflow (desktop against mobile *layout*), canvas/WebGL/video pages, or anything a DOM diff answers better.
