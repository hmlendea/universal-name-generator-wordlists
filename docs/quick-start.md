# Quick Start

This guide gets you from zero to generating names in under five minutes.

## Prerequisites

- Python 3.8 or later
- Git (for cloning)
- Bash shell (for command examples)
- `jq` (for JSON examples, optional)

## Option 1: Download Release (Recommended)

### Step 1: Download Latest Release

Visit the [releases page](https://github.com/hmlendea/universal-name-generator-wordlists/releases) and download `universal-name-generator-wordlists_X.Y.Z.zip`.

### Step 2: Extract Archive

```bash
unzip universal-name-generator-wordlists_1.0.0.zip
cd universal-name-generator-wordlists
```

### Step 3: Verify Contents

```bash
ls -la
# wordlists/  GenerationSchemas.xml
```

## Option 2: Clone Repository

```bash
git clone https://github.com/hmlendea/universal-name-generator-wordlists.git
cd universal-name-generator-wordlists
```

## Generate Your First Name (Python)

Save the following as `generate.py`:

```python
#!/usr/bin/env python3
"""Generate a name from the Universal Name Generator wordlists."""

import os
import random
import xml.etree.ElementTree as ET
from collections import defaultdict


def load_schemas(path="GenerationSchemas.xml"):
    """Load all schemas from XML catalogue."""
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


def parse_schema_string(schema_str):
    """Parse {algorithm,params,paths} into structured dict."""
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


def load_filters(filter_path, base_dir="wordlists"):
    """Load filter entries."""
    full_path = os.path.join(base_dir, filter_path + ".lst")
    with open(full_path, "r", encoding="utf-8") as f:
        return set(line.strip() for line in f if line.strip())


def apply_filter(entries, filters):
    """Remove filtered entries."""
    if not filters:
        return entries
    return [e for e in entries if e not in filters]


def generate_randomiser(entries, minimum_length=4, maximum_length=32):
    """Generate a name using random selection and concatenation."""
    for _ in range(1000):
        parts = [random.choice(entries) for _ in range(random.randint(1, 3))]
        name = "".join(parts)
        if minimum_length <= len(name) <= maximum_length:
            return name
    # Fallback
    candidates = [e for e in entries if minimum_length <= len(e) <= maximum_length]
    return random.choice(candidates) if candidates else random.choice(entries)


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


def generate_markov(entries, order=4, minimum_length=4, maximum_length=32):
    """Generate a name using the markov algorithm."""
    chains = build_markov_chains(entries, order)

    for _ in range(1000):
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

    return random.choice(entries)


def generate_name(schema_id, xml_path="GenerationSchemas.xml", base_dir="wordlists"):
    """Generate a name for a given schema ID."""
    # 1. Find schema
    schemas = load_schemas(xml_path)
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


if __name__ == "__main__":
    import sys
    schema_id = sys.argv[1] if len(sys.argv) > 1 else "arabic-persons-male"
    name = generate_name(schema_id)
    print(name)
```

Run it:

```bash
chmod +x generate.py
./generate.py arabic-persons-male
# Output: Abdullah ibn Rashid
```

## Explore Available Schemas (Bash + jq)

```bash
# List all schema IDs and names
xmllint --xpath '//GenerationSchemaDataObject' GenerationSchemas.xml 2>/dev/null | \
  sed 's/<GenerationSchemaDataObject>/\n/g' | \
  sed -n 's/.*<Id>\([^<]*\)<\/Id>.*<Name>\([^<]*\)<\/Name>.*/\1 \2/p' | \
  head -20

# List all categories
xmllint --xpath '//Category/text()' GenerationSchemas.xml 2>/dev/null | \
  tr ' ' '\n' | sort -u

# Count schemas per category
xmllint --xpath '//Category/text()' GenerationSchemas.xml 2>/dev/null | \
  tr ' ' '\n' | sort | uniq -c | sort -rn
```

## Explore Wordlists

```bash
# Count total wordlist files
find wordlists -name "*.lst" | wc -l
# 388

# Count entries in a specific file
wc -l wordlists/real/arabic/persons/males.lst

# Sample entries
head -5 wordlists/real/arabic/persons/males.lst

# List all universes
ls wordlists/
```

## Next Steps

| Goal | Document |
|------|----------|
| Understand schema string format | [Data Model](data-model.md) |
| Browse all available schemas | [API Reference](api-reference/INDEX.md) |
| Learn generation algorithms | [Application Services](components/application-services.md) |
| Integrate into your application | [API Usage Examples](api-usage-examples.md) |
| Contribute new wordlists | [Change Guide](change-guide.md) |

## Minimal Consumer Checklist

- [ ] Download and extract release archive
- [ ] Verify `wordlists/` and `GenerationSchemas.xml` present
- [ ] Implement schema parser (XML -> dict)
- [ ] Implement schema string parser (`{...}` -> parameters)
- [ ] Implement wordlist loader (`.lst` -> list of strings)
- [ ] Implement filter loader (optional)
- [ ] Implement randomiser algorithm
- [ ] Implement markov algorithm (optional)
- [ ] Test with `arabic-persons-male` schema
- [ ] Handle missing files gracefully

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Forgetting `.lst` extension | File not found | Loader appends `.lst` automatically |
| Not applying filter | Unwanted names generated | Check `FilterlistPath` and apply |
| Empty entries after filter | Generation fails | Fallback to unfiltered or random entry |
| Markov dead-end | Infinite loop | Limit attempts, fallback to random |
| Path resolution error | Wrong file loaded | Paths are `wordlists/`-relative, no extension |
| Encoding issues | Garbled characters | Open files as UTF-8 |

## Related Documentation

- [Repository Overview](repository-overview.md) -- High-level structure
- [Data Model](data-model.md) -- Schema and wordlist formats
- [API Usage Examples](api-usage-examples.md) -- More code examples
- [Change Guide](change-guide.md) -- Adding new data
- [Build and Deployment](build-and-deployment.md) -- Release process
