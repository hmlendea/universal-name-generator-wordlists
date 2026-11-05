# API Reference

This document indexes the API reference for the Universal Name Generator -- Wordlists repository. Since this is an assets-only repository with no runtime code, there is no HTTP API. The "API" is the data contract exposed through `GenerationSchemas.xml` and the `wordlists/` directory tree.

## API Surface

The repository exposes a data API with the following operations:

| Operation | Description | Document |
|-----------|-------------|----------|
| List schemas | Enumerate all generation schemas | [List Schemas](list-schemas.md) |
| Get schema | Retrieve a single schema by ID | [Get Schema](get-schema.md) |
| List categories | Enumerate schema categories | [List Categories](list-categories.md) |
| List wordlists | Enumerate wordlist files | [List Wordlists](list-wordlists.md) |
| Get wordlist | Retrieve entries from a wordlist file | [Get Wordlist](get-wordlist.md) |
| Generate name | Produce a name from a schema | [Generate Name](generate-name.md) |

## Data Contract

### Schema Catalogue

- **Source**: `GenerationSchemas.xml`
- **Format**: XML, `ArrayOfGenerationSchemaDataObject` root
- **Entries**: 328 `GenerationSchemaDataObject` elements
- **Stability**: `Id` values are immutable; changing an `Id` is a breaking change

### Wordlist Data

- **Source**: `wordlists/**/*.lst`
- **Format**: UTF-8 plain text, one entry per line
- **Files**: 388 `.lst` files
- **Stability**: Entries are append-only; removal is a breaking change

## Access Patterns

| Pattern | Method | Use Case |
|---------|--------|----------|
| Release archive download | HTTPS | Production consumption |
| Git clone | Git | Development, contribution |
| GitHub API | REST | Metadata, automation |
| Direct file access | Filesystem | Embedded deployment |

## Error Handling

All operations may raise errors. See [Error Handling](../error-handling.md) for the complete error taxonomy.

## Related Documentation

- [Data Model](../data-model.md) -- Entity definitions
- [API Usage Examples](../api-usage-examples.md) -- Code examples
- [Quick Start](../quick-start.md) -- Minimal integration
- [Integrations](../integrations.md) -- Distribution and access methods
