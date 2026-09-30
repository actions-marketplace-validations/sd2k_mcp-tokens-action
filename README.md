# mcp-tokens-action

GitHub Action to analyze token usage of MCP (Model Context Protocol) servers.

This action wraps [sd2k/mcp-tokens](https://github.com/sd2k/mcp-tokens), a CLI tool for analyzing the token footprint of MCP servers.

## Usage

### Basic analysis

```yaml
- name: Analyze MCP tokens
  uses: sd2k/mcp-tokens-action@v1
  with:
    command: npx my-mcp-server
    anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Token Counting Providers

The action supports two token counting providers:

- **anthropic** - Uses the Anthropic API for accurate token counts (requires API key)
- **tiktoken** - Uses the tiktoken library for offline approximation (no API key needed)

By default, the action auto-detects which provider to use:
- If `anthropic-api-key` is provided, uses Anthropic
- Otherwise, falls back to tiktoken

## Recommended Setup: Compare Against the PR's Base Commit

Build and count the PR's base commit in the same job, and use that as the baseline. This compares every PR against the branch it will actually merge into (including release or stacked branches), needs no separate baseline workflow or artifacts, and both sides are always counted by the same provider.

`.github/workflows/token-check.yml`:

```yaml
name: Token Analysis

on:
  pull_request:

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write # only needed for `comment: true`
    steps:
      - uses: actions/checkout@v4

      - name: Build server
        run: make build

      - name: Check out base
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.base.sha }}
          path: base

      - name: Build base server
        working-directory: base
        run: make build

      - name: Count base tokens
        uses: sd2k/mcp-tokens-action@v1
        with:
          command: ./base/dist/my-server
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          output: base-tokens.json

      - name: Analyze tokens
        uses: sd2k/mcp-tokens-action@v1
        with:
          command: ./dist/my-server
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          baseline: base-tokens.json
          threshold-percent: "5"
          comment: true
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

Both steps get the same key, so they use the same provider:
- Main repo PRs with an API key: Anthropic vs Anthropic
- Fork PRs, where secrets are unavailable: tiktoken vs tiktoken

Fork PRs also get a read-only `GITHUB_TOKEN`, so `comment: true` can't post on them. If fork contributors matter to you, write the action's outputs to `$GITHUB_STEP_SUMMARY` instead.

### Stored baselines

If you'd rather store a baseline (e.g. as an artifact from `main`) than rebuild the base on every PR, generate it with `all-providers: true`. The file then holds counts from both providers, so a PR is always compared against the provider it's using. Be aware that looking up "the latest artifact on main" through GitHub's API can occasionally return an old run, and it always compares against `main` whatever the PR's base branch is.

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `command` | Command to start the MCP server | Yes | - |
| `anthropic-api-key` | Anthropic API key for accurate counting | No | - |
| `baseline` | Path to baseline JSON file | No | - |
| `threshold-percent` | Max allowed % increase | No | `5` |
| `threshold-absolute` | Max allowed absolute increase | No | - |
| `all-providers` | Generate baseline with all providers | No | `false` |
| `provider` | Token counter (`anthropic` or `tiktoken`) | No | auto-detect |
| `model` | Model for token counting | No | - |
| `output` | Path to save report JSON | No | - |
| `timeout` | Server startup timeout (seconds) | No | `30` |
| `version` | Version of mcp-tokens to use | No | `latest` |
| `comment` | Post a PR comment with results | No | `false` |
| `comment-on-pass` | Post comment even when check passes | No | `false` |
| `max-tool-changes` | Max tool changes to show in comment (0 for all) | No | `5` |
| `github-token` | GitHub token for PR comments | No | - |

## Outputs

| Output | Description |
|--------|-------------|
| `total-tokens` | Total token count |
| `tool-tokens` | Token count for tools only |
| `provider` | Token counting provider used |
| `diff` | Token difference from baseline |
| `diff-percent` | Percentage difference |
| `passed` | Whether threshold check passed |
| `failure-reason` | Human-readable failure reason if threshold exceeded |
| `baseline-tokens` | Token count from baseline (for comparison) |
| `tool-changes` | JSON array of tool changes (added/removed/modified with diffs) |
| `tool-changes-count` | Number of tools that changed |

## PR Comments

The action can automatically post a comment on PRs when the threshold check fails (or always, if configured). This shows which tools caused the token increase:

```yaml
jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - name: Analyze tokens
        uses: sd2k/mcp-tokens-action@v1
        with:
          command: ./server
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          baseline: baseline/token-baseline.json
          threshold-percent: "5"
          comment: true
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

**Note:** The job needs `permissions: pull-requests: write` for the `GITHUB_TOKEN` to post comments.

By default, comments are only posted when the check fails. Use `comment-on-pass: true` to always post a comment.

The `max-tool-changes` input controls how many tool changes are shown (default: 5, sorted by largest impact first). Set to `0` to show all changes.

When posting a new comment, the action automatically minimizes (hides as "outdated") any previous MCP Token Analysis comments on the same PR to reduce noise.

This produces a comment like:

> ## MCP Token Analysis
>
> ❌ **Failed**
>
> | Metric | Value |
> |--------|-------|
> | Baseline | 1000 tokens |
> | Current | 1150 tokens |
> | Change | +150 (+15.0%) |
>
> **Reason:** Token increase of 15.0% exceeds threshold of 5.0%
>
> ### Tool Changes
>
> | Tool | Change | Tokens |
> |------|--------|--------|
> | `newTool` | 🆕 Added | +100 |
> | `existingTool` | 📝 Modified | +50 |

## License

Licensed under the Apache License, Version 2.0 or the MIT license, at your option.
