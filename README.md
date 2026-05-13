# zrok security review action

Multi-agent security code review on pull requests, driven by [zrok] and [OpenCode].

Unlike single-LLM-call security review actions, this one runs **N specialized
agents per PR** — chosen by zrok based on your project's detected stack
(injection, SSRF, validation, config, dependencies, …). Findings are stable
across runs (sha256 fingerprints in SARIF `partialFingerprints`) so the same
issue stops re-reporting on every commit.

## Usage

```yaml
name: zrok security review
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write       # post review comment
  security-events: write     # upload SARIF to code-scanning

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0      # action needs base ref for diff
      - uses: diffsec/zrok-review-action@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          severity-threshold: high
          block-on: critical
```

## Inputs

| Input | Default | Notes |
|---|---|---|
| `anthropic-api-key` | — | **Required.** Used by OpenCode to call Claude. |
| `github-token` | — | **Required.** Usually `secrets.GITHUB_TOKEN`. |
| `base-ref` | PR target | Override to diff against a specific ref. |
| `zrok-ref` | `main` | Pin zrok to a specific git ref. |
| `opencode-version` | `latest` | npm version of `opencode-ai`. |
| `top-n` | `10` | Max findings inlined in the PR comment. Full list still goes to SARIF. |
| `severity-threshold` | `high` | Inline only findings at or above this severity in the PR comment. |
| `block-on` | `''` (off) | Fail the check if any finding ≥ this severity exists. Set to `critical` or `high`. |
| `comment-pr` | `true` | Post the rendered comment to the PR. |
| `upload-sarif` | `true` | Upload SARIF to code-scanning. |

## Outputs

| Output | Description |
|---|---|
| `findings-count` | Total findings scoped to the diff. |
| `sarif-path` | Path to the generated SARIF (uploaded as `zrok-review` artifact). |
| `comment-path` | Path to the generated PR comment markdown. |

## What gets posted

A single PR conversation comment with:

- **Summary header** — total + per-severity counts.
- **Top-N findings inlined** — file/line/function, what it is, why it matters,
  suggested fix, confidence, and which agent found it. `top-n` and
  `severity-threshold` gate this; everything else lands in SARIF.
- **Link to code-scanning** for the full list.

The same findings also land in GitHub's code-scanning UI via SARIF upload,
with `partialFingerprints["zrokFingerprint/v1"]` so that **the same issue
will not re-flag on subsequent PRs** as long as the CWE, file, function,
and normalized title are unchanged.

## Cost

This action does multi-agent review — typically 6–15 LLM calls per PR
depending on project classification (vs the 1 call most competing actions
make). Expect:

- Small PRs (≤5 changed files): ~$0.50–$2 per run with Claude Sonnet.
- Large PRs (50+ files): can climb past $10.

If cost is a concern, gate the workflow on label
(`if: contains(github.event.pull_request.labels.*.name, 'security-review')`)
or scope `base-ref` to limit the diff.

## Permissions reference

The action's OpenCode agents are configured to:

- **Deny** `edit`, `write`, `webfetch`.
- **Allow** `zrok *`, `git diff/log/show`, `rg`, `grep`, `find`, `ls`, `cat`,
  `head`, `tail`, `wc`. Other bash commands fall through to OpenCode's
  default `ask` (which auto-denies in headless mode).

This means a prompt-injection attempt in the reviewed code cannot trick
the model into modifying files, fetching URLs, or executing arbitrary
shell commands.

## How it works

1. Build zrok at the requested ref, install OpenCode from npm.
2. `zrok init` + `zrok review pr setup --runner opencode` —
   detects tech stack, classifies project, selects applicable agents,
   writes `.opencode/agent/<name>.md` for each subagent plus a
   `zrok-orchestrator` primary agent.
3. `opencode run --agent zrok-orchestrator` — OpenCode dispatches
   subagents through the recon → analysis → validation → review phases.
   Findings persist via the `zrok finding create` CLI.
4. `zrok review pr report --base $BASE` — filters findings to the diff,
   renders the PR comment and SARIF.
5. Upload SARIF to code-scanning, post comment via `gh api`, enforce
   `block-on` if configured.

[zrok]: https://github.com/diffsec/zrok
[OpenCode]: https://opencode.ai
