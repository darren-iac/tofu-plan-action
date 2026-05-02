# Troubleshooting

Common bugs hit while building / extending this action. The first version of v1 needed eight iterations to land cleanly because composite-action behavior is full of small landmines — most of these are general GHA gotchas, not specific to tofu.

## "Plan generates output but no PR comment posted"

Symptom: workflow runs, `tofu plan` output appears in the log (including `Plan: 0 to add, 1 to change, 0 to destroy`), but no comment lands on the PR. Workflow concludes `failure` after only a few seconds.

**Root cause**: GitHub Actions composite-action steps default to `shell: bash` which is invoked with `-e -o pipefail`. This is the `shell:` flag at the *invocation* level — it fires BEFORE any `set` commands in your script can change it. So when `tofu plan` exits non-zero (which it does on `-detailed-exitcode` whenever there are changes, **and on any plan-time provider error**), the shell's `-e` kills the script before reaching the comment-posting code.

**Fix**: override `shell:` with explicit flags that don't include `-e`:

```yaml
runs:
  using: composite
  steps:
    - id: run
      # Don't use `shell: bash` — that gets `-e -o pipefail` injected.
      shell: 'bash --noprofile --norc -uo pipefail {0}'
      run: |
        ...
```

`set +e` inside the `run:` block doesn't help — by the time it runs, `-e` has already taken effect for any failing command.

## "Heredoc closer ignored, script silently swallows the rest of itself"

Symptom: a `cat <<EOF_X` heredoc is followed by what looks like a normal closing `EOF_X` line, but the rest of the script (curl POST, `exit "$STATUS"`, etc.) never runs. Script terminates with no diagnostic — just an immediate exit at end-of-input.

**Root cause**: Bash heredoc closing delimiters must appear at column 0 with no leading whitespace. YAML `run:` script blocks are always indented (the YAML parser strips one level of indentation when materializing the script, but everything BELOW the `run:` mapping key is still indented relative to that). So `        EOF_X` (with leading spaces) doesn't match — `cat` keeps reading until end-of-script.

**Fix**: don't use heredocs inside `run:`. Use a multi-line string instead:

```bash
PLAN_OUTPUT=$(cat /tmp/plan-final.txt)
BODY="$MARKER
### $EMOJI \`tofu plan\` — \`$WS\` — $SUMMARY

<details><summary>Plan output</summary>

\`\`\`hcl
$PLAN_OUTPUT
\`\`\`
</details>"
```

Or use `<<-EOF_X` with TAB indentation — but YAML uses spaces, so this is fragile.

## "GITHUB_PATH gets replaced with /home/runner/.../add_path_..."

Symptom: setting `env: GITHUB_PATH: github/darren-iac/token/branch-planner` and then using `${GITHUB_PATH}` in the script — the value at runtime is some weird `/home/runner/_work/_temp/_runner_file_commands/add_path_<uuid>` path. Subsequent `curl ${VAULT_ADDR}/v1/${GITHUB_PATH}` 404s or returns garbage.

**Root cause**: `GITHUB_PATH` is a built-in GitHub Actions environment variable — it points to a runner-side file used by the `actions/<action>@vN` runner machinery to communicate path-additions back to the workflow. Setting it via `env:` lets your value through but the runner overwrites it during actual step execution.

**Fix**: don't use `GITHUB_*` prefixed names for your own env vars. Pick a different name — `GH_TOKEN_PATH`, `BRANCH_PLANNER_PATH`, etc.

## "git config insteadOf rule didn't take effect"

Symptom: configured `git config --global url."https://..." .insteadOf "ssh://git@github.com/"` (and a second matching line for `git@github.com:`), but git still uses SSH and fails with `Host key verification failed`. Only ONE of the two rules takes effect.

**Root cause**: `url.<base>.insteadOf` is a multi-value config key. A second `git config --global url.X.insteadOf Y` call without `--add` *overwrites* the previous value rather than appending.

**Fix**: use `--add`:

```bash
git config --global --add url."https://x-access-token:${TOKEN}@github.com/".insteadOf "ssh://git@github.com/"
git config --global --add url."https://x-access-token:${TOKEN}@github.com/".insteadOf "git@github.com:"
```

You can sanity-check with `git config --global --get-all url.<base>.insteadOf`.

## "`tofu init` says 'No configuration files'"

Symptom: workspace detection picks the right-looking name (e.g. `slack`), but `tofu init` cd's into a directory containing no `.tf` files and bails with `Error: No configuration files`.

**Root cause**: detection logic took the first path component after `tofu-root/`. For a nested layout like `tofu/slack/sleepless_agent/main.tf`, the first component is `slack` — a directory containing only subdirectories, no actual workspace.

**Fix**: walk *up* from each changed file looking for `backend.tf` (or whatever you treat as the workspace marker). The action does this in the `detect` step.

## "OpenBao auth call fails immediately, JSON parse error"

Symptom: the auth-token mint script dies immediately with `json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)` — i.e. python tried to parse empty input.

**Root cause** (most likely): the OpenBao service URL is unreachable from the runner namespace. The active-leader Service (`openbao-active.openbao.svc.cluster.local`) is only reachable from inside the `openbao` namespace; cross-namespace callers need the round-robin Service (`openbao.openbao.svc.cluster.local`).

The active/passive split uses Endpoints and a label selector that doesn't admit traffic from external namespaces — possibly NetworkPolicy, possibly the active Service has no matching Endpoints from outside-pods' perspective.

**Fix**: use `openbao.openbao.svc.cluster.local:8200` from any non-openbao namespace. Same as the `mint-renovate-token` action under `darren-iac/.github`.

## "GitHub installation token is rejected with 401 even though it just minted"

Two flavors here.

**Flavor 1**: The token is short (40 chars total, `ghs_` + 36 alphanumeric). That's actually valid for tokens minted by the OpenBao github plugin — it's a real installation token, just short. The 401 is *not* about format.

**Flavor 2**: The token works for ~seconds then 401s. This is the **lease cascade-revoke** pattern documented in iac's `feedback_eso_dynamic_secrets_revoke` memory: ESO + the OpenBao github plugin causes Vault's lease GC to revoke the token at GitHub's side any time ESO's auth lease cycles, even though ESO itself sees `SecretSynced=True`. Symptom: a token tested from a debug pod (auth as openbao-admin) works; the same path read via ESO into a long-lived consumer's Secret 401s within minutes.

**Fix**: don't use ESO + the github plugin for long-lived consumers. For workflows that mint+use+exit (this action), the cascade-revoke is benign. For long-lived consumers, use a CronJob → Secret refresh pattern, or GHA OIDC, or VSO.

## Validation tip: a pristine test workspace

If you're hacking on the action and want a plan that produces a clean output (no external-API providers, no auth-time errors), drop a tiny tofu workspace that uses only the `local` and `random` providers:

```hcl
# tofu/_action-test/hello-world/main.tf
terraform {
  required_providers {
    local  = { source = "hashicorp/local",  version = "~> 2.0" }
    random = { source = "hashicorp/random", version = "~> 3.0" }
  }
}

resource "random_id" "demo" { byte_length = 8 }
resource "local_file" "demo" {
  filename = "/tmp/tofu-plan-action-demo-${random_id.demo.hex}.txt"
  content  = "validation probe"
}
```

A PR touching this directory will produce a 📝 `Plan: 2 to add` comment with no provider noise. Throwaway after the action's behavior is verified.
