# State and Persistence

This document describes state management and persistence considerations for the Universal Name Generator -- Wordlists repository and its consumers. Since this is an assets-only repository with no runtime, there is no repository-side state. All state concerns apply to consumer implementations.

## Repository State

### No Runtime State

| Aspect | State |
|--------|-------|
| In-memory state | None |
| Persistent state | None |
| Session state | None |
| Transaction state | None |
| Cache state | None |

### Immutable Data Assets

The repository contains only immutable data assets:

| Asset | Mutability | Persistence |
|-------|------------|-------------|
| `wordlists/**/*.lst` | Immutable (per release) | Git history + release archive |
| `GenerationSchemas.xml` | Immutable (per release) | Git history + release archive |
| Release archives | Immutable | GitHub Releases |

**Implication**: The repository itself has no state that changes between releases. Each release is a complete, self-contained snapshot.

### Git as Audit Trail

Git history serves as the authoritative audit trail:

| Operation | Recorded In |
|-----------|-------------|
| Wordlist additions | Git commits |
| Wordlist modifications | Git commits |
| Schema additions | Git commits |
| Schema modifications | Git commits |
| Releases | Git tags (`vX.Y.Z`) |

**Consumer use**: Consumers can track changes by comparing Git tags or commit ranges.

## Consumer-Side State Patterns

### Schema Selection State

Consumers typically maintain schema selection state:

```python
class SchemaSelector:
    """Manage schema selection state."""

    def __init__(self):
        self.selected_schema_id = None
        self.available_schemas = []
        self.categories = []

    def load_schemas(self, xml_path):
        self.available_schemas = load_schemas(xml_path)
        self.categories = sorted(set(s["category"] for s in self.available_schemas))

    def select_schema(self, schema_id):
        schema = next((s for s in self.available_schemas if s["id"] == schema_id), None)
        if schema:
            self.selected_schema_id = schema_id
            return schema
        raise ValueError(f"Unknown schema: {schema_id}")

    def get_selected(self):
        if self.selected_schema_id:
            return next((s for s in self.available_schemas if s["id"] == self.selected_schema_id), None)
        return None
```

### Generation Parameter State

Consumers maintain mutable generation parameters:

```python
class GenerationParams:
    """Manage mutable generation parameters."""

    def __init__(self, schema_params):
        self.algorithm = schema_params["algorithm"]
        self.minimum_length = schema_params["minimum_length"]
        self.maximum_length = schema_params["maximum_length"]
        self.paths = schema_params["paths"]
        self.order = schema_params.get("order")
        self.filterlist_path = None

        # Mutable user overrides
        self.user_minimum_length = schema_params["minimum_length"]
        self.user_maximum_length = schema_params["maximum_length"]
        self.count = 1
        self.seed = None

    def update_bounds(self, minimum_length=None, maximum_length=None):
        if minimum_length is not None:
            if minimum_length < 1:
                raise ValueError("Minimum length must be >= 1")
            self.user_minimum_length = minimum_length
        if maximum_length is not None:
            if maximum_length > 1000:
                raise ValueError("Maximum length must be <= 1000")
            self.user_maximum_length = maximum_length
        if self.user_minimum_length > self.user_maximum_length:
            raise ValueError("Minimum length cannot exceed maximum length")

    def set_count(self, count):
        if not 1 <= count <= 100:
            raise ValueError("Count must be 1-100")
        self.count = count

    def set_seed(self, seed):
        self.seed = seed
```

### Generation History State

Consumers may persist generation history:

```python
import json
import os
from datetime import datetime

class GenerationHistory:
    """Persist generation history to local storage."""

    def __init__(self, storage_path="generation_history.json"):
        self.storage_path = storage_path
        self.history = self._load()

    def _load(self):
        if os.path.exists(self.storage_path):
            with open(self.storage_path, "r", encoding="utf-8") as f:
                return json.load(f)
        return []

    def _save(self):
        with open(self.storage_path, "w", encoding="utf-8") as f:
            json.dump(self.history, f, ensure_ascii=False, indent=2)

    def add(self, schema_id, name, params):
        entry = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "schema_id": schema_id,
            "name": name,
            "params": {
                "algorithm": params.algorithm,
                "minimum_length": params.user_minimum_length,
                "maximum_length": params.user_maximum_length,
                "count": params.count,
                "seed": params.seed,
            },
        }
        self.history.append(entry)
        # Keep last 1000 entries
        if len(self.history) > 1000:
            self.history = self.history[-1000:]
        self._save()

    def get_recent(self, limit=10):
        return self.history[-limit:]

    def clear(self):
        self.history = []
        self._save()
```

### Saved Names State

Consumers may persist user-saved names:

```python
class SavedNames:
    """Persist user-saved names."""

    def __init__(self, storage_path="saved_names.json"):
        self.storage_path = storage_path
        self.saved = self._load()

    def _load(self):
        if os.path.exists(self.storage_path):
            with open(self.storage_path, "r", encoding="utf-8") as f:
                return json.load(f)
        return []

    def _save(self):
        with open(self.storage_path, "w", encoding="utf-8") as f:
            json.dump(self.saved, f, ensure_ascii=False, indent=2)

    def add(self, name, schema_id, metadata=None):
        entry = {
            "name": name,
            "schema_id": schema_id,
            "saved_at": datetime.utcnow().isoformat() + "Z",
            "metadata": metadata or {},
        }
        self.saved.append(entry)
        self._save()

    def remove(self, index):
        if 0 <= index < len(self.saved):
            self.saved.pop(index)
            self._save()

    def get_all(self):
        return self.saved
```

### Cached Wordlist State

Consumers may cache loaded wordlists in memory:

```python
class WordlistCache:
    """In-memory cache for loaded wordlists."""

    def __init__(self, base_dir="wordlists", max_entries=100000):
        self.base_dir = base_dir
        self.max_entries = max_entries
        self.cache = {}
        self.access_order = []

    def get(self, path):
        if path in self.cache:
            # Update access order (LRU)
            self.access_order.remove(path)
            self.access_order.append(path)
            return self.cache[path]

        # Load from disk
        entries = load_wordlist(path, self.base_dir)

        # Enforce cache limit
        if len(self.cache) >= self.max_entries:
            self._evict()

        self.cache[path] = entries
        self.access_order.append(path)
        return entries

    def _evict(self):
        # Remove least recently used
        lru = self.access_order.pop(0)
        del self.cache[lru]

    def clear(self):
        self.cache.clear()
        self.access_order.clear()
```

## Persistence Storage Options

### Local Storage (Web Consumers)

| Storage | Capacity | Persistence | Use Case |
|---------|----------|-------------|----------|
| localStorage | ~5 MB | Until cleared | Schema selection, params, history |
| IndexedDB | ~50 MB+ | Until cleared | Large history, cached wordlists |
| sessionStorage | ~5 MB | Tab session | Transient UI state |

### File System (Desktop/CLI Consumers)

| Location | Convention | Use Case |
|----------|------------|----------|
| `~/.config/app/` | XDG config | Settings, history, saved names |
| `~/.local/share/app/` | XDG data | Cached wordlists, indexes |
| `~/.cache/app/` | XDG cache | Temporary caches |

### Database (Server Consumers)

| Database | Use Case |
|----------|----------|
| SQLite | Embedded, single-user history |
| PostgreSQL | Multi-user, shared history |
| Redis | Session state, caching |

## State Synchronisation

### No Repository-Side Sync

The repository provides no state synchronisation mechanism. Consumers implement their own:

| Pattern | Description |
|---------|-------------|
| Local-only | State never leaves consumer device |
| Cloud sync | Consumer uses cloud storage (iCloud, Google Drive, etc.) |
| Server sync | Consumer syncs to own backend |
| Export/import | Manual JSON export/import |

### Version Compatibility

Consumers should handle schema evolution:

```python
def migrate_state(state, from_version, to_version):
    """Migrate consumer state across versions."""
    migrations = {
        ("1.0", "2.0"): migrate_v1_to_v2,
        ("2.0", "3.0"): migrate_v2_to_v3,
    }

    current = from_version
    while current != to_version:
        next_version = get_next_version(current)
        migrator = migrations.get((current, next_version))
        if migrator:
            state = migrator(state)
        current = next_version

    return state
```

## Data Integrity

### Repository Integrity

| Check | Method |
|-------|--------|
| Wordlist format | One entry per line, UTF-8, no trailing whitespace |
| Schema validity | XML well-formed, all paths exist |
| Archive structure | `wordlists/` and `GenerationSchemas.xml` at root |

### Consumer Integrity

| Check | Method |
|-------|--------|
| Loaded entries | Non-empty, valid UTF-8 |
| Schema parameters | Bounds valid, paths safe |
| Generated names | Within length bounds, non-empty |
| Persisted state | Valid JSON, schema version compatible |

## Related Documentation

- [Data Model](data-model.md) -- Schema and wordlist formats
- [API Usage Examples](api-usage-examples.md) -- Consumer implementation patterns
- [Quick Start](quick-start.md) -- Minimal consumer example
- [Troubleshooting](troubleshooting.md) -- State-related issues
