# DevOps

This section covers Infrastructure as Code (IaC), CI/CD pipelines, and deployment automation for Azure API Management. It provides comprehensive guidance on implementing GitOps practices and managing APIM infrastructure and APIs as code.

## Table of Contents

- [Overview](./overview.md) - Introduction to DevOps practices for APIM
- [Infrastructure as Code with Bicep](./bicep.md) - Bicep templates and patterns for APIM
- [ARM Templates](./arm-templates.md) - Azure Resource Manager template approach
- [GitHub Actions](./github-actions.md) - CI/CD pipelines with GitHub
- [Multi-Repository Strategy](./multi-repository-strategy.md) - Organizing infrastructure and service repos
- [Automated Deployments](./automated-deployments.md) - API deployment automation
- [Environment Management](./environment-management.md) - Managing dev, test, and prod environments
- [Security Best Practices](./security-best-practices.md) - Secure CI/CD and secrets management

## Why Automate APIM?

Manual configuration through the Azure Portal (click-ops) is:
- **Error-prone** - Easy to misconfigure or forget steps
- **Not repeatable** - Difficult to recreate environments consistently
- **Unauditable** - No clear history of who changed what and when
- **Slow** - Time-consuming for every deployment
- **Not scalable** - Doesn't scale across multiple environments or teams

Infrastructure as Code and automation provide:
- **Repeatability** - Identical deployments across environments
- **Version Control** - Git history of all changes
- **Auditability** - Clear trail of changes and approvals
- **Speed** - Automated deployments in minutes
- **Safety** - Validation and testing before production
- **Disaster Recovery** - Recreate infrastructure from code

## Core Patterns

### GitOps

**Principle:** Git is the single source of truth for infrastructure and configuration

**Implementation:**
- All APIM configuration in git repositories
- Changes made via pull requests
- Automated deployments triggered by merges
- Rollback by reverting commits

### Infrastructure as Code

**Principle:** Define infrastructure declaratively in code

**Tools:**
- **Bicep** - Simplified Azure template language (recommended)
- **ARM Templates** - JSON-based Azure templates
- **Terraform** - Multi-cloud IaC (if needed)

### Separation of Concerns

**Principle:** Separate different types of configuration

**Structure:**
1. **Infrastructure Repository** - APIM service, networking, shared resources
2. **Tooling Repository** - Reusable workflows and scripts
3. **Service Repositories** - Individual API definitions and configurations

### Idempotent Deployments

**Principle:** Deployments can be safely re-run without adverse effects

**Benefit:** Declarative templates converge to desired state automatically

## Quick Start

If you're new to DevOps with APIM:

1. **Understand Fundamentals** - Read [Overview](./overview.md)
2. **Choose IaC Tool** - Start with [Bicep](./bicep.md) (recommended)
3. **Set Up GitHub Actions** - Implement [CI/CD pipelines](./github-actions.md)
4. **Organize Repositories** - Follow [Multi-Repository Strategy](./multi-repository-strategy.md)
5. **Automate API Deployments** - Enable [Automated Deployments](./automated-deployments.md)

## Reference Implementations

This documentation references three companion repositories:

### 1. Azure APIM Bicep
**Repository:** [azure-apim-bicep](https://github.com/shane-m-holland/azure-apim-bicep)

**Contents:**
- Reusable Bicep templates for APIM infrastructure
- Shell scripts for deployment automation
- GitHub Actions reusable workflows
- Validation and testing tools

**Use Case:** Tooling repository that other repos reference

### 2. Example APIM Infrastructure
**Repository:** [example-apim-infrastructure](https://github.com/shane-m-holland/example-apim-infrastructure)

**Contents:**
- Environment-specific configurations
- Infrastructure deployment definitions
- Manual deployment workflows

**Use Case:** Infrastructure repository pattern reference

### 3. APIM Service Example (Python)
**Repository:** [apim-service-example-py](https://github.com/shane-m-holland/apim-service-example-py)

**Contents:**
- Example service repository structure
- Automated API deployment workflows
- API specification management

**Use Case:** Service repository pattern reference

## Key Concepts

### Environments

Separate configurations for each environment:
- **Development** - Rapid iteration, full logging
- **Testing/Staging** - Pre-production validation
- **Production** - Production-grade configuration

### Deployment Strategies

**Infrastructure Deployments:**
- **Manual** - Gated with approvals
- **Infrequent** - Only when infrastructure changes needed
- **Validated** - What-if preview before deployment

**API Deployments:**
- **Automatic** - Triggered on code merge
- **Frequent** - Every API change deployed
- **Tested** - Validation before deployment

### Revisions and Versions

**Revisions** - Safe, non-breaking updates:
- Create new revision for changes
- Test revision
- Make revision current when validated
- Rollback by reverting to previous revision

**Versions** - Breaking changes:
- Create new API version (v2, v3)
- Deploy alongside existing versions
- Migrate consumers gradually
- Deprecate old versions over time

## Workflow Examples

### Infrastructure Change Workflow

```mermaid
Developer → Create branch
         → Modify Bicep template
         → Create Pull Request
         → Automated validation runs
         → Team reviews PR
         → PR approved and merged
         → Manual deployment triggered
         → What-if analysis
         → Approve deployment
         → Infrastructure updated
```

### API Change Workflow

```mermaid
Developer → Update API spec
         → Commit to feature branch
         → Create Pull Request
         → API validation runs
         → Team reviews PR
         → PR merged to develop
         → Auto-deploy to dev environment
         → Test in dev
         → PR to main branch
         → Auto-deploy to production
```

## Security Considerations

### Secrets Management

**Never commit:**
- API keys or tokens
- Passwords or connection strings
- Certificates (except public certs)
- Environment-specific secrets

**Use instead:**
- **Azure Key Vault** - For runtime secrets
- **GitHub Secrets** - For CI/CD credentials
- **Managed Identities** - For Azure resource access
- **OIDC Federation** - For GitHub to Azure auth

### Access Control

**Repository Access:**
- Read-only for most developers
- Write access for designated approvers
- Branch protection rules
- Required reviews for merges

**Azure Permissions:**
- Least privilege principle
- Separate credentials per environment
- Managed identities where possible
- Regular credential rotation

## Monitoring Deployments

Track and monitor deployments:

### Deployment History

- Azure deployment history
- Git commit history
- GitHub Actions run logs
- Audit logs

### Deployment Metrics

- Deployment frequency
- Success/failure rates
- Deployment duration
- Rollback frequency

### Alerting

**Configure alerts for:**
- Deployment failures
- Long-running deployments
- Rollbacks
- Configuration drift

## Best Practices Summary

1. **Infrastructure as Code first** - All configuration in git
2. **Use Bicep** - Simpler than ARM, Azure-native
3. **Separate concerns** - Infrastructure, tooling, and service repos
4. **Automate API deployments** - Fast, safe, repeatable
5. **Manual infrastructure deployments** - Gated with approvals
6. **Use revisions** - Safe, testable updates
7. **Validate before deploy** - what-if analysis and linting
8. **Secrets in Key Vault** - Never in code
9. **Monitor deployments** - Track metrics and failures
10. **Document runbooks** - Incident response procedures

## Common Pitfalls

### Pitfall 1: Configuration Drift

**Problem:** Manual changes in portal not reflected in code

**Solution:** Export configuration regularly, compare with code, enforce code-only changes

### Pitfall 2: Hardcoded Values

**Problem:** Environment-specific values in templates

**Solution:** Use parameters and environment files

### Pitfall 3: No Rollback Plan

**Problem:** Failed deployment with no way to recover

**Solution:** Use revisions, maintain previous versions, test rollback procedures

### Pitfall 4: Inadequate Testing

**Problem:** Deployments fail in production

**Solution:** Deploy to dev/test first, automated validation, what-if analysis

### Pitfall 5: Poor Secret Management

**Problem:** Secrets committed to git

**Solution:** Use Key Vault, GitHub Secrets, never commit secrets

## Troubleshooting

### Deployment Failures

**Check:**
1. Azure Activity Log for error details
2. GitHub Actions logs
3. Template validation errors
4. Permission issues
5. Resource dependencies

### Configuration Drift

**Detect:**
1. Export current APIM configuration
2. Compare with code using diff tools
3. Identify manual changes
4. Update code or revert manual changes

### Performance Issues

**Optimize:**
1. Parallel API deployments
2. Incremental deployments (only changed APIs)
3. Efficient polling and waiting
4. Caching of unchanged resources

## Next Steps

1. **Review Overview** - Understand DevOps principles for APIM
2. **Choose IaC Approach** - Start with Bicep
3. **Set Up Repositories** - Implement multi-repo pattern
4. **Configure GitHub Actions** - Automate deployments
5. **Test Automation** - Validate in dev environment
6. **Deploy to Production** - Follow runbook and monitor

## Additional Resources

- [Azure Bicep Documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [APIM DevOps Resource Kit](https://github.com/Azure/azure-api-management-devops-resource-kit)
- [Azure DevOps for APIM](https://learn.microsoft.com/en-us/azure/api-management/devops-api-development-templates)
