● # Windows Services Release workflow 🚀

   Trigger
   - Manual only: workflow_dispatch

   Runners and scope
   - Runs on self-hosted Windows runners for both jobs
   - Uses global env for paths/service names to keep steps consistent

   Environment variables
   - OUTPUT_DIRECTORY: c:\myapp (MSBuild output)
   - SERVICE_NAME: SampleWindowsServicesGHA (target Windows service)
   - SERVICE_DIRECTORYNAME: C:\Services (service install folder)
   - DOWNLOAD_DIRECTORY: C:\ServicesDownload (artifact download root)

   Stage 1 — Build 🛠
   Job: build
   - Checkout 📥: actions/checkout@v2
   - NuGet setup 📦: NuGet/setup-nuget@v1.0.5
   - Restore 🔄: nuget restore $env:Solution_Name
   - MSBuild tool 🧰: microsoft/setup-msbuild@v1.0.2
   - Build 🏗️: msbuild with platform Any CPU, Configuration Release, OutputPath = OUTPUT_DIRECTOR
   - Publish artifact ⤴️: actions/upload-artifact (name: windowsservices, path: OUTPUT_DIRECTORY)

   Stage 2 — Deploy 🚢
   Job: deploy (needs: build)
   - Download artifact ⤵️: actions/download-artifact (to DOWNLOAD_DIRECTORY\SERVICE_NAME)
   - Inspect service 🔎: Get-Service -Name SERVICE_NAME
   - Stop service ⛔: run elevated Stop-Service and wait until Status = Stopped
   - Deploy files 📁: Copy-Item from download to SERVICE_DIRECTORYNAME\SERVICE_NAME (overwrite)
   - Start service ▶️: run elevated Start-Service and wait until Status = Running

   Mermaid diagram
   ```mermaid
   flowchart TD
     A[🚀 Manual trigger<br/>workflow_dispatch] --> B[🛠️ Stage 1: Build
     subgraph B[Build (self-hosted)]
       B1[📥 Checkout]
       B2[📦 Setup NuGet]
       B3[🔄 Restore packages]
       B4[🧰 Setup MSBuild]
       B5[🏗️ Build -> OUTPUT_DIRECTORY
       B6[⤴️ Upload artifact 'windowsservices']
       B1 --> B2 --> B3 --> B4 --> B5 --> B6
     end

     B --> C[🚢 Stage 2: Deploy]
     subgraph C[Deploy (self-hosted)]
       C1[⤵️ Download 'windowsservices'<br/>to DOWNLOAD_DIRECTORY\\SERVICE_NAME]
       C2[🔎 Get-Service SERVICE_NAME]
       C3[⛔ Stop-Service (elevated)<br/>wait until Stopped]
       C4[📁 Copy files -> SERVICE_DIRECTORYNAME\\SERVICE_NAME]
       C5[▶️ Start-Service (elevated)<br/>wait until Running]
       C1 --> C2 --> C3 --> C4 --> C5
     end
   ```

