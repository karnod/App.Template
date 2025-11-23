# Workflow Usage Guide

This document provides detailed information about each reusable workflow, including all inputs, secrets, and usage examples.

## Table of Contents

- [docker-publish.yml](#docker-publishyml)
- [docker-publish-self.yml](#docker-publish-selfyml)
- [dotnet-test.yml](#dotnet-testyml)
- [dotnet-publish.yml](#dotnet-publishyml)
- [helm-bootstrap.yml](#helm-bootstrapyml)
- [helm-deploy.yml](#helm-deployyml)
- [helm-lint.yml](#helm-lintyml)
- [helm-publish.yml](#helm-publishyml)
- [ionic-build.yml](#ionic-buildyml)
- [terraform-deploy.yml](#terraform-deployyml)

---

## docker-publish.yml

Build and publish Docker images to container registries with multi-platform support.

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `DOCKER_TARGET_PLATFORM` | string | Yes | - | Target platform architecture (e.g., `linux/amd64`, `linux/arm64`, or `linux/amd64,linux/arm64`) |
| `DOCKERFILE` | string | No | `Dockerfile` | Path to Dockerfile relative to repository root |
| `DOCKER_REGISTRY` | string | No | `ghcr.io` | Docker registry URL |
| `DOCKER_IMAGE` | string | No | `${{ github.repository }}` | Docker image name |
| `DOCKER_USERNAME` | string | No | `${{ github.actor }}` | Docker registry username |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `DOCKER_PASSWORD` | No | Docker registry password (defaults to `GITHUB_TOKEN`) |

### Usage Example

```yaml
jobs:
  build:
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64,linux/arm64
      DOCKERFILE: docker/prod.Dockerfile
      DOCKER_REGISTRY: ghcr.io
      DOCKER_IMAGE: ${{ github.repository }}
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
```

### Notes

- Runs on GitHub-hosted `ubuntu-latest` runners
- Uses Docker Buildx for multi-platform builds
- Automatically tags images with `:latest` and `:${GITHUB_RUN_NUMBER}`
- Images are automatically pushed to the registry

---

## docker-publish-self.yml

Build and publish Docker images using self-hosted runners (without Buildx).

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `DOCKER_TARGET_PLATFORM` | string | Yes | - | Target platform architecture |
| `DOCKERFILE` | string | No | `Dockerfile` | Path to Dockerfile relative to repository root |
| `DOCKER_REGISTRY` | string | No | `ghcr.io` | Docker registry URL |
| `DOCKER_IMAGE` | string | No | `${{ github.repository }}` | Docker image name |
| `DOCKER_USERNAME` | string | No | `${{ github.actor }}` | Docker registry username |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `DOCKER_PASSWORD` | No | Docker registry password (defaults to `GITHUB_TOKEN`) |

### Usage Example

```yaml
jobs:
  build:
    uses: karnod/App.Template/.github/workflows/docker-publish-self.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/arm64
      DOCKERFILE: Dockerfile.arm
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GHCR_TOKEN }}
```

### Notes

- Runs on self-hosted runners
- Uses standard Docker build (no Buildx)
- Suitable for ARM-based self-hosted runners

---

## dotnet-test.yml

Run .NET tests and upload results as artifacts.

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `PROJECT_PATH` | string | Yes | - | Path to .NET project or solution file |
| `DOTNET_VERSION` | string | No | `6.1.x` | .NET SDK version to use |

### Secrets

None required.

### Usage Example

```yaml
jobs:
  test:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./src/MyApp.Tests/MyApp.Tests.csproj
      DOTNET_VERSION: 8.0.x
```

### Notes

- Runs on `ubuntu-latest`
- Automatically restores dependencies
- Uploads test results as artifacts named `dotnet-results-{version}`
- Test results are in TRX format
- Artifacts are uploaded even if tests fail

---

## dotnet-publish.yml

Build and publish .NET applications as deployment packages.

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `PROJECT_PATH` | string | Yes | - | Path to .NET project file |
| `DOTNET_VERSION` | string | No | `3.1.x` | .NET SDK version to use |

### Secrets

None required.

### Usage Example

```yaml
jobs:
  publish:
    uses: karnod/App.Template/.github/workflows/dotnet-publish.yml@main
    with:
      PROJECT_PATH: ./src/MyApp/MyApp.csproj
      DOTNET_VERSION: 6.0.x
```

### Notes

- Runs on `ubuntu-latest`
- Publishes in Release configuration
- Creates a ZIP file of the published output
- Uploads artifact named `deploy-zip`
- Output is suitable for deployment to Azure, AWS, or other cloud platforms

---

## helm-bootstrap.yml

Bootstrap Kubernetes namespaces with configuration from bootstrap directory.

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `NAMESPACE` | string | Yes | - | Kubernetes namespace to bootstrap |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `CLUSTER_CONFIG` | Yes | Base64-encoded Kubernetes config file |
| `BASE64DOCKERAUTH` | Yes | Base64-encoded Docker registry authentication |

### Usage Example

```yaml
jobs:
  bootstrap:
    uses: karnod/App.Template/.github/workflows/helm-bootstrap.yml@main
    with:
      NAMESPACE: production
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG }}
      BASE64DOCKERAUTH: ${{ secrets.DOCKER_AUTH }}
```

### Notes

- Requires a `bootstrap/` directory in your repository with YAML manifests
- Automatically replaces `#NAMESPACE#` placeholder in all bootstrap files
- Replaces `#BASE64DOCKERAUTH#` in `imagepull.yml` if present
- Applies `namespace.yml` first if it exists
- Then applies all other YAML files in the bootstrap directory

### Bootstrap Directory Structure

```
bootstrap/
├── namespace.yml          # Created first (contains namespace definition)
├── imagepull.yml         # Docker registry secret (uses BASE64DOCKERAUTH)
├── configmap.yml         # ConfigMaps
└── secret.yml            # Secrets
```

---

## helm-deploy.yml

Deploy applications to Kubernetes using Helm charts.

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `RELEASE_NAME` | string | Yes | - | Helm release name |
| `NAMESPACE` | string | Yes | - | Kubernetes namespace |
| `CHARTNAME` | string | Yes | - | Name of the Helm chart to deploy |
| `ENVIRONMENT` | string | Yes | - | Environment name (used for values file) |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `CLUSTER_CONFIG` | Yes | Base64-encoded Kubernetes config file |
| `HELM_GIT_PASSWORD` | Yes | GitHub PAT for accessing private Helm chart repository |

### Usage Example

```yaml
jobs:
  deploy:
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp-prod
      NAMESPACE: production
      CHARTNAME: myapp
      ENVIRONMENT: production
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}
```

### Notes

- Uses Helm chart from `{org}/helm-charts` repository
- Expects values file at `environment/{ENVIRONMENT}/values.yaml`
- Automatically performs `helm install` or `helm upgrade` based on existing deployments
- Chart name has dots removed (e.g., `my.app` becomes `myapp`)

### Directory Structure Required

```
environment/
├── production/
│   └── values.yaml
├── staging/
│   └── values.yaml
└── development/
    └── values.yaml
```

---

## helm-lint.yml

Lint Helm charts for correctness and best practices.

### Inputs

None required.

### Secrets

None required.

### Usage Example

```yaml
jobs:
  lint:
    uses: karnod/App.Template/.github/workflows/helm-lint.yml@main
```

### Notes

- Expects Helm chart in `helm/` directory
- Runs `helm lint` to validate chart structure
- Fails the workflow if linting errors are found

---

## helm-publish.yml

Package and publish Helm charts to a chart repository.

### Inputs

None required (uses repository name as chart name).

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `HELM_GIT_USERNAME` | Yes | Git username for Helm chart repository |
| `HELM_GIT_EMAIL` | Yes | Git email for commits to Helm chart repository |
| `HELM_GIT_PASSWORD` | Yes | GitHub PAT for pushing to Helm chart repository |

### Usage Example

```yaml
jobs:
  publish:
    uses: karnod/App.Template/.github/workflows/helm-publish.yml@main
    secrets:
      HELM_GIT_USERNAME: ${{ secrets.GIT_USERNAME }}
      HELM_GIT_EMAIL: ${{ secrets.GIT_EMAIL }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}
```

### Notes

- Expects Helm chart in `helm/` directory
- Publishes to `{org}/helm-charts` repository
- Automatically updates `index.yaml`
- Creates git commit with timestamp
- Chart name is derived from repository name (dots removed)

---

## ionic-build.yml

Build Ionic/Cordova Android applications.

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `CONTAINER_IMAGE` | string | Yes | - | Docker image with Ionic/Cordova build tools |

### Secrets

None required.

### Usage Example

```yaml
jobs:
  build:
    uses: karnod/App.Template/.github/workflows/ionic-build.yml@main
    with:
      CONTAINER_IMAGE: beevelop/ionic:latest
```

### Notes

- Runs in specified container image
- Executes: `npm install`, `ionic cordova prepare android`, `ionic cordova build android`
- Uploads debug APK as artifact named `deploy-apk`
- APK location: `platforms/android/app/build/outputs/apk/debug/app-debug.apk`

---

## terraform-deploy.yml

Deploy infrastructure using Terraform.

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `ENVIRONMENT` | string | Yes | - | Environment name (used as GitHub environment) |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `ARM_CLIENT_ID` | Yes | Azure Service Principal client ID |
| `ARM_CLIENT_SECRET` | Yes | Azure Service Principal client secret |
| `ARM_SUBSCRIPTION_ID` | Yes | Azure subscription ID |
| `ARM_TENANT_ID` | Yes | Azure tenant ID |

### Usage Example

```yaml
jobs:
  deploy:
    uses: karnod/App.Template/.github/workflows/terraform-deploy.yml@main
    with:
      ENVIRONMENT: production
    secrets:
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
```

### Notes

- Expects Terraform code in `./terraform` directory
- Runs: format, init, validate, plan
- Apply/destroy steps are conditional (currently commented in workflow)
- Uses Terraform version 0.14.8
- Configured for Azure provider

### Directory Structure Required

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
└── provider.tf
```

---

## Common Patterns

### Chaining Multiple Workflows

```yaml
name: Full CI/CD Pipeline
on:
  push:
    branches: [main]

jobs:
  test:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./src/MyApp.Tests/MyApp.Tests.csproj
      DOTNET_VERSION: 8.0.x
  
  build:
    needs: test
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
  
  deploy:
    needs: build
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp
      NAMESPACE: production
      CHARTNAME: myapp
      ENVIRONMENT: production
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_TOKEN }}
```

### Using Different Branches/Tags

```yaml
jobs:
  # Use main branch (latest)
  test-latest:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./test.csproj
  
  # Use specific tag
  test-stable:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@v1.0.0
    with:
      PROJECT_PATH: ./test.csproj
  
  # Use specific commit
  test-pinned:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@abc1234
    with:
      PROJECT_PATH: ./test.csproj
```

### Matrix Strategy with Reusable Workflows

```yaml
jobs:
  test:
    strategy:
      matrix:
        dotnet-version: [6.0.x, 7.0.x, 8.0.x]
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./src/MyApp.Tests/MyApp.Tests.csproj
      DOTNET_VERSION: ${{ matrix.dotnet-version }}
```

## Tips and Best Practices

1. **Pin to specific versions** in production workflows using tags or commit SHAs
2. **Use `secrets: inherit`** to pass all secrets automatically when appropriate
3. **Test workflow changes** in a branch before merging to main
4. **Document required secrets** in your repository's README
5. **Use GitHub Environments** for deployment workflows with protection rules
6. **Monitor workflow runs** for deprecation warnings from GitHub Actions
7. **Keep workflows updated** - check for newer action versions periodically
