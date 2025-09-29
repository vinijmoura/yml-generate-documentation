# Workflow Documentation

This directory contains auto-generated documentation for GitHub Actions workflows.

## How it works

The workflow monitoring system automatically:

1. **Monitors** changes to `.github/workflows/*.yml` and `.github/workflows/*.yaml` files
2. **Detects** which workflow files have been modified in push or pull request events  
3. **Generates** comprehensive Markdown documentation for each changed workflow
4. **Commits** and pushes the updated documentation to the repository

## Documentation Structure

Each workflow file generates a corresponding documentation file:

- `.github/workflows/example.yml` → `docs/workflows/example.md`
- `.github/workflows/ci-cd.yml` → `docs/workflows/ci-cd.md`

## Generated Documentation Includes

- 📋 **Workflow Overview** - Name, triggers, and summary
- 🚀 **Triggers** - Events that start the workflow  
- 🏗️ **Jobs** - Detailed breakdown of each job and its steps
- 🔧 **Dependencies** - External actions and tools used
- 🔐 **Environment Variables and Secrets** - Configuration requirements
- 📄 **Complete Workflow File** - Full source code for reference

---

*Documentation is automatically updated when workflow files are modified.*