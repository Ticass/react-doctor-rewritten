# react-doctor (Ticass fork)

A fork of [millionco/react-doctor](https://github.com/millionco/react-doctor) (originally [aidenybai/react-doctor](https://github.com/aidenybai/react-doctor)) with extra knobs aimed at running React Doctor as part of a CI / pull-request workflow.

The upstream tool scans a React codebase for security, performance, correctness, and architecture issues and produces a 0–100 health score. This fork keeps all of that behavior intact and adds the pieces that were missing to drive it cleanly from GitHub Actions: a clean-output mode, a programmatic API for the HTML report, and a PR-scoped scan that posts to its own comment thread.

The published package is renamed to **`react-doctor-rewritten`** to avoid colliding with upstream releases on npm.

## What this fork adds

### 1. Three new lint rules upstream missed

Three real React footguns weren't covered by upstream's rule set, so we added them to the bundled `react-doctor` oxlint plugin.

#### `react-doctor/no-direct-state-mutation` (error)

Flags in-place mutation of a `useState` value — both property assignments and the nine mutating array methods (`push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill`, `copyWithin`). Nested member chains are walked, so `state.nested.push(x)` is caught too.

```jsx
const [items, setItems] = useState([]);
items.push(newItem); // ✗ no-direct-state-mutation
items[0] = newItem; // ✗ no-direct-state-mutation
setItems([...items, newItem]); // ✓
```

#### `react-doctor/no-set-state-in-render` (error)

Flags setter calls executed during render — the classic infinite-loop bug. Uses a two-pass scan of the component body so a setter used before its `useState` declaration is still detected.

```jsx
function Profile() {
  const [name, setName] = useState("");
  setName("Alice"); // ✗ no-set-state-in-render — infinite loop
  return <h1>{name}</h1>;
}
```

#### `react-doctor/no-uncontrolled-input` (warning)

Catches three uncontrolled-input mistakes on `<input>`, `<textarea>`, and `<select>`:

- `value={...}` without `onChange` or `readOnly` (silently read-only field).
- `value` and `defaultValue` both set (`defaultValue` is ignored on controlled inputs).
- `value={state}` where `state` was initialized as `undefined` — the input starts uncontrolled and flips to controlled on first update, which React warns about at runtime but no static rule was catching.

```jsx
const [name, setName] = useState();      // initialized as undefined
return <input value={name} onChange={...} />;  // ✗ uncontrolled→controlled flip
```

These are registered in [`oxlint-config.ts`](packages/react-doctor-rewritten/src/oxlint-config.ts) and run as part of the standard scan — no opt-in required.

### 2. `--hide-branding` — clean HTML report for CI

The default CLI output is ASCII-framed and meant for terminals. With `--hide-branding`, React Doctor prints a GitHub-flavored HTML report instead — a score line, severity counts, and a collapsible diagnostics table — suitable for pasting straight into a PR comment.

```bash
npx react-doctor-rewritten . --hide-branding
```

```html
<h3>🟢 React Doctor — Full Repository Scan — Score: 84 / 100 — Great</h3>
<p>🚨 <strong>2</strong> errors · ⚠️ <strong>5</strong> warnings</p>
<details>
  <summary>📋 View details</summary>
  ...
</details>
```

### 3. `--hide-branding-pr` — PR-scoped scan with its own comment thread

Same clean-output behavior as `--hide-branding`, but the scan is automatically restricted to files changed on the current branch (vs the base branch). If your PR touches 4 React files, only those 4 files are linted.

```bash
npx react-doctor-rewritten . --hide-branding-pr
```

The two flags emit different HTML comment markers so a GitHub Action can locate and update each thread independently:

| Flag                 | Marker                              | Heading                             |
| -------------------- | ----------------------------------- | ----------------------------------- |
| `--hide-branding`    | `<!-- react-doctor-thread:full -->` | React Doctor — Full Repository Scan |
| `--hide-branding-pr` | `<!-- react-doctor-thread:pr -->`   | React Doctor — Pull Request Changes |

This means you can run both flags in the same workflow and get two persistent, independently-updated PR comments — one for the whole-repo health check, one for "did this PR introduce new issues?".

### 4. `buildNoBrandingReport` exported from the API

The HTML-report builder is exposed on `react-doctor-rewritten/api`, so you can render the same comment body programmatically without going through the CLI:

```js
import { diagnose, buildNoBrandingReport } from "react-doctor-rewritten/api";

const result = await diagnose("./my-app");
const html = buildNoBrandingReport(result.diagnostics, result.score); // full
const prHtml = buildNoBrandingReport(result.diagnostics, result.score, "pr"); // pr thread
```

The optional third argument selects the thread marker/heading.

### 5. Refactored GitHub Action

The composite action was reworked so that:

- `--diff` mode actually filters the scan (it was running a full scan on PRs in upstream).
- The PR-comment step finds the existing react-doctor comment by its marker and updates it in place instead of creating a new one on every run.
- `fail-on` is configurable from the action input so CI can be set to fail on errors only, on any warning, or never.

## Using both threads in GitHub Actions

A typical workflow uses one job for the full-repo scan and a second job for the PR-scoped scan. Each job runs the CLI directly and posts/updates its own comment using its thread marker.

```yaml
name: react-doctor

on:
  pull_request:

permissions:
  contents: read
  issues: write
  pull-requests: write

jobs:
  full-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v6
        with:
          node-version: 22

      - name: Run full scan
        run: npx -y react-doctor-rewritten@0.0.7 . --hide-branding > report.html

      - name: Upload report artifact
        uses: actions/upload-artifact@v4
        with:
          name: react-doctor-full-report
          path: report.html

      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const run = require('./.github/scripts/comment-report.cjs');
            await run({ github, context, core });
        env:
          MARKER: "<!-- react-doctor-thread:full -->"
          ARTIFACT_URL: "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"

  pr-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v6
        with:
          node-version: 22

      - name: Run PR scan
        run: npx -y react-doctor-rewritten@0.0.7 . --hide-branding-pr > report.html

      - name: Upload report artifact
        uses: actions/upload-artifact@v4
        with:
          name: react-doctor-pr-report
          path: report.html

      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const run = require('./.github/scripts/comment-report.cjs');
            await run({ github, context, core });
        env:
          MARKER: "<!-- react-doctor-thread:pr -->"
          ARTIFACT_URL: "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
```

```
const fs = require("fs");

const MAX_COMMENT_LENGTH = 60000;

module.exports = async function ({ github, context, core }) {
  const marker = process.env.MARKER;
  const path = process.env.REPORT_PATH || "report.html";
  const artifactUrl = process.env.ARTIFACT_URL;

  if (!marker) {
    throw new Error("MARKER env var is required");
  }

  if (!fs.existsSync(path)) {
    core.info(`${path} not found, skipping comment.`);
    return;
  }

  const report = fs.readFileSync(path, "utf8").trim();

  if (!report) {
    core.info(`${path} is empty, skipping comment.`);
    return;
  }

  let finalReport = report;

  if (report.length > MAX_COMMENT_LENGTH) {
    core.info("Report exceeds GitHub comment limit, truncating.");

    finalReport =
      report.slice(0, MAX_COMMENT_LENGTH) +
      "\n\n---\n⚠️ Report truncated due to size limits.";

    if (artifactUrl) {
      finalReport += `\n📎 Full report: ${artifactUrl}`;
    }
  }

  const body = `${marker}\n${finalReport}`;

  const comments = await github.paginate(
    github.rest.issues.listComments,
    {
      ...context.repo,
      issue_number: context.issue.number,
      per_page: 100,
    }
  );

  const prev = comments.find(c => c.body?.includes(marker));

  if (prev) {
    await github.rest.issues.updateComment({
      ...context.repo,
      comment_id: prev.id,
      body,
    });
  } else {
    await github.rest.issues.createComment({
      ...context.repo,
      issue_number: context.issue.number,
      body,
    });
  }
};
```

`fetch-depth: 0` is required so `--hide-branding-pr` can compute the diff against the base branch.

## Install

```bash
npx -y react-doctor-rewritten@latest .
```

All upstream flags (`--no-lint`, `--no-dead-code`, `--diff`, `--score`, `--project`, `--fail-on`, `--fix`, etc.) work the same as documented in the [package README](packages/react-doctor-rewritten/README.md).

### CLI options added by this fork

| Flag                 | Behavior                                                                                                                             |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `--hide-branding`    | Suppress the ASCII branding and emit a clean HTML report tagged with the `full` thread marker. Scans the whole repo.                 |
| `--hide-branding-pr` | Same clean output, tagged with the `pr` thread marker, and forces diff mode so only files changed on the current branch are scanned. |

## Repository layout

```
packages/
  react-doctor-rewritten/   # the CLI + plugin (formerly `react-doctor`)
  website/                  # marketing/leaderboard site (unchanged from upstream)
action.yml                  # composite GitHub Action wrapper
```

## Development

```bash
git clone https://github.com/Ticass/react-doctor
cd react-doctor
pnpm install
pnpm -r run build
```

Run locally:

```bash
node packages/react-doctor-rewritten/dist/cli.js /path/to/your/react-project --hide-branding-pr
```

Run the test suite for the CLI package:

```bash
cd packages/react-doctor-rewritten
pnpm exec vitest run
```

## Credits

All of the actual scanning, scoring, and rule logic is the work of the upstream maintainers — see [millionco/react-doctor](https://github.com/millionco/react-doctor) and [aidenybai/react-doctor](https://github.com/aidenybai/react-doctor). This fork only adds the CI-ergonomics layer described above.

## License

MIT, same as upstream.
