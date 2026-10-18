# Spitfire load test — GitHub Action

Run a load test saved on your self-hosted [Spitfire](https://usespitfire.com) as a performance gate: the step
fails when a threshold fails, the pull request gets a summary comment (updated on later pushes), and the job
summary shows the per-step results. After a deploy, the same action tells Spitfire's synthetic monitors to
keep a close watch on it.

```yaml
name: Load test
on: [pull_request]
permissions:
  contents: read
  pull-requests: write   # the summary comment
jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: usespitfire/spitfire-action@v1
        with:
          url: https://spitfire.example.com
          token: ${{ secrets.SPITFIRE_TOKEN }}
          test: "Shop: log in, browse, add to cart, check out"
          environment: staging
```

The test runs on your Spitfire runners, not on the GitHub runner: the action starts it, waits for the
result and reports it. The GitHub runner only needs to reach the controller's URL (a self-hosted GitHub
runner inside your network works when the controller is not public).

## Announcing a deploy

Add `deploy: true` at the end of your deploy job: the synthetic monitors it touches check at once and then
every minute for 30 minutes, and the monitor page gives each deploy a verdict against the day before (clean,
slower, failing more) with a marker on the trend. Without `test`, the action only announces the deploy.

```yaml
      - uses: usespitfire/spitfire-action@v1
        with:
          url: https://spitfire.example.com
          token: ${{ secrets.SPITFIRE_TOKEN }}
          deploy: true
          environment: prod          # the monitors of this environment; leave out for all of yours
          deploy-note: checkout service
```

With both `deploy: true` and `test`, the deploy is announced first, then the test runs.

## Planned maintenance

`maintenance: 60` puts the monitors in planned maintenance for 60 minutes (1–1440): their checks are skipped
(and left out of uptime, SLO and deploy verdicts) and nothing alerts. `maintenance: end` ends it and they
check at once. It runs before the deploy announcement, so a deploy job can quiet the monitors at the start and
end the maintenance together with `deploy: true` at the end. Needs Spitfire 0.51 or later.

```yaml
      - uses: usespitfire/spitfire-action@v1
        with:
          url: https://spitfire.example.com
          token: ${{ secrets.SPITFIRE_TOKEN }}
          maintenance: 30
          environment: prod
          maintenance-note: database migration
      # … the deploy …
      - uses: usespitfire/spitfire-action@v1
        if: always()
        with:
          url: https://spitfire.example.com
          token: ${{ secrets.SPITFIRE_TOKEN }}
          maintenance: end
          deploy: true
          environment: prod
```

## Setup

1. In Spitfire: user menu (top right) → **API tokens** → **Create token**.
2. In GitHub: **Settings → Secrets and variables → Actions** → `SPITFIRE_TOKEN`.
3. Add the workflow above. For the pull request comment the job needs `pull-requests: write`; the comment
   needs a Spitfire plan that includes it (otherwise the run still gates, without the comment).

The action downloads the CLI from your controller (so it always matches its version) and checks it against
the SHA-256 the controller sends.

## Inputs

| Input | Default | |
|---|---|---|
| `url` | required | The controller's URL |
| `token` | required | A personal API token (from a secret) |
| `test` | | The saved test's name or id (leave out to only announce a deploy or set a maintenance) |
| `environment` | | One of the test's environments, e.g. `staging`; with `deploy`, the monitors of this environment wake |
| `deploy` | `false` | `true` announces a deploy to the monitors first |
| `deploy-version` | `github.sha` | The version shown on the monitors' trend |
| `deploy-note` | | A short note, e.g. the service's name |
| `deploy-monitors` | | Monitor ids to wake or put in maintenance, comma or newline separated |
| `maintenance` | | Minutes (1–1440) of planned maintenance, or `end` (Spitfire 0.51+) |
| `maintenance-note` | `deploy-note` | Why, shown on the monitors' pages |
| `compare` | | Compare two environments at the same time: `A=prod,B=staging` (A the reference, B the candidate) |
| `runners` | `1` | Idle runners to use (ignored with `locations`) |
| `locations` | | One per line: `name=percent[:runners]`, e.g. `istanbul=60:2` |
| `scale` | | Multiply the load: `0.5`, `2` |
| `vars` | | Test variables, one per line: `name=value` |
| `confirm-writes` | `false` | Allow a test whose steps modify data |
| `pr-comment` | `auto` | `true`, `false`, or `auto` (on pull request events) |
| `github-token` | `github.token` | Token for the comment |
| `report` | | Write the run report: `report.pdf` or `report.html` |
| `report-lang` | `en` | `en` or `tr` |
| `summary` | `spitfire-summary.json` | The run summary JSON |
| `timeout` | | e.g. `45m` (default: the planned duration + 15m) |
| `fail-on-inconclusive` | `false` | With `compare`: fail on an inconclusive verdict |
| `fail` | `true` | `false` records the result without failing the step |
| `workspace` | | Workspace name or id |
| `insecure` | `false` | Skip TLS verification (a test controller only) |

## Outputs

| Output | |
|---|---|
| `exit-code` | `0` passed, `99` thresholds failed, `97` aborted by a threshold, `98` the candidate is worse, `1` error, `2` writes not confirmed |
| `result` | `passed`, `failed`, `aborted`, `worse` or `error` |
| `summary` | Path of the summary JSON |
| `deploy-id` | With `deploy`: the announced deploy's id |
| `deploy-woken` | With `deploy`: how many monitors went on close watch |
| `maintenance-until` | With `maintenance` minutes: when it ends (RFC 3339); empty when ended |

More examples: [`examples/`](examples) (a pull request gate with the report as an artifact, a release
comparison across two locations, a deploy announcement, a deploy in planned maintenance). Documentation: [CI/CD with Spitfire](https://usespitfire.com/docs/ci-cd).

Runners: Linux (x64, arm64) and Windows (x64).

## Testing the action

`test/local.py` runs the action's steps against a live Spitfire as GitHub would (inputs and defaults,
`GITHUB_OUTPUT`, `GITHUB_STEP_SUMMARY`, `GITHUB_PATH`): the CLI download and its checksum, a passing test,
a failing threshold with `fail` true and false, an unknown test, and a deploy announcement (alone, for an
environment, and the error when neither a test nor `deploy` is given), and planned maintenance (start with
the note, end together with a deploy, a bad value and over 1440 minutes refused).

## License

MIT. Spitfire itself is commercial software with a free edition: [spitfire.tr/en](https://usespitfire.com).
