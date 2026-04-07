# Spring PetClinic — Secure DevSecOps Pipeline with JFrog Platform

A production-grade CI/CD pipeline demonstrating DevSecOps best practices using **GitHub Actions** + **JFrog Artifactory** + **JFrog Xray** on the [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) Java application.

> **Live demo instance**: `axde.jfrog.io`
> **GitHub repo**: `github.com/axlde/spring-petclinic`

---

## What This Demonstrates

| Requirement | Implementation |
|---|---|
| Compile the code | `./mvnw compile` via GitHub Actions |
| Run the tests | `./mvnw test` — JUnit, all passing |
| Package as runnable Docker image | Multi-stage Dockerfile, pushed to Artifactory |
| Secure dependency resolution | All Maven deps routed through Artifactory `libs-virtual` — zero direct internet |
| Repository management best practices | Local + Remote + Virtual repos, promotion workflow |
| Quality gates (bonus) | Xray policy blocks on High/Critical CVEs |
| Traceability & auditability (bonus) | Build info linked to Git SHA, branch, pipeline run |

---

## Architecture

```
Git Push (axlde/spring-petclinic)
        ↓
GitHub Actions (ubuntu-latest)
        ↓
┌─────────────────────────────────────────┐
│  Job 1: Build & Test                    │
│  Maven → Artifactory libs-virtual       │
│  (proxies Maven Central, caches deps)   │
│  Compile → Test → Package JAR           │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Job 2: Docker Build & Push             │
│  Multi-stage build (JDK → JRE)         │
│  Push to axde.jfrog.io/docker-local    │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Job 3: Xray Security Scan              │
│  Scans all 258 components               │
│  Blocks pipeline on High/Critical CVEs  │
│  Results: 20 CVEs, 0 Critical,          │
│           0 Applicable (contextual)     │
└────────────────┬────────────────────────┘
                 ↓
    JFrog Xray Scan (Quality Gate)
      CVE  │  Secrets  │  Licenses
           │
┌──────────┴──────────┐
▼                     ▼
Block + Alert   Promote to docker-prod
```

---

## JFrog Platform Setup

### Repositories Created in Artifactory

| Repository Key | Type | Package | Purpose |
|---|---|---|---|
| `libs-release-local` | Local | Maven | Published JARs |
| `maven-remote` | Remote | Maven | Proxy → Maven Central |
| `libs-virtual` | Virtual | Maven | Single resolver URL for builds |
| `docker-local` | Local | Docker | Dev/staging images post-build |
| `docker-prod` | Local | Docker | Promoted production images |

### Why Virtual Repositories?

Instead of pointing Maven directly at Maven Central, all builds use a single virtual URL:

```
https://axde.jfrog.io/artifactory/libs-virtual
```

This virtual repo aggregates `libs-release-local` (local artifacts) and `maven-remote` (proxied Maven Central). Benefits:
- Builds never touch the internet directly
- If Maven Central goes down, builds still work (cached)
- Full audit trail of every dependency ever downloaded
- Add/remove upstream sources without changing build config

### Xray Security Policy

- **Policy name**: `block-critical-vulnerabilities`
- **Rule**: Block on High and Critical CVEs
- **Watch**: `docker-local-watch` monitors `docker-local` repo
- **Result**: Pipeline exits with code 1 if violation found

---

## Prerequisites

| Tool | Version |
|---|---|
| Java | 21 (Temurin) |
| Maven | Wrapper included (`./mvnw`) |
| Docker | 24+ |
| JFrog Platform | Trial or Enterprise |

---

## How Secure Dependency Resolution Works

When GitHub Actions runs the pipeline, it creates a `~/.m2/settings.xml` on the runner that mirrors ALL Maven requests to Artifactory:

```xml
<mirrors>
  <mirror>
    <id>jfrog-artifactory</id>
    <mirrorOf>*</mirrorOf>
    <url>https://axde.jfrog.io/artifactory/libs-virtual</url>
  </mirror>
</mirrors>
```

The `mirrorOf: *` means every dependency request — including transitive dependencies — goes through Artifactory. The runner never contacts Maven Central directly. Artifactory fetches it once, caches it in `maven-remote`, and serves it on every subsequent build instantly.

---

## Dockerfile Explained

```dockerfile
# Stage 1: Build — JDK needed to compile
FROM eclipse-temurin:21-jdk-jammy AS build
# compiles and packages the JAR

# Stage 2: Runtime — only JRE needed to run
FROM eclipse-temurin:21-jre-jammy AS runtime
# Non-root user for least privilege
RUN groupadd --system appgroup && useradd --system --gid appgroup appuser
USER appuser
```

Key security decisions:
- **Multi-stage**: Final image has no JDK, no build tools, no source code — attack surface minimised
- **Non-root user**: Container cannot write to system paths even if compromised
- **JRE only**: ~200MB smaller than JDK image
- **Health check**: Kubernetes and Docker know when app is ready via `/actuator/health`

---

## Running the Pipeline

The pipeline triggers automatically on every push to `main`.

### GitHub Secrets Required

Add these in `Settings → Secrets and variables → Actions`:

| Secret | Value |
|---|---|
| `JFROG_URL` | `https://axde.jfrog.io` |
| `JFROG_USER` | your JFrog email |
| `JFROG_TOKEN` | your JFrog identity token |

### Pipeline Jobs

| Job | What it does |
|---|---|
| `build-and-test` | Configures Maven to use Artifactory, compiles, tests, packages JAR |
| `docker-build` | Builds multi-stage Docker image, pushes to `docker-local` |
| `xray-scan` | Pulls image, runs Xray scan, fails build on policy violation |

---

## Running the Docker Image Locally

### Pull from Artifactory

```bash
# Authenticate
docker login axde.jfrog.io \
  --username YOUR_EMAIL \
  --password YOUR_TOKEN

# Pull
docker pull axde.jfrog.io/docker-local/spring-petclinic:latest

# Run
docker run --rm \
  -p 8080:8080 \
  --name petclinic \
  axde.jfrog.io/docker-local/spring-petclinic:latest
```

Open **http://localhost:8080** in your browser.

---

## Kubernetes Deployment

```bash
# Create namespace
kubectl create namespace petclinic

# Create image pull secret
kubectl create secret docker-registry jfrog-registry-secret \
  --namespace petclinic \
  --docker-server=axde.jfrog.io \
  --docker-username=YOUR_EMAIL \
  --docker-password=YOUR_TOKEN

# Deploy
kubectl apply -f k8s-deployment.yaml

# Verify
kubectl get pods -n petclinic

# Port-forward for local access
kubectl port-forward svc/spring-petclinic 8080:80 -n petclinic
```

---

## Xray Scan Results (Bonus Deliverable)

Real scan results from `spring-petclinic:latest` scanned on 06 Apr 2026:

| Severity | Count | Applicable |
|---|---|---|
| Critical | 0 | 0 |
| High | 4 | 0 |
| Medium | 6 | — |
| Low | 10 | — |
| **Total** | **20** | **0** |

**Key finding**: Xray contextual analysis confirmed that none of the 4 High CVEs are reachable in our code — eliminating false positives that would waste developer time.

The full JSON export is available in the `xray-results/` folder in this repo:

| File | Contents |
|---|---|
| `Docker_0616aed_Security_Export.json` | Full CVE details, severity, affected packages |
| `Docker_0616aed_Violations_Export.json` | Policy violations against `block-critical-vulnerabilities` |
| `Docker_0616aed_Scan_Status_Export.json` | Overall scan status and component summary |

---

## DevSecOps Best Practices Applied

**Secure dependencies**: All Maven requests mirror to `libs-virtual`. No artifact reaches the build from the internet directly.

**Multi-stage Docker build**: JDK only in build stage. Minimal JRE in runtime. Non-root user. Smaller, more secure image.

**Shift-left security**: Xray scans before the image reaches any environment — not post-deployment.

**Promotion workflow**: Images flow `docker-local → docker-prod` only after passing the Xray gate. Nothing unscanned runs in production.

**Full traceability**: Every artifact linked to its Git SHA, pipeline run, branch, and committer via JFrog Build Info.

**SBOM ready**: 258 components inventoried. Exportable as SPDX or CycloneDX for SOC2, NIS2, FedRAMP, CRA compliance.

---

## Repository Structure

```
spring-petclinic/
├── .github/
│   └── workflows/
│       └── ci-pipeline.yml     # GitHub Actions pipeline
├── xray-results/               # Xray scan JSON exports (bonus deliverable)
│   ├── Docker_0616aed_Security_Export.json
│   ├── Docker_0616aed_Violations_Export.json
│   └── Docker_0616aed_Scan_Status_Export.json
├── src/                        # Spring PetClinic source code
├── Dockerfile                  # Multi-stage hardened Docker image
├── k8s-deployment.yaml         # Kubernetes deployment + service
├── pom.xml                     # Maven build descriptor
└── README.md                   # This file
```

---

## License

Apache 2.0
