# cicd-pipeline-examples

Production Jenkins pipelines used at BizmerlinHR (ClayHR) for building, tagging, and pushing Docker images to AWS ECR. Both pipelines run on AWS Graviton (ARM64) infrastructure and follow a branch-based promotion strategy.

> These are real pipelines running in production — not written as examples.

---

## 📁 Repository structure

```
cicd-pipeline-examples/
├── Jenkinsfile.flask-application      # AskHR Python Flask app — branch-based build and ECR push
└── Jenkinsfile.docker-aggregator      # Multi-project WAR aggregator — collects 4 WAR files and builds combined Tomcat image
```

---

## 🔍 Pipeline 1 — Flask Application (AskHR)

**File:** `Jenkinsfile.flask-application`

### What it does
Builds a Python Flask application (AskHR) into a Docker image and pushes to AWS ECR based on branch. Uses Docker Buildx for ARM64 image builds targeting AWS Graviton instances.

### Branch strategy

| Branch | Build type | ECR repo | Image tag format |
|---|---|---|---|
| `developer` | Staging | `askhr-dev` | `askhrstaging-YYYYMMDD-HHMM` |
| `release` | Production release | `askhr-prod` | `askhrrelease-YYYYMMDD-HHMM` |
| Any feature branch | Build check only | `askhr-dev` | `feature-YYYYMMDD-HHMM` (no push) |

### Key features

- **Smart change detection** — pipeline only builds and pushes if files changed under `src/main/python/AskHR/**`. No code change = no unnecessary build.
- **Dynamic ECR registry** — account ID fetched at runtime via `aws sts get-caller-identity`. No hardcoded AWS account IDs in the pipeline.
- **Docker Buildx ARM64** — builds `linux/arm64` images for AWS Graviton (`t4g`) instances via `clayhrbuilder` Buildx builder.
- **Git commit traceability** — short and full Git SHA baked into every image as Docker labels (`git.commit.short`, `git.commit.full`, `git.branch`).
- **Feature branch safety** — feature branches run a build check (no `--push`) to catch Dockerfile errors before merge.
- **30-minute timeout** — prevents stuck builds from blocking the pipeline queue.
- **Release deploy instructions** — on `release` branch, pipeline prints exact deploy commands for core-prod and other regions (DEV/SG/CA/ZA).

### Pipeline stages

```
Set Build Context → ECR Login + Setup Buildx → Build Check (feature only)
                                              → Build and Push Staging (developer branch)
                                              → Build and Push Release (release branch)
                                              → Deploy Instructions (release branch only)
```

### Image label example
```
git.commit.short = a1b2c3d
git.commit.full  = a1b2c3d4e5f6...
git.branch       = release
build.date       = 20260620
build.type       = release
```

---

## 🔍 Pipeline 2 — Docker Aggregator (ClayHR Tomcat)

**File:** `Jenkinsfile.docker-aggregator`

### What it does
Collects the latest successful WAR file from 4 separate Java projects, combines them into a single Tomcat Docker image, and pushes to AWS ECR. Triggered by any downstream project build.

### Projects aggregated

| Project | Source WAR | Renamed to |
|---|---|---|
| HCM | `rm.war` | `HCM.war` |
| BMOne | `bmone.war` | `BMOne.war` |
| BMIntegration | `bmintegrations.war` | `BMIntegration.war` |
| Jobboard | `job-board.war` | `Jobboard.war` |

### Key features

- **Multi-project aggregation** — uses Jenkins `copyArtifacts` plugin to pull the last successful WAR from each of 4 projects automatically.
- **WAR verification** — verifies all 4 WARs exist before attempting Docker build. Fails fast with a clear error if any WAR is missing.
- **Smart image tagging** — tag format: `clayhrstaging-{YYYYMMDD-HHMM}-{buildnum}-{user}-{branch}`. Username sanitised — strips `@clayhr.com`, replaces special characters.
- **Full audit trail** — Docker image labels record triggering user, source job, branch, and build number.
- **Slack notifications** — every stage (WAR collection, build, push, final status) sends a Slack message to `#jenkins-build-notifications`.
- **Concurrent build protection** — `disableConcurrentBuilds()` prevents race conditions when multiple projects trigger simultaneously.
- **Automatic cleanup** — removes local Docker images and WAR files after successful ECR push to keep disk usage clean.

### Pipeline stages

```
Prepare → Collect WARs → Build Docker Image → Push to ECR → Cleanup
```

### Image tag example
```
clayhrstaging-20260620-1300-42-aniket-channa-main
│             │              │  │              │
│             │              │  │              └── sanitised branch name
│             │              │  └── sanitised username (stripped @domain)
│             │              └── Jenkins build number
│             └── timestamp YYYYMMDD-HHMM
└── environment prefix
```

### Triggered by parameters

```groovy
TRIGGERED_BY_USER    // Jenkins user who triggered (e.g. aniket.channa@clayhr.com)
TRIGGERED_BY_JOB     // Which project triggered (e.g. HCM, BMOne)
TRIGGERED_BY_BRANCH  // Branch that was built
```

---

## ⚙️ Prerequisites

**Jenkins plugins required:**
- `copyArtifacts` — for WAR collection in aggregator pipeline
- `Slack Notification` — for build notifications
- `Docker Pipeline` — for Docker build steps
- `Git` — for branch detection and commit SHA

**AWS setup required:**
- IAM role on Jenkins agent with permissions: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `sts:GetCallerIdentity`
- ECR repositories created: `askhr-dev`, `askhr-prod`, `clayhr-tomcat`
- Docker Buildx builder named `clayhrbuilder` configured on Jenkins agent

---

## 🖥️ Infrastructure context

Both pipelines build `linux/arm64` images targeting AWS Graviton (`t4g`) EC2 instances — part of a cost optimisation initiative that achieved ~30% cost reduction by migrating from `t3.large` to Graviton-based instances.

Images are stored in AWS ECR (`us-east-1`) and deployed to ECS clusters via separate deployment scripts.

---

## 📌 Related repos

- [dockerfile-and-compose-examples](https://github.com/aniketchanna/dockerfile-and-compose-examples) — Dockerfiles and Compose setup these pipelines build from
- [aws-cloudformation-templates](https://github.com/aniketchanna/aws-cloudformation-templates) — CloudFormation templates for the underlying ECS and ECR infrastructure
- [linux-automation-scripts](https://github.com/aniketchanna/linux-automation-scripts) — Server automation scripts used alongside these pipelines

---

## 👤 Author

**Aniket Channa** — Senior DevOps Engineer
8 years experience · AWS (70%) · Azure/GCP (30%) · Open to remote worldwide
[LinkedIn](https://linkedin.com/in/aniketchanna) · [GitHub](https://github.com/aniketchanna) · IST (UTC+5:30)
