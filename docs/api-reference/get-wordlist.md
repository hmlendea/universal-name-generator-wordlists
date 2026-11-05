# Get Wordlist

Retrieve all entries from a single wordlist file.

## Operation

| Attribute | Value |
|-----------|-------|
| **Name** | Get Wordlist |
| **Source** | `wordlists/<path>.lst` |
| **Parameter** | `path` (string, required) |
| **Returns** | List of entry strings |

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | Yes | Repository-relative path under `wordlists/` (no `.lst` extension) |

## Implementation

### Python

```python
def get_wordlist(path, base_dir="wordlists"):
    """Load entries from a single wordlist file."""
    full_path = os.path.join(base_dir, path + ".lst")
    with open(full_path, "r", encoding="utf-8") as f:
        return [line.strip() for line in f if line.strip()]
```

### Bash

```bash
# Get entries from a wordlist
cat wordlists/real/arabic/persons/males.lst

# Count entries
wc -l wordlists/real/arabic/persons/males.lst

# Sample first 5 entries
head -5 wordlists/real/arabic/persons/males.lst
```

## Response

### Success

Returns a list of entry strings. Example:

```python
[
    "Abdullah",
    "Abdul",
    "Abu",
    "Ahmad",
    "Akbar",
    "Aladdin",
    "Ali",
    "Amir",
    "Anwar",
    "Aziz",
]
```

### Empty Result

Returns an empty list when the file exists but contains no entries. This is not an error.

## Errors

| Error | Cause | Response |
|-------|-------|----------|
| File not found | Path does not exist | Raise `FileNotFoundError` |
| Encoding error | Non-UTF-8 content | Raise `UnicodeDecodeError` |
| Path traversal | Path contains `..` or absolute | Raise `ValueError` (consumer-side validation) |

## Path Resolution

Paths are repository-relative under `wordlists/` without the `.lst` extension:

| Input Path | Resolved File |
|------------|---------------|
| `real/arabic/persons/males` | `wordlists/real/arabic/persons/males.lst` |
| `elder-scrolls/elven/persons/males` | `wordlists/elder-scrolls/elven/persons/males.lst` |
| `no-mans-sky/gek/persons/neutral` | `wordlists/no-mans-sky/gek/persons/neutral.lst` |

**Safety**: Consumers must validate paths to prevent directory traversal:

```python
def resolve_path(relative_path, base_dir="wordlists"):
    """Safely resolve repository-relative path."""
    normalised = os.path.normpath(relative_path)
    if normalised.startswith("..") or os.path.isabs(normalised):
        raise ValueError(f"Unsafe path: {relative_path}")
    return os.path.join(base_dir, normalised + ".lst")
```

## Performance

| Metric | Value |
|--------|-------|
| File count | 388 |
| Average file size | ~250 KB |
| Load time (single file) | < 10 ms |
| Memory (single file) | < 1 MB |

**Optimisation**: Cache loaded wordlists for repeated access. See [State and Persistence](../state-and-persistence.md).

## Related Documentation

- [List Wordlists](list-wordlists.md) -- Enumerate all wordlists
- [Data Model](../data-model.md) -- Wordlist entity definition
- [Error Handling](../error-handling.md) -- Error taxonomy
