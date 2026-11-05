# Integration Models

This document describes how consumers integrate with this repository's data assets. Since this is an assets-only repository, integration models are patterns for consuming the data.

## Integration Patterns

### Pattern 1: Embedded Distribution

**Description**: Ship release archive with consumer application.

**Workflow**:
```
1. Consumer build: Download release archive
2. Consumer build: Extract to app bundle
3. Consumer runtime: Load from bundle
```

**Use cases**: Desktop apps, mobile apps, game installers

**Implementation**:
```bash
# Build script
VERSION=3.2.0
wget https://github.com/.../releases/download/v${VERSION}/universal-name-generator-wordlists_${VERSION}.zip
unzip -q universal-name-generator-wordlists_${VERSION}.zip -d src/main/resources/data/
```

```python
# Runtime
DATA_PATH = "data/universal-name-generator-wordlists_3.2.0"
schemas = load_schemas(f"{DATA_PATH}/GenerationSchemas.xml")
```

**Pros**: Offline, version-locked, no runtime deps
**Cons**: Larger bundle, manual updates

### Pattern 2: Runtime Download

**Description**: Download and extract at application startup or on demand.

**Workflow**:
```
1. App startup: Check for updates
2. If new version: Download release archive
3. Extract to cache directory
4. Load from cache
```

**Use cases**: Launchers, auto-updating apps, web apps with service workers

**Implementation**:
```python
import requests, zipfile, os

CACHE_DIR = os.path.expanduser("~/.cache/universal-name-generator")
LATEST_URL = "https://api.github.com/repos/.../releases/latest"

def ensure_latest():
    latest = requests.get(LATEST_URL).json()
    version = latest["tag_name"][1:]  # strip 'v'
    asset = next(a for a in latest["assets"] if a["name"].endswith(".zip"))

    local_zip = f"{CACHE_DIR}/universal-name-generator-wordlists_{version}.zip"
    if not os.path.exists(local_zip):
        download(asset["browser_download_url"], local_zip)
        extract(local_zip, CACHE_DIR)

    return f"{CACHE_DIR}/universal-name-generator-wordlists_{version}"
```

**Pros**: Always current, smaller initial bundle
**Cons**: Network dependency, startup latency

### Pattern 3: Build-Time Compilation

**Description**: Compile wordlists into application binary.

**Workflow**:
```
1. Build: Download release archive
2. Build: Parse schemas and wordlists
3. Build: Generate code/data structures
4. Build: Embed in binary
```

**Use cases**: Go (go:embed), Rust (include_bytes!), C# (embedded resources), WASM

**Implementation (Go)**:
```go
//go:embed data/wordlists/**/*.lst data/GenerationSchemas.xml
var dataFS embed.FS

func loadSchemas() ([]Schema, error) {
    content, _ := dataFS.ReadFile("data/GenerationSchemas.xml")
    return parseXML(content)
}
```

**Pros**: Fastest runtime, no filesystem access, tamper-proof
**Cons**: Rebuild for updates, larger binary

### Pattern 4: Server-Side Proxy

**Description**: Backend serves data to frontend clients.

**Workflow**:
```
1. Backend: Download and cache release
2. Backend: Expose /schemas, /wordlists/:path endpoints
3. Frontend: Fetch schemas, generate names via API
```

**Use cases**: Web apps, multi-tenant, rate-limited generation

**Implementation**:
```python
# Backend (FastAPI example)
@app.get("/schemas")
def get_schemas():
    return load_schemas(CACHED_DATA_PATH)

@app.get("/generate/{schema_id}")
def generate(schema_id: str):
    schema = get_schema(schema_id)
    return {"name": generate_name(schema)}
```

**Pros**: Centralized updates, client-light, analytics
**Cons**: Server required, latency, scaling

### Pattern 5: Git Submodule / Subtree

**Description**: Include repository as submodule in consumer repo.

**Workflow**:
```
1. Consumer: git submodule add https://github.com/.../universal-name-generator-wordlists.git data/wordlists
2. Consumer: git submodule update --remote (for updates)
3. Consumer build: Use data/wordlists/ directly
```

**Use cases**: Tightly coupled projects, monorepos

**Pros**: Version-controlled together, easy local edits
**Cons**: Repo bloat, submodule complexity

## Data Access Interfaces

### File System Interface (All Patterns)

```python
class FileSystemDataSource:
    def __init__(self, base_path: str):
        self.base_path = base_path

    def load_schema(self, schema_id: str) -> Schema:
        # Parse GenerationSchemas.xml, find by ID
        pass

    def load_wordlist(self, path: str) -> List[str]:
        # Read wordlists/{path}.lst
        pass

    def load_filter(self, path: str) -> Set[str]:
        # Read wordlists/{path}.lst
        pass
```

### HTTP Interface (Pattern 4)

```yaml
# OpenAPI spec
paths:
  /schemas:
    get:
      summary: List all generation schemas
      responses:
        '200':
          description: Array of schemas
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Schema'

  /schemas/{id}:
    get:
      summary: Get schema by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string

  /generate/{schema_id}:
    post:
      summary: Generate name using schema
      parameters:
        - name: schema_id
          in: path
          required: true
      responses:
        '200':
          description: Generated name
          content:
            application/json:
              schema:
                type: object
                properties:
                  name:
                    type: string
```

### In-Memory Interface (Patterns 1, 2, 3)

```python
class InMemoryDataSource:
    def __init__(self, schemas: List[Schema], wordlists: Dict[str, List[str]], filters: Dict[str, Set[str]]):
        self.schemas = {s.id: s for s in schemas}
        self.wordlists = wordlists
        self.filters = filters

    def get_schema(self, schema_id: str) -> Schema:
        return self.schemas[schema_id]

    def get_entries(self, paths: List[str]) -> List[str]:
        entries = []
        for path in paths:
            entries.extend(self.wordlists.get(path, []))
        return entries

    def get_filters(self, path: str) -> Set[str]:
        return self.filters.get(path, set())
```

## Schema Resolution

### Universal Resolution Algorithm

All patterns use the same resolution logic:

```python
def resolve_schema(schema: Schema, data_source: DataSource) -> ResolvedSchema:
    # 1. Parse schema string
    params = parse_schema_string(schema.schema)

    # 2. Load wordlist entries
    entries = data_source.get_entries(params.paths)

    # 3. Load and apply filter
    if schema.filterlist_path:
        filters = data_source.get_filters(schema.filterlist_path)
        entries = [e for e in entries if e not in filters]

    return ResolvedSchema(
        id=schema.id,
        name=schema.name,
        category=schema.category,
        algorithm=params.algorithm,
        params=params,
        entries=entries,
    )
```

## Version Management

### Consumer Version Strategy

| Strategy | Description | Update Mechanism |
|----------|-------------|------------------|
| Pin exact | Lock to specific version | Manual bump |
| Pin major | Accept patch/minor | Auto patch, manual minor |
| Latest | Always use newest | Auto on startup/build |
| Channel | Stable/beta/nightly | Config-driven |

### Version Discovery

```python
def get_latest_version() -> str:
    """Query GitHub API for latest release."""
    resp = requests.get("https://api.github.com/repos/.../releases/latest")
    return resp.json()["tag_name"][1:]  # strip 'v'

def get_version_asset_url(version: str) -> str:
    """Get download URL for specific version."""
    resp = requests.get(f"https://api.github.com/repos/.../releases/tags/v{version}")
    for asset in resp.json()["assets"]:
        if asset["name"].endswith(".zip"):
            return asset["browser_download_url"]
    raise ValueError("No zip asset found")
```

## Error Handling in Integration

### Common Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Network timeout | HTTP timeout | Retry with backoff |
| Invalid archive | Zip corruption check | Re-download |
| Missing schema | Schema ID not found | Fallback to default |
| Missing wordlist | File not found | Log error, skip schema |
| Parse error | XML/JSON exception | Log, use cached version |

### Resilience Patterns

```python
import time
from functools import wraps

def retry(max_attempts=3, backoff=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(backoff ** attempt)
            return wrapper
        return decorator

@retry(max_attempts=3)
def download_with_retry(url, dest):
    # Download with validation
    pass
```

## Testing Integration

### Contract Tests

```python
def test_schema_catalogue_loads():
    """Verify schema catalogue parses correctly."""
    schemas = load_schemas("GenerationSchemas.xml")
    assert len(schemas) == 328  # Known count
    assert all(s.id for s in schemas)
    assert all(s.schema.startswith("{") for s in schemas)

def test_wordlist_paths_exist():
    """Verify all schema paths reference existing files."""
    schemas = load_schemas("GenerationSchemas.xml")
    for schema in schemas:
        paths = parse_schema_string(schema.schema).paths
        for path in paths:
            assert os.path.exists(f"wordlists/{path}.lst"), f"Missing: {path}"

def test_generation_produces_valid_output():
    """Verify generation algorithms produce names within constraints."""
    schema = get_schema("arabic-persons-male")
    resolved = resolve_schema(schema, data_source)
    for _ in range(100):
        name = generate(resolved)
        assert 4 <= len(name) <= 32
```

## Related Documentation

- Application services: `./application-services.md` (service implementations)
- Architecture: `../architecture.md` (schema format, algorithms)
- Data model: `../data-model.md` (entities)
- Quick start: `../quick-start.md` (consumer setup)
- Build and deployment: `../build-and-deployment.md` (release assets)
