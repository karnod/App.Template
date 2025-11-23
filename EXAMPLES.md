# Workflow Examples

This document provides real-world examples of using the reusable workflows in common scenarios.

## Table of Contents

- [Basic Examples](#basic-examples)
- [Multi-Stage Pipelines](#multi-stage-pipelines)
- [Environment-Based Deployments](#environment-based-deployments)
- [Matrix Builds](#matrix-builds)
- [Conditional Workflows](#conditional-workflows)
- [Monorepo Patterns](#monorepo-patterns)

---

## Basic Examples

### Simple .NET Application CI

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main, develop]

jobs:
  test:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./MyApp.Tests/MyApp.Tests.csproj
      DOTNET_VERSION: 8.0.x
```

### Docker Image Build on Tag

```yaml
name: Release
on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64,linux/arm64
      DOCKER_REGISTRY: ghcr.io
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
```

### Helm Chart Validation

```yaml
name: Helm Lint
on:
  pull_request:
    paths:
      - 'helm/**'

jobs:
  lint:
    uses: karnod/App.Template/.github/workflows/helm-lint.yml@main
```

---

## Multi-Stage Pipelines

### Complete CI/CD Pipeline

```yaml
name: CI/CD
on:
  push:
    branches: [main]

jobs:
  # Stage 1: Test
  test:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./src/MyApp.Tests/MyApp.Tests.csproj
      DOTNET_VERSION: 8.0.x

  # Stage 2: Build application
  publish:
    needs: test
    uses: karnod/App.Template/.github/workflows/dotnet-publish.yml@main
    with:
      PROJECT_PATH: ./src/MyApp/MyApp.csproj
      DOTNET_VERSION: 8.0.x

  # Stage 3: Build Docker image
  docker:
    needs: publish
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

  # Stage 4: Deploy to staging
  deploy-staging:
    needs: docker
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp-staging
      NAMESPACE: staging
      CHARTNAME: myapp
      ENVIRONMENT: staging
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG_STAGING }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}

  # Stage 5: Deploy to production (manual approval)
  deploy-production:
    needs: deploy-staging
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp-prod
      NAMESPACE: production
      CHARTNAME: myapp
      ENVIRONMENT: production
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG_PROD }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}
```

### Microservices Build Pipeline

```yaml
name: Microservices Build
on:
  push:
    branches: [main]

jobs:
  # Build API service
  api:
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
      DOCKERFILE: services/api/Dockerfile
      DOCKER_IMAGE: ${{ github.repository }}/api
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

  # Build Worker service
  worker:
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
      DOCKERFILE: services/worker/Dockerfile
      DOCKER_IMAGE: ${{ github.repository }}/worker
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

  # Build Web frontend
  web:
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
      DOCKERFILE: services/web/Dockerfile
      DOCKER_IMAGE: ${{ github.repository }}/web
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
```

---

## Environment-Based Deployments

### Deploy to Multiple Environments

```yaml
name: Deploy
on:
  push:
    branches:
      - main
      - staging
      - develop

jobs:
  # Determine environment based on branch
  deploy:
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp-${{ github.ref_name }}
      NAMESPACE: ${{ github.ref_name == 'main' && 'production' || github.ref_name }}
      CHARTNAME: myapp
      ENVIRONMENT: ${{ github.ref_name == 'main' && 'production' || github.ref_name }}
    secrets:
      CLUSTER_CONFIG: ${{ secrets[format('K8S_CONFIG_{0}', github.ref_name == 'main' && 'PROD' || upper(github.ref_name))] }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}
```

### Progressive Deployment

```yaml
name: Progressive Deployment
on:
  push:
    tags:
      - 'v*'

jobs:
  # Deploy to dev first
  deploy-dev:
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp-dev
      NAMESPACE: development
      CHARTNAME: myapp
      ENVIRONMENT: development
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG_DEV }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}

  # Wait and deploy to staging
  deploy-staging:
    needs: deploy-dev
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp-staging
      NAMESPACE: staging
      CHARTNAME: myapp
      ENVIRONMENT: staging
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG_STAGING }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}

  # Manual approval for production
  approve-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production-approval
    steps:
      - name: Approval step
        run: echo "Approved for production deployment"

  # Deploy to production
  deploy-production:
    needs: approve-production
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp-prod
      NAMESPACE: production
      CHARTNAME: myapp
      ENVIRONMENT: production
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG_PROD }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}
```

---

## Matrix Builds

### Multi-Version .NET Testing

```yaml
name: Test Multiple Versions
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    strategy:
      matrix:
        dotnet-version: ['6.0.x', '7.0.x', '8.0.x']
        os: [ubuntu-latest, windows-latest]
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./MyApp.Tests/MyApp.Tests.csproj
      DOTNET_VERSION: ${{ matrix.dotnet-version }}
```

### Multi-Platform Docker Builds

```yaml
name: Multi-Platform Build
on:
  push:
    branches: [main]

jobs:
  build:
    strategy:
      matrix:
        platform:
          - linux/amd64
          - linux/arm64
          - linux/arm/v7
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: ${{ matrix.platform }}
      DOCKER_IMAGE: ${{ github.repository }}-${{ matrix.platform }}
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
```

### Multiple Helm Chart Deployments

```yaml
name: Deploy Multiple Apps
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options:
          - staging
          - production

jobs:
  deploy:
    strategy:
      matrix:
        app: [api, worker, frontend]
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: ${{ matrix.app }}-${{ inputs.environment }}
      NAMESPACE: ${{ inputs.environment }}
      CHARTNAME: ${{ matrix.app }}
      ENVIRONMENT: ${{ inputs.environment }}
    secrets:
      CLUSTER_CONFIG: ${{ secrets[format('K8S_CONFIG_{0}', upper(inputs.environment))] }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}
```

---

## Conditional Workflows

### Deploy Only on Main Branch

```yaml
name: Build and Deploy
on:
  push:
    branches: [main, develop]

jobs:
  build:
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp-prod
      NAMESPACE: production
      CHARTNAME: myapp
      ENVIRONMENT: production
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG_PROD }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}
```

### Path-Based Triggering

```yaml
name: Selective Build
on:
  push:
    paths:
      - 'src/**'
      - 'Dockerfile'
      - '.github/workflows/**'

jobs:
  test:
    if: contains(github.event.head_commit.modified, 'src/')
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./src/Tests/Tests.csproj
      DOTNET_VERSION: 8.0.x

  docker:
    if: contains(github.event.head_commit.modified, 'Dockerfile')
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
```

### Time-Based Deployments

```yaml
name: Scheduled Deployment
on:
  schedule:
    - cron: '0 2 * * 1'  # Every Monday at 2 AM UTC

jobs:
  deploy:
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp-prod
      NAMESPACE: production
      CHARTNAME: myapp
      ENVIRONMENT: production
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG_PROD }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_REPO_TOKEN }}
```

---

## Monorepo Patterns

### Build Multiple Projects

```yaml
name: Monorepo Build
on:
  push:
    branches: [main]

jobs:
  # Test all projects
  test-api:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./projects/api/Tests/Tests.csproj
      DOTNET_VERSION: 8.0.x

  test-worker:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./projects/worker/Tests/Tests.csproj
      DOTNET_VERSION: 8.0.x

  # Build Docker images
  build-api:
    needs: test-api
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
      DOCKERFILE: projects/api/Dockerfile
      DOCKER_IMAGE: ${{ github.repository }}/api
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

  build-worker:
    needs: test-worker
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
      DOCKERFILE: projects/worker/Dockerfile
      DOCKER_IMAGE: ${{ github.repository }}/worker
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
```

### Path-Based Matrix Build

```yaml
name: Monorepo Smart Build
on:
  push:
    branches: [main]

jobs:
  # Detect changed projects
  changes:
    runs-on: ubuntu-latest
    outputs:
      projects: ${{ steps.filter.outputs.projects }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            api:
              - 'projects/api/**'
            worker:
              - 'projects/worker/**'
            frontend:
              - 'projects/frontend/**'

  # Build only changed projects
  build:
    needs: changes
    if: needs.changes.outputs.projects != '[]'
    strategy:
      matrix:
        project: ${{ fromJson(needs.changes.outputs.projects) }}
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
      DOCKERFILE: projects/${{ matrix.project }}/Dockerfile
      DOCKER_IMAGE: ${{ github.repository }}/${{ matrix.project }}
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
```

---

## Advanced Patterns

### Parallel Testing with Aggregation

```yaml
name: Comprehensive Testing
on:
  pull_request:

jobs:
  # Run tests in parallel
  unit-tests:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./tests/Unit/Unit.csproj
      DOTNET_VERSION: 8.0.x

  integration-tests:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./tests/Integration/Integration.csproj
      DOTNET_VERSION: 8.0.x

  # Aggregate results
  test-summary:
    needs: [unit-tests, integration-tests]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Check test results
        run: |
          if [ "${{ needs.unit-tests.result }}" != "success" ] || \
             [ "${{ needs.integration-tests.result }}" != "success" ]; then
            echo "Tests failed!"
            exit 1
          fi
          echo "All tests passed!"
```

### Reusable Workflow with Custom Post-Processing

```yaml
name: Build with Notifications
on:
  push:
    branches: [main]

jobs:
  build:
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

  notify:
    needs: build
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Notify on success
        if: needs.build.result == 'success'
        run: |
          echo "Build successful! Sending notification..."
          # Send notification to Slack, Teams, etc.

      - name: Notify on failure
        if: needs.build.result == 'failure'
        run: |
          echo "Build failed! Alerting team..."
          # Send alert
```

---

## Tips for Complex Scenarios

1. **Use `needs:` for dependencies** between jobs
2. **Use `if:` conditions** to control job execution
3. **Use `strategy.matrix`** for parallel builds
4. **Use `outputs`** to pass data between jobs
5. **Use `secrets: inherit`** to pass all secrets automatically
6. **Pin workflow versions** with `@v1.0.0` or commit SHA for stability
7. **Test in feature branches** before merging to main

For more examples, see the [GitHub Actions documentation](https://docs.github.com/en/actions/examples).
