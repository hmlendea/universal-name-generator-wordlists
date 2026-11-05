# Logging

This document describes logging considerations for this repository and its consumers. Since this is an assets-only repository with no runtime code, there is no logging framework. Logging is a consumer-side responsibility.

## Repository State

### No Runtime Logging

| Aspect | State |
|--------|-------|
| Logging framework | None (no runtime) |
| Log files | None |
| Log levels | N/A |
| Log output | N/A |
| Telemetry | None |
| Analytics | None |

### Why No Logging

- **Assets-only**: Repository contains data files, not executable code
- **No runtime**: No process executes; nothing to log
- **No services**: No server, daemon, or background process
- **Static distribution**: Data distributed via GitHub Releases; no request logging

## CI/CD Logging

### GitHub Actions Workflow Logs

The `release-asset.yml` workflow produces logs accessible via GitHub Actions UI.

**Log contents**:
| Step | Log Output |
|------|-----------|
| Checkout | Clone progress, commit SHA |
| Version extraction | Extracted version string |
| Zip packaging | Files added to archive |
| Asset upload | Upload progress, asset URL |

**Access**:
- GitHub Actions tab -> Workflow run -> Jobs -> Steps
- Logs retained per GitHub retention policy (default 90 days)

**Log characteristics**:
- Not persisted in repository
- Not searchable via repository search
- Accessible to repository collaborators
- May contain `GITHUB_TOKEN` references (redacted by GitHub)

### Workflow Log Example
```
Run actions/checkout@v4
Syncing repository: <owner>/universal-name-generator-wordlists
...
Run version extraction
Extracted version: 3.2.0
...
Run zip packaging
  adding: wordlists/ (stored 0%)
  adding: wordlists/real/ (stored 0%)
  ...
  adding: GenerationSchemas.xml (deflated 72%)
...
Run softprops/action-gh-release@v2
  Upload asset: universal-name-generator-wordlists_3.2.0.zip
  Asset uploaded successfully
```

## Consumer-Side Logging

### What Consumers Should Log

Since consumers implement generation, they own all runtime logging.

#### Startup Logging
```python
logger.info("Loading schema catalogue", path=xml_path)
logger.info("Loaded schemas", count=len(schemas))
logger.info("Loading wordlists", base_dir=base_dir)
```

#### Schema Resolution Logging
```python
logger.debug("Resolving schema", schema_id=schema_id)
logger.debug("Parsed schema string", algorithm=params["algorithm"], paths=params["paths"])
logger.debug("Loaded entries", count=len(entries))
if filters:
    logger.debug("Applied filters", removed=raw_count - filtered_count)
```

#### Generation Logging
```python
logger.debug("Generating name", schema_id=schema_id, algorithm=params["algorithm"])
logger.debug("Generated name", name=name, length=len(name))
```

#### Error Logging
```python
logger.error("Schema not found", schema_id=schema_id)
logger.error("Wordlist missing", path=missing_path)
logger.error("Schema parse error", schema_id=schema_id, error=str(e))
logger.warning("Filter load failed, proceeding unfiltered", path=filter_path, error=str(e))
logger.warning("Generation failed, using fallback", schema_id=schema_id, error=str(e))
```

### Recommended Log Levels

| Level | Use Case | Example |
|-------|----------|---------|
| ERROR | Unrecoverable failure | Schema catalogue unparseable |
| WARNING | Degraded operation | Filter missing, generation fallback |
| INFO | Major milestones | Schemas loaded, generation complete |
| DEBUG | Detailed tracing | Schema parsing, wordlist loading |
| TRACE | Verbose detail | Individual entry selection |

### What Consumers Should NOT Log

| Avoid | Reason |
|-------|--------|
| Generated names at INFO+ | Privacy (names may be PII in some contexts) |
| Full wordlist contents | Noise, potential sensitivity |
| Schema strings at ERROR | Already in catalogue; log schema ID instead |
| File paths with user data | Path traversal information disclosure |

### Log Structure Recommendations

```python
# Structured logging (recommended)
logger.info("schema_resolved",
    schema_id="arabic-persons-male",
    algorithm="randomiser",
    entry_count=500,
    filter_count=12,
)

# Avoid unstructured logging
logger.info(f"Resolved schema arabic-persons-male with 500 entries and 12 filters")
```

## Data Access Logging

### Repository Access (GitHub)

GitHub provides access logs for repository events (not exposed to maintainers directly):

| Event | Logged By | Accessible To |
|-------|-----------|---------------|
| Clone | GitHub | Repository owners (audit log) |
| Release download | GitHub | Repository owners (traffic stats) |
| API request | GitHub | Repository owners (traffic stats) |
| PR creation | GitHub | Repository collaborators |
| Issue creation | GitHub | Repository collaborators |

### Consumer Access Patterns

Consumers should log their own data access:

```python
logger.debug("wordlist_loaded", path="real/arabic/persons/males", entries=250)
logger.debug("wordlist_cached", path="real/arabic/persons/males")
logger.debug("schema_generated", schema_id="arabic-persons-male", duration_ms=2)
```

## Audit Logging

### Repository Audit Trail

Git provides the audit trail for repository changes:

| Audit Need | Git Mechanism |
|------------|---------------|
| Who changed a wordlist | `git log -- wordlists/path/file.lst` |
| When schema was added | `git log -- GenerationSchemas.xml` |
| What changed in release | `git diff v3.1.0 v3.2.0` |
| Who approved PR | GitHub PR review history |

### Consumer Audit Trail

Consumers should maintain their own audit logs for:
- Which repository version they loaded
- Which schemas were used
- Generation request counts (not content)
- Error rates

```python
logger.info("data_version_loaded",
    repository_version="3.2.0",
    schema_count=328,
    wordlist_count=388,
    source="github_release",
)
```

## Security Logging

### Repository Security Events

| Event | Logging | Response |
|-------|---------|----------|
| Unauthorised access attempt | GitHub security log | GitHub notification |
| Malicious PR | GitHub audit log | Maintainer review |
| Compromised release | GitHub audit log | Delete release, investigate |

See root `SECURITY.md` for security incident procedures.

### Consumer Security Logging

Consumers should log security-relevant events:

```python
logger.warning("unexpected_schema_format", schema_id=schema_id, schema_string=raw)
logger.warning("path_traversal_attempt", path=suspicious_path)
logger.error("data_integrity_failure", expected_hash=expected, actual_hash=actual)
```

## Logging Invariants

| Invariant | Description |
|-----------|-------------|
| No repository runtime logs | Repository produces no log files |
| CI logs ephemeral | Workflow logs retained by GitHub, not in repo |
| Consumer owns logging | All runtime logging is consumer responsibility |
| No PII in logs | Generated names should not be logged at INFO+ |
| No secrets in logs | No credentials exist in repository; none should appear in consumer logs |

## Future Considerations

### If Logging Framework Is Introduced

If the repository ever adds runtime code (unlikely per design decisions), logging would need:

| Consideration | Recommendation |
|---------------|----------------|
| Framework | Standard for runtime (e.g., Microsoft.Extensions.Logging for .NET) |
| Levels | Structured levels (Trace, Debug, Info, Warning, Error, Critical) |
| Output | Configurable (console, file, structured) |
| Sensitive data | Masking for any user-provided input |
| Correlation | Request IDs for generation operations |

This scenario contradicts the assets-only design decision (see `design-decisions.md`).

## Related Documentation

- Design decisions: `./design-decisions.md` (assets-only decision)
- Build and deployment: `./build-and-deployment.md` (CI/CD logging)
- Error handling: `./error-handling.md` (consumer error logging)
- Application services: `./components/application-services.md` (consumer implementation)
- Security: `./security.md` (security logging)
- Root SECURITY: `../SECURITY.md` (incident procedures)
