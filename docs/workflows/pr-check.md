# Pull Request Check ⚡ — Workflow Documentation

This reusable workflow validates .NET projects on PRs or reusable calls: builds the app, runs tests and code coverage, performs security/quality scans (Gitleaks, SonarCloud, Snyk), optionally lints OpenAPI specs with Spectral, and reports whether an image with the computed version tag already exists.

## Trigger and Identity
- Trigger: workflow_call (consumed by other workflows)
- Name: Pull Request Check ⚡
- Run name: "${{ github.actor }} - ${{ github.run_id }}"

## Inputs (selected)
- environment (required, default: DEV): Target environment
- testsFolder (default: tests): Root folder for test projects
- dotnetVersion (required, default: 9.x)
- projectBaseDir (required, default: ./src)
- solutionFileName (required): Solution file, e.g., MyProject.slnx
- mainProject (required): Main project folder name
- reportTitle (required, default: Canal DEPLOY)
- sonarExclusions (required, default: empty): Comma-separated exclusions
- imageName (required): Image name
- openAPIFolder (required): Folder under output_build/ that holds OpenAPI JSONs

Secrets/vars used:
- Secrets: GITLEAKS_LICENSE, SONAR_TOKEN, SONAR_PROJECT_KEY, SNYK_TOKEN, GITHUB_TOKEN
- Vars: SONAR_ORGANIZATION

Runtime env:
- Java: zulu 23
- Node: 22

---

## Jobs Overview

### 1) build (ubuntu-latest)
Outputs:
- tag-version: Computed from csproj Version/(Prefix)/(Suffix)
- now: Current timestamp

Key steps:
- Environment and checkout
  - Get Environment: logs environment and sets NOW
  - Checkout (sparse): src and tests
- Versioning
  - Get-Version from csproj: reads <Version>, <VersionPrefix>, <VersionSuffix> from ${{ inputs.projectBaseDir }}/${{ inputs.mainProject }}/${{ inputs.mainProject }}.csproj and composes TAG
- .NET setup
  - setup-dotnet with inputs.dotnetVersion
  - dotnet --info
- Test detection and tools
  - Find *Tests.csproj or *Test.csproj under testsFolder
  - If tests exist: install dotnet-reportgenerator-globaltool and dotnet-coverage; also set up Docker (get.docker.com)
- Security and quality
  - Gitleaks (skips on pull_request_target)
  - Setup Java, cache Sonar, install dotnet-sonarscanner
- Testing and coverage (when tests exist)
  - dotnet test for each test project (XPlat coverage)
  - reportgenerator → Cobertura, HTML, Markdown, SonarQube reports
  - Upload Coverage-Report artifact (coveragereport directory)
- SonarCloud analysis
  - With tests and coverage: dotnet-sonarscanner begin → restore/build → dotnet-coverage collect → end
  - Without tests: dotnet-sonarscanner begin → restore/build → end
- Build + Security scanning
  - dotnet restore/build to ./output_build
  - Snyk setup
  - Docker buildx build image (tag: felipementel/${{ inputs.imageName }})
  - Snyk container test (writes vuln.json)
  - Snyk .NET scan → SARIF upload to GitHub code scanning
  - Upload Snyk Container test artifact (vuln.json)
- OpenAPI linting (when openAPIFolder provided)
  - Setup Node 22, install Spectral
  - Run DevOps Linter ruleset → spectral-devops-linter.html
  - Run OWASP ruleset → spectral-owasp.html
  - Upload Spectral-Results artifacts
- Summary
  - Writes version and timestamp plus a custom message to the job summary

### 2) check-image (ubuntu-latest)
- Logs in to GHCR with GITHUB_TOKEN
- Checks if image tag exists:
  - ghcr.io/${{ github.actor }}/deploy.${{ inputs.imageName }}:${{ needs.build.outputs.tag-version }}
- Output: image-exists = true/false

### 3) notify-image-not-exists (ubuntu-latest)
- Needs: [build, check-image]
- Runs if image-exists == 'false'
- Writes a summary telling the image can be deployed and should be pushed

### 4) notify-image-exists (ubuntu-latest)
- Needs: [build, check-image]
- Runs if image-exists == 'true'
- Writes a summary telling the image already exists and push/publish is skipped

---

## Mermaid Diagram

```mermaid
flowchart TD
  A[Workflow Call 🔁] --> B[build 🧱]
  subgraph Build Job
    B1[Get env + checkout] --> B2[Read csproj version → TAG]
    B2 --> B3[Setup .NET] --> B4[Check tests]
    B4 -- HAS_TESTS=true --> B5[Install test tools + Docker]
    B4 -- HAS_TESTS=false --> B6[Skip test tooling]
    B5 --> B7[dotnet test + coverage + reportgenerator + upload artifact]
    B6 --> B8[Sonar (no tests)]
    B7 --> B9[Sonar (with tests + coverage)]
    B8 --> B10[Build to output_build]
    B9 --> B10[Build to output_build]
    B10 --> B11[Docker build]
    B11 --> B12[Snyk container test + upload]
    B10 --> B13[Snyk .NET → SARIF → upload]
    B10 --> B14{openAPIFolder set?}
    B14 -- yes --> B15[Node + Spectral DevOps+OWASP + upload]
    B14 -- no --> B16[Skip Spectral]
    B12 --> B17[Job summary]
    B13 --> B17
    B15 --> B17
    B16 --> B17
  end
  B --> C[check-image 🔎]
  C -->|IMAGE_EXISTS=false| D[notify-image-not-exists 🔔]
  C -->|IMAGE_EXISTS=true| E[notify-image-exists 🔕]
```

---

## Notes and Gotchas
- Sonar step with tests is gated by inputs.tests and inputs.sonarQube, which are not defined; that step will never run unless those inputs are added.  
- Coverage artifact upload appears twice with swapped name/path (Coverage-Report vs coveragereport); verify the intended artifact name and directory.  
- The built image tag (felipementel/${{ inputs.imageName }}) differs from the registry/name checked later (ghcr.io/${{ github.actor }}/deploy.${{ inputs.imageName }}:TAG); align registry/namespace/name if you expect the image to be found.  
- The “Sonar (no tests)” condition uses env.HAS_TESTS; since HAS_TESTS is a step output, prefer steps.check-tests.outputs.HAS_TESTS != 'true' to avoid unintended always-on execution.

