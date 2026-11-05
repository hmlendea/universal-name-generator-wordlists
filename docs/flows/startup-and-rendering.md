# Startup and Rendering Flow

This document describes the end-to-end flow of a consumer application starting up, loading repository data, and rendering available schemas. Since this repository has no runtime, this flow describes consumer-side behaviour.

## Flow Overview

```
+-------------+    +--------------+    +-------------+    +--------------+    +-------------+
| App Start   | -> | Load Archive | -> | Parse XML   | -> | Load         | -> | Render      |
|             |    | / Data       |    | Schemas     |    | Wordlists    |    | Schema List |
+-------------+    +--------------+    +-------------+    +--------------+    +-------------+
```

## Phase 1: Application Start

### Trigger
- User launches application
- Web page loads
- Service starts

### Actions
1. Determine data source location:
   - Embedded (Pattern 1/3): Bundle path
   - Downloaded (Pattern 2): Cache directory
   - Server (Pattern 4): Backend URL
2. Initialize data source client
3. Begin loading schema catalogue

### State
- No schemas loaded
- No wordlists loaded
- UI shows loading state

## Phase 2: Load Archive / Data

### Embedded Pattern
```python
# Data already in application bundle
DATA_PATH = "app_data/universal-name-generator-wordlists_3.2.0"
schemas_xml = read_file(f"{DATA_PATH}/GenerationSchemas.xml")
```

### Download Pattern
```python
# Download and extract if not cached
CACHE_DIR = "~/.cache/universal-name-generator"

def ensure_data():
    if not os.path.exists(f"{CACHE_DIR}/GenerationSchemas.xml"):
        download_latest_release(CACHE_DIR)
    return CACHE_DIR
```

### Server Pattern
```python
# Backend serves schemas via API
def fetch_schemas():
    resp = requests.get("https://api.example.com/schemas")
    return resp.json()
```

### Validation
- Archive integrity (ZIP CRC check)
- XML well-formedness
- Required files present

### Error Handling
| Error | Response |
|-------|----------|
| Archive missing | Download or error |
| XML malformed | Error, abort startup |
| Files missing | Error, abort startup |

## Phase 3: Parse Schema Catalogue

### Parse XML
```python
import xml.etree.ElementTree as ET

def parse_schemas(xml_content):
    root = ET.fromstring(xml_content)
    schemas = []
    for obj in root.findall("GenerationSchemaDataObject"):
        schema = {
            "id": obj.findtext("Id"),
            "name": obj.findtext("Name"),
            "category": obj.findtext("Category"),
            "schema_string": obj.findtext("Schema"),
            "filterlist_path": obj.findtext("FilterlistPath"),
        }
        schemas.append(schema)
    return schemas
```

### Parse Schema Strings
```python
def parse_schema_string(schema_str):
    """Parse {algorithm,params,paths} into structured form."""
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
```

### Build In-Memory Index
```python
def build_index(schemas):
    by_id = {s["id"]: s for s in schemas}
    by_category = {}
    for s in schemas:
        by_category.setdefault(s["category"], []).append(s)
    return {"by_id": by_id, "by_category": by_category}
```

### State After Phase 3
- 328 schemas parsed
- Schema strings decomposed
- Index built (by ID, by category)
- Wordlists NOT yet loaded (lazy)

## Phase 4: Load Wordlists (Lazy)

### Lazy Loading Strategy
```python
class WordlistCache:
    def __init__(self, base_path):
        self.base_path = base_path
        self.cache = {}

    def load(self, path):
        if path not in self.cache:
            full_path = os.path.join(self.base_path, "wordlists", path + ".lst")
            with open(full_path, "r", encoding="utf-8") as f:
                self.cache[path] = [line.strip() for line in f if line.strip()]
        return self.cache[path]
```

### Eager Loading (Alternative)
```python
def load_all_wordlists(schemas, base_path):
    """Pre-load all referenced wordlists at startup."""
    cache = {}
    for schema in schemas:
        params = parse_schema_string(schema["schema_string"])
        for path in params["paths"]:
            if path not in cache:
                cache[path] = load_wordlist(path, base_path)
    return cache
```

### Trade-offs

| Strategy | Startup Time | Memory | First-Generation Latency |
|----------|-------------|--------|-------------------------|
| Lazy | Fast | Low | Higher (load on demand) |
| Eager | Slower | High | Zero (pre-loaded) |
| Hybrid | Medium | Medium | Low (load category on select) |

## Phase 5: Render Schema List

### UI Rendering
```javascript
// Web app example
function renderSchemaList(schemas) {
    const categories = groupBy(schemas, s => s.category);

    for (const [category, categorySchemas] of Object.entries(categories)) {
        renderCategoryHeader(category);
        for (const schema of categorySchemas) {
            renderSchemaItem({
                id: schema.id,
                name: schema.name,
                algorithm: parseSchemaString(schema.schema_string).algorithm,
            });
        }
    }
}
```

### Rendered Output
```
+-------------------------------------+
| Universal Name Generator            |
+-------------------------------------+
| v Arabic                            |
|   o Arabic male persons             |
|   o Arabic female persons           |
|   o Arabic locations                |
| v Elven                             |
|   o Elven male persons              |
|   o Elven female persons            |
| v RuneScape                         |
|   o RuneScape demonic names         |
|   ...                               |
+-------------------------------------+
```

### Rendered Metadata
| Field | Source | Display |
|-------|--------|---------|
| Schema name | `Name` | Primary label |
| Category | `Category` | Grouping header |
| Algorithm | Parsed from `Schema` | Badge/icon |
| Entry count | Loaded wordlist | Secondary info |

## Phase 6: Ready State

### State After Startup
```
Application State:
+-- schemas: 328 parsed schemas
+-- index: by_id, by_category
+-- wordlist_cache: {} (empty, lazy)
+-- selected_schema: null
+-- generated_names: []
```

### User Can Now
- Browse schema categories
- Search schemas by name
- Select a schema (triggers wordlist load)
- Generate names (triggers algorithm)

## Error Handling

### Startup Failures

| Failure | Detection | User Experience |
|---------|-----------|-----------------|
| Data source unavailable | Connection error | "Data unavailable, retry" |
| XML parse error | Parser exception | "Corrupt data, reinstall" |
| Missing wordlists | File not found | "Incomplete data, redownload" |
| Out of memory | Allocation failure | "Insufficient memory" |

### Recovery
```python
def startup():
    try:
        data = load_data()
        schemas = parse_schemas(data)
        index = build_index(schemas)
        render(index)
    except DataUnavailableError:
        show_error("Cannot load name data. Check connection and retry.")
    except CorruptDataError:
        show_error("Data is corrupted. Please reinstall.")
    except Exception as e:
        log_error(e)
        show_error("Unexpected error. See logs.")
```

## Performance Characteristics

### Startup Time Budget

| Phase | Target | Notes |
|-------|--------|-------|
| Load archive | <100ms | Embedded; <1s if download |
| Parse XML | <50ms | 328 schemas, small XML |
| Build index | <10ms | In-memory dict |
| Render list | <100ms | UI framework dependent |
| **Total** | **<300ms** | Embedded; <2s with download |

### Memory Footprint

| Component | Size | Notes |
|-----------|------|-------|
| Schema catalogue | ~100KB | 328 entries |
| Wordlists (all) | ~50MB | 388 files, lazy-loaded |
| Wordlists (per schema) | ~10-500KB | Depends on referenced files |
| Generated names | Negligible | Per-request |

## Related Documentation

- Application services: `../components/application-services.md`
- Integration models: `../components/integration-models.md`
- Browser state: `../components/browser-state-and-localisation.md`
- Architecture: `../architecture.md`
- Quick start: `../quick-start.md`
