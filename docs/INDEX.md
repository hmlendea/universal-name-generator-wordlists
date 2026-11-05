# Documentation Index

This document provides a repository summary and a complete catalogue of the documentation set under `docs/`.

## Repository Summary

The Universal Name Generator -- Wordlists repository is a curated data-asset collection providing structured naming datasets and generation schemas for the Universal Name Generator ecosystem. It contains 388 UTF-8 plain-text wordlist files (`.lst`) organised by universe and category, plus a schema catalogue (`GenerationSchemas.xml`) with 328 `GenerationSchemaDataObject` entries defining generation algorithms and wordlist references.

**Key facts**:

| Property | Value |
|----------|-------|
| Repository type | Assets-only (no runtime code) |
| Wordlist files | 388 `.lst` files under `wordlists/` |
| Schema entries | 328 in `GenerationSchemas.xml` |
| Algorithms | Randomiser, Markov |
| Distribution | GitHub Releases (zip archive) |
| Licence | GNU GPL v3.0 |
| CI/CD | GitHub Actions (`release-asset.yml`) |

**Root documents**:
- [README.md](../README.md) -- Project overview and usage
- [ARCHITECTURE.md](../ARCHITECTURE.md) -- System architecture
- [CONTRIBUTING.md](../CONTRIBUTING.md) -- Contribution process
- [SECURITY.md](../SECURITY.md) -- Security policy
- [PRIVACY.md](../PRIVACY.md) -- Data handling
- [ROADMAP.md](../ROADMAP.md) -- Planned work
- [LICENSE](../LICENSE) -- GNU GPL v3.0

## Documentation Catalogue

### Top-Level Documents

| Document | Description |
|----------|-------------|
| [Quick Start](quick-start.md) | Minimal integration guide with working code |
| [Repository Overview](repository-overview.md) | Purpose, scope, capabilities, audience |
| [Repository Structure](repository-structure.md) | Directory layout and file organisation |
| [Architecture](architecture.md) | Schema string grammar, algorithms, path resolution |
| [Data Model](data-model.md) | Entity definitions, relationships, formats |
| [Configuration](configuration.md) | Schema configuration and parameters |
| [Dependencies](dependencies.md) | External dependencies (none at runtime) |
| [Build and Deployment](build-and-deployment.md) | Release packaging and distribution |
| [Change Guide](change-guide.md) | How to add wordlists and schemas |
| [Testing](testing.md) | Validation procedures and checks |
| [Error Handling](error-handling.md) | Error taxonomy and responses |
| [Logging](logging.md) | Logging framework (N/A, consumer responsibility) |
| [Security](security.md) | Data integrity, supply chain, consumer responsibilities |
| [State and Persistence](state-and-persistence.md) | Consumer-side state patterns |
| [Concurrency and Scheduling](concurrency-and-scheduling.md) | Consumer-side concurrency patterns |
| [Integrations](integrations.md) | Distribution and access integration points |
| [Troubleshooting](troubleshooting.md) | Diagnostic procedures and resolution steps |
| [FAQ](faq.md) | Frequently asked questions |
| [Design Decisions](design-decisions.md) | Key decisions and rationale |
| [Invariants](invariants.md) | Repository invariants and guarantees |
| [Ambiguities and Open Questions](ambiguities-and-open-questions.md) | Spec interpretations and open items |
| [Documentation Maintenance](documentation-maintenance.md) | How to maintain this documentation set |

### API Reference

| Document | Description |
|----------|-------------|
| [API Reference Index](api-reference/INDEX.md) | API surface overview and access patterns |
| [List Schemas](api-reference/list-schemas.md) | Enumerate all generation schemas |
| [Get Schema](api-reference/get-schema.md) | Retrieve a single schema by ID |
| [List Categories](api-reference/list-categories.md) | Enumerate schema categories |
| [List Wordlists](api-reference/list-wordlists.md) | Enumerate wordlist files |
| [Get Wordlist](api-reference/get-wordlist.md) | Retrieve entries from a wordlist file |
| [Generate Name](api-reference/generate-name.md) | Produce a name from a schema |

### Behaviour

| Document | Description |
|----------|-------------|
| [Browse and Search](behaviour/browse-and-search.md) | Schema discovery and search patterns |
| [Inspect, Edit, Delete](behaviour/inspect-edit-delete.md) | Schema inspection and data editing patterns |

### Components

| Document | Description |
|----------|-------------|
| [Presentation](components/presentation.md) | Data packaging and formatting |
| [Host and Composition](components/host-and-composition.md) | Distribution and composition model |
| [Application Services](components/application-services.md) | Data services and generation algorithms |
| [Browser State and Localisation](components/browser-state-and-localisation.md) | Consumer-side state and localisation |
| [Integration Models](components/integration-models.md) | Consumer integration patterns |

### Flows

| Document | Description |
|----------|-------------|
| [Startup and Rendering](flows/startup-and-rendering.md) | Consumer startup and rendering flow |
| [Calendar Search](flows/calendar-search.md) | Schema discovery flow (calendar search analogue) |
| [Detail Mutation](flows/detail-mutation.md) | Parameter adjustment and generation flow |

### Examples

| Document | Description |
|----------|-------------|
| [API Usage Examples](api-usage-examples.md) | Complete code examples for common operations |

## Reading Order

### New Consumers

1. [Quick Start](quick-start.md) -- Get generating names immediately
2. [Repository Overview](repository-overview.md) -- Understand scope and capabilities
3. [Data Model](data-model.md) -- Learn schema and wordlist formats
4. [API Reference Index](api-reference/INDEX.md) -- Explore the API surface
5. [API Usage Examples](api-usage-examples.md) -- Copy working code

### Contributors

1. [Repository Structure](repository-structure.md) -- Understand layout
2. [Change Guide](change-guide.md) -- Learn how to add data
3. [Testing](testing.md) -- Validate changes
4. [Invariants](invariants.md) -- Understand guarantees
5. [Documentation Maintenance](documentation-maintenance.md) -- Keep docs current

### Integrators

1. [Integrations](integrations.md) -- Choose integration method
2. [Build and Deployment](build-and-deployment.md) -- Understand distribution
3. [Security](security.md) -- Supply-chain integrity
4. [State and Persistence](state-and-persistence.md) -- Consumer state patterns
5. [Concurrency and Scheduling](concurrency-and-scheduling.md) -- Consumer concurrency patterns

## Related Documentation

- [Root README.md](../README.md) -- Project overview
- [Root ARCHITECTURE.md](../ARCHITECTURE.md) -- System architecture
- [Root CONTRIBUTING.md](../CONTRIBUTING.md) -- Contribution process
- [Root SECURITY.md](../SECURITY.md) -- Security policy
- [Root PRIVACY.md](../PRIVACY.md) -- Data handling
- [Root ROADMAP.md](../ROADMAP.md) -- Planned work
- [Root LICENSE](../LICENSE) -- GNU GPL v3.0
