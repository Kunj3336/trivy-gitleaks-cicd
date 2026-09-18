# Research & Development: Trivy & Gitleaks on CI/CD Pipeline

## Executive Summary
This project establishes an automated DevSecOps pipeline evaluating **Gitleaks** (secret detection) and **Aqua Security Trivy** (container vulnerability scanning)[cite: 4]. The implementation defines enforceable security gates within GitHub Actions to prevent sensitive credentials from leaking into Git history and block vulnerable container images from reaching deployment environments such as HashiCorp Nomad clusters[cite: 1, 2, 4].

---

## Architecture & Scanning Workflow

Git Push / Pull Request
         │
         ▼
[Stage 1: Gitleaks] ────────(Secret Found?)────────► [BUILD FAILED / Pipeline Blocked]
         │ (Clean)
         ▼
[Stage 2: Application Tests]
         │
         ▼
[Stage 3: Docker Build]
         │
         ▼
[Stage 4: Trivy Container Scan] ──(Critical CVEs?)──► [PROMOTION BLOCKED]
         │ (Threshold Passed)
         ▼
[Stage 5: SARIF & JSON Reporting]
         │
         ▼
Artifact Registry / Nomad Cluster Deployment

### Strategic Placement
* **Pre-Build (Gitleaks):** Evaluates raw source code and Git commit history before Docker build steps execute[cite: 2, 4]. Catching secrets before containerization prevents API tokens and keys from becoming embedded in intermediate image layers[cite: 2, 4].
* **Post-Build (Trivy):** Analyzes the built container image artifact (`trivy-gitleaks-poc-app:latest`)[cite: 2, 4, 5]. Scans the base OS file system (Debian) and application language dependencies (Python packages) for Common Vulnerabilities and Exposures (CVEs)[cite: 2, 4, 10].
* **Pre-Deploy (Security Gate):** Applies failure policies to prevent insecure artifacts from being tagged, pushed to the registry, or deployed to orchestration clusters[cite: 2, 3, 4].

---

## Security Gate & Failure Policies

| Stage | Tool | Trigger Condition | Enforcement Policy |
| :--- | :--- | :--- | :--- |
| **Commit / PR** | Gitleaks | Hardcoded API keys, tokens, or private credentials detected. | **Hard Block (Exit Code 1):** Halts the workflow immediately before build execution. |
| **Post-Build** | Trivy | `CRITICAL` or `HIGH` severity vulnerabilities with available vendor fixes. | **Hard Block (Exit Code 1):** Prevents artifact promotion and halts deployment to Nomad clusters. |
| **Vulnerability Visibility** | Trivy | `MEDIUM` or `LOW` severity findings, or vulnerabilities without current fixes. | **Warn & Log (Exit Code 0):** Issues are recorded in SARIF/JSON and uploaded to GitHub Security tab for scheduled triage. |

---

## Output Format Evaluation

* **Table (CLI Standard):** Formatted tabular output printed to terminal `stdout`[cite: 4]. Optimized for local developer execution, debugging, and immediate triage during manual testing[cite: 4].
* **JSON (`trivy-report.json`):** Machine-readable structured payload containing full vulnerability metadata, layer traces, and CVSS scoring[cite: 2, 4]. Used for SIEM integration, log aggregation, and custom security scripts[cite: 2, 4].
* **SARIF (`trivy-report.sarif`):** Static Analysis Results Interchange Format (OASIS standard)[cite: 4]. Natively ingests into GitHub Security (`Code scanning alerts`), SonarQube, and enterprise vulnerability dashboards for centralized security tracking[cite: 2, 4, 16].

---

## Tool Comparison & Trade-Offs

### Container Vulnerability Scanners
| Feature | Aqua Security Trivy | Anchore Grype | Clair | Snyk Container |
| :--- | :--- | :--- | :--- | :--- |
| **Scan Scope** | OS packages, language dependencies, IaC, secrets, misconfigurations | OS packages and language packages | OS package analysis (RPM, DEB, APK) | Container layers, application code, dependencies |
| **Database & Cache** | Daily updated DB cache; offline scanning supported | Frequent DB updates via Anchore feed | Requires dedicated database engine (PostgreSQL) | Cloud-assisted vulnerability database |
| **CI/CD Integration** | Official GitHub Action, standalone binary, Docker image | CLI / GitHub Action | Complex server/client deployment | CLI / SaaS webhooks |
| **License Model** | Open Source (Apache 2.0) | Open Source (Apache 2.0) | Open Source (Apache 2.0) | Commercial freemium with scan limits |

### Secret Detection Scanners
| Feature | Gitleaks | Trufflehog | detect-secrets (Yelp) |
| :--- | :--- | :--- | :--- |
| **Detection Engine** | Regular expression patterns + Shannon entropy | Regex rules + live verification (API pinging) | Heuristic rules + baseline tracking |
| **Performance** | High-speed Go binary; low memory footprint | Slower execution due to network verification calls | Fast, optimized for local developer hooks |
| **Pre-commit / CI** | Native support across pre-commit hooks and CI engines | CI/CD and terminal execution | Primarily focused on pre-commit baselines |

---

## Harbor Registry & Nomad Deployment Integration

* **Pluggable Harbor Integration:** Trivy operates as the default pluggable security scanner inside VMware Harbor[cite: 1, 4]. Container images pushed to Harbor can be automatically analyzed on push[cite: 1, 4]. Harbor's **Deployment Security Policy** can be configured to block the pulling or replication of any image containing `CRITICAL` CVEs[cite: 1, 4].
* **Nomad Deployment Gating:** Nomad job specifications schedule workloads referencing specific image tags[cite: 2, 3, 4]. Integrating Trivy into the CI pipeline ensures that only images that have passed all defined vulnerability thresholds receive the production tag (`:vX.Y.Z`) and push confirmation[cite: 2, 3, 4]. Jobs targeting Nomad (`nomad job run`) only execute after the container image passes the pre-deploy gate[cite: 2, 3, 4].

---

## Proof of Concept Verification

### 1. Application Container Build
Building the baseline containerized service (`trivy-gitleaks-poc-app:latest`) locally using Docker:
![Docker Build Success](screenshots/01_docker_build_success.png)

### 2. Local Secret Detection (Negative Test)
Gitleaks detecting an injected mock secret token pattern in `config.env` using regular expressions and entropy calculations:
![Gitleaks Secret Detected](screenshots/02_gitleaks_secret_detected.png)

### 3. Local Secret Scan Clean Pass (Positive Test)
Gitleaks executing against the repository after fixture cleanup, confirming zero leaks detected:
![Gitleaks Clean Pass](screenshots/03_gitleaks_clean_pass.png)

### 4. Local Trivy Container Image Scan
Trivy inspecting the base image layers and application packages, outputting findings in a tabular terminal grid:
![Trivy Local Scan Table](screenshots/04_trivy_local_scan_table.png)

### 5. Automated CI Pipeline Success
GitHub Actions workflow executing the full sequence cleanly (Checkout -> Gitleaks -> Docker Build -> Trivy Scan -> SARIF Upload):
![GitHub Actions Pass](screenshots/05_github_actions_pass.png)

### 6. Security Gate Pipeline Failure
CI security gate blocking the build immediately at the Gitleaks stage when a test credential is introduced:
![GitHub Actions Gate Failure](screenshots/06_github_actions_gate_failure.png)

### 7. Centralized Security Reporting (SARIF Ingestion)
GitHub Code Scanning dashboard displaying findings uploaded via Trivy's SARIF report:
![GitHub Security SARIF Alerts](screenshots/07_github_security_sarif_alerts.png)

---

## Engineering Challenges & Remediation

### Challenge 1: Cross-Scanner Report Conflict
* **Problem:** During local testing, running Gitleaks against the project root after running a Trivy report export caused Gitleaks to flag dummy GPG keys embedded inside Trivy's generated `reports/trivy-report.json`[cite: 8].
* **Root Cause:** Scanning output directories containing other security tool metadata introduces false positives from test data or base image package signatures[cite: 8].
* **Solution:** Decoupled scanning stages inside the CI workflow[cite: 4]. Gitleaks runs strictly on raw source code in pre-build, before any container image scans or report artifacts are generated[cite: 4].

### Challenge 2: GitHub Actions Code Scanning API Authorization
* **Problem:** The GitHub Actions workflow failed during the `Upload Trivy SARIF Report` step with error: `Resource not accessible by integration` (exit code failure)[cite: 13, 17].
* **Evidence:**
![GitHub Action Upload Error](screenshots/08_github_action_error.png)
* **Root Cause:** By default, the runner's `GITHUB_TOKEN` does not possess write access to the GitHub Code Scanning API endpoint[cite: 13].
* **Solution:** Added explicit token permissions in the workflow definition:
```
permissions:
  contents: read
  security-events: write
```
This granted the upload action permission to publish SARIF data to the repository's **Security** tab, resolving the error and allowing the pipeline to pass[cite: 14].