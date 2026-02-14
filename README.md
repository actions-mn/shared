# Shared Workflows for actions-mn

This repository contains reusable GitHub Actions workflows shared across all actions-mn actions.

## Available Workflows

### lint-test.yml

Reusable workflow for linting and testing Node.js-based GitHub Actions.

**Usage:**
```yaml
jobs:
  lint-test:
    uses: actions-mn/shared/.github/workflows/lint-test.yml@main
```

**Inputs:**
| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `node-version` | Node.js version to use | No | `24.x` |
| `run-lint` | Whether to run lint command | No | `false` |

### check-dist.yml

Reusable workflow for verifying the `dist/` directory is up to date.

**Usage:**
```yaml
jobs:
  check-dist:
    uses: actions-mn/shared/.github/workflows/check-dist.yml@main
```

**Inputs:**
| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `node-version` | Node.js version to use | No | `24.x` |

## Requirements

Actions using these workflows should have the following npm scripts defined:
- `npm run build` - Build the action
- `npm run format-check` - Check code formatting
- `npm test` - Run unit tests
- `npm run lint` - (Optional) Run linting

## License

MIT
