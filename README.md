# Agent Guard Check

A GitHub Action that shows what each AI agent configured in your repository can do, and fails a
pull request that gives an agent more authority than it needs.

It reads Claude Code settings and subagents, `.mcp.json`, Codex and Gemini settings, Copilot
custom agents, plugin marketplaces and GitHub workflows that run agents (including GitHub Agentic
Workflows, `claude-code-action`, `codex-action` and `run-gemini-cli`). For each agent it reports
whether it can read private data, take in untrusted input, write, run code, send data out, spend
money or change its own instructions, and whether a person is asked first. On a pull request it
reports only what the change made worse, so existing findings do not block unrelated work.

It only reads files. It never runs code from the repository.

## Usage

```yaml
name: Agent config
on: [pull_request, merge_group]
permissions:
  contents: read
  pull-requests: write # for comment: true
  security-events: write # for sarif: true
jobs:
  check:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: ShieldaAI/agent-guard-check-action@v1
        with:
          comment: true
          sarif: true
```

Pull requests from forks get a read-only token, so `comment: true` cannot post there; the report
is still in the job summary. Version 1.6.2 fails the job in that case; later versions warn and go
on.

For a security check, pin the action to a full commit SHA instead of `@v1`, as GitHub recommends
for third-party actions: `uses: ShieldaAI/agent-guard-check-action@<commit> # v1.6.2`. Each
release tag names the scanner version it runs.

## What a run does

- Compares the pull request with its base commit (or, in a merge queue, the queue's base commit)
  and fails when the change adds a finding at or above `fail-on` (high by default).
- Writes the Markdown report to the job summary. With `comment: true`, keeps one pull-request
  comment up to date. With `sarif: true`, uploads findings to GitHub code scanning.
- Runs an exact version of the open-source scanner
  [`@shieldaai/agent-guard-check`](https://www.npmjs.com/package/@shieldaai/agent-guard-check)
  with npx, from a folder outside the checkout and with the public npm registry, so a
  `node_modules` folder or `.npmrc` committed in the pull request cannot replace the scanner.
- Sets up Node.js 22 with `actions/setup-node` to run the scanner. Steps after it in the same
  job see that Node.js; run this action in its own job, or set up your Node.js version again
  after it.
- With `github-rules: true` (the default), asks GitHub with the workflow's own token, read-only,
  whether the target branch requires code-owner review for agent configuration (rule AGC-GOV-004).
  Set it to `false` to make no GitHub API call.

A finding as the job log shows it (the job summary and the comment show the same in Markdown):

```text
  HIGH     AGC-PERM-003  .claude/settings.json:3
           "Bash(curl:*)" runs without asking and can reach the network or run arbitrary code.
           Fix: Keep this command behind approval, or narrow it to a fixed script that cannot take arbitrary arguments.
```

## Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `path` | `.` | Directory to scan, relative to the workspace. |
| `fail-on` | from `.agc.yaml`, else `high` | Fail at or above this severity: `critical`, `high`, `medium`, `low`, `info` or `none`. |
| `base` | the pull request's or merge queue's base commit | Git ref to compare with. Only what the change made worse is reported. |
| `baseline` | | Baseline file of accepted findings. |
| `agent-guard-files` | | Newline-separated Agent Guard records (`describe()` output, inventory snapshots, host manifests) to compare with. |
| `sarif` | `false` | Upload results to code scanning (needs `security-events: write`). |
| `comment` | `false` | Post or update one pull-request comment with the report (needs `pull-requests: write`). |
| `github-rules` | `true` | Check whether code-owner review is required for agent configuration. |
| `version` | the release's scanner version | Exact version of `@shieldaai/agent-guard-check` to run. |

## Outputs

| Output | Meaning |
| --- | --- |
| `exit-code` | `0` no findings at or above `fail-on`, `1` findings, `2` error. |
| `sarif-file` | Path of the SARIF report. |

## Configuration and accepted findings

Settings, ignores with a reason, and review records with an owner and a date live in `.agc.yaml`
in your repository. The change under review does not set its own gate: `.agc.yaml` and a baseline
inside the repository are read from the base commit. See the
[scanner documentation](https://github.com/ShieldaAI/agent-guard/tree/main/packages/check#readme)
and the [rule list](https://github.com/ShieldaAI/agent-guard/blob/main/packages/check/docs/rules.md).

## Source and support

This repository publishes the action to the GitHub Marketplace. Its source, tests and the scanner
live in [ShieldaAI/agent-guard](https://github.com/ShieldaAI/agent-guard/tree/main/packages/check/action);
open issues there. Report vulnerabilities through
[private vulnerability reporting](https://github.com/ShieldaAI/agent-guard/security/advisories/new).

Licensed under Apache-2.0. Made by [Shielda](https://shielda.ai).
