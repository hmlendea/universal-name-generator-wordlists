# Inspect, Edit, and Delete Behaviour

This document describes how users interact with repository content for inspection, modification, and removal.

## Inspect Behaviour

### Wordlist File Inspection

**View entire file**:
```bash
cat wordlists/real/arabic/persons/males.lst
```

**View with line numbers**:
```bash
cat -n wordlists/real/arabic/persons/males.lst
```

**View first/last N entries**:
```bash
head -20 wordlists/real/arabic/persons/males.lst
tail -20 wordlists/real/arabic/persons/males.lst
```

**Count entries**:
```bash
wc -l wordlists/real/arabic/persons/males.lst
```

**Check encoding**:
```bash
file -i wordlists/real/arabic/persons/males.lst
# Output: text/plain; charset=utf-8
```

**Check for formatting issues**:
```bash
# Trailing whitespace
grep -n '[[:space:]]$' wordlists/real/arabic/persons/males.lst

# Empty lines
grep -n '^$' wordlists/real/arabic/persons/males.lst

# Non-UTF8 characters
iconv -f utf-8 -t utf-8 wordlists/real/arabic/persons/males.lst > /dev/null && echo "Valid UTF-8"
```

### Schema Inspection

**View entire schema catalogue**:
```bash
cat GenerationSchemas.xml
```

**Pretty-print XML**:
```bash
xmllint --format GenerationSchemas.xml
```

**Extract specific schema**:
```bash
xmllint --xpath '//GenerationSchemaDataObject[Id="arabic-persons-male"]' GenerationSchemas.xml
```

**List all schema IDs**:
```bash
xmllint --xpath '//Id/text()' GenerationSchemas.xml | tr ' ' '\n'
```

**List schemas by category**:
```bash
xmllint --xpath '//GenerationSchemaDataObject[Category="Arabic"]/Id/text()' GenerationSchemas.xml
```

**Inspect schema string**:
```bash
xmllint --xpath '//GenerationSchemaDataObject[Id="arabic-persons-male"]/Schema/text()' GenerationSchemas.xml
# Output: {randomiser, ,4,32,real/arabic/persons/males|real/arabic/persons/surnames}
```

**Parse schema string components**:
| Component | Example Value | Description |
|-----------|---------------|-------------|
| Algorithm | `randomiser` / `markov` | Generation method |
| Order | `4` (markov only) | N-gram order |
| Min length | `4` | Minimum output length |
| Max length | `32` | Maximum output length |
| Paths | `real/arabic/persons/males|real/arabic/persons/surnames` | Pipe-separated wordlist paths |

### Release Asset Inspection

**List archive contents without extracting**:
```bash
unzip -l universal-name-generator-wordlists_3.2.0.zip
```

**Extract and inspect specific file**:
```bash
unzip -p universal-name-generator-wordlists_3.2.0.zip wordlists/real/arabic/persons/males.lst | head
```

**Verify archive structure**:
```bash
unzip -l universal-name-generator-wordlists_3.2.0.zip | grep -E '^(wordlists/|GenerationSchemas\.xml)'
```

## Edit Behaviour

### Editing Wordlist Files

**Prerequisites**:
- Understand format: one entry per line, UTF-8, no trailing whitespace
- Changes should be additive (append) unless correcting errors

**Procedure**:
```bash
# 1. Open in editor
vim wordlists/real/arabic/persons/males.lst

# 2. Make changes (add entries, fix typos)
# 3. Validate
git diff --check wordlists/real/arabic/persons/males.lst

# 4. Check for duplicates (within file)
sort wordlists/real/arabic/persons/males.lst | uniq -d

# 5. Stage and commit
git add wordlists/real/arabic/persons/males.lst
git commit -m "arabic: add 5 male names to persons/males"
```

**Common edit scenarios**:

| Scenario | Action | Validation |
|----------|--------|------------|
| Add new names | Append to file | `git diff --check`, no duplicates |
| Fix typo | Edit line in place | `git diff --check` |
| Remove offensive entry | Delete line | `git diff --check`, document reason |
| Re-sort file | `sort -o file file` | `git diff --check`, verify no data loss |

**Batch editing**:
```bash
# Add prefix to all entries (rare)
sed -i 's/^/prefix_/' wordlists/real/arabic/persons/males.lst

# Normalise line endings
dos2unix wordlists/real/arabic/persons/males.lst
```

### Editing GenerationSchemas.xml

**Prerequisites**:
- XML well-formedness required
- Unique `Id` values required
- Referenced paths must exist

**Procedure**:
```bash
# 1. Backup
cp GenerationSchemas.xml GenerationSchemas.xml.bak

# 2. Edit with XML-aware tool or carefully with text editor
vim GenerationSchemas.xml

# 3. Validate well-formedness
xmllint --noout GenerationSchemas.xml

# 4. Validate references (advisory)
xmllint --schema GenerationSchemas.xml wordlists/**/*.lst 2>&1 | head -20

# 5. Check unique IDs
xmllint --xpath '//Id' GenerationSchemas.xml | grep -oP '(?<=>)[^<]+' | sort | uniq -d

# 6. Stage and commit
git add GenerationSchemas.xml
git commit -m "schemas: add arabic-persons-female schema"
```

**Common edit scenarios**:

| Scenario | Action | Validation |
|----------|--------|------------|
| Add new schema | Insert `GenerationSchemaDataObject` | All validations pass |
| Update Name/Category | Edit text content | Well-formed |
| Adjust min/max length | Edit `Schema` string | Well-formed, paths exist |
| Add filter list | Add `FilterlistPath` element | Path exists |
| Deprecate schema | Add comment, don't remove | Well-formed |

**Schema string editing**:
```xml
<!-- Before -->
<Schema>{randomiser, ,4,32,real/arabic/persons/males|real/arabic/persons/surnames}</Schema>

<!-- After (change max length) -->
<Schema>{randomiser, ,4,64,real/arabic/persons/males|real/arabic/persons/surnames}</Schema>

<!-- After (switch to markov) -->
<Schema>{markov,4,4,32,real/arabic/persons/males|real/arabic/persons/surnames}</Schema>
```

### Editing Release Workflow

**Procedure**:
```bash
# 1. Edit workflow
vim .github/workflows/release-asset.yml

# 2. Validate YAML syntax
yamllint .github/workflows/release-asset.yml

# 3. Test in fork (create test release)
# 4. PR with test evidence
```

## Delete Behaviour

### Deleting Wordlist Files

**Impact**: Breaks any schema referencing the file.

**Procedure (if absolutely necessary)**:
```bash
# 1. Find referencing schemas
xmllint --xpath '//GenerationSchemaDataObject[contains(Schema, "path/to/file")]/Id/text()' GenerationSchemas.xml

# 2. Coordinate with consumers (breaking change)
# 3. Atomic PR: delete file + update all referencing schemas
git rm wordlists/real/arabic/persons/old-file.lst
# Edit GenerationSchemas.xml to remove/update references
git add GenerationSchemas.xml
git commit -m "arabic: remove obsolete file, update schemas"

# 4. Major version release
```

**Preferred alternative**: Deprecate, don't delete.
- Mark schema as deprecated in documentation
- Stop using in new schemas
- Remove in future major version cleanup

### Deleting Schemas

**Impact**: Breaks consumers using that schema ID.

**Procedure (never do this directly)**:
```bash
# WRONG - breaks consumers
# xmllint --xpath '//GenerationSchemaDataObject[Id="old-id"]' GenerationSchemas.xml | xargs -I {} xmlstarlet ed -d {} GenerationSchemas.xml

# CORRECT - deprecation pattern
# 1. Add new schema with new ID
# 2. Document deprecation
# 3. Wait for consumer migration
# 4. Remove in major version (X.0.0)
```

### Deleting Filter Lists

**Impact**: Schemas with `FilterlistPath` referencing it will fail at runtime.

**Procedure**:
```bash
# 1. Find referencing schemas
xmllint --xpath '//GenerationSchemaDataObject[FilterlistPath="path/to/filter"]/Id/text()' GenerationSchemas.xml

# 2. Remove FilterlistPath from those schemas
# 3. Delete filter file
git rm wordlists/real/arabic/persons/filters.lst
git commit -m "arabic: remove unused filter list"
```

## Validation After Edits

### Local Validation Checklist

```bash
# 1. Wordlist format
find wordlists -name "*.lst" -exec sh -c 'git diff --check "$1" || exit 1' _ {} \;

# 2. XML well-formed
xmllint --noout GenerationSchemas.xml

# 3. Unique schema IDs
xmllint --xpath '//Id' GenerationSchemas.xml | grep -oP '(?<=>)[^<]+' | sort | uniq -d

# 4. Schema path references (advisory)
xmllint --schema GenerationSchemas.xml wordlists/**/*.lst 2>&1 | grep -v "validates"

# 5. No code files introduced
find . -name "*.cs" -o -name "*.js" -o -name "*.py" -o -name "*.exe" | grep -v ".git" | grep -v "docs/" && echo "CODE FILES FOUND" || echo "OK"
```

### Pre-PR Validation

```bash
# Run all checks
./validate.sh  # If validation script exists (see testing.md)

# Or manually:
git diff --check
xmllint --noout GenerationSchemas.xml
# ... other checks
```

## Related Documentation

- Change guide: `../change-guide.md` (procedures for common changes)
- Invariants: `../invariants.md` (what must remain true)
- Data model: `../data-model.md` (format specifications)
- Architecture: `../architecture.md` (schema string format)
- Testing: `../testing.md` (validation methods)
