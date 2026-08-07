[![Donate](https://img.shields.io/badge/-%E2%99%A5%20Donate-%23ff69b4)](https://hmlendea.go.ro/funding)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://gnu.org/licenses/gpl-3.0)

# Universal Name Generator — Wordlists

This repository contains the curated wordlist assets used by the Universal Name Generator.

It is an assets-only repository. No application code is included here.

## 📑 Table of Contents

- [✨ Features](#-features)
- [Usage](#-usage)
- [Known Limitations](#-known-limitations)
- [Project Structure](#-project-structure)
- [Contributing](#-contributing)
- [Related Projects](#-related-projects)
- [Helping out](#-helping-out)
- [License](#-license)

## ✨ Features

The `wordlists/` directory contains plain text `.lst` files organised by universe and category. Each file contains one entry per line, either full names or word fragments used for name generation.

Wordlists are provided for both real-world languages and fictional settings, including:

- real-world languages and naming traditions (Arabic, English, French, German, Irish, Romanian, and many more)
- fantasy races and worlds (Age of Wonders, Divinity, Elder Scrolls, Middle-earth)
- science-fiction factions and species (No Man's Sky, StarCraft, Age of Wonders: Planetfall)
- game-inspired naming sets (RuneScape, Yohoho! Puzzle Pirates)
- username-oriented wordlists collected from various online communities

## 🚀 Usage

Use these wordlists as source data for name generation tools that read newline-delimited entries.

```text
wordlists/real/romanian/phone-numbers/phone-numbers.lst
```

Each line in a `.lst` file represents one candidate value.

## ⚠️ Known Limitations

- This repository includes only data files and schemas; it does not include the generator application logic.
- Data quality and style vary across universes and language groups because sources are intentionally diverse.

## 🗂️ Project Structure

The key directories are:

| Directory | Purpose |
|-----------|---------|
| `wordlists/` | All naming datasets, organised by setting and category |
| `.github/` | Repository metadata such as funding configuration |

Each wordlist is a plain text file with one entry per line:

```
al
dor
mar
ther
```

Files are named and organised to reflect their intended use (e.g. `persons/males`, `locations/cities`, `units`).

## 🤝 Contributing

You are welcome to bring any suggestion, feedback or modification to this project.

When doing so, please:
- Maintain cross-platform compatibility
- Maintain the pull requests as focused and consistent with the existing code style
- Maintain your branch up-to-date with `master`
- Revise the documentation when behaviour changes
- Properly test all changes

Contributions are welcome, especially:

- new language datasets
- new fictional naming sets
- improved wordlist quality
- duplicate and collision filtering

Please keep pull requests focused and consistent with the existing naming and directory structure.

All contributions are welcome, especially those that bring new languages or new categories to the existing ones.
The goal of this project is to support as many languages as possible, and that is not possible without the help of the community.

Note: This project seeks to include fictional languages as well, so feel free to contribute your Valyrian, Klingon or Daedric skills to our cause!

## 🔗 Related Projects

- [Universal Name Generator API](https://github.com/hmlendea/universal-name-generator-api): The API that consumes these datasets to generate names
- [Universal Name Generator](https://github.com/hmlendea/universal-name-generator): The main project ecosystem entry point

## 💝 Helping out

Discovered a problem or have a suggestion? [Open an issue](https://github.com/hmlendea/universal-name-generator-wordlists/issues)!

If you find this project useful, consider [funding it](https://hmlendea.go.ro/funding) or starring ⭐️ it on GitHub!

[![Donate](https://raw.githubusercontent.com/hmlendea/readme-assets/master/donate_generic.png)](https://hmlendea.go.ro/funding)

## 📄 License

This project is being distributed under the `GNU General Public License v3.0` or later.
See [LICENSE](./LICENSE) for details.
