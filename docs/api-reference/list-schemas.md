# List Schemas

Enumerate all generation schemas from the schema catalogue.

## Operation

| Attribute | Value |
|-----------|-------|
| **Name** | List Schemas |
| **Source** | `GenerationSchemas.xml` |
| **Returns** | List of schema objects |
| **Filtering** | By category, algorithm, or ID substring |
| **Sorting** | By ID, name, or category |

## Schema Object

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Globally unique kebab-case identifier |
| `name` | string | Human-readable display name |
| `category` | string | Logical grouping (universe or language) |
| `schema` | string | Generation algorithm specification string |
| `filterlist_path` | string or null | Repository-relative filter list path |

## Implementation

### Python

```python
import xml.etree.ElementTree as ET

def list_schemas(xml_path="GenerationSchemas.xml"):
    """List all schemas from the catalogue."""
    tree = ET.parse(xml_path)
    root = tree.getroot()
    schemas = []
    for obj in root.findall("GenerationSchemaDataObject"):
        schemas.append({
            "id": obj.findtext("Id"),
            "name": obj.findtext("Name"),
            "category": obj.findtext("Category"),
            "schema": obj.findtext("Schema"),
            "filterlist_path": obj.findtext("FilterlistPath"),
        })
    return schemas
```

### Bash

```bash
# List all schema IDs
xmllint --xpath '//Id/text()' GenerationSchemas.xml 2>/dev/null | tr ' ' '\n'

# List all schema names
xmllint --xpath '//Name/text()' GenerationSchemas.xml 2>/dev/null | tr ' ' '\n'
```

## Filtering

### By Category

```python
def list_schemas_by_category(schemas, category):
    """Filter schemas by category."""
    return [s for s in schemas if s["category"] == category]
```

### By Algorithm

```python
def list_schemas_by_algorithm(schemas, algorithm):
    """Filter schemas by algorithm."""
    return [s for s in schemas if s["schema"].startswith("{" + algorithm + ",")]
```

### By ID Substring

```python
def search_schemas(schemas, query):
    """Search schemas by ID or name substring."""
    query_lower = query.lower()
    return [
        s for s in schemas
        if query_lower in s["id"].lower() or query_lower in s["name"].lower()
    ]
```

## Sorting

```python
# Sort by ID
schemas.sort(key=lambda s: s["id"])

# Sort by name
schemas.sort(key=lambda s: s["name"])

# Sort by category, then name
schemas.sort(key=lambda s: (s["category"], s["name"]))
```

## Response

### Success

Returns a list of schema objects. Example (first two entries):

```python
[
    {
        "id": "arabic-persons-male",
        "name": "Arabic male persons",
        "category": "Arabic",
        "schema": "{randomiser, ,4,32,real/arabic/persons/males|real/arabic/persons/surnames}",
        "filterlist_path": None,
    },
    {
        "id": "elven-persons-male",
        "name": "Elven male persons",
        "category": "Elven",
        "schema": "{markov,4,4,32,elder-scrolls/elven/persons/males}",
        "filterlist_path": None,
    },
]
```

### Empty Result

Returns an empty list when no schemas match the filter. This is not an error.

## Errors

| Error | Cause | Response |
|-------|-------|----------|
| File not found | `GenerationSchemas.xml` missing | Raise `FileNotFoundError` |
| XML malformed | Invalid XML syntax | Raise `ParseError` |
| Encoding error | Non-UTF-8 content | Raise `UnicodeDecodeError` |

## Performance

| Metric | Value |
|--------|-------|
| Schema count | 328 |
| XML file size | ~50 KB |
| Parse time | < 10 ms |
| Memory footprint | < 1 MB |

## Related Documentation

- [Get Schema](get-schema.md) -- Retrieve a single schema
- [List Categories](list-categories.md) -- Enumerate categories
- [Data Model](../data-model.md) -- Schema entity definition
- [Error Handling](../error-handling.md) -- Error taxonomy
