---
name: pr-visual-check
description: Checks what a branch, a pull request or uncommitted work changes visually. It serves the base commit and the working tree side by side, captures every configured page on both with the same browser, and names the DOM elements that changed, with no baselines to keep. Use when the user asks whether their branch, PR or CSS change altered the UI, wants a visual check before merging, or wants that check in CI with a GitHub job summary. Needs a git repository and an elastishot config listing the pages; runs a command-line tool, so it needs a terminal with Node.js 20+ (Claude Code).
when_to_use: Trigger phrases include "what did my branch change visually", "check this PR's UI", "did my CSS change break any page", "compare with main", "visual check before merging", "visual diff of this pull request", "this PR should not change anything visually", "add a visual check to CI". Not for two given screenshots or two arbitrary URLs (use /elastishot:compare-screenshots), nor for baselines kept across deployments (use /elastishot:visual-baselines).
---

# Check what a branch changed visually

`npx elastishot@0.3.0 pr` compares the working tree with a base commit.
1. It checks the base out into a temporary git worktree.
2. It prepares and serves that worktree next to the working tree.
3. It captures every target on both sides in one run, with one browser.
4. It reports what changed and names the elements behind each change.

Nothing is committed, and font or browser differences between machines cannot creep in, because both sides are captured here.

## Prerequisites

- `node --version` is 20 or newer. Without a shell, say plainly that this needs Claude Code; do not guess the visual result.
- A git repository with the base commit available locally.
- Playwright and Chromium, always, since both sides are page captures:
  `npm install -D playwright && npx playwright install chromium`
- An `elastishot.config.mjs` in the project (the nearest one from the current folder up) listing the pages as paths:

```js
export default {
  viewports: [{ name: 'desktop', width: 1280, height: 800 }, { name: 'mobile', width: 390, height: 844 }],
  capture: { fullPage: true },
  targets: [
    { name: 'home', url: '/' },
    { name: 'pricing', url: '/pricing' },
  ],
}
```

**A static site needs nothing more:** the config folder is served as it is. **A site with a build step needs a `pr` block:**

```js
pr: {
  install: 'npm ci',                    // a string runs in the base worktree only
  build: 'npm run build',               // a string runs in both trees
  serve: { dir: 'dist', spa: true },    // or a server command:
  // serve: { command: 'npx vite preview --port {port} --strictPort', ready: { log: 'Local:' } },
},
```

**A server command must listen on `{port}`** (or on `$PORT`), which elastishot picked and checked is free. Use `--strictPort` or its equivalent: a server that falls back to another port is never compared, and the run fails instead.

Existing configs with absolute URLs on one origin, such as `http://127.0.0.1:4174/`, work as they are; that origin is moved onto each side's port. Targets on other origins are an error under `pr`.

## Run it

```
npx elastishot@0.3.0 pr                     # against where this branch left origin/HEAD (or main, or master)
npx elastishot@0.3.0 pr --base origin/release
npx elastishot@0.3.0 pr home pricing        # only some targets
```

Progress goes to stderr: which base was chosen and why, then install, build and server output prefixed `[base]` and `[head]`. Stdout has one line per pair, a totals line and the report path. Capture flags (`--full-page`, `--wait-for`, `--hide`) apply to both sides. `--keep` leaves the worktree and its temporary folder for inspection.

Exit codes:

- **0**: no visual change. Say so plainly.
- **1**: the UI changed, or a page is new on the branch. The run folder holds the report.
- **2**:
  - git, install, build or server trouble;
  - a base that could not capture any target;
  - a capture error on the branch.

  stderr reads `elastishot: <message>`.

## Read the result

The arrow on each line lists the elements with the most pixel evidence. That is usually the answer to give the user, as locators: `[data-testid="cta"]`, `#faq`.

- **A `new` pair** is a page the base does not have: its reason reads `not on main@1a2b3c4: HTTP 404`. That is expected for a page the branch adds.
- **The run folder** has `summary.md` with the same table. In GitHub Actions it is also appended to the job summary.
- **Details:** read `<runDir>/pairs/<pairId>/result.json`. Do not `cat` report.json: it embeds image thumbnails. `report.json` → `context` names the base and head commits.
- **For the user to look at:** `<runDir>/index.html`, and per pair `pairs/<pairId>/report.html` with the slider, flip and overlay viewer.
- **No approve step:** `pr` keeps no baselines, so `approve` refuses pr runs.

## In CI (GitHub Actions)

```yaml
on: pull_request            # never pull_request_target: it would build fork code with your secrets
jobs:
  visual:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 2 }          # the base side of the merge commit
      - uses: actions/setup-node@v7
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - uses: Osman19702/elastishot-action@v0.3
```

The action installs Chromium, runs the check, uploads the report and links it from the job summary. A visual change is a notice unless `fail-on-diff: true` is set. Without the action: `npx playwright install --with-deps chromium`, then `npx elastishot@0.3.0 pr --junit --out .elastishot/pr`, then upload that folder.

## When something fails

- **`no default branch to compare against`:** pass `--base <ref>`, or fetch the default branch.
- **`nothing could be captured on <base>`:** the base cannot serve the pages.
  - A missing `pr.install` or `pr.build`: the base worktree has no `node_modules` and no build output.
  - Or the target paths are wrong.
- **`exited before its server was ready`:** read the output lines after the message. A missing dependency usually means `pr.install` is needed.
- **`nothing answered at … Does the command listen on {port}`:** the server ignored the port; add `{port}` and `--strictPort`.
- **In a pull request job, `fetch-depth: 2`:** the checkout was shallower than the merge commit's parents.
- **A worktree left behind by a killed run** is pruned by the next run (`git worktree prune` does it by hand).

## Do not

- Put a dependency install on the head side by hand: the working tree is the one running elastishot. `install` as a plain string runs in the base worktree only.
- Use it to compare two deployed environments: use `/elastishot:compare-screenshots` with two URLs, or baselines with `/elastishot:visual-baselines`.
- Read a high similarity as "no change" without looking at the locator list: one changed word in a price is a small region with a name.
