# Data Model

## Domain Entities

### Wordlist Entry

**Representation**: Single line in a `.lst` file (UTF-8 plain text)

**Properties**:
- `value` (string) -- The candidate name or word fragment
- No metadata, identifiers, or structure beyond the raw string

**Constraints**:
- One entry per line
- No trailing whitespace
- UTF-8 encoding
- No empty lines (by convention)

**Lifecycle**: Append-only by default; entries added via pull request. No automated deletion policy.

### Generation Schema (GenerationSchemaDataObject)

**Representation**: XML element in `GenerationSchemas.xml`

**Properties**:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `Id` | string | Yes | Globally unique, kebab-case identifier (e.g., `arabic-persons-male`) |
| `Name` | string | Yes | Human-readable display name |
| `Category` | string | Yes | Logical grouping (e.g., `Arabic`, `RuneScape`, `Romanian`) |
| `Schema` | string | Yes | Generation algorithm specification |
| `FilterlistPath` | string | No | Repository-relative path to filter list |

**Schema String Format**:

The `Schema` property encodes the generation algorithm and source references:

```
{algorithm,param1,param2,param3,path1|path2|...}
```

**Algorithms**:

| Algorithm | Parameters | Description |
|-----------|------------|-------------|
| `randomiser` | `{randomiser, ,minLength,maxLength,path1|path2|...}` | Random selection from combined wordlists; min/max length constraints |
| `markov` | `{markov,order,minLength,maxLength,path1|path2|...}` | Markov chain generation; order = n-gram size |

**Parameter Details**:
- `order` (markov only) -- N-gram order for chain construction (typically 4)
- `minLength` -- Minimum generated name length
- `maxLength` -- Maximum generated name length
- `pathN` -- Repository-relative paths under `wordlists/` separated by `|`

**Examples**:
```
{randomiser, ,4,32,real/arabic/persons/males|real/arabic/persons/surnames}
{markov,4,24,real/romanian/locations/toponyms}
{randomiser,,4,32,no-mans-sky/gek/persons/neutral}
```

**Id Stability Contract**: `Id` values are stable identifiers for downstream consumers. Changing an `Id` is a breaking change requiring consumer migration.

**Category Grouping**: The `Category` field groups schemas logically (e.g., all Romanian schemas share `Category: Romanian`). This is used by consumers for UI organisation.

### Filter List

**Representation**: Plain text `.lst` file (same format as wordlists)

**Purpose**: Post-generation filtering to exclude undesirable outputs

**Reference**: Referenced via `FilterlistPath` in `GenerationSchemaDataObject`

**Location**: Typically under `wordlists/<universe>/<category>/filters/` (e.g., `real/german/persons/filters`)

## Relationships

```
GenerationSchemaDataObject
    +-- references (via Schema.paths) -> Wordlist Entries (.lst files)
    +-- references (via FilterlistPath) -> Filter List (.lst file)
```

**Cardinality**:
- One `GenerationSchemaDataObject` references 1+ wordlist files
- Multiple schemas may reference the same wordlist file
- Zero or one filter list per schema

## Data Architecture

```mermaid
flowchart LR
    subgraph "Repository"
        LST[.lst files\n(wordlists/)]
        XML[GenerationSchemas.xml]
    end

    LST -->|referenced by| XML
    XML -->|validated by| CI[GitHub Actions]
    LST -->|packaged with| XML
    XML -->|packaged with| LST
    CI -->|produces| Archive[Release zip\n(wordlists/ + GenerationSchemas.xml)]
    Archive -->|downloaded by| Consumer[Downstream generator]
    Consumer -->|embeds at build| Runtime[Generator runtime]
```

## Storage and Representation

| Data Store | Format | Encoding | Size (approx) |
|------------|--------|----------|---------------|
| `wordlists/` | Plain text (`.lst`) | UTF-8 | ~100+ MB (388 files) |
| `GenerationSchemas.xml` | XML | UTF-8 | ~50 KB (328 entries) |
| Release archive | Zip | Binary | ~100+ MB |

## Lifecycle and Consistency

### Wordlist Entries
- **Creation**: Via pull request adding new `.lst` files or appending to existing
- **Modification**: Rare; typically corrections to existing entries
- **Deletion**: No automated policy; manual removal via PR if needed
- **Consistency**: No cross-file referential integrity enforced; duplicates possible

### Generation Schemas
- **Creation**: Via pull request adding new `GenerationSchemaDataObject` entries
- **Modification**: `Schema` string updates, `FilterlistPath` additions
- **Deletion**: Breaking change; requires consumer coordination
- **Consistency**: `Id` uniqueness enforced by XML structure; path references not validated in CI

### Release Archives
- **Creation**: Automated via `release-asset.yml` on GitHub Release publish
- **Immutability**: Once published, archives are immutable
- **Versioning**: Semantic versioning via Git tags (`vX.Y.Z`)
- **Distribution**: GitHub Releases only; no package managers or mirrors

## Validation Rules

### Structural (Enforced)
- XML well-formedness (`xmllint --noout GenerationSchemas.xml`)
- `.lst` files: UTF-8, no trailing whitespace (`git diff --check`)

### Advisory (Not Enforced in CI)
- Schema path references exist: `xmllint --schema GenerationSchemas.xml wordlists/**/*.lst`
- No duplicate entries within a single `.lst` file
- `Id` uniqueness across all `GenerationSchemaDataObject` elements

### Semantic (Not Automated)
- Generated name quality and appropriateness
- Algorithm parameter suitability
- Filter list effectiveness
- Category grouping consistency
