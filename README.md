# tofu-plan-action

Composite GitHub Action that runs `tofu plan` for every workspace touched by a pull request and posts the diff as an idempotent PR comment per workspace.

Designed for the OpenTofu monorepo pattern where each `<tofu-root>/<workspace>/` directory is an independent workspace with its own backend.

## Why composite, not reusable workflow

The first cut of this action was a reusable workflow (`workflow_call`). It worked for detection + planning, but the credentials story didn't survive contact with reality: reusable workflows can't share env with the calling job, so the only way to get cloud creds into `tofu plan` was for the runner to have ambient creds — which most ARC runners don't. Switching to a composite action means callers can mint creds in pre-steps within the same job, share env with the action, and the action just runs `tofu`.

See [darren-iac/iac docs/ROADMAP.md → BranchPlanner outcome](https://github.com/darren-iac/iac/blob/main/docs/ROADMAP.md) for the upstream context (we tried in-cluster BranchPlanner first; ESO + Vault dynamic-secret cascade-revoke killed it).

## Usage

```yaml
# .github/workflows/tofu-plan.yaml in your monorepo
name: tofu-plan

on:
  pull_request:
    paths: ['tofu/**']
    types: [opened, synchronize, reopened]

permissions:
  pull-requests: write   # required: how the action posts comments
  contents: read

jobs:
  plan:
    runs-on: arc-runners-darren-iac
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # action diffs PR head against base — needs both

      # === YOUR CREDENTIAL MINT STEPS GO HERE ===
      # The action runs `tofu init` + `tofu plan` and expects:
      #   - cloud-provider creds in env (e.g. AWS_SHARED_CREDENTIALS_FILE)
      #   - github provider tokens (if any tofu workspace uses the github provider)
      #   - backend access (S3 + DynamoDB locking, GCS, etc)
      #
      # Mint these however your runner can — vault-agent annotations on
      # the runner pod, OIDC → cloud, OpenBao via composite action, etc.
      # The action stays neutral.

      - uses: darren-iac/tofu-plan-action@v1
        with:
          tofu-root: tofu/                # default
          # tofu-version: latest          # default
          # workspaces: ''                # default: auto-detect from PR diff
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `tofu-root` | `tofu/` | Root directory containing workspaces. Trailing slash optional. |
| `tofu-version` | `latest` | OpenTofu version (passed to `opentofu/setup-opentofu`). |
| `workspaces` | `''` (auto) | Comma-separated explicit list. Overrides PR-diff auto-detection. |
| `github-token` | `${{ github.token }}` | Token for posting PR comments. Default needs `pull-requests: write`. |

## Outputs

| Output | Description |
|---|---|
| `workspaces` | Newline-separated list of workspaces planned |
| `failed` | `true` if any workspace's plan exited with an unrecoverable error (init or plan crash, not "changes detected") |

## Behavior

1. Set up OpenTofu on the runner.
2. Diff the PR (or read explicit `workspaces` input) → list of workspace directories under `tofu-root/`.
3. For each workspace:
   - `cd <tofu-root>/<workspace>`
   - `tofu init` (skip if init fails — record the failure)
   - `tofu plan -detailed-exitcode -no-color`
   - Build a comment body:
     - Status emoji: ✅ no changes (exit 0), 📝 changes (exit 2), ❌ failed (other)
     - Plan output in a collapsible `<details>` block, syntax-highlighted, tail-trimmed if >58kB
     - Marker: `<!-- tofu-plan-action: <workspace> -->`
   - Find existing marker comment on the PR — update in place if found, post new otherwise.
4. Exit 1 (action fails) if any workspace had an unrecoverable error; otherwise exit 0.

## Status indicators

| Emoji | Meaning | tofu plan exit |
|---|---|---|
| ✅ | No changes | 0 |
| 📝 | Changes detected | 2 |
| ❌ | Plan failed | other |

A failure on one workspace doesn't skip planning the others — every changed workspace gets a comment.

## Versioning

- `v1` (current) — composite action, multi-workspace detection, idempotent comments.
- `v0` — initial reusable-workflow shape (deprecated; see "Why composite" above).

`vN` floats to the latest minor/patch on that major. Pin to `@v1` for current behavior.

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — common bugs hit while building / extending this action (composite-action `-e` defaults, heredoc closer indentation, GITHUB_* env collisions, multi-value `git config insteadOf`, lease cascade-revoke, etc.).

## License

Apache-2.0.
