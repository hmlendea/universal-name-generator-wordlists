# Security

This document describes security considerations for the Universal Name Generator -- Wordlists repository. It complements the root [SECURITY.md](../SECURITY.md) with implementation-level detail for contributors and integrators.

## Repository Security Model

### Assets-Only Repository

| Property | Value |
|----------|-------|
| Runtime code | None |
| Executable binaries | None |
| Network services | None |
| Database | None |
| Authentication | None |
| Authorisation | None |
| Secrets management | None |

**Implication**: Traditional application security concerns (injection, authentication bypass, privilege escalation) do not apply to this repository directly. Security concerns centre on data integrity, supply-chain integrity, and consumer-side usage.

## Data Content Considerations

### Wordlist Content

The repository contains 388 wordlist files (`.lst`) with name/word entries sourced from:
- Public domain sources
- Community contributions
- Game/literature wikis (fair use for name generation)
- Real-world language datasets

**Security implications**:

| Concern | Assessment | Mitigation |
|---------|------------|------------|
| Offensive content | Possible in user-sourced data | Consumer-side filtering recommended |
| Personal data | None (names are generic) | No PII in wordlists |
| Copyrighted content | Fair use for name generation | Sources documented per universe |
| Malicious injection | Low (curated, reviewed) | PR review, no automated ingestion |

### Schema Definitions

`GenerationSchemas.xml` contains 328 schema entries defining generation parameters.

**Security implications**:

| Concern | Assessment | Mitigation |
|---------|------------|------------|
| Path traversal | Paths are `wordlists/`-relative | Consumer must validate paths |
| Resource exhaustion | Large wordlists possible | Consumer should limit memory |
| Infinite generation | Markov dead-ends possible | Consumer must implement attempt limits |

## Supply-Chain Integrity

### Release Distribution

Releases are distributed exclusively through GitHub Releases.

**Integrity guarantees**:

| Layer | Mechanism |
|-------|-----------|
| Source integrity | Git commit history (immutable) |
| Build integrity | GitHub Actions (ephemeral runners) |
| Artifact integrity | GitHub Releases (HTTPS, signed by GitHub) |
| Transport integrity | HTTPS download |

**Verification steps for consumers**:

```bash
# 1. Verify download via HTTPS
curl -L -o release.zip https://github.com/hmlendea/universal-name-generator-wordlists/releases/download/v1.0.0/universal-name-generator-wordlists_1.0.0.zip

# 2. Verify archive structure
unzip -l release.zip | head -20
# Should show wordlists/ and GenerationSchemas.xml at root

# 3. Verify no wrapper directory
unzip -l release.zip | grep -E '^[^/]+/$' | wc -l
# Should be 1 (wordlists/) not 2+
```

### GPL v3.0 Licensing

The repository is licensed under GNU GPL v3.0.

**Security implications**:

| Aspect | Implication |
|--------|-------------|
| Copyleft | Derivative works must be GPL v3.0 compatible |
| Source distribution | Consumers must provide source or offer |
| Patent grant | Contributors grant patent licence |
| No warranty | As-is distribution, no liability |

**Consumer responsibilities**:
- Comply with GPL v3.0 if distributing derivative works
- Provide source code or written offer for source
- Preserve licence notices in distributed copies
- State changes if modifying wordlists or schemas

## Consumer-Side Security

### Input Validation

Consumers must validate all repository data before use:

```python
def validate_schema(schema):
    """Validate schema before generation."""
    # Check required fields
    required = ["id", "name", "category", "schema"]
    for field in required:
        if not schema.get(field):
            raise ValueError(f"Missing required field: {field}")

    # Validate schema string format
    if not schema["schema"].startswith("{") or not schema["schema"].endswith("}"):
        raise ValueError("Invalid schema string format")

    # Validate paths are safe
    params = parse_schema_string(schema["schema"])
    for path in params["paths"]:
        if ".." in path or path.startswith("/"):
            raise ValueError(f"Unsafe path: {path}")

    # Validate length bounds
    if params["minimum_length"] < 1 or params["maximum_length"] > 1000:
        raise ValueError("Length bounds out of range")
    if params["minimum_length"] > params["maximum_length"]:
        raise ValueError("Minimum length exceeds maximum length")
```

### Path Resolution Safety

```python
import os

def resolve_path(relative_path, base_dir="wordlists"):
    """Safely resolve repository-relative path."""
    # Normalise path
    normalised = os.path.normpath(relative_path)

    # Prevent directory traversal
    if normalised.startswith("..") or os.path.isabs(normalised):
        raise ValueError(f"Path traversal attempt: {relative_path}")

    # Construct full path
    full_path = os.path.join(base_dir, normalised + ".lst")

    # Verify within base directory
    if not os.path.commonpath([base_dir, full_path]) == base_dir:
        raise ValueError(f"Path escapes base directory: {relative_path}")

    return full_path
```

### Resource Limits

```python
def generate_with_limits(generator_func, *args, max_attempts=1000, max_memory_mb=100, **kwargs):
    """Wrap generation with resource limits."""
    import resource

    # Limit memory (Unix only)
    try:
        resource.setrlimit(resource.RLIMIT_AS, (max_memory_mb * 1024 * 1024, max_memory_mb * 1024 * 1024))
    except (AttributeError, ValueError):
        pass  # Not available on all platforms

    # Limit attempts
    for attempt in range(max_attempts):
        try:
            return generator_func(*args, **kwargs)
        except Exception:
            if attempt == max_attempts - 1:
                raise
            continue

    raise RuntimeError("Generation failed after maximum attempts")
```

## Vulnerability Reporting

See [SECURITY.md](../SECURITY.md) for the official vulnerability reporting process.

**In-scope for this repository**:
- Malicious entries in wordlist files
- Schema definitions causing unsafe generation
- Release archive integrity issues

**Out-of-scope**:
- Downstream consumer vulnerabilities
- Generated content quality issues
- Infrastructure access issues

## Security Checklist for Contributors

- [ ] No personal data in wordlist entries
- [ ] No secrets or credentials in any file
- [ ] Paths in schemas are `wordlists/`-relative
- [ ] No absolute paths or directory traversal in schemas
- [ ] Filter lists follow same path convention
- [ ] XML is well-formed and validates
- [ ] Release workflow produces correct archive structure
- [ ] GPL v3.0 licence header preserved

## Security Checklist for Consumers

- [ ] Download only from official GitHub Releases
- [ ] Verify archive structure after extraction
- [ ] Validate all schema data before use
- [ ] Implement path traversal protection
- [ ] Implement resource limits (memory, attempts)
- [ ] Apply content filtering if needed
- [ ] Comply with GPL v3.0 if distributing
- [ ] Monitor for security advisories

## Related Documentation

- [SECURITY.md](../SECURITY.md) -- Official security policy
- [Data Model](data-model.md) -- Schema and wordlist formats
- [Change Guide](change-guide.md) -- Contribution process
- [Build and Deployment](build-and-deployment.md) -- Release process
- [Licence](../LICENSE) -- GNU GPL v3.0 full text
