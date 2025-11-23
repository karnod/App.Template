# Security Guide for Private Workflow Templates

This document provides comprehensive security guidance for using reusable workflows from this private repository.

## Overview

When using reusable workflows from a private repository, you need to ensure secure authentication while maintaining the principle of least privilege. This guide covers best practices for managing access tokens, secrets, and security configurations.

## Authentication Methods

### Option 1: Fine-Grained Personal Access Tokens (Recommended)

Fine-grained PATs provide better security with repository-specific access and granular permissions.

#### Creating a Fine-Grained PAT

1. Navigate to: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. Click "Generate new token"
3. Configure token settings:

```
Token name: Workflow Templates Access
Description: Access to private reusable workflow repository
Expiration: 90 days (recommended)
Resource owner: karnod (or your organization)
Repository access: Only select repositories
  - Selected repositories: karnod/App.Template
```

4. Set Repository permissions:
   - **Actions**: Read-only ✓
   - **Contents**: Read-only ✓
   - **Metadata**: Read-only ✓ (automatically included)

5. Click "Generate token" and save it immediately

#### Benefits
- ✅ Repository-specific access
- ✅ Granular permission control
- ✅ Better audit trail
- ✅ Can be scoped to specific organizations

### Option 2: Classic Personal Access Tokens

Classic PATs are simpler but have broader access.

#### Creating a Classic PAT

1. Navigate to: GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Configure:
```
Note: Workflow Templates Access
Expiration: 90 days
Scopes:
  - ✓ repo (Full control of private repositories)
    or
  - ✓ repo:status (Access commit status)
  - ✓ repo_deployment (Access deployment status)
  - ✓ public_repo (Access public repositories)
```

4. Click "Generate token" and save immediately

#### Considerations
- ⚠️ Broader access than fine-grained tokens
- ⚠️ Access to all repositories the user can access
- ✅ Simpler to set up
- ✅ Works with older GitHub Enterprise versions

### Option 3: GitHub Apps (Enterprise/Organization)

For organization-wide use, GitHub Apps provide the best security model.

#### Benefits
- ✅ Independent of user accounts
- ✅ Fine-grained repository access
- ✅ Better audit logging
- ✅ Tokens auto-expire (1 hour)
- ✅ Can be installed organization-wide

#### Setup Steps

1. Create GitHub App:
   - Navigate to Organization Settings → Developer settings → GitHub Apps
   - Click "New GitHub App"
   - Configure:
     ```
     Name: Workflow Templates Access
     Homepage URL: https://github.com/karnod/App.Template
     Webhook: Disable (uncheck "Active")
     ```

2. Set Permissions:
   - Repository permissions:
     - Actions: Read
     - Contents: Read
     - Metadata: Read

3. Install App:
   - Install the app in your organization
   - Grant access to repositories that need workflow templates

4. Use in Workflows:
   ```yaml
   - uses: actions/create-github-app-token@v1
     id: app-token
     with:
       app-id: ${{ vars.APP_ID }}
       private-key: ${{ secrets.APP_PRIVATE_KEY }}
   ```

## Secret Management

### Storing Secrets

#### Repository Secrets
For individual repositories:
```
Settings → Secrets and variables → Actions → New repository secret
```

#### Organization Secrets
For organization-wide access:
```
Organization Settings → Secrets and variables → Actions → New organization secret
```

Select which repositories can access the secret:
- All repositories
- Private repositories
- Selected repositories (recommended)

#### Environment Secrets
For deployment workflows:
```
Settings → Environments → [Environment Name] → Add secret
```

Benefits:
- Environment-specific credentials
- Protection rules (required reviewers, wait timers)
- Deployment branches restrictions

### Secret Naming Conventions

Use clear, consistent naming:

```yaml
# Good examples
WORKFLOW_TOKEN          # For accessing this template repository
GHCR_TOKEN             # For GitHub Container Registry
HELM_REPO_TOKEN        # For Helm chart repository
K8S_CONFIG_PROD        # Production Kubernetes config
K8S_CONFIG_STAGING     # Staging Kubernetes config

# Avoid
TOKEN                  # Too generic
SECRET                 # Not descriptive
PAT                    # Unclear purpose
```

## Token Permissions Matrix

### Minimum Required Permissions

| Workflow | Token Type | Permissions Required |
|----------|-----------|---------------------|
| docker-publish.yml | Classic PAT | `write:packages`, `repo` (if private registry) |
| dotnet-test.yml | Fine-grained | Actions: Read, Contents: Read |
| dotnet-publish.yml | Fine-grained | Actions: Read, Contents: Read |
| helm-deploy.yml | Classic PAT | `repo` (for helm-charts repo access) |
| helm-publish.yml | Classic PAT | `repo` (for helm-charts repo write) |
| terraform-deploy.yml | Service Principal | Azure ARM credentials |

### Calling Reusable Workflows

For accessing reusable workflows in this repository:

| Token Type | Minimum Permissions |
|-----------|---------------------|
| Fine-grained PAT | Actions: Read, Contents: Read, Metadata: Read |
| Classic PAT | `repo` scope |
| GitHub App | Actions: Read, Contents: Read |

## Token Lifecycle Management

### Token Expiration

Set appropriate expiration periods:

- **Development/Testing**: 30 days
- **Production**: 90 days maximum
- **CI/CD Service Accounts**: 90 days with rotation

### Rotation Schedule

```
T-14 days: Create new token
T-7 days:  Update development environments
T-3 days:  Update staging environments
T-1 day:   Update production environments
T-day:     Verify all systems operational
T+1 day:   Revoke old token
```

### Rotation Process

1. **Create new token** with same permissions
2. **Test in non-production** environment
3. **Update secrets** in all repositories/organizations
4. **Monitor workflows** for 24-48 hours
5. **Revoke old token** after verification

### Automated Rotation Reminders

Create a reminder issue:

```yaml
name: Token Rotation Reminder
on:
  schedule:
    - cron: '0 9 1 * *'  # First day of each month

jobs:
  remind:
    runs-on: ubuntu-latest
    steps:
      - name: Check token expiration
        run: |
          echo "🔐 Monthly reminder: Check PAT expiration dates"
          echo "1. Review tokens in GitHub Settings"
          echo "2. Rotate any tokens expiring in <30 days"
          echo "3. Update SECURITY.md with rotation date"
```

## Security Best Practices

### 1. Principle of Least Privilege

```yaml
# ❌ Bad: Overly permissive
# Using admin token for read-only operation

# ✅ Good: Minimal permissions
# Fine-grained token with read-only access to specific repository
```

### 2. Scope Limitation

```yaml
# ❌ Bad: Organization-wide token
# Token with access to all organization repositories

# ✅ Good: Repository-specific token
# Fine-grained token scoped to karnod/App.Template only
```

### 3. Regular Auditing

Monthly checklist:
- [ ] Review all PATs in use
- [ ] Check token expiration dates
- [ ] Audit secret access logs
- [ ] Verify principle of least privilege
- [ ] Remove unused tokens/secrets
- [ ] Update this document with findings

### 4. Secret Scanning

Enable GitHub secret scanning:
```
Repository Settings → Security → Code security and analysis
  ✓ Secret scanning
  ✓ Push protection
```

### 5. Environment Protection

For production deployments:
```
Settings → Environments → production
  ✓ Required reviewers: 2
  ✓ Wait timer: 5 minutes
  ✓ Deployment branches: main only
```

## Incident Response

### If a Token is Compromised

1. **Immediate Actions** (within 1 hour):
   ```
   - Revoke compromised token immediately
   - Review access logs for unauthorized use
   - Create new token with different name
   - Update all consuming repositories
   ```

2. **Investigation** (within 24 hours):
   ```
   - Identify scope of access
   - Review audit logs for suspicious activity
   - Check for unauthorized repository access
   - Document findings
   ```

3. **Remediation** (within 48 hours):
   ```
   - Rotate all related secrets
   - Update access controls
   - Implement additional monitoring
   - Conduct security review
   ```

4. **Post-Incident** (within 1 week):
   ```
   - Update security documentation
   - Share lessons learned with team
   - Enhance detection mechanisms
   - Review and update response procedures
   ```

### Detection Indicators

Watch for:
- 🚨 Failed authentication attempts
- 🚨 Unusual access patterns
- 🚨 Workflows triggered at unusual times
- 🚨 Unexpected repository access
- 🚨 Secret scanning alerts
- 🚨 Rate limit warnings

## Compliance Considerations

### Audit Trail

GitHub provides audit logs for:
- Token creation and deletion
- Secret access (organization level)
- Workflow execution
- Repository access

Access audit logs:
```
Organization Settings → Audit log
```

Export audit logs for compliance:
```bash
# Using GitHub CLI
gh api /orgs/karnod/audit-log \
  --paginate \
  -H "Accept: application/vnd.github+json" \
  > audit-log.json
```

### Regulatory Requirements

| Requirement | Implementation |
|------------|----------------|
| SOC 2 | Use organization secrets with restricted access |
| HIPAA | Implement environment protection rules |
| PCI DSS | Rotate tokens every 90 days |
| ISO 27001 | Maintain audit trail and access reviews |

## Advanced: Token Management Automation

### Automated Token Rotation Script

```bash
#!/bin/bash
# token-rotation-helper.sh

echo "🔐 Token Rotation Helper"
echo "======================="
echo ""
echo "Current tokens to rotate:"
echo "1. WORKFLOW_TOKEN - Expires: $(date -d '+30 days')"
echo "2. HELM_REPO_TOKEN - Expires: $(date -d '+45 days')"
echo ""
echo "Steps:"
echo "1. Create new tokens with same permissions"
echo "2. Update secrets in repositories:"
echo "   - Repository A"
echo "   - Repository B"
echo "   - Repository C"
echo "3. Test workflows in staging"
echo "4. Roll out to production"
echo "5. Revoke old tokens after 48 hours"
```

### Monitoring Token Usage

```yaml
name: Token Health Check
on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - name: Check token validity
        env:
          WORKFLOW_TOKEN: ${{ secrets.WORKFLOW_TOKEN }}
        run: |
          response=$(curl -s -o /dev/null -w "%{http_code}" \
            -H "Authorization: token $WORKFLOW_TOKEN" \
            https://api.github.com/user)
          
          if [ "$response" = "200" ]; then
            echo "✅ Token is valid"
          else
            echo "❌ Token validation failed: $response"
            exit 1
          fi
```

## Resources

### GitHub Documentation
- [Encrypted Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Automatic Token Authentication](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)
- [Security Hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

### Security Tools
- [GitHub Secret Scanning](https://docs.github.com/en/code-security/secret-scanning)
- [Dependabot](https://docs.github.com/en/code-security/dependabot)
- [CodeQL](https://codeql.github.com/)

### Best Practices
- [OWASP Security Principles](https://owasp.org/www-project-top-ten/)
- [CIS GitHub Benchmark](https://www.cisecurity.org/)

## Support

For security concerns or questions:
1. Review this document
2. Check GitHub's security documentation
3. Contact repository maintainers
4. For security vulnerabilities, follow responsible disclosure

---

**Last Updated**: 2025-11-16  
**Next Review**: 2026-02-16  
**Version**: 1.0
