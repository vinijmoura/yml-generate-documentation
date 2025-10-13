# GitHub Actions Workflow: Mobile Security Framework (MobSF) Security Scan

   ## Overview

   This workflow automates mobile application security testing using MobSF (Mobile Security
   Framework). It deploys MobSF in a Docker container, uploads a mobile app build file (APK or
   IPA), performs security analysis, and generates a PDF report as an artifact.

   ## Workflow Triggers

   The workflow executes on three events:
   - **Push** to the `main` branch
   - **Pull Request** targeting the `main` branch
   - **Manual trigger** via `workflow_dispatch`

   ## Environment Variables

   The workflow configures three environment variables at the job level:
   - `MOBSF_API_KEY`: Authentication key for MobSF API (`'githubactionskey'`)
   - `BUILD_FILE`: Target application file (`'DVIA-v2.ipa'` - an iOS app; APK option commented out)
   - `MOBSF_PDF_FILE`: Output report filename (`'mobsf-report.pdf'`)

   ## Job: build

   **Runtime Environment:** `ubuntu-latest`

   ### Steps Breakdown

   **1. Checkout Code**
   - Uses `actions/checkout@v4`
   - Retrieves the repository source code to the runner

   **2. Pull MobSF Docker Image**
   - Downloads the latest MobSF Docker image from
   `opensecurity/mobile-security-framework-mobsf:latest`

   **3. Display Container Images**
   - Lists all Docker images available in the environment for verification

   **4. Create MobSF Container**
   - Launches MobSF in detached mode (`-d`)
   - Configures API key via environment variable
   - Maps port 8000 (container) to 8000 (host)
   - Uses `--rm` flag for automatic cleanup after container stops

   **5. Display Running Containers**
   - Lists active Docker containers to verify MobSF is running

   **6. Wait for Container Initialization**
   - Sleeps for 40 seconds to ensure MobSF service is fully operational

   **7. Upload Build File to MobSF**
   - Uses PowerShell to upload the mobile app file
   - Navigates to `builds` directory
   - Sends multipart/form-data POST request to `/api/v1/upload` endpoint
   - Validates HTTP 200 response
   - Extracts file hash from response and stores in `HASH_BUILD_FILE` environment variable

   **8. Test Hash Environment Variable**
   - Echoes the stored hash value for verification

   **9. Initiate Security Scan**
   - Sends POST request to `/api/v1/scan` endpoint with the file hash
   - MobSF performs comprehensive security analysis
   - Validates successful scan completion (HTTP 200)

   **10. Generate and Download PDF Report**
   - Requests PDF report via `/api/v1/download_pdf` endpoint
   - Writes binary response content to `mobsf-report.pdf` file
   - Validates successful download (HTTP 200)

   **11. Upload PDF Artifact**
   - Uses `actions/upload-artifact@v4.6.1`
   - Stores the PDF report as a workflow artifact for download
   - Artifact name and path both reference `MOBSF_PDF_FILE` variable

   ## Workflow Diagram

   ```mermaid
   graph TD
       A[Workflow Triggered] -->|push/PR/manual| B[Job: build on ubuntu-latest]
       B --> C[Checkout Code]
       C --> D[Pull MobSF Docker Image]
       D --> E[Display Docker Images]
       E --> F[Start MobSF Container<br/>Port 8000, API Key Set]
       F --> G[Display Running Containers]
       G --> H[Wait 40 seconds]
       H --> I[Upload Build File<br/>POST /api/v1/upload]
       I --> J{Upload<br/>Success?}
       J -->|No HTTP 200| K[Exit with Error]
       J -->|Yes| L[Extract & Store Hash]
       L --> M[Verify Hash Variable]
       M --> N[Trigger Security Scan<br/>POST /api/v1/scan]
       N --> O{Scan<br/>Success?}
       O -->|No HTTP 200| P[Exit with Error]
       O -->|Yes| Q[Download PDF Report<br/>POST /api/v1/download_pdf]
       Q --> R{Download<br/>Success?}
       R -->|No HTTP 200| S[Exit with Error]
       R -->|Yes| T[Save PDF to Disk]
       T --> U[Upload PDF as Artifact]
       U --> V[Workflow Complete]

       style A fill:#e1f5ff
       style B fill:#fff4e1
       style F fill:#ffe1e1
       style I fill:#e1ffe1
       style N fill:#e1ffe1
       style Q fill:#e1ffe1
       style U fill:#f0e1ff
       style V fill:#e1ffe1
       style K fill:#ffcccc
       style P fill:#ffcccc
       style S fill:#ffcccc
   ```

   ## Key Features

   **Security Analysis**: Automated mobile app vulnerability scanning for both Android (APK) and
   iOS (IPA) applications

   **Containerized Execution**: Isolated MobSF environment ensures consistent analysis without
   dependency conflicts

   **API-Driven**: Fully automated via MobSF REST API endpoints for upload, scan, and report
   generation

   **Artifact Preservation**: PDF security reports stored as workflow artifacts for audit trails
   and review

   **Error Handling**: HTTP status code validation at each API interaction with explicit failure
   paths

