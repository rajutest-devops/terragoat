# DevSecOps Demo Runbook: Fail PR vs Pass PR

## Objective

Show two pull requests into `demo-main`:

1. Insecure branch fails hard gates.
2. Fixed branch passes and can be merged.

## Branches Used

- Failing branch: `demo/failing-insecure-terraform`
- Passing branch: `fixed-terraform-aws`
- Base branch: `demo-main`

## Pre-demo Checks (2 minutes)

Run from repo root:

```bash
cd /Users/rthirumalai/Documents/Devsecops/terragoat
git fetch origin
git branch --show-current
git branch --all --list '*demo*' '*fixed*'
```

Confirm branch protection on `demo-main` has required checks:

- DevSecOps Pipeline / Terraform Lint (aws)
- DevSecOps Pipeline / Checkov IaC Scan
- DevSecOps Pipeline / TFSec IaC Scan
- DevSecOps Pipeline / GitLeaks Secret Scan

## Part A: Show Failing PR

### A1) Ensure failing branch is up to date

```bash
git checkout demo/failing-insecure-terraform
git pull --ff-only origin demo/failing-insecure-terraform
```

### A2) Create PR to `demo-main`

Use GitHub UI:

- Base: `demo-main`
- Compare: `demo/failing-insecure-terraform`

Optional via CLI (if `gh` is installed):

```bash
gh pr create \
  --base demo-main \
  --head demo/failing-insecure-terraform \
  --title "Demo: failing insecure terraform" \
  --body "Intentional failing PR to demonstrate hard security gates."
```

### A3) What to show while checks run

In the PR checks tab, point out failures in one or more of:

- Checkov IaC Scan
- TFSec IaC Scan
- GitLeaks Secret Scan

In PR comment summary, show:

- Gate Decision = `YES`
- Hard fail checks and top finding IDs
- Merge blocked by required checks

## Part B: Show Passing Fixed PR

### B1) Switch to fixed branch

```bash
git checkout fixed-terraform-aws
git pull --ff-only origin fixed-terraform-aws
```

### B2) Create PR to `demo-main`

Use GitHub UI:

- Base: `demo-main`
- Compare: `fixed-terraform-aws`

Optional via CLI:

```bash
gh pr create \
  --base demo-main \
  --head fixed-terraform-aws \
  --title "Demo: fixed terraform passes security gates" \
  --body "Remediated Terraform + strict pipeline; expected to pass hard gates."
```

### B3) What to show while checks run

In the PR checks tab, show all required checks pass:

- Terraform Lint
- Checkov IaC Scan
- TFSec IaC Scan
- GitLeaks Secret Scan

In PR comment summary, show:

- Gate Decision = `NO`
- TFSec failed checks = `0`
- GitLeaks findings = `0` (current terraform scope)
- Merge is allowed

## Optional Validation Commands (Local)

From repo root:

```bash
# tfsec high/critical count
docker run --rm -v "$PWD":/src -w /src aquasec/tfsec:v1.28.5 --format json terraform/aws > /tmp/tfsec-demo.json || true
jq -r '[.results[]? | select((.severity // "" | ascii_upcase)=="HIGH" or (.severity // "" | ascii_upcase)=="CRITICAL")]|length' /tmp/tfsec-demo.json

# gitleaks current terraform files only
docker run --rm -v "$PWD":/repo -w /repo zricethezav/gitleaks:latest detect --source=terraform/aws --no-git --report-format sarif --report-path /tmp/gitleaks-demo.sarif --verbose || true
jq -r '[.runs[].results[]?] | length' /tmp/gitleaks-demo.sarif

# checkov total failed checks
docker run --rm -v "$PWD":/tf -w /tf bridgecrew/checkov:2.0.545 -d terraform/aws -o json --quiet > /tmp/checkov-demo.json || true
jq -r '[.[]?.results?.failed_checks[]?] | length' /tmp/checkov-demo.json
```

## Suggested Talk Track (Short)

1. "This PR intentionally contains insecure Terraform, and the pipeline blocks it with hard security gates."
2. "Now I open the remediated PR from fixed-terraform-aws to the same base branch."
3. "The same checks pass, the gate decision is clean, and merge is allowed."
4. "This demonstrates the pipeline is enforcing policy, not just generating reports."
