● ## Workflow: SELENIUM and JMETER
   - Trigger: Manual (workflow_dispatch)
   - Global env:
     - Solution_Name: PartsUnlimited.sln
     - buildPlatform: Any CPU
     - buildConfiguration: Release
     - PACKAGE_LOCATION: C:\PartsUnlimited
     - ARTIFACTS_DOWNLOAD: C:\Artifacts\PartsUnlimited
     - WEB_SITE_NAME: PartsUnlimited

   ### Stage 1 — build (windows-2019)
   - Steps:
     1) Checkout repo (actions/checkout@v2)
     2) Setup NuGet (NuGet/setup-nuget@v1.0.5)
     3) Restore packages: nuget restore $env:Solution_Name
     4) Setup MSBuild (microsoft/setup-msbuild@v1.0.2)
     5) Build & package WebDeploy zip:
        msbuild PartsUnlimited.sln /p:Platform="Any CPU" /p:Configuration=Release
   /p:DeployOnBuild=true /p:WebPublishMethod=Package /p:PackageAsSingleFile=true
   /p:SkipInvalidConfigurations=true /p:PackageLocation=C:\PartsUnlimited
     6) Copy test assets to package dir:
        - LoadTest.jmx
        - jtl_junit_converter.py
     7) Upload artifact "JMeter" from C:\PartsUnlimited

   ### Stage 2 — dev (needs: build, environment: dev, runs-on: jmeter-test)
   - Steps:
     1) Download artifact "JMeter" to C:\Artifacts\PartsUnlimited\<run_number>
     2) Deploy site via MSDeploy from PartsUnlimitedWebsite.zip; set IIS Web Application
   Name=PartsUnlimited
     3) Run JMeter non-GUI on LoadTest.jmx; output <run_number>.jtl to same folder
     4) Convert JTL to JUnit XML via python jtl_junit_converter.py -> <run_number>.xml
     5) Upload artifact "JMeterTestFile" with the XML report

   ```mermaid
   flowchart TD
     A[workflow_dispatch] --> B[build (windows-2019)]
     B --> B1[Checkout]
     B1 --> B2[Setup NuGet]
     B2 --> B3[Restore packages]
     B3 --> B4[Setup MSBuild]
     B4 --> B5[Build & package -> C:\PartsUnlimited]
     B5 --> B6[Copy JMeter files]
     B6 --> B7[Upload artifact 'JMeter']
     B7 --> C[dev (env: dev, runs-on: jmeter-test)]
     C --> C1[Download 'JMeter' -> C:\Artifacts\PartsUnlimited\<run_number>]
     C1 --> C2[MSDeploy PartsUnlimitedWebsite.zip to IIS site 'PartsUnlimited']
     C2 --> C3[Run JMeter -n -t LoadTest.jmx -l <run_number>.jtl]
     C3 --> C4[Convert JTL -> JUnit XML <run_number>.xml]
     C4 --> C5[Upload artifact 'JMeterTestFile']
   ```

