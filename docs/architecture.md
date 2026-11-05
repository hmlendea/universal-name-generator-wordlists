# Architecture (Complementary Detail)

This document elaborates on the root `ARCHITECTURE.md` with implementation-level detail for contributors and integrators. It does not duplicate the root document; refer to `../ARCHITECTURE.md` for system context, architectural style, runtime flow, and high-level contracts.

## Schema String Grammar

The `Schema` property in `GenerationSchemaDataObject` follows a precise grammar:

```
Schema := '{' Algorithm ',' Params '}'
Algorithm := 'randomiser' | 'markov'
Params := RandomiserParams | MarkovParams
RandomiserParams := ',' MinLength ',' MaxLength ',' PathList
MarkovParams := Order ',' MinLength ',' MaxLength ',' PathList
Order := Integer (typically 4)
MinLength := Integer
MaxLength := Integer
PathList := Path ('|' Path)*
Path := Repository-relative path under wordlists/
```

### Parameter Semantics

| Parameter | randomiser | markov | Description |
|-----------|------------|--------|-------------|
| Order | N/A | Required | N-gram order for Markov chain (typically 4) |
| MinLength | Required | Required | Minimum generated name length |
| MaxLength | Required | Required | Maximum generated name length |
| PathList | Required | Required | Pipe-separated `.lst` file paths |

### Path Resolution

Paths in `Schema` strings are **repository-relative** under `wordlists/`. Examples:
- `real/arabic/persons/males` -> `wordlists/real/arabic/persons/males.lst`
- `no-mans-sky/gek/persons/neutral` -> `wordlists/no-mans-sky/gek/persons/neutral.lst`

The downstream consumer (Universal Name Generator API) resolves these paths at build time when embedding the release archive.

## Generation Algorithms

### Randomiser

**Algorithm**: Uniform random selection from the combined entries of all referenced wordlists.

**Process**:
1. Load all entries from each path in `PathList`
2. Combine into single candidate pool
3. Filter by length constraints (`MinLength` ≤ length ≤ `MaxLength`)
4. Select uniformly at random

**Characteristics**:
- Stateless between generations
- Output distribution mirrors input distribution
- Suitable for combining distinct name parts (e.g., given names + surnames)

### Markov Chain

**Algorithm**: N-gram Markov chain generation trained on referenced wordlists.

**Process**:
1. Load all entries from each path in `PathList`
2. Build N-gram model (order = `Order` parameter)
3. Generate names by walking the chain
4. Apply length constraints (`MinLength` ≤ length ≤ `MaxLength`)

**Characteristics**:
- Produces novel combinations not present in source data
- Output respects phonotactic patterns of training data
- Requires sufficient training data for quality results
- `Order` parameter controls fidelity vs. novelty trade-off

## Filter Lists

### Purpose

Post-generation filtering to exclude undesirable outputs (profanity, duplicates, malformed names).

### Format

Same as wordlists: UTF-8 plain text, one entry per line, no trailing whitespace.

### Integration

Referenced via optional `FilterlistPath` in `GenerationSchemaDataObject`:

```xml
<GenerationSchemaDataObject>
    <Id>german-persons-male</Id>
    <FilterlistPath>real/german/persons/filters</FilterlistPath>
    <Schema>{randomiser, ,4,32,real/german/persons/males|real/german/persons/surnames}</Schema>
    ...
</GenerationSchemaDataObject>
```

### Consumer Responsibility

The downstream generator is responsible for:
1. Loading the filter list at build/runtime
2. Applying filters after generation
3. Retrying generation if output is filtered (implementation-defined)

This repository only declares the filter reference; it does not enforce filtering behaviour.

## Directory Taxonomy

### Universe Categories

| Universe Type | Examples | Characteristics |
|---------------|----------|-----------------|
| Fictional settings | `age-of-wonders`, `elder-scrolls`, `middle-earth` | Game/literature IP; race/faction subdirectories |
| Sci-fi settings | `no-mans-sky`, `starcraft`, `age-of-wonders-planetfall` | Species/faction subdirectories |
| Real-world languages | `real/arabic`, `real/english`, `real/romanian` | Language subdirectories under `real/` |
| Community sources | `usernames/` | Platform-specific username datasets |
| Original settings | `narivia`, `pinched-universe` | Custom fictional worlds |

### Category Subdirectories

Standard categories (used across universes where applicable):

| Category | Purpose | Typical Schemas |
|----------|---------|-----------------|
| `persons/` | Personal names | `*-persons-male`, `*-persons-female`, `*-persons-neutral` |
| `locations/` | Place names | `*-toponyms`, `*-settlements`, `*-cities` |
| `units/` | Military/vehicle names | `*-ships-space`, `*-units` |
| `groups/` | Organisation/faction names | `*-groups` |
| `flora/` | Plant species names | `*-flora`, `*-trees`, `*-plants` |
| `fauna/` | Animal species names | `*-fauna`, `*-birds`, `*-fishes` |

### Special Cases

- `real/roman/` uses `cognomen`/`pronomen` for Roman naming conventions
- `real/romanian/` has extensive `flora/` and `fauna/` subcategories
- `usernames/` is flat (no subdirectories); each `.lst` is a platform dataset

## Release Archive Structure

```
universal-name-generator-wordlists_X.Y.Z.zip
+-- wordlists/           # Complete wordlists directory tree
|   +-- ...              # All 388 .lst files
+-- GenerationSchemas.xml # Schema catalogue
```

**Consumer Expectation**: Downstream consumers (Universal Name Generator API) expect this exact layout. The archive root must contain `wordlists/` and `GenerationSchemas.xml` directly (no nested wrapper directory).

## Versioning and Compatibility

### Schema Versioning

No explicit schema version in XML. Compatibility managed through:
1. **Id stability** -- `Id` values never change; new schemas get new Ids
2. **Additive only** -- New `GenerationSchemaDataObject` entries are non-breaking
3. **Path stability** -- `.lst` paths in `Schema` strings should not change

### Breaking Changes

| Change | Impact | Migration |
|--------|--------|-----------|
| `Id` rename/removal | Consumer build fails (missing schema) | Consumer must update schema references |
| `Schema` path change | Consumer references missing `.lst` | Consumer must update or repository must coordinate |
| `FilterlistPath` removal | Consumer filter logic breaks | Consumer must handle missing filter |
| Archive layout change | Consumer build fails | Consumer must update extraction logic |

### Release Versioning

- Git tags: `vX.Y.Z` (Semantic Versioning)
- Archive name: `universal-name-generator-wordlists_X.Y.Z.zip` (no `v` prefix)
- Workflow extracts version via `${{ github.event.release.tag_name }}` -> strip `v` prefix

## CI/CD Pipeline Detail

### Release Asset Workflow

```yaml
Trigger: release.published
Runner: ubuntu-latest
Steps:
  1. actions/checkout@v4
  2. Package: zip -r "universal-name-generator-wordlists_${version}.zip" wordlists GenerationSchemas.xml
  3. Upload: softprops/action-gh-release@v2 with archive asset
```

### Validation Gaps (Current)

- No XML schema validation in CI (advisory only via `xmllint`)
- No `.lst` format validation in CI (whitespace check via `git diff --check` locally)
- No duplicate detection
- No path reference validation (Schema paths vs. actual `.lst` files)

### Planned Enhancements (per ROADMAP.md)

- Enforce `xmllint` validation in CI
- Add duplicate detection reporting
- Canonical formatting conventions for `.lst` files

## Extension Points (Implementation Detail)

### Adding a New Universe

1. Create `wordlists/new-universe/` (lowercase, hyphen-separated)
2. Add category subdirectories (plural nouns: `persons/`, `locations/`, etc.)
3. Add `.lst` files (UTF-8, one entry per line, no trailing whitespace)
4. Add `GenerationSchemaDataObject` entries in `GenerationSchemas.xml`
5. Submit PR for review

### Adding a New Language (under `real/`)

1. Create `wordlists/real/new-language/` (lowercase)
2. Add standard category subdirectories
3. Add `.lst` files following real-world naming conventions
4. Add schemas with `Category` matching language name (e.g., `Category: Romanian`)
5. Consider `FilterlistPath` for languages with known problematic outputs

### Proposing a New Algorithm

1. Define new `Schema` string format (e.g., `{weighted,weights,path1|path2}`)
2. Document semantics in PR description
3. Ensure Universal Name Generator API can parse new format before merge
4. Update this documentation

## Cross-References

- Root architecture: `../ARCHITECTURE.md`
- Data model: `./data-model.md`
- Repository structure: `./repository-structure.md`
- Build and deployment: `./build-and-deployment.md`
- Configuration: `./configuration.md`
- Dependencies: `./dependencies.md`
