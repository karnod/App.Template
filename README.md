# App.Template - Reusable GitHub Actions Workflows

This repository contains a collection of reusable GitHub Actions workflows that can be called from other repositories. These workflows provide standardized CI/CD pipelines for various technologies including Docker, .NET, Helm, Terraform, and Ionic.

## 📋 Available Workflows

- **docker-publish.yml** - Build and publish Docker images to container registries
- **docker-publish-self.yml** - Build and publish Docker images using self-hosted runners
- **dotnet-test.yml** - Run .NET tests with result artifacts
- **dotnet-publish.yml** - Build and publish .NET applications
- **helm-bootstrap.yml** - Bootstrap Kubernetes namespaces with Helm
- **helm-deploy.yml** - Deploy applications using Helm charts
- **helm-lint.yml** - Lint Helm charts for correctness
- **helm-publish.yml** - Package and publish Helm charts to a chart repository
- **ionic-build.yml** - Build Ionic/Cordova Android applications
- **terraform-deploy.yml** - Deploy infrastructure using Terraform

## 🔒 Using Workflows from a Private Repository

Since this repository is **private**, workflows in other repositories need authentication to reference these reusable workflows. Follow these steps to set up access:

### Step 1: Create a Personal Access Token (PAT)

1. Go to GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens (or Classic tokens)
2. Click "Generate new token"
3. Configure the token:
   - **Name**: "Workflow Templates Access" (or similar)
   - **Expiration**: Choose an appropriate duration
   - **Repository access**: Select "Only select repositories" and choose `karnod/App.Template`
   - **Permissions** (for Fine-grained tokens):
     - Repository permissions → Actions: Read
     - Repository permissions → Contents: Read
   - **Scopes** (for Classic tokens):
     - `repo` (Full control of private repositories)
4. Click "Generate token" and **copy the token immediately**

### Step 2: Add Token as a Repository Secret

In each repository that needs to use these workflows:

1. Go to Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Name: `WORKFLOW_TOKEN` (or `GH_PAT`)
4. Value: Paste your PAT
5. Click "Add secret"

### Step 3: Reference Workflows in Your Repository

When calling these reusable workflows, you need to check out the code using the PAT:

```yaml
name: My Workflow
on:
  push:
    branches: [main]

jobs:
  test:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./src/MyProject/MyProject.csproj
      DOTNET_VERSION: 6.0.x
    secrets: inherit
```

**Important**: GitHub automatically handles authentication for `uses:` statements when calling reusable workflows. The PAT is only needed if you need to explicitly checkout or access files from this private repository within your workflow steps.

## 📖 Usage Examples

### Docker Build and Publish

```yaml
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  build-docker:
    uses: karnod/App.Template/.github/workflows/docker-publish.yml@main
    with:
      DOCKER_TARGET_PLATFORM: linux/amd64
      DOCKERFILE: Dockerfile
      DOCKER_REGISTRY: ghcr.io
    secrets:
      DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
```

### .NET Test

```yaml
name: Run Tests
on:
  pull_request:
    branches: [main]

jobs:
  test:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./src/MyApp.Tests/MyApp.Tests.csproj
      DOTNET_VERSION: 8.0.x
```

### Helm Deploy

```yaml
name: Deploy to Kubernetes
on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: my-app
      NAMESPACE: production
      CHARTNAME: my-app
      ENVIRONMENT: production
    secrets:
      CLUSTER_CONFIG: ${{ secrets.CLUSTER_CONFIG }}
      HELM_GIT_PASSWORD: ${{ secrets.HELM_GIT_PASSWORD }}
```

For detailed documentation on each workflow's inputs and secrets, see [USAGE.md](USAGE.md).

## 🔐 Security Best Practices

### PAT Management

1. **Use Fine-grained tokens** when possible for better security
2. **Minimum permissions**: Only grant read access to Contents and Actions
3. **Scope to specific repositories**: Don't use organization-wide tokens
4. **Set expiration**: Use 90-day or shorter expiration periods
5. **Rotate regularly**: Update tokens before expiration
6. **Use organization secrets**: For workflows across multiple repositories, use organization-level secrets

### Secret Rotation

When rotating PATs:

1. Create a new PAT with the same permissions
2. Update the secret in all consuming repositories
3. Wait 24-48 hours for propagation
4. Delete the old PAT

## 🛠️ Alternative: GitHub App (Enterprise)

For enterprise environments, consider using a GitHub App instead of PATs:

1. Create a GitHub App with necessary permissions
2. Install the app in your organization
3. Use the app's credentials in workflows
4. Benefits: Better audit trail, fine-grained permissions, no expiration

## 📝 Workflow Authentication Flow

When a workflow in Repository A calls a reusable workflow in this private repository:

```
Repository A Workflow
    ↓
GitHub Actions checks caller's permissions
    ↓
Verifies access to karnod/App.Template
    ↓
Uses GITHUB_TOKEN (automatic) or provided PAT
    ↓
Executes reusable workflow
```

**Note**: As of GitHub Actions' current implementation, the `GITHUB_TOKEN` in the calling workflow must have permission to access the private reusable workflow repository. This typically requires:
- The calling repository and this repository are in the same organization, OR
- A PAT with appropriate access is configured

## ❓ Troubleshooting

### Error: "Repository not found"
- **Cause**: Insufficient permissions to access this private repository
- **Solution**: Verify PAT has correct permissions and is not expired

### Error: "Workflow does not exist"
- **Cause**: Incorrect path or branch reference
- **Solution**: Ensure you're using the correct path format: `karnod/App.Template/.github/workflows/WORKFLOW.yml@REF`

### Error: "Invalid workflow file"
- **Cause**: Missing required inputs or secrets
- **Solution**: Check USAGE.md for required parameters

### Workflows not triggering
- **Cause**: The `workflow_call` trigger only responds to workflow calls, not direct events
- **Solution**: These workflows must be called from another workflow using `uses:`

## 📚 Additional Resources

- [GitHub Docs: Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub Docs: Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [GitHub Docs: Automatic token authentication](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)

## 🤝 Contributing

To add or modify workflows:

1. Ensure workflows use `workflow_call` trigger
2. Document all inputs and secrets clearly
3. Provide usage examples
4. Test with a consuming repository before merging

## 📄 License

Internal use only - karnod organization
