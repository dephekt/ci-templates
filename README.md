# ci-templates

Reusable GitHub Actions workflows shared across `dephekt` repos. Each repo opts
in with a small **caller** workflow that references a template here at `@main`.

Templates live in [`.github/workflows/`](.github/workflows/) and are keyed on
`on: workflow_call`, so they don't run on their own — a caller in the consuming
repo defines the trigger and delegates to them.

## Available workflows

| Template | Purpose |
|----------|---------|
| [`docker-build.yml`](.github/workflows/docker-build.yml) | Build a Docker image, derive tags from the upstream `FROM` version, and push to GHCR. |
| [`claude-code-review.yml`](.github/workflows/claude-code-review.yml) | Automatic Claude review on every pull request. |
| [`claude-assistant.yml`](.github/workflows/claude-assistant.yml) | Interactive `@claude` assistant for comments and issues. |

## Opting in

Add the caller file(s) below to `.github/workflows/` in your repo. Callers must
declare the `permissions:` the template needs (a called workflow can't escalate
its own token) and pass any required secrets explicitly.

### Docker build

```yaml
name: Build and Publish Image
on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

jobs:
  build:
    uses: dephekt/ci-templates/.github/workflows/docker-build.yml@main
    permissions:
      contents: read
      packages: write
    with:
      image_name: ghcr.io/dephekt/your-image
      upstream_image_search: docker.io/node
      upstream_name_short: node
```

### Claude PR review

Requires a repo secret `CLAUDE_CODE_OAUTH_TOKEN` (generate with
`claude setup-token`).

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  review:
    uses: dephekt/ci-templates/.github/workflows/claude-code-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      issues: read
      id-token: write
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Defaults reproduce the reference config (Opus 4.8 at `xhigh`, the `code-review`
plugin). Override per repo via `with:` — e.g. `model`, `effort`, `plugins`,
`prompt`, `allowed_tools`.

### Claude `@claude` assistant

Same `CLAUDE_CODE_OAUTH_TOKEN` secret. The template only acts when a comment,
review, or issue mentions `@claude`.

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    uses: dephekt/ci-templates/.github/workflows/claude-assistant.yml@main
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
      actions: read
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```
