# yml-generate-documentation

Generate YML documentation (stages, jobs, actions) automatically for GitHub Actions workflows.

## 🚀 Features

This repository implements an automated pipeline monitoring system that:

1. **Monitors** any changes to GitHub Actions workflow files (`.github/workflows/*.yml`)
2. **Installs** GitHub CLI and necessary dependencies automatically  
3. **Generates** comprehensive Markdown documentation describing the workflow flow
4. **Commits** and pushes the generated documentation back to the repository

## 📋 How It Works

### Automatic Monitoring
- Triggers on push or pull request events that modify workflow files
- Detects exactly which workflow files have changed
- Processes each changed file individually

### Documentation Generation
- Uses `yq` for advanced YAML parsing and analysis
- Extracts workflow metadata (name, triggers, jobs, dependencies)
- Generates structured Markdown documentation with:
  - Workflow overview and triggers
  - Detailed job breakdown with steps
  - External dependencies and actions used
  - Environment variables and secrets
  - Complete workflow source code

### Automatic Updates
- Commits generated documentation with descriptive messages
- Pushes changes back to the repository
- Provides workflow summaries in GitHub Actions

## 📁 Repository Structure

```
.github/
├── workflows/
│   ├── monitor-workflows.yml    # Main monitoring pipeline
│   └── sample-cicd.yml         # Example workflow for testing
docs/
└── workflows/
    ├── README.md               # Documentation guide
    ├── monitor-workflows.md    # Auto-generated docs for monitor
    └── sample-cicd.md         # Auto-generated docs for sample
```

## 🛠️ Setup

The monitoring system is automatically active once the workflow files are in place. No additional setup is required!

### Requirements
- Repository with GitHub Actions enabled
- `GITHUB_TOKEN` with appropriate permissions (automatically provided)

## 📖 Usage

1. **Add or modify** any workflow file in `.github/workflows/`
2. **Push** the changes or create a pull request
3. **Watch** as the monitoring pipeline automatically:
   - Detects the changes
   - Generates documentation
   - Commits the results

## 🔍 Example

When you modify `.github/workflows/ci-cd.yml`, the system will:
- Generate `docs/workflows/ci-cd.md` with complete documentation
- Include workflow overview, triggers, jobs, and dependencies
- Commit with message: "📝 Auto-update workflow documentation"

## 📝 Generated Documentation

Each workflow generates documentation that includes:

- **📋 Workflow Overview** - Summary and metadata
- **🚀 Triggers** - Events and conditions  
- **🏗️ Jobs** - Step-by-step job breakdown
- **🔧 Dependencies** - External actions used
- **🔐 Environment Variables** - Configuration requirements
- **📄 Complete Source** - Full workflow file content

View examples in the [`docs/workflows/`](docs/workflows/) directory.

---

*This project demonstrates automated documentation generation for DevOps workflows using GitHub Actions.*
