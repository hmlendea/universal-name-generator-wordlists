# Documentation Maintenance

This document defines how the `docs/` directory is maintained, updated, and kept in sync with the repository.

## Maintenance Principles

- **Single source of truth**: Documentation reflects actual repository state, not aspirations
- **Change-driven updates**: Documentation updated in same PR as code/data changes
- **No duplication**: Root documents (ARCHITECTURE.md, README.md, etc.) are authoritative; `docs/` elaborates
- **Version alignment**: Documentation matches released version (updated on release)

## Update Triggers

| Trigger | Required Action | Files Typically Affected |
|---------|----------------|-------------------------|
| New wordlist added | Update structure, data model, architecture | `repository-structure.md`, `data-model.md`, `architecture.md` |
| New schema added | Update data model, architecture, invariants | `data-model.md`, `architecture.md`, `invariants.md` |
| Schema modified | Update architecture, invariants, change guide | `architecture.md`, `invariants.md`, `change-guide.md` |
| File moved/renamed | Update structure, architecture, change guide | `repository-structure.md`, `architecture.md`, `change-guide.md` |
| Release workflow changed | Update build-and-deployment, invariants | `build-and-deployment.md`, `invariants.md` |
| Root document changed | Sync corresponding `docs/` section | Varies |
| ROADMAP.md updated | Update architecture, testing, dependencies | `architecture.md`, `testing.md`, `dependencies.md` |

## Maintenance Workflow

### Per-PR Updates

1. **Identify affected docs** -- Use trigger table above
2. **Update in same PR** -- Documentation changes committed with code/data
3. **Validate links** -- Check all repository-relative links resolve
4. **Review for consistency** -- Cross-reference related documents

### Per-Release Updates

1. **Version references** -- Update any hardcoded version numbers
2. **Archive size/structure** -- Update `build-and-deployment.md` metrics
3. **Schema count** -- Update `data-model.md`, `repository-overview.md` counts
4. **File count** -- Update `repository-structure.md`, `repository-overview.md` counts

### Periodic Reviews (Quarterly)

- Verify all links resolve
- Check for outdated "planned" items now complete
- Align with ROADMAP.md status
- Remove superseded content

## Document-Specific Maintenance

### INDEX.md
- **Updated**: Every PR adding/removing documentation files
- **Source of truth**: File system scan of `docs/`
- **Automation candidate**: Could be generated

### repository-overview.md
- **Updated**: Major capability changes, audience shifts
- **Stable**: Purpose, scope, entry points rarely change

### repository-structure.md
- **Updated**: Every structural change (new universe, category, file)
- **Automation candidate**: Could be generated from `find` output

### data-model.md
- **Updated**: Schema format changes, new entity types
- **Stable**: Core entities rarely change

### architecture.md
- **Updated**: Algorithm changes, taxonomy changes, compatibility policy
- **Cross-reference**: Must stay aligned with root `ARCHITECTURE.md`

### configuration.md
- **Updated**: Workflow changes, new metadata files
- **Stable**: "No runtime config" is a design invariant

### build-and-deployment.md
- **Updated**: Workflow changes, release process changes
- **Metrics**: Archive size, file counts updated per release

### testing.md
- **Updated**: New validation methods, CI changes
- **Roadmap alignment**: Track planned enhancements

### dependencies.md
- **Updated**: Action version updates, new tooling
- **Stable**: No runtime dependencies is invariant

### design-decisions.md
- **Updated**: New ADRs added (append-only)
- **Immutable**: Existing decisions never modified

### invariants.md
- **Updated**: New invariants discovered, verification methods added
- **Stable**: Core invariants rarely change

### change-guide.md
- **Updated**: New procedures, lessons learned
- **Living document**: Evolves with repository practices

### logging.md
- **Updated**: If logging framework introduced (currently N/A)
- **Stable**: "No runtime logging" is invariant

### security.md
- **Updated**: New threat vectors, policy changes
- **Alignment**: Must match root `SECURITY.md`

### quick-start.md
- **Updated**: Consumer workflow changes
- **Stable**: Core consumption pattern stable

### state-and-persistence.md
- **Updated**: If persistence model changes (currently N/A)
- **Stable**: "No runtime state" is invariant

### error-handling.md
- **Updated**: New error categories, consumer guidance
- **Stable**: No runtime errors is invariant

### troubleshooting.md
- **Updated**: New failure modes, resolutions
- **Living document**: Grows with operational experience

### faq.md
- **Updated**: New recurring questions
- **Living document**: Grows with community interaction

### ambiguities-and-open-questions.md
- **Updated**: New ambiguities discovered, questions resolved
- **Living document**: Shrinks as questions answered

### api-usage-examples.md
- **Updated**: New consumer patterns, schema examples
- **Living document**: Grows with integration patterns

### Flows/
- **Updated**: New end-to-end flows documented
- **Stable**: Core flows rarely change

### Components/
- **Updated**: New component deep-dives
- **Stable**: Component architecture stable

### Behaviour/
- **Updated**: New interaction patterns
- **Stable**: Core behaviours stable

### Api-Reference/
- **Updated**: New schema entries, catalogue changes
- **Automation candidate**: Could be generated from XML

## Link Validation

### Internal Links
- All `./` and `../` links must resolve
- Check: `find docs -name "*.md" -exec grep -lE '\][(].*\.md' {} \;`

### External Links
- GitHub repository links (issues, PRs, releases)
- Check periodically for 404s

### Root Document Links
- `docs/` documents reference root documents
- Root documents reference `docs/` sections
- Both directions must be valid

## Style Conventions

### Formatting
- Headings: ATX style (`#`, `##`, `###`)
- Lists: Hyphen for unordered, numbers for ordered
- Code blocks: Fenced with language hint
- Tables: GitHub-flavored markdown

### Naming
- Files: kebab-case (`repository-overview.md`)
- Headings: Title Case (`Repository Overview`)
- Cross-references: Relative paths (`./architecture.md`)

### Content
- Evidence-based: Claims grounded in repository observation
- No speculation: Mark unknowns explicitly
- Current state: Document what is, not what should be

## Automation Opportunities

| Task | Current | Potential |
|------|---------|-----------|
| File count/structure | Manual | `find` + script |
| Schema count | Manual | XML query |
| Link validation | Manual | Markdown link checker |
| INDEX.md generation | Manual | Directory scan + template |
| API reference | Manual | XML to Markdown transform |

## Review Checklist

### Per PR
- [ ] Affected `docs/` files identified and updated
- [ ] All internal links resolve
- [ ] No duplication of root document content
- [ ] Style conventions followed
- [ ] Evidence-based claims

### Per Release
- [ ] Version references updated
- [ ] Metrics updated (counts, sizes)
- [ ] ROADMAP.md alignment checked
- [ ] INDEX.md regenerated/updated

### Quarterly
- [ ] All links validated
- [ ] Outdated "planned" items removed/updated
- [ ] Cross-document consistency verified
- [ ] Automation opportunities assessed

## Related Documentation

- Root documents: `../ARCHITECTURE.md`, `../README.md`, `../ROADMAP.md`, `../CONTRIBUTING.md`, `../SECURITY.md`, `../PRIVACY.md`, `../CODE_OF_CONDUCT.md`, `../LICENSE`
- Change procedures: `./change-guide.md`
- Invariants: `./invariants.md`
