# Determine Deploy Directory

Intelligently determines the correct deployment directory based on repository structure and configuration.

## Features

- 🔍 **Auto-detection** - Finds deployment directories
- 📁 **Multi-structure support** - Monorepo and single-repo
- 🎯 **Path resolution** - Handles various path patterns
- ✅ **Validation** - Ensures directories exist
- 🔄 **Fallback logic** - Smart defaults

## Usage

```yaml
- name: Determine deployment directory
  uses: skyhook-io/determine-deploy-dir@v1
  id: deploy-dir
  with:
    environment: production
    service_dir: services/backend
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `environment` | Environment/overlay name | ✅ | - |
| `service_dir` | Service directory | ❌ | `.` |
| `deployment_repo` | Deployment repository (if different from current) | ❌ | - |
| `deployment_path` | Path in deployment repository | ❌ | - |
| `current_repo` | Current repository | ❌ | `${{ github.repository }}` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `deploy_dir` | Full path to deployment overlay directory | `services/api/deploy/overlays/prod` |
| `is_custom_path` | Whether using custom deployment path | `false` |
| `base_dir` | Base directory before overlay path | `services/api` |

## Directory Patterns

The action constructs the deployment directory path based on inputs:

1. If `deployment_path` is provided and different from service_dir: `{deployment_path}/deploy/overlays/{environment}`
2. If `service_dir` is provided and not '.': `{service_dir}/deploy/overlays/{environment}`
3. Default: `deploy/overlays/{environment}`

When `deployment_repo` differs from `current_repo`, the action sets `is_custom_path` to true.

## Examples

### Monorepo with Explicit Path
```yaml
- name: Find deploy directory
  uses: skyhook-io/determine-deploy-dir@v1
  id: dir
  with:
    environment: production
    service_dir: services/backend
    
- name: Use directory
  run: |
    echo "Deploying from: ${{ steps.dir.outputs.deploy_dir }}"
    kustomize build ${{ steps.dir.outputs.deploy_dir }}
```

### Single Service Repo
```yaml
- name: Find deploy directory
  uses: skyhook-io/determine-deploy-dir@v1
  id: dir
  with:
    environment: staging
    # service_dir defaults to '.', will find deploy/overlays/staging
```

### With Service Directory
```yaml
- name: Find deploy directory
  uses: skyhook-io/determine-deploy-dir@v1
  id: dir
  with:
    environment: ${{ inputs.env }}
    service_dir: services/${{ matrix.service }}
```

### With Validation
```yaml
- name: Find deploy directory
  uses: skyhook-io/determine-deploy-dir@v1
  id: dir
  with:
    environment: production
    deployment_path: ${{ inputs.path }}

- name: Check if custom path
  if: steps.dir.outputs.is_custom_path == 'true'
  run: |
    echo "Using custom deployment path"
```

## Complete Workflow Example

```yaml
jobs:
  deploy:
    steps:
      - uses: actions/checkout@v4
      
      - name: Determine deployment directory
        uses: skyhook-io/determine-deploy-dir@v1
        id: deploy-dir
        with:
          environment: ${{ inputs.environment }}
          service_dir: ${{ inputs.service_dir }}
          deployment_path: ${{ inputs.deployment_path }}
      
      - name: Update manifests
        uses: skyhook-io/kustomize-edit@v1
        with:
          overlay_dir: ${{ steps.deploy-dir.outputs.deploy_dir }}
          image: ${{ inputs.service }}
          tag: ${{ inputs.tag }}
      
      - name: Deploy
        uses: skyhook-io/kustomize-apply@v1
        with:
          overlay_dir: ${{ steps.deploy-dir.outputs.deploy_dir }}
```

## Directory Structure Examples

### Monorepo
```
repo/
├── services/
│   ├── api/
│   │   └── deploy/
│   │       └── overlays/
│   │           ├── dev/
│   │           └── prod/
│   └── web/
│       └── deploy/
│           └── overlays/
│               ├── dev/
│               └── prod/
```

### Single Service
```
repo/
├── src/
├── deploy/
│   ├── base/
│   └── overlays/
│       ├── staging/
│       └── production/
```

### Alternative Structure
```
repo/
├── src/
├── k8s/
│   ├── base/
│   └── overlays/
│       ├── dev/
│       └── prod/
```

## Notes

- Searches multiple common patterns
- Validates kustomization.yaml exists
- Provides clear error messages
- Handles nested monorepo structures
- Works with various naming conventions