Here is the same workshop rewritten in a simple, human-created style without icons or flashy formatting.

---

# Workshop: AI Co-Pilot in CI/CD Pipeline

## Objective

Demonstrate how AI assists developers in CI/CD pipelines without replacing them. Focus areas:

* Fixing build failures
* Improving test coverage
* Debugging pipeline issues

---

## Tools Used

* GitHub repository
* GitHub Actions
* GitHub Copilot
* Optional: Jenkins

---

## Lab Architecture

Developer pushes code → CI pipeline runs → Build/Test fails → AI suggests fix → Developer reviews and applies → Pipeline passes

---

# Lab 1: AI Helping in Build Failure

## Step 1: Create sample application

```python
# app.py
def add(a, b):
    return a + b

print(add(2, "3"))  # intentional error
```

---

## Step 2: Create CI pipeline

```yaml
name: CI Pipeline

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run App
        run: python app.py
```

Pipeline will fail due to a type error.

---

## Step 3: Use AI Co-Pilot

Ask GitHub Copilot:
"Fix this TypeError in Python code"

Suggested fix:

```python
print(add(2, int("3")))
```

---

## Step 4: Commit fix

Push changes and pipeline should pass.

Learning: AI helps identify and fix issues quickly, but the developer still decides.

---

# Lab 2: AI for Test Generation

## Step 1: Add new function

```python
def multiply(a, b):
    return a * b
```

---

## Step 2: Ask AI

"Generate unit tests for this function using pytest"

Suggested test:

```python
def test_multiply():
    assert multiply(2, 3) == 6
```

---

## Step 3: Add test stage in pipeline

```yaml
- name: Install pytest
  run: pip install pytest

- name: Run Tests
  run: pytest
```

Learning: AI reduces effort in writing tests and improves coverage.

---

# Lab 3: AI Detecting Flaky Tests

## Step 1: Add flaky test

```python
import random

def test_random():
    assert random.randint(1, 10) > 2
```

This test may randomly fail.

---

## Step 2: Ask AI

"Why is this test flaky?"

AI explanation:

* Test uses random values
* Results are not consistent

---

## Step 3: Fix test

```python
def test_random_fixed():
    assert 5 > 2
```

Learning: AI helps identify unstable test patterns.

---

# Lab 4: AI Log Analysis

## Step 1: Introduce dependency error

```bash
pip install nonexist-package
```

---

## Step 2: Pipeline error

Error shows package not found.

---

## Step 3: Ask AI

"Explain this CI error and suggest a fix"

AI identifies:

* Incorrect package name
* Suggests correct dependency or removal

Learning: AI simplifies troubleshooting.

---

# Lab 5: Smart Test Selection (Concept)

Ask AI:
"Which tests should run if only app.py changed?"

AI suggests:

* Run related unit tests only
* Skip unrelated tests

Learning: AI can optimize pipeline execution time.

---

# Key Takeaways

* AI speeds up debugging and troubleshooting
* AI helps generate and improve tests
* AI explains failures clearly
* Final decisions remain with the developer

---

# Optional Extensions

* Use AWS CodeGuru Reviewer for pull request reviews
* Integrate AI with Jenkins pipelines
* Add security scanning with AI suggestions



---

If you want, I can also make a version of this workshop for Jenkins + Kubernetes pipelines or convert it into a classroom slide deck.
