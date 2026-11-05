# Invariants

This document records system-wide invariants and contracts that must remain true for the repository to function correctly. Violations indicate defects or breaking changes.

## Structural Invariants

### Repository Structure

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| `wordlists/` exists | Root data directory present | `test -d wordlists` |
| `GenerationSchemas.xml` exists | Schema catalogue present | `test -f GenerationSchemas.xml` |
| No code files | No `.cs`, `.js`, `.py`, `.exe`, etc. | `find . -name "*.cs" -o -name "*.js" ...` |
| `.github/workflows/release-asset.yml` exists | Release workflow present | `test -f .github/workflows/release-asset.yml` |

### Wordlist Format

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| UTF-8 encoding | All `.lst` files are valid UTF-8 | `file -i wordlists/**/*.lst \| grep utf-8` |
| One entry per line | No multi-line entries | `grep -c $ wordlists/**/*.lst` matches line count |
| No trailing whitespace | Lines end cleanly | `git diff --check` / `grep -r '[[:space:]]$' wordlists/` |
| No empty lines | No blank lines in files | `grep -c '^$' wordlists/**/*.lst` returns 0 |

### Schema Structure

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| XML well-formed | `GenerationSchemas.xml` parses | `xmllint --noout GenerationSchemas.xml` |
| Root element | `ArrayOfGenerationSchemaDataObject` | `xmllint --xpath 'name(/*)' GenerationSchemas.xml` |
| Child elements | All children are `GenerationSchemaDataObject` | `xmllint --xpath 'count(/*/*)' GenerationSchemas.xml` |
| Required fields | Each entry has `Id`, `Name`, `Category`, `Schema` | XPath queries for each field |
| Unique Ids | No duplicate `Id` values | `xmllint --xpath '//Id' GenerationSchemas.xml \| sort \| uniq -d` |

## Referential Invariants

### Schema to Wordlist Path References

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| Paths exist | Every `Schema` path references existing `.lst` | `xmllint --schema GenerationSchemas.xml wordlists/**/*.lst` (advisory) |
| Path format | Paths use `wordlists/` relative format | Regex: `^[a-z0-9-]+(/[a-z0-9-]+)+$` |
| No absolute paths | No `/wordlists/` or `C:\` paths | Grep for leading slash or drive letter |

### Filter List References

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| Optional | `FilterlistPath` may be absent | XPath: `count(//FilterlistPath) <= count(//GenerationSchemaDataObject)` |
| Paths exist | Referenced filter files exist | Check each `FilterlistPath` + `.lst` exists |

## Release Invariants

### Archive Structure

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| Root layout | Archive contains `wordlists/` and `GenerationSchemas.xml` at root | `unzip -l archive.zip \| grep -E '^(wordlists/|GenerationSchemas\.xml)$'` |
| No wrapper | No single top-level directory wrapping contents | `unzip -l archive.zip \| head -5` |
| Complete | All 388 `.lst` files included | `unzip -l archive.zip \| grep '\.lst$' \| wc -l` |
| Schema included | `GenerationSchemas.xml` in archive | `unzip -l archive.zip \| grep GenerationSchemas.xml` |

### Versioning

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| Tag format | Git tags follow `vX.Y.Z` | `git tag -l 'v*.*.*'` |
| Archive name | `universal-name-generator-wordlists_X.Y.Z.zip` | Workflow output matches pattern |
| Version extraction | Workflow correctly strips `v` prefix | Manual test with sample tag |

## Compatibility Invariants

### Schema Id Stability

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| Ids never change | Existing `Id` values immutable | `git log --oneline --all -- GenerationSchemas.xml \| grep -i 'id.*rename'` |
| New Ids unique | New entries don't conflict | XPath uniqueness check on each PR |
| Kebab-case | Ids follow `lowercase-with-hyphens` | Regex: `^[a-z0-9]+(-[a-z0-9]+)*$` |

### Schema Path Stability

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| Paths stable | `.lst` paths in `Schema` strings don't change | `git log -p GenerationSchemas.xml \| grep 'Schema.*path'` |
| Coordinated moves | File moves accompanied by schema updates | PR review checklist |

### Archive Layout Stability

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| Layout fixed | Archive root structure unchanged | Workflow `zip` command unchanged |
| Consumer expectation | Downstream consumers depend on layout | Documented in `ARCHITECTURE.md` |

## Data Quality Invariants (Aspirational)

### Duplicate Management

| Invariant | Status | Description |
|-----------|--------|-------------|
| No intra-file duplicates | Not enforced | No duplicate lines within a single `.lst` |
| No cross-file duplicates | Not enforced | No duplicate entries across related `.lst` files |
| Duplicate reporting | Planned | ROADMAP.md: "repository-wide duplicate detection reporting" |

### Formatting Consistency

| Invariant | Status | Description |
|-----------|--------|-------------|
| Canonical formatting | Not enforced | All `.lst` files follow documented conventions |
| Category naming | Enforced by review | Plural nouns for categories |
| Universe naming | Enforced by review | Lowercase, hyphen-separated |

## Operational Invariants

### CI/CD

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| Workflow triggers | Only on `release.published` | Workflow `on:` clause |
| Permissions | `contents: write` for asset upload | Workflow `permissions:` |
| No secrets | Only `GITHUB_TOKEN` used | Workflow `env:` section |

### Governance

| Invariant | Description | Verification |
|-----------|-------------|--------------|
| Root documentation | All policy docs at repository root | `ls *.md LICENSE` |
| No `docs/` in root | Documentation under `docs/` only | `test ! -d docs` at root (this is the new docs) |
| License | GPL v3.0 in `LICENSE` | `head -20 LICENSE` |

## Invariant Violation Response

| Invariant Category | Violation Severity | Response |
|--------------------|-------------------|----------|
| Structural | Critical | Block release; revert or fix immediately |
| Referential | High | Block release; fix schema or restore files |
| Release | Critical | Delete release; re-package after fix |
| Compatibility | Breaking | Coordinate with consumers; major version bump |
| Data Quality | Medium | Track in ROADMAP; remediate over time |
| Operational | Medium | Fix workflow/config; verify next release |

## Monitoring Invariants

### Automated (Current)

- Workflow success/failure (GitHub Actions)
- Release asset presence (GitHub Releases API)

### Manual (Current)

- Post-release archive inspection
- PR review for schema/wordlist changes

### Planned (Per ROADMAP.md)

- CI-enforced XML validation
- CI-enforced path reference validation
- Duplicate detection reporting
- Canonical formatting enforcement

## Related Documentation

- Architecture: `./architecture.md` (compatibility contracts)
- Data model: `./data-model.md` (validation rules)
- Testing: `./testing.md` (verification methods)
- Build and deployment: `./build-and-deployment.md` (release verification)
- Roadmap: `../ROADMAP.md` (planned enhancements)
