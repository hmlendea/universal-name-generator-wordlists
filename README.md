[![Donate](https://img.shields.io/badge/-%E2%99%A5%20Donate-%23ff69b4)](https://hmlendea.go.ro/fund.html)
[![Latest Release](https://img.shields.io/github/v/release/hmlendea/universal-name-generator)](https://github.com/hmlendea/universal-name-generator/releases/latest)
[![Build Status](https://github.com/hmlendea/universal-name-generator/actions/workflows/dotnet.yml/badge.svg)](https://github.com/hmlendea/universal-name-generator/actions/workflows/dotnet.yml)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://gnu.org/licenses/gpl-3.0)

# Universal Name Generator

Universal Name Generator is a cross-platform console application for generating names from curated wordlists and generation schemas.

It combines predefined word fragments, full-name lists, and Markov-chain generation to produce names for real-world languages, fictional settings, places, people, ships, usernames, and more.

## What It Does

- loads generation schemas from `GenerationSchemas.xml`
- loads source wordlists from the `Wordlists/` directory
- groups generators into interactive menu categories
- prints batches of generated names directly in the terminal
- supports both combinational generation and Markov-based generation

The application currently starts as an interactive terminal menu. There are no command-line subcommands or flags exposed by the project itself.

## Included Data

The repository ships with wordlists for both real and fictional sources, including:

- real-world languages and naming traditions
- fantasy races and worlds
- science-fiction factions and species
- game-inspired naming sets
- username-oriented wordlists

Examples from the current dataset include Arabic, English, French, German, Irish, Welsh, RuneScape, No Man's Sky, StarCraft, Divinity, Elder Scrolls, and Age of Wonders.

## Requirements

- .NET SDK/runtime targeting `net10.0`

## Quick Start

```bash
dotnet run
```

When the application starts, it will:

1. show a list of categories
2. open a submenu for the selected category
3. generate and print a batch of names for the selected schema

The current menu flow generates 60 names per selection and prints them in columns sized to the terminal width.

## How Generation Works

Each generator is defined by a schema entry in `GenerationSchemas.xml`. A schema declares:

- a unique identifier
- a display name
- a category shown in the main menu
- a generation expression
- an optional filter list
- an output casing mode

The application supports these schema operations:

| Operation | Purpose |
| --- | --- |
| `random` | Creates strings by sampling from explicit character or fragment choices |
| `randomiser` | Builds names from one or more wordlists |
| `markov` | Learns patterns from source words and produces new names |

### Schema Examples

Randomiser example:

```xml
<GenerationSchemaEntity>
	<Id>romanian-persons-male</Id>
	<Name>Romanian male persons</Name>
	<Category>Romanian</Category>
	<Schema>{randomiser, ,4,32,real/romanian/persons/males|real/romanian/persons/surnames}</Schema>
</GenerationSchemaEntity>
```

Markov example:

```xml
<GenerationSchemaEntity>
	<Id>latin-toponyms</Id>
	<Name>Latin toponyms</Name>
	<Category>Latin</Category>
	<Schema>{markov,4,24,real/latin/locations/toponyms}</Schema>
</GenerationSchemaEntity>
```

### Filter Lists

Some schemas define `FilterlistPath`. When present, the generator loads `Wordlists/<filterlist>.lst` and excludes matching strings from the output.

This is useful for avoiding existing names, common collisions, or source words that should not be reproduced.

## Project Structure

```text
.
|- Program.cs
|- GenerationSchemas.xml
|- Menus/
|- Service/
|- DataAccess/
`- Wordlists/
```

- `Program.cs`: application entry point
- `Menus/`: interactive CLI menus
- `Service/`: schema loading and name generation logic
- `DataAccess/`: XML and wordlist access helpers
- `GenerationSchemas.xml`: generator definitions
- `Wordlists/`: source data used by the generators

## Adding New Generators

To add a new name set:

1. add one or more `.lst` files under `Wordlists/`
2. add a new schema entry to `GenerationSchemas.xml`
3. set `Category` to control where it appears in the main menu
4. set `WordCase` if you need something other than title case
5. run `dotnet run` and verify the new generator appears and produces sensible output

### Wordlist Format

Wordlists are plain text files with one entry per line.

Example:

```text
al
dor
mar
ther
```

### Choosing a Generator Type

Use `randomiser` when you want to combine known fragments or known name parts.

Use `markov` when you want names that resemble the training data but are not limited to exact combinations of the original input pieces.

Use `random` only for very simple synthetic patterns built from explicit choices.

## Development

### Build

```bash
dotnet build
```

### Run

```bash
dotnet run
```

### Publish

The repository includes `release.sh`, which delegates to the upstream deployment script used by the project maintainer.

```bash
bash ./release.sh 1.0.0
```

This script downloads and executes an external release helper from: `https://raw.githubusercontent.com/hmlendea/deployment-scripts/master/release/dotnet/10.0.sh`

**Note:** Piping into `bash` is an intensely controversial topic. Please review any external scripts before running them in your environment!

## Contributing

Contributions are welcome, especially:

- new language datasets
- new fictional naming sets
- improved schema quality
- duplicate and collision filtering
- better documentation
- bug fixes and portability improvements

Please:

- keep changes cross-platform
- preserve public APIs unless the change is intentionally breaking
- keep pull requests focused and consistent with existing style
- update documentation when behaviour changes
- add or update tests for new behaviour

Feel free to contribute to this repository!

All contributions are welcome, especially those that bring new languages or new categories to the existing ones.
The goal of this project is to support as many languages as possible, and that's not possible without the help of the community.

Note: This project seeks to include fictional languages as well, so feel free to contribute your Valyrian, Klingon or Daedric skills to our cause!

## License

Licensed under the GNU General Public License v3.0 or later.
See [LICENSE](./LICENSE) for details.
