# Quick Start Guide

Get started with private reusable workflows in 5 minutes.

## Prerequisites

- Access to the `karnod/App.Template` repository
- A repository where you want to use these workflows
- GitHub account with appropriate permissions

## Step 1: Create a Personal Access Token (2 minutes)

1. Go to: https://github.com/settings/tokens?type=beta
2. Click **"Generate new token"**
3. Fill in the details:
   - **Token name**: `Workflow-Templates-Access`
   - **Expiration**: `90 days`
   - **Repository access**: "Only select repositories"
   - Select: `karnod/App.Template`
4. Under **Permissions**, set:
   - Repository permissions → **Actions**: `Read-only`
   - Repository permissions → **Contents**: `Read-only`
5. Click **"Generate token"**
6. **Copy the token** (you won't see it again!)

## Step 2: Add Token to Your Repository (1 minute)

1. Go to your repository: `https://github.com/YOUR-ORG/YOUR-REPO`
2. Navigate to: **Settings** → **Secrets and variables** → **Actions**
3. Click **"New repository secret"**
4. Set:
   - **Name**: `WORKFLOW_TOKEN`
   - **Value**: Paste the token from Step 1
5. Click **"Add secret"**

## Step 3: Use a Workflow (2 minutes)

Create `.github/workflows/ci.yml` in your repository:

### Example 1: .NET Test

```yaml
name: Test
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./src/MyApp.Tests/MyApp.Tests.csproj
      DOTNET_VERSION: 8.0.x
```

### Example 2: Docker Build

```yaml
name: Build
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
```

### Example 3: Helm Deploy

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: karnod/App.Template/.github/workflows/helm-deploy.yml@main
    with:
      RELEASE_NAME: myapp
      NAMESPACE: production
      CHARTNAME: myapp
      ENVIRONMENT: production
    secrets:
      CLUSTER_CONFIG: ${{ secrets.K8S_CONFIG }}
      HELM_GIT_PASSWORD: ${{ secrets.WORKFLOW_TOKEN }}
```

## Step 4: Test Your Workflow

1. Commit and push your workflow file
2. GitHub Actions will automatically trigger
3. Go to the **Actions** tab to view the results

## Common Issues

### ❌ "Repository not found" error

**Solution**: Your PAT doesn't have access to the template repository.
- Verify the token has access to `karnod/App.Template`
- Check token hasn't expired
- Ensure token permissions include Actions and Contents read

### ❌ Workflow doesn't trigger

**Solution**: Reusable workflows only respond to `workflow_call`, not direct triggers.
- Don't use `on: push` in the reusable workflow itself
- Call it from another workflow using `uses:`

### ❌ Missing required inputs

**Solution**: Check the workflow requirements in [USAGE.md](USAGE.md)
- Each workflow has required `inputs` and `secrets`
- Ensure all required parameters are provided

## Next Steps

- 📖 Read the full [README.md](README.md) for comprehensive documentation
- 📚 Check [USAGE.md](USAGE.md) for detailed workflow parameters
- 🔐 Review [SECURITY.md](SECURITY.md) for security best practices
- 🎯 See workflow examples in [EXAMPLES.md](EXAMPLES.md)

## Need Help?

1. Check the [troubleshooting section](README.md#-troubleshooting) in README
2. Review workflow logs in the Actions tab
3. Consult [GitHub's reusable workflows documentation](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

## Token Expiration Reminder

⏰ **Your token will expire in 90 days!**

Set a calendar reminder to rotate your token before expiration:
1. Create a new token (same process as Step 1)
2. Update the `WORKFLOW_TOKEN` secret (same as Step 2)
3. Delete the old token from GitHub settings

---

**Ready to use reusable workflows from this private repository!** 🚀
