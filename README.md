# agents-market/check-shared-actions-staleness

Lint consumer repos for stale pinned `web3eco/shared-actions` SHAs in `.github/workflows/*.yml`. Posts a PR comment (warning by default) when any pinned SHA is older than `threshold_days` (default 90).

## Why

When you SHA-pin a reusable workflow (`uses: org/repo/.github/workflows/x.yml@<sha>`) for supply-chain hygiene, the SHA **stops moving** even when the upstream workflow keeps getting bugfixes and security patches. The pin locks you to that moment in time — forever.

This action catches that drift automatically by:

1. Scanning every `*.yml` / `*.yaml` file under `.github/workflows/`.
2. Extracting every SHA-pinned reference to the target repo (default `web3eco/shared-actions`).
3. Calling the GitHub API to look up the commit date for each pinned SHA.
4. Computing age in days against the threshold (default 90).
5. Posting a PR comment (when run on `pull_request`) listing all stale pins + their ages + the latest `main` SHA so you can `sed`-bump them in one commit.

The action runs purely as an inline `github-script` step — no Docker, no `npm install`, no extra permissions beyond what you already grant to your CI.

## Usage

Drop this into any consumer workflow file (e.g. `.github/workflows/quality-checks.yml`):

```yaml
name: Quality Checks

on:
  pull_request:
    branches: [main, develop]
  workflow_dispatch:

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint shared-actions SHAs
        uses: agents-market/check-shared-actions-staleness@v1
        with:
          threshold_days: 90      # anything older than 90d is flagged (default)
          target_repo: web3eco/shared-actions  # (default)
          # fail_on_stale: false    # default — warn + comment, don't fail
```

To **block** merge on stale pins, set `fail_on_stale: 'true'`:

```yaml
      - name: Lint shared-actions SHAs
        uses: agents-market/check-shared-actions-staleness@v1
        with:
          threshold_days: 90
          fail_on_stale: 'true'
```

> **Tip:** run on `pull_request` to get the auto-PR-comment. Running on `push` still produces warnings in the Actions log, just no PR comment.

## Inputs

| Input | Default | Required | Description |
| ----- | ------- | -------- | ----------- |
| `threshold_days` | `'90'` | no | Any pinned SHA whose commit is older than this many days is reported as stale. Must be a non-negative integer. |
| `target_repo` | `'web3eco/shared-actions'` | no | Target repo whose SHAs we lint against. Use the form `owner/name`. The action builds its own regex from this value, so any repo with SHA-pinned reusable workflows works. |
| `fail_on_stale` | `'false'` | no | When `'true'`, the action calls `core.setFailed` and exits non-zero if any pin is stale. Default is warning-only — the workflow still passes, but a PR comment is posted so reviewers can ask for a bump. |

## Outputs

| Output | Type | Description |
| ------ | ---- | ----------- |
| `stale_count` | string (number) | Number of pinned references detected as stale. Empty / `"0"` when nothing was stale or no pins were found. |
| `total_pinned` | string (number) | Total unique pinned references detected across all workflow files. |
| `latest_main_sha` | string (sha) | Latest commit SHA on the `main` branch of the target repo. Empty string on fetch error (e.g. private repo with insufficient token scope). |

All outputs are always set — you can branch downstream steps on `steps.<id>.outputs.stale_count`.

## Auth setup

The action needs read access to the **target repo's** commit history (not just your own repo). The token used by `actions/github-script@v7` is your workflow's `GITHUB_TOKEN` by default. Three auth modes, pick the one that matches your setup:

### Case 1 — Public target repo (e.g. `web3eco/shared-actions` is public)

No extra setup needed. The default `GITHUB_TOKEN` granted to every workflow can read public repos' commits.

```yaml
      - uses: agents-market/check-shared-actions-staleness@v1
```

### Case 2 — Private target repo + shared org

If the target repo lives in the **same org** as your consumer repo and the org setting "Allow GitHub Actions to create and approve pull requests" + SSO is configured for shared `GITHUB_TOKEN` access, you also get this for free. The default token inherits `contents: read` across the org for public-private pairings configured at the org level.

If your org blocks cross-repo `GITHUB_TOKEN` access (a stricter security posture), you must mint a PAT. Continue to case 3.

### Case 3 — Private target repo, isolated org, or cross-org GitHub App

Mint a **fine-grained PAT** that can read the target repo's commits, then expose it as a secret:

1. Go to `https://github.com/settings/personal-access-tokens/new` (or org-level equivalent).
2. Repository access: select the target repo (`web3eco/shared-actions`) **or** all repos in the target org.
3. Permissions: **Contents: Read-only**. Nothing else needed.
4. Copy the token, then in the consumer repo go to **Settings → Secrets and variables → Actions → New repository secret**. Name it e.g. `CROSS_REPO_TOKEN` (PAT case) or `ACTIONS_BOT_TOKEN` (App case).
5. Pass it explicitly:

```yaml
      - name: Lint shared-actions SHAs
        uses: agents-market/check-shared-actions-staleness@v1
        env:
          GITHUB_TOKEN: ${{ secrets.CROSS_REPO_TOKEN }}   # or ACTIONS_BOT_TOKEN for a GitHub App installation token
        with:
          threshold_days: 90
```

The action uses `github.rest.repos.getCommit` via the octokit instance, which automatically picks up `GITHUB_TOKEN` from the env. No code changes needed.

> **Gotcha:** GitHub Apps installed on the consumer repo **do not** automatically read other orgs. If the target repo belongs to a different org, create a separate App installation there and pass the resulting installation token.

## How it works (the JS in the script block)

```text
glob.sync('.github/workflows/*.{yml,yaml}')       // discover workflow files
           ↓
regex match <owner>/<repo>/path@<40hex>           // extract pinned refs
           ↓
github.rest.repos.getCommit({ ref: <sha> })       // fetch committer.date
           ↓
floor((now - committer.date) / 86_400_000)         // compute age in days
           ↓
age > threshold_days  →  stale[]                  // bucket stale refs
           ↓
if pull_request: github.rest.issues.createComment  // post the PR comment
if fail_on_stale: core.setFailed                   // (optional) block
```

Each network call is wrapped in try/catch — a transient 5xx or missing SHA surfaces as a `core.warning` in the log without breaking the rest of the lint.

## Example output

### PR comment (auto-posted on `pull_request` runs)

> ## ⏰ Shared Actions Staleness Check
>
> Found **4** pinned reference(s) to `web3eco/shared-actions` older than **90 days**:
>
> - `.github/workflows/quality-checks.yml` pins `f326f6cb7622` (61 days old)
> - `.github/workflows/quality-checks.yml` pins `f326f6cb7622` (61 days old)
> - `.github/workflows/quality-checks.yml` pins `f326f6cb7622` (61 days old)
> - `.github/workflows/quality-checks.yml` pins `f326f6cb7622` (61 days old)
>
> **Latest `main` SHA:** `de0dbbffe099` — consider bumping pinned references to absorb upstream bugfixes + security patches.
>
> _Auto-posted by [`agents-market/check-shared-actions-staleness@v1`](https://github.com/agents-market/check-shared-actions-staleness). Set `fail_on_stale: true` to block merge until SHAs are bumped._

> Note: the example above renders one file/pin per line. Real output groups same-SHA-different-paths together — duplicate file/pin pairs in the same workflow are deduped before the API call.

### Actions log (when not running on a PR, or when GH_TOKEN doesn't have `pull-requests: write`)

```
::warning::Found 4 stale web3eco/shared-actions reference(s) older than 90 days:
- .github/workflows/quality-checks.yml pins f326f6cb7622 (61 days old)
...
```

## License

MIT — see [LICENSE](./LICENSE).
