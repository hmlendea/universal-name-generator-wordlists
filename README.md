[![Donate](https://img.shields.io/badge/-%E2%99%A5%20Donate-%23ff69b4)](https://hmlendea.go.ro/funding)
[![Latest Release](https://img.shields.io/github/v/release/hmlendea/universal-name-generator-wordlists)](https://github.com/hmlendea/universal-name-generator-wordlists/releases/latest)
[![Build Status](https://github.com/hmlendea/universal-name-generator-wordlists/actions/workflows/release-asset.yml/badge.svg)](https://github.com/hmlendea/universal-name-generator-wordlists/actions/workflows/release-asset.yml)
[![License](https://img.shields.io/github/license/hmlendea/universal-name-generator-wordlists)](https://github.com/hmlendea/universal-name-generator-wordlists/blob/master/LICENSE)

# Universal Name Generator — Wordlists

Curated wordlist assets used by the Universal Name Generator.

## 📑 Table of Contents

- [Capabilities](#-capabilities)
- [Usage](#-usage)
- [Known Limitations](#-known-limitations)
- [Installation](#-installation)
- [GitHub Actions](#-github-actions)
- [Project Structure](#-project-structure)
- [Architecture](#-architecture)
- [Roadmap](#-roadmap)
- [Documentation](#-documentation)
- [API Reference](#-api-reference)
- [Data Formats and Protocols](#-data-formats-and-protocols)
- [Troubleshooting](#-troubleshooting)
- [FAQ](#-faq)
- [Contributing](#-contributing)
- [Code of Conduct](#-code-of-conduct)
- [Related Projects](#-related-projects)
- [Security](#-security)
- [Project Engagement](#-project-engagement)
- [License](#-license)

## ✨ Capabilities

- Provides 388 UTF-8 wordlist files (`.lst`) organised by universe and category
- Supports 30+ real-world languages and 15+ fictional universes
- Defines 328 generation schemas in `GenerationSchemas.xml`
- Implements two generation algorithms: Randomiser and Markov chain
- Supports optional filter lists for post-generation exclusion
- Distributes immutable versioned archives via GitHub Releases
- Maintains schema stability through unique, stable `Id` identifiers

## 🚀 Usage

This repository is an assets-only collection. Consumers download a release archive, extract it, and implement the generation algorithms in their own applications.

### Minimal Python Example

```python
#!/usr/bin/env python3
"""Generate a name from the Universal Name Generator wordlists."""

import os
import random
import xml.etree.ElementTree as ET
from collections import defaultdict


def load_schemas(path="GenerationSchemas.xml"):
    """Load all schemas from XML catalogue."""
    tree = ET.parse(path)
    root = tree.getroot()
    schemas = []
    for obj in root.findall("GenerationSchemaDataObject"):
        schemas.append({
            "id": obj.findtext("Id"),
            "name": obj.findtext("Name"),
            "category": obj.findtext("Category"),
            "schema": obj.findtext("Schema"),
            "filterlist_path": obj.findtext("FilterlistPath"),
        })
    return schemas


def parse_schema_string(schema_str):
    """Parse {algorithm,params,paths} into structured dict."""
    inner = schema_str.strip("{}")
    parts = [p.strip() for p in inner.split(",")]
    algorithm = parts[0]
    paths = parts[-1].split("|")

    if algorithm == "randomiser":
        return {
            "algorithm": "randomiser",
            "minimum_length": int(parts[2]),
            "maximum_length": int(parts[3]),
            "paths": paths,
        }
    elif algorithm == "markov":
        return {
            "algorithm": "markov",
            "order": int(parts[1]),
            "minimum_length": int(parts[2]),
            "maximum_length": int(parts[3]),
            "paths": paths,
        }
    raise ValueError(f"Unknown algorithm: {algorithm}")


def load_wordlist(path, base_dir="wordlists"):
    """Load entries from a single .lst file."""
    full_path = os.path.join(base_dir, path + ".lst")
    with open(full_path, "r", encoding="utf-8") as f:
        return [line.strip() for line in f if line.strip()]


def load_wordlists(paths, base_dir="wordlists"):
    """Load and union entries from multiple .lst files."""
    entries = []
    for path in paths:
        entries.extend(load_wordlist(path, base_dir))
    return entries


def load_filters(filter_path, base_dir="wordlists"):
    """Load filter entries."""
    full_path = os.path.join(base_dir, filter_path + ".lst")
    with open(full_path, "r", encoding="utf-8") as f:
        return set(line.strip() for line in f if line.strip())


def apply_filter(entries, filters):
    """Remove filtered entries."""
    if not filters:
        return entries
    return [e for e in entries if e not in filters]


def generate_randomiser(entries, minimum_length=4, maximum_length=32):
    """Generate a name using random selection and concatenation."""
    for _ in range(1000):
        parts = [random.choice(entries) for _ in range(random.randint(1, 3))]
        name = "".join(parts)
        if minimum_length <= len(name) <= maximum_length:
            return name
    # Fallback
    candidates = [e for e in entries if minimum_length <= len(e) <= maximum_length]
    return random.choice(candidates) if candidates else random.choice(entries)


def build_markov_chains(entries, order=4):
    """Build n-gram transition table from entries."""
    chains = defaultdict(list)
    for entry in entries:
        padded = " " * order + entry + " "
        for i in range(len(padded) - order):
            state = padded[i:i+order]
            next_char = padded[i+order]
            chains[state].append(next_char)
    return chains


def generate_markov(entries, order=4, minimum_length=4, maximum_length=32):
    """Generate a name using the markov algorithm."""
    chains = build_markov_chains(entries, order)

    for _ in range(1000):
        state = " " * order
        result = []
        for _ in range(maximum_length + order):
            if state not in chains:
                break
            next_char = random.choice(chains[state])
            if next_char == " ":
                if minimum_length <= len(result) <= maximum_length:
                    return "".join(result)
                break
            result.append(next_char)
            state = state[1:] + next_char

    return random.choice(entries)


def generate_name(schema_id, xml_path="GenerationSchemas.xml", base_dir="wordlists"):
    """Generate a name for a given schema ID."""
    # 1. Find schema
    schemas = load_schemas(xml_path)
    schema = next((s for s in schemas if s["id"] == schema_id), None)
    if not schema:
        raise ValueError(f"Unknown schema: {schema_id}")

    # 2. Parse schema string
    params = parse_schema_string(schema["schema"])

    # 3. Load wordlists
    entries = load_wordlists(params["paths"], base_dir)

    # 4. Apply filter
    if schema["filterlist_path"]:
        filters = load_filters(schema["filterlist_path"], base_dir)
        entries = apply_filter(entries, filters)

    # 5. Generate
    if params["algorithm"] == "randomiser":
        return generate_randomiser(entries, params["minimum_length"], params["maximum_length"])
    elif params["algorithm"] == "markov":
        return generate_markov(entries, params["order"], params["minimum_length"], params["maximum_length"])


if __name__ == "__main__":
    import sys
    schema_id = sys.argv[1] if len(sys.argv) > 1 else "arabic-persons-male"
    name = generate_name(schema_id)
    print(name)
```

## ⚠️ Known Limitations

- No automated test suite; verification is manual and advisory only
- No semantic validation of generated name quality or algorithm parameter suitability
- No cross-reference validation of schema paths against actual `.lst` files in CI
- No duplicate detection within or across wordlist files
- No consumer integration tests against downstream generators
- No regression detection for changes to existing schemas or wordlists

## 📦 Installation

This repository is distributed exclusively through GitHub Releases as versioned zip archives.

[![Obtain it from GitHub](https://raw.githubusercontent.com/hmlendea/readme-assets/master/badges/stores/github.png)](https://github.com/hmlendea/universal-name-generator-wordlists/releases)

### Manual Installation

1. Visit the [releases page](https://github.com/hmlendea/universal-name-generator-wordlists/releases) and download `universal-name-generator-wordlists_X.Y.Z.zip` for the desired version.
2. Extract the archive:
   ```bash
   unzip universal-name-generator-wordlists_3.2.0.zip
   cd universal-name-generator-wordlists
   ```
3. Verify the extracted structure:
   ```bash
   ls -la
   # wordlists/  GenerationSchemas.xml
   ```

## ⚙️ GitHub Actions

| Workflow | Purpose | What it does |
|----------|---------|--------------|
| [release-asset.yml](https://github.com/hmlendea/universal-name-generator-wordlists/blob/master/.github/workflows/release-asset.yml) | Release packaging | On `release.published`, packages `wordlists/` and `GenerationSchemas.xml` into `universal-name-generator-wordlists_<version>.zip` and uploads it as a release asset |

## 🗂️ Project Structure

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

### Directories

| Directory | Purpose |
|-----------|---------|
| `wordlists/` | 388 `.lst` files organised by universe and category |
| `docs/` | 39 markdown files covering architecture, API reference, data model, testing, and more |
| `.github/workflows/` | CI/CD workflow for release packaging |

## 🏗️ Architecture

Universal Name Generator — Wordlists provides the data assets and schema catalogue for the Universal Name Generator ecosystem.

See the [Architecture Documentation](./ARCHITECTURE.md) for the system context, principal components, runtime flows, ownership boundaries, dependencies, constraints, and extension points.

## 🗺️ Roadmap

See [ROADMAP.md](./ROADMAP.md) for planned work and forthcoming changes.

## 📚 Documentation

| Resource | Description |
|----------|-------------|
| [Documentation Index](./docs/INDEX.md) | Entry point for all documentation |
| [Quick Start](./docs/quick-start.md) | Get generating names in under five minutes |
| [Repository Structure](./docs/repository-structure.md) | Directory layout and file organisation |
| [Data Model](./docs/data-model.md) | Entity definitions and relationships |
| [Architecture](./docs/architecture.md) | Implementation-level architectural detail |
| [Build and Deployment](./docs/build-and-deployment.md) | Release packaging pipeline |
| [Testing](./docs/testing.md) | Verification methods and coverage gaps |
| [Change Guide](./docs/change-guide.md) | Safe modification procedures |
| [Dependencies](./docs/dependencies.md) | External and internal dependencies |
| [Configuration](./docs/configuration.md) | Repository metadata configuration |
| [Integrations](./docs/integrations.md) | Consumer integration patterns |
| [Security](./docs/security.md) | Security considerations |
| [Error Handling](./docs/error-handling.md) | Error taxonomy and handling |
| [Logging](./docs/logging.md) | Logging considerations |
| [State and Persistence](./docs/state-and-persistence.md) | State management patterns |
| [Concurrency and Scheduling](./docs/concurrency-and-scheduling.md) | Concurrency considerations |
| [API Usage Examples](./docs/api-usage-examples.md) | Code examples for consumption |
| [FAQ](./docs/faq.md) | Frequently asked questions |
| [Troubleshooting](./docs/troubleshooting.md) | Diagnostic procedures |

## 🌐 API Reference

| Operation | Description | Document |
|-----------|-------------|----------|
| List schemas | Enumerate all generation schemas | [List Schemas](./docs/api-reference/list-schemas.md) |
| Get schema | Retrieve a single schema by ID | [Get Schema](./docs/api-reference/get-schema.md) |
| List categories | Enumerate schema categories | [List Categories](./docs/api-reference/list-categories.md) |
| List wordlists | Enumerate wordlist files | [List Wordlists](./docs/api-reference/list-wordlists.md) |
| Get wordlist | Retrieve entries from a wordlist file | [Get Wordlist](./docs/api-reference/get-wordlist.md) |
| Generate name | Produce a name from a schema | [Generate Name](./docs/api-reference/generate-name.md) |

## 🧾 Data Formats and Protocols

| Format or Protocol | Specification | Purpose |
|--------------------|---------------|---------|
| Wordlist (`.lst`) | UTF-8 plain text, one entry per line, no trailing whitespace | Name/word candidate data |
| Schema Catalogue (`GenerationSchemas.xml`) | XML with `ArrayOfGenerationSchemaDataObject` root, 328 `GenerationSchemaDataObject` entries | Generation algorithm and source mapping |
| Schema String | `{randomiser, ,min,max,path1\|path2}` or `{markov,order,min,max,path1\|path2}` | Encodes algorithm, parameters, and wordlist paths |
| Release Archive | ZIP with `wordlists/` and `GenerationSchemas.xml` at root | Distribution package |

## 🩺 Troubleshooting

See [Troubleshooting Guide](./docs/troubleshooting.md) for diagnostic procedures and resolutions to recurring problems.

## ❓ FAQ

See the [Frequently Asked Questions](./docs/faq.md) for responses to frequently raised inquiries.

## 🤝 Contributing

You are welcome to submit any suggestion, feedback, or modification to this project.

When doing so, please:
- Maintain cross-platform compatibility
- Submit focused pull requests that conform to the existing code style
- Maintain your branch synchronised with `master`
- Revise the documentation when functionality changes
- Raise a new [issue](https://github.com/hmlendea/universal-name-generator-wordlists/issues) for problems or suggestions

See the [Contributing Guidelines](./CONTRIBUTING.md) for details on how to report issues, suggest enhancements, and submit changes.

## 📜 Code of Conduct

See the [Code of Conduct](./CODE_OF_CONDUCT.md) for the standards of conduct expected in all project spaces.

## 🔗 Related Projects

- **Universal Name Generator API**: Primary consumer that embeds these wordlists and implements the generation algorithms at build time.

## 🔒 Security

Universal Name Generator — Wordlists follows the security policy documented in [SECURITY.md](./SECURITY.md).

For information on reporting security vulnerabilities, see the [Security Policy](./SECURITY.md).

## 💝 Project Engagement

Discovered a problem or have a suggestion? [Open an issue](https://github.com/hmlendea/universal-name-generator-wordlists/issues)!

If you find this project useful, consider [funding it](https://hmlendea.go.ro/funding) or starring ⭐️ it on GitHub!

[![Donate](https://raw.githubusercontent.com/hmlendea/readme-assets/master/donate_generic.png)](https://hmlendea.go.ro/funding)

## 📄 License

This project is being distributed under the `GNU General Public License v3.0 or later`.
See [LICENSE](./LICENSE) for additional information.
