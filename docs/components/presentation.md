# Presentation Layer

This document describes how repository data is presented to consumers. Since this is an assets-only repository with no runtime, "presentation" refers to data packaging, formatting, and access patterns.

## Data Presentation Formats

### Raw Wordlist Files (`.lst`)

**Format**: UTF-8 plain text, one entry per line
**Access**: Direct file access in repository or release archive
**Structure**:
```
entry1
entry2
entry3
...
```

**Characteristics**:
- No headers, metadata, or delimiters
- No escaping (entries cannot contain newlines)
- Order is significant for markov algorithm (training sequence)
- Order is irrelevant for randomiser algorithm (uniform selection)

**Consumer presentation**:
```python
# Python example
with open('wordlists/real/arabic/persons/males.lst', 'r', encoding='utf-8') as f:
    names = [line.strip() for line in f if line.strip()]
```

### Generation Schema Catalogue (`GenerationSchemas.xml`)

**Format**: XML with `ArrayOfGenerationSchemaDataObject` root
**Access**: Direct file access or parsed by consumer
**Structure**:
```xml
<ArrayOfGenerationSchemaDataObject>
  <GenerationSchemaDataObject>
    <Id>arabic-persons-male</Id>
    <Name>Arabic male persons</Name>
    <Category>Arabic</Category>
    <Schema>{randomiser, ,4,32,real/arabic/persons/males|real/arabic/persons/surnames}</Schema>
    <FilterlistPath>real/arabic/persons/filters</FilterlistPath>
  </GenerationSchemaDataObject>
  ...
</ArrayOfXMLSchemaDataObject>
```

**Consumer presentation**:
```csharp
// C# example
var schemas = XDocument.Load("GenerationSchemas.xml")
    .Descendants("GenerationSchemaDataObject")
    .Select(e => new {
        Id = e.Element("Id").Value,
        Name = e.Element("Name").Value,
        Category = e.Element("Category").Value,
        Schema = e.Element("Schema").Value,
        FilterlistPath = e.Element("FilterlistPath")?.Value
    });
```

### Release Archive (`.zip`)

**Format**: ZIP archive with fixed root layout
**Access**: GitHub Releases download
**Structure**:
```
universal-name-generator-wordlists_3.2.0.zip
+-- wordlists/
|   +-- a-song-of-ice-and-fire/
|   |   +-- ...
|   +-- real/
|   |   +-- ...
|   +-- ...
+-- GenerationSchemas.xml
```

**Consumer presentation**:
```bash
# Download and extract
wget https://github.com/.../releases/download/v3.2.0/universal-name-generator-wordlists_3.2.0.zip
unzip universal-name-generator-wordlists_3.2.0.zip
# Data ready at ./wordlists/ and ./GenerationSchemas.xml
```

## Schema String Presentation

### Randomiser Format
```
{randomiser, ,minLength,maxLength,path1|path2|...}
```

**Components**:
| Position | Name | Type | Example |
|----------|------|------|---------|
| 1 | Algorithm | Literal | `randomiser` |
| 2 | (Reserved) | Empty | `` |
| 3 | Min length | Integer | `4` |
| 4 | Max length | Integer | `32` |
| 5+ | Paths | Pipe-separated strings | `real/arabic/persons/males|real/arabic/persons/surnames` |

**Consumer parsing**:
```python
import re

def parse_randomiser(schema_str):
    # {randomiser, ,4,32,path1|path2}
    match = re.match(r'\{randomiser\s*,\s*(\d+)\s*,\s*(\d+)\s*,\s*(.+)\}', schema_str)
    if match:
        minimum_length, maximum_length, paths_str = match.groups()
        paths = paths_str.split('|')
        return {
            'algorithm': 'randomiser',
            'minimum_length': int(minimum_length),
            'maximum_length': int(maximum_length),
            'paths': paths
        }
```

### Markov Format
```
{markov,order,minLength,maxLength,path1|path2|...}
```

**Components**:
| Position | Name | Type | Example |
|----------|------|------|---------|
| 1 | Algorithm | Literal | `markov` |
| 2 | Order | Integer | `4` |
| 3 | Min length | Integer | `4` |
| 4 | Max length | Integer | `32` |
| 5+ | Paths | Pipe-separated strings | `real/elven/persons/males|real/elven/persons/females` |

**Consumer parsing**:
```python
def parse_markov(schema_str):
    # {markov,4,4,32,path1|path2}
    match = re.match(r'\{markov\s*,\s*(\d+)\s*,\s*(\d+)\s*,\s*(\d+)\s*,\s*(.+)\}', schema_str)
    if match:
        order, minimum_length, maximum_length, paths_str = match.groups()
        paths = paths_str.split('|')
        return {
            'algorithm': 'markov',
            'order': int(order),
            'minimum_length': int(minimum_length),
            'maximum_length': int(maximum_length),
            'paths': paths
        }
```

## Category Presentation

### Universe Categories

**Real-world languages** (30+):
- Grouped by language family/region in `wordlists/real/`
- Categories: `persons` (males, females, surnames), `locations` (toponyms)
- Example: `real/arabic/persons/males.lst`

**Fictional universes** (15+):
- Game/literature settings in `wordlists/<universe>/`
- Categories vary by universe
- Example: `elder-scrolls/elven/persons/males.lst`

**Special categories**:
- `usernames/` -- Real platform username dumps (18 sources)
- `age-of-wonders-planetfall/companies.lst` -- Flat file at universe root

### Schema Categories

**Grouping dimension**: `Category` field in `GenerationSchemaDataObject`
**Purpose**: UI grouping for schema selection
**Values**: Universe names, language groups, or thematic groups
**Examples**: `Arabic`, `Elven`, `RuneScape`, `StarCraft`, `Warcraft`

**Consumer presentation**:
```python
# Group schemas by category for UI
from collections import defaultdict

categories = defaultdict(list)
for schema in schemas:
    categories[schema['Category']].append(schema)

# Result: {'Arabic': [...], 'Elven': [...], 'RuneScape': [...]}
```

## Filter List Presentation

**Format**: Same as wordlist (`.lst`, UTF-8, one per line)
**Access**: Referenced by `FilterlistPath` in schema
**Purpose**: Exclude entries from generation (e.g., offensive words, duplicates)
**Consumer application**:
```python
def apply_filter(names, filter_path):
    with open(filter_path, 'r', encoding='utf-8') as f:
        filters = set(line.strip() for line in f if line.strip())
    return [n for n in names if n not in filters]
```

## Access Patterns

### Direct Repository Access (Contributors)
```bash
# Browse
ls wordlists/real/arabic/persons/

# Inspect
cat wordlists/real/arabic/persons/males.lst

# Search
grep -r "pattern" wordlists/
```

### Release Archive Access (Consumers)
```bash
# Download specific version
wget https://github.com/.../releases/download/v3.2.0/universal-name-generator-wordlists_3.2.0.zip

# Extract
unzip universal-name-generator-wordlists_3.2.0.zip

# Use
ls wordlists/
cat GenerationSchemas.xml
```

### Programmatic Access (Integrators)
```python
# Parse schema catalogue
schemas = parse_generation_schemas("GenerationSchemas.xml")

# Load wordlists for a schema
def load_wordlists(schema, base_path="wordlists"):
    all_entries = []
    for path in schema['paths']:
        full_path = os.path.join(base_path, path + ".lst")
        with open(full_path, 'r', encoding='utf-8') as f:
            all_entries.extend(line.strip() for line in f if line.strip())
    return all_entries

# Apply filter if present
if schema.get('filterlist_path'):
    entries = apply_filter(entries, os.path.join(base_path, schema['filterlist_path'] + ".lst"))
```

## Presentation Invariants

| Invariant | Description |
|-----------|-------------|
| `.lst` files always UTF-8 | Encoding never changes |
| One entry per line | No multi-line entries |
| No trailing whitespace | Clean lines |
| Schema paths relative to `wordlists/` | No absolute paths |
| Archive root layout fixed | `wordlists/` + `GenerationSchemas.xml` at root |
| Schema IDs stable | Never change once published |
| Category values stable | Used for UI grouping |

## Related Documentation

- Data model: `../data-model.md`
- Architecture: `../architecture.md` (schema string grammar)
- Repository structure: `../repository-structure.md`
- Build and deployment: `../build-and-deployment.md` (archive format)
- API reference: `../api-reference/INDEX.md` (schema catalogue)
