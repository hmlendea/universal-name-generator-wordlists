# Integrations

This document describes integration points for the Universal Name Generator -- Wordlists repository. Since this is an assets-only repository with no runtime, integrations are consumer-side and distribution-side.

## Integration Model

### Repository as Data Provider

| Role | Entity | Responsibility |
|------|--------|----------------|
| Provider | This repository | Curate wordlists, define schemas, distribute releases |
| Consumer | Universal Name Generator application | Load data, implement algorithms, generate names |
| Integrator | Third-party applications | Embed or reference repository data |

### Integration Points

| Point | Type | Direction | Protocol |
|-------|------|-----------|----------|
| GitHub Releases | Distribution | Outbound | HTTPS download |
| GitHub API | Metadata | Outbound | REST API |
| Git clone | Distribution | Outbound | Git protocol |
| Release archive | Packaging | Outbound | ZIP archive |
| Schema catalogue | Data contract | Outbound | XML document |
| Wordlist files | Data contract | Outbound | UTF-8 text |

## GitHub Releases Integration

### Download Latest Release

```bash
# Get latest release info
curl -s https://api.github.com/repos/hmlendea/universal-name-generator-wordlists/releases/latest | \
  jq -r '.assets[0].browser_download_url'

# Download latest release
curl -L -o wordlists.zip \
  https://github.com/hmlendea/universal-name-generator-wordlists/releases/latest/download/universal-name-generator-wordlists.zip
```

### Download Specific Version

```bash
# Get release by tag
curl -s https://api.github.com/repos/hmlendea/universal-name-generator-wordlists/releases/tags/v1.0.0 | \
  jq -r '.assets[0].browser_download_url'

# Download specific version
curl -L -o wordlists.zip \
  https://github.com/hmlendea/universal-name-generator-wordlists/releases/download/v1.0.0/universal-name-generator-wordlists_1.0.0.zip
```

### List All Releases

```bash
curl -s https://api.github.com/repos/hmlendea/universal-name-generator-wordlists/releases | \
  jq -r '.[] | "\(.tag_name) \(.published_at) \(.assets[0].browser_download_url)"'
```

## GitHub API Integration

### Repository Metadata

```bash
# Repository info
curl -s https://api.github.com/repos/hmlendea/universal-name-generator-wordlists | \
  jq '{name, description, license: .license.spdx_id, updated_at}'

# Latest commit
curl -s https://api.github.com/repos/hmlendea/universal-name-generator-wordlists/commits/main | \
  jq '{sha: .sha, date: .commit.committer.date, message: .commit.message}'
```

### Content Access

```bash
# Get file content (base64 encoded)
curl -s https://api.github.com/repos/hmlendea/universal-name-generator-wordlists/contents/GenerationSchemas.xml | \
  jq -r '.content' | base64 -d

# List wordlists directory
curl -s https://api.github.com/repos/hmlendea/universal-name-generator-wordlists/contents/wordlists | \
  jq -r '.[].name'
```

### Rate Limits

| Client | Rate Limit | Notes |
|--------|------------|-------|
| Unauthenticated | 60 requests/hour | Per IP address |
| Authenticated | 5000 requests/hour | Per user/token |
| GitHub Actions | 1000 requests/hour | Per repository |

**Recommendation**: Authenticate API requests to avoid rate limits.

## Git Integration

### Clone for Development

```bash
git clone https://github.com/hmlendea/universal-name-generator-wordlists.git
cd universal-name-generator-wordlists
```

### Clone for Consumption (Shallow)

```bash
# Shallow clone (latest only)
git clone --depth 1 https://github.com/hmlendea/universal-name-generator-wordlists.git

# Sparse checkout (wordlists only)
git clone --filter=blob:none --sparse https://github.com/hmlendea/universal-name-generator-wordlists.git
cd universal-name-generator-wordlists
git sparse-checkout set wordlists GenerationSchemas.xml
```

### Submodule Integration

```bash
# Add as submodule
git submodule add https://github.com/hmlendea/universal-name-generator-wordlists.git extern/wordlists

# Update submodule
git submodule update --remote extern/wordlists
```

### Subtree Integration

```bash
# Add as subtree
git subtree add --prefix=extern/wordlists \
  https://github.com/hmlendea/universal-name-generator-wordlists.git main --squash

# Pull updates
git subtree pull --prefix=extern/wordlists \
  https://github.com/hmlendea/universal-name-generator-wordlists.git main --squash
```

## Release Archive Integration

### Embed in Application

```python
import zipfile
import io
import requests

def load_embedded_wordlists(release_url):
    """Download and extract release archive in memory."""
    response = requests.get(release_url, timeout=30)
    response.raise_for_status()

    with zipfile.ZipFile(io.BytesIO(response.content)) as archive:
        # Load schema catalogue
        with archive.open("GenerationSchemas.xml") as f:
            schemas = parse_schemas_xml(f.read())

        # Load wordlists on demand
        def load_wordlist(path):
            with archive.open(f"wordlists/{path}.lst") as f:
                return [line.strip() for line in f.read().decode("utf-8").splitlines() if line.strip()]

    return schemas, load_wordlist
```

### Verify Archive Structure

```python
import zipfile

def verify_archive(archive_path):
    """Verify release archive structure."""
    with zipfile.ZipFile(archive_path) as archive:
        names = archive.namelist()

        # Check root files
        assert "GenerationSchemas.xml" in names, "Missing GenerationSchemas.xml"

        # Check wordlists directory
        wordlist_files = [n for n in names if n.startswith("wordlists/") and n.endswith(".lst")]
        assert len(wordlist_files) > 0, "No wordlist files found"

        # Check no wrapper directory
        top_level = set(n.split("/")[0] for n in names)
        assert top_level == {"wordlists", "GenerationSchemas.xml"}, f"Unexpected top-level: {top_level}"

        return {
            "total_files": len(names),
            "wordlist_files": len(wordlist_files),
            "valid": True,
        }
```

## CI/CD Integration

### GitHub Actions (Consumer)

Consumers can automate wordlist updates:

```yaml
name: Update Wordlists
on:
  schedule:
    - cron: '0 0 * * 1'  # Weekly
  workflow_dispatch:

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Download latest wordlists
        run: |
          curl -L -o wordlists.zip \
            https://github.com/hmlendea/universal-name-generator-wordlists/releases/latest/download/universal-name-generator-wordlists.zip
          unzip -o wordlists.zip -d data/
      - name: Commit changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/
          git diff --staged --quiet || git commit -m "Update wordlists"
          git push
```

### Release Monitoring

```python
import requests
import hashlib

class ReleaseMonitor:
    """Monitor for new wordlist releases."""

    API_URL = "https://api.github.com/repos/hmlendea/universal-name-generator-wordlists/releases/latest"

    def __init__(self, known_version=None):
        self.known_version = known_version

    def check(self):
        response = requests.get(self.API_URL, timeout=10)
        response.raise_for_status()
        release = response.json()
        version = release["tag_name"].lstrip("v")

        if version != self.known_version:
            return {
                "update_available": True,
                "version": version,
                "url": release["html_url"],
                "assets": [a["browser_download_url"] for a in release["assets"]],
            }
        return {"update_available": False}
```

## Downstream Consumer Integration

### Python Package

```python
# setup.py or pyproject.toml dependency
# Download wordlists at install time or first run

from pathlib import Path
import urllib.request

def ensure_wordlists(base_dir="data"):
    """Download wordlists if not present."""
    base = Path(base_dir)
    if (base / "GenerationSchemas.xml").exists():
        return base

    base.mkdir(parents=True, exist_ok=True)
    url = "https://github.com/hmlendea/universal-name-generator-wordlists/releases/latest/download/universal-name-generator-wordlists.zip"
    archive_path = base / "wordlists.zip"
    urllib.request.urlretrieve(url, archive_path)

    import zipfile
    with zipfile.ZipFile(archive_path) as archive:
        archive.extractall(base)
    archive_path.unlink()
    return base
```

### JavaScript/Node.js

```javascript
const https = require('https');
const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

async function downloadWordlists(targetDir = './data') {
  const url = 'https://github.com/hmlendea/universal-name-generator-wordlists/releases/latest/download/universal-name-generator-wordlists.zip';
  const archivePath = path.join(targetDir, 'wordlists.zip');

  await new Promise((resolve, reject) => {
    https.get(url, (response) => {
      if (response.statusCode === 302) {
        // Follow redirect
        return downloadWordlists(targetDir); // Simplified
      }
      const file = fs.createWriteStream(archivePath);
      response.pipe(file);
      file.on('finish', () => { file.close(); resolve(); });
    }).on('error', reject);
  });

  execSync(`unzip -o ${archivePath} -d ${targetDir}`);
  fs.unlinkSync(archivePath);
}
```

### Docker Integration

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Download wordlists at build time
RUN apt-get update && apt-get install -y unzip curl && rm -rf /var/lib/apt/lists/*
RUN curl -L -o wordlists.zip \
      https://github.com/hmlendea/universal-name-generator-wordlists/releases/latest/download/universal-name-generator-wordlists.zip \
    && unzip wordlists.zip \
    && rm wordlists.zip

COPY . .
CMD ["python", "generate.py"]
```

## Integration Checklist

### For New Consumers

- [ ] Choose integration method (release download, git clone, API)
- [ ] Implement schema catalogue parser (XML)
- [ ] Implement schema string parser
- [ ] Implement wordlist loader
- [ ] Implement filter loader (if needed)
- [ ] Implement generation algorithms
- [ ] Implement error handling
- [ ] Implement update checking (optional)
- [ ] Comply with GPL v3.0
- [ ] Test with multiple schemas

### For Release Automation

- [ ] Monitor releases (API or webhook)
- [ ] Download and verify archive
- [ ] Extract to known location
- [ ] Invalidate caches
- [ ] Notify application of update
- [ ] Handle rollback on failure

## Related Documentation

- [Build and Deployment](build-and-deployment.md) -- Release process
- [Data Model](data-model.md) -- Data formats
- [API Usage Examples](api-usage-examples.md) -- Consumer code examples
- [Quick Start](quick-start.md) -- Minimal integration
- [Security](security.md) -- Supply-chain integrity
