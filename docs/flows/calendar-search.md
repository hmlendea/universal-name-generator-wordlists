# Calendar Search Flow

This document describes the end-to-end flow for searching and filtering generation schemas in a consumer application. "Calendar search" is interpreted as the schema discovery/browsing flow (the repository has no calendar feature; this documents the closest analogue: searching the schema catalogue).

## Flow Overview

```
+-------------+    +--------------+    +-------------+    +--------------+    +-------------+
| User Input  | -> | Filter       | -> | Match       | -> | Rank/Sort    | -> | Display     |
| (query)     |    | Schemas      |    | Against     |    | Results      |    | Results     |
|             |    |              |    | Query       |    |              |    |             |
+-------------+    +--------------+    +-------------+    +--------------+    +-------------+
```

## Phase 1: User Input

### Input Types

| Input | Example | Intent |
|-------|---------|--------|
| Free text | "arabic male" | Find matching schemas |
| Category select | "Arabic" | Filter by category |
| Algorithm filter | "markov" | Filter by algorithm |
| Universe select | "Elder Scrolls" | Filter by universe |
| Combined | Category=Elven, text="male" | Multi-facet search |

### Input Handling
```javascript
// Web app example
const searchState = {
    query: "",           // Free text
    category: null,      // Category filter
    algorithm: null,     // "randomiser" | "markov"
    universe: null,      // Universe filter
};

function onSearchChange(newState) {
    searchState = { ...searchState, ...newState };
    executeSearch(searchState);
}
```

## Phase 2: Filter Schemas

### Load Schema Index
```python
# Pre-loaded at startup (see startup-and-rendering.md)
schemas = load_schemas("GenerationSchemas.xml")
index = build_index(schemas)  # by_id, by_category
```

### Apply Filters
```python
def filter_schemas(schemas, search_state):
    results = schemas

    # Category filter
    if search_state.category:
        results = [s for s in results if s["category"] == search_state.category]

    # Algorithm filter
    if search_state.algorithm:
        results = [s for s in results
                   if parse_schema_string(s["schema_string"])["algorithm"] == search_state.algorithm]

    # Universe filter (derived from schema paths)
    if search_state.universe:
        results = [s for s in results
                   if any(p.startswith(search_state.universe)
                          for p in parse_schema_string(s["schema_string"])["paths"])]

    # Free text query
    if search_state.query:
        query = search_state.query.lower()
        results = [s for s in results
                   if query in s["name"].lower()
                   or query in s["id"].lower()
                   or query in s["category"].lower()]

    return results
```

### Filter Execution Order
1. Category (most selective, O(n))
2. Algorithm (selective, O(n))
3. Universe (moderately selective, O(n))
4. Free text (least selective, O(n·m) where m = name length)

**Optimisation**: Apply most selective filters first to reduce candidate set.

## Phase 3: Match Against Query

### Matching Strategies

| Strategy | Description | Example |
|----------|-------------|---------|
| Exact match | Query equals field | "arabic-persons-male" == id |
| Substring | Query contained in field | "arabic" in "Arabic male persons" |
| Prefix | Query prefixes field | "arab" prefixes "arabic-persons-male" |
| Fuzzy | Edit distance ≤ threshold | "arabic" ~ "arabiic" |
| Token | All query tokens present | "arabic male" -> both tokens in name |

### Recommended: Token Matching
```python
def token_match(schema, query):
    """Match if all query tokens appear in searchable fields."""
    tokens = query.lower().split()
    searchable = f"{schema['id']} {schema['name']} {schema['category']}".lower()
    return all(token in searchable for token in tokens)
```

### Searchable Fields
| Field | Source | Example |
|-------|--------|---------|
| `id` | `Id` element | `arabic-persons-male` |
| `name` | `Name` element | `Arabic male persons` |
| `category` | `Category` element | `Arabic` |
| `algorithm` | Parsed from `Schema` | `randomiser` |
| `paths` | Parsed from `Schema` | `real/arabic/persons/males` |

## Phase 4: Rank and Sort

### Ranking (Relevance)
```python
def rank_schema(schema, query):
    """Higher score = more relevant."""
    score = 0
    tokens = query.lower().split()

    # Exact ID match: highest
    if schema["id"] == query.lower():
        score += 1000

    # ID prefix match
    if schema["id"].startswith(query.lower()):
        score += 100

    # Name contains all tokens
    name_lower = schema["name"].lower()
    if all(t in name_lower for t in tokens):
        score += 50

    # Category exact match
    if schema["category"].lower() == query.lower():
        score += 30

    # Token count match (more tokens matched = better)
    score += sum(10 for t in tokens if t in name_lower)

    return score
```

### Sorting
```python
def sort_results(results, query):
    # Primary: relevance score (descending)
    # Secondary: category (ascending)
    # Tertiary: name (ascending)
    return sorted(results, key=lambda s: (
        -rank_schema(s, query),
        s["category"],
        s["name"],
    ))
```

### Sort Options (User-Selectable)
| Sort | Key | Use Case |
|------|-----|----------|
| Relevance | Score desc | Search results |
| Name A-Z | Name asc | Browsing |
| Category | Category asc, Name asc | Browsing |
| Algorithm | Algorithm asc, Name asc | Algorithm comparison |

## Phase 5: Display Results

### Result Card
```
+----------------------------------------------+
| Arabic male persons                    [>]   |
| ID: arabic-persons-male                      |
| Category: Arabic | Algorithm: randomiser     |
| Sources: 2 wordlists | ~500 entries          |
+----------------------------------------------+
```

### Result Metadata
| Field | Source | Display |
|-------|--------|---------|
| Name | `Name` | Primary label |
| ID | `Id` | Secondary text |
| Category | `Category` | Badge/tag |
| Algorithm | Parsed | Badge/icon |
| Wordlist count | Parsed paths | "N sources" |
| Entry count | Loaded wordlists | "~N entries" |

### Pagination
```python
def paginate(results, page=1, page_size=20):
    start = (page - 1) * page_size
    end = start + page_size
    return results[start:end], len(results)
```

### Empty State
```
No schemas match "xyz".
Try:
- Different keywords
- Browse by category
- Clear filters
```

## Phase 6: Result Selection

### Select Schema
```python
def select_schema(schema_id):
    schema = index["by_id"][schema_id]
    # Trigger wordlist load (lazy)
    resolved = resolve_schema(schema, data_source)
    # Navigate to generation view
    navigate_to_generation(resolved)
```

### Selection State
```
Selected Schema:
+-- id: arabic-persons-male
+-- name: Arabic male persons
+-- category: Arabic
+-- algorithm: randomiser
+-- params: {minimum_length: 4, maximum_length: 32}
+-- paths: [real/arabic/persons/males, real/arabic/persons/surnames]
+-- filterlist_path: real/arabic/persons/filters
+-- entries: [loaded lazily]
```

## Search Performance

### Complexity
| Operation | Complexity | Notes |
|-----------|------------|-------|
| Filter by category | O(n) | n = 328 schemas |
| Filter by algorithm | O(n) | Parse each schema string |
| Free text search | O(n·m) | m = avg field length |
| Rank | O(n·k) | k = query tokens |
| Sort | O(n log n) | Standard sort |
| **Total** | **O(n·(m+k) + n log n)** | Fast for n=328 |

### Optimisation Opportunities
| Optimisation | Impact | Effort |
|--------------|--------|--------|
| Pre-parse schema strings | Eliminates parse per search | Low |
| Inverted index on tokens | O(1) token lookup | Medium |
| Cache search results | Avoid recompute | Low |
| Debounce input | Reduce search frequency | Low |
| Web Worker | Non-blocking UI | Medium |

## Edge Cases

| Case | Handling |
|------|----------|
| Empty query | Return all schemas (browse mode) |
| No results | Show empty state with suggestions |
| Special characters | Escape/normalise before matching |
| Very long query | Truncate or tokenize |
| Unicode query | UTF-8 aware matching |
| Case sensitivity | Case-insensitive by default |

## Related Documentation

- Startup and rendering: `./startup-and-rendering.md` (schema loading)
- Application services: `../components/application-services.md` (schema catalogue service)
- Presentation: `../components/presentation.md` (category presentation)
- API reference: `../api-reference/INDEX.md` (schema catalogue)
- Browse and search: `../behaviour/browse-and-search.md` (repository-level search)
