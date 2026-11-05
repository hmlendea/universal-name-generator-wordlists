# List Categories

Enumerate all schema categories from the schema catalogue.

## Operation

| Attribute | Value |
|-----------|-------|
| **Name** | List Categories |
| **Source** | `GenerationSchemas.xml` |
| **Returns** | List of category strings |
| **Sorting** | Alphabetical |

## Implementation

### Python

```python
def list_categories(xml_path="GenerationSchemas.xml"):
    """List all unique categories."""
    schemas = list_schemas(xml_path)
    categories = sorted(set(s["category"] for s in schemas))
    return categories
```

### Bash

```bash
# List all categories
xmllint --xpath '//Category/text()' GenerationSchemas.xml 2>/dev/null | tr ' ' '\n' | sort -u
```

## Response

### Success

Returns a list of category strings. Example (first 10):

```python
[
    "Arabic",
    "Asian",
    "Chinese",
    "Dacian",
    "Danish",
    "Dutch",
    "Draconian",
    "Dragon",
    "Dwarven",
    "Elven",
]
```

### Empty Result

Returns an empty list when no schemas exist. This is not an error.

## Errors

| Error | Cause | Response |
|-------|-------|----------|
| File not found | `GenerationSchemas.xml` missing | Raise `FileNotFoundError` |
| XML malformed | Invalid XML syntax | Raise `ParseError` |
| Encoding error | Non-UTF-8 content | Raise `UnicodeDecodeError` |

## Performance

| Metric | Value |
|--------|-------|
| Category count | ~50 |
| Lookup time | O(n) linear scan |

## Related Documentation

- [List Schemas](list-schemas.md) -- Enumerate all schemas
- [Get Schema](get-schema.md) -- Retrieve a single schema
- [Data Model](../data-model.md) -- Schema entity definition
