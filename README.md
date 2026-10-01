# Elastishot plugin for Claude Code

Teaches Claude to compare two screenshots or two page URLs and tell you **which DOM elements changed**, not just where the red pixels are. It wraps the [`elastishot`](https://github.com/Osman19702/elastishot) CLI from npm — the plugin ships no code of its own, so it stays tiny and pulls the CLI on demand with `npx`, pinned to the version range it documents.

Elastishot aligns the two images first (feature matching plus a row-band alignment), so it still works when the viewport changed size, the page zoomed, or a section collapsed — the cases where a plain pixel diff goes all-red from the first shifted row down.

## Install

```
/plugin marketplace add Osman19702/claude-plugins
/plugin install elastishot@osman-plugins
```

Then restart Claude Code, or run `/reload-plugins`.

This repository is the plugin’s source. The `elastishot` entry in the `osman-plugins` marketplace points at it, so the install above fetches this repository.

## Requirements

- **Node.js 20 or newer.** `npx elastishot@^0.2.0` fetches the package on first use (about 13 MB, no native compile step).
- **Playwright, only for page URLs.** Comparing two image files needs no browser. Capturing a page does:
  ```
  npm install -D playwright && npx playwright install chromium
  ```
- **Built against elastishot 0.2.x.** The flags, defaults and exit codes below are that contract, which is why the skills invoke `npx elastishot@^0.2.0` rather than bare `npx elastishot`.

## What you get

Two skills. Claude picks one on its own from what you ask; you can also invoke them by name.

### `/elastishot:compare-screenshots`

One-shot comparison. No baselines needed — but a project config in the directory you run from still applies.

```
npx elastishot@^0.2.0 compare before.png after.png
npx elastishot@^0.2.0 compare https://staging.example.com/ https://example.com/ --full-page
```

Ask Claude things like:

- "I refactored the CSS on the pricing page — did anything else move?"
- "Does staging still look like production?"
- "Here are two screenshots, what's different?"
- "This PR isn't supposed to change the UI. Check it."

You get a line per pair with a similarity score, added/removed/changed/moved counts and the top changed locators (`#faq`, `[data-testid="cta"]`), plus an HTML report with a slider / flip / blink / overlay / diff viewer — send the whole run folder, or add `--single-file` to inline every image into one page you can attach.

### `/elastishot:visual-baselines`

The config-driven suite: baselines per target and viewport, a CI gate, and approving an intentional redesign.

```
npx elastishot@^0.2.0 run --update      # bootstrap the baselines
npx elastishot@^0.2.0 run --junit       # the CI check
npx elastishot@^0.2.0 approve home      # accept a redesign as the new reference
```

Ask Claude things like:

- "Set up visual regression tests for these four pages."
- "We bumped the design system — sweep every page."
- "Wire this into CI with JUnit output."
- "The new header is intentional, make it the baseline."

The skill carries the parts that are easy to get wrong: the config file must be in the directory you run from (there is no upward search), `run` takes its capture settings from the config rather than from CLI flags, and only pairs produced by `run` can be approved.

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | No differences above the threshold |
| 1 | Differences found — the reports are still written |
| 2 | Usage or runtime error (missing Playwright, unreadable input, capture timeout) |

That makes `npx elastishot@^0.2.0 run --junit` usable directly as a CI gate, once your app is being served at the target URLs.

## Developing

Load the plugin straight from a checkout, without installing it:

```
claude --plugin-dir .
```

Validate before publishing — the same check the community review pipeline runs:

```
claude plugin validate . --strict
```

## Links

- Source project: <https://github.com/Osman19702/elastishot>
- npm package: <https://www.npmjs.com/package/elastishot>
- Marketplace that lists it (`osman-plugins`): <https://github.com/Osman19702/claude-plugins>

## Licence

MIT, same as the project it wraps.
