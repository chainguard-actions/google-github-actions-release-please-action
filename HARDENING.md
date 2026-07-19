<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--release-please-action/v4.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--release-please-action/v4.1.1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references in workflow files are pinned to mutable tags or branch names instead of immutable 40-character SHA commit hashes, making them vulnerable to supply-chain attacks.

**.github/workflows/ci.yaml** (all three jobs):
- `uses: actions/checkout@v4`
- `uses: actions/setup-node@v4`

**.github/workflows/release-please.yaml**:
- `uses: actions/checkout@v4`
- `uses: actions/setup-node@v4`
- `uses: peter-evans/create-pull-request@v5`
- `uses: google-github-actions/release-please-action@main` (used twice — including a branch ref `@main`)

Locations:

- `.github/workflows/ci.yaml:14`
- `.github/workflows/ci.yaml:15`
- `.github/workflows/ci.yaml:30`
- `.github/workflows/ci.yaml:31`
- `.github/workflows/ci.yaml:39`
- `.github/workflows/ci.yaml:40`
- `.github/workflows/release-please.yaml:13`
- `.github/workflows/release-please.yaml:14`
- `.github/workflows/release-please.yaml:42`
- `.github/workflows/release-please.yaml:58`
- `.github/workflows/release-please.yaml:61`
- `.github/workflows/release-please.yaml:84`

### missing-permissions (severity: medium)

Neither `.github/workflows/ci.yaml` nor `.github/workflows/release-please.yaml` declares a top-level `permissions:` key, and no individual job within either file declares its own `permissions:` block. This means all jobs run with the default (potentially broad) repository permissions. Explicit minimal permissions should be declared at the top level or per-job.

Locations:

- `.github/workflows/ci.yaml:1`
- `.github/workflows/release-please.yaml:1`

### script-injection (severity: high)

Sub-rule (a): The `tag major and patch versions` run: block in `.github/workflows/release-please.yaml` directly interpolates GitHub Actions expressions into shell commands. Specifically:
- `${{ secrets.GITHUB_TOKEN}}` is embedded directly in a `git remote add` URL string
- `${{ steps.release.outputs.major }}` and `${{ steps.release.outputs.minor }}` are interpolated directly into `git tag` and `git push` commands

Any `${{ ... }}` expression inside a `run:` block is substituted by the YAML template engine before the shell ever sees the string, allowing shell metacharacters in the expanded value to be interpreted. These values should be passed via `env:` variables and then double-quoted in the shell script.

Locations:

- `.github/workflows/release-please.yaml:64`
- `.github/workflows/release-please.yaml:65`
- `.github/workflows/release-please.yaml:66`
- `.github/workflows/release-please.yaml:67`
- `.github/workflows/release-please.yaml:68`
- `.github/workflows/release-please.yaml:69`
- `.github/workflows/release-please.yaml:70`
- `.github/workflows/release-please.yaml:71`

### github-env-injection (severity: high)

The `commit` step in `.github/workflows/release-please.yaml` writes unsanitized values to `$GITHUB_ENV` without applying the required sanitization step (`printf '%s' ... | tr -d '\n\r'`):

1. `echo "CURRENT_HASH=${CURRENT_HASH}" >> $GITHUB_ENV` — CURRENT_HASH comes from `git rev-parse HEAD`, which is generally safe but still unsanitized.
2. `echo "LAST_BUILD_HASH=${LAST_BUILD_HASH}" >> $GITHUB_ENV` — similarly unsanitized.
3. `echo 'DIFF<<EOF' >> $GITHUB_ENV` / `echo "${DIFF}" >> $GITHUB_ENV` / `echo 'EOF' >> $GITHUB_ENV` — **critically**, DIFF is populated from `git log ... --oneline`, which includes commit messages that can be controlled by any contributor or pull request author. A commit message containing a newline followed by `KEY=malicious_value` would inject an arbitrary environment variable into subsequent steps via the heredoc-style `DIFF<<EOF` multiline write pattern.

Locations:

- `.github/workflows/release-please.yaml:40`
- `.github/workflows/release-please.yaml:41`
- `.github/workflows/release-please.yaml:42`
- `.github/workflows/release-please.yaml:43`
- `.github/workflows/release-please.yaml:44`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings across .github/workflows/ci.yaml and .github/workflows/release-please.yaml:

1. **unpinned-uses**: Pinned all action references to full SHA hashes with tag comments preserved: actions/checkout@v4→SHA, actions/setup-node@v4→SHA, peter-evans/create-pull-request@v5→SHA, google-github-actions/release-please-action@main→SHA (both occurrences).

2. **missing-permissions**: Added `permissions: {}` at the top level of both files. Per-job permissions added: ci.yaml jobs get `contents: read`; release-please.yaml jobs get `contents: write` and `pull-requests: write` (required for creating PRs, tags, and releases).

3. **script-injection**: Moved `${{ secrets.GITHUB_TOKEN }}`, `${{ steps.release.outputs.major }}`, and `${{ steps.release.outputs.minor }}` out of the `run:` block into an `env:` block as `GITHUB_TOKEN`, `RELEASE_MAJOR`, and `RELEASE_MINOR`. Shell script now references these as double-quoted `${VAR}` variables.

4. **github-env-injection**: Replaced the heredoc-style multiline DIFF write (which allowed commit message injection) with sanitized single-line writes using `printf '%s' "$VAR" | tr -d '\n\r'` for CURRENT_HASH, LAST_BUILD_HASH, and DIFF before writing to $GITHUB_ENV.

