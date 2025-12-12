# Pull Request Check Workflow ⚡

A reusable GitHub Actions workflow designed to validate .NET projects, scan for secrets and vulnerabilities, analyze code quality with SonarCloud, lint OpenAPI specs, and manage Docker image existence notifications—optimized for pull request pipelines.

## Overview
- Trigger: workflow_call (reusable workflow)
- Environments: Java (Zulu 23), Node.js (v22), .NET (configurable via input)
- Key Tools: Gitleaks, SonarCloud, Docker, Snyk, Spectral, ReportGenerator, dotnet-coverage

## Inputs
- environment (string, required): Runtime environment (default: DEV)
- testsFolder (string): Root tests dir (default: tests)
- msgCanalDeploy (string): Summary message (default provided)
- dotnetVersion (string, required): .NET SDK version (default: 9.x)
- projectBaseDir (string, required): Path to src (default: ./src)
- solutionFileName (string, required): Full .slnx file name
- mainProject (string, required): Main project name (used for paths)
- reportTitle (string, required): Test report display title (default: Canal DEPLOY)
- sonarExclusions (string, required): Comma-separated path exclusions for Sonar
- imageName (string, required): Image name for publishing
- openAPIFolder (string, required): Subfolder under build output for OpenAPI JSON

## Environment Variables
ANSI color codes; javaVersion=23; javaDistro=zulu; nodeVersion=22.

---

## Jobs and Steps

### 1) Build
- Purpose: Build .NET, run tests and coverage, security scans, SonarCloud analysis, OpenAPI linting, and publish artifacts.
- Runs-on: ubuntu-latest
- Outputs:
  - tag-version: from csproj extracted tag
  - now: timestamp
- Steps:
  - 🧭 Get Environment: Logs environment; sets NOW output.
  - 📥 Checkout: Sparse checkout src and tests.
  - 🔍 Get-Version from csproj: Reads Version/Prefix/Suffix from csproj to build a TAG; exports TAG and components.
  - 🛠️ Setup .NET: Install specified SDK.
  - ✅ Check .NET version: Print runtime info.
  - 🔍 Check tests: Detects presence of *Tests.csproj or *Test.csproj; sets HAS_TESTS output.
  - 🛠️🧪 Install reporting tools: dotnet-reportgenerator-globaltool & dotnet-coverage (if tests exist).
  - Event name: Echo event type.
  - 🧱📤 Gitleaks: Secret scanning with summary and artifact upload.
  - 🛠️ Setup JDK: Java for SonarCloud scanner.
  - 🔐 Sonar caches: Cache packages and scanner.
  - 🔐 Sonar scanner install: Install dotnet-sonarscanner globally.
  - 🐋 Setup Docker: Install Docker and check version (if tests exist).
  - 🧪 Run tests + coverage: dotnet test across testsFolder; generate HTML, Cobertura, MarkdownSummaryGithub, SonarQube reports; apply sonarExclusions; tag with run ids.
  - 🧪📤 Upload coverage report: Upload coveragereport directory.
  - 🔐 Build + Sonar (Test ✅, Sonar ✅): Full Sonar scan with coverage via dotnet-coverage (when both tests and sonar inputs flags are set).
  - 🔐 Build + Sonar (Test ⛔, Sonar ✅): Sonar scan without tests if no HAS_TESTS.
  - 🧪📤 Upload coverage report artifact (second upload): Upload Coverage-Report directory (note: name/path swap vs previous).
  - 🍵 Restore + Build: Restore and build main project to output_build.
  - 🍵 Snyk Setup: Install Snyk.
  - 🍵 Docker build: Build image with tag felipementel/<imageName> using project Dockerfile.
  - 🍵 Snyk Container test: Scan built image; export JSON results; requires SNYK_TOKEN.
  - 🍵 Snyk .NET scan: Scan solution; upload SARIF.
  - 🍵📤 Upload Snyk SARIF: Publish to GitHub Code Scanning.
  - 🍵📤 Upload Snyk Container artifact: Upload vuln.json.
  - 🎗️ Setup Node.js: Install Node v22 (if openAPIFolder provided).
  - 🎗️ Install Spectral: Global install.
  - 🎗️ Spectral Lint (DevOps rules): Lint OpenAPI JSON in output_build/<openAPIFolder>; stylish + HTML output; continue-on-error.
  - 🎗️ Spectral Lint (OWASP rules): Same inputs with OWASP ruleset; stylish + HTML; continue-on-error.
  - 🎗️📤 Upload Spectral results: Upload spectral-*.html artifacts.
  - 🧾 Job summary: Writes version and message to workflow summary.

### 2) Check Image
- Purpose: Determine if an image with generated tag exists in GHCR.
- Needs: build
- Outputs:
  - image-exists: true/false
- Steps:
  - 🔑 Login to GHCR with GITHUB_TOKEN
  - 🔎 Manifest inspect: Check ghcr.io/<actor>/deploy.<imageName>:<tag-version>; set IMAGE_EXISTS accordingly.

### 3) Notify Image Not Exists
- Purpose: Guide next steps when image is missing.
- Needs: [build, check-image], condition: image-exists == 'false'
- Steps:
  - 🧾 Summary: Note that image can be deployed; push to registry and publish on Azure Cloud.

### 4) Notify Image Exists
- Purpose: Avoid duplicate publishing.
- Needs: [build, check-image], condition: image-exists == 'true'
- Steps:
  - 🧾 Summary: Note that image with tag already exists; skip push and Azure publish.

---

## Secrets and Vars
- Secrets: GITLEAKS_LICENSE, SONAR_TOKEN, SONAR_PROJECT_KEY, SNYK_TOKEN, GITHUB_TOKEN (built-in)
- Vars: SONAR_ORGANIZATION

---

## Mermaid Diagram

```mermaid
flowchart TD
  A[workflow_call trigger] --> B(Build Job)
  B -->|outputs: tag-version, now| C(Check Image)
  C -->|IMAGE_EXISTS == false| D(Notify Image Not Exists)
  C -->|IMAGE_EXISTS == true| E(Notify Image Exists)

  subgraph Build
    B1[🧭 Get Environment (NOW)]
    B2[📥 Checkout (src, tests)]
    B3[🔍 Get-Version from csproj (TAG)]
    B4[🛠️ Setup .NET & ✅ dotnet --info]
    B5[🔍 Check tests (HAS_TESTS)]
    B6[🧱📤 Gitleaks]
    B7[🛠️ Setup JDK + 🔐 Sonar caches + install scanner]
    B8[🐋 Setup Docker (if tests)]
    B9[🧪 Test + Coverage + Report (if tests)]
    B10[🧪📤 Upload Coverage Report]
    B11[🔐 Sonar Scan w/ Coverage (if tests & sonar flags)]
    B12[🔐 Sonar Scan no tests (if no tests)]
    B13[🧪📤 Upload Coverage Report (alt path)]
    B14[🍵 Restore + Build → output_build]
    B15[🍵 Snyk setup + Container build]
    B16[🍵 Snyk Container test → vuln.json]
    B17[🍵 Snyk .NET scan → SARIF]
    B18[🍵📤 Upload SARIF]
    B19[🍵📤 Upload vuln.json]
    B20[🎗️ Node setup + Spectral install (if openAPIFolder)]
    B21[🎗️ Spectral DevOps lint → HTML]
    B22[🎗️ Spectral OWASP lint → HTML]
    B23[🎗️📤 Upload Spectral HTMLs]
    B24[🧾 Job Summary]
  end

  B --> B1 --> B2 --> B3 --> B4 --> B5 --> B6 --> B7 --> B8 --> B9 --> B10 --> B11 --> B12 --> B13 --> B14 --> B15 --> B16 --> B17 --> B18 --> B19 --> B20 --> B21 --> B22 --> B23 --> B24
```

---

## Notes and Tips
- Ensure solutionFileName, mainProject, and imageName are correctly provided; the workflow relies on strict path conventions.
- Sonar conditional step requires both tests to exist and flags inputs.tests and inputs.sonarQube to be set; otherwise, it runs a build-only analysis.
- OpenAPI linting targets JSON files inside output_build/<openAPIFolder>; confirm your build emits API specs there.

