# Error Handling

This document describes error conditions, their detection, and handling strategies. Since this repository has no runtime code, errors are categorized as data errors (detectable at rest), packaging errors (detectable at release), and consumer errors (encountered during integration).

## Error Categories

### Category 1: Data Errors (At Rest)

Errors in repository assets detectable without runtime execution.

#### XML Malformed

| Attribute | Value |
|-----------|-------|
| **Condition** | `GenerationSchemas.xml` is not well-formed XML |
| **Detection** | `xmllint --noout GenerationSchemas.xml` returns non-zero |
| **Impact** | All consumers fail to parse schema catalogue |
| **Severity** | Critical |
| **Response** | Revert offending change; fix XML syntax; re-release |

**Common causes**:
- Unescaped special characters (`&`, `<`, `>`) in `Name` or `Schema`
- Missing closing tags
- Invalid UTF-8 sequences
- Truncated file

**Prevention**:
- XML-aware editing tools
- Pre-commit validation (planned per ROADMAP.md)
- PR review checklist

#### Schema Path Missing

| Attribute | Value |
|-----------|-------|
| **Condition** | `Schema` string references non-existent `.lst` file |
| **Detection** | `xmllint --schema GenerationSchemas.xml wordlists/**/*.lst` (advisory) |
| **Impact** | Consumer generation fails for that schema |
| **Severity** | High |
| **Response** | Restore file or update schema path; coordinate with consumers |

**Common causes**:
- Wordlist file moved without schema update
- Typo in schema path
- File deleted without schema cleanup

**Prevention**:
- Atomic PRs for file moves + schema updates
- CI path validation (planned)
- Pre-release verification

#### Filter Path Missing

| Attribute | Value |
|-----------|-------|
| **Condition** | `FilterlistPath` references non-existent `.lst` file |
| **Detection** | Manual check or CI validation |
| **Impact** | Consumer filter application fails or silently skips |
| **Severity** | Medium |
| **Response** | Restore filter file or remove `FilterlistPath` element |

#### Duplicate Schema ID

| Attribute | Value |
|-----------|-------|
| **Condition** | Two `GenerationSchemaDataObject` entries share same `Id` |
| **Detection** | `xmllint --xpath '//Id' GenerationSchemas.xml \| grep -oP '(?<=>)[^<]+' \| sort \| uniq -d` |
| **Impact** | Consumer lookup by ID ambiguous |
| **Severity** | High |
| **Response** | Rename one ID; coordinate with consumers |

#### Wordlist Format Violation

| Attribute | Value |
|-----------|-------|
| **Condition** | `.lst` file has trailing whitespace, empty lines, or non-UTF-8 |
| **Detection** | `git diff --check`, `file -i`, `grep '^$'` |
| **Impact** | Parser may include empty entries or fail |
| **Severity** | Medium |
| **Response** | Fix file formatting; re-release |

### Category 2: Packaging Errors (At Release)

Errors in release archive assembly.

#### Archive Structure Violation

| Attribute | Value |
|-----------|-------|
| **Condition** | Archive has wrapper directory or missing root files |
| **Detection** | `unzip -l archive.zip` inspection |
| **Impact** | Consumer extraction path assumptions break |
| **Severity** | Critical |
| **Response** | Delete release; fix workflow; re-release |

**Expected structure**:
```
archive.zip
+-- wordlists/
+-- GenerationSchemas.xml
```

**Violation examples**:
```
archive.zip
+-- universal-name-generator-wordlists-3.2.0/  <- WRONG: wrapper directory
    +-- wordlists/
    +-- GenerationSchemas.xml
```

#### Incomplete Archive

| Attribute | Value |
|-----------|-------|
| **Condition** | Archive missing some `.lst` files or schema catalogue |
| **Detection** | Compare `unzip -l` count vs `find wordlists -name "*.lst" \| wc -l` |
| **Impact** | Consumer missing data for some schemas |
| **Severity** | Critical |
| **Response** | Delete release; verify workflow; re-release |

#### Version Mismatch

| Attribute | Value |
|-----------|-------|
| **Condition** | Archive filename version differs from git tag |
| **Detection** | Compare `vX.Y.Z` tag vs `universal-name-generator-wordlists_X.Y.Z.zip` |
| **Impact** | Consumer version tracking incorrect |
| **Severity** | Medium |
| **Response** | Delete release; fix workflow version extraction; re-release |

### Category 3: Consumer Errors (Runtime)

Errors encountered by consumers during integration.

#### Schema Parse Failure

| Attribute | Value |
|-----------|-------|
| **Condition** | Consumer parser cannot parse schema string |
| **Detection** | Parser exception at runtime |
| **Impact** | Schema unusable in consumer |
| **Severity** | High (consumer-side) |
| **Response** | Consumer: log error, skip schema, report issue. Repository: verify schema format |

**Common causes**:
- Unexpected algorithm name
- Malformed schema string (extra/missing commas)
- Non-integer parameters

#### Wordlist Load Failure

| Attribute | Value |
|-----------|-------|
| **Condition** | Consumer cannot read `.lst` file |
| **Detection** | File not found or read error |
| **Impact** | Generation fails for referencing schemas |
| **Severity** | High (consumer-side) |
| **Response** | Consumer: log error, skip schema. Repository: verify file exists in release |

#### Generation Failure

| Attribute | Value |
|-----------|-------|
| **Condition** | Algorithm produces no valid output |
| **Detection** | Generation timeout or empty result |
| **Impact** | No name generated |
| **Severity** | Medium (consumer-side) |
| **Response** | Consumer: fallback to random entry, log warning |

**Common causes**:
- Filter removed all entries
- minimum_length > maximum_length
- Markov chain dead-end (no valid transitions)
- Empty wordlist file

#### Filter Application Failure

| Attribute | Value |
|-----------|-------|
| **Condition** | Consumer cannot load or apply filter list |
| **Detection** | File not found or parse error |
| **Impact** | Unfiltered generation (may include unwanted entries) |
| **Severity** | Low (consumer-side) |
| **Response** | Consumer: log warning, proceed without filter |

## Error Handling Strategies

### Repository-Side (Prevention)

| Strategy | Implementation | Status |
|----------|---------------|--------|
| XML validation | `xmllint --noout` | Manual (CI planned) |
| Path validation | `xmllint --schema` | Manual (CI planned) |
| Format validation | `git diff --check` | Manual |
| Duplicate detection | `sort \| uniq -d` | Manual (CI planned) |
| Archive verification | `unzip -l` inspection | Manual post-release |
| PR review | Human checklist | Active |

### Consumer-Side (Resilience)

| Strategy | Implementation |
|----------|---------------|
| Fail fast | Abort startup if schema catalogue unparseable |
| Graceful degradation | Skip broken schemas, continue with valid ones |
| Fallback generation | Return random entry if algorithm fails |
| Retry with backoff | Retry downloads on network failure |
| Version pinning | Lock to known-good release version |
| Validation on load | Verify paths exist before generation |
| Error reporting | Log errors with schema ID for debugging |

### Consumer Error Handling Pattern

```python
def safe_generate(schema_id, schemas, data_source):
    """Generate with comprehensive error handling."""
    try:
        # Resolve
        schema = schemas.get(schema_id)
        if not schema:
            return ErrorResult(f"Unknown schema: {schema_id}")

        # Parse
        try:
            params = parse_schema_string(schema["schema"])
        except ParseError as e:
            return ErrorResult(f"Schema parse error: {e}")

        # Load
        try:
            entries = data_source.load_entries(params["paths"])
        except FileNotFoundError as e:
            return ErrorResult(f"Wordlist missing: {e}")

        # Filter
        if schema["filterlist_path"]:
            try:
                filters = data_source.load_filters(schema["filterlist_path"])
                entries = apply_filter(entries, filters)
            except Exception as e:
                log_warning(f"Filter failed, proceeding unfiltered: {e}")

        # Validate
        if not entries:
            return ErrorResult("No entries after filtering")

        # Generate
        try:
            name = generate(params, entries)
            return SuccessResult(name)
        except GenerationError as e:
            log_warning(f"Generation failed, fallback: {e}")
            return SuccessResult(random.choice(entries))

    except Exception as e:
        log_error(f"Unexpected error: {e}")
        return ErrorResult("Unexpected error")
```

## Error Reporting

### Repository Issues

**When to file**: Data errors, packaging errors, schema format issues
**Where**: GitHub Issues on this repository
**Template**:
```
Title: [Data Error] <description>

## Observed
<what you found>

## Expected
<what should be true>

## Reproduction
<steps to reproduce>

## Evidence
<xmllint output, file paths, etc.>
```

### Consumer Issues

**When to file**: Consumer-side errors that may indicate repository data problems
**Where**: Consumer application's issue tracker
**Include**: Schema ID, error message, repository version

## Error Recovery

### Recovery Procedures

| Error | Recovery |
|-------|----------|
| Bad release published | Delete release + tag, fix, re-release |
| Schema corruption | Fix XML, patch release |
| Missing wordlist | Restore file, patch release |
| Consumer parse failure | Consumer update or repository format fix |
| Generation failure | Consumer fallback or data fix |

### Rollback Strategy

```bash
# 1. Delete bad release
gh release delete vX.Y.Z --yes

# 2. Delete bad tag
git push origin --delete vX.Y.Z

# 3. Fix issue on master
# ... make fixes ...

# 4. Re-release (same or new version)
git tag vX.Y.Z
git push origin vX.Y.Z
# Workflow auto-packages and uploads
```

## Related Documentation

- Testing: `./testing.md` (validation methods)
- Invariants: `./invariants.md` (what must remain true)
- Troubleshooting: `./troubleshooting.md` (diagnostic procedures)
- Build and deployment: `./build-and-deployment.md` (release verification)
- Change guide: `./change-guide.md` (emergency procedures)
