# Change Guide

This document describes how to safely modify common areas of the repository. Follow these procedures to maintain invariants and avoid breaking changes.

## Adding a New Wordlist Dataset

### Prerequisites
- Determine universe (existing or new) and category
- Prepare UTF-8 plain text file with one entry per line, no trailing whitespace

### Procedure

1. **Choose location**:
   - Existing universe: `wordlists/<universe>/<category>/<name>.lst`
   - New universe: `wordlists/<new-universe>/<category>/<name>.lst`
   - New category: `wordlists/<universe>/<new-category>/<name>.lst`

2. **Naming conventions**:
   - Universe: lowercase, hyphen-separated (e.g., `age-of-wonders`)
   - Category: plural noun (e.g., `persons`, `locations`, `units`)
   - File: descriptive, lowercase, hyphen-separated (e.g., `males.lst`, `toponyms.lst`)

3. **Create file**:
   ```bash
   mkdir -p wordlists/<universe>/<category>
   # Edit wordlists/<universe>/<category>/<name>.lst
   ```

4. **Validate format**:
   ```bash
   # Check UTF-8, no trailing whitespace
   git diff --check wordlists/<universe>/<category>/<name>.lst
   ```

5. **Add schema entry** (required for usability):
   - Follow "Adding a Generation Schema" procedure below

6. **Submit PR** with clear description of universe, category, and source

## Adding a Generation Schema

### Prerequisites
- Wordlist files exist and are referenced correctly
- Unique `Id` determined (kebab-case, universe/category prefix)

### Procedure

1. **Determine schema properties**:
   - `Id`: globally unique, kebab-case (e.g., `arabic-persons-male`)
   - `Name`: human-readable (e.g., `Arabic male persons`)
   - `Category`: logical grouping (e.g., `Arabic`, `RuneScape`)
   - `Schema`: algorithm string (see format below)
   - `FilterlistPath`: optional, if filter list exists

2. **Schema string format**:
   ```
   Randomiser: {randomiser, ,minLength,maxLength,path1|path2|...}
   Markov:     {markov,order,minLength,maxLength,path1|path2|...}
   ```
   - Paths: repository-relative under `wordlists/` (no `.lst` extension)
   - Example: `real/arabic/persons/males|real/arabic/persons/surnames`

3. **Add to GenerationSchemas.xml**:
   ```xml
   <GenerationSchemaDataObject>
       <Id>your-schema-id</Id>
       <Name>Your Schema Name</Name>
       <Category>Your Category</Category>
       <Schema>{randomiser, ,4,32,path1|path2}</Schema>
       <FilterlistPath>optional/filter/path</FilterlistPath>
   </GenerationSchemaDataObject>
   ```

4. **Validate**:
   ```bash
   xmllint --noout GenerationSchemas.xml
   xmllint --schema GenerationSchemas.xml wordlists/**/*.lst
   ```

5. **Submit PR** with schema rationale and algorithm choice explanation

## Modifying an Existing Schema

### Non-Breaking Changes (Safe)

| Change | Procedure |
|--------|-----------|
| Update `Name` | Edit `Name` element; no consumer impact |
| Update `Category` | Edit `Category` element; UI grouping only |
| Add `FilterlistPath` | Add element; consumers handle gracefully |
| Adjust `Schema` parameters (min/max length, order) | Edit `Schema` string; test with consumer |

### Breaking Changes (Require Coordination)

| Change | Impact | Procedure |
|--------|--------|-----------|
| Change `Id` | Consumer build fails | **Do not do this**. Deprecate old, add new. |
| Remove `GenerationSchemaDataObject` | Consumer build fails | **Do not do this**. Mark deprecated in docs. |
| Change `Schema` paths | Consumer references missing files | Coordinate with consumers; update atomically |
| Change algorithm | Consumer parser may fail | Add new schema with new algorithm; deprecate old |

### Deprecation Pattern

1. Add new schema with new `Id` (e.g., `arabic-persons-male-v2`)
2. Document deprecation in PR/issue
3. Wait for consumers to migrate
4. Remove old schema in future major version

## Moving or Renaming Wordlist Files

### Impact
Breaks all schemas referencing the old path.

### Procedure (If Absolutely Necessary)

1. **Coordinate with consumers** -- Announce breaking change
2. **Atomic PR** -- Single PR with:
   - File move/rename
   - All affected `Schema` string updates in `GenerationSchemas.xml`
   - All affected `FilterlistPath` updates
3. **Validate** -- Full `xmllint --schema` validation
4. **Major version** -- Release as major version bump (X.0.0)

### Preferred Alternative
- Add new file at new location
- Add new schema referencing new location
- Deprecate old schema (see above)
- Remove old file in future cleanup

## Adding a Filter List

### Procedure

1. **Create filter file**:
   ```bash
   mkdir -p wordlists/<universe>/<category>/filters
   # Edit wordlists/<universe>/<category>/filters.lst
   ```

2. **Reference in schema**:
   ```xml
   <FilterlistPath>wordlists/<universe>/<category>/filters</FilterlistPath>
   ```

3. **Validate** -- Filter file follows `.lst` format conventions

## Updating Release Workflow

### Procedure

1. **Test in fork** -- Modify `.github/workflows/release-asset.yml` in fork
2. **Create test release** -- Tag `v0.0.0-test` in fork, verify workflow
3. **Verify archive** -- Download and inspect structure
4. **PR to main** -- Submit with test evidence

### Common Modifications

| Change | Risk | Notes |
|--------|------|-------|
| Add files to archive | Medium | Must maintain root layout invariant |
| Change archive naming | High | Breaks consumer download logic |
| Add validation steps | Low | Improves quality; test thoroughly |
| Update action versions | Low | Pin major versions; test workflow |

## Version Bumping

### Semantic Versioning

| Change Type | Version Bump | Example |
|-------------|--------------|---------|
| New wordlists/schemas (additive) | Patch (Z) | 3.2.0 -> 3.2.1 |
| Schema parameter adjustments | Patch (Z) | 3.2.0 -> 3.2.1 |
| New algorithms (additive) | Minor (Y) | 3.2.0 -> 3.3.0 |
| Breaking schema changes | Major (X) | 3.2.0 -> 4.0.0 |
| Archive layout changes | Major (X) | 3.2.0 -> 4.0.0 |

### Release Procedure

1. Ensure all changes merged to `master`
2. Determine version per above
3. Create tag: `git tag vX.Y.Z && git push origin vX.Y.Z`
4. Create GitHub Release from tag
5. Verify workflow completes
6. Verify asset structure

## PR Review Checklist

### For Wordlist Changes
- [ ] UTF-8 encoding
- [ ] One entry per line
- [ ] No trailing whitespace (`git diff --check`)
- [ ] No empty lines
- [ ] Descriptive file name
- [ ] Correct directory placement
- [ ] Schema entry added/updated

### For Schema Changes
- [ ] Unique `Id` (kebab-case)
- [ ] `Name`, `Category`, `Schema` present
- [ ] `Schema` paths exist
- [ ] `FilterlistPath` exists (if present)
- [ ] XML well-formed (`xmllint --noout`)
- [ ] Schema validation passes (`xmllint --schema`)

### For All Changes
- [ ] No code files introduced
- [ ] Documentation updated if needed
- [ ] ROADMAP.md consulted for alignment
- [ ] Breaking changes identified and communicated

## Emergency Procedures

### Bad Release Published

1. Delete GitHub Release (removes asset)
2. Delete Git tag: `git push origin --delete vX.Y.Z`
3. Fix issue
4. Re-release with same or new version

### Schema Corruption in Release

1. Delete release + tag
2. Fix `GenerationSchemas.xml` on `master`
3. Re-release

### Consumer Reports Breakage

1. Assess if breaking change was intentional
2. If unintentional: emergency fix + patch release
3. If intentional: coordinate migration timeline
4. Document in ROADMAP.md or release notes

## Related Documentation

- Architecture: `./architecture.md` (extension points, compatibility)
- Invariants: `./invariants.md` (what must remain true)
- Data model: `./data-model.md` (schema format detail)
- Testing: `./testing.md` (validation commands)
- Roadmap: `../ROADMAP.md` (planned work)
