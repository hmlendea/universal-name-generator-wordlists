# Browse and Search Behaviour

This document describes how users (contributors, consumers, integrators) navigate and discover content in the repository.

## Repository Navigation

### Directory Structure Browsing

**Universe-level browsing**:
```
wordlists/
+-- a-song-of-ice-and-fire/
+-- age-of-wonders/
+-- age-of-wonders-planetfall/
+-- divinity/
+-- dragon-age/
+-- elder-scrolls/
+-- middle-earth/
+-- narivia/
+-- no-mans-sky/
+-- pinched-universe/
+-- real/
+-- runescape/
+-- starcraft/
+-- usernames/
+-- warcraft/
+-- yohoho-puzzle-pirates/
```

**Category-level browsing** (within a universe):
```
wordlists/real/
+-- arabic/
+-- asian/
+-- chinese/
+-- dacian/
+-- danish/
+-- dutch/
+-- english/
+-- finnish/
+-- french/
+-- german/
+-- greek/
+-- hausa/
+-- hawaiian/
+-- hindi/
+-- hungarian/
+-- indonesian/
+-- irish/
+-- italian/
+-- japanese/
+-- latin/
+-- lithuanian/
+-- maori/
+-- norwegian/
+-- polish/
+-- portuguese/
+-- romanian/
+-- russian/
+-- serbocroatian/
+-- slovak/
+-- somali/
+-- spanish/
+-- swedish/
+-- thracian/
+-- turkish/
+-- venetian/
+-- welsh/
```

**File-level browsing** (within a category):
```
wordlists/real/arabic/
+-- persons/
|   +-- males.lst
|   +-- females.lst
|   +-- surnames.lst
+-- locations/
    +-- toponyms.lst
```

### GitHub Web Interface

| View | URL Pattern | Use Case |
|------|-------------|----------|
| Repository root | `https://github.com/<owner>/universal-name-generator-wordlists` | Overview, README |
| Wordlists tree | `https://github.com/<owner>/universal-name-generator-wordlists/tree/master/wordlists` | Browse universes |
| Universe tree | `https://github.com/<owner>/universal-name-generator-wordlists/tree/master/wordlists/<universe>` | Browse categories |
| Category tree | `https://github.com/<owner>/universal-name-generator-wordlists/tree/master/wordlists/<universe>/<category>` | Browse files |
| File view | `https://github.com/<owner>/universal-name-generator-wordlists/blob/master/wordlists/<universe>/<category>/<file>.lst` | Inspect entries |
| Schema catalogue | `https://github.com/<owner>/universal-name-generator-wordlists/blob/master/GenerationSchemas.xml` | View all schemas |
| Releases | `https://github.com/<owner>/universal-name-generator-wordlists/releases` | Download packaged data |

### Local Filesystem Browsing

```bash
# List all universes
ls wordlists/

# List categories in a universe
ls wordlists/real/

# List files in a category
ls wordlists/real/arabic/persons/

# View file contents
cat wordlists/real/arabic/persons/males.lst

# Count entries in file
wc -l wordlists/real/arabic/persons/males.lst

# Search across all wordlists
grep -r "pattern" wordlists/
```

## Search Behaviours

### By Universe

**Goal**: Find all data for a specific fictional universe or real-world language group.

**Method**:
```bash
# Local
ls wordlists/elder-scrolls/
find wordlists/elder-scrolls -name "*.lst"

# GitHub
# Navigate to wordlists/elder-scrolls/ in web UI
```

**Result**: All categories and files for that universe.

### By Category

**Goal**: Find all files of a specific type (persons, locations, etc.) across universes.

**Method**:
```bash
# Local - all "persons" categories
find wordlists -type d -name "persons"

# Local - all "locations" categories
find wordlists -type d -name "locations"

# Local - all files in any "persons" category
find wordlists -path "*/persons/*.lst"
```

**Result**: Cross-universe category view.

### By Schema (GenerationSchemas.xml)

**Goal**: Find all generation schemas, filter by category, algorithm, or referenced files.

**Method**:
```bash
# All schema IDs
xmllint --xpath '//Id/text()' GenerationSchemas.xml | tr ' ' '\n'

# Schemas by category
xmllint --xpath '//GenerationSchemaDataObject[Category="Arabic"]/Id/text()' GenerationSchemas.xml

# Schemas by algorithm (randomiser)
xmllint --xpath '//GenerationSchemaDataObject[contains(Schema, "randomiser")]/Id/text()' GenerationSchemas.xml

# Schemas by algorithm (markov)
xmllint --xpath '//GenerationSchemaDataObject[contains(Schema, "markov")]/Id/text()' GenerationSchemas.xml

# Schemas referencing a specific wordlist
xmllint --xpath '//GenerationSchemaDataObject[contains(Schema, "real/arabic/persons/males")]/Id/text()' GenerationSchemas.xml
```

**Result**: Schema catalogue queries for integration planning.

### By Content (Entry Values)

**Goal**: Find wordlists containing specific names or patterns.

**Method**:
```bash
# Exact match
grep -r "^Aragorn$" wordlists/

# Prefix match
grep -r "^Arag" wordlists/

# Case-insensitive
grep -ri "dragon" wordlists/

# Regex pattern
grep -rE "^[A-Z][a-z]+ [A-Z][a-z]+$" wordlists/  # Two-word names
```

**Result**: Files and lines matching pattern.

### By Release Asset

**Goal**: Discover what's in a specific release without downloading.

**Method**:
```bash
# List archive contents (GitHub API)
curl -s https://api.github.com/repos/<owner>/universal-name-generator-wordlists/releases/tags/v3.2.0 | jq '.assets[0].browser_download_url' | xargs curl -sL | unzip -l

# Or download and inspect
wget https://github.com/<owner>/universal-name-generator-wordlists/releases/download/v3.2.0/universal-name-generator-wordlists_3.2.0.zip
unzip -l universal-name-generator-wordlists_3.2.0.zip
```

**Result**: Complete file manifest for that version.

## Discovery Workflows

### Contributor: Adding New Data

1. **Explore existing structure** -- `ls wordlists/` to understand taxonomy
2. **Find appropriate universe/category** -- Browse or search
3. **Check for duplicates** -- `grep -r "newentry" wordlists/`
4. **Verify schema coverage** -- Check `GenerationSchemas.xml` for existing schemas
5. **Create PR** -- Follow change guide

### Consumer: Finding Usable Schemas

1. **Browse schemas by category** -- `xmllint --xpath '//Category/text()' GenerationSchemas.xml | sort -u`
2. **Filter by algorithm** -- Randomiser vs Markov per use case
3. **Check referenced wordlists** -- Verify data scope matches need
4. **Download release** -- Get packaged data for offline use

### Integrator: Mapping Schema to Implementation

1. **Parse GenerationSchemas.xml** -- Extract all `GenerationSchemaDataObject`
2. **Group by Category** -- UI presentation
3. **Resolve Schema strings** -- Parse algorithm, parameters, paths
4. **Load wordlist files** -- Read `.lst` files from resolved paths
5. **Apply filters** -- If `FilterlistPath` present, load and apply

## Search Limitations

| Limitation | Impact | Workaround |
|------------|--------|------------|
| No full-text index | `grep -r` slow on large repos | Clone locally; use `ripgrep` |
| No semantic search | Can't search by meaning | Manual category browsing |
| No cross-reference index | Hard to find all schemas using a wordlist | XPath queries on XML |
| No web search | GitHub search limited to code | Local clone + tooling |

## Related Documentation

- Repository structure: `../repository-structure.md`
- Data model: `../data-model.md`
- Architecture: `../architecture.md` (schema string format)
- Change guide: `../change-guide.md` (adding data)
- API reference: `../api-reference/INDEX.md` (schema catalogue)
