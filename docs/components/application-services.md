# Application Services

This document describes the "services" provided by this repository to consumers. Since this is an assets-only repository with no runtime code, "application services" refers to the data services, algorithms, and capabilities that consumers implement using repository assets.

## Service Model

### Repository as Service Provider

**Provider**: This repository (data + schema catalogue)
**Consumer**: Universal Name Generator application (or any downstream integrator)
**Contract**: Schema catalogue format, wordlist format, archive layout

**Service boundary**:
```
+-------------------------------------+
| Repository (this repo)              |
| +---------------------------------+ |
| | GenerationSchemas.xml           | | <- Schema catalogue service
| | wordlists/**/*.lst              | | <- Wordlist data service
| | Release archive (.zip)          | | <- Distribution service
| +---------------------------------+ |
+-------------------------------------+
              v (consumed by)
+-------------------------------------+
| Consumer (Universal Name Generator) |
| +---------------------------------+ |
| | Schema parser                   | |
| | Wordlist loader                 | |
| | Randomiser algorithm            | | <- Implemented by consumer
| | Markov algorithm                | | <- Implemented by consumer
| | Filter application              | | <- Implemented by consumer
| +---------------------------------+ |
+-------------------------------------+
```

## Data Services

### Schema Catalogue Service

**Purpose**: Provide machine-readable list of all generation schemas
**Source**: `GenerationSchemas.xml`
**Format**: XML, `ArrayOfGenerationSchemaDataObject` root
**Operations**:
- List all schemas
- Filter by category
- Filter by algorithm
- Lookup by ID
- Resolve schema string to parameters + paths

**Consumer implementation**:
```python
import xml.etree.ElementTree as ET

def load_schemas(path="GenerationSchemas.xml"):
    tree = ET.parse(path)
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

def get_schema_by_id(schemas, schema_id):
    return next((s for s in schemas if s["id"] == schema_id), None)

def get_schemas_by_category(schemas, category):
    return [s for s in schemas if s["category"] == category]
```

### Wordlist Data Service

**Purpose**: Provide name/word entries for generation
**Source**: `wordlists/**/*.lst`
**Format**: UTF-8 plain text, one entry per line
**Operations**:
- Load all entries from a file
- Load entries from multiple files (union)
- Count entries
- Check membership

**Consumer implementation**:
```python
import os

def load_wordlist(path, base_dir="wordlists"):
    """Load entries from a single .lst file."""
    full_path = os.path.join(base_dir, path + ".lst")
    with open(full_path, "r", encoding="utf-8") as f:
        return [line.strip() for line in f if line.strip()]

def load_wordlists(paths, base_dir="wordlists"):
    """Load and union entries from multiple .lst files."""
    entries = []
    for path in paths:
        entries.extend(load_wordlist(path, base_dir))
    return entries

def count_entries(path, base_dir="wordlists"):
    return len(load_wordlist(path, base_dir))
```

### Filter List Service

**Purpose**: Provide exclusion lists for generation
**Source**: Referenced by `FilterlistPath` in schema
**Format**: Same as wordlist (`.lst`, UTF-8, one per line)
**Operations**:
- Load filter entries
- Apply filter to candidate set
- Check if entry is filtered

**Consumer implementation**:
```python
def load_filters(filter_path, base_dir="wordlists"):
    """Load filter entries."""
    full_path = os.path.join(base_dir, filter_path + ".lst")
    with open(full_path, "r", encoding="utf-8") as f:
        return set(line.strip() for line in f if line.strip())

def apply_filter(entries, filters):
    """Remove filtered entries from candidate set."""
    return [e for e in entries if e not in filters]
```

## Generation Services (Consumer-Implemented)

### Randomiser Service

**Purpose**: Uniform random selection from wordlist entries
**Algorithm**: Select random entry, concatenate until length constraints met
**Parameters**: minimum_length, maximum_length, paths
**Implementation**: Consumer responsibility

**Reference implementation**:
```python
import random

def generate_randomiser(schema, base_dir="wordlists"):
    """Generate a name using the randomiser algorithm."""
    entries = load_wordlists(schema["paths"], base_dir)
    minimum_length = schema["minimum_length"]
    maximum_length = schema["maximum_length"]

    while True:
        # Select random entries and concatenate
        parts = [random.choice(entries) for _ in range(random.randint(1, 3))]
        name = "".join(parts)
        if minimum_length <= len(name) <= maximum_length:
            return name
```

### Markov Service

**Purpose**: Generate names using n-gram chain model
**Algorithm**: Build n-gram chains from training data, generate by chaining
**Parameters**: order, minimum_length, maximum_length, paths
**Implementation**: Consumer responsibility

**Reference implementation**:
```python
import random
from collections import defaultdict

def build_markov_chains(entries, order=4):
    """Build n-gram transition table from entries."""
    chains = defaultdict(list)
    for entry in entries:
        padded = " " * order + entry + " "
        for i in range(len(padded) - order):
            state = padded[i:i+order]
            next_char = padded[i+order]
            chains[state].append(next_char)
    return chains

def generate_markov(schema, base_dir="wordlists"):
    """Generate a name using the markov algorithm."""
    entries = load_wordlists(schema["paths"], base_dir)
    order = schema["order"]
    minimum_length = schema["minimum_length"]
    maximum_length = schema["maximum_length"]

    chains = build_markov_chains(entries, order)

    while True:
        state = " " * order
        result = []
        for _ in range(maximum_length + order):
            if state not in chains:
                break
            next_char = random.choice(chains[state])
            if next_char == " ":
                if minimum_length <= len(result) <= maximum_length:
                    return "".join(result)
                break
            result.append(next_char)
            state = state[1:] + next_char
```

## Composition Services

### Schema Resolution Service

**Purpose**: Parse schema string into structured parameters
**Input**: Raw schema string from XML
**Output**: Algorithm, parameters, paths

**Implementation**:
```python
import re

def parse_schema_string(schema_str):
    """Parse {algorithm,params,paths} schema string."""
    inner = schema_str.strip("{}")
    parts = [p.strip() for p in inner.split(",")]

    algorithm = parts[0]
    paths = parts[-1].split("|")

    if algorithm == "randomiser":
        return {
            "algorithm": "randomiser",
            "minimum_length": int(parts[2]),
            "maximum_length": int(parts[3]),
            "paths": paths,
        }
    elif algorithm == "markov":
        return {
            "algorithm": "markov",
            "order": int(parts[1]),
            "minimum_length": int(parts[2]),
            "maximum_length": int(parts[3]),
            "paths": paths,
        }
    raise ValueError(f"Unknown algorithm: {algorithm}")
```

### Generation Orchestration Service

**Purpose**: Coordinate schema resolution, data loading, filtering, generation
**Input**: Schema ID
**Output**: Generated name

**Implementation**:
```python
def generate_name(schema_id, schemas, base_dir="wordlists"):
    """End-to-end name generation for a schema ID."""
    # 1. Resolve schema
    schema = get_schema_by_id(schemas, schema_id)
    if not schema:
        raise ValueError(f"Unknown schema: {schema_id}")

    # 2. Parse schema string
    params = parse_schema_string(schema["schema"])

    # 3. Load wordlists
    entries = load_wordlists(params["paths"], base_dir)

    # 4. Apply filter if present
    if schema["filterlist_path"]:
        filters = load_filters(schema["filterlist_path"], base_dir)
        entries = apply_filter(entries, filters)

    # 5. Generate
    if params["algorithm"] == "randomiser":
        return generate_randomiser({**params, "paths": entries})
    elif params["algorithm"] == "markov":
        return generate_markov({**params, "paths": entries})
```

## Service Quality Attributes

| Attribute | Target | Current State |
|-----------|--------|---------------|
| Availability | 99.9% (GitHub) | GitHub SLA |
| Durability | Permanent (git history) | Git-backed |
| Consistency | Schema IDs stable | Enforced by review |
| Latency | N/A (static assets) | Download speed only |
| Scalability | GitHub CDN | Unlimited reads |

## Service Boundaries

### What Repository Provides
- Schema catalogue (XML)
- Wordlist data (.lst files)
- Filter lists (.lst files)
- Release archives (ZIP)
- Version history (git tags)

### What Consumer Implements
- Schema parsing
- Wordlist loading
- Randomiser algorithm
- Markov algorithm
- Filter application
- Name generation orchestration
- UI presentation

### What Neither Provides
- Runtime API (no HTTP endpoint)
- Database (no query service)
- Authentication (public data)
- Real-time updates (release-based)

## Related Documentation

- Architecture: `../architecture.md` (algorithms, schema format)
- Data model: `../data-model.md` (entities)
- API reference: `../api-reference/INDEX.md` (schema catalogue)
- Flows: `../flows/` (end-to-end usage)
- Quick start: `../quick-start.md` (consumer integration)
