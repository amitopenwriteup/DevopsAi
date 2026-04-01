Here is a **repo-specific assignment** based on your Dockerfile, focused on using AI as a co-pilot in a practical way.

---

# Assignment: AI Co-Pilot for Dockerized Java App (Tomcat)

## Context

You are given a Dockerfile that deploys a WAR file on Tomcat:

```dockerfile
FROM tomcat:latest

LABEL maintainer="Nidhi Gupta"

ADD ./target/LoginWebApp-1.war /usr/local/tomcat/webapps/

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

---

# Objective

Use AI tools like GitHub Copilot to:

* Improve Dockerfile quality
* Fix issues
* Optimize build and deployment
* Add CI/CD intelligence

---

# Task 1: Dockerfile Review using AI

## Step

Ask AI:

* “Review this Dockerfile and suggest improvements”

## Expected AI Observations

* `tomcat:latest` is not version-pinned
* `ADD` should be replaced with `COPY`
* No health check
* WAR name is hardcoded
* No cleanup or optimization

## Deliverable

* List of issues identified
* AI suggestions
* Your final improved Dockerfile

---

# Task 2: Fix and Optimize Dockerfile

## Step

Ask AI:

* “Rewrite this Dockerfile with best practices”

## Example Improved Output (reference)

```dockerfile
FROM tomcat:9.0-jdk17

LABEL maintainer="Nidhi Gupta"

COPY ./target/LoginWebApp-1.war /usr/local/tomcat/webapps/

EXPOSE 8080

HEALTHCHECK CMD curl --fail http://localhost:8080 || exit 1

CMD ["catalina.sh", "run"]
```

## Deliverable

* Final optimized Dockerfile
* Explanation of each change

---

# Task 3: AI for Build Failure Debugging

## Step

Break the build intentionally:

* Rename WAR file incorrectly
* Remove `target/` folder

Run:

```bash
docker build -t login-app .
```

## Then Ask AI:

* “Why is Docker build failing?”

## Deliverable

* Error log
* AI explanation
* Fix applied

---

# Task 4: AI for Runtime Issue

## Step

Run container:

```bash
docker run -p 8080:8080 login-app
```

Introduce issue:

* WAR not loading
* App context not accessible

## Ask AI:

* “Why is my Tomcat app not loading?”

## Deliverable

* Problem description
* AI suggestion
* Fix

---

# Task 5: AI for CI/CD Pipeline

## Step

Ask AI:

* “Create a CI pipeline for this Dockerized app”

## Expected Output (example with GitHub Actions)

```yaml
name: Docker CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker Image
        run: docker build -t login-app .
```

## Deliverable

* CI pipeline YAML
* Explanation

---

# Task 6: AI for Security Improvement

## Step

Ask AI:

* “How to secure this Dockerfile?”

## Expected Suggestions

* Use non-root user
* Scan image
* Avoid latest tag
* Reduce image size

## Deliverable

* Security improvements applied

---


* Convert this into a 1-day hands-on lab
* Provide instructor solution sheet (with answers)
