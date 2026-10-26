# Elastishot plugin for Claude Code

Teaches Claude to check what a branch changed visually, and to compare two screenshots or two page URLs, and to tell you **which DOM elements changed**, not just where the red pixels are. It wraps the [`elastishot`](https://github.com/Osman19702/elastishot) CLI from npm. The plugin ships no code of its own, so it stays tiny and pulls the CLI on demand with `npx`, pinned to the exact version it documents.

Elastishot aligns the two images first (feature matching plus a row-band alignment), so it still works when the viewport changed size, the page zoomed, or a section collapsed. Those are the cases where a plain pixel diff goes all-red from the first shifted row down.

**Runs in Claude Code.** The skills run a command-line tool, so they need a terminal with Node.js 20 or newer. In claude.ai chat or Cowork the skill text loads, and Claude says the check needs Claude Code instead of guessing a result.

## Install

From the Claude plugin directory: **Customize → Plugins → Discover** on claude.ai, or `/plugin directory` in Claude Code. Or from its own marketplace:

```
/plugin marketplace add Osman19702/claude-plugins
/plugin install elastishot@osman-plugins
```

Then restart Claude Code, or run `/reload-plugins`.

This repository is the plugin's source. The directory listing and the `elastishot` entry in the `osman-plugins` marketplace both point at it.

## Requirements

- **Claude Code**, with a terminal.
- **Node.js 20 or newer.** `npx elastishot@0.3.0` fetches the package on first use (about 16 MB, no native compile step).
- **Playwright, for pages.** Comparing two image files needs no browser. Capturing a page does:
  ```
  npm install -D playwright && npx playwright install chromium
  ```
- **Built against elastishot 0.3.0, pinned exactly.** The flags, defaults and exit codes below are that contract, which is why the skills invoke `npx elastishot@0.3.0` rather than bare `npx elastishot`. A newer elastishot reaches plugin users only through a new plugin version, never silently.

## What you get

Three skills. Claude picks one on its own from what you ask; you can also invoke them by name.

### `/elastishot:pr-visual-check`

What did this branch, pull request or uncommitted change do to the UI?
1. It checks the base commit out next to your working tree.
2. It serves and captures both.
3. It names the elements that changed.

There are no baselines to keep. It needs a git repository and a small `elastishot.config.mjs` listing the pages.

```
npx elastishot@0.3.0 pr
npx elastishot@0.3.0 pr --base origin/release
```

Ask Claude things like:

- "What did my branch change visually?"
- "This PR isn't supposed to change the UI. Check it."
- "Did my CSS refactor move anything on the other pages?"
- "Add a visual check to our pull requests."

In CI it pairs with the [elastishot-action](https://github.com/Osman19702/elastishot-action), which writes the table to the GitHub job summary.

### `/elastishot:compare-screenshots`

One-shot comparison of two images or two URLs. No baselines needed.

```
npx elastishot@0.3.0 compare before.png after.png
npx elastishot@0.3.0 compare https://staging.example.com/ https://example.com/ --full-page
```

Ask Claude things like:

- "Does staging still look like production?"
- "Here are two screenshots, what's different?"
- "Compare the design mock with what we built."

Each pair gets a line with a similarity score, the added/removed/changed/moved counts and the top changed locators (`#faq`, `[data-testid="cta"]`). There is also an HTML report with a slider / flip / blink / overlay / diff viewer: send the whole run folder, or add `--single-file` to inline every image into one page you can attach.

### `/elastishot:visual-baselines`

The config-driven suite: baselines per target and viewport for deployed pages, a CI gate, and approving an intentional redesign.

```
npx elastishot@0.3.0 run --update      # bootstrap the baselines
npx elastishot@0.3.0 run --junit       # the CI check
npx elastishot@0.3.0 approve home      # accept a redesign as the new reference
```

Ask Claude things like:

- "Set up visual regression tests for these four pages."
- "We bumped the design system: sweep every page."
- "The new header is intentional, make it the baseline."

## What it runs and fetches

The plugin ships no code and runs nothing on its own. Everything happens through the commands above, which Claude runs in your terminal:

- `npx elastishot@0.3.0` downloads the `elastishot` package from the public npm registry on first use and caches it; later runs do not download it again.
- For pages, Playwright launches a local Chromium that loads only the URLs you pass, or the pages `pr` serves on this machine. Installing Playwright and its browser is a separate, explicit step (see Requirements).
- `pr` runs git (a temporary worktree of the base commit, removed afterwards) and the install, build and serve commands in your project's config.
- The elastishot CLI has no telemetry and contacts only the URLs you pass it. Reports, baselines and run folders are written locally, under your project.
- No account, API key or credential is read or sent.

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | No differences above the threshold |
| 1 | Differences found (or, for `pr`, a page the base does not have): the reports are still written |
| 2 | Usage or runtime error (missing Playwright, unreadable input, capture timeout, a base that could not be served) |

## Developing

Load the plugin straight from a checkout, without installing it:

```
claude --plugin-dir .
```

Validate before publishing, with the same check the community review pipeline runs:

```
claude plugin validate . --strict
```

## Links

- Source project: <https://github.com/Osman19702/elastishot>
- npm package: <https://www.npmjs.com/package/elastishot>
- GitHub Action: <https://github.com/Osman19702/elastishot-action>
- Marketplace that lists it (`osman-plugins`): <https://github.com/Osman19702/claude-plugins>

## Licence

MIT, same as the project it wraps.
