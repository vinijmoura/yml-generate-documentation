● # 🚀 Workshop CI Workflow

   Trigger: Manual (workflow_dispatch)

   Env vars:
   - URL_GITLEAKS_RELEASE: download URL for Gitleaks
   - DIR_DOCKERFILE: ./src/ConsoleAppJobHttpRequest
   - DIR_DEVOPS_ARTIFACTS: ./devops
   - DIR_TESTS: ./tests

   Execution order: secrets_scanning ➜ dockerfile_scanning ➜ kubernetes_yaml_scanning ➜ tests
   (continues even if previous jobs fail via if: always).

   ## 🔐 Job: secrets_scanning (Ubuntu)
   Purpose: Scan repository for leaked secrets and publish SARIF.
   - Checkout code
   - Install Gitleaks from URL_GITLEAKS_RELEASE (untar, chmod, move to /usr/local/bin)
   - Sanity check: gitleaks --version and --help
   - Prepare results directory
   - Run gitleaks dir . with SARIF report to ./results/results-secrets.sarif (downloads basic.tmpl)
   - List results
   - Upload SARIF (category: sast-tools)

   ## 🐳 Job: dockerfile_scanning (Ubuntu, needs: secrets_scanning, always)
   Purpose: Lint Dockerfiles with Checkov and publish SARIF.
   - Checkout code
   - pip install checkov
   - Sanity check: checkov and --help
   - Prepare results directory
   - Run Checkov on $DIR_DOCKERFILE (framework dockerfile, skip CKV_DOCKER_2, CKV_DOCKER_3) output
   SARIF to ./results/results-dockerfile.sarif
   - List results
   - Upload SARIF (category: sast-tools)

   ## ☸️ Job: kubernetes_yaml_scanning (Ubuntu, needs: dockerfile_scanning, always)
   Purpose: Scan Kubernetes YAML with KICS and publish PDF report.
   - Checkout code
   - docker pull checkmarx/kics:latest and show local images
   - Show KICS CLI help (docker run … scan --help)
   - Set permissive rights on $DIR_DEVOPS_ARTIFACTS for container volume
   - Run KICS scan mounting $DIR_DEVOPS_ARTIFACTS to /path, exclude severities low/info/trace,
   exclude specific queries, output PDF results-kubernetes.pdf into that directory
   - List generated files
   - Upload artifact: results-kubernetes.pdf

   ## 🧪 Job: tests (Ubuntu, needs: kubernetes_yaml_scanning)
   Purpose: Build image, create Kind cluster, deploy Job, and verify execution.
   - Checkout code
   - Build Docker image: workshop/job-httprequest:$GITHUB_RUN_NUMBER from $DIR_DOCKERFILE
   - Show images
   - Verify kind installed; create cluster from $DIR_TESTS/kind-cluster-config-automation-env.yaml
   - Basic cluster check: kubectl get services; list running containers
   - Load built image into kind cluster
   - Create namespace integrationtests
   - Show $DIR_DEVOPS_ARTIFACTS/job-teste.yaml (pre-change)
   - Token replacement: cschleiden/replace-tokens@v1 on **/job-teste.yaml
   - Show job-teste.yaml (post-change)
   - Deploy: kubectl apply -f job-teste.yaml -n integrationtests
   - Verify: kubectl get all/jobs -n integrationtests, wait 45s, re-check
   - Fetch logs: kubectl logs job.batch/exemplo-job -n integrationtests

   ## 🗺️ Mermaid Diagra
   ```mermaid
   flowchart LR
  trigger([Manual Trigger - workflow_dispatch]) --> J1[secrets_scanning]
  J1 --> J2[dockerfile_scanning]
  J2 --> J3[kubernetes_yaml_scanning]
  J3 --> J4[tests]

  subgraph S1[secrets_scanning]
    A1[Checkout] --> A2[Install Gitleaks]
    A2 --> A3[Test Gitleaks]
    A3 --> A4["Run scan -> SARIF"]
    A4 --> A5[Upload SARIF]
  end

  subgraph S2[dockerfile_scanning]
    B1[Checkout] --> B2[Install Checkov]
    B2 --> B3[Test Checkov]
    B3 --> B4["Run scan (skip CKV_2 and CKV_3) -> SARIF"]
    B4 --> B5[Upload SARIF]
  end

  subgraph S3[kubernetes_yaml_scanning]
    C1[Checkout] --> C2[Pull KICS]
    C2 --> C3[Prep perms + Help]
    C3 --> C4["Run scan (excludes) -> PDF"]
    C4 --> C5[Upload PDF]
  end

  subgraph S4[tests]
    D1[Build image] --> D2[Create Kind cluster]
    D2 --> D3[Load image + ns]
    D3 --> D4[Token replace + Apply YAML]
    D4 --> D5[Verify + Logs]
  end

  click J1 href "https://github.com/gitleaks/gitleaks" "Gitleaks"
  click J2 href "https://www.checkov.io/" "Checkov"
  click J3 href "https://github.com/Checkmarx/kics" "KICS"
  click J4 href "https://kind.sigs.k8s.io/" "Kind"

   ```

   Tips:
   - Jobs 2–3 run even if previous fail due to if: always.
   - Artifacts: two SARIF reports and one PDF for KICS findings.

