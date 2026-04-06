# Spring PetClinic — Secure DevSecOps Pipeline with JFrog Platform

A production-grade CI/CD pipeline demonstrating DevSecOps best practices using **GitHub Actions** + **JFrog Artifactory** + **JFrog Xray** on the [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) application.

---

## Architecture Overview

```
Git Push ──► GitHub Actions ──► Maven Build (via Artifactory proxy)
                                      │
                                      ▼
                              JUnit Tests Run
                                      │
                                      ▼
                            Docker Image Build
                                      │
                                      ▼
                        JFrog Xray Scan (Quality Gate)
                          CVE  │  Secrets  │  Licenses
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
              Block + Alert       Promote to Prod Repo
```

---

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── ci-pipeline.yml     # GitHub Actions pipeline
├── src/                        # Spring PetClinic source
├── Dockerfile                  # Multi-stage, hardened Docker image
├── k8s-deployment.yaml         # Kubernetes deployment + service + ingress
├── README.md                   # This file
```

---

## Prerequisites

| Tool | Version |
|------|---------|
| Java | 21 (Temurin) |
| Maven | 3.9+ (wrapper included) |
| Docker | 24+ |
| JFrog Platform | Trial or Enterprise |
| kubectl | 1.28+ (for K8s deploy) |

---

## JFrog Platform Setup

### 1. Create Repositories in Artifactory

| Repo Key | Type | Package Type | Purpose |
|----------|------|--------------|---------|
| `libs-release-local` | Local | Maven | Published JARs |
| `libs-snapshot-local`| Local | Maven | Snapshot builds |
| `maven-remote`       | Remote| Maven | Proxy Maven Central |
| `libs-virtual`       | Virtual| Maven | Unified resolver endpoint |
| `docker-local`       | Local | Docker | Dev/staging images |
| `docker-prod`        | Local | Docker | Promoted prod images |

### 2. Configure Xray Policy

In **Xray → Policies**, create a policy named `devsecops-watch` with:
- **Block** on CVSS ≥ 8.0 (Critical/High)
- **Block** on exposed secrets
- **Warn** on GPL/AGPL license violations

Assign it to a **Watch** that covers `docker-local`.

### 3. GitHub Secrets

Add these to your GitHub repository secrets:

```
JFROG_URL       = https://yourorg.jfrog.io
JFROG_USER      = your-username
JFROG_PASSWORD  = your-api-key-or-token
```

---

## Running the Pipeline

The pipeline triggers automatically on push to `main` or `develop`, and on pull requests.

### Pipeline Jobs

| Job | Description |
|-----|-------------|
| `build-and-test` | Compile + run tests, resolve all deps via Artifactory |
| `docker-build` | Multi-stage Docker build, push to Artifactory |
| `security-scan` | Xray CVE + secrets + license scan — blocks on violations |
| `promote` | Promote image from `docker-local` to `docker-prod` (main only) |

---

## Running the Docker Image Locally

### Pull from Artifactory

```bash
# Authenticate
docker login yourorg.jfrog.io \
  --username YOUR_USER \
  --password YOUR_API_KEY

# Pull
docker pull yourorg.jfrog.io/docker-prod/spring-petclinic:latest

# Run
docker run --rm \
  -p 8080:8080 \
  --name petclinic \
  yourorg.jfrog.io/docker-prod/spring-petclinic:latest
```

Open **http://localhost:8080** in your browser.

### Build & Run Locally (without Artifactory)

```bash
# Build
docker build -t spring-petclinic:local .

# Run
docker run --rm -p 8080:8080 spring-petclinic:local
```

---

## Kubernetes Deployment

```bash
# Create namespace
kubectl create namespace petclinic

# Create image pull secret (from Artifactory)
kubectl create secret docker-registry jfrog-registry-secret \
  --namespace petclinic \
  --docker-server=yourorg.jfrog.io \
  --docker-username=YOUR_USER \
  --docker-password=YOUR_API_KEY

# Deploy
kubectl apply -f k8s-deployment.yaml

# Verify
kubectl get pods -n petclinic
kubectl get svc  -n petclinic

# Port-forward for local access
kubectl port-forward svc/spring-petclinic 8080:80 -n petclinic
```

---

## DevSecOps Best Practices Applied

### Repository Management
- **Virtual repositories** as the single resolver endpoint — no direct internet access from builds
- **Remote repository proxy** caches Maven Central through Artifactory
- **Promotion workflow**: images flow `docker-local → docker-prod` only after security gates pass
- **Immutable artifacts**: published packages are never overwritten; SHA-256 checksums enforced

### Secure Dependencies
- `settings.xml` mirrors `*` to Artifactory — all Maven deps go through the proxy
- Remote repository caching eliminates runtime internet exposure
- Transitive dependency resolution is scanned by Xray

### Docker Image Hardening
- Multi-stage build: JDK only in build stage; minimal JRE in runtime image
- Images pinned by **digest** (not tag) to prevent mutable base image attacks
- **Non-root user** (`appuser`) with least privilege
- `readOnlyRootFilesystem: true` in Kubernetes spec
- All capabilities dropped (`drop: ["ALL"]`)

### Quality Gates (Bonus)
- Xray watch blocks image push if any **Critical/High CVE** is found
- Secrets detection catches hardcoded tokens before they reach the registry
- License policy enforces approved SPDX identifiers

### Traceability & Auditability (Bonus)
- JFrog **Build Info** links every artifact to: Git SHA, pipeline run, environment, and committer
- **SBOM** (SPDX and CycloneDX) generated at Docker build time via Buildx
- **SLSA provenance attestations** published with every image

---

## Xray Scan Results

An automated `xray-scan-results.json` export is attached as a GitHub Actions artifact on every pipeline run. Download it from the **Actions → security-scan → Artifacts** section.

---

## License

Apache 2.0 — see [LICENSE](LICENSE)
