# Azure API Management (APIM)

## The multi-repository approach

### Why Multi-Repository?

Instead of one giant repository, the system is broken into three specailized repo types, each handling a differend responsibility. This separation increases resuability and provides teams with greater autonomy.

---

### APIM Tooling Repository

Contains reusable deployment logic and shared workflows that the *other* repositories reference.

---

### Infrastructure Configuration Repository

* Manages environment configurations and infrasturcture deployments.
* I.e. stores environment-level settings (like dev, uat, prod).
* Handles the *Manual* APIM Infrastructure deployment. 
    > note: Infrastructure deployments are **manual only** to prevent accidental changes.

**Typical Flow**:

1. Developer updates environment configuration
2. Creates PR → Configuration validation runs
3. After PR merge → Manual infrastructure deployment
4. APIM instance ready for API deployments

---

### Service Repositories (many)

* Individual service repos with automated API deployment
* I.e. each application or API service has its own repo
* APIs automatically deployed after branch merge event

```yaml
# Service repo: .github/workflows/deploy-api.yml

develop branch → dev environment    # Automatic
main branch    → prod environment   # Automatic
manual trigger → any environment    # Manual with environment selection
```

**Typical Flow**:

1. Developer updates API spec/config
2. Creates PR → API validation runs
3. Merge to develop → Automatic deployment to dev
4. Merge to main → Automatic deployment to prod

---

### How It All Fits Together

* Developers work in their service repo -> commit API code + spec -> GitHub Actions trigger -> calls tooling repo workflows -> deploys API into APIM.

* Ops/Platform team manages the infra repo -> provisions APIM instances and environment configs -> uses the tooling repo to standardize deployments.

* Tooling repo is the "shared library" -> ensures consistency and scalability across many teams/services.

---

### Repository Management considerations

* Use separate repositories for infrastructure, services, and tooling
* Pin workflow versions for stability
* Regular security scanning and dependency updates
* Implement branch protection and required reviews
    * __Infrastructure Repository__:
        * Restrict write access to infrastructure team
        * Require PR reviews for environment changes
        * Use branch protection rules
    * __Service Repositories__:
        * Service teams have full access to their repos
        * Read-only access to infrastructure config repo
        * Separate Azure credentials if needed

---
