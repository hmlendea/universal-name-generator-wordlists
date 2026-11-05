# Repository Structure

This document describes the directory layout and file organisation of the Universal Name Generator -- Wordlists repository.

## Root Directory

```
.
+-- ARCHITECTURE.md
+-- CODE_OF_CONDUCT.md
+-- CONTRIBUTING.md
+-- GenerationSchemas.xml
+-- LICENSE
+-- PRIVACY.md
+-- README.md
+-- ROADMAP.md
+-- SECURITY.md
+-- docs/
|   +-- INDEX.md
|   +-- api-reference/
|   +-- behaviour/
|   +-- components/
|   +-- flows/
|   \-- *.md (top-level docs)
+-- wordlists/
|   +-- a-song-of-ice-and-fire/
|   +-- age-of-wonders/
|   +-- age-of-wonders-planetfall/
|   +-- divinity/
|   +-- dragon-age/
|   +-- elder-scrolls/
|   +-- middle-earth/
|   +-- narivia/
|   +-- no-mans-sky/
|   +-- pinched-universe/
|   +-- real/
|   +-- runescape/
|   +-- starcraft/
|   +-- usernames/
|   +-- warcraft/
|   \-- yohoho-puzzle-pirates/
\-- .github/
    \-- workflows/
        \-- release-asset.yml
```

## Wordlists Directory

The `wordlists/` directory contains 388 `.lst` files organised by universe and category.

### Universe Structure

Each universe is a top-level directory under `wordlists/`:

| Universe | Description | Categories |
|----------|-------------|------------|
| `a-song-of-ice-and-fire/` | Game of Thrones setting | dragons (english, valyrian) |
| `age-of-wonders/` | Age of Wonders games | archon, draconian, dragon, dwarven, elven, fey, frostling, giant, goblin, halfling, human, merfolk, naga, orcish, tigran |
| `age-of-wonders-planetfall/` | Age of Wonders: Planetfall | companies, amazon, assembly, autonom, dvar, growth, human, kirko, paragon, psifish, spacer, syndicate |
| `divinity/` | Divinity games | demonic, dwarven, elven, goblin, lizard, orcish |
| `dragon-age/` | Dragon Age games | dragon |
| `elder-scrolls/` | Elder Scrolls games | argonian, dragon, dremora, elven, goblin, orcish, taagra |
| `middle-earth/` | Lord of the Rings | dragon, elven, orcish |
| `narivia/` | Narivia setting | letian, rodah |
| `no-mans-sky/` | No Man's Sky | gek, korvax, locations, traveller, vykeen |
| `pinched-universe/` | Pinched Universe | nucian, zaganian |
| `real/` | Real-world languages | 30+ language directories |
| `runescape/` | RuneScape | demonic, goblins, human |
| `starcraft/` | StarCraft | protoss, zerg |
| `usernames/` | Real platform usernames | 18 source files |
| `warcraft/` | Warcraft | dragon |
| `yohoho-puzzle-pirates/` | Yohoho Puzzle Pirates | units |

### Category Structure

Within each universe, categories vary:

| Category | Description | Example |
|----------|-------------|---------|
| `persons/` | Personal names | `males.lst`, `females.lst`, `surnames.lst` |
| `locations/` | Place names | `cities.lst`, `settlements.lst`, `toponyms.lst` |
| `groups/` | Group/organisation names | `clans.lst`, `factions.lst` |
| `units/` | Military units | `infantry.lst`, `cavalry.lst` |
| `flora/` | Plant names | `trees.lst`, `flowers.lst` |
| `fauna/` | Animal names | `beasts.lst`, `creatures.lst` |
| `companies.lst` | Flat file at universe root | `age-of-wonders-planetfall/companies.lst` |

### Real-World Languages

The `real/` directory contains 30+ language directories:

```
real/
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
\-- welsh/
```

Each language directory typically contains:
- `persons/males.lst`
- `persons/females.lst`
- `persons/surnames.lst`
- `locations/toponyms.lst`
- `persons/filters.lst` (optional)

## Schema Catalogue

`GenerationSchemas.xml` at repository root contains 328 `GenerationSchemaDataObject` entries.

### XML Structure

```xml
<ArrayOfGenerationSchemaDataObject>
  <GenerationSchemaDataObject>
    <Id>arabic-persons-male</Id>
    <Name>Arabic male persons</Name>
    <Category>Arabic</Category>
    <Schema>{randomiser, ,4,32,real/arabic/persons/males|real/arabic/persons/surnames}</Schema>
    <FilterlistPath>real/arabic/persons/filters</FilterlistPath>
  </GenerationSchemaDataObject>
  ...
</ArrayOfGenerationSchemaDataObject>
```

### Schema String Format

```
{algorithm,param1,param2,param3,path1|path2|...}
```

| Algorithm | Format |
|-----------|--------|
| Randomiser | `{randomiser, ,minLength,maxLength,path1|path2|...}` |
| Markov | `{markov,order,minLength,maxLength,path1|path2|...}` |

Paths are repository-relative under `wordlists/` without `.lst` extension.

## Documentation Directory

The `docs/` directory contains 39 markdown files organised by topic:

```
docs/
+-- INDEX.md
+-- api-reference/
|   +-- INDEX.md
|   +-- generate-name.md
|   +-- get-schema.md
|   +-- get-wordlist.md
|   +-- list-categories.md
|   +-- list-schemas.md
|   \-- list-wordlists.md
+-- behaviour/
|   +-- browse-and-search.md
|   \-- inspect-edit-delete.md
+-- components/
|   +-- application-services.md
|   +-- browser-state-and-localisation.md
|   +-- host-and-composition.md
|   +-- integration-models.md
|   \-- presentation.md
+-- flows/
|   +-- calendar-search.md
|   +-- detail-mutation.md
|   \-- startup-and-rendering.md
+-- ambiguities-and-open-questions.md
+-- api-usage-examples.md
+-- architecture.md
+-- build-and-deployment.md
+-- change-guide.md
+-- concurrency-and-scheduling.md
+-- configuration.md
+-- data-model.md
+-- dependencies.md
+-- design-decisions.md
+-- documentation-maintenance.md
+-- error-handling.md
+-- faq.md
+-- integrations.md
+-- invariants.md
+-- logging.md
+-- quick-start.md
+-- repository-overview.md
+-- repository-structure.md
+-- security.md
+-- state-and-persistence.md
+-- testing.md
\-- troubleshooting.md
```

## CI/CD Directory

`.github/workflows/` contains the release workflow:

```
.github/
\-- workflows/
    \-- release-asset.yml
```

### Release Workflow

- **Trigger**: GitHub Release published
- **Actions**: Checkout, extract version, zip `wordlists/` and `GenerationSchemas.xml`, upload as release asset
- **Output**: `universal-name-generator-wordlists_X.Y.Z.zip` with `wordlists/` and `GenerationSchemas.xml` at archive root

## File Counts

| Category | Count |
|----------|-------|
| Wordlist files (`.lst`) | 388 |
| Schema entries | 328 |
| Documentation files | 39 |
| Universes | 17 |
| Real-world languages | 30+ |
| Release workflow files | 1 |

## Naming Conventions

| Element | Convention |
|---------|------------|
| Universe directories | kebab-case |
| Category directories | kebab-case |
| Wordlist files | kebab-case + `.lst` |
| Schema IDs | kebab-case |
| Schema categories | Title Case |
| Documentation files | kebab-case + `.md` |

## Path Resolution

All paths in schema strings are repository-relative under `wordlists/`:

| Schema Path | Resolved File |
|-------------|---------------|
| `real/arabic/persons/males` | `wordlists/real/arabic/persons/males.lst` |
| `elder-scrolls/elven/persons/males` | `wordlists/elder-scrolls/elven/persons/males.lst` |
| `no-mans-sky/gek/persons/neutral` | `wordlists/no-mans-sky/gek/persons/neutral.lst` |

## Related Documentation

- [Data Model](data-model.md) -- Entity definitions
- [Architecture](architecture.md) -- Schema string grammar
- [Change Guide](change-guide.md) -- Adding new data
- [Build and Deployment](build-and-deployment.md) -- Release process
