# Testing

## Test Strategy

This repository has **no automated test suite**. It is an assets-only repository with no application code, runtime, or executable components to test.

Verification is performed through **manual checks** and **advisory validation** by contributors and maintainers.

## Verification Methods

### 1. XML Well-Formedness (Structural)

```bash
xmllint --noout GenerationSchemas.xml
```

**Purpose**: Validates XML syntax and structure.
**When**: Before committing schema changes; during PR review.
**Automation**: Not enforced in CI (advisory only).

### 2. Schema Path Reference Validation (Advisory)

```bash
xmllint --schema GenerationSchemas.xml wordlists/**/*.lst
```

**Purpose**: Validates that `Schema` string paths reference existing `.lst` files.
**When**: Before committing schema or wordlist changes.
**Automation**: Not enforced in CI; documented in CONTRIBUTING.md as optional.

### 3. Whitespace and Formatting (Structural)

```bash
git diff --check
```

**Purpose**: Detects trailing whitespace, non-ASCII characters, and line-ending issues.
**When**: Before committing any `.lst` or `.xml` changes.
**Automation**: Not enforced in CI; standard Git practice.

### 4. Duplicate Detection (Content Quality)

**Current state**: No automated duplicate detection.
**Planned**: ROADMAP.md identifies "repository-wide duplicate detection reporting" as planned labour.
**Manual approach**: Contributors should check for duplicates within a single `.lst` file before submitting.

### 5. Release Asset Verification (Post-Release)

```bash
# After release publication
wget <release-asset-url>
unzip -l universal-name-generator-wordlists_X.Y.Z.zip
# Verify: wordlists/ and GenerationSchemas.xml at root
unzip -p universal-name-generator-wordlists_X.Y.Z.zip GenerationSchemas.xml | xmllint --noout -
```

**Purpose**: Confirms release archive structure and XML validity.
**When**: After every release by maintainers.
**Automation**: Workflow success indicates packaging completed; manual verification recommended.

## Test Boundaries

| Boundary | Tested | Not Tested |
|----------|--------|------------|
| XML syntax | ✅ (advisory) | -- |
| Schema path existence | ✅ (advisory) | -- |
| `.lst` formatting | ✅ (advisory) | -- |
| Duplicate entries | ❌ | Within/between files |
| Generation algorithm correctness | ❌ | Semantic output quality |
| Filter list effectiveness | ❌ | Filtering behaviour |
| Consumer integration | ❌ | Downstream build/runtime |
| Archive layout | ✅ (manual post-release) | -- |

## Coverage Gaps

### Material Gaps (Acknowledged)

1. **No semantic validation** -- Generated name quality, algorithm parameter suitability, and filter effectiveness are not verified.
2. **No cross-reference validation** -- Schema `Schema` paths vs. actual `.lst` files not enforced in CI.
3. **No duplicate detection** -- Duplicate entries within or across `.lst` files not detected.
4. **No consumer integration tests** -- Downstream generator build/runtime not tested against releases.
5. **No regression detection** -- Changes to existing schemas/wordlists not validated against prior behaviour.

### Accepted Gaps (By Design)

1. **No runtime tests** -- Repository has no runtime; generation logic executes in downstream consumers.
2. **No algorithm implementation tests** -- Algorithms (randomiser, markov) implemented in consumers, not here.
3. **No UI/UX tests** -- No user interface in this repository.

## Contributor Validation Checklist

Before submitting a PR, contributors should:
- [ ] Run `xmllint --noout GenerationSchemas.xml` (if schema modified)
- [ ] Run `xmllint --schema GenerationSchemas.xml wordlists/**/*.lst` (if schema or wordlists modified)
- [ ] Run `git diff --check` (all changes)
- [ ] Verify new `.lst` files: UTF-8, one entry per line, no trailing whitespace
- [ ] Verify new schema `Id` is unique and follows kebab-case convention
- [ ] Verify `Schema` paths reference existing `.lst` files
- [ ] Check for obvious duplicates in new/updated `.lst` files

## Maintainer Release Checklist

Before publishing a release, maintainers should:
- [ ] Verify all PRs merged to `master` since last release
- [ ] Create Git tag `vX.Y.Z` following Semantic Versioning
- [ ] Create GitHub Release from tag
- [ ] Monitor `release-asset.yml` workflow completion
- [ ] Download and inspect release asset structure
- [ ] Verify `GenerationSchemas.xml` in archive is well-formed
- [ ] Confirm asset size is reasonable (> 0 bytes, < 2 GB)

## Future Test Enhancements (Per ROADMAP.md)

| Enhancement | Priority | Description |
|-------------|----------|-------------|
| CI-enforced XML validation | High | Add `xmllint --noout` to workflow |
| CI-enforced path validation | High | Add schema path reference check to workflow |
| Duplicate detection | Medium | Script to report duplicates within/between `.lst` files |
| Canonical formatting | Medium | Document and enforce `.lst` formatting conventions |
| Consumer smoke test | Low | Test Universal Name Generator API build against release |

## Related Documentation

- Contributing guidelines: `../CONTRIBUTING.md`
- Roadmap: `../ROADMAP.md`
- Architecture: `./architecture.md` (validation gaps section)
- Data model: `./data-model.md` (validation rules section)
