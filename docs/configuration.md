# Configuration

## Configuration Sources

This repository has **no runtime configuration**. It is an assets-only repository with no application code, services, or executable components.

Configuration exists only at the **repository metadata level**:

| Configuration | Location | Purpose |
|---------------|----------|---------|
| GitHub Actions workflow | `.github/workflows/release-asset.yml` | Release packaging automation |
| Funding configuration | `.github/FUNDING.yml` | GitHub Sponsors / funding links |
| Repository settings | GitHub UI (branch protection, rulesets) | Contribution governance |
| Git configuration | `.gitattributes`, `.gitignore` | Line endings, ignored files |

## Schema Configuration

The `GenerationSchemas.xml` file acts as a **declarative configuration** for downstream generators:

### GenerationSchemaDataObject Fields

| Field | Role | Constraints |
|-------|------|-------------|
| `Id` | Primary key for consumers | Unique, stable, kebab-case |
| `Name` | Display name | Human-readable |
| `Category` | Logical grouping | Used for UI organisation |
| `Schema` | Algorithm + parameters + paths | Must reference existing `.lst` paths |
| `FilterlistPath` | Optional filter reference | Repository-relative path |

### Algorithm Parameters as Configuration

The `Schema` string encodes generation behaviour:

```
{randomiser, ,4,32,path1|path2}     -> minLen=4, maxLen=32, random selection
{markov,4,24,path1|path2}           -> order=4, minLen=4, maxLen=24, Markov chain
```

These parameters are **consumer-executed**; this repository only declares them.

## Release Configuration

### Version Extraction

The workflow extracts version from the Git tag:

```bash
version="${{ github.event.release.tag_name }}"
version="${version#v}"  # Strip 'v' prefix
```

**Requirement**: Tags must follow `vX.Y.Z` format (Semantic Versioning).

### Archive Naming

```
universal-name-generator-wordlists_${version}.zip
```

Example: `v3.2.0` -> `universal-name-generator-wordlists_3.2.0.zip`

### Packaging Scope

Fixed in workflow: `wordlists/` and `GenerationSchemas.xml` only.

No configuration to include/exclude specific universes or categories.

## Contributor Configuration

### Local Validation (Optional)

Contributors may configure local validation:

```bash
# XML well-formedness
xmllint --noout GenerationSchemas.xml

# Schema validation (advisory)
xmllint --schema GenerationSchemas.xml wordlists/**/*.lst

# Whitespace check
git diff --check
```

No enforced pre-commit hooks or CI validation currently.

### Directory Naming Conventions (Enforced by Review)

| Level | Convention | Example |
|-------|------------|---------|
| Universe | lowercase, hyphen-separated | `age-of-wonders` |
| Category | plural noun | `persons`, `locations` |
| Data file | descriptive, lowercase, hyphen-separated | `males.lst`, `toponyms.lst` |
| Schema Id | kebab-case, universe/category prefix | `arabic-persons-male` |

## Downstream Consumer Configuration

Consumers (Universal Name Generator API) configure at **build time**:
1. Download release archive from GitHub Releases
2. Extract to known location
3. Configure generator to reference extracted `wordlists/` and `GenerationSchemas.xml`
4. Embed or load at runtime per consumer implementation

No runtime configuration provided by this repository.

## Environment Variables

None. The repository has no runtime environment.

## Secrets Management

No secrets used or stored. GitHub Actions uses `GITHUB_TOKEN` (automatic) for release asset upload.

## Configuration Drift Prevention

- Schema `Id` stability enforced by review (breaking change policy)
- Archive layout fixed in workflow (no configuration)
- Version format enforced by workflow string manipulation
- No external configuration files or services
