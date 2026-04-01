# Workshop: AI-Powered CI/CD — Smarter Pipelines, Fewer Failures

> **Duration:** 4 hours (half-day) | **Level:** Intermediate–Advanced  
> **Audience:** DevOps Engineers, Platform Engineers, Senior Developers  
> **Prerequisites:** Basic Git, familiarity with a CI system (GitHub Actions, Jenkins, or similar)

---

## Table of Contents

1. [Workshop Overview](#workshop-overview)
2. [Module 1 — Intelligent Dependency Resolution & Conflict Detection](#module-1)
3. [Module 2 — AI-Assisted Code Review Gates](#module-2)
4. [Module 3 — Predictive Build Failure Detection](#module-3)
5. [Module 4 — Tool Deep-Dives](#module-4)
6. [Module 5 — Capstone: Build Your Own AI Pipeline](#module-5)
7. [Reference & Resources](#reference)

---

## Workshop Overview

Traditional CI/CD pipelines react to problems — they fail, report, and wait for humans. This workshop introduces a new paradigm: **pipelines that anticipate, reason, and act**.

By the end of this workshop you will be able to:

- Configure intelligent dependency scanning that flags conflicts before merge
- Embed AI code review gates that evaluate semantics, security intent, and architectural fit
- Deploy predictive models that estimate build failure probability mid-run
- Evaluate and integrate GitHub Copilot, Qodana, SonarQube AI, and Amazon CodeGuru into real pipelines

---

## Module 1 — Intelligent Dependency Resolution & Conflict Detection {#module-1}

**Duration:** 45 minutes

### 1.1 The Problem With Naive Dependency Management

Traditional lock files (`package-lock.json`, `poetry.lock`, `go.sum`) guarantee reproducibility but do not reason about compatibility. Common failure modes include:

- Transitive dependency version collisions that pass `npm install` but break at runtime
- Security patches that satisfy a version range but introduce breaking API changes
- Peer dependency drift between major library upgrades

### 1.2 How AI-Assisted Resolution Works

Modern AI dependency tools operate in two phases:

**Phase 1 — Graph Analysis**  
The tool builds a full dependency graph, identifies all resolution paths, and scores each path using a model trained on historical breakage data from public package registries.

**Phase 2 — Conflict Prediction**  
Using embeddings of changelogs, release notes, and issue histories, the model predicts whether two packages at their resolved versions have known incompatibilities — even if none is declared in the manifest.

### 1.3 Lab: Configuring Renovate Bot with AI Grouping

Renovate Bot is an open-source dependency update tool that supports AI-assisted grouping of related upgrades.

**Step 1: Install Renovate**

```bash
npm install -g renovate
```

**Step 2: Add `renovate.json` to your repository root**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "ai": {
    "enabled": true,
    "groupRelatedUpdates": true,
    "conflictDetection": "semantic"
  },
  "packageRules": [
    {
      "matchUpdateTypes": ["major"],
      "addLabels": ["ai-reviewed", "needs-human-approval"],
      "automerge": false
    },
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true,
      "automergeType": "pr",
      "platformAutomerge": true
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  }
}
```

**Step 3: Trigger a dry-run**

```bash
LOG_LEVEL=debug renovate --dry-run=full --platform=github \
  --token=$GITHUB_TOKEN \
  your-org/your-repo
```

**Step 4: Inspect the conflict report**

Look for the `conflictMatrix` output in the log. It lists pairs of packages where the AI model has detected a higher-than-threshold incompatibility score:

```
[INFO] Conflict Matrix (threshold: 0.72)
  react@19.0.0 <> react-dom@18.3.1     score: 0.91  BLOCK
  eslint@9.0.0 <> eslint-plugin-react@7  score: 0.78  WARN
```

### 1.4 Discussion: Semantic vs. Syntactic Conflict Detection

| Approach | What It Catches | What It Misses |
|---|---|---|
| Version range parsing | Range incompatibilities | Runtime API breaks |
| Changelog NLP | Announced breaking changes | Undocumented behavior changes |
| Historical crash correlation | Empirically observed pairs | New package combinations |
| AI embedding similarity | Conceptual API drift | Intentional redesigns |

> **Key Insight:** No single method is sufficient. Best practice is layered detection — syntactic gates first (fast), then semantic/AI gates (slower, run async).

### 1.5 Knowledge Check

1. What is the difference between a version conflict and a semantic conflict?
2. Why should major dependency updates always require human approval even when AI confidence is high?
3. Name two data sources an AI model might use to predict package incompatibility.

---

## Module 2 — AI-Assisted Code Review Gates {#module-2}

**Duration:** 60 minutes

### 2.1 Beyond Linting: What Semantic Review Means

Syntactic analysis answers: *"Does this code compile and conform to style rules?"*

Semantic analysis answers: *"Does this code do what the author intended, safely and correctly?"*

A semantic code review gate evaluates:

- **Intent alignment** — Does the implementation match the PR description?
- **Security posture** — Does the change inadvertently widen an attack surface?
- **Architectural conformance** — Does the change respect established patterns (e.g., no direct DB calls from presentation layer)?
- **Test adequacy** — Are the tests actually exercising the changed logic?

### 2.2 Tool: GitHub Copilot Code Review

GitHub Copilot's review feature can be triggered automatically on pull requests via GitHub Actions.

**Step 1: Add the workflow**

Create `.github/workflows/copilot-review.yml`:

```yaml
name: Copilot AI Code Review

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

      - name: Run Copilot Review
        uses: github/copilot-code-review-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          review-level: detailed          # minimal | standard | detailed
          block-on-severity: high         # none | medium | high | critical
          custom-instructions: |
            - Flag any database queries not using parameterized statements
            - Warn if error handling swallows exceptions silently
            - Ensure all public API endpoints have authentication checks
            - Check that new environment variables are documented in .env.example
```

**Step 2: Configure blocking thresholds**

In your repository settings under **Copilot > Code Review**:

```
Minimum review score to merge:  70 / 100
Auto-dismiss stale reviews:     true
Require re-review on new push:  true
```

**Step 3: Interpret the review output**

Copilot posts structured comments tagged by category:

```
[SECURITY HIGH] Line 47: User input passed directly to exec() without sanitization.
  Suggestion: Use subprocess.run() with a list of arguments instead of a shell string.

[INTENT MISMATCH MEDIUM] PR description says "read-only operation" but line 112 
  calls user.save(). Verify this is intentional.

[TEST COVERAGE LOW] The branch on line 89 (error path) has no corresponding test case.
```

### 2.3 Tool: SonarQube AI Code Review

SonarQube's AI-powered analysis goes beyond its traditional rule set by applying a large language model to evaluate code quality holistically.

**Step 1: Add SonarQube to your pipeline**

```yaml
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
  with:
    args: >
      -Dsonar.projectKey=my-project
      -Dsonar.ai.codeReview.enabled=true
      -Dsonar.ai.codeReview.model=gpt-4o
      -Dsonar.qualitygate.wait=true
```

**Step 2: Define a Quality Gate with AI metrics**

In the SonarQube UI (**Quality Gates > Create**):

| Metric | Operator | Threshold |
|---|---|---|
| AI Security Severity | is worse than | HIGH |
| AI Cognitive Complexity Delta | is greater than | 15 |
| New Code Coverage | is less than | 80% |
| Duplicated Lines on New Code | is greater than | 3% |
| AI-flagged Intent Mismatches | is greater than | 0 |

**Step 3: Enforce the gate in CI**

```yaml
- name: Check Quality Gate
  run: |
    STATUS=$(curl -s -u $SONAR_TOKEN: \
      "$SONAR_HOST_URL/api/qualitygates/project_status?projectKey=my-project" \
      | jq -r '.projectStatus.status')
    echo "Quality Gate status: $STATUS"
    if [ "$STATUS" != "OK" ]; then
      echo "::error::Quality Gate failed. Review SonarQube findings before merging."
      exit 1
    fi
```

### 2.4 Lab: Writing Effective AI Review Instructions

The quality of AI code review depends heavily on the context you provide. Practice writing custom instructions using this template:

```
CONTEXT:
  This is a [language] [application type] using [framework].
  The codebase follows [architectural pattern].

ALWAYS FLAG:
  - [specific anti-pattern for your domain]
  - [security concern relevant to your stack]

ALWAYS VERIFY:
  - [invariant that must hold, e.g., "all DB writes are wrapped in transactions"]

NEVER FLAG AS ISSUES:
  - [intentional deviation from standard patterns with known reason]
```

**Exercise:** Write custom instructions for a Python FastAPI microservice that handles payment processing.

### 2.5 Gate Design Patterns

```
PR Opened
    │
    ▼
┌─────────────────────┐
│  Fast Gates (< 2m)  │  ← Linting, type checking, unit tests
│  Block on failure   │
└────────┬────────────┘
         │ Pass
         ▼
┌─────────────────────┐
│  AI Gates (2–8m)    │  ← Copilot, SonarQube AI
│  Block on HIGH+     │
└────────┬────────────┘
         │ Pass
         ▼
┌─────────────────────┐
│  Human Review       │  ← Required for MEDIUM findings
│  AI summary shown   │
└────────┬────────────┘
         │ Approved
         ▼
       Merge
```

### 2.6 Knowledge Check

1. What is the difference between a syntactic code quality issue and a semantic one? Give an example of each.
2. Why should AI review gates run *after* fast syntactic gates rather than in parallel?
3. What are the risks of setting the AI gate threshold too low? Too high?

---

## Module 3 — Predictive Build Failure Detection {#module-3}

**Duration:** 45 minutes

### 3.1 The Cost of Late Failure

Build pipelines fail. The question is *when* in the pipeline the failure is detected:

| Failure detected at | Avg. cost (engineering time) |
|---|---|
| Pre-commit hook | ~2 minutes |
| PR CI gate | ~10 minutes |
| Integration test stage | ~25 minutes |
| Post-merge main branch | ~45 minutes |
| Production deployment | ~2–4 hours |

Predictive failure detection shifts detection left — ideally before the expensive stages even run.

### 3.2 How Prediction Works

A predictive failure model is trained on historical pipeline run data. Features typically include:

**Static features (available before build starts):**
- Number of files changed
- Which directories were touched
- Author's historical failure rate on those paths
- Time since last successful build of affected components
- Dependency update included in changeset (yes/no)

**Dynamic features (available mid-build):**
- CPU/memory usage trajectory vs. baseline
- Test execution time deviation from expected
- Log anomaly score (embedding distance from normal logs)
- Flaky test history of currently-running tests

The model outputs a failure probability score (0–1). Above a configurable threshold, it can post a warning, cancel the build early, or reallocate resources.

### 3.3 Lab: Integrating Amazon CodeGuru into Your Pipeline

Amazon CodeGuru Profiler and Reviewer provide build-time and runtime intelligence.

**Step 1: Set up CodeGuru Reviewer**

```yaml
- name: Amazon CodeGuru Reviewer
  uses: aws-actions/codeguru-reviewer@v1.1
  with:
    s3_bucket: my-codeguru-artifacts-bucket
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_DEFAULT_REGION: us-east-1
```

**Step 2: Enable build profiling**

Add the CodeGuru Profiler agent to your build:

```bash
# Install the profiler CLI
pip install amazon-codeguru-profiler-python-agent

# Wrap your build command
codeguru-profiler --profiling-group-name my-build-group \
  --region us-east-1 \
  -- ./gradlew build
```

**Step 3: Query failure predictions mid-build**

```python
import boto3
import json

codeguru = boto3.client('codeguru-reviewer', region_name='us-east-1')

def get_failure_risk(association_arn: str, commit_sha: str) -> dict:
    """
    Queries CodeGuru for a risk assessment of the current build.
    Returns a dict with 'riskLevel' and 'recommendations'.
    """
    response = codeguru.list_recommendations(
        CodeReviewArn=association_arn
    )
    
    high_severity = [
        r for r in response['RecommendationSummaries']
        if r['Severity'] in ('High', 'Critical')
    ]
    
    risk_level = 'HIGH' if len(high_severity) > 0 else \
                 'MEDIUM' if len(response['RecommendationSummaries']) > 5 else \
                 'LOW'
    
    return {
        'riskLevel': risk_level,
        'totalFindings': len(response['RecommendationSummaries']),
        'highSeverityCount': len(high_severity),
        'recommendations': high_severity[:3]  # Top 3 for display
    }

risk = get_failure_risk(
    association_arn=os.environ['CODEGURU_ASSOCIATION_ARN'],
    commit_sha=os.environ['GITHUB_SHA']
)

if risk['riskLevel'] == 'HIGH':
    print(f"::warning::High failure risk detected. {risk['highSeverityCount']} critical findings.")
    # Optionally: exit(1) to block the build
```

**Step 4: Visualize build health trends**

```bash
# Pull the last 30 days of build data from your CI system
aws codeguru-reviewer list-code-reviews \
  --type PullRequest \
  --provider-types GitHub \
  --states Completed \
  --max-results 100 \
  --query 'CodeReviewSummaries[*].{Name:Name,Date:CreatedTimeStamp,Findings:NumberOfRecommendations}' \
  --output table
```

### 3.4 Early Cancellation Strategy

```yaml
# .github/workflows/predictive-ci.yml

jobs:
  risk-assessment:
    runs-on: ubuntu-latest
    outputs:
      risk_level: ${{ steps.assess.outputs.risk_level }}
    steps:
      - name: Assess Build Risk
        id: assess
        run: |
          RISK=$(python scripts/assess_risk.py --sha ${{ github.sha }})
          echo "risk_level=$RISK" >> $GITHUB_OUTPUT

  build:
    needs: risk-assessment
    runs-on: ubuntu-latest
    steps:
      - name: Notify on High Risk
        if: needs.risk-assessment.outputs.risk_level == 'HIGH'
        run: |
          echo "::warning::High failure risk. Consider reviewing findings before proceeding."
          # Post to Slack, create GitHub issue, etc.

      - name: Run Full Build
        run: ./scripts/build.sh

      - name: Run Expensive Integration Tests
        # Skip expensive tests if risk is low (fast path)
        if: needs.risk-assessment.outputs.risk_level != 'LOW'
        run: ./scripts/integration-tests.sh
```

### 3.5 Knowledge Check

1. What is the tradeoff between a high and low failure probability threshold for early cancellation?
2. Name three dynamic features that a predictive model could use mid-build to update its failure estimate.
3. Why does failure detection cost increase with pipeline stage? What makes it more expensive at each stage?

---

## Module 4 — Tool Deep-Dives {#module-4}

**Duration:** 30 minutes

### 4.1 GitHub Copilot in the Pipeline

| Capability | Description | Integration Point |
|---|---|---|
| PR Code Review | Semantic review of changed code with inline suggestions | GitHub Actions, PR webhook |
| Autofix | Automatically patches low-risk issues (typos, formatting, simple bugs) | Commit-level, optional |
| Workspace Context | Understands your full codebase when reviewing | Configured via `.github/copilot-instructions.md` |
| Custom Instructions | Repository-level guidance for review focus | `.github/copilot-instructions.md` |

**Configuring `.github/copilot-instructions.md`:**

```markdown
# Copilot Review Instructions

## Stack
- Backend: Python 3.12, FastAPI, SQLAlchemy
- Frontend: TypeScript, React 19, Tanstack Query
- Infrastructure: Terraform, AWS

## Review Priorities (in order)
1. Security vulnerabilities (OWASP Top 10)
2. Data validation on all API boundaries
3. Proper error handling and logging
4. Performance: N+1 query patterns, unbounded loops

## Known Intentional Patterns
- Direct SQL in `/scripts/migrations/` — do not flag as unsafe
- `# noqa` comments on specific lines are intentional suppressions

## Do Not Flag
- Test files using `assert` without pytest.raises (our convention)
```

### 4.2 Qodana

JetBrains Qodana is a code quality platform that runs IDE-quality inspections in CI, with AI-enhanced analysis.

**Key differentiator:** Qodana applies the same deep static analysis that powers IntelliJ IDEA — understanding data flow, null safety, and complex type inference — in a headless CI context.

```yaml
- name: Qodana Scan
  uses: JetBrains/qodana-action@v2024.3
  env:
    QODANA_TOKEN: ${{ secrets.QODANA_TOKEN }}
  with:
    pr-mode: true               # Only analyze changed files
    post-pr-comment: true       # Post findings as PR comments
    upload-result: true         # Push to Qodana Cloud dashboard
    args: >
      --baseline qodana.sarif.json
      --fail-threshold 0
```

**`qodana.yaml` configuration:**

```yaml
version: "1.0"
linter: jetbrains/qodana-jvm:latest

profile:
  name: qodana.recommended

exclude:
  - name: All
    paths:
      - build/
      - .gradle/

failureConditions:
  severityThresholds:
    any: 0        # Fail on any critical issue
    high: 5       # Allow up to 5 high issues
    medium: 20

ai:
  assistedInspections: true    # AI-enhanced analysis
  contextualSuggestions: true  # Contextual fix recommendations
```

### 4.3 SonarQube AI

SonarQube's AI features (as of SonarQube 10.x with SonarQube AI) include:

- **AI Code Fix** — suggests single-click fixes for detected issues
- **AI-generated issue explanations** — plain-English explanations of why something is flagged
- **Smart exclusions** — AI learns which exclusion rules are consistently overridden by your team and adapts

**Key configuration in `sonar-project.properties`:**

```properties
sonar.projectKey=my-project
sonar.projectName=My Project
sonar.sources=src
sonar.tests=tests
sonar.python.coverage.reportPaths=coverage.xml

# AI features
sonar.ai.codeReview.enabled=true
sonar.ai.generateFixes=true
sonar.ai.contextualExplanations=true

# Quality gate
sonar.qualitygate.wait=true
sonar.qualitygate.timeout=300
```

### 4.4 Amazon CodeGuru

| Feature | Use Case | CI Stage |
|---|---|---|
| Reviewer | PR-level code analysis | Pull Request |
| Profiler | Build & runtime performance regression | Build + Production |
| Security Detector | Security policy violations | Pull Request + Build |
| Guru Findings | Aggregated recommendations with fix suggestions | PR Comments |

**CodeGuru Security inline in CI:**

```yaml
- name: CodeGuru Security Scan
  run: |
    aws codeguru-security create-scan \
      --resource-id resourceId=$(git remote get-url origin) \
      --scan-name "build-${{ github.run_id }}" \
      --scan-type Standard

    # Wait for scan completion
    aws codeguru-security wait scan-complete \
      --scan-name "build-${{ github.run_id }}"

    # Fail if critical findings exist
    CRITICAL=$(aws codeguru-security get-findings \
      --scan-name "build-${{ github.run_id }}" \
      --status Open \
      --severity CRITICAL \
      --query 'length(findings)' --output text)

    if [ "$CRITICAL" -gt "0" ]; then
      echo "::error::$CRITICAL critical security findings. Build blocked."
      exit 1
    fi
```

---

## Module 5 — Capstone: Build Your Own AI Pipeline {#module-5}

**Duration:** 60 minutes

### 5.1 Objective

Design and partially implement an AI-powered CI/CD pipeline for a provided sample repository. Your pipeline must include all three capabilities covered in this workshop.

### 5.2 Sample Repository

Clone the workshop sample:

```bash
git clone https://github.com/workshop-ai-cicd/sample-app
cd sample-app
```

The repository is a Python FastAPI application with intentional issues seeded at known locations.

### 5.3 Deliverables

**Deliverable 1: Dependency Configuration**  
Create `renovate.json` with AI grouping, conflict detection, and appropriate auto-merge policies for this application type.

**Deliverable 2: AI Review Gate**  
Write a GitHub Actions workflow that:
- Runs Copilot review on every PR
- Runs SonarQube AI analysis
- Blocks merge if any HIGH or above finding exists
- Posts a summary comment to the PR with finding counts by severity

**Deliverable 3: Predictive Failure Integration**  
Extend the workflow to:
- Query CodeGuru after each push
- Post a risk score badge to the PR
- Skip integration tests when risk score is LOW (optional fast path)

**Deliverable 4: Custom Review Instructions**  
Write a `.github/copilot-instructions.md` tailored for the sample app, referencing its specific stack, known patterns, and security requirements.

### 5.4 Architecture Diagram (Target State)

```
Developer Push
      │
      ▼
┌──────────────────────────────────────────────────┐
│              PR Trigger (GitHub Actions)          │
└─────────────────────┬────────────────────────────┘
                      │
          ┌───────────┼────────────┐
          ▼           ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────────┐
    │ Renovate │ │  Lint +  │ │  CodeGuru    │
    │ Conflict │ │  Type    │ │  Risk Score  │
    │ Check    │ │  Check   │ │  Assessment  │
    └────┬─────┘ └────┬─────┘ └──────┬───────┘
         │            │              │
         └────────────┼──────────────┘
                      │ All pass
                      ▼
          ┌───────────────────────┐
          │   AI Review Gates     │
          │  ┌─────────────────┐  │
          │  │ Copilot Review  │  │
          │  │ SonarQube AI    │  │
          │  │ Qodana Scan     │  │
          │  └────────┬────────┘  │
          └───────────┼───────────┘
                      │ No HIGH+ findings
                      ▼
          ┌───────────────────────┐
          │   Conditional Tests   │
          │  Risk LOW → fast path │
          │  Risk MED+ → full run │
          └───────────┬───────────┘
                      │ Pass
                      ▼
              Human Approval
                      │
                      ▼
                    Merge
```

### 5.5 Review Rubric

| Criterion | Points |
|---|---|
| Renovate config is syntactically valid with AI conflict detection enabled | 10 |
| Gate workflow correctly blocks on HIGH severity | 20 |
| PR comment is posted with finding summary | 15 |
| Risk score integration is functional | 20 |
| Fast-path conditional test skipping is implemented | 15 |
| Custom Copilot instructions are specific and actionable | 10 |
| Documentation explains design decisions | 10 |
| **Total** | **100** |

---

## Reference & Resources {#reference}

### Tool Documentation

| Tool | Documentation | Key Concepts |
|---|---|---|
| GitHub Copilot | https://docs.github.com/en/copilot | Code Review, Workspace Context |
| Qodana | https://www.jetbrains.com/help/qodana | Baselines, Profiles, Cloud |
| SonarQube | https://docs.sonarsource.com/sonarqube | Quality Gates, Security Hotspots |
| Amazon CodeGuru | https://docs.aws.amazon.com/codeguru | Reviewer, Profiler, Security |
| Renovate | https://docs.renovatebot.com | Dependency Dashboard, AI grouping |

### Recommended Reading

- *Accelerate* — Forsgren, Humble, Kim (DORA metrics & deployment frequency)  
- *Continuous Delivery* — Jez Humble & David Farley (pipeline design principles)  
- OWASP CI/CD Security Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/CI_CD_Security_Cheat_Sheet.html

### Key Terminology

**Quality Gate** — A set of conditions that code must satisfy before it can progress to the next pipeline stage. Can be syntactic (linting), metric-based (coverage), or AI-evaluated (semantic review score).

**Semantic Analysis** — Analysis that considers the *meaning* and *intent* of code, not just its syntax. Requires understanding context, data flow, and relationships between components.

**Shift Left** — Moving quality checks earlier in the development process to reduce the cost of finding and fixing issues.

**Transitive Dependency** — A dependency of a dependency. Package A depends on Package B which depends on Package C — C is a transitive dependency of A.

**Predictive Cancellation** — Terminating a build early when a model predicts it will fail, to save compute resources and provide faster feedback.

---

## Facilitator Notes

### Timing Guide

| Module | Ideal Duration | Can Cut To |
|---|---|---|
| Module 1 — Dependencies | 45 min | 30 min (skip Lab 1.3) |
| Module 2 — Code Review | 60 min | 45 min (skip SonarQube lab) |
| Module 3 — Prediction | 45 min | 30 min (skip hands-on query) |
| Module 4 — Tool Deep Dives | 30 min | 20 min (reference only) |
| Module 5 — Capstone | 60 min | 45 min (partial deliverables) |

### Common Issues & Resolutions

**Issue:** Copilot review action not triggering  
**Resolution:** Check that the `pull-requests: write` permission is present in the workflow YAML.

**Issue:** SonarQube quality gate times out  
**Resolution:** Increase `sonar.qualitygate.timeout` or use a self-hosted SonarQube instance.

**Issue:** CodeGuru scan returns no findings  
**Resolution:** Ensure the S3 bucket policy allows CodeGuru to write to it and that the IAM role has `codeguru-reviewer:*` permissions.

**Issue:** Renovate runs but doesn't apply AI grouping  
**Resolution:** Confirm your Renovate version is ≥ 37.0 and that the Renovate App has been authorized for the repository.

---

*Workshop authored for internal developer platform enablement. Review annually against tool version updates.*
