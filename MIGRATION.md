# Migration Guide: Public to Private Repository

This guide helps existing users migrate their workflows when this repository transitions from public to private.

## Overview

When this repository becomes private, workflows referencing it from other repositories will need authentication to continue working. This guide provides step-by-step instructions for a smooth transition.

## Pre-Migration Checklist

Before making the repository private:

- [ ] Identify all repositories using these workflows
- [ ] Create and distribute access tokens
- [ ] Update repository secrets
- [ ] Test workflows in a staging environment
- [ ] Notify all users of the migration timeline

## Migration Timeline (Recommended)

### Week 1: Preparation
- Create documentation (this is complete!)
- Identify affected repositories
- Create GitHub PATs for each team/repository

### Week 2: Communication
- Notify all users via email/Slack
- Share migration guide
- Offer support channels

### Week 3: Token Distribution
- Add secrets to all consuming repositories
- Verify workflows still function (while repo is public)

### Week 4: Go Private
- Make repository private
- Monitor for issues
- Provide support for any problems

## Step-by-Step Migration

### For Repository Owners

#### 1. Audit Current Usage

Find all repositories using these workflows:

```bash
# Search GitHub for references (requires gh CLI)
gh api search/code \
  -f q='org:karnod uses: karnod/App.Template' \
  --paginate | jq -r '.items[].repository.full_name' | sort -u
```

Or manually search: `org:karnod uses: karnod/App.Template`

#### 2. Create Access Token

Follow the instructions in [QUICKSTART.md](QUICKSTART.md#step-1-create-a-personal-access-token-2-minutes) to create a PAT.

**Options:**
- **One token per team**: Simpler management
- **One token per repository**: Better security, more management overhead
- **Organization secret**: Easiest for organization-wide access

#### 3. Distribute Tokens

**Option A: Organization Secret (Recommended)**
```bash
# Using gh CLI
gh secret set WORKFLOW_TOKEN \
  --org karnod \
  --visibility all \
  --body "ghp_your_token_here"
```

**Option B: Individual Repository Secrets**
```bash
# For each repository
gh secret set WORKFLOW_TOKEN \
  --repo karnod/example-repo \
  --body "ghp_your_token_here"
```

#### 4. Update Documentation

Add a note to consuming repositories:
```markdown
## GitHub Actions Authentication

This repository uses reusable workflows from a private template repository.
The required `WORKFLOW_TOKEN` secret has been configured for you.

If you encounter authentication issues, contact DevOps team.
```

#### 5. Make Repository Private

1. Go to repository Settings
2. Scroll to "Danger Zone"
3. Click "Change visibility"
4. Select "Make private"
5. Confirm

#### 6. Monitor and Support

- Watch for failed workflows
- Monitor GitHub Actions notifications
- Be available for support requests

### For Workflow Consumers

#### 1. Verify You Have Access

Check if your PAT is configured:
```bash
# Using gh CLI
gh secret list --repo karnod/your-repo | grep WORKFLOW_TOKEN
```

If not present, contact your repository admin.

#### 2. Test Your Workflows

Before the repository goes private, verify everything works:

```bash
# Trigger a workflow manually
gh workflow run ci.yml --repo karnod/your-repo
```

#### 3. Update Workflow Files (if needed)

Most workflows won't need changes, but verify your syntax:

**Before (works for both public and private):**
```yaml
jobs:
  test:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./test.csproj
```

**After (same - no changes needed!):**
```yaml
jobs:
  test:
    uses: karnod/App.Template/.github/workflows/dotnet-test.yml@main
    with:
      PROJECT_PATH: ./test.csproj
```

The `WORKFLOW_TOKEN` is used automatically by GitHub Actions when accessing private repositories.

#### 4. Handle Failures

If workflows fail after migration:

1. **Check token exists:**
   ```bash
   gh secret list --repo karnod/your-repo
   ```

2. **Verify token permissions:**
   - Go to GitHub Settings → Developer settings → Tokens
   - Check token hasn't expired
   - Verify it has access to `karnod/App.Template`

3. **Check error messages:**
   - "Repository not found" → Token permissions issue
   - "Resource not accessible" → Token expired or revoked
   - "Invalid workflow" → Different issue (not auth related)

## Breaking Changes

### What Changes

✅ **No changes required** to workflow files themselves  
✅ **No changes** to workflow syntax  
✅ **No changes** to inputs or secrets  
❌ **New requirement**: Authentication token must be configured

### What Stays the Same

- Workflow syntax: `uses: karnod/App.Template/...@main`
- All workflow inputs and secrets
- Workflow behavior and outputs
- Triggering mechanisms

## Rollback Plan

If issues arise after making the repository private:

### Immediate Rollback
```
1. Go to Settings → Change visibility
2. Make repository public
3. Notify users of delay
4. Investigate issues
```

### Investigate Common Issues
- Token not distributed to all repos
- Token permissions insufficient
- Token expired
- Organization settings blocking access

### Re-attempt Migration
```
1. Fix identified issues
2. Test thoroughly in a fork
3. Set new migration date
4. Communicate clearly with users
```

## Testing Strategy

### Before Going Private

Test these scenarios while the repository is still public:

#### Test 1: Token Works with Public Repo
```yaml
# Add token to a test repository
# Verify workflow runs successfully
# Confirms token is valid
```

#### Test 2: Token Has Correct Permissions
```bash
# Using GitHub API
curl -H "Authorization: token YOUR_TOKEN" \
  https://api.github.com/repos/karnod/App.Template

# Should return repository details (not 404)
```

#### Test 3: All Workflows Function
```bash
# Run each workflow type in test repo
# - dotnet-test
# - docker-publish
# - helm-deploy
# etc.
```

### After Going Private

#### Test 4: Workflows Still Work
```bash
# Trigger workflows in consuming repos
# All should complete successfully
```

#### Test 5: New Repository Can Access
```bash
# Create new test repository
# Add WORKFLOW_TOKEN secret
# Create workflow using template
# Should work immediately
```

## Communication Templates

### Email Template: Pre-Migration Notice

```
Subject: Action Required: GitHub Actions Template Repository Going Private

Hi Team,

The karnod/App.Template repository will become private on [DATE].

ACTION REQUIRED:
- No code changes needed
- Token has been configured in your repositories
- Test your workflows before [DATE]
- Report any issues to #devops

Resources:
- Migration Guide: [LINK]
- Quick Start: [LINK]
- Support: #devops channel

Questions? Contact DevOps team.

Thanks,
DevOps Team
```

### Slack Message Template

```
:warning: **GitHub Actions Migration** :warning:

The App.Template repository goes private on [DATE]

✅ Already done:
• Authentication tokens configured
• Documentation updated
• Test environment validated

📋 What you need to do:
• Test your workflows this week
• Report issues in #devops
• Read migration guide: [LINK]

:question: Questions? Ask in #devops
```

## Troubleshooting Common Issues

### Issue: "Resource not accessible by integration"

**Cause**: GitHub App or GITHUB_TOKEN lacks permissions

**Solution**: Use PAT instead
```yaml
# Don't rely on GITHUB_TOKEN for private repo access
secrets:
  WORKFLOW_TOKEN: ${{ secrets.WORKFLOW_TOKEN }}
```

### Issue: "Workflow does not exist"

**Cause**: Token can't access private repository

**Solution**: 
1. Verify token permissions
2. Check token hasn't expired
3. Ensure token has repo access

### Issue: Workflows work locally but fail in CI

**Cause**: Local environment has your personal credentials

**Solution**: 
1. Verify secret is configured in repository
2. Check secret name matches workflow expectations
3. Test in fresh clone without personal credentials

### Issue: Some repositories work, others don't

**Cause**: Inconsistent secret configuration

**Solution**:
1. Use organization secrets for consistency
2. Audit all repositories
3. Add missing secrets

## Verification Checklist

After migration, verify:

- [ ] All known consuming repositories have tokens
- [ ] Workflows execute successfully
- [ ] No authentication errors in logs
- [ ] New repositories can be onboarded easily
- [ ] Documentation is up to date
- [ ] Support channels are responsive
- [ ] Token rotation process is documented

## Long-term Maintenance

### Monthly Tasks
- [ ] Review token expiration dates
- [ ] Audit consuming repositories
- [ ] Check for failed workflows
- [ ] Update documentation as needed

### Quarterly Tasks
- [ ] Rotate access tokens
- [ ] Review access permissions
- [ ] Update migration guide
- [ ] Survey users for feedback

### Annual Tasks
- [ ] Security audit
- [ ] Review authentication strategy
- [ ] Consider GitHub App migration
- [ ] Update best practices

## Support Resources

- **Documentation**: See README.md, QUICKSTART.md, SECURITY.md
- **Examples**: See EXAMPLES.md for common patterns
- **Support Channel**: #devops on Slack
- **Issues**: Open issue in App.Template repository
- **Urgent**: Contact DevOps on-call

## Success Criteria

Migration is successful when:

✅ Repository is private  
✅ All consuming repositories continue to work  
✅ Zero production incidents related to authentication  
✅ New repositories can onboard in <5 minutes  
✅ Documentation is comprehensive and clear  
✅ Team is trained on new process  

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-16  
**Next Review**: After migration complete
