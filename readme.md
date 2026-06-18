# Reusable GitHub Actions

A collection of reusable GitHub Actions workflows for use across USChamber web projects.

---

## Tag and Release

This reusable GitHub Action automates the versioning and release process across multiple projects. It creates semantic versioning tags based on the calendar year (YYYY.N format), generates release notes, and optionally integrates with Sentry and Jira.

### Features

- **Automatic Version Tagging**: Creates version tags in YYYY.N format (e.g., 2023.1, 2023.2)
- **Release Notes Generation**: Automatically generates release notes for each version
- **Sentry Integration**: Creates releases in Sentry with optional sourcemap uploads
- **Jira Integration**:
  - Creates version entries in Jira
  - Identifies related Jira issues from commit messages
  - Transitions issues to "Released to Prod" status
  - Adds issues to the appropriate version
  - Marks versions as released when issues are found

### Usage

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

### Configuration Options

#### Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `branch_name` | The branch name to monitor for merged PRs | No | `production` |
| `sentry_projects` | JSON array of Sentry projects to create releases for | No | `[]` |
| `jira_project_id` | Jira project ID to create versions for | No | `10017` |
| `jira_transition_id` | ID of the "Released to Prod" transition in Jira | No | `41` |
| `commits_to_search` | Number of recent commits to search for Jira IDs | No | `5` |
| `jira_issue_prefixes` | Regex pattern to identify Jira issue keys in commit messages | No | `(ENG\|CO\|FND\|USCC\|CL)-[0-9]+` |

#### Secrets

| Name | Description | Required |
|------|-------------|----------|
| `app_id` | GitHub App ID for generating token | Yes |
| `private_key` | GitHub App private key for generating token | Yes |
| `sentry_auth_token` | Sentry authentication token | No* |
| `sentry_org` | Sentry organization name | No* |
| `jira_auth` | Base64 encoded Jira authentication | No** |

\* Required if using Sentry integration
\** Required if using Jira integration

### Sentry Projects Format

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

### Requirements

- GitHub Actions workflow with permissions to create tags and releases
- For Sentry integration: Valid Sentry auth token and organization
- For Jira integration: Base64 encoded Jira authentication (username:api_token)

### How It Works

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

---

## Auto PR on Release

This reusable workflow polls a Composer package's GitHub releases and automatically opens a Claude-audited draft PR when a newer version is available. It is idempotent — if an open PR already exists for the detected version, no duplicate is created.

### Features

- **Release polling**: Compares the version in `composer.lock` against the latest GitHub release for the package
- **Automated PR**: Opens a draft PR on a `deps/{package}-{version}` branch with a bumped `composer.json` and updated `composer.lock`
- **Claude audit**: Runs the [`dependency-needs-audit`](https://github.com/USChamber/claude-plugins) skill to analyze breaking changes, behavioral changes, deprecations, and new capabilities between the old and new versions
- **Idempotent**: Skips if already up to date or an open PR for that version already exists
- **Graceful composer failures**: If `composer update` fails (e.g. due to constraint conflicts), the PR is still created with a note — the Claude audit can still run against the version diff

### Usage

Create a caller workflow in your repo. To watch multiple packages, add one job per package:

```yaml
name: auto-pr-on-release

on:
  schedule:
    - cron: '0 15 * * 0'  # Every Sunday at 3pm UTC
  workflow_dispatch:       # Allow manual runs

jobs:
  check-craftcms:
    uses: USChamber/handy-actions/.github/workflows/auto-pr-on-release.yml@main
    with:
      package: 'craftcms/cms'
      pr_target_branch: 'production'
      php_version: '8.3'
    secrets:
      app_id: ${{ vars.GITHUB_APP_ID }}
      private_key: ${{ secrets.GITHUB_APP_PRIVATE_KEY }}
      anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}

  # Add more jobs here to watch additional packages:
  # check-another-package:
  #   uses: USChamber/handy-actions/.github/workflows/auto-pr-on-release.yml@main
  #   with:
  #     package: 'vendor/package'
  #     pr_target_branch: 'production'
  #   secrets:
  #     app_id: ${{ vars.GITHUB_APP_ID }}
  #     private_key: ${{ secrets.GITHUB_APP_PRIVATE_KEY }}
  #     anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Configuration Options

#### Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `package` | Composer package to watch — must match its GitHub org/repo slug | No | `craftcms/cms` |
| `composer_json_path` | Path to `composer.json` in the calling repo | No | `composer.json` |
| `pr_target_branch` | Branch to open the PR against | No | `production` |
| `php_version` | PHP version for composer operations | No | `8.3` |

> **Note:** The `package` input assumes the Composer package name matches the GitHub repository slug (e.g. `craftcms/cms` → `github.com/craftcms/cms`). This holds true for most major open-source packages.

#### Secrets

| Name | Description | Required |
|------|-------------|----------|
| `app_id` | GitHub App ID for generating a token with write access | Yes |
| `private_key` | GitHub App private key | Yes |
| `anthropic_api_key` | Anthropic API key for Claude Code | Yes |

### How It Works

1. The workflow runs on your chosen schedule (or manually via `workflow_dispatch`)
2. It reads the currently installed version of the package from `composer.lock`
3. It fetches the latest release from the GitHub API for the package
4. If the versions match, it exits — nothing to do
5. If a newer version exists, it checks for an existing open PR for that version — if one exists, it exits
6. It updates the version constraint in `composer.json` and runs `composer update` for the package
7. It commits the changes to a new `deps/{package}-{version}` branch and pushes it
8. It opens a draft PR against the target branch with a compare diff link
9. Claude runs the `dependency-needs-audit` skill and posts a full audit (breaking changes, behavioral changes, deprecations, new capabilities) as a PR comment
