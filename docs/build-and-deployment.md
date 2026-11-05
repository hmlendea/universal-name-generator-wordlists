# Build and Deployment

## Build Process

This repository has **no build process**. It is an assets-only repository containing only plain-text data files and an XML schema catalogue. There is no compilation, transpilation, bundling, or artefact generation within the repository itself.

The only automated processing is **release packaging** performed by GitHub Actions.

## Release Packaging Pipeline

### Trigger

GitHub Release publication (`release.published` event).

### Workflow: `.github/workflows/release-asset.yml`

```yaml
name: Publish release asset
on:
  release:
    types: [published]
permissions:
  contents: write
jobs:
  build-and-upload:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Package release archive
        id: package
        shell: bash
        run: |
          version="${{ github.event.release.tag_name }}"
          version="${version#v}"
          archive_name="universal-name-generator-wordlists_${version}.zip"
          zip -r "$archive_name" wordlists GenerationSchemas.xml
          echo "archive_name=$archive_name" >> "$GITHUB_OUTPUT"
      - name: Upload release asset
        uses: softprops/action-gh-release@v2
        with:
          files: ${{ steps.package.outputs.archive_name }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Execution Steps

1. **Checkout** -- `actions/checkout@v4` fetches repository at the release tag
2. **Version extraction** -- Strip `v` prefix from tag (e.g., `v3.2.0` -> `3.2.0`)
3. **Packaging** -- `zip -r` creates archive containing:
   - `wordlists/` (entire directory tree, 388 `.lst` files)
   - `GenerationSchemas.xml` (schema catalogue)
4. **Upload** -- `softprops/action-gh-release@v2` attaches archive to GitHub Release

### Archive Structure

```
universal-name-generator-wordlists_X.Y.Z.zip
+-- wordlists/
|   +-- ... (complete directory tree)
+-- GenerationSchemas.xml
```

**Critical invariant**: Archive root must contain `wordlists/` and `GenerationSchemas.xml` directly (no wrapper directory).

## Deployment Model

### Distribution Channel

**GitHub Releases only**. No package managers (NuGet, npm, PyPI), container registries, or mirrors.

### Consumer Integration

Downstream consumers (primarily Universal Name Generator API):
1. Monitor GitHub Releases for new versions
2. Download `universal-name-generator-wordlists_X.Y.Z.zip` at build time
3. Extract to build environment
4. Reference `wordlists/` and `GenerationSchemas.xml` in generator build
5. Embed datasets into generator binary or load at runtime

### Version Pinning

Consumers pin to specific release versions. No automatic updates.

## Environments

| Environment | Purpose | Configuration |
|-------------|---------|---------------|
| GitHub Actions (ubuntu-latest) | Release packaging | Workflow-defined; no secrets beyond `GITHUB_TOKEN` |
| GitHub Releases | Asset hosting | GitHub-managed; public downloads |
| Consumer build environments | Dataset embedding | Consumer-defined; no repository involvement |

## Operational Procedures

### Creating a Release

**Prerequisites**:
- All desired changes merged to `master`
- Version number determined (Semantic Versioning)

**Steps**:
1. Create and push Git tag: `git tag vX.Y.Z && git push origin vX.Y.Z`
2. Create GitHub Release from tag (via GitHub UI or `gh release create`)
3. Verify `release-asset.yml` workflow completes successfully
4. Confirm release asset appears on GitHub Release page
5. Verify asset structure: `unzip -l universal-name-generator-wordlists_X.Y.Z.zip`

### Rollback

- **Release deletion**: Delete GitHub Release -> asset removed
- **Tag deletion**: `git push origin --delete vX.Y.Z` (requires admin permission)
- **Re-release**: Create new tag/release with corrected assets

No automated rollback; manual maintainer action required.

## Scaling Considerations

### Current Scale

- Wordlists: ~388 files, ~100+ MB
- Schemas: ~50 KB (328 entries)
- Release archive: ~100+ MB

### Limits

- GitHub Release asset limit: 2 GB (not approached)
- GitHub Actions runner storage: ~14 GB (sufficient)
- `zip` memory usage: proportional to archive size

### Growth Projections

- Adding new universes/languages increases archive size linearly
- No architectural changes needed until approaching 2 GB limit
- Potential future splitting by universe if needed (not currently planned)

## Verification

### Post-Release Checks

```bash
# Download and inspect archive
wget https://github.com/hmlendea/universal-name-generator-wordlists/releases/download/vX.Y.Z/universal-name-generator-wordlists_X.Y.Z.zip
unzip -l universal-name-generator-wordlists_X.Y.Z.zip

# Verify structure
# Should show:
# wordlists/
# GenerationSchemas.xml

# Verify content
unzip -p universal-name-generator-wordlists_X.Y.Z.zip GenerationSchemas.xml | xmllint --noout -
```

### CI Success Criteria

- Workflow completes without errors
- Asset uploaded to release
- Asset size > 0 bytes
- Asset contains expected files

## Troubleshooting

| Issue | Cause | Resolution |
|-------|-------|------------|
| Workflow fails at checkout | Tag doesn't exist | Verify tag pushed: `git tag -l vX.Y.Z` |
| Version extraction fails | Tag format not `vX.Y.Z` | Use SemVer tag format |
| Archive missing files | Paths changed in workflow | Verify `wordlists GenerationSchemas.xml` in zip command |
| Upload fails | `GITHUB_TOKEN` permissions | Ensure workflow has `contents: write` |
| Asset not downloadable | Release is draft/pre-release | Publish as full release |

## Related Documentation

- Architecture: `./architecture.md` (release packaging detail)
- Configuration: `./configuration.md` (version extraction, archive naming)
- Repository structure: `./repository-structure.md` (packaged paths)
