# Host and Composition

This document describes how the repository's data assets are composed, packaged, and hosted for distribution. Since this is an assets-only repository with no runtime host, "host" refers to the distribution platform (GitHub) and "composition" refers to archive assembly.

## Distribution Host

### GitHub Repository

**Primary host**: `https://github.com/<owner>/universal-name-generator-wordlists`

**Capabilities**:
- Source code (data + schema) browsing
- Issue tracking for contributions
- Pull request workflow
- Release management
- GitHub Actions CI/CD

**Access patterns**:
| Audience | Access Method |
|----------|---------------|
| Contributors | Clone, PR, push (with permissions) |
| Consumers | Download release assets |
| Integrators | Clone or download release |
| Browsers | Web UI navigation |

### GitHub Releases

**Release host**: `https://github.com/<owner>/universal-name-generator-wordlists/releases`

**Asset hosting**: GitHub Release assets (CDN-backed)
**Asset naming**: `universal-name-generator-wordlists_X.Y.Z.zip`
**Retention**: Permanent (unless manually deleted)
**Bandwidth**: GitHub-provided (subject to fair use)

**Release metadata**:
- Tag: `vX.Y.Z` (SemVer)
- Title: `Universal Name Generator Wordlists vX.Y.Z`
- Notes: Auto-generated or manual changelog
- Assets: Single ZIP archive

## Composition Model

### Source Composition (Repository)

```
universal-name-generator-wordlists/
+-- wordlists/                    # 388 .lst files
|   +-- <universe>/               # 30+ universes
|   |   +-- <category>/           # persons, locations, etc.
|   |   |   +-- <file>.lst
|   |   +-- ...
|   +-- ...
+-- GenerationSchemas.xml         # 328 schema entries
+-- .github/workflows/
|   +-- release-asset.yml         # Packaging workflow
+-- *.md                          # Root documentation
+-- LICENSE                       # GPL v3.0
```

**Composition principles**:
- Flat schema catalogue (single XML file)
- Hierarchical wordlists (universe -> category -> file)
- No build-time transformation
- Source = distribution content

### Release Archive Composition

**Archive name**: `universal-name-generator-wordlists_X.Y.Z.zip`

**Internal structure**:
```
universal-name-generator-wordlists_X.Y.Z.zip
+-- wordlists/                    # Complete wordlists tree
|   +-- a-song-of-ice-and-fire/
|   +-- age-of-wonders/
|   +-- ... (all 30+ universes)
|   +-- yohoho-puzzle-pirates/
+-- GenerationSchemas.xml         # Complete schema catalogue
```

**Composition process** (`.github/workflows/release-asset.yml`):
```yaml
# Simplified workflow steps
- uses: actions/checkout@v4
- run: |
    VERSION=${GITHUB_REF_NAME#v}
    zip -r "universal-name-generator-wordlists_${VERSION}.zip" \
      wordlists/ GenerationSchemas.xml
- uses: softprops/action-gh-release@v2
  with:
    files: universal-name-generator-wordlists_${VERSION}.zip
```

**Composition invariants**:
- No wrapper directory (contents at archive root)
- Complete wordlists tree (all 388 files)
- Complete schema catalogue (all 328 entries)
- Deterministic ordering (zip -r preserves filesystem order)

### Consumer Composition

**Consumer receives**: Extracted archive with fixed layout

**Integration patterns**:

| Pattern | Description | Example |
|---------|-------------|---------|
| Embedded | Ship archive with application | Game installer includes wordlists |
| Downloaded | Fetch at runtime | Launcher downloads latest release |
| Bundled | Compile into binary | `go:embed` or C# embedded resource |
| Referenced | Point to local path | Config: `wordlists_path: ./data` |

**Consumer composition responsibilities**:
- Extract archive to known location
- Resolve `wordlists/` relative paths from schema
- Load `GenerationSchemas.xml` for schema catalogue
- Implement generation algorithms (randomiser, markov)
- Apply filter lists if present

## Version Composition

### Version Sources

| Component | Version Source |
|-----------|----------------|
| Release | Git tag `vX.Y.Z` |
| Archive | Derived from tag (strip `v`) |
| Schema IDs | Independent (stable, not versioned) |
| Wordlist files | Independent (append-only, not versioned) |

### Version Propagation

```
Git tag v3.2.0
    v
Workflow extracts "3.2.0"
    v
Archive named universal-name-generator-wordlists_3.2.0.zip
    v
Consumer sees version in filename
    v
Consumer may embed version in their app
```

### Compatibility Composition

**Schema ID stability**: IDs never change -> consumers can hardcode
**Path stability**: Wordlist paths in schemas stable -> consumers can cache
**Archive layout stability**: Root structure fixed -> consumers can script extraction

**Breaking change composition**:
- Major version (X.0.0) for:
  - Archive layout changes
  - Schema ID removals
  - Path restructuring
- Minor/patch for additive changes only

## Hosting Constraints

### GitHub Limits

| Limit | Value | Impact |
|-------|-------|--------|
| Repo size | ~100MB (current) | Well within 1GB soft limit |
| Release asset | 2GB max | Current ~100MB, room to grow |
| Release count | Unlimited | No practical limit |
| Bandwidth | Fair use | Suitable for current scale |

### Scaling Considerations

**Current scale**: ~388 files, ~100MB archive, 328 schemas
**Projected growth**: Linear with new universes/languages
**Mitigation**:
- Split archives by universe if >2GB (future)
- Git LFS for binary assets (not needed currently)
- CDN for high-volume consumers (self-hosted)

## Composition Validation

### Pre-Release Validation

```bash
# 1. Verify archive structure
unzip -l universal-name-generator-wordlists_3.2.0.zip | head -20

# 2. Verify completeness
unzip -l universal-name-generator-wordlists_3.2.0.zip | grep '\.lst$' | wc -l
# Should match: find wordlists -name "*.lst" | wc -l

# 3. Verify schema included
unzip -l universal-name-generator-wordlists_3.2.0.zip | grep GenerationSchemas.xml

# 4. Verify no wrapper directory
unzip -l universal-name-generator-wordlists_3.2.0.zip | awk '{print $4}' | grep -v '^$' | cut -d'/' -f1 | sort -u
# Should output: wordlists GenerationSchemas.xml
```

### Post-Release Validation

```bash
# Consumer-side verification
wget -q https://github.com/.../releases/download/v3.2.0/universal-name-generator-wordlists_3.2.0.zip
unzip -q universal-name-generator-wordlists_3.2.0.zip
test -d wordlists && test -f GenerationSchemas.xml && echo "OK"
```

## Related Documentation

- Build and deployment: `../build-and-deployment.md`
- Architecture: `../architecture.md` (release packaging)
- Repository structure: `../repository-structure.md`
- Invariants: `../invariants.md` (release invariants)
- Change guide: `../change-guide.md` (version bumping)
