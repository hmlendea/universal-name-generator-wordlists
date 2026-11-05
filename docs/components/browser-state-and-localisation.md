# Browser State and Localisation

This document describes state management and localisation considerations for consumers of this repository. Since this is an assets-only repository with no runtime, these concerns apply to downstream applications (e.g., the Universal Name Generator web app) that consume the data.

## State Management (Consumer-Side)

### Repository Data as Immutable State

**Principle**: Repository assets are read-only at runtime.

| State Type | Source | Mutability | Consumer Handling |
|------------|--------|------------|-------------------|
| Schema catalogue | `GenerationSchemas.xml` | Immutable per release | Load once, cache in memory |
| Wordlist entries | `wordlists/**/*.lst` | Immutable per release | Load once, cache in memory |
| Filter lists | Referenced `.lst` files | Immutable per release | Load once, cache in memory |
| Generated names | Consumer algorithm output | Mutable (per request) | Not persisted by repository |

### Consumer State Patterns

**Build-time loading** (recommended):
```
Build process:
1. Download release archive
2. Extract to application bundle
3. Parse GenerationSchemas.xml
4. Pre-load wordlists into memory
5. Ship with application
```

**Runtime loading** (alternative):
```
Application startup:
1. Locate wordlists directory (config)
2. Parse GenerationSchemas.xml
3. Lazy-load wordlists on first schema use
4. Cache loaded wordlists
```

**Browser-specific** (web consumers):
```
Web app:
1. Fetch GenerationSchemas.xml (or embedded JSON)
2. Fetch wordlist files on demand
3. Cache in IndexedDB or memory
4. Generate names client-side
```

### State Consistency

| Concern | Repository Guarantee | Consumer Responsibility |
|---------|---------------------|------------------------|
| Schema ID stability | IDs never change | Can cache by ID |
| Path stability | Schema paths stable | Can cache resolved paths |
| Version consistency | Archive is atomic | Use single archive version |
| Data integrity | UTF-8, one per line | Validate on load |

**Mixed-version hazard**:
```
WRONG: Load schemas from v3.2.0, wordlists from v3.1.0
RIGHT: Load all assets from same release archive
```

### Browser Storage (Web Consumers)

| Storage | Use Case | Size Limit | Persistence |
|---------|----------|------------|-------------|
| Memory | Active generation | RAM | Session |
| IndexedDB | Cached wordlists | ~50% disk | Persistent |
| LocalStorage | Schema metadata | 5-10MB | Persistent |
| Service Worker | Offline archive | Varies | Configurable |

**Recommended pattern**:
```javascript
// Load schemas once, cache in memory
const schemas = await loadSchemas(); // from embedded JSON or fetch

// Lazy-load wordlists, cache in IndexedDB
async function loadWordlist(path) {
  const cached = await idb.get('wordlists', path);
  if (cached) return cached;
  const entries = await fetch(`wordlists/${path}.lst`).then(r => r.text());
  await idb.put('wordlists', entries, path);
  return entries;
}
```

## Localisation

### Repository Data Localisation

**Current state**: Data is inherently multilingual.

| Language Group | Location | Example |
|----------------|----------|---------|
| Arabic | `wordlists/real/arabic/` | Arabic script names |
| Chinese | `wordlists/real/chinese/` | CJK characters |
| Japanese | `wordlists/real/japanese/` | CJK characters |
| Russian | `wordlists/real/russian/` | Cyrillic names |
| Greek | `wordlists/real/greek/` | Greek script names |
| Hebrew | `wordlists/real/` (if present) | RTL script |
| Hindi | `wordlists/real/hindi/` | Devanagari script |
| Thai | `wordlists/real/` (if present) | Thai script |
| Latin | `wordlists/real/latin/` | Latin script |
| ... | 30+ languages | Various scripts |

**Encoding**: All files UTF-8 -> no encoding conversion needed
**Direction**: Repository does not store direction metadata; consumer must handle RTL

### Consumer Localisation Responsibilities

**UI localisation** (consumer responsibility):
- Schema names are English-only in `GenerationSchemas.xml`
- Category names are English-only
- Consumer must translate for non-English UI

**Name localisation** (repository provides data):
- Wordlists contain native-script entries
- Consumer displays as-is (no transliteration)
- Sorting/collation is consumer responsibility

**Script handling**:
```python
# Consumer must handle mixed scripts
names = load_wordlist("real/chinese/persons/males")
# Names may contain CJK characters
# Display requires UTF-8 capable renderer
# Sorting requires locale-aware collation
```

### Localisation Invariants

| Invariant | Description |
|-----------|-------------|
| UTF-8 everywhere | All `.lst` and `.xml` files UTF-8 |
| No BOM | Files should not have BOM (check with `file`) |
| No encoding metadata | Files don't declare encoding; UTF-8 assumed |
| English metadata | Schema `Name` and `Category` are English |
| Native entries | Wordlist entries in native script |

### Localisation Limitations

| Limitation | Impact | Workaround |
|------------|--------|------------|
| No locale metadata | Consumer doesn't know script/direction | Infer from content or maintain mapping |
| No transliteration | No romanisation provided | Consumer implements if needed |
| No pronunciation | No phonetic data | Consumer implements if needed |
| No gender metadata | Gender only via file path | Parse path or maintain mapping |
| No cultural context | No usage notes | External documentation |

## Session State (Consumer-Side)

### Generation Session State

**Consumer-managed state** (repository provides no state):
```
Session:
+-- Selected schema ID
+-- Generated names history
+-- User preferences (length, algorithm params)
+-- Favorite/saved names
+-- Filter overrides
```

**Repository provides**: None of the above
**Consumer implements**: All of the above

### Persistence Patterns

| Pattern | Storage | Use Case |
|---------|---------|----------|
| In-memory | RAM | Single-session web app |
| LocalStorage | Browser | User preferences |
| IndexedDB | Browser | Cached wordlists |
| Server DB | Backend | Multi-user history |
| File | Local disk | Desktop app config |

## Configuration State

### Consumer Configuration

**Repository provides**: No configuration files
**Consumer manages**:
- Path to extracted archive
- Default schema selection
- Generation parameters (if overridable)
- Cache location
- Update check frequency

**Example consumer config**:
```json
{
  "wordlists_path": "./data/universal-name-generator-wordlists_3.2.0",
  "default_schema": "arabic-persons-male",
  "cache_wordlists": true,
  "check_updates": true,
  "update_channel": "latest"
}
```

### Repository Configuration State

**Repository has**: No runtime configuration
**Repository metadata**:
- `.github/workflows/release-asset.yml` (CI/CD config)
- `GenerationSchemas.xml` (data catalogue, not config)
- Root documentation files

## Related Documentation

- Architecture: `../architecture.md` (data formats)
- Data model: `../data-model.md` (entities)
- Application services: `./application-services.md` (consumer implementation)
- Integration models: `./integration-models.md` (consumer patterns)
- Quick start: `../quick-start.md` (consumer setup)
- State and persistence: `../state-and-persistence.md` (detailed state model)
