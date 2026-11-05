# Design Decisions

This document records key architectural and design decisions with rationale. Decisions are derived from repository evidence and documented intent.

## Decision: Assets-Only Repository

**Decision**: This repository contains only data assets (`.lst` files, `GenerationSchemas.xml`) and CI/CD configuration. No application code, build scripts, or runtime components.

**Rationale**:
- Separation of concerns: data curation vs. generator implementation
- Allows independent versioning and release cycles for data vs. code
- Enables multiple downstream consumers (API, CLI tools, other generators) to consume same assets
- Reduces repository complexity and attack surface
- Aligns with "curated data-asset collection" architectural style

**Evidence**: `ARCHITECTURE.md` Purpose section; `README.md` "assets-only repository" statement; no code files in repository.

**Consequences**:
- Consumers must implement generation algorithms themselves
- No reference implementation in this repository
- Schema semantics documented but not enforced here
- Quality of generated names depends on consumer implementation

## Decision: Plain Text Wordlist Format (`.lst`)

**Decision**: Wordlists are UTF-8 plain text files with one entry per line, no trailing whitespace.

**Rationale**:
- Maximum portability across languages and platforms
- Human-readable and editable with any text editor
- Line-oriented format maps naturally to array/list loading in any language
- No parsing ambiguity or format versioning needed
- Git-friendly (diffable, mergeable)

**Evidence**: `README.md` "Each line in a `.lst` file represents one candidate value"; `CONTRIBUTING.md` code style requirements; `ARCHITECTURE.md` Data Architecture table.

**Consequences**:
- No metadata per entry (frequency, gender, region, etc.)
- No hierarchical structure within a file
- Consumers must infer semantics from file path/category
- Extensibility requires new files or schema changes

## Decision: XML Schema Catalogue (`GenerationSchemas.xml`)

**Decision**: Generation schemas defined in a single XML file with `ArrayOfGenerationSchemaDataObject` root containing `GenerationSchemaDataObject` elements.

**Rationale**:
- XML provides structured, validated format with tooling support (`xmllint`)
- Single file simplifies distribution and versioning
- `Id`/`Name`/`Category`/`Schema`/`FilterlistPath` fields map to consumer needs
- Extensible for future fields without breaking existing consumers
- .NET ecosystem (primary consumer) has strong XML support

**Evidence**: `GenerationSchemas.xml` structure; `ARCHITECTURE.md` Generation Schemas section; 328 entries in single file.

**Consequences**:
- Schema changes require XML editing (not JSON/YAML)
- Path references are strings; no compile-time validation
- Algorithm parameters encoded in `Schema` string (custom format)
- Consumers must parse `Schema` string format

## Decision: Custom Schema String Format

**Decision**: Generation algorithm and parameters encoded in a custom string format within `Schema` field:
- `{randomiser, ,min,max,path1|path2}`
- `{markov,order,min,max,path1|path2}`

**Rationale**:
- Compact representation fitting in single XML attribute/element
- Human-readable for review and debugging
- Extensible for new algorithms (new prefix in braces)
- Parameters map directly to algorithm requirements

**Evidence**: `GenerationSchemas.xml` content; `ARCHITECTURE.md` Extension Points section.

**Consequences**:
- Consumers must implement custom parser
- No standard schema validation for `Schema` string content
- Parameter order and count fixed per algorithm
- Adding parameters requires format versioning or new algorithm

## Decision: Two Generation Algorithms (Randomiser + Markov)

**Decision**: Support only two algorithms: `randomiser` (uniform selection) and `markov` (n-gram chain).

**Rationale**:
- Covers primary name generation needs: combination vs. synthesis
- Randomiser: combines existing name parts (given + surname)
- Markov: generates novel names respecting phonotactics
- Simplicity: two well-understood algorithms sufficient for current scope
- Avoids algorithm proliferation without clear need

**Evidence**: `GenerationSchemas.xml` -- all 328 entries use only these two algorithms.

**Consequences**:
- No weighted selection, phonetic matching, or ML-based generation
- Consumers needing other algorithms must implement separately
- Algorithm choice baked into schema; not configurable at runtime

## Decision: Optional Filter Lists

**Decision**: Filter lists referenced via optional `FilterlistPath` in schema; same `.lst` format as wordlists.

**Rationale**:
- Separates generation from post-processing
- Reusable across schemas (same filter for multiple generators)
- Same format as wordlists (no new format)
- Optional: schemas work without filters

**Evidence**: `GenerationSchemas.xml` entries with `FilterlistPath` (e.g., German, Romanian persons); `ARCHITECTURE.md` Filter Lists section.

**Consequences**:
- Filtering behaviour defined by consumer (not enforced here)
- No standard filter categories (profanity, duplicates, length)
- Filter list maintenance separate from wordlist maintenance
- Consumer must handle missing filter gracefully

## Decision: GitHub Releases as Sole Distribution

**Decision**: Versioned zip archives published exclusively to GitHub Releases.

**Rationale**:
- Native GitHub integration (no external services)
- Immutable assets with version tags
- Public downloads without authentication
- Integrates with GitHub Actions for automated packaging
- Consumers can use `gh` CLI or HTTPS for retrieval

**Evidence**: `ARCHITECTURE.md` System Context diagram; `release-asset.yml` workflow; `README.md` Latest Release badge.

**Consequences**:
- No package manager distribution (NuGet, npm, etc.)
- No CDN or mirror strategy
- Consumers must handle GitHub API rate limits if automated
- Single point of failure (GitHub availability)

## Decision: Archive Layout Fixed (No Wrapper Directory)

**Decision**: Release archive root contains `wordlists/` and `GenerationSchemas.xml` directly.

**Rationale**:
- Predictable structure for consumer extraction logic
- Avoids extra directory level in consumer build paths
- Matches consumer expectation documented in `ARCHITECTURE.md`

**Evidence**: `release-asset.yml` `zip -r "$archive_name" wordlists GenerationSchemas.xml`; `ARCHITECTURE.md` Release Packaging section.

**Consequences**:
- Changing layout breaks all consumers
- Cannot include additional metadata files at root
- Consumer extraction logic coupled to this layout

## Decision: Semantic Versioning via Git Tags

**Decision**: Releases tagged as `vX.Y.Z`; archive named `universal-name-generator-wordlists_X.Y.Z.zip`.

**Rationale**:
- Standard SemVer communicates change impact
- `v` prefix convention for Git tags
- Workflow strips `v` for archive name (cleaner)
- Enables consumer version pinning and dependency management

**Evidence**: `release-asset.yml` version extraction; `ARCHITECTURE.md` Compatibility Contracts table.

**Consequences**:
- Tag format change breaks workflow version extraction
- No pre-release or build metadata in archive name
- Consumers must parse version from tag or archive name

## Decision: No Automated Validation in CI

**Decision**: CI workflow only packages and uploads; no `xmllint` validation, no duplicate detection, no path reference checking.

**Rationale**:
- Simplicity: workflow stays minimal and fast
- Validation is advisory (documented in CONTRIBUTING.md)
- Contributors run validation locally
- Avoids false positives blocking legitimate contributions

**Evidence**: `release-asset.yml` has no validation steps; `CONTRIBUTING.md` documents optional validation; `ROADMAP.md` lists validation enforcement as planned.

**Consequences**:
- Invalid schemas or broken paths can reach release
- Duplicate entries accumulate undetected
- Formatting inconsistencies not caught automatically
- Quality relies on PR review diligence

## Decision: Directory Taxonomy by Universe to Category

**Decision**: `wordlists/<universe>/<category>/<file>.lst` with lowercase hyphen-separated names.

**Rationale**:
- Logical grouping by fictional setting or language family
- Scales to hundreds of datasets without flat namespace collision
- Category subdirectories enable partial consumption (e.g., only `persons/`)
- Consistent naming aids discovery and tooling

**Evidence**: `wordlists/` directory structure; `ARCHITECTURE.md` Architectural Areas section; `repository-structure.md`.

**Consequences**:
- Moving files between categories/universes breaks schema paths
- New universe requires new top-level directory
- `real/` language grouping differs from fictional universe grouping
- `usernames/` is flat (platform datasets don't fit taxonomy)

## Decision: Stable Schema Ids (No Renaming)

**Decision**: `GenerationSchemaDataObject.Id` values are immutable; changing an Id is a breaking change.

**Rationale**:
- Consumers reference schemas by Id in code/configuration
- Id stability enables consumer version pinning
- New schemas get new Ids (additive, non-breaking)
- Clear breaking change policy

**Evidence**: `ARCHITECTURE.md` Compatibility Contracts table; `GenerationSchemas.xml` Id values (e.g., `arabic-persons-male`); `architecture.md` Versioning and Compatibility section.

**Consequences**:
- Typos in Ids are permanent (must deprecate, not rename)
- Id namespace must be managed carefully
- Consumer migration required for any Id change

## Decision: No Runtime, No Tests

**Decision**: Repository has no runtime execution and no automated test suite.

**Rationale**:
- Assets-only repository has no behaviour to test at runtime
- Generation algorithms execute in consumers, not here
- Validation is structural (XML, formatting), not semantic
- Test infrastructure would add complexity without value

**Evidence**: `ARCHITECTURE.md` Runtime Flow ("no runtime execution"); `TESTING.md`; `ROADMAP.md` (no test infrastructure planned).

**Consequences**:
- Semantic correctness (name quality) not verified
- Algorithm parameter suitability not validated
- Consumer integration not tested
- Regression detection relies on consumer feedback

## Decision: GPL v3.0 License

**Decision**: Repository licensed under GNU General Public License v3.0.

**Rationale**:
- Copyleft ensures derivative works remain free
- Compatible with downstream GPL projects
- Standard, well-understood license
- Aligns with Universal Name Generator project licensing

**Evidence**: `LICENSE` file; `README.md` license badge; `SECURITY.md` scope.

**Consequences**:
- Consumers must comply with GPL v3.0
- Proprietary downstream use requires license consideration
- Contributions inherit GPL v3.0
