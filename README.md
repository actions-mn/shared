# Shared Workflows for actions-mn

This repository contains reusable GitHub Actions workflows shared across all [actions-mn](https://github.com/actions-mn) GitHub Actions.

## Overview

The actions-mn organization provides GitHub Actions for working with [Metanorma](https://www.metanorma.org/), a suite of tools for authoring standards documents. To ensure consistency and reduce duplication, this repository provides reusable workflows that can be called from any action in the organization.

## Available Workflows

### lint-test.yml

Reusable workflow for building, linting, and testing Node.js-based GitHub Actions.

**What it does:**
1. Checks out the repository
2. Sets up Node.js
3. Installs dependencies with `npm ci`
4. Builds the action with `npm run build`
5. Checks code formatting with `npm run format-check`
6. (Optionally) Runs linting with `npm run lint`
7. Runs unit tests with `npm test`
8. Verifies `dist/` integrity (no uncommitted changes after build)

**Usage:**
```yaml
name: test

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint-test:
    uses: actions-mn/shared/.github/workflows/lint-test.yml@main
```

**With optional linting:**
```yaml
jobs:
  lint-test:
    uses: actions-mn/shared/.github/workflows/lint-test.yml@main
    with:
      run-lint: true
```

**Inputs:**

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `node-version` | Node.js version to use | No | `24.x` |
| `run-lint` | Whether to run `npm run lint` | No | `false` |

### check-dist.yml

Reusable workflow for verifying the `dist/` directory is up to date with the source code.

**What it does:**
1. Checks out the repository
2. Sets up Node.js
3. Installs dependencies with `npm ci`
4. Rebuilds with `npm run build`
5. Compares the rebuilt `dist/` with the committed version
6. If different, fails and uploads the expected `dist/` as an artifact

**Usage:**
```yaml
name: Check dist

on:
  push:
    branches: [main]
    paths_ignore:
      - '**.md'
  pull_request:
    paths_ignore:
      - '**.md'
  workflow_dispatch:

jobs:
  check-dist:
    uses: actions-mn/shared/.github/workflows/check-dist.yml@main
```

**Inputs:**

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `node-version` | Node.js version to use | No | `24.x` |

### release.yml

Reusable workflow for releasing new versions of GitHub Actions. Handles version bumping, building, tagging, and GitHub releases.

**What it does:**
1. Checks out the repository
2. Sets up Node.js
3. Installs dependencies
4. Gets version from input or package.json
5. Updates package.json version (if specified)
6. Runs tests
7. Builds the action
8. Commits dist/ changes
9. Creates and pushes version tag
10. Updates major version tag (e.g., v3 -> v3.2.1)
11. Creates GitHub Release (optional)

**Usage:**
```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release (e.g., 1.2.0, leave empty to use package.json version)'
        required: false
        type: string
      create_release:
        description: 'Create a GitHub Release'
        required: false
        type: boolean
        default: true

permissions:
  contents: write
  id-token: write

jobs:
  release:
    uses: actions-mn/shared/.github/workflows/release.yml@main
    with:
      version: ${{ inputs.version }}
      create_release: ${{ inputs.create_release }}
```

**Inputs:**

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `version` | Version to release (empty = use package.json) | No | `''` |
| `create_release` | Whether to create a GitHub Release | No | `true` |
| `git_user_email` | Git user email for commits | No | `41898282+github-actions[bot]@users.noreply.github.com` |
| `git_user_name` | Git user name for commits | No | `github-actions[bot]` |
| `release_body` | Custom release body template | No | `''` |

### update-major-version.yml

Reusable workflow for updating major version tags. Can also be used as a rollback tool to point a major version tag to a specific commit.

**What it does:**
1. Checks out the repository with full history
2. Configures git user
3. Force-updates the major version tag to point to the target
4. Pushes the updated tag

**Usage:**
```yaml
name: Update Main Version

on:
  workflow_dispatch:
    inputs:
      major_version:
        description: 'The major version tag to update'
        required: true
        type: choice
        options:
          - v3
          - v2
          - v1
      target:
        description: 'The tag or reference to point to'
        required: true
        type: string

permissions:
  contents: write

jobs:
  update-tag:
    uses: actions-mn/shared/.github/workflows/update-major-version.yml@main
    with:
      major_version: ${{ inputs.major_version }}
      target: ${{ inputs.target }}
```

**Inputs:**

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `major_version` | The major version tag to update (e.g., v1, v2, v3) | Yes | - |
| `target` | The tag or reference to point to | Yes | - |

## Setting Up a New actions-mn Action

### 1. Package Structure

Your action should have the following structure:

```
my-action/
├── .github/
│   └── workflows/
│       ├── test.yml
│       ├── check-dist.yml
│       ├── release.yml
│       └── update-main-version.yml
├── src/
│   └── main.ts
├── dist/
│   └── index.cjs
├── action.yml
├── package.json
├── tsconfig.json
└── README.md
```

### 2. Required npm Scripts

Your `package.json` must have these scripts:

```json
{
  "scripts": {
    "build": "tsc --noEmit && esbuild src/main.ts --bundle --platform=node --target=node24 --format=cjs --outfile=dist/index.cjs --minify --sourcemap",
    "format": "prettier --write '**/*.ts'",
    "format-check": "prettier --check '**/*.ts'",
    "test": "vitest"
  }
}
```

NOTE: We use `.cjs` extension because `package.json` has `"type": "module"` which makes Node.js treat `.js` files as ESM. Since esbuild outputs CJS format (for compatibility with `@actions/cache` dependencies), we use `.cjs` to ensure proper module resolution.

Optional scripts:
```json
{
  "scripts": {
    "lint": "eslint src"
  }
}
```

### 3. Workflow Files

Create `.github/workflows/test.yml`:

```yaml
name: test

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  lint-test:
    uses: actions-mn/shared/.github/workflows/lint-test.yml@main

  # Add your integration tests here, depending on the lint-test job
  integration-test:
    needs: lint-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: ./
      - run: your-command --version
```

Create `.github/workflows/check-dist.yml`:

```yaml
name: Check dist

on:
  push:
    branches: [main]
    paths_ignore:
      - '**.md'
  pull_request:
    paths_ignore:
      - '**.md'
  workflow_dispatch:

jobs:
  check-dist:
    uses: actions-mn/shared/.github/workflows/check-dist.yml@main
```

Create `.github/workflows/release.yml`:

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release (e.g., 1.2.0)'
        required: false
        type: string
      create_release:
        description: 'Create a GitHub Release'
        required: false
        type: boolean
        default: true

permissions:
  contents: write
  id-token: write

jobs:
  release:
    uses: actions-mn/shared/.github/workflows/release.yml@main
    with:
      version: ${{ inputs.version }}
      create_release: ${{ inputs.create_release }}
```

Create `.github/workflows/update-main-version.yml`:

```yaml
name: Update Main Version

on:
  workflow_dispatch:
    inputs:
      major_version:
        description: 'The major version to update'
        required: true
        type: choice
        options:
          - v3
          - v2
          - v1
      target:
        description: 'The tag or reference to use'
        required: true

permissions:
  contents: write

jobs:
  tag:
    uses: actions-mn/shared/.github/workflows/update-major-version.yml@main
    with:
      major_version: ${{ inputs.major_version }}
      target: ${{ inputs.target }}
```

### 4. TypeScript Configuration

Recommended `tsconfig.json` for actions-mn actions:

```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2024"],
    "outDir": "./lib",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "verbatimModuleSyntax": true
  },
  "include": ["src/**/*"],
  "exclude": ["__tests__", "lib", "node_modules", "dist", "vitest.config.ts"]
}
```

### 5. Recommended Dependencies

```json
{
  "dependencies": {
    "@actions/core": "^3.0.0",
    "@actions/exec": "^3.0.0",
    "@actions/io": "^3.0.2"
  },
  "devDependencies": {
    "@types/node": "^24.x",
    "esbuild": "^0.27.0",
    "prettier": "^3.8.0",
    "typescript": "^5.9.0",
    "vitest": "^4.0.0"
  }
}
```

## Best Practices

### Versioning

- Use semantic versioning for your action
- Create major version tags (e.g., `v1`, `v2`, `v3`) that point to the latest minor/patch release
- Update the `uses:` reference in README examples when releasing new major versions

### README Badges

Include CI status badges in your README:

```markdown
[![Build and Test](https://github.com/actions-mn/YOUR-ACTION/actions/workflows/test.yml/badge.svg)](https://github.com/actions-mn/YOUR-ACTION/actions/workflows/test.yml)
```

### Committing dist/

For GitHub Actions, the `dist/index.cjs` file must be committed to the repository. Always run `npm run build` before committing changes to ensure `dist/` is up to date.

## Templates

### .gitignore

A canonical `.gitignore` template is provided at `templates/gitignore`. All actions-mn repositories should use this template to ensure consistency.

**To update your .gitignore:**
```bash
cp /path/to/shared/templates/gitignore /path/to/your-action/.gitignore
```

**What it covers:**
- Node.js (node_modules, lock files)
- Build outputs (lib/, *.tsbuildinfo) - NOTE: dist/ is NOT ignored since GitHub Actions require committed dist/
- IDE/Editors (.vscode, .idea, vim swap files)
- OS files (.DS_Store, Thumbs.db)
- Testing (coverage/, .nyc_output/)
- Temporary files (*.log, *.tmp, *.tgz)
- AI context files (CLAUDE.md, .claude/, .cursor/)

## Adding New Shared Workflows

When adding a new reusable workflow:

1. Create the workflow file in `.github/workflows/`
2. Use `on: workflow_call` to make it reusable
3. Document all inputs and secrets
4. Update this README with usage examples
5. Test the workflow from a consumer repository

## Related Repositories

- [actions-mn/setup](https://github.com/actions-mn/setup) - Set up Metanorma toolchain
- [actions-mn/site-gen](https://github.com/actions-mn/site-gen) - Generate Metanorma sites

## License

MIT
