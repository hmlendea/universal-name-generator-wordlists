# Generate Name

Produce a generated name from a schema by loading referenced wordlists and applying the generation algorithm.

## Operation

| Attribute | Value |
|-----------|-------|
| **Name** | Generate Name |
| **Source** | Schema catalogue + wordlist files |
| **Parameter** | `schema_id` (string, required) |
| **Returns** | Generated name string |

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `schema_id` | string | Yes | Globally unique kebab-case schema identifier |
| `count` | integer | No | Number of names to generate (default 1, maximum 100) |
| `seed` | integer | No | Random seed for reproducibility |
| `minimum_length` | integer | No | Override minimum length (default from schema) |
| `maximum_length` | integer | No | Override maximum length (default from schema) |

## Implementation

### Python

```python
def generate_name(schema_id, xml_path="GenerationSchemas.xml", base_dir="wordlists"):
    """Generate a name for a given schema ID."""
    # 1. Find schema
    schemas = list_schemas(xml_path)
    schema = next((s for s in schemas if s["id"] == schema_id), None)
    if not schema:
        raise ValueError(f"Unknown schema: {schema_id}")

    # 2. Parse schema string
    params = parse_schema_string(schema["schema"])

    # 3. Load wordlists
    entries = load_wordlists(params["paths"], base_dir)

    # 4. Apply filter
    if schema["filterlist_path"]:
        filters = load_filters(schema["filterlist_path"], base_dir)
        entries = apply_filter(entries, filters)

    # 5. Generate
    if params["algorithm"] == "randomiser":
        return generate_randomiser(entries, params["minimum_length"], params["maximum_length"])
    elif params["algorithm"] == "markov":
        return generate_markov(entries, params["order"], params["minimum_length"], params["maximum_length"])
```

### Bash

```bash
# Generate using Python script
python generate.py arabic-persons-male

# Generate multiple names
for i in {1..5}; do python generate.py arabic-persons-male; done
```

## Algorithms

### Randomiser

Uniform random selection from combined wordlist entries, concatenated until length constraints are met.

| Property | Value |
|----------|-------|
| Selection | Uniform random |
| Concatenation | 1-3 entries joined |
| Length bounds | `minimum_length` to `maximum_length` |
| Attempts | Up to 1000, then fallback |

### Markov

N-gram Markov chain generation trained on referenced wordlists.

| Property | Value |
|----------|-------|
| Order | Typically 4 (from schema) |
| Training | All referenced entries |
| Length bounds | `minimum_length` to `maximum_length` |
| Attempts | Up to 1000, then fallback |

## Response

### Success

Returns a generated name string. Example:

```
Abdullah ibn Rashid
```

### Multiple Names

When `count` > 1, returns a list of name strings:

```python
["Abdullah ibn Rashid", "Omar al-Khalil", "Yusuf ibn Tariq"]
```

## Errors

| Error | Cause | Response |
|-------|-------|----------|
| Unknown schema | `schema_id` not found | Raise `ValueError` |
| File not found | Wordlist path missing | Raise `FileNotFoundError` |
| Empty entries | Filter removed all entries | Warn, disable generation |
| Length bounds | `minimum_length` > `maximum_length` | Raise `ValueError` |
| Algorithm failure | Markov dead-end | Fallback to random entry |
| Timeout | Generation exceeds limit | Raise `TimeoutError` |

## Performance

| Metric | Randomiser | Markov |
|--------|------------|--------|
| Generation time | < 1 ms | < 10 ms |
| Memory (entries) | < 1 MB | < 5 MB |
| Throughput | > 1000/s | > 100/s |

## Related Documentation

- [Application Services](../components/application-services.md) -- Algorithm implementations
- [Data Model](../data-model.md) -- Schema string format
- [Error Handling](../error-handling.md) -- Error taxonomy
- [API Usage Examples](../api-usage-examples.md) -- Complete code examples
