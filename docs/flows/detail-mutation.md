# Detail Mutation Flow

This document describes the end-to-end flow of a consumer application generating names from a selected schema, including parameter adjustment, generation, and result handling. "Detail mutation" is interpreted as the generation flow with parameter mutation (the repository has no CRUD UI; this documents the closest analogue: schema detail view with parameter adjustment and name generation).

## Flow Overview

```
+-------------+    +--------------+    +-------------+    +--------------+    +-------------+
| Select      | -> | Load         | -> | Adjust      | -> | Generate     | -> | Display     |
| Schema      |    | Wordlists    |    | Parameters  |    | Name         |    | Result      |
+-------------+    +--------------+    +-------------+    +--------------+    +-------------+
```

## Phase 1: Select Schema

### Trigger
- User clicks schema in search results (see calendar-search.md)
- User selects from category browser
- User enters schema ID directly

### Actions
```python
def select_schema(schema_id):
    schema = index["by_id"].get(schema_id)
    if not schema:
        raise SchemaNotFoundError(schema_id)

    # Parse schema string
    params = parse_schema_string(schema["schema_string"])

    # Initialize UI state
    state = {
        "schema": schema,
        "params": params,
        "entries": None,  # Not loaded yet
        "filters": None,
        "generated": [],
        "user_params": {
            "minimum_length": params["minimum_length"],
            "maximum_length": params["maximum_length"],
        },
    }
    return state
```

### State After Selection
```
Selected Schema State:
+-- schema: {id, name, category, schema_string, filterlist_path}
+-- params: {algorithm, minimum_length, maximum_length, paths, [order]}
+-- entries: null (lazy)
+-- filters: null (lazy)
+-- generated: []
+-- user_params: {minimum_length, maximum_length} (mutable)
```

## Phase 2: Load Wordlists

### Lazy Load Trigger
- Schema selected (eager) OR
- First generation requested (lazy) OR
- User navigates to detail view (hybrid)

### Load Entries
```python
def load_entries(params, base_path="wordlists"):
    """Load all wordlist entries referenced by schema paths."""
    entries = []
    for path in params["paths"]:
        full_path = os.path.join(base_path, path + ".lst")
        with open(full_path, "r", encoding="utf-8") as f:
            file_entries = [line.strip() for line in f if line.strip()]
            entries.extend(file_entries)
    return entries
```

### Load Filters
```python
def load_filters(filterlist_path, base_path="wordlists"):
    """Load filter entries if FilterlistPath present."""
    if not filterlist_path:
        return set()
    full_path = os.path.join(base_path, filterlist_path + ".lst")
    with open(full_path, "r", encoding="utf-8") as f:
        return set(line.strip() for line in f if line.strip())
```

### Apply Filters
```python
def apply_filters(entries, filters):
    """Remove filtered entries from candidate set."""
    if not filters:
        return entries
    return [e for e in entries if e not in filters]
```

### Load Summary
```python
def load_schema_data(schema, params, base_path="wordlists"):
    entries = load_entries(params, base_path)
    filters = load_filters(schema["filterlist_path"], base_path)
    filtered_entries = apply_filters(entries, filters)
    return {
        "raw_count": len(entries),
        "filtered_count": len(filtered_entries),
        "entries": filtered_entries,
        "filters": filters,
    }
```

### State After Load
```
Loaded Schema State:
+-- schema: {...}
+-- params: {...}
+-- entries: [list of filtered entries]
+-- filters: {set of filter entries}
+-- generated: []
+-- user_params: {minimum_length, maximum_length}
```

## Phase 3: Adjust Parameters

### Mutable Parameters
| Parameter | Default | Range | UI Control |
|-----------|---------|-------|------------|
| minimum_length | From schema | 1-100 | Slider/input |
| maximum_length | From schema | 1-100 | Slider/input |
| count | 1 | 1-100 | Input |
| seed | Random | Any int | Input (optional) |

### Immutable Parameters
| Parameter | Reason |
|-----------|--------|
| algorithm | Core to schema identity |
| order (markov) | Training data dependent |
| paths | Schema definition |
| filterlist_path | Schema definition |

### Parameter Validation
```python
def validate_user_params(user_params, schema_params):
    errors = []

    if user_params["minimum_length"] < 1:
        errors.append("minimum_length must be >= 1")
    if user_params["maximum_length"] > 100:
        errors.append("maximum_length must be <= 100")
    if user_params["minimum_length"] > user_params["maximum_length"]:
        errors.append("minimum_length must be <= maximum_length")

    return errors
```

### Parameter Mutation
```python
def update_params(state, updates):
    new_params = {**state["user_params"], **updates}
    errors = validate_user_params(new_params, state["params"])
    if errors:
        return state, errors
    return {**state, "user_params": new_params}, []
```

## Phase 4: Generate Name

### Generation Dispatch
```python
def generate(state, count=1):
    """Generate names using schema's algorithm."""
    algorithm = state["params"]["algorithm"]

    if algorithm == "randomiser":
        generator = generate_randomiser
    elif algorithm == "markov":
        generator = generate_markov
    else:
        raise ValueError(f"Unknown algorithm: {algorithm}")

    results = []
    for _ in range(count):
        name = generator(state["entries"], state["user_params"], state["params"])
        results.append(name)
    return results
```

### Randomiser Generation
```python
import random

def generate_randomiser(entries, user_params, schema_params):
    """Generate name by random selection and concatenation."""
    minimum_length = user_params["minimum_length"]
    maximum_length = user_params["maximum_length"]

    attempts = 0
    while attempts < 1000:
        # Select 1-3 random entries and concatenate
        parts = [random.choice(entries) for _ in range(random.randint(1, 3))]
        name = "".join(parts)
        if minimum_length <= len(name) <= maximum_length:
            return name
        attempts += 1

    # Fallback: return any entry within bounds
    candidates = [e for e in entries if minimum_length <= len(e) <= maximum_length]
    return random.choice(candidates) if candidates else random.choice(entries)
```

### Markov Generation
```python
import random
from collections import defaultdict

def build_chains(entries, order):
    """Build n-gram transition table."""
    chains = defaultdict(list)
    for entry in entries:
        padded = " " * order + entry + " "
        for i in range(len(padded) - order):
            state = padded[i:i+order]
            next_char = padded[i+order]
            chains[state].append(next_char)
    return chains

def generate_markov(entries, user_params, schema_params):
    """Generate name using markov chain."""
    order = schema_params["order"]
    minimum_length = user_params["minimum_length"]
    maximum_length = user_params["maximum_length"]

    chains = build_chains(entries, order)

    attempts = 0
    while attempts < 1000:
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
        attempts += 1

    # Fallback
    return random.choice(entries)
```

### Generation with Seed (Reproducibility)
```python
def generate_seeded(state, count=1, seed=None):
    """Generate with optional seed for reproducibility."""
    rng = random.Random(seed) if seed is not None else random
    # Use rng instead of random module
    ...
```

## Phase 5: Display Result

### Result Presentation
```
+----------------------------------------------+
| Arabic male persons                    [R]   |
+----------------------------------------------+
|                                              |
|  +----------------------------------------+  |
|  |  Abdullah ibn Rashid                 |  |
|  +----------------------------------------+  |
|                                              |
|  [Generate] [Save] [Copy] [Share]           |
|                                              |
|  Parameters:                                 |
|  Length: 4-32  Algorithm: randomiser         |
|  Sources: 2 wordlists (~500 entries)         |
|  Filter: active (12 exclusions)              |
|                                              |
|  History:                                    |
|  1. Abdullah ibn Rashid                      |
|  2. Omar al-Khalil                           |
|  3. Yusuf ibn Tariq                          |
+----------------------------------------------+
```

### Result Metadata
| Field | Source | Display |
|-------|--------|---------|
| Generated name | Algorithm output | Primary display |
| Length | `len(name)` | Character count |
| Algorithm | Schema params | Badge |
| Schema name | `Name` | Context |
| Timestamp | System clock | History ordering |

### Result Actions
| Action | Behaviour |
|--------|-----------|
| Generate | New generation with same params |
| Save | Add to user's saved list (consumer storage) |
| Copy | Copy to clipboard |
| Share | Share via system share sheet |
| Regenerate | Same as Generate |
| Adjust params | Return to Phase 3 |

## State Transitions

### Full State Machine
```
[Idle] --select--> [Selected] --load--> [Loaded] --adjust--> [Adjusted]
                                                              |
                                                              v
                                                        [Generating]
                                                              |
                                                              v
                                                        [Result]
                                                              |
                                    +--------------------------+
                                    |                          |
                                    v                          v
                              [Adjusted]               [History Updated]
                                    |                          |
                                    +----------+---------------+
                                               v
                                          [Generating]
```

### State Persistence (Consumer-Side)
| State | Persisted? | Storage |
|-------|-----------|---------|
| Selected schema ID | Optional | LocalStorage/config |
| User params | Optional | LocalStorage/config |
| Generated history | Optional | LocalStorage/DB |
| Saved names | Yes (user data) | Consumer storage |
| Loaded entries | No (session) | Memory |

## Error Handling

### Generation Errors

| Error | Cause | Response |
|-------|-------|----------|
| Empty entries | Filter removed all | Warn, disable generate |
| No valid length | min > max | Validation error |
| Algorithm failure | Edge case in markov | Fallback to random entry |
| Wordlist missing | Path invalid | Error, report schema |

### Error Recovery
```python
def generate_with_recovery(state, count=1):
    try:
        if not state["entries"]:
            raise EmptyEntriesError("No entries after filtering")
        return generate(state, count)
    except EmptyEntriesError as e:
        log_warning(e)
        show_warning("No entries available. Adjust filters or select another schema.")
        return []
    except Exception as e:
        log_error(e)
        show_error("Generation failed. Try again.")
        return []
```

## Performance Characteristics

### Generation Latency

| Algorithm | Typical | Worst Case | Notes |
|-----------|---------|------------|-------|
| Randomiser | <1ms | <10ms | O(1) per selection |
| Markov | <5ms | <50ms | Chain build O(n), generate O(m) |

### Memory During Generation
| Component | Size |
|-----------|------|
| Entries (per schema) | 10-500KB |
| Markov chains | 2-5x entries |
| Generated names | Negligible |

## Related Documentation

- Startup and rendering: `./startup-and-rendering.md` (schema loading)
- Calendar search: `./calendar-search.md` (schema selection)
- Application services: `../components/application-services.md` (generation services)
- Architecture: `../architecture.md` (algorithms)
- Quick start: `../quick-start.md` (consumer integration)
