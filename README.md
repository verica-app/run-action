# Verica Run — GitHub Action

Run a [Verica](https://verica.app) eval from a pull request and **block the merge on the
result**. Thin wrapper over [`@verica-app/cli`](https://www.npmjs.com/package/@verica-app/cli):
it triggers the run, waits for the gate verdict, publishes a JUnit report, sets the job's
exit code, and posts a **PR comment** with the outcome.

The only secret you need is a Verica workspace token — provider keys stay in Verica (BYOK),
never in CI.

## Usage

```yaml
on: { pull_request: { paths: ['prompts/**'] } }

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: verica-app/run-action@v1
        with:
          token: ${{ secrets.VERICA_TOKEN }}
          manifest: .verica.yml
```

Mono-prompt (no manifest):

```yaml
      - uses: verica-app/run-action@v1
        with:
          token: ${{ secrets.VERICA_TOKEN }}
          eval: eval_8x2k9d
          prompt: prompts/support-agent.txt
          system-prompt: prompts/support-agent.system.txt   # optional — omit to inherit
          tools: prompts/support-agent.tools.json           # optional — omit to inherit
          model: gpt-4.1-mini
          baseline-ref: main
```

## Inputs

| Input           | Required | Notes                                                       |
| --------------- | -------- | ----------------------------------------------------------- |
| `token`         | yes      | Workspace API token (store as a secret).                    |
| `eval`          | —        | Eval id (mono-prompt).                                      |
| `manifest`      | —        | `.verica.yml` path (multi-prompt).                          |
| `prompt`        | —        | User-template file (mono-prompt). Omit to inherit.          |
| `system-prompt` | —        | System-prompt file (mono-prompt). Omit to inherit.          |
| `tools`         | —        | Tool-definitions JSON file (mono-prompt). Omit to inherit.  |
| `model`         | —        | Model to sample under.                                      |
| `threshold`     | —        | Override the gate minimum pass rate (0..1).                 |
| `baseline-ref`  | —        | No-regression baseline = last run on this ref.              |
| `reuse-if-unchanged` | —   | Reuse a recent completed run when the config is unchanged (default `false`). |
| `reuse-max-age` | —        | Max age in hours of a reusable run (default 24, max 720).   |
| `reuse-same-ref`| —        | Only reuse a run on the same git ref (default `false`).     |
| `junit`         | —        | JUnit output path (default `verica-results.xml`).           |
| `comment`       | —        | Post/update a PR comment (default `true`).                  |
| `base-url`      | —        | Dev/self-host override; defaults to the hosted API.         |
| `cli-version`   | —        | `@verica-app/cli` version to run (default `^0.1`).          |

The commit SHA, ref, and **repo URL** are auto-detected from the runner env
(`GITHUB_SHA` / `GITHUB_REF` / `GITHUB_SERVER_URL` / `GITHUB_REPOSITORY`), so the run's
SHA links back to the commit in the Verica UI. On a `pull_request` the action stamps the
**PR head** commit (not the ephemeral merge commit) so that link points at a real commit
on the branch.

**Prompt push is field-level.** `prompt`, `system-prompt`, and `tools` are
independent — pass only what you changed and every omitted field is inherited
from the eval's current prompt version (a new version is created only if the
merged content differs). For multi-prompt, set `systemPrompt:` / `tools:` per
entry in the `.verica.yml` manifest instead.

**Reuse is opt-in.** Set `reuse-if-unchanged: true` to skip re-execution when the
config (prompt + model + sampling + dataset + graders) matches a recent **completed**
run; the action returns that run's frozen verdict (marked ♻️ in the PR comment).
It's freshness-bounded (`reuse-max-age`, default 24h) because a reused verdict can't
see provider-side drift. It can't be combined with `threshold` / `baseline-ref`: a
cached verdict can't be recomputed for a new `threshold`, and no-regression compares
against a moving baseline (the last run on the ref), so it can never be a fresh check —
gate on either and you must run fresh. Leave reuse `false` (the default) to always run
fresh — usually what you want, since re-running is how an eval catches model drift.

## Exit codes

The job passes (`0`) or fails (`1`) based on the eval's pass condition (configured in Verica).
A transport/validation error is `2`. JUnit `<failure>`s are informational — the gate, not the
JUnit count, decides the job status.

## Permissions

To post the PR comment, the workflow needs:

```yaml
permissions:
  pull-requests: write
  contents: read
```

(Default on most repos; add it explicitly if your org locks permissions down.)

## How it relates to the CLI

Everything portable lives in the CLI — you can skip this Action and run
`npx @verica-app/cli run …` in any CI. This Action only adds the PR comment and a tidy
`uses:` interface. See the [CLI docs](https://www.npmjs.com/package/@verica-app/cli).

MIT licensed.
