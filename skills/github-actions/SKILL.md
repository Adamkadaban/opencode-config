---
name: github-actions
description: Expert guidance for GitHub Actions CI/CD workflows. Use when creating or modifying workflow files, configuring triggers, setting up jobs and steps, using matrix strategies, managing secrets and variables, selecting runners, or troubleshooting workflow issues. Covers workflow syntax, events, contexts, expressions, reusable workflows, and common actions from the GitHub Marketplace.
---

# GitHub Actions: CI/CD Workflow Automation

## Overview

GitHub Actions is a CI/CD platform that automates build, test, and deployment pipelines directly from GitHub repositories. Workflows are defined in YAML files stored in `.github/workflows/` and triggered by events like pushes, pull requests, schedules, or manual dispatch.

## When to Use This Skill

Use this skill when:

- Creating new CI/CD workflow files
- Configuring workflow triggers (push, pull_request, schedule, workflow_dispatch)
- Setting up jobs with steps, actions, and shell commands
- Using matrix strategies for parallel testing
- Managing secrets, variables, and environment configuration
- Selecting appropriate runners (ubuntu-latest, windows-latest, macos-latest)
- Implementing job dependencies and conditional execution
- Using reusable workflows and composite actions
- Integrating common actions (checkout, setup-*, cache, artifacts)
- Troubleshooting workflow failures and debugging

> [!IMPORTANT]
> **Security is not optional.** A workflow file is privileged code with access to
> your repository, secrets, and a write-capable `GITHUB_TOKEN`. **After writing or
> modifying *any* file in `.github/workflows/`, you must audit it before
> committing** — pin actions with `pinact`, scan with `zizmor`, and apply
> least-privilege permissions. See the [Security](#security) section below and the
> full [./references/security.md](./references/security.md) guide.

## Workflow File Structure

Workflows are YAML files in `.github/workflows/`:

```text
.github/
└── workflows/
    ├── ci.yml           # Continuous integration
    ├── deploy.yml       # Deployment pipeline
    ├── release.yml      # Release automation
    └── scheduled.yml    # Scheduled tasks
```

## Core Workflow Syntax

### Minimal Workflow

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Hello, World!"
```

### Complete Workflow Structure

```yaml
name: Complete CI Pipeline

on:
  push:
    branches: [main, develop]
    paths:
      - "src/**"
      - "tests/**"
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: "Deployment environment"
        required: true
        default: "staging"
        type: choice
        options:
          - staging
          - production

env:
  NODE_VERSION: "20"
  REGISTRY: ghcr.io

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

  build:
    name: Build Application
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    outputs:
      version: ${{ steps.version.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Get version
        id: version
        run: echo "version=$(cat VERSION)" >> $GITHUB_OUTPUT

      - name: Build
        run: npm run build

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 5
```

## Trigger Events

### Common Event Triggers

```yaml
on:
  # Code changes
  push:
    branches: [main, "release/*"]
    tags: ["v*"]
    paths:
      - "src/**"
      - "!src/**/*.md"
    paths-ignore:
      - "docs/**"
      - "*.md"

  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, ready_for_review]

  # Manual triggers
  workflow_dispatch:
    inputs:
      version:
        description: "Release version"
        required: true
        type: string
      deploy:
        description: "Deploy after build"
        type: boolean
        default: false

  # Scheduled runs (UTC timezone)
  schedule:
    - cron: "0 2 * * *" # Daily at 2 AM UTC
    - cron: "0 */6 * * *" # Every 6 hours

  # Repository events
  release:
    types: [published, created]

  issues:
    types: [opened, labeled]

  # Cross-workflow
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
    secrets:
      token:
        required: true

  workflow_run:
    workflows: ["Build"]
    types: [completed]
    branches: [main]
```

### Event Filters

```yaml
on:
  push:
    branches:
      - main
      - "releases/**"
      - "!releases/**-alpha" # Exclude alpha releases
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+" # Semantic version tags only
    paths:
      - "**.js"
      - "**.ts"
      - "package*.json"
```

For complete event reference, see [./references/events.md](./references/events.md).

## Jobs and Steps

### Job Configuration

```yaml
jobs:
  build:
    name: Build Job
    runs-on: ubuntu-latest

    # Job-level environment variables
    env:
      CI: true
      NODE_ENV: test

    # Timeout (default: 360 minutes)
    timeout-minutes: 30

    # Continue workflow if this job fails
    continue-on-error: false

    # Concurrency control
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    # Required permissions
    permissions:
      contents: read
      packages: write

    # Environment for deployments
    environment:
      name: production
      url: https://example.com

    # Service containers
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

### Step Types

```yaml
steps:
  # Action step
  - name: Checkout code
    uses: actions/checkout@v4
    with:
      fetch-depth: 0
      token: ${{ secrets.GITHUB_TOKEN }}

  # Shell command step
  - name: Run tests
    run: npm test
    shell: bash
    working-directory: ./app
    env:
      NODE_ENV: test

  # Multi-line command
  - name: Build and test
    run: |
      npm ci
      npm run build
      npm test

  # Conditional step
  - name: Deploy
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh

  # Step with outputs
  - name: Get version
    id: version
    run: echo "version=$(cat VERSION)" >> $GITHUB_OUTPUT

  # Use output from previous step
  - name: Tag release
    run: echo "Creating release ${{ steps.version.outputs.version }}"
```

### Job Dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building..."

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing..."

  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    if: needs.build.result == 'success' && needs.test.result == 'success'
    steps:
      - run: echo "Deploying..."
```

## Matrix Strategies

### Basic Matrix

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest, windows-latest]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

### Advanced Matrix Configuration

```yaml
jobs:
  test:
    strategy:
      fail-fast: false # Don't cancel other jobs if one fails
      max-parallel: 4 # Limit concurrent jobs
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        python-version: ["3.10", "3.11", "3.12"]

        # Include additional combinations
        include:
          - os: ubuntu-latest
            python-version: "3.13"
            experimental: true

        # Exclude specific combinations
        exclude:
          - os: macos-latest
            python-version: "3.10"

    runs-on: ${{ matrix.os }}
    continue-on-error: ${{ matrix.experimental || false }}

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pytest
```

## Contexts and Expressions

### Common Contexts

```yaml
steps:
  - name: Context examples
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run number: ${{ github.run_number }}"
      echo "Workflow: ${{ github.workflow }}"
      echo "Runner OS: ${{ runner.os }}"
      echo "Runner arch: ${{ runner.arch }}"

  - name: Access secrets
    env:
      API_KEY: ${{ secrets.API_KEY }}
    run: ./script.sh

  - name: Access variables
    env:
      ENVIRONMENT: ${{ vars.ENVIRONMENT }}
    run: ./deploy.sh
```

### Conditional Expressions

```yaml
steps:
  # Branch conditions
  - if: github.ref == 'refs/heads/main'
    run: echo "On main branch"

  - if: startsWith(github.ref, 'refs/tags/v')
    run: echo "Tagged release"

  # Event conditions
  - if: github.event_name == 'pull_request'
    run: echo "Pull request"

  - if: github.event.pull_request.draft == false
    run: echo "Ready for review"

  # Job status conditions
  - if: success()
    run: echo "Previous steps succeeded"

  - if: failure()
    run: echo "Previous step failed"

  - if: always()
    run: echo "Always runs"

  - if: cancelled()
    run: echo "Workflow was cancelled"

  # Complex conditions
  - if: |
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main' &&
      !contains(github.event.head_commit.message, '[skip ci]')
    run: ./deploy.sh
```

### Expression Functions

```yaml
steps:
  - name: String functions
    run: |
      # contains, startsWith, endsWith
      echo "${{ contains(github.event.head_commit.message, 'fix') }}"
      echo "${{ startsWith(github.ref, 'refs/tags/') }}"
      echo "${{ endsWith(github.repository, '-app') }}"

      # format
      echo "${{ format('Hello {0} {1}', 'World', '!') }}"

      # join
      echo "${{ join(matrix.os, ', ') }}"

  - name: JSON functions
    run: |
      echo '${{ toJSON(github.event) }}'
      echo '${{ fromJSON(needs.build.outputs.config).version }}'

  - name: Hash functions
    run: |
      echo "${{ hashFiles('**/package-lock.json') }}"
      echo "${{ hashFiles('**/*.rs', '**/Cargo.lock') }}"
```

For complete context reference, see [./references/contexts.md](./references/contexts.md).

## Secrets and Variables

### Using Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # Environment variable
      - name: Deploy with API key
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: ./deploy.sh

      # Action input
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

### Repository and Organization Variables

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Use repository variable
        env:
          APP_ENV: ${{ vars.ENVIRONMENT }}
          API_URL: ${{ vars.API_URL }}
        run: ./build.sh
```

### GITHUB_TOKEN Permissions

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Create releases
      packages: write # Push to GHCR
      issues: write # Comment on issues
      pull-requests: write # Comment on PRs
      id-token: write # OIDC token for trusted publishing

    steps:
      - uses: actions/checkout@v4
      - name: Create release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: gh release create v1.0.0
```

## Runners

### GitHub-Hosted Runners

| Runner Label       | OS              | Architecture | CPUs | RAM  |
| ------------------ | --------------- | ------------ | ---- | ---- |
| `ubuntu-latest`    | Ubuntu 24.04    | x64          | 4    | 16GB |
| `ubuntu-24.04`     | Ubuntu 24.04    | x64          | 4    | 16GB |
| `ubuntu-22.04`     | Ubuntu 22.04    | x64          | 4    | 16GB |
| `ubuntu-24.04-arm` | Ubuntu 24.04    | arm64        | 4    | 16GB |
| `windows-latest`   | Windows Server  | x64          | 4    | 16GB |
| `windows-2025`     | Windows 2025    | x64          | 4    | 16GB |
| `windows-2022`     | Windows 2022    | x64          | 4    | 16GB |
| `macos-latest`     | macOS 14 Sonoma | arm64 (M1)   | 3    | 7GB  |
| `macos-15`         | macOS 15        | arm64 (M1)   | 3    | 7GB  |
| `macos-14`         | macOS 14        | arm64 (M1)   | 3    | 7GB  |
| `macos-13`         | macOS 13        | x64 (Intel)  | 4    | 14GB |

### Runner Selection

```yaml
jobs:
  # Single runner
  linux:
    runs-on: ubuntu-latest

  # Matrix of runners
  cross-platform:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}

  # Self-hosted runner
  deploy:
    runs-on: [self-hosted, linux, x64]

  # Larger runner (GitHub Team/Enterprise)
  heavy-build:
    runs-on: ubuntu-latest-16-cores
```

## Common Actions

### Checkout

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0 # Full history for tags/versioning
    token: ${{ secrets.PAT }} # For private submodules
    submodules: recursive
    lfs: true
```

### Setup Actions

```yaml
# Node.js
- uses: actions/setup-node@v4
  with:
    node-version: "20"
    node-version-file: ".nvmrc" # Or package.json
    cache: "npm" # Or 'yarn', 'pnpm'
    registry-url: "https://npm.pkg.github.com"

# Python
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
    python-version-file: ".python-version"
    cache: "pip"

# Go
- uses: actions/setup-go@v5
  with:
    go-version: "1.22"
    cache: true

# Java
- uses: actions/setup-java@v4
  with:
    distribution: "temurin"
    java-version: "21"
    cache: "maven"

# .NET
- uses: actions/setup-dotnet@v4
  with:
    dotnet-version: "8.0.x"
```

### Caching

```yaml
# Generic cache
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/pip
      ~/.npm
      node_modules
    key: ${{ runner.os }}-deps-${{ hashFiles('**/package-lock.json', '**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-deps-

# Cache with save-always
- uses: actions/cache@v4
  with:
    path: ~/.cache
    key: cache-${{ github.run_id }}
    restore-keys: cache-
    save-always: true # Save even if job fails
```

### Artifacts

```yaml
# Upload artifact
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: |
      dist/
      !dist/**/*.map
    retention-days: 5
    if-no-files-found: error # 'warn' or 'ignore'

# Download artifact
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: ./artifacts

# Download all artifacts
- uses: actions/download-artifact@v4
  with:
    path: ./all-artifacts
    merge-multiple: true
```

For more actions, see [./references/common-actions.md](./references/common-actions.md).

## Reusable Workflows

### Calling a Reusable Workflow

```yaml
jobs:
  call-workflow:
    uses: owner/repo/.github/workflows/reusable.yml@main
    with:
      environment: production
    secrets:
      token: ${{ secrets.DEPLOY_TOKEN }}
    # Or inherit all secrets
    secrets: inherit
```

### Defining a Reusable Workflow

```yaml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      version:
        required: false
        type: string
        default: "latest"
    secrets:
      deploy_token:
        required: true
    outputs:
      url:
        description: "Deployment URL"
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        id: deploy
        env:
          TOKEN: ${{ secrets.deploy_token }}
        run: |
          echo "Deploying to ${{ inputs.environment }}"
          echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
```

## Limits and Quotas

### Execution Limits

| Limit                          | Value                       |
| ------------------------------ | --------------------------- |
| Job execution time             | 6 hours (hosted), 5 days    |
| Workflow run time              | 35 days                     |
| Job queue time                 | 24 hours                    |
| API requests per hour          | 1,000 per repository        |
| Concurrent jobs (Free)         | 20                          |
| Concurrent jobs (Pro)          | 40                          |
| Concurrent jobs (Team)         | 60                          |
| Concurrent jobs (Enterprise)   | 500                         |
| Matrix jobs per workflow       | 256                         |
| Workflows per push event       | 500                         |
| Nested workflow calls          | 4 levels                    |
| Environment variables per step | 100 (4KB per variable)      |
| Log size                       | 64KB per step, 6GB per run  |
| Artifact storage               | 500MB (Free), varies by plan|
| Cache storage                  | 10GB per repository         |

### Minutes and Storage by Plan

| Plan       | Included Minutes | Storage |
| ---------- | ---------------- | ------- |
| Free       | 2,000/month      | 500MB   |
| Pro        | 3,000/month      | 1GB     |
| Team       | 3,000/month      | 2GB     |
| Enterprise | 50,000/month     | 50GB    |

## Best Practices

### Security

> **Mandatory after every workflow change.** Before committing any new or edited
> workflow, run the audit pipeline and fix what it reports:
>
> ```bash
> pinact run .github/workflows/   # pin every action to an immutable commit SHA
> zizmor .github/workflows/       # static security audit (injection, perms, ...)
> actionlint                      # syntax + expression validation
> ```
>
> `zizmor` is the auditing tool of choice for finding template injection,
> excessive permissions, unpinned actions, credential persistence, and dangerous
> triggers. `pinact` rewrites mutable tags (`@v4`) to immutable SHAs while keeping
> a readable version comment. Full guidance, install steps, CI gates, and the
> `sha_pinning_required` caveat are in [./references/security.md](./references/security.md).

```yaml
# Principle of least privilege
permissions:
  contents: read
  # Add only what's needed

# Pin actions to SHA
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

# Harden checkout: don't leave the token on disk unless you must push
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
  with:
    persist-credentials: false

# Never interpolate untrusted input into a shell — pass it via env instead
- env:
    TITLE: ${{ github.event.pull_request.title }}
  run: echo "Building $TITLE"

# Use environments for deployments
jobs:
  deploy:
    environment:
      name: production
      url: ${{ steps.deploy.outputs.url }}

# Validate inputs
- name: Validate version
  run: |
    if [[ ! "${{ inputs.version }}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
      echo "Invalid version format"
      exit 1
    fi
```

### Performance

```yaml
# Use caching
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}

# Cancel redundant runs
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Filter paths to avoid unnecessary runs
on:
  push:
    paths:
      - "src/**"
      - "tests/**"
    paths-ignore:
      - "**.md"
      - "docs/**"

# Use matrix for parallel testing
strategy:
  matrix:
    shard: [1, 2, 3, 4]
```

### Maintainability

```yaml
# Use environment variables for repeated values
env:
  NODE_VERSION: "20"
  PYTHON_VERSION: "3.12"

# Extract complex logic to scripts
- name: Complex operation
  run: ./scripts/complex-operation.sh

# Use reusable workflows
jobs:
  ci:
    uses: ./.github/workflows/ci-template.yml

# Document workflow purpose
name: CI Pipeline
# Description: Runs tests, linting, and builds on every push
```

## Local Development Tools

### act - Run Workflows Locally

Use [act](https://github.com/nektos/act) to run GitHub Actions locally using Docker. Useful for fast iteration without pushing to GitHub.

**Installation:**

```bash
# macOS
brew install act

# Linux (script)
curl -s https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Go install
go install github.com/nektos/act@latest

# Windows (scoop/choco)
scoop install act
choco install act-cli
```

**Basic Usage:**

```bash
# Run default event (push) for all workflows
act

# Run specific event
act pull_request
act workflow_dispatch

# Run specific workflow
act -W .github/workflows/ci.yml

# Run specific job
act -j build

# Dry run (show what would run)
act -n

# List available jobs
act -l
```

**Runner Images:**

```bash
# Use micro image (fastest, minimal tools)
act -P ubuntu-latest=catthehacker/ubuntu:act-latest

# Use medium image (better compatibility)
act -P ubuntu-latest=catthehacker/ubuntu:act-22.04

# Use full image (closest to GitHub, largest)
act -P ubuntu-latest=catthehacker/ubuntu:full-22.04

# Set default in ~/.actrc
echo "-P ubuntu-latest=catthehacker/ubuntu:act-latest" >> ~/.actrc
```

**Passing Secrets and Variables:**

```bash
# From environment
act -s GITHUB_TOKEN

# Inline
act -s MY_SECRET=value

# From file
act --secret-file .secrets

# Variables
act --var MY_VAR=value
act --var-file .vars
```

**Simulating GitHub Context:**

```bash
# Pass inputs for workflow_dispatch
act workflow_dispatch --input version=1.0.0

# Set actor
act --actor=octocat

# Simulate event payload
act pull_request --eventpath=event.json
```

**Limitations:**

- No Windows runner support (`windows-latest`)
- No macOS runner support (`macos-latest`)
- Some GitHub-hosted runner features unavailable
- Service containers require Docker-in-Docker
- Large runner images (full image ~20GB)
- Some actions may behave differently locally
- GITHUB_TOKEN has limited functionality

**Best practices for act compatibility:**

```yaml
# Use environment variable fallbacks
env:
  RUNNER_TOOL_CACHE: ${{ runner.tool_cache || '/opt/hostedtoolcache' }}

# Check if running in act
- if: ${{ !env.ACT }}
  run: echo "Running on GitHub"

# Avoid GitHub-specific features in local testing
- uses: actions/cache@v4
  if: ${{ !env.ACT }}  # Cache doesn't work well in act
```

### actionlint

Use [actionlint](https://github.com/rhysd/actionlint) to validate workflow files before pushing.

**Installation:**

```bash
# macOS
brew install actionlint

# Go install
go install github.com/rhysd/actionlint/cmd/actionlint@latest

# Download binary
curl -sL https://github.com/rhysd/actionlint/releases/latest/download/actionlint_linux_amd64.tar.gz | tar xz
```

**Usage:**

```bash
# Check all workflows
actionlint

# Check specific file
actionlint .github/workflows/ci.yml

# Output as JSON (for CI integration)
actionlint -format json

# With shellcheck integration (checks run: scripts)
actionlint -shellcheck
```

**What actionlint catches:**

- Invalid YAML syntax
- Unknown workflow keys
- Invalid event configurations
- Type mismatches in expressions
- Undefined action inputs
- Invalid glob patterns
- Shell script issues (with shellcheck)
- Deprecated commands and features

**Pre-commit integration:**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/rhysd/actionlint
    rev: v1.7.4
    hooks:
      - id: actionlint
```

**CI integration:**

```yaml
name: Lint Workflows
on: [push, pull_request]

jobs:
  actionlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: reviewdog/action-actionlint@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          reporter: github-pr-review
```

## Monitoring Workflows with gh CLI

The GitHub CLI (`gh`) is essential for watching workflow runs and debugging failures from the terminal.

### List Workflow Runs

```bash
# List recent runs for current repo
gh run list

# Filter by workflow
gh run list --workflow=ci.yml

# Filter by branch
gh run list --branch=main

# Filter by status
gh run list --status=failure
gh run list --status=in_progress

# Filter by user
gh run list --user=octocat

# Limit results
gh run list --limit=5

# JSON output for scripting
gh run list --json status,conclusion,name,databaseId
```

### Watch a Running Workflow

```bash
# Watch the most recent run (live updates)
gh run watch

# Watch specific run by ID
gh run watch 1234567890

# Watch with exit code matching run result (useful in scripts)
gh run watch --exit-status

# Watch after triggering a workflow
gh workflow run deploy.yml && gh run watch
```

### View Run Details and Logs

```bash
# View summary of most recent run
gh run view

# View specific run
gh run view 1234567890

# View with full log output
gh run view --log

# View failed steps only
gh run view --log-failed

# View specific job's logs
gh run view --job=9876543210

# View logs in web browser
gh run view --web
```

### Download Logs and Artifacts

```bash
# Download logs for a run
gh run download 1234567890 --log

# Download artifacts
gh run download 1234567890

# Download specific artifact
gh run download 1234567890 --name=build-output

# Download to specific directory
gh run download 1234567890 --dir=./artifacts
```

### Trigger Workflows Manually

```bash
# Trigger workflow_dispatch event
gh workflow run deploy.yml

# With inputs
gh workflow run deploy.yml -f environment=production -f version=1.2.3

# From specific branch
gh workflow run deploy.yml --ref=feature-branch

# Trigger and watch
gh workflow run ci.yml && sleep 2 && gh run watch
```

### Rerun Failed Workflows

```bash
# Rerun all jobs in a run
gh run rerun 1234567890

# Rerun only failed jobs
gh run rerun 1234567890 --failed

# Rerun specific job
gh run rerun 1234567890 --job=build

# Rerun with debug logging enabled
gh run rerun 1234567890 --debug
```

### Cancel a Running Workflow

```bash
gh run cancel 1234567890
```

### Common Debugging Workflow

```bash
# 1. Check recent failures
gh run list --status=failure --limit=5

# 2. View the failed run
gh run view <run-id> --log-failed

# 3. If still unclear, view full logs
gh run view <run-id> --log

# 4. Fix the issue, push, and watch
git push && gh run watch --exit-status
```

## Troubleshooting

### Debug Logging

```yaml
# Enable debug logging via repository secret
# Set ACTIONS_RUNNER_DEBUG=true
# Set ACTIONS_STEP_DEBUG=true

steps:
  - name: Debug info
    run: |
      echo "Event: ${{ toJSON(github.event) }}"
      echo "Context: ${{ toJSON(github) }}"
    env:
      ACTIONS_STEP_DEBUG: true
```

### Common Issues

**Workflow not triggering:**

- Check branch filters match your branch
- Check path filters include changed files
- Verify workflow file is in default branch for some events
- Check if workflow is disabled in repository settings

**Permission denied errors:**

```yaml
# Ensure correct permissions
permissions:
  contents: write
  packages: write
```

**Cache not restoring:**

- Verify key pattern matches exactly
- Check cache hasn't expired (7 days unused)
- Ensure path is correct for runner OS

**Action version issues:**

```yaml
# Pin to specific version
- uses: actions/checkout@v4.2.2

# Or SHA for maximum reproducibility
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
```

## Resources

### Reference Documentation

- [./references/security.md](./references/security.md) - **Security hardening, `zizmor` auditing, `pinact` SHA-pinning (read after writing any workflow)**
- [./references/syntax.md](./references/syntax.md) - Complete workflow syntax
- [./references/events.md](./references/events.md) - All trigger events
- [./references/contexts.md](./references/contexts.md) - Context objects and expressions
- [./references/common-actions.md](./references/common-actions.md) - Popular GitHub Actions

### External Resources

- Official docs: <https://docs.github.com/en/actions>
- Workflow syntax: <https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions>
- Actions Marketplace: <https://github.com/marketplace?type=actions>
- Starter workflows: <https://github.com/actions/starter-workflows>
- Security hardening: <https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions>
- zizmor (workflow security auditor): <https://docs.zizmor.sh>
- pinact (pin actions to SHAs): <https://github.com/suzuki-shunsuke/pinact>
