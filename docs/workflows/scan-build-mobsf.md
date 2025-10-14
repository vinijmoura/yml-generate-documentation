# scan-build-mobsf — MobSF Static Analysis via GitHub Actions 🚀🔐

   ## What this workflow does
   - Manually triggered (workflow_dispatch) run on Ubuntu.
   - Starts MobSF via Docker, uploads an iOS build (IPA), triggers a scan, downloads the PDF
   report, and uploads it as a workflow artifact.

   ## Trigger
   - ▶️ Manual: on: workflow_dispatch

   ## Job: build 🧱
   - Runner: ubuntu-latest
   - Env:
     - MOBSF_API_KEY: API key passed to MobSF (example value in YAML; use a secret in real usage).
     - BUILD_FILE: DVIA-v2.ipa (expected at ./builds/DVIA-v2.ipa).
     - MOBSF_PDF_FILE: mobsf-report.pdf (output report filename).

   ## Stages and steps
   1) Setup 🧰
   - actions/checkout@v4 — fetch repository code.

   2) MobSF container 🐳
   - docker pull opensecurity/mobile-security-framework-mobsf:latest — get image.
   - docker images — show images.
   - docker run -it --rm -e MOBSF_API_KEY -p 8000:8000 -d … — launch MobSF API on localhost:8000.
   - docker container ls — confirm it’s running.
   - sleep 40s ⏳ — wait for MobSF to be ready.

   3) Upload build 📤
   - PowerShell step:
     - cd builds; list files for visibility.
     - POST /api/v1/upload with X-Mobsf-Api-Key and file=@./${BUILD_FILE}.
     - On 200 OK, parse JSON response to get .hash and export HASH_BUILD_FILE to GITHUB_ENV.

   4) Scan 🔍
   - PowerShell step:
     - POST /api/v1/scan with hash=<HASH_BUILD_FILE> and API key.
     - Validate 200 OK.

   5) Report (PDF) 📄
   - PowerShell step:
     - POST /api/v1/download_pdf with hash=<HASH_BUILD_FILE>.
     - Save response bytes to $MOBSF_PDF_FILE.

   6) Artifact 📦
   - actions/upload-artifact — upload mobsf-report.pdf.

   ## Output
   - Artifact: mobsf-report.pdf containing the MobSF scan results.

   ## Tips and caveats ⚠️
   - Use secrets: set MOBSF_API_KEY via GitHub Secrets and map to env (e.g., env: MOBSF_API_KEY:
   ${{ secrets.MOBSF_API_KEY }}).
   - File location: ensure ./builds/${BUILD_FILE} exists in the repo or created beforehand.
   - Env propagation: values written to $GITHUB_ENV must be read as environment variables in later
   steps (PowerShell: $env:HASH_BUILD_FILE, Bash: $HASH_BUILD_FILE). Using ${{ env.HASH_BUILD_FILE
   }} won’t pick up runtime changes.
   - PDF saving: Prefer Invoke-WebRequest … -OutFile $env:MOBSF_PDF_FILE to reliably save bytes.

   ## Mermaid diagram
   ```mermaid
   flowchart TD
       A[Manual Dispatch ▶️] --> B[Checkout Code 🧰]
       B --> C[Pull MobSF Docker Image 🐳]
       C --> D[List Docker Images]
       D --> E[Run MobSF Container :8000]
       E --> F[List Running Containers]
       F --> G[Wait 40s ⏳]
       G --> H[Upload IPA to /api/v1/upload 📤]
       H --> I[Export HASH_BUILD_FILE 🔑]
       I --> J[Trigger Scan /api/v1/scan 🔍]
       J --> K[Download PDF /api/v1/download_pdf 📄]
       K --> L[Upload Artifact mobsf-report.pdf 📦]
       L --> M[Done ✅]
   ```

