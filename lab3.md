# Module 1 — Lab Workshop: Pipeline Failure Diagnosis & AI-Assisted Fix

> **Course:** AI in CI/CD Pipelines
> **Duration:** 90 minutes
> **Level:** Intermediate
> **Platform:** GitHub Actions
> **Stack:** Python / FastAPI

---

## Table of Contents

1. [Lab Overview](#1-lab-overview)
2. [Theory Recap](#2-theory-recap)
3. [Scenario: The Broken Pipeline](#3-scenario-the-broken-pipeline)
4. [Lab Objectives](#4-lab-objectives)
5. [Prerequisites & Setup](#5-prerequisites--setup)
6. [Part 1 — Intelligent Dependency Resolution](#part-1--intelligent-dependency-resolution)
7. [Part 2 — AI Code Review Gate](#part-2--ai-code-review-gate)
8. [Part 3 — Test Selection & Flaky Test Fix](#part-3--test-selection--flaky-test-fix)
9. [Part 4 — Automated Deployment Rollback](#part-4--automated-deployment-rollback)
10. [Debrief & Discussion](#10-debrief--discussion)
11. [Rubric](#11-rubric)
12. [Reference](#12-reference)

---

## 1. Lab Overview

Traditional CI/CD pipelines are reactive — they fail, report the error, and wait for a human to diagnose and fix. This lab introduces a new workflow: pipelines that detect problems earlier, reason about the code semantically, prioritise which tests matter, and respond to failures automatically.

You will work on a pre-seeded broken pipeline for **`payments-api`**, a Python FastAPI service. Three deliberate faults have been introduced:

| Fault | Category | Where |
|---|---|---|
| `httpx` version pin conflicts with `fastapi 0.110` | Dependency | `pyproject.toml` |
| `test_payment_webhook` uses `time.sleep` causing random timeouts | Flaky test | `tests/test_webhooks.py` |
| Argo Rollout has no analysis template — bad canaries never roll back | Deploy config | `k8s/rollout.yaml` |

By the end of the lab you will have applied an AI-assisted fix to each fault and instrumented the pipeline so these categories of failure are caught automatically in future.

---

## 2. Theory Recap

Before starting the lab, review how the three AI capabilities you will implement map to the module theory.

### 2.1 AI at the Build Stage

The build stage is where dependency resolution and code review happen. AI adds two capabilities here:

**Intelligent dependency conflict detection** goes beyond version range arithmetic. Tools like Renovate group related packages so that `fastapi`, `httpx`, and `starlette` — which share API surface — always update together. This prevents the class of bug you are about to fix: a dependency pair that individually satisfies their declared ranges but breaks at runtime because one assumes an API the other has removed.

**Semantic code review gates** evaluate the *meaning* of a change, not just its syntax. A linter checks formatting; an LLM-based gate asks whether the change introduces a SQL injection risk, inadvertently removes an auth check, or adds hardcoded credentials. In Part 2 you will build one using the Anthropic API.

### 2.2 AI at the Test Stage

Running the full test suite on every commit is expensive and slow. **Change-impact analysis** uses the dependency graph of the codebase to determine which tests are actually exercised by the changed files. Launchable and similar tools record historical test runs, learn which tests are predictive of failure for a given change, and produce a minimal subset that gives you the same confidence in a fraction of the time.

**Flaky test identification** is a related problem. A test that uses real timers, network sockets, or shared mutable state will fail randomly under CI load. AI-assisted tooling can identify flaky tests by tracking their pass/fail variance over hundreds of runs and flagging them for quarantine or rewrite.

### 2.3 AI at the Deploy Stage

Progressive delivery splits traffic gradually between a canary release and the stable version. Without automation, a bad canary requires a human to notice rising error rates and trigger a rollback. **AI-driven canary analysis** continuously queries metrics — error rate, latency percentiles, saturation — and rolls back automatically when a statistical threshold is breached. Argo Rollouts implements this via `AnalysisTemplate` resources backed by Prometheus, Datadog, or custom providers.

---

## 3. Scenario: The Broken Pipeline

The `main` branch CI pipeline for `payments-api` is failing intermittently. On-call has escalated. Your task is to diagnose each of the three faults, apply the correct fix, and instrument the pipeline to prevent recurrence.

### Fault 1 — Dependency Conflict

`pyproject.toml` pins `httpx==0.24.0`. The project recently upgraded to `fastapi==0.110.0`, which requires `httpx>=0.27`. The install step completes without error because pip resolves the declared constraint, but the application fails at import time with:

```
ImportError: cannot import name 'AsyncClient' from 'httpx'
```

The fix requires relaxing the pin and grouping the FastAPI ecosystem packages in Renovate so future upgrades move them together.

### Fault 2 — Flaky Test

`test_payment_webhook` fires a webhook and then calls `time.sleep(0.1)` before asserting the result. Under load on a shared CI runner, the async callback sometimes takes longer than 100ms, causing random test failures that are hard to reproduce locally:

```
AssertionError: expected 'processed', got None
```

The fix requires rewriting the test using `pytest-asyncio` and `await` so it is deterministic regardless of system load.

### Fault 3 — Deploy Misconfiguration

The staging Argo Rollout uses `maxUnavailable: 100%` and has no `AnalysisTemplate` attached. This means:

- All pods are replaced at once with no safety net
- If the canary has a 50% error rate, it will promote to 100% traffic with no intervention

The fix requires attaching a Prometheus-backed `AnalysisTemplate` that measures the 5xx error rate and triggers automatic rollback after two consecutive failures.

---

## 4. Lab Objectives

By the end of this lab you will be able to:

- Fix a transitive dependency conflict and configure Renovate to group related packages
- Write a GitHub Actions workflow that calls an LLM API to perform semantic code review and block merges on HIGH severity findings
- Eliminate a flaky test by replacing timer-based synchronisation with proper async test patterns
- Configure Launchable (or an equivalent heuristic) to run only the tests impacted by a change
- Attach an Argo Rollouts `AnalysisTemplate` to a canary deployment and verify automatic rollback

---

## 5. Prerequisites & Setup

### Required Accounts and Tools

| Tool | Required | Notes |
|---|---|---|
| GitHub account | Yes | For Actions and secrets |
| Python 3.11+ | Yes | Local development |
| `kubectl` | Yes | Parts 4 only |
| Anthropic API key | Yes | Part 2 |
| Launchable token | Optional | Part 3 has a heuristic fallback |
| Argo Rollouts CLI | Yes for Part 4 | `brew install argoproj/tap/kubectl-argo-rollouts` |

### Step 1: Fork the Sample Repository

```bash
# Fork workshop-ai-cicd/payments-api on GitHub, then:
git clone https://github.com/YOUR_USERNAME/payments-api
cd payments-api
```

The repository structure is:

```
payments-api/
├── app/
│   ├── main.py
│   ├── routers/
│   │   └── payments.py
│   └── services/
│       └── webhook.py
├── tests/
│   ├── test_payments.py
│   └── test_webhooks.py
├── k8s/
│   ├── rollout.yaml
│   └── service.yaml
├── pyproject.toml
├── .github/
│   └── workflows/
│       └── ci.yml
└── renovate.json
```

### Step 2: Observe the Failing Baseline

Push an empty commit to trigger the baseline CI run before making any changes. Take note of which step fails and copy the error message — you will refer to it during the debrief.

```bash
git commit --allow-empty -m "chore: trigger baseline ci run"
git push
```

**Expected result:** The pipeline fails on the `install` step with an `ImportError`. The test stage may also show intermittent failures on `test_payment_webhook`.

### Step 3: Add Secrets

In your GitHub repository, go to **Settings → Secrets and variables → Actions** and add:

| Secret name | Value | Used in |
|---|---|---|
| `ANTHROPIC_API_KEY` | Your Anthropic API key | Part 2 |
| `LAUNCHABLE_TOKEN` | Your Launchable token | Part 3 (optional) |

---

## Part 1 — Intelligent Dependency Resolution

**Duration:** 20 minutes
**Goal:** Fix the `httpx` version conflict and configure Renovate to prevent this class of problem automatically.

### Background

The conflict here is not a declared incompatibility — both `fastapi==0.110.0` and `httpx==0.24.0` will install without error. The problem is a **runtime API break**: `fastapi 0.110` internally uses `httpx.AsyncClient`, which was added in `httpx 0.27`. Pinning to `0.24` means the API does not exist at import time.

This is the kind of conflict that syntactic version range analysis misses entirely. It requires knowing the changelog history of both packages, which is exactly the kind of multi-source reasoning that AI-assisted dependency tooling applies.

### Step 1.1 — Reproduce the Error Locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
python -c "from app.main import app"
```

Expected output:

```
ImportError: cannot import name 'AsyncClient' from 'httpx'
  (site-packages/httpx/__init__.py)
```

### Step 1.2 — Fix the Version Pin

Open `pyproject.toml` and update the `httpx` constraint:

```toml
# Before — too strict
[project.dependencies]
fastapi = ">=0.110,<1.0"
httpx = "==0.24.0"          # FAULT: this conflicts at runtime

# After — range that allows compatible versions
[project.dependencies]
fastapi = ">=0.110,<1.0"
httpx = ">=0.27,<1.0"       # FIXED
```

Regenerate and verify:

```bash
pip install -e ".[dev]"
python -c "from app.main import app"
echo "Import successful: $?"
```

### Step 1.3 — Configure Renovate to Group Related Packages

The root cause of this fault is that `fastapi` and `httpx` were updated independently. A properly configured Renovate rule groups them so they always move together in a single PR, which makes the incompatibility visible at review time rather than at runtime.

Create or replace `renovate.json` in the repository root:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "dependencyDashboard": true,
  "packageRules": [
    {
      "description": "Group FastAPI ecosystem — these packages share API surface and must upgrade together",
      "matchPackageNames": ["fastapi", "httpx", "starlette", "anyio"],
      "groupName": "fastapi ecosystem",
      "automerge": false,
      "addLabels": ["dependencies", "needs-review"],
      "stabilityDays": 3
    },
    {
      "description": "Auto-merge safe patch updates for test/lint tooling",
      "matchPackageNames": ["pytest", "pytest-asyncio", "black", "ruff", "mypy"],
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "description": "Always require human review for major version bumps",
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "addLabels": ["major-update", "needs-human-approval"]
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security", "priority-high"]
  },
  "semanticCommits": "enabled",
  "prConcurrentLimit": 5,
  "prHourlyLimit": 2
}
```

> **Note on the `ai` config key:** Earlier versions of the workshop referenced an `ai` block in `renovate.json`. This key does not exist in any released version of Renovate. The `groupName` and `stabilityDays` rules above achieve the same goal using real, supported configuration. Renovate ≥ 37 is required.

### Step 1.4 — Trigger a Renovate Dry Run

```bash
# Requires Renovate >= 37 installed globally
LOG_LEVEL=debug renovate \
  --dry-run=full \
  --platform=github \
  --token=$GITHUB_TOKEN \
  YOUR_USERNAME/payments-api 2>&1 | grep -E "(groupName|packageRules|conflict|WARN|INFO)"
```

Look for log lines showing the `fastapi ecosystem` group being applied and any packages that Renovate has detected as outdated.

### Knowledge Check — Part 1

1. Why did `pip install` succeed even though there was a runtime incompatibility?
2. What is `stabilityDays` in Renovate and why is it useful for the FastAPI ecosystem group?
3. What would happen if you set `automerge: true` on the FastAPI ecosystem group rule?

---

## Part 2 — AI Code Review Gate

**Duration:** 25 minutes
**Goal:** Build a GitHub Actions workflow that calls the Anthropic API to perform semantic code review on every pull request and blocks merge on HIGH severity findings.

### Background

Linters and static analysis tools check *syntax* — formatting, type annotations, undefined variables. They cannot answer the question: *does this change introduce a security vulnerability?*

An LLM-based review gate reads the diff as a human reviewer would, understanding the intent of the change and reasoning about its security implications. The gate you build here looks for the OWASP Top 10 patterns most relevant to a payments API: SQL injection, hardcoded credentials, missing authentication checks, and unvalidated user input flowing into sensitive operations.

### Step 2.1 — Create the Review Workflow

Create `.github/workflows/ai-review.yml`:

```yaml
name: AI Code Review Gate

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install Anthropic SDK
        run: pip install anthropic

      - name: Extract PR diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD \
            -- '*.py' > /tmp/pr_diff.txt
          echo "Diff size: $(wc -l < /tmp/pr_diff.txt) lines"

      - name: Run AI code review
        id: ai-review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          REPO: ${{ github.repository }}
        run: python .github/scripts/llm_review.py

      - name: Post review summary to PR
        if: always()
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          if [ -f /tmp/review_summary.txt ]; then
            gh pr comment ${{ github.event.pull_request.number }} \
              --body-file /tmp/review_summary.txt
          fi
```

### Step 2.2 — Write the Review Script

Create `.github/scripts/llm_review.py`:

```python
import anthropic
import json
import os
import sys

SYSTEM_PROMPT = """You are a senior security engineer reviewing a Python code diff
for a payments API. Your job is to identify security vulnerabilities and logic errors
in the changed code only — do not comment on unchanged context lines.

Evaluate the diff for:
- SQL injection via string formatting or concatenation in queries
- Hardcoded credentials, API keys, or secrets
- Missing authentication or authorisation checks on endpoints
- Unvalidated user input passed to exec, eval, subprocess, or file operations
- Insecure direct object references (IDOR)
- Missing input length or type validation

Respond with a single JSON object. No preamble, no markdown fences.
Schema:
{
  "severity": "LOW | MEDIUM | HIGH",
  "summary": "one sentence description of the overall change",
  "findings": [
    {
      "severity": "LOW | MEDIUM | HIGH",
      "line_hint": "approximate file:line if identifiable",
      "description": "what the issue is",
      "recommendation": "how to fix it"
    }
  ]
}

If there are no issues, return severity LOW with an empty findings array."""

def main():
    diff_path = "/tmp/pr_diff.txt"

    try:
        with open(diff_path) as f:
            diff = f.read()
    except FileNotFoundError:
        print("No diff file found — skipping review.")
        sys.exit(0)

    if len(diff.strip()) == 0:
        print("Empty diff — no Python files changed.")
        sys.exit(0)

    # Truncate very large diffs to stay within context limits
    if len(diff) > 12000:
        diff = diff[:12000] + "\n\n[diff truncated for review]"

    client = anthropic.Anthropic()

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1000,
        system=SYSTEM_PROMPT,
        messages=[
            {
                "role": "user",
                "content": f"Review this diff:\n\n{diff}"
            }
        ]
    )

    raw = response.content[0].text.strip()

    try:
        result = json.loads(raw)
    except json.JSONDecodeError:
        print(f"Warning: LLM returned non-JSON response:\n{raw}")
        sys.exit(0)

    # Write PR comment body
    lines = [
        "## AI Code Review",
        f"**Overall severity:** `{result['severity']}`",
        f"**Summary:** {result['summary']}",
        "",
    ]

    if result.get("findings"):
        lines.append("### Findings\n")
        for i, finding in enumerate(result["findings"], 1):
            lines.append(f"**{i}. [{finding['severity']}]** {finding['description']}")
            if finding.get("line_hint"):
                lines.append(f"  - Location: `{finding['line_hint']}`")
            lines.append(f"  - Recommendation: {finding['recommendation']}")
            lines.append("")
    else:
        lines.append("No security issues found.")

    summary = "\n".join(lines)

    with open("/tmp/review_summary.txt", "w") as f:
        f.write(summary)

    print(summary)

    # Emit GitHub Actions annotations
    for finding in result.get("findings", []):
        level = "error" if finding["severity"] == "HIGH" else "warning"
        print(f"::{level}::{finding['description']}")

    # Block merge on HIGH severity
    if result["severity"] == "HIGH":
        print("\n::error::AI review found HIGH severity issues. Merge is blocked.")
        sys.exit(1)

    sys.exit(0)


if __name__ == "__main__":
    main()
```

### Step 2.3 — Test Against the Seeded Vulnerability

The sample repository includes a branch with a seeded SQL injection vulnerability. Open a PR from it to observe the gate in action.

```bash
# The fault branch already exists in the sample repo
git fetch origin fault/sql-injection
git checkout -b test-ai-gate origin/fault/sql-injection
git push origin test-ai-gate
```

Then open a PR from `test-ai-gate` → `main` in the GitHub UI.

**Expected result:** The `ai-review` job fails with a HIGH severity finding, the PR is blocked, and a comment is posted with the findings table.

### Step 2.4 — Extend the Gate (Optional Exercise)

Modify `llm_review.py` to:

- Post a `success` status when severity is LOW, a `warning` when MEDIUM, and `failure` when HIGH
- Add a check that counts the number of new lines added without a corresponding test — flag if the ratio exceeds 5:1

### Knowledge Check — Part 2

1. What is the difference between what a linter like `ruff` catches and what the LLM review gate catches?
2. Why is it important to truncate very large diffs before sending them to the API?
3. What risks exist if you set the blocking threshold to MEDIUM instead of HIGH? What risks exist if you raise it to CRITICAL?

---

## Part 3 — Test Selection & Flaky Test Fix

**Duration:** 20 minutes
**Goal:** Eliminate the flaky test and configure the pipeline to run only the tests impacted by a given change.

### Background

The flaky test issue is straightforward: `time.sleep(0.1)` is not a reliable synchronisation mechanism for async code. Under CI load the async callback takes longer than 100ms, the assertion runs before the result is available, and the test fails non-deterministically.

Test selection is more subtle. Running the full test suite on every commit gives maximum confidence but is expensive. Change-impact analysis examines which source files changed, traces through the import graph to find all test files that import those modules, and runs only that subset. For a large suite, this can reduce test time by 60–80% on small PRs while maintaining the same failure detection rate for changes that actually affect the code under test.

### Step 3.1 — Reproduce the Flaky Failure

Run the failing test 10 times in quick succession to observe the non-determinism:

```bash
for i in {1..10}; do
  pytest tests/test_webhooks.py::test_payment_webhook -x -q 2>&1 | tail -1
done
```

You should see a mix of `PASSED` and `FAILED` results.

### Step 3.2 — Fix the Flaky Test

Open `tests/test_webhooks.py` and replace the flaky test:

```python
# Before — flaky: uses real timer, non-deterministic under load
def test_payment_webhook():
    trigger_webhook()
    time.sleep(0.1)   # FAULT: race condition
    assert get_result() == "processed"


# After — deterministic: awaits the async result properly
import pytest
import pytest_asyncio

@pytest.mark.asyncio
async def test_payment_webhook(async_client):
    response = await async_client.post(
        "/webhooks/payment",
        json={"event": "payment.completed", "amount": 100}
    )
    assert response.status_code == 200
    assert response.json()["status"] == "processed"
```

Add `pytest-asyncio` to your test dependencies:

```toml
# pyproject.toml
[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "httpx>=0.27",        # required by AsyncClient in tests
    "anyio[trio]"
]
```

Add a `pytest.ini` or `pyproject.toml` section to set the asyncio mode:

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

Verify the fix is deterministic:

```bash
pip install -e ".[dev]"
for i in {1..10}; do
  pytest tests/test_webhooks.py::test_payment_webhook -x -q 2>&1 | tail -1
done
# Expected: 10x PASSED
```

### Step 3.3 — Configure Test Selection

#### Option A — With Launchable Token

If you have a Launchable token, add the following to your CI workflow:

```yaml
- name: Set up Launchable
  run: pip install launchable

- name: Record build
  env:
    LAUNCHABLE_TOKEN: ${{ secrets.LAUNCHABLE_TOKEN }}
  run: |
    launchable record build \
      --name "${{ github.run_id }}" \
      --source .

- name: Select impacted tests
  env:
    LAUNCHABLE_TOKEN: ${{ secrets.LAUNCHABLE_TOKEN }}
  run: |
    launchable subset \
      --build "${{ github.run_id }}" \
      --target 30% \
      pytest tests/ > /tmp/selected_tests.txt
    echo "Selected tests:"
    cat /tmp/selected_tests.txt

- name: Run selected tests
  run: pytest $(cat /tmp/selected_tests.txt) -v

- name: Record test results
  env:
    LAUNCHABLE_TOKEN: ${{ secrets.LAUNCHABLE_TOKEN }}
  if: always()
  run: |
    launchable record tests \
      --build "${{ github.run_id }}" \
      pytest /tmp/test_results.xml
```

#### Option B — Heuristic Fallback (No Token Required)

If you do not have a Launchable token, use this import-graph heuristic. It is less precise but demonstrates the concept:

```bash
#!/bin/bash
# .github/scripts/select_tests.sh
# Finds tests that import any of the changed source files

CHANGED_FILES=$(git diff --name-only origin/$BASE_REF...HEAD -- '*.py' \
  | grep -v '^tests/' \
  | sed 's|app/||; s|\.py$||; s|/|.|g')

if [ -z "$CHANGED_FILES" ]; then
  echo "No source files changed — running full suite"
  echo "tests/"
  exit 0
fi

echo "Changed modules: $CHANGED_FILES"
SELECTED=""

for module in $CHANGED_FILES; do
  # Find test files that import this module
  matches=$(grep -rl "from app.$module\|import $module" tests/ 2>/dev/null)
  SELECTED="$SELECTED $matches"
done

if [ -z "$SELECTED" ]; then
  echo "No impacted tests found — running full suite as fallback"
  echo "tests/"
else
  echo "Impacted tests: $SELECTED"
  echo $SELECTED
fi
```

Add to your CI workflow:

```yaml
- name: Select impacted tests
  env:
    BASE_REF: ${{ github.base_ref }}
  run: |
    chmod +x .github/scripts/select_tests.sh
    TESTS=$(.github/scripts/select_tests.sh)
    echo "SELECTED_TESTS=$TESTS" >> $GITHUB_ENV

- name: Run impacted tests
  run: pytest ${{ env.SELECTED_TESTS }} -v --tb=short
```

### Step 3.4 — Measure the Improvement

Record both run times. You will report these numbers during the debrief.

```bash
# Full suite baseline — record this time
time pytest tests/ -q

# Selective run on a branch that only touches app/routers/payments.py
git checkout -b test-selection-demo
# make a trivial change to app/routers/payments.py
time pytest $(bash .github/scripts/select_tests.sh) -q
```

**Target:** selective run should be at least 40% faster on a branch with localised changes.

### Knowledge Check — Part 3

1. Why is `time.sleep` fundamentally unreliable as a synchronisation mechanism in async tests?
2. What is the tradeoff between running 30% of tests (Launchable target) versus 100% on every PR?
3. Describe a scenario where the heuristic fallback script would fail to select the right tests.

---

## Part 4 — Automated Deployment Rollback

**Duration:** 20 minutes
**Goal:** Attach a Prometheus-backed `AnalysisTemplate` to the staging Argo Rollout so that a canary with an elevated error rate is automatically rolled back without human intervention.

### Background

The current Rollout has two problems. First, `maxUnavailable: 100%` means all pods are replaced simultaneously during a canary deployment — there is no safety net if the new version is broken. Second, without an `AnalysisTemplate`, Argo Rollouts has no way to evaluate whether the canary is healthy and will promote it to full traffic after the configured pause duration regardless of its error rate.

The fix introduces a gradual rollout (10% → 50% → 100%) with a Prometheus query measuring the 5xx error rate at each step. If the error rate exceeds 5% for two consecutive measurement intervals, the rollout is automatically aborted and the stable version is restored.

### Step 4.1 — Review the Broken Rollout

Inspect the current configuration:

```bash
kubectl get rollout payments-api -n staging -o yaml
```

Or review `k8s/rollout.yaml` directly:

```yaml
# Current — dangerous configuration
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payments-api
  namespace: staging
spec:
  replicas: 4
  strategy:
    canary:
      maxUnavailable: 100%     # FAULT: all pods replaced at once
      steps:
        - setWeight: 20
        - pause:
            duration: 2m
        - setWeight: 100
      # FAULT: no analysis template — bad canaries are never detected
  selector:
    matchLabels:
      app: payments-api
  template:
    spec:
      containers:
        - name: payments-api
          image: payments-api:latest
```

### Step 4.2 — Create the Analysis Template

Create `k8s/analysis-template.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
  namespace: staging
spec:
  metrics:
    - name: http-error-rate
      interval: 30s
      failureLimit: 2
      successCondition: "result[0] < 0.05"
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc.cluster.local:9090
          query: |
            sum(
              rate(
                http_requests_total{
                  job="payments-api",
                  namespace="staging",
                  status=~"5.."
                }[1m]
              )
            )
            /
            sum(
              rate(
                http_requests_total{
                  job="payments-api",
                  namespace="staging"
                }[1m]
              )
            )
```

The query computes the ratio of 5xx responses to total requests over a 1-minute window. If this ratio exceeds 0.05 (5%) for two consecutive 30-second intervals, the analysis is marked as `Failed` and the rollout is aborted.

Apply it:

```bash
kubectl apply -f k8s/analysis-template.yaml
```

### Step 4.3 — Fix the Rollout Manifest

Replace `k8s/rollout.yaml` with the corrected version:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payments-api
  namespace: staging
spec:
  replicas: 4
  strategy:
    canary:
      maxUnavailable: 1              # safe: replace one pod at a time
      canaryService: payments-api-canary
      stableService: payments-api-stable
      analysis:
        templates:
          - templateName: error-rate-check
        startingStep: 1              # begin analysis from step 1
      steps:
        - setWeight: 10              # 10% of traffic to canary
        - pause:
            duration: 2m
        - setWeight: 50
        - pause:
            duration: 2m
        - setWeight: 100
  selector:
    matchLabels:
      app: payments-api
  template:
    metadata:
      labels:
        app: payments-api
    spec:
      containers:
        - name: payments-api
          image: payments-api:latest
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
```

Apply the updated rollout:

```bash
kubectl apply -f k8s/rollout.yaml
```

### Step 4.4 — Simulate a Bad Deployment

The sample repository includes a tagged image that returns 500 errors on 60% of requests. Deploy it as a canary to verify that the analysis template triggers an automatic rollback.

```bash
# Deploy the bad image
kubectl argo rollouts set image payments-api \
  payments-api=payments-api:fault-high-error-rate \
  -n staging

# Watch the rollout in real time
kubectl argo rollouts get rollout payments-api -n staging --watch
```

You should see the status progress through:

```
Status:        Progressing
Canary weight: 10
Analysis run:  error-rate-check.1  Running

# After ~60 seconds, once the error rate threshold is breached twice:

Status:        Degraded
Message:       RolloutAborted: AnalysisRun "error-rate-check.1" failed
Canary weight: 0
```

The rollout automatically reverts to the previous stable image.

### Step 4.5 — Verify Recovery

```bash
# Confirm stable version is serving all traffic
kubectl argo rollouts get rollout payments-api -n staging

# Check current image
kubectl get deployment payments-api -n staging \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# Expected: payments-api:latest (the last known good image)
```

### Knowledge Check — Part 4

1. Why is `maxUnavailable: 100%` dangerous in a canary deployment?
2. What does `failureLimit: 2` mean in the `AnalysisTemplate`? What is the tradeoff of setting it to 1?
3. How would you modify the `AnalysisTemplate` to also check p99 latency in addition to error rate?

---

## 10. Debrief & Discussion

Reserve the final 10 minutes for group discussion. Each participant should be prepared to answer:

**Fault 1 — Dependency**
- What was the root cause of the conflict and why did `pip install` not catch it?
- How does the Renovate `groupName` rule prevent this class of failure?

**Fault 2 — Flaky test**
- What made `test_payment_webhook` flaky and how did the fix eliminate the non-determinism?
- What was the difference in test run time between the full suite and the selected subset?

**Fault 3 — Deploy**
- What would have happened to production traffic if this misconfigured rollout had been promoted?
- How long did it take for the analysis template to detect the bad canary and trigger rollback?

**Broader discussion**
- Where in this lab did the AI intervention happen *before* the pipeline ran (shift-left) and where did it happen *during* a running deployment?
- What are the risks of a fully autonomous pipeline that rolls back and re-deploys without human approval?

---

## 11. Rubric

| Criterion | Points |
|---|---|
| `httpx` constraint is fixed correctly and local import passes | 10 |
| `renovate.json` groups `fastapi`, `httpx`, `starlette` with `automerge: false` | 10 |
| AI review workflow runs on PRs and exits 1 on HIGH severity | 20 |
| Review script posts a structured findings comment to the PR | 10 |
| Flaky test is rewritten with `pytest-asyncio` and passes 10/10 times | 15 |
| Test selection reduces run time by ≥ 40% on a localised branch | 10 |
| `AnalysisTemplate` queries error rate and has `failureLimit: 2` | 15 |
| Automatic rollback fires within one analysis window on the bad image | 10 |
| **Total** | **100** |

---

## 12. Reference

### Files Modified in This Lab

| File | Part | Change |
|---|---|---|
| `pyproject.toml` | 1 | Relax `httpx` version pin |
| `renovate.json` | 1 | Add FastAPI ecosystem group rule |
| `.github/workflows/ai-review.yml` | 2 | New AI review gate workflow |
| `.github/scripts/llm_review.py` | 2 | LLM review script |
| `tests/test_webhooks.py` | 3 | Rewrite flaky test with asyncio |
| `.github/workflows/ci.yml` | 3 | Add test selection step |
| `.github/scripts/select_tests.sh` | 3 | Heuristic test selector |
| `k8s/analysis-template.yaml` | 4 | New Prometheus analysis template |
| `k8s/rollout.yaml` | 4 | Fix rollout strategy and attach analysis |

### Tool Documentation

| Tool | Docs | Key Concept Used |
|---|---|---|
| Renovate | https://docs.renovatebot.com | `packageRules`, `groupName`, `stabilityDays` |
| Anthropic API | https://docs.anthropic.com | `messages.create`, system prompts |
| pytest-asyncio | https://pytest-asyncio.readthedocs.io | `asyncio_mode = auto` |
| Launchable | https://docs.launchableinc.com | `subset`, `record build`, `record tests` |
| Argo Rollouts | https://argo-rollouts.readthedocs.io | `AnalysisTemplate`, `canary.analysis` |

### Key Terminology

**Transitive dependency conflict** — An incompatibility between two packages that is not declared in their manifests but manifests at runtime because one package relies on an internal API of the other that has been removed in the version resolved by the package manager.

**Flaky test** — A test that produces inconsistent results (pass or fail) across runs without any change to the code under test, typically due to timing dependencies, shared mutable state, or undetermined ordering of asynchronous operations.

**Change-impact analysis** — The process of determining which tests are exercised by a given set of code changes by tracing the import and call graph from the changed files to the test files that depend on them.

**Analysis template** — An Argo Rollouts resource that defines one or more metric queries evaluated at each step of a progressive delivery rollout. If the metric fails its success condition a configured number of times, the rollout is aborted and the stable version is restored.

**Shift left** — The practice of moving quality, security, and reliability checks earlier in the software delivery lifecycle — ideally before code is merged — to reduce the cost and impact of failures discovered late.

---

*Lab designed for Module 1 of the AI in CI/CD Pipelines workshop series. Review after each tool release cycle to ensure API and configuration compatibility.*
