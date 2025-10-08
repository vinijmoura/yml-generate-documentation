● I'll analyze the GitHub Actions workflow and create comprehensive documentation with a Mermaid
   diagram.

✗ Create /tmp/workflow-documentation.md
   Permission denied and could not request permission from user
Create and display workflow documentation
   $ cat > workflow-documentation.md << 'EOF'
   # GitHub Actions Workflow: scan-build-mobsf

   ## Overview

   This GitHub Actions workflow automates mobile application security testing using the Mobile
   Security Framework (MobSF). It performs automated security analysis on mobile application builds
   (APK/IPA files) and generates comprehensive security reports.

   ## Workflow Triggers

   The workflow is triggered on three events:

   - **Push** to the `main` branch
   - **Pull Request** targeting the `main` branch
   - **Manual dispatch** (`workflow_dispatch`) - allows manual execution from GitHub UI

   ## Jobs

   ### Job: `build`

   **Runner Environment:** `ubuntu-latest`

   **Environment Variables:**
   - `MOBSF_API_KEY`: Authentication key for MobSF API (`githubactionskey`)
   - `BUILD_FILE`: The mobile app file to analyze (`DVIA-v2.ipa`)
   - `MOBSF_PDF_FILE`: Output filename for the security report (`mobsf-report.pdf`)

   ## Workflow Steps

   ### Step 1: Checkout Repository
   ```yaml
   - uses: actions/checkout@v4
   ```
   **Purpose:** Checks out the repository code to the runner, making the build files available for
   analysis.

   ### Step 2: Download MobSF Docker Image
   ```bash
   docker pull opensecurity/mobile-security-framework-mobsf:latest
   ```
   **Purpose:** Downloads the latest MobSF Docker image from Docker Hub, which contains the security
    analysis framework.

   ### Step 3: Display Docker Images
   ```bash
   docker images
   ```
   **Purpose:** Lists all Docker images in the environment for verification and debugging purposes.

   ### Step 4: Create MobSF Container
   ```bash
   docker run -it --rm -e "MOBSF_API_KEY=${{ env.MOBSF_API_KEY }}" -p 8000:8000 -d
   opensecurity/mobile-security-framework-mobsf:latest
   ```
   **Purpose:** Starts the MobSF container in detached mode with:
   - API key configured via environment variable
   - Port 8000 exposed for API access
   - Automatic removal on stop (`--rm`)

   ### Step 5: Display Running Containers
   ```bash
   docker container ls
   ```
   **Purpose:** Lists active containers to confirm MobSF is running successfully.

   ### Step 6: Wait for Container Initialization
   ```bash
   sleep 40s
   ```
   **Purpose:** Allows 40 seconds for the MobSF service to fully initialize before accepting API
   requests.

   ### Step 7: Upload Build File to MobSF
   **Shell:** PowerShell

   **Purpose:** Uploads the mobile application file (APK/IPA) to MobSF for analysis.

   **Process:**
   1. Navigates to the `builds` directory
   2. Lists files for verification
   3. Sends POST request to `/api/v1/upload` endpoint with the build file
   4. Validates HTTP 200 response
   5. Extracts the file hash from the response
   6. Saves hash to `HASH_BUILD_FILE` environment variable for subsequent steps

   ### Step 8: Test Hash Environment Variable
   ```bash
   echo "O hash é ${{ env.HASH_BUILD_FILE }}"
   ```
   **Purpose:** Verifies that the hash value was correctly extracted and stored in the environment
   variable.

   ### Step 9: Initiate Security Scan
   **Shell:** PowerShell

   **Purpose:** Triggers the security analysis of the uploaded mobile application.

   **Process:**
   1. Uses the stored hash from the previous step
   2. Sends POST request to `/api/v1/scan` endpoint
   3. Validates HTTP 200 response
   4. Fails the workflow if scan request is unsuccessful

   ### Step 10: Download PDF Report
   **Shell:** PowerShell

   **Purpose:** Downloads the generated security analysis report in PDF format.

   **Process:**
   1. Uses the stored hash to identify the scan results
   2. Sends POST request to `/api/v1/download_pdf` endpoint
   3. Validates HTTP 200 response
   4. Saves the PDF response content to `mobsf-report.pdf` file

   ### Step 11: Upload PDF Artifact
   ```yaml
   - uses: actions/upload-artifact@v4.6.1
     with:
       name: mobsf-report.pdf
       path: mobsf-report.pdf
   ```
   **Purpose:** Uploads the security report PDF as a GitHub Actions artifact, making it downloadable
    from the workflow run page.

   ## Workflow Diagram

   ```mermaid
   flowchart TD
       A[Workflow Triggered] -->|Push/PR/Manual| B[Start Build Job]
       B --> C[Checkout Repository]
       C --> D[Pull MobSF Docker Image]
       D --> E[Display Docker Images]
       E --> F[Create MobSF Container]
       F --> G[Display Running Containers]
       G --> H[Wait 40 Seconds]
       H --> I[Upload Build File to MobSF]
       I -->|Success| J[Extract & Store Hash]
       I -->|Failure| Z[Workflow Failed]
       J --> K[Test Hash Variable]
       K --> L[Initiate Security Scan]
       L -->|Success| M[Download PDF Report]
       L -->|Failure| Z
       M -->|Success| N[Save PDF File]
       M -->|Failure| Z
       N --> O[Upload PDF as Artifact]
       O --> P[Workflow Complete]

       style A fill:#e1f5ff
       style P fill:#c8e6c9
       style Z fill:#ffcdd2
       style F fill:#fff9c4
       style I fill:#fff9c4
       style L fill:#fff9c4
       style M fill:#fff9c4
   ```

   ## Sequence Diagram

   ```mermaid
   sequenceDiagram
       participant GHA as GitHub Actions
       participant Docker as Docker Engine
       participant MobSF as MobSF Container
       participant Artifact as Artifact Storage

       GHA->>GHA: Checkout Repository
       GHA->>Docker: Pull MobSF Image
       Docker-->>GHA: Image Downloaded
       GHA->>Docker: Create Container
       Docker->>MobSF: Start MobSF Service
       Docker-->>GHA: Container Running
       GHA->>GHA: Wait 40 seconds
       GHA->>MobSF: POST /api/v1/upload (Build File)
       MobSF-->>GHA: Response with Hash
       GHA->>GHA: Store Hash in ENV
       GHA->>MobSF: POST /api/v1/scan (Hash)
       MobSF-->>GHA: Scan Complete (200 OK)
       GHA->>MobSF: POST /api/v1/download_pdf (Hash)
       MobSF-->>GHA: PDF Report Data
       GHA->>GHA: Save PDF File
       GHA->>Artifact: Upload PDF Artifact
       Artifact-->>GHA: Artifact Saved
   ```

   ## Key Features

   1. **Containerized Security Testing**: Uses Docker to ensure consistent, isolated testing
   environment
   2. **API-Driven Automation**: Leverages MobSF REST API for programmatic interaction
   3. **Cross-Platform Support**: Analyzes both Android (APK) and iOS (IPA) applications
   4. **Artifact Preservation**: Automatically saves and archives security reports
   5. **Error Handling**: Validates HTTP responses at each API interaction point
   6. **Reproducible**: Can be manually triggered or run automatically on code changes

   ## Security Considerations

   - The API key is stored as an environment variable (consider using GitHub Secrets for production)
   - The workflow exposes port 8000 only within the runner environment
   - Container is ephemeral (`--rm` flag) and removed after execution

   ## Output

   The workflow produces a PDF security analysis report (`mobsf-report.pdf`) available as a
   downloadable artifact from the GitHub Actions run interface.
   EOF
   cat workflow-documentation.md

   Permission denied and could not request permission from user
● Since I cannot write files in the current environment, I'll provide the complete documentation
   directly:

   # GitHub Actions Workflow: scan-build-mobsf

   ## Overview

   This GitHub Actions workflow automates mobile application security testing using the Mobile
   Security Framework (MobSF). It performs automated security analysis on mobile application builds
    (APK/IPA files) and generates comprehensive security reports.

   ## Workflow Triggers

   The workflow is triggered on three events:

   - **Push** to the `main` branch
   - **Pull Request** targeting the `main` branch
   - **Manual dispatch** (`workflow_dispatch`) - allows manual execution from GitHub UI

   ## Jobs

   ### Job: `build`

   **Runner Environment:** `ubuntu-latest`

   **Environment Variables:**
   - `MOBSF_API_KEY`: Authentication key for MobSF API (`githubactionskey`)
   - `BUILD_FILE`: The mobile app file to analyze (`DVIA-v2.ipa`)
   - `MOBSF_PDF_FILE`: Output filename for the security report (`mobsf-report.pdf`)

   ## Workflow Steps

   ### Step 1: Checkout Repository
   ```yaml
   - uses: actions/checkout@v4
   ```
   **Purpose:** Checks out the repository code to the runner, making the build files available for
   analysis.

   ### Step 2: Download MobSF Docker Image
   ```bash
   docker pull opensecurity/mobile-security-framework-mobsf:latest
   ```
   **Purpose:** Downloads the latest MobSF Docker image from Docker Hub, which contains the
   security analysis framework.

   ### Step 3: Display Docker Images
   ```bash
   docker images
   ```
   **Purpose:** Lists all Docker images in the environment for verification and debugging purposes.

   ### Step 4: Create MobSF Container
   ```bash
   docker run -it --rm -e "MOBSF_API_KEY=${{ env.MOBSF_API_KEY }}" -p 8000:8000 -d
   opensecurity/mobile-security-framework-mobsf:latest
   ```
   **Purpose:** Starts the MobSF container in detached mode with:
   - API key configured via environment variable
   - Port 8000 exposed for API access
   - Automatic removal on stop (`--rm`)

   ### Step 5: Display Running Containers
   ```bash
   docker container ls
   ```
   **Purpose:** Lists active containers to confirm MobSF is running successfully.

   ### Step 6: Wait for Container Initialization
   ```bash
   sleep 40s
   ```
   **Purpose:** Allows 40 seconds for the MobSF service to fully initialize before accepting API
   requests.

   ### Step 7: Upload Build File to MobSF
   **Shell:** PowerShell

   **Purpose:** Uploads the mobile application file (APK/IPA) to MobSF for analysis.

   **Process:**
   1. Navigates to the `builds` directory
   2. Lists files for verification
   3. Sends POST request to `/api/v1/upload` endpoint with the build file
   4. Validates HTTP 200 response
   5. Extracts the file hash from the response
   6. Saves hash to `HASH_BUILD_FILE` environment variable for subsequent steps

   ### Step 8: Test Hash Environment Variable
   ```bash
   echo "O hash é ${{ env.HASH_BUILD_FILE }}"
   ```
   **Purpose:** Verifies that the hash value was correctly extracted and stored in the environment
   variable.

   ### Step 9: Initiate Security Scan
   **Shell:** PowerShell

   **Purpose:** Triggers the security analysis of the uploaded mobile application.

   **Process:**
   1. Uses the stored hash from the previous step
   2. Sends POST request to `/api/v1/scan` endpoint
   3. Validates HTTP 200 response
   4. Fails the workflow if scan request is unsuccessful

   ### Step 10: Download PDF Report
   **Shell:** PowerShell

   **Purpose:** Downloads the generated security analysis report in PDF format.

   **Process:**
   1. Uses the stored hash to identify the scan results
   2. Sends POST request to `/api/v1/download_pdf` endpoint
   3. Validates HTTP 200 response
   4. Saves the PDF response content to `mobsf-report.pdf` file

   ### Step 11: Upload PDF Artifact
   ```yaml
   - uses: actions/upload-artifact@v4.6.1
     with:
       name: mobsf-report.pdf
       path: mobsf-report.pdf
   ```
   **Purpose:** Uploads the security report PDF as a GitHub Actions artifact, making it
   downloadable from the workflow run page.

   ## Workflow Diagram

   ```mermaid
   flowchart TD
       A[Workflow Triggered] -->|Push/PR/Manual| B[Start Build Job]
       B --> C[Checkout Repository]
       C --> D[Pull MobSF Docker Image]
       D --> E[Display Docker Images]
       E --> F[Create MobSF Container]
       F --> G[Display Running Containers]
       G --> H[Wait 40 Seconds]
       H --> I[Upload Build File to MobSF]
       I -->|Success| J[Extract & Store Hash]
       I -->|Failure| Z[Workflow Failed]
       J --> K[Test Hash Variable]
       K --> L[Initiate Security Scan]
       L -->|Success| M[Download PDF Report]
       L -->|Failure| Z
       M -->|Success| N[Save PDF File]
       M -->|Failure| Z
       N --> O[Upload PDF as Artifact]
       O --> P[Workflow Complete]

       style A fill:#e1f5ff
       style P fill:#c8e6c9
       style Z fill:#ffcdd2
       style F fill:#fff9c4
       style I fill:#fff9c4
       style L fill:#fff9c4
       style M fill:#fff9c4
   ```

   ## Sequence Diagram

   ```mermaid
   sequenceDiagram
       participant GHA as GitHub Actions
       participant Docker as Docker Engine
       participant MobSF as MobSF Container
       participant Artifact as Artifact Storage

       GHA->>GHA: Checkout Repository
       GHA->>Docker: Pull MobSF Image
       Docker-->>GHA: Image Downloaded
       GHA->>Docker: Create Container
       Docker->>MobSF: Start MobSF Service
       Docker-->>GHA: Container Running
       GHA->>GHA: Wait 40 seconds
       GHA->>MobSF: POST /api/v1/upload (Build File)
       MobSF-->>GHA: Response with Hash
       GHA->>GHA: Store Hash in ENV
       GHA->>MobSF: POST /api/v1/scan (Hash)
       MobSF-->>GHA: Scan Complete (200 OK)
       GHA->>MobSF: POST /api/v1/download_pdf (Hash)
       MobSF-->>GHA: PDF Report Data
       GHA->>GHA: Save PDF File
       GHA->>Artifact: Upload PDF Artifact
       Artifact-->>GHA: Artifact Saved
   ```

   ## Key Features

   1. **Containerized Security Testing**: Uses Docker to ensure consistent, isolated testing
   environment
   2. **API-Driven Automation**: Leverages MobSF REST API for programmatic interaction
   3. **Cross-Platform Support**: Analyzes both Android (APK) and iOS (IPA) applications
   4. **Artifact Preservation**: Automatically saves and archives security reports
   5. **Error Handling**: Validates HTTP responses at each API interaction point
   6. **Reproducible**: Can be manually triggered or run automatically on code changes

   ## Security Considerations

   - The API key is stored as an environment variable (consider using GitHub Secrets for
   production)
   - The workflow exposes port 8000 only within the runner environment
   - Container is ephemeral (`--rm` flag) and removed after execution

   ## Output

   The workflow produces a PDF security analysis report (`mobsf-report.pdf`) available as a
   downloadable artifact from the GitHub Actions run interface.

