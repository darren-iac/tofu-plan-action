# tofu-plan-action

Reusable GitHub Actions workflow that runs `tofu plan` for every workspace touched by a pull request and posts the diff as an idempotent PR comment per workspace.

Designed for the OpenTofu monorepo pattern where each `<tofu-root>/<workspace>/` directory is an independent workspace with its own backend.

## Why

`flux-iac/tofu-controller`'s [BranchPlanner](https://flux-iac.github.io/tofu-controller/branch-planner/branch-planner-getting-started/) does this in-cluster, but its long-lived pod design conflicts with Vault dynamic-secret lease lifecycle (the github plugin's child leases cascade-revoke when ESO's auth lease cycles, killing the token at GitHub's side). This action moves the work to ephemeral GHA runners that mint creds, plan, post, and exit before any cascade matters.

See the originating debug session: [darren-iac/iac docs/ROADMAP.md → BranchPlanner outcome](https://github.com/darren-iac/iac/blob/main/docs/ROADMAP.md).

## Usage

```yaml
# .github/workflows/tofu-plan.yaml in your monorepo
name: tofu-plan

on:
  pull_request:
    paths: ['tofu/**']
    types: [opened, synchronize, reopened]

permissions:
  pull-requests: write
  contents: read

jobs:
  plan:
    uses: darren-iac/tofu-plan-action/.github/workflows/plan.yaml@v0
    with:
      tofu-root: tofu/                    # default
      runner: arc-runner-darren-iac       # any self-hosted runner with creds
      tofu-version: latest                # default
```

## Caller contract

- **Permissions** — the calling workflow must grant `pull-requests: write` so the comment can be posted via the default `GITHUB_TOKEN`.
- **Credentials** — `tofu plan` itself needs cloud-provider creds (AWS, GitHub provider tokens, Cloudflare, etc). This action does **not** mint those — it expects them to already be available on the runner. For an in-cluster ARC runner with vault-agent annotations, the runner has `/vault/secrets/aws-credentials`, `/vault/secrets/github-token`, etc. mounted at startup and the workspace's `providers.tf` reads from those paths. See [darren-iac/iac/tofu/coder/providers.tf](https://github.com/darren-iac/iac/blob/main/tofu/coder/providers.tf) for an example.
- **Backend access** — the runner needs network access to the workspace's S3/DynamoDB backend (or whatever backend is configured).

## Behavior

1. **detect** job (always on `ubuntu-latest`) diffs the PR head against base, extracts the first path component under `tofu-root` for every changed file, deduplicates. Skips the plan job if no workspaces changed.
2. **plan** job (matrix per changed workspace, on `runner`) runs `tofu init && tofu plan -detailed-exitcode` in each `<tofu-root>/<workspace>/` directory.
3. **comment** step looks for a marker-comment on the PR for that workspace; updates in-place if found, posts new otherwise. Output is wrapped in a collapsible `<details>` block with a syntax-highlighted code fence.

Marker format: `<!-- tofu-plan-action: <workspace> -->` — one comment per workspace per PR.

## Comment status indicators

| Emoji | Meaning |
|---|---|
| ✅ | No changes (`tofu plan` exit 0) |
| 📝 | Changes detected (`tofu plan` exit 2) |
| ❌ | Plan failed (any other exit) — the workflow step also fails so branch protection can block the merge |

## Output trimming

GitHub PR comments cap at ~65kB. Plans larger than ~58kB are tail-trimmed with a leading note. Trimming preserves the most-recent (and usually most informative) part of the output.

## Versioning

Tags follow `v0`, `v1`, etc. with `vN` floating to the latest patch. Pin to `@v0` for current behavior; we'll update the major when behavior changes.

## License

Apache-2.0.
