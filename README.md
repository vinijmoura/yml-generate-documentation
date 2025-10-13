# yml-generate-documentation

This project simplifies and automates the documentation process for your GitHub Actions workflows. With this utility, you can transform YAML workflow files into human-readable Markdown documentation, making it easy to understand the stages, jobs, and actions configured in your CI/CD pipeline.

## Motivation

Manually documenting GitHub Actions workflows is time-consuming and error-prone. With `yml-generate-documentation`, your documentation is generated directly from the YAML files themselves, ensuring it is always up-to-date and accurately reflects your workflow configuration.

> For more details, read this article: [From YAML to Markdown: Auto-document your GitHub Actions Workflows](https://vinijmoura.medium.com/from-yaml-to-markdown-auto-document-your-github-actions-workflows-51feaebd3cf7).

## How It Works

The utility reads your application's workflow YAML files, parses the structure (stages, jobs, actions, etc.), and generates a detailed Markdown file, ready to be used as your official documentation.

### Main Steps:

1. **Read the YAML file:** The script loads your GitHub Actions workflow file.
2. **Parse key elements:** It extracts stages, jobs, and actions, as well as comments and descriptions.
3. **Automatically generate Markdown:** Produces a `.md` file with clear, structured documentation.

### Parameters

- `path/to/your/workflow.yml`: Path to the YAML workflow file you want to document.

## Sample Output

```markdown
# CI Workflow

## Jobs

### build
- Runs on: ubuntu-latest
- Steps:
  - Checkout code
  - Install dependencies
  - Run tests

### deploy
- Runs on: ubuntu-latest
- Steps:
  - Build application
  - Deploy to production
```

## Customization

You can adapt the output templates to include project-specific information, such as badges, links, or extra job details.

## Contributing

Contributions are welcome! Feel free to open issues, submit pull requests, or suggest improvements.
