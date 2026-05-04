# MuleSoft 4.9 CI/CD Pipeline

Automated build and deploy pipeline for MuleSoft 4.9 using GitHub Actions on a self-hosted EC2 Linux runner. Supports three environments: **Development**, **Staging**, and **Production**.

---

## Table of Contents

- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Branching Strategy](#branching-strategy)
- [Prerequisites](#prerequisites)
- [EC2 Runner Setup](#ec2-runner-setup)
- [GitHub Configuration](#github-configuration)
- [Workflow Overview](#workflow-overview)
- [Deployment Targets](#deployment-targets)
- [Troubleshooting](#troubleshooting)

---

## Architecture

```
GitHub Repo
    │
    ├── PR to develop      →  CI Build Check (compile only)
    │
    ├── Merge to develop   →  Build → Deploy to DEV
    │
    ├── Merge to release/* →  Build → Deploy to STG  (requires approval)
    │
    └── Merge to main      →  Build → Deploy to PROD (requires approval)
                                        ↓
                                   Auto Git Tag
```

**Runner:** Self-hosted GitHub Actions agent on AWS EC2 (Linux)  
**Build Tool:** Maven 3.9+  
**Runtime:** MuleSoft 4.9  
**Target Platform:** Anypoint CloudHub

---

## Repository Structure

```
your-mule-project/
├── .github/
│   └── workflows/
│       ├── ci.yml              # Build check on every PR
│       ├── deploy-dev.yml      # Deploy to Development on merge to develop
│       ├── deploy-stg.yml      # Deploy to Staging on merge to release/*
│       └── deploy-prod.yml     # Deploy to Production on merge to main
├── src/
│   └── main/
│       ├── mule/               # Mule application flows
│       └── resources/          # Properties and configs
│           ├── dev.properties
│           ├── stg.properties
│           └── prod.properties
├── pom.xml
└── README.md
```

---

## Branching Strategy

| Branch | Purpose | Deploys To | Approval Required |
|--------|---------|------------|-------------------|
| `feature/*` | Developer work | — | — |
| `develop` | Integration branch | DEV | No |
| `release/*` | Release candidates | STG | Yes (1 reviewer) |
| `main` | Production-ready code | PROD | Yes (2 reviewers) |

### Workflow

```
feature/my-feature
        │
        └── PR → develop ──► DEV auto-deploy
                    │
                    └── PR → release/1.0 ──► STG (with approval)
                                  │
                                  └── PR → main ──► PROD (with approval)
```

---

## Prerequisites

### EC2 Instance Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| OS | Amazon Linux 2 / Ubuntu 22.04 | Ubuntu 22.04 LTS |
| RAM | 4 GB | 8 GB |
| Disk | 20 GB | 40 GB |
| Java | 17 | 17 |
| Maven | 3.9+ | 3.9+ |

### Software Installation

**Java 17 (Ubuntu):**
```bash
sudo apt update && sudo apt install -y openjdk-17-jdk
java -version
```

**Java 17 (Amazon Linux 2):**
```bash
sudo amazon-linux-extras enable corretto17
sudo yum install -y java-17-amazon-corretto
java -version
```

**Maven 3.9:**
```bash
wget https://downloads.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
sudo tar -xvzf apache-maven-3.9.6-bin.tar.gz -C /opt
sudo ln -s /opt/apache-maven-3.9.6 /opt/maven

# Add to PATH
sudo tee /etc/profile.d/maven.sh <<EOF
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export M2_HOME=/opt/maven
export PATH=\${M2_HOME}/bin:\${PATH}
EOF

source /etc/profile.d/maven.sh
mvn -version
```

---

## EC2 Runner Setup

### 1. Register the Runner

In your GitHub repository, go to:  
**Settings → Actions → Runners → New self-hosted runner → Linux**

Copy the commands shown in the GitHub UI and run them on your EC2 instance:

```bash
mkdir actions-runner && cd actions-runner

# Download runner (use the exact URL from GitHub UI)
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/vX.X.X/actions-runner-linux-x64-X.X.X.tar.gz

tar xzf ./actions-runner-linux-x64.tar.gz

# Configure (token is provided by GitHub UI — it expires in 1 hour)
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO --token YOUR_TOKEN
```

### 2. Install as a System Service

```bash
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

### 3. Verify Runner is Online

Go to **GitHub → Settings → Actions → Runners** and confirm the runner shows as **Idle** (green dot).

### 4. Configure Maven Settings

Create `~/.m2/settings.xml` on the EC2 instance to authenticate with Anypoint:

```xml
<settings>
  <servers>
    <server>
      <id>anypoint-exchange-v3</id>
      <username>${anypoint.username}</username>
      <password>${anypoint.password}</password>
    </server>
    <server>
      <id>MuleRepository</id>
      <username>${anypoint.username}</username>
      <password>${anypoint.password}</password>
    </server>
  </servers>

  <profiles>
    <profile>
      <id>mule</id>
      <repositories>
        <repository>
          <id>MuleRepository</id>
          <url>https://repository.mulesoft.org/nexus/content/repositories/public/</url>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>MuleRepository</id>
          <url>https://repository.mulesoft.org/nexus/content/repositories/public/</url>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>mule</activeProfile>
  </activeProfiles>
</settings>
```

---

## GitHub Configuration

### 1. Secrets

Go to **Settings → Secrets and Variables → Actions** and add:

| Secret | Description |
|--------|-------------|
| `ANYPOINT_USERNAME` | Anypoint Platform username (or Connected App Client ID) |
| `ANYPOINT_PASSWORD` | Anypoint Platform password (or Connected App Client Secret) |
| `ANYPOINT_ENV_DEV` | Anypoint environment name, e.g. `Development` |
| `ANYPOINT_ENV_STG` | Anypoint environment name, e.g. `Staging` |
| `ANYPOINT_ENV_PROD` | Anypoint environment name, e.g. `Production` |
| `APP_NAME` | Your CloudHub application base name |
| `BUSINESS_GROUP_ID` | Your Anypoint Business Group ID (GUID) |

> **Security Note:** Prefer **Connected Apps** over username/password credentials.  
> Create one in: Anypoint Platform → Access Management → Connected Apps  
> Grant scope: `Deploy Applications`, `Read Applications`

### 2. Environments

Go to **Settings → Environments** and create three environments:

| Environment | Protection Rules |
|-------------|-----------------|
| `development` | No approval required |
| `staging` | Require 1 reviewer |
| `production` | Require 2 reviewers + 5-minute wait timer |

### 3. Branch Protection Rules

Go to **Settings → Branches → Add rule**:

| Branch Pattern | Rules |
|----------------|-------|
| `main` | Require PR, require status checks to pass, require 2 approvals, no force push |
| `develop` | Require PR, require status checks to pass, require 1 approval |
| `release/*` | Require PR, require status checks to pass |

---

## Workflow Overview

### `ci.yml` — Build Check (runs on every PR)

Triggers on pull requests targeting `develop`, `release/*`, or `main`.  
Compiles the project to catch build errors before merge. No deployment occurs.

```yaml
name: CI - Build Check

on:
  pull_request:
    branches: [develop, release/*, main]

jobs:
  build:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Compile
        run: |
          mvn clean compile -DskipTests \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -B -q
```

---

### `deploy-dev.yml` — Deploy to Development

Triggers automatically on every push/merge to the `develop` branch.

```yaml
name: Deploy to Development

on:
  push:
    branches: [develop]

jobs:
  deploy-dev:
    runs-on: self-hosted
    environment: development

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Build
        run: |
          mvn clean package -DskipTests \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -B -q

      - name: Deploy to DEV
        run: |
          mvn deploy -DskipTests \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -Denvironment=${{ secrets.ANYPOINT_ENV_DEV }} \
            -Dapp.name=${{ secrets.APP_NAME }}-dev \
            -Dbusiness.group.id=${{ secrets.BUSINESS_GROUP_ID }} \
            -B
```

---

### `deploy-stg.yml` — Deploy to Staging

Triggers on push/merge to `release/*` branches. Requires reviewer approval (set in GitHub Environments).

```yaml
name: Deploy to Staging

on:
  push:
    branches: [release/*]

jobs:
  deploy-stg:
    runs-on: self-hosted
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Build
        run: |
          mvn clean package -DskipTests \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -B -q

      - name: Deploy to STG
        run: |
          mvn deploy -DskipTests \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -Denvironment=${{ secrets.ANYPOINT_ENV_STG }} \
            -Dapp.name=${{ secrets.APP_NAME }}-stg \
            -Dbusiness.group.id=${{ secrets.BUSINESS_GROUP_ID }} \
            -B
```

---

### `deploy-prod.yml` — Deploy to Production

Triggers on push/merge to `main`. Requires 2 reviewer approvals and a wait timer. Automatically tags the release after a successful deployment.

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy-prod:
    runs-on: self-hosted
    environment: production

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Build
        run: |
          mvn clean package -DskipTests \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -B -q

      - name: Deploy to PROD
        run: |
          mvn deploy -DskipTests \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -Denvironment=${{ secrets.ANYPOINT_ENV_PROD }} \
            -Dapp.name=${{ secrets.APP_NAME }} \
            -Dbusiness.group.id=${{ secrets.BUSINESS_GROUP_ID }} \
            -B

      - name: Tag Release
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
          git tag -a "v${VERSION}" -m "Production release v${VERSION}"
          git push origin --tags
```

---

## Deployment Targets

| Environment | Branch | App Name in CloudHub | Approval |
|-------------|--------|----------------------|----------|
| Development | `develop` | `{APP_NAME}-dev` | None |
| Staging | `release/*` | `{APP_NAME}-stg` | 1 reviewer |
| Production | `main` | `{APP_NAME}` | 2 reviewers |

### pom.xml — Mule Maven Plugin (CloudHub)

Ensure the following plugin block is in your `pom.xml`:

```xml
<plugin>
  <groupId>org.mule.tools.maven</groupId>
  <artifactId>mule-maven-plugin</artifactId>
  <version>4.2.0</version>
  <extensions>true</extensions>
  <configuration>
    <cloudHubDeployment>
      <uri>https://anypoint.mulesoft.com</uri>
      <muleVersion>4.9.0</muleVersion>
      <username>${anypoint.username}</username>
      <password>${anypoint.password}</password>
      <applicationName>${app.name}</applicationName>
      <environment>${environment}</environment>
      <businessGroupId>${business.group.id}</businessGroupId>
      <workerType>MICRO</workerType>
      <workers>1</workers>
      <region>us-east-1</region>
      <objectStoreV2>true</objectStoreV2>
    </cloudHubDeployment>
  </configuration>
</plugin>
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Runner shows as **Offline** in GitHub | Runner service stopped | SSH into EC2 → `cd actions-runner && sudo ./svc.sh status` → restart if needed |
| `mvn: command not found` | Maven not on PATH for runner user | Verify `/etc/profile.d/maven.sh` is sourced; restart runner service |
| `401 Unauthorized` on deploy | Invalid Anypoint credentials | Re-check GitHub secrets; rotate credentials if needed; consider switching to Connected Apps |
| `OutOfMemoryError` during build | Insufficient JVM heap | Add `MAVEN_OPTS: "-Xmx2g"` as an env var in the workflow step |
| Artifact not found in Exchange | Wrong `groupId` in `pom.xml` | Ensure `groupId` matches your Anypoint Org or Business Group ID exactly |
| Deployment stuck / timed out | CloudHub provisioning delay | Increase `deploymentTimeout` in the Mule Maven Plugin config |
| Build cache not working | Cache key mismatch | Clear the cache in **GitHub → Actions → Caches** and re-run |
| App deploys but won't start | Bad property config for environment | Check CloudHub logs in Anypoint Runtime Manager |

---

## Quick Reference — Runner Service Commands

```bash
# From the actions-runner directory on EC2
sudo ./svc.sh status    # Check runner status
sudo ./svc.sh start     # Start the runner
sudo ./svc.sh stop      # Stop the runner
sudo ./svc.sh restart   # Restart the runner

# View runner logs
journalctl -u actions.runner.* -f
```

---

## Checklist — First-Time Setup

- [ ] EC2 instance meets minimum requirements (Java 17, Maven 3.9+)
- [ ] GitHub Actions runner registered and showing **Idle** in GitHub
- [ ] `~/.m2/settings.xml` configured on EC2 with Anypoint credentials
- [ ] All GitHub Secrets added (credentials, environment names, app name, business group ID)
- [ ] GitHub Environments created (`development`, `staging`, `production`) with correct approval rules
- [ ] Branch protection rules enabled on `develop`, `release/*`, and `main`
- [ ] `pom.xml` has Mule Maven Plugin configured for CloudHub
- [ ] All four workflow YAML files committed under `.github/workflows/`
- [ ] End-to-end smoke test: merge a small change to `develop` and verify DEV deploys successfully
