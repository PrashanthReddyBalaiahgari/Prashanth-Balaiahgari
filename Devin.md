# Devin AI & Harness OCP Integration Guide

This repository documents the end-to-end integration of **Devin AI** for automated security vulnerability remediation, GitHub Pull Request (PR) workflows, release management, and deployment using **Harness** onto **Red Hat OpenShift Container Platform (OCP)**.

---

## 🏗️ Architecture & Workflow Overview

```
 ┌────────────────┐     ┌────────────────┐     ┌────────────────┐
 │  Vulnerability │ ──> │    Devin AI    │ ──> │  GitHub Repos  │
 │   Scanner      │     │  Fix Creation  │     │ Pull Request   │
 └────────────────┘     └────────────────┘     └────────────────┘
                                                       │
                                                       ▼
 ┌────────────────┐     ┌────────────────┐     ┌────────────────┐
 │ Deployment on  │ <── │ Harness CI/CD  │ <── │ PR Merge / Tag │
 │  Red Hat OCP   │     │ Pipeline Run   │     │ Release Event  │
 └────────────────┘     └────────────────┘     └────────────────┘
```

---

## 🔗 Connecting Devin with GitHub Repositories

To enable Devin to interact directly with your codebase, you must connect it via the official Devin GitHub App or a Personal Access Token (PAT).

### Option 1: Official GitHub App (Recommended)

1. **Navigate to Devin Integrations:**
   * Log into your Devin workspace at [devin.ai](https://devin.ai).
   * Go to **Organization Settings** > **Integrations**.
2. **Install the GitHub App:**
   * Click **Connect** next to GitHub.
   * Select the target GitHub Organization or User Account.
3. **Grant Repository Permissions:**
   * Choose **All repositories** or select specific code repositories.
   * Ensure Devin is granted permissions to read contents, create branches, and open/update Pull Requests (PRs).
   * Click **Install & Authorize**.
4. **Usage:**
   * Reference your repositories by URL in your Devin prompts (e.g., `"Clone https://github.com/your-org/your-repo and fix issue #12"`).

### Option 2: Fine-Grained Personal Access Token (PAT)

For self-hosted GitHub Enterprise instances or granular privilege constraints:
1. In GitHub, go to **Settings** > **Developer Settings** > **Personal Access Tokens** > **Fine-grained tokens**.
2. Generate a token with access to your target repository and grant **Read & Write** permissions for *Contents*, *Issues*, and *Pull Requests*.
3. Add the token to Devin's **Workspace Secrets** under `GITHUB_TOKEN`.
4. Pass the authenticated clone URL format to Devin when prompting:
   `https://<token>@github.com/your-org/your-repo.git`

---

## 🛠️ End-to-End Components & Setup

### 1. Automated Vulnerability Remediation
* **Triggering Devin:**
  * Configure your vulnerability scanner (e.g., Snyk, Veracode, Dependabot, or Trivy) to invoke Devin via API or Webhook when a high/critical CVE is identified.
  * Alternatively, assign a GitHub Issue with the label `devin-fix` to trigger the Devin GitHub App integration.
* **Pull Request Rules:**
  * Devin opens a draft PR targeting the designated default branch (`main` or `develop`).
  * PRs include:
    * Detailed description of the remediated vulnerability.
    * Links to relevant CVE advisories.
    * Unit/integration test results confirming the fix doesn't break existing functionality.

### 2. PR Review & Validation Guidelines
* **Automated Checks:** GitHub Actions / Status Checks execute linting, unit tests, and security scans on Devin's PR branch.
* **Human Review:** Require at least **one peer review** approval prior to merging.
* **Merge Strategy:** Use **Squash and Merge** to keep the Git history clean and traceable.

---

## 🚀 Release Management

We follow **Semantic Versioning (SemVer)** for release tagging and artifact generation.

* **Branching Strategy:** Short-lived feature/fix branches with trunk-based integration.
* **Creating a Release:**
  1. Once Devin's PR is merged into `main`, trigger a release workflow via GitHub Releases or push a tag (`vX.Y.Z`).
  2. The release trigger generates:
     * Built container image pushed to the enterprise registry (e.g., Quay.io, JFrog Artifactory, or Harbor).
     * Updated release notes aggregating merged PRs and vulnerability fixes.

---

## ☸️ Harness Deployments on OpenShift (OCP)

Deployments to OpenShift are orchestrated automatically via **Harness Pipelines**.

### Harness Pipeline Architecture
* **Trigger:** GitHub Webhook on Release Tag Creation or Container Registry Image Push event.
* **Pipeline Stages:**
  1. **Dev/QA Deployment:** Automated deployment to non-production OCP namespaces.
  2. **Integration Verification:** Automated smoke and regression testing in OCP.
  3. **Production Gate:** Manual approval step within Harness.
  4. **Production Deployment:** Rolling update / Canary deployment to the Production OCP cluster.

### OCP Prerequisites & Configuration
* **Harness Delegate:** Must be installed inside your OCP cluster with appropriate `ServiceAccount` and `RoleBinding` permissions.
* **Kubernetes Manifests / Helm / Kustomize:** Stored in this repository under `/deploy/ocp`.
* **OpenShift Specifics:**
  * Define `Route` resources for external network access.
  * Align `SecurityContextConstraints` (SCC) with your organizational security policies (e.g., `restricted-v2`).

---

## 📋 Step-by-Step Operator Checklist

1. **Vulnerability Flagged (Automated Trigger):** Scanner identifies a vulnerability in the codebase and triggers Devin via Webhook/API.
   * *Verification:* Confirm Devin session creation in the Devin dashboard.
2. **Devin PR Review (GitHub Workflow):** Review Devin's generated PR, confirm test suite execution, and approve the PR.
   * *Verification:* Status checks pass green on the GitHub PR before merging.
3. **Tag & Release (Release Management):** Merge the PR into `main` and publish a release tag (e.g., `v1.2.1`).
   * *Verification:* Release tag is created and container artifact is published to the registry.
4. **Harness OCP Pipeline Execution (Deployment):** Harness detects the new tag, pulls artifacts, and executes deployment onto OpenShift.
   * *Verification:* Validate running pods via OpenShift CLI (`oc get pods -n <namespace>`) and ensure the Route endpoint responds.

---

## 🔐 Troubleshooting & Support

* **Devin Integration / PR Issues:** Check the execution logs provided in the Devin PR comment link.
* **Harness Pipeline Failures:** Inspect **Harness Execution Logs** for deployment step failures (e.g., image pull secrets, resource quotas).
* **OCP Pod Errors:** Execute `oc logs deployment/<app-name> -n <namespace>` or `oc describe pod <pod-name>`.