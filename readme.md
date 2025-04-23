# Reusable Tag and Release GitHub Action

This reusable GitHub Action automates the versioning and release process across multiple projects. It creates semantic versioning tags based on the calendar year (YYYY.N format), generates release notes, creates Sentry releases, and updates Jira issues and versions.

## Features

- **Automatic Version Tagging**: Creates version tags in YYYY.N format (e.g., 2023.1, 2023.2)
- **Release Notes Generation**: Automatically generates release notes for each version
- **Sentry Integration**: Creates releases in Sentry with optional sourcemap uploads
- **Jira Integration**: 
  - Creates version entries in Jira
  - Identifies related Jira issues from commit messages
  - Transitions issues to "Released to Prod" status
  - Adds issues to the appropriate version
  - Marks versions as released when issues are found

## Usage

1. Create a `.github/workflows` directory in your repository if it doesn't exist already
2. Create a new workflow file (e.g., `tag-and-release.yml`) with the following content:

```yaml
name: Tag and Release Production

on:
  pull_request:
    types: [ closed ]
    branches:
      - production

jobs:
  call-tag-and-release:
    if: github.event.pull_request.merged == true
    uses: USChamber/handy-actions/.github/workflows/tag-and-release.yml@main
    with:
      # Optional configurations
      branch_name: 'production'
      sentry_projects: '[{"name": "my-project-js", "sourceMapPath": "dist/assets"}, {"name": "my-project-php"}]'
      jira_project_id: '10017'
    secrets:
      app_id: ${{ vars.GITHUB_APP_ID }}
      private_key: ${{ secrets.GITHUB_APP_PRIVATE_KEY }}
      sentry_auth_token: ${{ secrets.SENTRY_AUTH_TOKEN }}
      sentry_org: 'your-sentry-org'
      jira_auth: ${{ secrets.JIRA_AUTH }}
```

## Configuration Options

### Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `branch_name` | The branch name to monitor for merged PRs | No | `production` |
| `sentry_projects` | JSON array of Sentry projects to create releases for | No | `[]` |
| `jira_project_id` | Jira project ID to create versions for | No | `10017` |
| `jira_transition_id` | ID of the "Released to Prod" transition in Jira | No | `41` |
| `commits_to_search` | Number of recent commits to search for Jira IDs | No | `5` |
| `jira_issue_prefixes` | Regex pattern to identify Jira issue keys in commit messages | No | `(ENG\|CO\|FND\|USCC\|CL)-[0-9]+` |

### Secrets

| Name | Description | Required |
|------|-------------|----------|
| `app_id` | GitHub App ID for generating token | Yes |
| `private_key` | GitHub App private key for generating token | Yes |
| `sentry_auth_token` | Sentry authentication token | No* |
| `sentry_org` | Sentry organization name | No* |
| `jira_auth` | Base64 encoded Jira authentication | No** |

\* Required if using Sentry integration  
\** Required if using Jira integration

## Sentry Projects Format

The `sentry_projects` input should be a JSON array with the following structure:

```json
[
  {
    "name": "project-name",
    "sourceMapPath": "optional/path/to/sourcemaps"
  },
  {
    "name": "another-project"
  }
]
```

## Requirements

- GitHub Actions workflow with permissions to create tags and releases
- For Sentry integration: Valid Sentry auth token and organization
- For Jira integration: Base64 encoded Jira authentication (username:api_token)

## How It Works

1. When a PR is merged to the specified branch (default: production), the action runs
2. It creates a new version tag in YYYY.N format
3. It creates a GitHub release with automatically generated release notes
4. If configured, it creates releases in Sentry
5. If configured, it:
   - Creates a new version in Jira
   - Finds Jira issues referenced in recent commits
   - Transitions these issues to "Released to Prod" status
   - Adds these issues to the new version
   - Also finds any other issues already in "Released to Prod" status without a version
   - Marks the version as released if any issues were found and added
