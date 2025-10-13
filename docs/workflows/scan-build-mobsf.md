● I'll analyze this GitHub Actions workflow and create comprehensive documentation with a Mermaid
   diagram.

✗ Create /tmp/workflow-documentation.md
   Permission denied and could not request permission from user
Create and display workflow documentation
   $ cat > /tmp/mobsf-workflow-docs.md << 'EOF'
   # Mobile Security Framework (MobSF) Scan Workflow

   ## Overview

   This GitHub Actions workflow automates mobile application security scanning using the Mobile
   Security Framework (MobSF). It performs static analysis on mobile application builds (APK/IPA
   files) and generates a comprehensive security report in PDF format.

   ## Workflow Triggers

   The workflow is triggered by three events:

   - **Push events** to the `main` branch
   - **Pull request events** targeting the `main` branch
   - **Manual workflow dispatch** through the GitHub Actions UI

   ## Workflow Configuration

   ### Environment Variables

   The workflow uses the following environment variables:

   - `MOBSF_API_KEY`: API key for authenticating with the MobSF service (set to 'githubactionskey')
   - `BUILD_FILE`: The mobile application file to scan (currently set to 'DVIA-v2.ipa', with
   'VulnApp.apk' commented out)
   - `MOBSF_PDF_FILE`: Output filename for the security report (set to 'mobsf-report.pdf')

   ### Runner Environment

   - **Operating System**: Ubuntu Latest
   - **Container Technology**: Docker

   ## Job: build

   This workflow contains a single job named `build` that executes on an Ubuntu runner.

   ### Step-by-Step Breakdown

   #### Step 1: Checkout Code
   **Action**: `actions/checkout@v4`

   Checks out the repository code to the runner's workspace, making the build files available for
   scanning.

   #### Step 2: Download MobSF Docker Image
   **Command**: `docker pull opensecurity/mobile-security-framework-mobsf:latest`

   Downloads the latest version of the Mobile Security Framework Docker image from Docker Hub. This
   image contains all necessary tools and dependencies for mobile security analysis.

   #### Step 3: Display Container Images
   **Command**: `docker images`

   Lists all Docker images present in the environment, useful for verification and debugging
   purposes.

   #### Step 4: Create MobSF Container
   **Command**: `docker run -it --rm -e "MOBSF_API_KEY=${{ env.MOBSF_API_KEY }}" -p 8000:8000 -d
   opensecurity/mobile-security-framework-mobsf:latest`

   Launches the MobSF container with the following configuration:
   - **Interactive mode** (`-it`)
   - **Auto-remove** (`--rm`) after container stops
   - **Environment variable** for API authentication
   - **Port mapping** 8000:8000 to expose the MobSF web service
   - **Detached mode** (`-d`) to run in background

   #### Step 5: Display Running Containers
   **Command**: `docker container ls`

   Lists all running containers to verify the MobSF container started successfully.

   #### Step 6: Wait for Container Initialization
   **Command**: `sleep 40s`

   Pauses execution for 40 seconds to allow the MobSF service to fully start and become ready to
   accept API requests.

   #### Step 7: Upload Build File to MobSF
   **Shell**: PowerShell

   This step performs several operations:
   1. Navigates to the `builds` directory
   2. Lists files in the directory for verification
   3. Uploads the specified build file (APK/IPA) to MobSF via REST API
   4. Validates the HTTP response status code (expecting 200)
   5. Extracts the file hash from the JSON response
   6. Stores the hash in the `HASH_BUILD_FILE` environment variable for subsequent steps

   **API Endpoint**: `POST http://localhost:8000/api/v1/upload`

   #### Step 8: Verify Hash Environment Variable
   **Command**: `echo "O hash é ${{ env.HASH_BUILD_FILE }}"`

   Displays the stored hash value for debugging and verification purposes.

   #### Step 9: Initiate Security Scan
   **Shell**: PowerShell

   Sends a POST request to the MobSF scan endpoint with the file hash, triggering the static
   analysis of the mobile application. The step validates the response and exits with an error if
   the scan fails.

   **API Endpoint**: `POST http://localhost:8000/api/v1/scan`

   #### Step 10: Download PDF Report
   **Shell**: PowerShell

   Requests the security analysis report in PDF format from MobSF using the file hash. The response
   content (binary PDF data) is written to disk as the specified filename.

   **API Endpoint**: `POST http://localhost:8000/api/v1/download_pdf`

   #### Step 11: Upload Artifact
   **Action**: `actions/upload-artifact@v4.6.1`

   Uploads the generated PDF report as a GitHub Actions artifact, making it available for download
   from the workflow run page. The artifact is named according to the `MOBSF_PDF_FILE` environment
   variable.

   ## Workflow Diagram

   ```mermaid
   flowchart TD
       A[Workflow Triggered] --> B{Trigger Type}
       B -->|Push to main| C[Start Job: build]
       B -->|Pull Request| C
       B -->|Manual Dispatch| C

       C --> D[Step 1: Checkout Code]
       D --> E[Step 2: Pull MobSF Docker Image]
       E --> F[Step 3: Display Docker Images]
       F --> G[Step 4: Create & Run MobSF Container]
       G --> H[Step 5: List Running Containers]
       H --> I[Step 6: Wait 40s for Initialization]
       I --> J[Step 7: Upload Build File to MobSF]

       J --> K{Upload Successful?}
       K -->|No - HTTP ≠ 200| L[Fail Workflow]
       K -->|Yes - HTTP = 200| M[Extract & Store File Hash]

       M --> N[Step 8: Verify Hash Variable]
       N --> O[Step 9: Trigger Security Scan]

       O --> P{Scan Successful?}
       P -->|No - HTTP ≠ 200| L
       P -->|Yes - HTTP = 200| Q[Step 10: Download PDF Report]

       Q --> R{Download Successful?}
       R -->|No - HTTP ≠ 200| L
       R -->|Yes - HTTP = 200| S[Write PDF to Disk]

       S --> T[Step 11: Upload PDF as Artifact]
       T --> U[Workflow Complete]

       style A fill:#e1f5ff
       style C fill:#ffe1e1
       style L fill:#ff9999
       style U fill:#99ff99
       style J fill:#fff4e1
       style O fill:#fff4e1
       style Q fill:#fff4e1
   ```

   ## API Integration

   The workflow interacts with three MobSF REST API endpoints:

   ### 1. Upload API
   - **Endpoint**: `/api/v1/upload`
   - **Method**: POST
   - **Purpose**: Upload mobile application file for analysis
   - **Returns**: File hash for subsequent operations

   ### 2. Scan API
   - **Endpoint**: `/api/v1/scan`
   - **Method**: POST
   - **Purpose**: Initiate static security analysis
   - **Requires**: File hash from upload step

   ### 3. Download PDF API
   - **Endpoint**: `/api/v1/download_pdf`
   - **Method**: POST
   - **Purpose**: Generate and retrieve security report
   - **Requires**: File hash from upload step
   - **Returns**: Binary PDF file

   ## Security Considerations

   1. **API Key Management**: The API key is hardcoded in the workflow. For production use, consider
    using GitHub Secrets.
   2. **Local Execution**: The MobSF container runs locally on the runner, avoiding external data
   transmission.
   3. **Container Cleanup**: The `--rm` flag ensures containers are automatically removed after
   execution.

   ## Output Artifacts

   The workflow produces one artifact:
   - **mobsf-report.pdf**: Comprehensive security analysis report containing vulnerability findings,
    code quality metrics, and security recommendations.

   ## Dependencies

   - **Docker**: Required for container operations
   - **PowerShell**: Used for API interactions and file operations
   - **GitHub Actions Runner**: Ubuntu-based runner with Docker support

   ## Workflow Execution Time

   Estimated duration: 5-10 minutes depending on:
   - Docker image download time
   - Application size and complexity
   - MobSF analysis duration

   ## Error Handling

   The workflow implements basic error handling:
   - HTTP status code validation after each API call
   - Workflow fails immediately if any API call returns non-200 status
   - Exit code 1 triggers workflow failure and stops subsequent steps
   EOF
   cat /tmp/mobsf-workflow-docs.md

   Permission denied and could not request permission from user
● # Mobile Security Framework (MobSF) Scan Workflow

   ## Overview

   This GitHub Actions workflow automates mobile application security scanning using the Mobile
   Security Framework (MobSF). It performs static analysis on mobile application builds (APK/IPA
   files) and generates a comprehensive security report in PDF format.

   ## Workflow Triggers

   The workflow is triggered by three events:

   - **Push events** to the `main` branch
   - **Pull request events** targeting the `main` branch
   - **Manual workflow dispatch** through the GitHub Actions UI

   ## Workflow Configuration

   ### Environment Variables

   The workflow uses the following environment variables:

   - `MOBSF_API_KEY`: API key for authenticating with the MobSF service (set to 'githubactionskey')
   - `BUILD_FILE`: The mobile application file to scan (currently set to 'DVIA-v2.ipa', with
   'VulnApp.apk' commented out)
   - `MOBSF_PDF_FILE`: Output filename for the security report (set to 'mobsf-report.pdf')

   ### Runner Environment

   - **Operating System**: Ubuntu Latest
   - **Container Technology**: Docker

   ## Job: build

   This workflow contains a single job named `build` that executes on an Ubuntu runner.

   ### Step-by-Step Breakdown

   #### Step 1: Checkout Code
   **Action**: `actions/checkout@v4`

   Checks out the repository code to the runner's workspace, making the build files available for
   scanning.

   #### Step 2: Download MobSF Docker Image
   **Command**: `docker pull opensecurity/mobile-security-framework-mobsf:latest`

   Downloads the latest version of the Mobile Security Framework Docker image from Docker Hub. This
    image contains all necessary tools and dependencies for mobile security analysis.

   #### Step 3: Display Container Images
   **Command**: `docker images`

   Lists all Docker images present in the environment, useful for verification and debugging
   purposes.

   #### Step 4: Create MobSF Container
   **Command**: `docker run -it --rm -e "MOBSF_API_KEY=${{ env.MOBSF_API_KEY }}" -p 8000:8000 -d
   opensecurity/mobile-security-framework-mobsf:latest`

   Launches the MobSF container with the following configuration:
   - **Interactive mode** (`-it`)
   - **Auto-remove** (`--rm`) after container stops
   - **Environment variable** for API authentication
   - **Port mapping** 8000:8000 to expose the MobSF web service
   - **Detached mode** (`-d`) to run in background

   #### Step 5: Display Running Containers
   **Command**: `docker container ls`

   Lists all running containers to verify the MobSF container started successfully.

   #### Step 6: Wait for Container Initialization
   **Command**: `sleep 40s`

   Pauses execution for 40 seconds to allow the MobSF service to fully start and become ready to
   accept API requests.

   #### Step 7: Upload Build File to MobSF
   **Shell**: PowerShell

   This step performs several operations:
   1. Navigates to the `builds` directory
   2. Lists files in the directory for verification
   3. Uploads the specified build file (APK/IPA) to MobSF via REST API
   4. Validates the HTTP response status code (expecting 200)
   5. Extracts the file hash from the JSON response
   6. Stores the hash in the `HASH_BUILD_FILE` environment variable for subsequent steps

   **API Endpoint**: `POST http://localhost:8000/api/v1/upload`

   #### Step 8: Verify Hash Environment Variable
   **Command**: `echo "O hash é ${{ env.HASH_BUILD_FILE }}"`

   Displays the stored hash value for debugging and verification purposes.

   #### Step 9: Initiate Security Scan
   **Shell**: PowerShell

   Sends a POST request to the MobSF scan endpoint with the file hash, triggering the static
   analysis of the mobile application. The step validates the response and exits with an error if
   the scan fails.

   **API Endpoint**: `POST http://localhost:8000/api/v1/scan`

   #### Step 10: Download PDF Report
   **Shell**: PowerShell

   Requests the security analysis report in PDF format from MobSF using the file hash. The response
    content (binary PDF data) is written to disk as the specified filename.

   **API Endpoint**: `POST http://localhost:8000/api/v1/download_pdf`

   #### Step 11: Upload Artifact
   **Action**: `actions/upload-artifact@v4.6.1`

   Uploads the generated PDF report as a GitHub Actions artifact, making it available for download
   from the workflow run page. The artifact is named according to the `MOBSF_PDF_FILE` environment
   variable.

   ## Workflow Diagram

   ```mermaid
   flowchart TD
       A[Workflow Triggered] --> B{Trigger Type}
       B -->|Push to main| C[Start Job: build]
       B -->|Pull Request| C
       B -->|Manual Dispatch| C

       C --> D[Step 1: Checkout Code]
       D --> E[Step 2: Pull MobSF Docker Image]
       E --> F[Step 3: Display Docker Images]
       F --> G[Step 4: Create & Run MobSF Container]
       G --> H[Step 5: List Running Containers]
       H --> I[Step 6: Wait 40s for Initialization]
       I --> J[Step 7: Upload Build File to MobSF]

       J --> K{Upload Successful?}
       K -->|No - HTTP ≠ 200| L[Fail Workflow]
       K -->|Yes - HTTP = 200| M[Extract & Store File Hash]

       M --> N[Step 8: Verify Hash Variable]
       N --> O[Step 9: Trigger Security Scan]

       O --> P{Scan Successful?}
       P -->|No - HTTP ≠ 200| L
       P -->|Yes - HTTP = 200| Q[Step 10: Download PDF Report]

       Q --> R{Download Successful?}
       R -->|No - HTTP ≠ 200| L
       R -->|Yes - HTTP = 200| S[Write PDF to Disk]

       S --> T[Step 11: Upload PDF as Artifact]
       T --> U[Workflow Complete]

       style A fill:#e1f5ff
       style C fill:#ffe1e1
       style L fill:#ff9999
       style U fill:#99ff99
       style J fill:#fff4e1
       style O fill:#fff4e1
       style Q fill:#fff4e1
   ```

   ## API Integration

   The workflow interacts with three MobSF REST API endpoints:

   ### 1. Upload API
   - **Endpoint**: `/api/v1/upload`
   - **Method**: POST
   - **Purpose**: Upload mobile application file for analysis
   - **Returns**: File hash for subsequent operations

   ### 2. Scan API
   - **Endpoint**: `/api/v1/scan`
   - **Method**: POST
   - **Purpose**: Initiate static security analysis
   - **Requires**: File hash from upload step

   ### 3. Download PDF API
   - **Endpoint**: `/api/v1/download_pdf`
   - **Method**: POST
   - **Purpose**: Generate and retrieve security report
   - **Requires**: File hash from upload step
   - **Returns**: Binary PDF file

   ## Security Considerations

   1. **API Key Management**: The API key is hardcoded in the workflow. For production use,
   consider using GitHub Secrets.
   2. **Local Execution**: The MobSF container runs locally on the runner, avoiding external data
   transmission.
   3. **Container Cleanup**: The `--rm` flag ensures containers are automatically removed after
   execution.

   ## Output Artifacts

   The workflow produces one artifact:
   - **mobsf-report.pdf**: Comprehensive security analysis report containing vulnerability
   findings, code quality metrics, and security recommendations.

   ## Dependencies

   - **Docker**: Required for container operations
   - **PowerShell**: Used for API interactions and file operations
   - **GitHub Actions Runner**: Ubuntu-based runner with Docker support

   ## Workflow Execution Time

   Estimated duration: 5-10 minutes depending on:
   - Docker image download time
   - Application size and complexity
   - MobSF analysis duration

   ## Error Handling

   The workflow implements basic error handling:
   - HTTP status code validation after each API call
   - Workflow fails immediately if any API call returns non-200 status
   - Exit code 1 triggers workflow failure and stops subsequent steps

