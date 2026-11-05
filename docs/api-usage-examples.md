# API Usage Examples

This document provides concrete examples of consuming this repository's data assets. Since this is an assets-only repository with no HTTP API, "API usage" refers to programmatic consumption of the schema catalogue and wordlist files.

## Example 1: List All Schemas (Python)

```python
import xml.etree.ElementTree as ET

def list_all_schemas(xml_path="GenerationSchemas.xml"):
    """List all generation schemas with their properties."""
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

# Usage
schemas = list_all_schemas()
print(f"Total schemas: {len(schemas)}")
for s in schemas[:5]:
    print(f"  {s['id']}: {s['name']} ({s['category']})")
```

**Output**:
```
Total schemas: 328
  arabic-persons-male: Arabic male persons (Arabic)
  arabic-persons-female: Arabic female persons (Arabic)
  arabic-locations-toponyms: Arabic location toponyms (Arabic)
  ...
```

## Example 2: Filter Schemas by Category (Python)

```python
def get_schemas_by_category(schemas, category):
    """Return schemas matching a category."""
    return [s for s in schemas if s["category"] == category]

# Usage
schemas = list_all_schemas()
arabic_schemas = get_schemas_by_category(schemas, "Arabic")
print(f"Arabic schemas: {len(arabic_schemas)}")
for s in arabic_schemas:
    print(f"  {s['id']}")
```

## Example 3: Parse Schema String (Python)

```python
import re

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

# Usage
schema_str = "{randomiser, ,4,32,real/arabic/persons/males|real/arabic/persons/surnames}"
params = parse_schema_string(schema_str)
print(params)
# {'algorithm': 'randomiser', 'minimum_length': 4, 'maximum_length': 32,
#  'paths': ['real/arabic/persons/males', 'real/arabic/persons/surnames']}
```

## Example 4: Load Wordlist Entries (Python)

```python
import os

def load_wordlist(path, base_dir="wordlists"):
    """Load entries from a .lst file."""
    full_path = os.path.join(base_dir, path + ".lst")
    with open(full_path, "r", encoding="utf-8") as f:
        return [line.strip() for line in f if line.strip()]

def load_wordlists(paths, base_dir="wordlists"):
    """Load and union entries from multiple .lst files."""
    entries = []
    for path in paths:
        entries.extend(load_wordlist(path, base_dir))
    return entries

# Usage
entries = load_wordlists(["real/arabic/persons/males", "real/arabic/persons/surnames"])
print(f"Total entries: {len(entries)}")
print(f"Sample: {entries[:5]}")
```

## Example 5: Generate Name with Randomiser (Python)

```python
import random

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

# Usage
entries = load_wordlists(["real/arabic/persons/males", "real/arabic/persons/surnames"])
name = generate_randomiser(entries, minimum_length=4, maximum_length=32)
print(f"Generated: {name}")
# Output: Generated: Abdullah ibn Rashid
```

## Example 6: Generate Name with Markov (Python)

```python
import random
from collections import defaultdict

def build_markov_chains(entries, order=4):
    """Build n-gram transition table."""
    chains = defaultdict(list)
    for entry in entries:
        padded = " " * order + entry + " "
        for i in range(len(padded) - order):
            state = padded[i:i+order]
            next_char = padded[i+order]
            chains[state].append(next_char)
    return chains

def generate_markov(entries, order=4, minimum_length=4, maximum_length=32):
    """Generate a name using markov chain."""
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

# Usage
entries = load_wordlists(["elder-scrolls/elven/persons/males"])
name = generate_markov(entries, order=4, minimum_length=4, maximum_length=32)
print(f"Generated: {name}")
# Output: Generated: Aelindor
```

## Example 7: Apply Filter List (Python)

```python
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

# Usage
entries = load_wordlists(["real/arabic/persons/males"])
filters = load_filters("real/arabic/persons/filters")
filtered = apply_filter(entries, filters)
print(f"Before: {len(entries)}, After: {len(filtered)}")
```

## Example 8: End-to-End Generation (Python)

```python
def generate_name(schema_id, xml_path="GenerationSchemas.xml", base_dir="wordlists"):
    """Generate a name for a given schema ID."""
    # 1. Find schema
    schemas = list_all_schemas(xml_path)
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

# Usage
name = generate_name("arabic-persons-male")
print(f"Generated: {name}")
```

## Example 9: List All Categories (Bash)

```bash
# List all schema categories
xmllint --xpath '//Category/text()' GenerationSchemas.xml | tr ' ' '\n' | sort -u

# List all schema IDs
xmllint --xpath '//Id/text()' GenerationSchemas.xml | tr ' ' '\n'

# List schemas by category
xmllint --xpath '//GenerationSchemaDataObject[Category="Arabic"]/Id/text()' GenerationSchemas.xml

# List schemas using markov
xmllint --xpath '//GenerationSchemaDataObject[contains(Schema, "markov")]/Id/text()' GenerationSchemas.xml
```

## Example 10: Validate Schema References (Bash)

```bash
# Check all schema paths exist
xmllint --xpath '//Schema/text()' GenerationSchemas.xml | \
  grep -oP '(?<=,)[a-z0-9/-]+(?=})' | tr '|' '\n' | sort -u | \
  while read path; do
    if [ ! -f "wordlists/${path}.lst" ]; then
      echo "MISSING: $path"
    fi
  done

# Check all filter paths exist
xmllint --xpath '//FilterlistPath/text()' GenerationSchemas.xml | \
  tr ' ' '\n' | sort -u | \
  while read path; do
    if [ ! -f "${path}.lst" ]; then
      echo "MISSING FILTER: $path"
    fi
  done
```

## Example 11: C# Consumer (Universal Name Generator)

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Xml.Linq;

public class GenerationSchema
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Category { get; set; }
    public string Schema { get; set; }
    public string FilterlistPath { get; set; }
}

public class SchemaLoader
{
    public static List<GenerationSchema> LoadSchemas(string xmlPath)
    {
        var doc = XDocument.Load(xmlPath);
        return doc.Root
            .Elements("GenerationSchemaDataObject")
            .Select(e => new GenerationSchema
            {
                Id = e.Element("Id")?.Value,
                Name = e.Element("Name")?.Value,
                Category = e.Element("Category")?.Value,
                Schema = e.Element("Schema")?.Value,
                FilterlistPath = e.Element("FilterlistPath")?.Value,
            })
            .ToList();
    }
}

public class WordlistLoader
{
    public static List<string> LoadWordlist(string path, string baseDir = "wordlists")
    {
        var fullPath = Path.Combine(baseDir, path + ".lst");
        return File.ReadAllLines(fullPath)
            .Select(line => line.Trim())
            .Where(line => !string.IsNullOrEmpty(line))
            .ToList();
    }
}

// Usage
var schemas = SchemaLoader.LoadSchemas("GenerationSchemas.xml");
var arabic = schemas.Where(s => s.Category == "Arabic").ToList();
var entries = WordlistLoader.LoadWordlist("real/arabic/persons/males");
```

## Example 12: JavaScript/TypeScript Consumer (Web)

```typescript
interface GenerationSchema {
  id: string;
  name: string;
  category: string;
  schema: string;
  filterlistPath?: string;
}

async function loadSchemas(url: string): Promise<GenerationSchema[]> {
  const response = await fetch(url);
  const xmlText = await response.text();
  const doc = new DOMParser().parseFromString(xmlText, "application/xml");

  return Array.from(doc.querySelectorAll("GenerationSchemaDataObject")).map(obj => ({
    id: obj.querySelector("Id")?.textContent ?? "",
    name: obj.querySelector("Name")?.textContent ?? "",
    category: obj.querySelector("Category")?.textContent ?? "",
    schema: obj.querySelector("Schema")?.textContent ?? "",
    filterlistPath: obj.querySelector("FilterlistPath")?.textContent ?? undefined,
  }));
}

async function loadWordlist(path: string): Promise<string[]> {
  const response = await fetch(`wordlists/${path}.lst`);
  const text = await response.text();
  return text.split("\n").map(l => l.trim()).filter(l => l.length > 0);
}

// Usage
const schemas = await loadSchemas("GenerationSchemas.xml");
const arabicSchemas = schemas.filter(s => s.category === "Arabic");
const entries = await loadWordlist("real/arabic/persons/males");
```

## Example 13: Download and Extract Release (Bash)

```bash
#!/bin/bash
# Download latest release and extract

REPO="<owner>/universal-name-generator-wordlists"
API_URL="https://api.github.com/repos/${REPO}/releases/latest"

# Get latest release info
LATEST=$(curl -s "$API_URL")
VERSION=$(echo "$LATEST" | jq -r '.tag_name' | sed 's/^v//')
ASSET_URL=$(echo "$LATEST" | jq -r '.assets[] | select(.name | endswith(".zip")) | .browser_download_url')

# Download and extract
wget -q "$ASSET_URL" -O "wordlists_${VERSION}.zip"
unzip -q "wordlists_${VERSION}.zip" -d "data/"

echo "Extracted version ${VERSION} to data/"
ls data/wordlists/ | head
```

## Example 14: Count Entries Per Universe (Bash)

```bash
# Count .lst files per universe
for dir in wordlists/*/; do
    universe=$(basename "$dir")
    count=$(find "$dir" -name "*.lst" | wc -l)
    echo "$universe: $count files"
done

# Count total entries across all wordlists
find wordlists -name "*.lst" -exec cat {} + | wc -l
```

## Example 15: Find Duplicate Entries (Bash)

```bash
# Find duplicates within a file
sort wordlists/real/arabic/persons/males.lst | uniq -d

# Find duplicates across all files (advisory)
find wordlists -name "*.lst" -exec cat {} + | sort | uniq -d | head -20
```

## Related Documentation

- Architecture: `./architecture.md` (schema string format, algorithms)
- Data model: `./data-model.md` (entity structure)
- API reference: `./api-reference/INDEX.md` (schema catalogue)
- Components: `./components/` (service implementations)
- Flows: `./flows/` (end-to-end workflows)
- Quick start: `./quick-start.md` (getting started)
