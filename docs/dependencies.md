# Dependencies

## External Dependencies

This repository has **no runtime dependencies**. It is an assets-only repository with no application code, services, or executable components.

Dependencies exist only at the **CI/CD and tooling level**:

### CI/CD Dependencies

| Dependency | Version | Purpose | Integration |
|------------|---------|---------|-------------|
| `actions/checkout` | v4 | Repository checkout in GitHub Actions | `.github/workflows/release-asset.yml` |
| `softprops/action-gh-release` | v2 | GitHub Release asset upload | `.github/workflows/release-asset.yml` |
| `ubuntu-latest` runner | GitHub-hosted | CI execution environment | GitHub Actions platform |
| `zip` (Info-ZIP) | Ubuntu default | Archive packaging | Workflow `zip -r` command |
| `bash` | Ubuntu default | Workflow script execution | Workflow `shell: bash` |

### Optional Local Tooling (Contributor Use)

| Tool | Purpose | Required | Installation |
|------|---------|----------|--------------|
| `xmllint` (libxml2) | XML validation, schema validation | No (advisory) | `apt install libxml2-utils` / `brew install libxml2` |
| `git` | Version control, whitespace check | Yes | Standard |
| Text editor | Editing `.lst` and `.xml` files | Yes | Any UTF-8 capable editor |

### Downstream Consumer Dependencies (Not This Repository's Concern)

Consumers (Universal Name Generator API) have their own dependencies for:
- XML parsing (`GenerationSchemas.xml`)
- File I/O (reading `.lst` files)
- Generation algorithm implementation (randomiser, markov)
- Filter list processing
- Build system integration

This repository does not declare, manage, or constrain consumer dependencies.

## Internal Dependencies

### Schema to Wordlist Coupling

`GenerationSchemas.xml` references `wordlists/` paths in `Schema` strings:

```
{randomiser, ,4,32,real/arabic/persons/males|real/arabic/persons/surnames}
```

**Dependency direction**: Schemas depend on wordlists (not vice versa).

**Coupling strength**:
- Path references are strings; no compile-time validation
- Breaking if `.lst` file moved/renamed without schema update
- Multiple schemas may reference same `.lst` file (shared dependency)

### Filter List to Schema Coupling

Optional `FilterlistPath` in `GenerationSchemaDataObject` references filter `.lst` files:

```xml
<FilterlistPath>real/german/persons/filters</FilterlistPath>
```

**Dependency direction**: Schemas depend on filter lists (optional).

### Release Packaging to Assets Coupling

Workflow packages fixed paths: `wordlists/` and `GenerationSchemas.xml`.

**Dependency direction**: Workflow depends on asset locations (fixed).

## Dependency Graph

```mermaid
flowchart TD
    subgraph "This Repository"
        Workflow[release-asset.yml]
        Schemas[GenerationSchemas.xml]
        Wordlists[wordlists/*.lst]
        Filters[wordlists/*/filters/*.lst]
    end

    subgraph "External CI"
        Checkout[actions/checkout@v4]
        GHRelease[softprops/action-gh-release@v2]
        Runner[ubuntu-latest]
        Zip[zip]
    end

    subgraph "Downstream Consumers"
        API[Universal Name Generator API]
        Others[Other consumers]
    end

    Workflow -->|uses| Checkout
    Workflow -->|uses| GHRelease
    Workflow -->|runs on| Runner
    Workflow -->|uses| Zip
    Workflow -->|packages| Wordlists
    Workflow -->|packages| Schemas
    Schemas -->|references| Wordlists
    Schemas -.->|optional references| Filters
    Wordlists -.->|shared by| Schemas
    Filters -.->|referenced by| Schemas
    Workflow -->|produces| Release[GitHub Release asset]
    Release -->|consumed by| API
    Release -->|consumed by| Others
```

## Version Constraints

### Pinned Dependencies

| Dependency | Pinning | Update Policy |
|------------|---------|---------------|
| `actions/checkout` | v4 (major version) | Update when v5 released; test workflow |
| `softprops/action-gh-release` | v2 (major version) | Update when v3 released; test workflow |
| `ubuntu-latest` | Floating (GitHub-managed) | Automatic; may introduce breaking changes |

### No Version Constraints

- `zip` -- Uses Ubuntu default; no version pinning
- `bash` -- Uses Ubuntu default; no version pinning
- `xmllint` -- Optional local tooling; no version requirement

## Vulnerability Management

### Repository Scope

This repository has **no direct vulnerabilities** because:
- No application code
- No runtime
- No network services
- No user input processing
- No dependencies with known vulnerabilities (only CI actions)

### Indirect Exposure

| Vector | Risk | Mitigation |
|--------|------|------------|
| Compromised GitHub Action | Supply chain attack | Pinned major versions; monitor advisories |
| Malicious `.lst` content | Downstream generator produces harmful output | PR review; schema validation; ROADMAP duplicate detection |
| Malicious schema | Downstream generator misbehaves | PR review; Id stability policy |

### Monitoring

- GitHub Dependabot alerts for workflow actions (if enabled)
- Manual review of action updates before major version bumps
- Security policy in `../SECURITY.md` covers reporting

## Dependency Update Procedure

### GitHub Actions Updates

1. Monitor Dependabot PRs or release notes for `actions/checkout` and `softprops/action-gh-release`
2. Test workflow with new version in fork/branch
3. Update version pin in `.github/workflows/release-asset.yml`
4. Create PR, verify workflow passes
5. Merge to `master`

### Ubuntu Runner Updates

- Automatic via GitHub; no repository action needed
- Monitor workflow failures after runner updates
- Adjust workflow if breaking changes introduced

### Local Tooling Updates

- Contributor responsibility; no repository enforcement
- Document minimum versions if validation becomes mandatory in CI

## Prohibited Dependencies

Per design constraints (`../ARCHITECTURE.md`):
- **No application code dependencies** -- No NuGet, npm, PyPI, Cargo, Go modules, etc.
- **No runtime dependencies** -- No databases, message queues, caches, external APIs
- **No build dependencies** -- No compilers, transpilers, bundlers, test frameworks
- **No development dependencies** -- No linters, formatters, analysers (except optional `xmllint`)

## Related Documentation

- Architecture: `./architecture.md` (external dependencies table)
- Build and deployment: `./build-and-deployment.md` (workflow detail)
- Configuration: `./configuration.md` (workflow configuration)
- Security: `../SECURITY.md` (vulnerability reporting)
