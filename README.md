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

## Setting Up a New actions-mn Action

### 1. Package Structure

Your action should have the following structure:

```
my-action/
├── .github/
│   └── workflows/
│       ├── test.yml
│       └── check-dist.yml
├── src/
│   └── main.ts
├── dist/
│   └── index.js
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
    "build": "tsc --noEmit && esbuild src/main.ts --bundle --platform=node --target=node20 --format=cjs --outfile=dist/index.js --minify --sourcemap",
    "format": "prettier --write '**/*.ts'",
    "format-check": "prettier --check '**/*.ts'",
    "test": "vitest"
  }
}
```

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
    "esbuild": "^0.25.0",
    "prettier": "^3.8.0",
    "typescript": "^5.8.0",
    "vitest": "^3.2.0"
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

For GitHub Actions, the `dist/index.js` file must be committed to the repository. Always run `npm run build` before committing changes to ensure `dist/` is up to date.

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
