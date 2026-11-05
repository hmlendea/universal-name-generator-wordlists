# Troubleshooting

This document provides diagnostic procedures and resolution steps for common issues encountered when using the Universal Name Generator -- Wordlists repository.

## Diagnostic Procedures

### Verify Repository Integrity

```bash
# Check XML well-formedness
xmllint --noout GenerationSchemas.xml
# Expected: no output (success)

# Check all wordlist files are UTF-8
find wordlists -name "*.lst" -exec file {} \; | grep -v "UTF-8"
# Expected: no output (all UTF-8)

# Check for trailing whitespace
grep -rn ' $' wordlists/ | head
# Expected: no output (no trailing whitespace)

# Check for empty lines
grep -rn '^$' wordlists/ | head
# Expected: no output (no empty lines)

# Check for CRLF line endings
file wordlists/**/*.lst | grep CRLF
# Expected: no output (all LF)
```

### Verify Schema Catalogue

```bash
# Count schemas
grep -c '<GenerationSchemaDataObject>' GenerationSchemas.xml
# Expected: 328

# List all schema IDs
xmllint --xpath '//Id/text()' GenerationSchemas.xml 2>/dev/null | tr ' ' '\n' | sort

# Check for duplicate IDs
xmllint --xpath '//Id/text()' GenerationSchemas.xml 2>/dev/null | tr ' ' '\n' | sort | uniq -d
# Expected: no output (no duplicates)

# List categories
xmllint --xpath '//Category/text()' GenerationSchemas.xml 2>/dev/null | tr ' ' '\n' | sort -u
```

### Verify Wordlist References

```bash
# Extract all paths from schema strings
grep -oP '(?<=\{)(randomiser|markov)[^}]+' GenerationSchemas.xml | \
  sed 's/.*,\([0-9]*\),\([0-9]*\),//' | tr '|' '\n' | sort -u > /tmp/schema_paths.txt

# Check each path exists
while read -r path; do
  if [ ! -f "wordlists/${path}.lst" ]; then
    echo "MISSING: wordlists/${path}.lst"
  fi
done < /tmp/schema_paths.txt
# Expected: no output (all paths exist)
```

### Verify Release Archive

```bash
# Download latest release
curl -L -o release.zip \
  https://github.com/hmlendea/universal-name-generator-wordlists/releases/latest/download/universal-name-generator-wordlists.zip

# List archive contents
unzip -l release.zip | head -20

# Verify structure (wordlists/ and GenerationSchemas.xml at root)
unzip -l release.zip | grep -E '^\s+[0-9]+.*wordlists/$'
unzip -l release.zip | grep -E '^\s+[0-9]+.*GenerationSchemas.xml$'

# Extract and verify
unzip -q release.zip -d /tmp/release-check
ls /tmp/release-check/
# Expected: wordlists/  GenerationSchemas.xml
```

## Common Issues

### Issue: XML Parse Error

| Attribute | Value |
|-----------|-------|
| **Symptom** | `xml.etree.ElementTree.ParseError: not well-formed` |
| **Cause** | Corrupted `GenerationSchemas.xml` |
| **Severity** | Critical |
| **Detection** | `xmllint --noout GenerationSchemas.xml` returns non-zero |

**Resolution**:
1. Re-download the release archive
2. Verify download integrity (HTTPS, correct file size)
3. If using git clone, run `git status` to check for local modifications
4. If modified, restore with `git checkout GenerationSchemas.xml`

### Issue: File Not Found for Wordlist Path

| Attribute | Value |
|-----------|-------|
| **Symptom** | `FileNotFoundError: wordlists/real/arabic/persons/males.lst` |
| **Cause** | Incorrect extraction path or missing archive contents |
| **Severity** | High |
| **Detection** | `ls wordlists/real/arabic/persons/males.lst` fails |

**Resolution**:
1. Verify archive extracted correctly (no wrapper directory)
2. Check that `wordlists/` is at extraction root
3. Re-extract if nested: `unzip release.zip -d correct_location/`
4. Verify path is `wordlists/`-relative without `.lst` extension in schema

### Issue: Empty Generation Result

| Attribute | Value |
|-----------|-------|
| **Symptom** | Generation returns empty string or raises exception |
| **Cause** | Filter removed all entries, or length bounds too restrictive |
| **Severity** | Medium |
| **Detection** | Check entry count after filter application |

**Resolution**:

```python
# Diagnose
entries = load_wordlists(params["paths"])
print(f"Loaded: {len(entries)} entries")

if schema["filterlist_path"]:
    filters = load_filters(schema["filterlist_path"])
    filtered = apply_filter(entries, filters)
    print(f"After filter: {len(filtered)} entries")
    if not filtered:
        print("Filter removed all entries; consider relaxing filter")

# Check length bounds
valid = [e for e in entries if params["minimum_length"] <= len(e) <= params["maximum_length"]]
print(f"Within bounds: {len(valid)} entries")
```

### Issue: Markov Generation Hangs

| Attribute | Value |
|-----------|-------|
| **Symptom** | Generation never returns (infinite loop) |
| **Cause** | Markov chain dead-end with no valid transitions |
| **Severity** | Medium |
| **Detection** | Generation exceeds reasonable time (seconds) |

**Resolution**:
1. Implement attempt limits (see [Application Services](components/application-services.md))
2. Add timeout wrapper
3. Fallback to randomiser or random entry

```python
import signal

class TimeoutException(Exception):
    pass

def generate_with_timeout(generator_func, timeout_seconds=5, *args, **kwargs):
    def handler(signum, frame):
        raise TimeoutException("Generation timed out")

    signal.signal(signal.SIGALRM, handler)
    signal.alarm(timeout_seconds)
    try:
        result = generator_func(*args, **kwargs)
    finally:
        signal.alarm(0)
    return result
```

### Issue: Generated Names Too Short or Too Long

| Attribute | Value |
|-----------|-------|
| **Symptom** | Names outside expected length range |
| **Cause** | Incorrect length bounds in schema or user override |
| **Severity** | Low |
| **Detection** | Compare generated name length to schema bounds |

**Resolution**:
1. Verify schema string bounds: `{randomiser, ,4,32,path}` means 4-32 characters
2. Check user parameter overrides
3. Validate bounds before generation:

```python
if not (params["minimum_length"] <= len(name) <= params["maximum_length"]):
    print(f"Warning: name length {len(name)} outside bounds "
          f"{params['minimum_length']}-{params['maximum_length']}")
```

### Issue: Encoding Errors (UnicodeDecodeError)

| Attribute | Value |
|-----------|-------|
| **Symptom** | `UnicodeDecodeError: 'ascii' codec can't decode byte` |
| **Cause** | File opened without UTF-8 encoding |
| **Severity** | High |
| **Detection** | Error occurs during wordlist loading |

**Resolution**:
1. Always open files with `encoding="utf-8"`:

```python
with open(path, "r", encoding="utf-8") as f:
    entries = [line.strip() for line in f if line.strip()]
```

2. Verify file encoding: `file -i wordlists/real/arabic/persons/males.lst`
3. Expected: `charset=utf-8`

### Issue: Duplicate Schema IDs

| Attribute | Value |
|-----------|-------|
| **Symptom** | Lookup by ID returns wrong schema |
| **Cause** | Duplicate `Id` values in `GenerationSchemas.xml` |
| **Severity** | High |
| **Detection** | See diagnostic procedure above |

**Resolution**:
1. Identify duplicates: `xmllint --xpath '//Id/text()' GenerationSchemas.xml | tr ' ' '\n' | sort | uniq -d`
2. Report to maintainers (see [SECURITY.md](../SECURITY.md))
3. Consumer workaround: use index-based lookup instead of ID-based

### Issue: Release Archive Has Wrapper Directory

| Attribute | Value |
|-----------|-------|
| **Symptom** | Extraction creates nested directory structure |
| **Cause** | Archive built with wrapper directory |
| **Severity** | Medium |
| **Detection** | `unzip -l release.zip` shows wrapper prefix |

**Resolution**:
1. Extract and move contents up one level
2. Or extract with path stripping: `unzip release.zip "wordlists/*" "GenerationSchemas.xml"`
3. Report to maintainers if archive structure is incorrect

### Issue: Schema String Parse Failure

| Attribute | Value |
|-----------|-------|
| **Symptom** | `ValueError: Unknown algorithm` or index error |
| **Cause** | Malformed schema string or parser bug |
| **Severity** | High |
| **Detection** | Parser raises exception on valid schema |

**Resolution**:
1. Print raw schema string for debugging
2. Verify format: `{algorithm,params,min,max,paths}`
3. Check for unexpected whitespace or characters
4. Test parser against known-good schema strings

```python
# Debug
schema_str = schema["schema"]
print(f"Raw: {repr(schema_str)}")
print(f"Stripped: {repr(schema_str.strip('{}'))}")
print(f"Parts: {schema_str.strip('{}').split(',')}")
```

### Issue: Filter Not Applied

| Attribute | Value |
|-----------|-------|
| **Symptom** | Filtered names still appear in generation |
| **Cause** | Filter path not resolved or filter not loaded |
| **Severity** | Medium |
| **Detection** | Check `filterlist_path` handling in consumer code |

**Resolution**:
1. Verify `FilterlistPath` is present in schema XML
2. Verify filter file exists: `ls wordlists/<filterlist_path>.lst`
3. Verify filter is loaded and applied before generation
4. Check filter file format (one entry per line, UTF-8)

## Issue Resolution Matrix

| Symptom | Likely Cause | First Check | Resolution Document |
|---------|--------------|-------------|---------------------|
| XML parse error | Corrupted file | `xmllint --noout` | This document |
| File not found | Wrong extraction | `ls wordlists/` | This document |
| Empty result | Filter too aggressive | Entry count after filter | [Error Handling](error-handling.md) |
| Generation hangs | Markov dead-end | Attempt limits | [Application Services](components/application-services.md) |
| Wrong length | Bounds error | Schema string | [Data Model](data-model.md) |
| Encoding error | Missing UTF-8 | `file -i` | This document |
| Duplicate IDs | Data error | `uniq -d` check | [Invariants](invariants.md) |
| Wrapper directory | Archive issue | `unzip -l` | [Build and Deployment](build-and-deployment.md) |
| Parse failure | Format mismatch | Debug print | [API Usage Examples](api-usage-examples.md) |
| Filter ignored | Path resolution | Filter file exists | [Architecture](architecture.md) |

## When to Report

Report to maintainers when:
- Repository data is corrupted (XML malformed, missing files)
- Schema definitions reference non-existent paths
- Release archive structure is incorrect
- Duplicate schema IDs exist
- Wordlist content contains malicious entries

Do not report:
- Consumer implementation bugs
- Generated name quality issues
- Feature requests for new wordlists (use PR instead)
- Performance issues in consumer code

## Related Documentation

- [Error Handling](error-handling.md) -- Error categories and responses
- [Invariants](invariants.md) -- Repository invariants
- [Data Model](data-model.md) -- Expected data formats
- [Build and Deployment](build-and-deployment.md) -- Release structure
- [SECURITY.md](../SECURITY.md) -- Vulnerability reporting
