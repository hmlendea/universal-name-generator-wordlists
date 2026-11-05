# List Wordlists

Enumerate all wordlist files in the repository.

## Operation

| Attribute | Value |
|-----------|-------|
| **Name** | List Wordlists |
| **Source** | `wordlists/**/*.lst` |
| **Returns** | List of wordlist paths |
| **Filtering** | By universe, category, or path substring |

## Implementation

### Python

```python
import os

def list_wordlists(base_dir="wordlists"):
    """List all .lst files under wordlists/."""
    wordlists = []
    for root, dirs, files in os.walk(base_dir):
        for file in files:
            if file.endswith(".lst"):
                full_path = os.path.join(root, file)
                # Repository-relative path without extension
                relative = os.path.relpath(full_path, base_dir)
                relative = relative[:-4]  # Remove .lst
                wordlists.append(relative.replace(os.sep, "/"))
    return sorted(wordlists)
```

### Bash

```bash
# List all wordlist paths (without .lst extension)
find wordlists -name "*.lst" | sed 's|^wordlists/||; s|\.lst$||' | sort

# Count wordlist files
find wordlists -name "*.lst" | wc -l
# Expected: 388
```

## Response

### Success

Returns a list of repository-relative paths (without `.lst` extension). Example (first 10):

```python
[
    "age-of-wonders/archon/locations/settlements",
    "age-of-wonders/draconian/locations/cities",
    "age-of-wonders/draconian/persons/males",
    "age-of-wonders/draconian/persons/females",
    "age-of-wonders/dragon/locations/lairs",
    "age-of-wonders/dwarven/locations/holds",
    "age-of-wonders/dwarven/persons/males",
    "age-of-wonders/elven/locations/realms",
    "age-of-wonders/elven/persons/males",
    "age-of-wonders/elven/persons/females",
]
```

### Empty Result

Returns an empty list when no `.lst` files exist. This is not an error.

## Filtering

### By Universe

```python
def list_wordlists_by_universe(wordlists, universe):
    """Filter wordlists by universe prefix."""
    return [w for w in wordlists if w.startswith(universe + "/")]
```

### By Category

```python
def list_wordlists_by_category(wordlists, category):
    """Filter wordlists by category segment."""
    return [w for w in wordlists if "/" + category + "/" in w]
```

### By Path Substring

```python
def search_wordlists(wordlists, query):
    """Search wordlists by path substring."""
    query_lower = query.lower()
    return [w for w in wordlists if query_lower in w.lower()]
```

## Errors

| Error | Cause | Response |
|-------|-------|----------|
| Directory not found | `wordlists/` missing | Raise `FileNotFoundError` |
| Permission error | Cannot read directory | Raise `PermissionError` |

## Performance

| Metric | Value |
|--------|-------|
| File count | 388 |
| Directory walk time | < 100 ms |
| Memory | < 1 MB |

## Related Documentation

- [Get Wordlist](get-wordlist.md) -- Retrieve entries from a wordlist
- [Data Model](../data-model.md) -- Wordlist entity definition
- [Repository Structure](../repository-structure.md) -- Directory layout
