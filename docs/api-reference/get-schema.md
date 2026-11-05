# Get Schema

Retrieve a single generation schema by its unique ID.

## Operation

| Attribute | Value |
|-----------|-------|
| **Name** | Get Schema |
| **Source** | `GenerationSchemas.xml` |
| **Parameter** | `schema_id` (string, required) |
| **Returns** | Single schema object or null |

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `schema_id` | string | Yes | Globally unique kebab-case identifier |

## Implementation

### Python

```python
def get_schema(schema_id, xml_path="GenerationSchemas.xml"):
    """Retrieve a single schema by ID."""
    schemas = list_schemas(xml_path)
    return next((s for s in schemas if s["id"] == schema_id), None)
```

### Bash

```bash
# Get schema by ID (requires xmlstarlet or similar)
xmllint --xpath "//GenerationSchemaDataObject[Id='arabic-persons-male']" GenerationSchemas.xml 2>/dev/null
```

## Response

### Success

Returns a schema object:

```python
{
    "id": "arabic-persons-male",
    "name": "Arabic male persons",
    "category": "Arabic",
    "schema": "{randomiser, ,4,32,real/arabic/persons/males|real/arabic/persons/surnames}",
    "filterlist_path": None,
}
```

### Not Found

Returns `None` (Python) or empty output (Bash) when no schema matches the ID. This is not an error.

## Errors

| Error | Cause | Response |
|-------|-------|----------|
| File not found | `GenerationSchemas.xml` missing | Raise `FileNotFoundError` |
| XML malformed | Invalid XML syntax | Raise `ParseError` |
| Encoding error | Non-UTF-8 content | Raise `UnicodeDecodeError` |

## Performance

| Metric | Value |
|--------|-------|
| Lookup time | O(n) linear scan (328 entries) |
| Memory | < 1 MB |

**Optimisation**: Build an in-memory index for repeated lookups:

```python
def build_schema_index(schemas):
    """Build ID-to-schema index."""
    return {s["id"]: s for s in schemas}

# Usage
index = build_schema_index(list_schemas())
schema = index.get("arabic-persons-male")
```

## Related Documentation

- [List Schemas](list-schemas.md) -- Enumerate all schemas
- [Data Model](../data-model.md) -- Schema entity definition
- [Error Handling](../error-handling.md) -- Error taxonomy
