# Concurrency and Scheduling

This document describes concurrency and scheduling considerations for the Universal Name Generator -- Wordlists repository and its consumers. Since this is an assets-only repository with no runtime, there is no repository-side concurrency or scheduling. All concurrency concerns apply to consumer implementations.

## Repository Concurrency Model

### No Runtime Concurrency

| Aspect | State |
|--------|-------|
| Threads | None |
| Processes | None |
| Async tasks | None |
| Locks | None |
| Semaphores | None |
| Queues | None |
| Schedulers | None |
| Cron jobs | None |

### Why No Concurrency

- **Assets-only**: Repository contains static data files, not executable code
- **No runtime**: No process executes; nothing to schedule
- **No services**: No server, daemon, or background worker
- **Static distribution**: Data distributed via GitHub Releases; no request handling

### Git Concurrency (Contributor-Side)

The only concurrency in this repository is Git-level, managed by GitHub:

| Mechanism | Purpose |
|-----------|---------|
| Branch protection | Prevents direct pushes to main |
| Pull request review | Serialises changes through review |
| Merge queue | Optional serialisation of merges |
| Git locking | Prevents concurrent pushes to same ref |

**Contributor guidance**:
- One logical change per pull request
- Rebase before merge to avoid conflicts
- Do not force-push shared branches
- Resolve merge conflicts locally before pushing

## Consumer-Side Concurrency Patterns

### Thread Safety

Consumers may generate names concurrently. Repository data is read-only, so thread safety depends on consumer implementation:

```python
import threading
import random

class ThreadSafeGenerator:
    """Thread-safe name generator."""

    def __init__(self, schemas, base_dir="wordlists"):
        self.schemas = schemas
        self.base_dir = base_dir
        self._lock = threading.Lock()
        self._cache = {}

    def generate(self, schema_id, count=1):
        """Generate names (thread-safe)."""
        with self._lock:
            schema = next((s for s in self.schemas if s["id"] == schema_id), None)
            if not schema:
                raise ValueError(f"Unknown schema: {schema_id}")
            params = parse_schema_string(schema["schema"])
            cache_key = tuple(params["paths"])
            if cache_key not in self._cache:
                self._cache[cache_key] = load_wordlists(params["paths"], self.base_dir)
            entries = self._cache[cache_key]

        # Generation outside lock (entries list is read-only)
        results = []
        for _ in range(count):
            name = generate_randomiser(entries, params["minimum_length"], params["maximum_length"])
            results.append(name)
        return results
```

### Process Safety

Consumers may run generation in separate processes:

```python
from multiprocessing import Pool

def generate_batch(schema_ids, schemas, base_dir="wordlists", workers=4):
    """Generate names in parallel processes."""
    with Pool(workers) as pool:
        results = pool.starmap(
            _generate_single,
            [(schema_id, schemas, base_dir) for schema_id in schema_ids]
        )
    return results

def _generate_single(schema_id, schemas, base_dir):
    """Generate single name (process-safe)."""
    schema = next((s for s in schemas if s["id"] == schema_id), None)
    if not schema:
        raise ValueError(f"Unknown schema: {schema_id}")
    params = parse_schema_string(schema["schema"])
    entries = load_wordlists(params["paths"], base_dir)
    return generate_randomiser(entries, params["minimum_length"], params["maximum_length"])
```

### Async Pattern

Consumers may use async I/O for loading:

```python
import asyncio
import aiofiles

async def load_wordlist_async(path, base_dir="wordlists"):
    """Load wordlist asynchronously."""
    full_path = os.path.join(base_dir, path + ".lst")
    async with aiofiles.open(full_path, "r", encoding="utf-8") as f:
        content = await f.read()
    return [line.strip() for line in content.splitlines() if line.strip()]

async def load_wordlists_async(paths, base_dir="wordlists"):
    """Load multiple wordlists concurrently."""
    tasks = [load_wordlist_async(path, base_dir) for path in paths]
    results = await asyncio.gather(*tasks)
    entries = []
    for result in results:
        entries.extend(result)
    return entries
```

## Scheduling Patterns

### No Repository-Side Scheduling

The repository has no scheduler, cron, or periodic task mechanism. All scheduling is consumer-side.

### Consumer Scheduling Patterns

| Pattern | Use Case | Implementation |
|---------|----------|----------------|
| On-demand | User clicks generate | Direct function call |
| Periodic refresh | Update wordlist cache | `cron`, `APScheduler`, `node-cron` |
| Event-driven | Generate on user action | Event handler |
| Batch | Pre-generate name pool | Background job |
| Scheduled release check | Detect new releases | Periodic GitHub API poll |

### Release Update Checking

Consumers may schedule periodic checks for new releases:

```python
import requests

def check_for_updates(current_version):
    """Check GitHub Releases for newer version."""
    url = "https://api.github.com/repos/hmlendea/universal-name-generator-wordlists/releases/latest"
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    latest = response.json()
    latest_version = latest["tag_name"].lstrip("v")

    if latest_version != current_version:
        return {
            "update_available": True,
            "latest_version": latest_version,
            "download_url": latest["assets"][0]["browser_download_url"],
        }
    return {"update_available": False}
```

### Cache Refresh Scheduling

```python
import time

class ScheduledCacheRefresher:
    """Periodically refresh wordlist cache."""

    def __init__(self, generator, interval_seconds=3600):
        self.generator = generator
        self.interval_seconds = interval_seconds
        self._running = False

    def start(self):
        self._running = True
        while self._running:
            time.sleep(self.interval_seconds)
            self.generator.clear_cache()

    def stop(self):
        self._running = False
```

## Resource Contention

### Read-Only Data

Repository data is read-only, so read contention is not a concern:

| Resource | Contention Risk | Mitigation |
|----------|-----------------|------------|
| Wordlist files | None (read-only) | None needed |
| Schema XML | None (read-only) | None needed |
| Release archives | None (immutable) | None needed |

### Consumer Resources

Consumer-side resources may contend:

| Resource | Contention Risk | Mitigation |
|----------|-----------------|------------|
| Memory (cached wordlists) | High with many schemas | LRU cache with limit |
| CPU (generation) | High with parallel generation | Thread/process pool with limit |
| Disk I/O (loading) | High with cold cache | Async I/O, preloading |
| Network (release checks) | Low | Rate limiting, caching |

## Performance Considerations

### Generation Throughput

| Algorithm | Relative Speed | Bottleneck |
|-----------|----------------|------------|
| Randomiser | Fast | Random selection, string concatenation |
| Markov | Moderate | Chain building, state transitions |

### Scaling Guidance

| Scale | Recommendation |
|-------|----------------|
| Single user, occasional use | Load on demand, no cache |
| Single user, frequent use | Cache wordlists in memory |
| Multi-user server | Shared cache, connection pooling |
| High throughput | Pre-load all wordlists, pool generators |

### Benchmarking

```python
import time

def benchmark_generation(generator_func, schema_id, iterations=1000):
    """Benchmark generation throughput."""
    start = time.perf_counter()
    for _ in range(iterations):
        generator_func(schema_id)
    elapsed = time.perf_counter() - start
    return {
        "iterations": iterations,
        "total_seconds": elapsed,
        "per_generation_ms": (elapsed / iterations) * 1000,
        "per_second": iterations / elapsed,
    }
```

## Related Documentation

- [Application Services](components/application-services.md) -- Generation algorithms
- [State and Persistence](state-and-persistence.md) -- Consumer state patterns
- [Error Handling](error-handling.md) -- Timeout and failure handling
- [API Usage Examples](api-usage-examples.md) -- Implementation examples
