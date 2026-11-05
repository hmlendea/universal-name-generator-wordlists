# Frequently Asked Questions

This document answers common questions about the repository, its data, and its usage.

## General

### What is this repository?
A curated collection of wordlists (name datasets) and generation schemas for the Universal Name Generator. It contains only data assets -- no application code.

### What does it contain?
- 388 UTF-8 `.lst` wordlist files (one entry per line)
- `GenerationSchemas.xml` with 328 generation schema definitions
- GitHub Actions workflow for release packaging
- Root documentation (README, ARCHITECTURE, ROADMAP, etc.)

### What is it not?
- Not an application (no runtime code)
- Not a library (no importable package)
- Not a service (no HTTP API)
- Not a database (no query engine)

### Who uses it?
- **Consumers**: Applications that generate names (games, tools, websites)
- **Contributors**: People adding new wordlists or schemas
- **Integrators**: Developers embedding data in their products

### What license applies?
GNU General Public License v3.0 (see `LICENSE`). Any derivative work distributing this data must also be GPL v3.0.

## Data

### What format are wordlist files?
UTF-8 plain text, one entry per line, no trailing whitespace, no empty lines, no headers.

### What is a schema?
A `GenerationSchemaDataObject` in `GenerationSchemas.xml` defining:
- `Id`: Unique identifier (kebab-case)
- `Name`: Human-readable name
- `Category`: Grouping label
- `Schema`: Algorithm string (`{algorithm,params,paths}`)
- `FilterlistPath`: Optional filter list reference

### What algorithms are supported?
- **Randomiser**: Uniform random selection from wordlist entries
- **Markov**: N-gram chain generation (typically order 4)

### What is a filter list?
An optional `.lst` file listing entries to exclude from generation (e.g., offensive words, duplicates). Referenced by `FilterlistPath` in a schema.

### How many languages/universes are covered?
- 30+ real-world languages (Arabic, Chinese, Japanese, Russian, etc.)
- 15+ fictional universes (Elder Scrolls, Warcraft, StarCraft, etc.)
- 18 username sources (Reddit, GitHub, Steam, etc.)

### Are entries sorted?
Not guaranteed. Order is significant for Markov training; irrelevant for randomiser.

### Are there duplicates?
Possible. No automated duplicate detection exists (planned per ROADMAP.md).

## Usage

### How do I use the data in my application?
1. Download a release archive (ZIP)
2. Extract to a known location
3. Parse `GenerationSchemas.xml`
4. Load referenced `.lst` files
5. Implement randomiser/markov algorithms
6. Apply filters if present

See `quick-start.md` and `api-usage-examples.md` for code examples.

### How do I get the latest version?
- GitHub Releases page: `https://github.com/<owner>/universal-name-generator-wordlists/releases`
- GitHub API: `GET /repos/<owner>/universal-name-generator-wordlists/releases/latest`

### Can I use a specific version?
Yes. Download the archive for a specific tag:
```
https://github.com/<owner>/universal-name-generator-wordlists/releases/download/v3.2.0/universal-name-generator-wordlists_3.2.0.zip
```

### How do I find schemas for a specific category?
```bash
xmllint --xpath '//GenerationSchemaDataObject[Category="Arabic"]/Id/text()' GenerationSchemas.xml
```

### How do I find all schemas using a wordlist?
```bash
xmllint --xpath '//GenerationSchemaDataObject[contains(Schema, "real/arabic/persons/males")]/Id/text()' GenerationSchemas.xml
```

### Can I generate names without downloading?
No. This repository provides data only. Generation happens in consumer applications.

## Contributing

### How do I add a new wordlist?
1. Create `.lst` file in appropriate `wordlists/<universe>/<category>/` directory
2. Follow format conventions (UTF-8, one per line, no trailing whitespace)
3. Add schema entry in `GenerationSchemas.xml`
4. Submit PR

See `change-guide.md` for detailed procedures.

### How do I add a new schema?
1. Determine unique `Id` (kebab-case)
2. Add `GenerationSchemaDataObject` to `GenerationSchemas.xml`
3. Validate XML well-formedness
4. Verify referenced paths exist
5. Submit PR

### Can I modify existing schemas?
- **Safe**: Update `Name`, `Category`, adjust parameters (min/max length)
- **Breaking**: Change `Id`, remove schema, change paths -- requires coordination

See `change-guide.md` for the deprecation pattern.

### Can I delete a wordlist or schema?
Not recommended. Deprecate instead:
1. Add replacement (if applicable)
2. Document deprecation
3. Wait for consumer migration
4. Remove in future major version

### What happens if I break something?
- **Data errors**: Caught in PR review or post-release verification
- **Breaking changes**: Consumer build/generation failures
- **Recovery**: Delete release, fix, re-release (see `error-handling.md`)

## Releases

### How are releases created?
1. Maintainer creates Git tag `vX.Y.Z`
2. Maintainer creates GitHub Release from tag
3. `release-asset.yml` workflow triggers on `release.published`
4. Workflow packages `wordlists/` + `GenerationSchemas.xml` into ZIP
5. Workflow uploads ZIP as release asset

### What is the archive structure?
```
universal-name-generator-wordlists_X.Y.Z.zip
+-- wordlists/          (all 388 .lst files)
+-- GenerationSchemas.xml
```
No wrapper directory. Contents at archive root.

### How is versioning done?
Semantic Versioning via Git tags (`vX.Y.Z`):
- **Patch (Z)**: New wordlists, new schemas, parameter adjustments
- **Minor (Y)**: New algorithms (additive)
- **Major (X)**: Breaking changes (schema ID removal, path restructuring, archive layout changes)

### Can I request a release?
File a GitHub Issue requesting specific data or a release. Maintainers decide cadence.

## Troubleshooting

### My consumer can't parse GenerationSchemas.xml
- Verify file is valid XML: `xmllint --noout GenerationSchemas.xml`
- Verify UTF-8 encoding: `file -i GenerationSchemas.xml`
- Check for truncation (download incomplete)

### A schema references a missing file
- Verify path exists: `ls wordlists/<path>.lst`
- Check if file was moved (breaking change)
- File GitHub issue with schema ID and missing path

### Generated names look wrong
- Verify algorithm implementation (randomiser vs markov)
- Check parameter parsing (min/max length, order)
- Verify wordlist loading (correct paths, no empty entries)
- Check filter application (if filterlist present)

### Archive structure doesn't match expectations
- Verify you downloaded the ZIP, not a source tarball
- Check `unzip -l` output for root layout
- File GitHub issue if structure violated

## Governance

### Who maintains this repository?
See `README.md` and `CONTRIBUTING.md` for maintainer information.

### How do I report a security issue?
See `SECURITY.md` for responsible disclosure process.

### What is the roadmap?
See `ROADMAP.md` for planned enhancements (CI validation, duplicate detection, etc.).

### Can I fork this repository?
Yes, under GPL v3.0. Derivative works must also be GPL v3.0.

## Related Documentation

- Quick start: `./quick-start.md`
- Change guide: `./change-guide.md`
- Error handling: `./error-handling.md`
- Troubleshooting: `./troubleshooting.md`
- Architecture: `./architecture.md`
- Root documents: `../README.md`, `../ARCHITECTURE.md`, `../ROADMAP.md`
