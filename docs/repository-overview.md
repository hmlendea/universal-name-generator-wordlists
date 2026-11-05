# Repository Overview

## Purpose

The Universal Name Generator -- Wordlists repository is a curated data-asset collection that provides structured naming datasets and generation schemas for the Universal Name Generator ecosystem. It serves as the authoritative source for wordlist assets consumed by downstream generators at build or runtime.

## Scope

This repository owns:
- **Wordlist datasets** (`wordlists/`) -- 388 plain-text `.lst` files organised by universe (fictional settings, real-world languages) and category (persons, locations, units, flora, fauna, etc.)
- **Generation schemas** (`GenerationSchemas.xml`) -- 328 `GenerationSchemaDataObject` entries defining generation algorithms and mapping logical generators to concrete `.lst` file paths
- **Release packaging** -- GitHub Actions workflow that validates and packages assets into versioned zip archives published to GitHub Releases

This repository deliberately does not own:
- Generator application logic or runtime execution
- API behaviour or HTTP contracts
- User-facing applications or interfaces
- Build scripts, compilation, or executable artefacts

## Principal Capabilities

- Provides naming datasets for 30+ real-world languages and 15+ fictional universes
- Supports multiple generation algorithms (randomiser, markov) with configurable parameters
- Enables filtering via optional `FilterlistPath` references
- Distributes immutable versioned archives via GitHub Releases
- Maintains schema stability through unique, stable `Id` identifiers

## Entry Points

- **Data consumption**: Downstream consumers download release archives from GitHub Releases
- **Contribution**: Contributors submit `.lst` files and schema entries via pull requests
- **Validation**: Optional local validation via `xmllint --schema GenerationSchemas.xml wordlists/**/*.lst`
- **Release**: Maintainers create Git tags and GitHub Releases to trigger automated packaging

## Audience

- Maintainers of the Universal Name Generator API (primary consumer)
- Contributors adding new datasets or schemas
- Integrators validating schema compliance
- Third-party tools consuming release archives
