# security-action

Development Seed composite GitHub Action that gives any repo default security scanning with a single `uses:` line.

| Scanner | Default | What it checks |
|---------|---------|----------------|
| [zizmor](https://github.com/zizmorcore/zizmor) | ON | GitHub Actions workflow security |
| [osv-scanner](https://github.com/google/osv-scanner) | ON | Dependency vulnerabilities |
| [bandit + pip-audit](https://github.com/developmentseed/action-python-security-auditing) | OFF | Python-specific security (opt-in) |
| [OSSF Scorecard](https://github.com/ossf/scorecard) | OFF | Repository security posture (opt-in, dedicated workflow recommended) |

## Quick start

```yaml
name: Security

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write   # required for SARIF upload to Code Scanning

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      - uses: developmentseed/security-action@v0
```

That's it. zizmor and osv-scanner run automatically. Results appear in the repository's **Security → Code scanning** tab.

## Required permissions

| Permission | Why |
|------------|-----|
| `contents: read` | Required by all scanners to read the repository |
| `security-events: write` | Upload SARIF results to Code Scanning |
| `pull-requests: write` | Only needed when `python_audit_comment_on` is not `never` |
| `actions: read` | Required by Scorecard to evaluate workflow security posture |
| `id-token: write` | Scorecard OIDC token for publishing results to OSSF dashboard |

> The last two permissions are only needed when `enable_scorecard: 'true'`.

## All inputs

### Toggles

| Input | Default | Description |
|-------|---------|-------------|
| `enable_zizmor` | `'true'` | Run zizmor workflow linter (opt-out with `'false'`) |
| `enable_osv` | `'true'` | Run osv-scanner (opt-out with `'false'`) |
| `enable_python_audit` | `'false'` | Run bandit + pip-audit (opt-in with `'true'`) |
| `enable_scorecard` | `'false'` | Run OSSF Scorecard (opt-in with `'true'`) |

### zizmor

| Input | Default | Description |
|-------|---------|-------------|
| `zizmor_persona` | `'regular'` | `regular` / `pedantic` / `auditor` |
| `zizmor_min_severity` | `''` | Minimum severity to report: `low` / `medium` / `high` |
| `zizmor_min_confidence` | `''` | Minimum confidence to report: `low` / `medium` / `high` |
| `zizmor_online_audits` | `'true'` | Enable online audit checks |
| `zizmor_config` | `''` | Path to custom zizmor config file |

### osv-scanner

| Input | Default | Description |
|-------|---------|-------------|
| `osv_scan_args` | `--recursive\n./` | CLI args, one per line |
| `osv_fail_on_vuln` | `'true'` | Fail when vulnerabilities are found |
| `osv_upload_sarif` | `'true'` | Upload results to Code Scanning |
| `osv_results_file_name` | `'results.sarif'` | SARIF output filename |

### Python audit

| Input | Default | Description |
|-------|---------|-------------|
| `python_audit_tools` | `'bandit,pip-audit'` | Comma-separated tools to run |
| `python_audit_bandit_scan_dirs` | `'.'` | Comma-separated directories for bandit |
| `python_audit_bandit_severity_threshold` | `'high'` | Minimum blocking severity: `high` / `medium` / `low` |
| `python_audit_pip_audit_block_on` | `'fixable'` | Block on: `fixable` / `all` / `none` |
| `python_audit_package_manager` | `'requirements'` | `uv` / `pip` / `poetry` / `pipenv` / `requirements` |
| `python_audit_requirements_file` | `'requirements.txt'` | Requirements file path |
| `python_audit_comment_on` | `'never'` | Post PR comment: `never` / `blocking` / `always` |
| `python_audit_working_directory` | `'.'` | Working directory |
| `python_audit_debug` | `'false'` | Enable debug logging |

### Scorecard

| Input | Default | Description |
|-------|---------|-------------|
| `enable_scorecard` | `'false'` | Run OSSF Scorecard analysis (opt-in with `'true'`) |
| `scorecard_publish_results` | `'true'` | Publish results to OSSF dashboard (default-branch only) |

### Shared

| Input | Default | Description |
|-------|---------|-------------|
| `github_token` | `github.token` | Token for SARIF upload and PR comments |

## Examples

### Python project

```yaml
- uses: developmentseed/security-action@v0
  with:
    enable_python_audit: 'true'
    python_audit_package_manager: 'uv'
    python_audit_comment_on: 'blocking'
```

### Opt out of a scanner

```yaml
- uses: developmentseed/security-action@v0
  with:
    enable_osv: 'false'   # no dependency scanning in this repo
```

### Reduce noise (stricter thresholds)

```yaml
- uses: developmentseed/security-action@v0
  with:
    zizmor_min_severity: 'medium'
    zizmor_persona: 'auditor'
    python_audit_bandit_severity_threshold: 'medium'
    python_audit_pip_audit_block_on: 'all'
```

### OSSF Scorecard

Scorecard requires `id-token: write` and `actions: read` permissions that are broader than what a standard CI job should hold. The recommended pattern is to run it in its **own dedicated workflow** triggered on push to `main` and on a weekly schedule, rather than adding those permissions to your main CI job.

```yaml
# .github/workflows/scorecard.yml
name: Scorecard

on:
  push:
    branches: [main]
  schedule:
    - cron: "30 1 * * 6" # weekly on Saturdays

permissions:
  contents: read
  actions: read # required by Scorecard

jobs:
  scorecard:
    runs-on: ubuntu-latest
    permissions:
      security-events: write # upload SARIF to Code Scanning
      id-token: write         # required by Scorecard
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: developmentseed/security-action@v1
        with:
          enable_scorecard: 'true'
          enable_zizmor: 'false'  # already runs in CI
          enable_osv: 'false'     # already runs in CI
```

Then disable Scorecard in your main CI workflow to avoid the permission requirement there:

```yaml
# .github/workflows/ci.yml
- uses: developmentseed/security-action@v1
  with:
    enable_scorecard: 'false'
```

### Custom osv-scanner scope

```yaml
- uses: developmentseed/security-action@v0
  with:
    osv_scan_args: |
      --recursive
      --skip-git
      ./src
```

## How it works

The composite action routes your configuration to four independent sub-actions — [zizmor-action](https://github.com/zizmorcore/zizmor-action), [osv-scanner-action](https://github.com/google/osv-scanner-action), [action-python-security-auditing](https://github.com/developmentseed/action-python-security-auditing), and optionally [scorecard-action](https://github.com/ossf/scorecard-action) — each running with `continue-on-error: true` so all scanners complete regardless of individual failures. A final aggregation step checks each enabled scanner's outcome and fails the job if any enabled scanner found issues (subject to each scanner's own fail flag).
