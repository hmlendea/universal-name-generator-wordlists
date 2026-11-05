# Ambiguities and Open Questions

This document records ambiguities discovered during documentation, open questions about repository behaviour, and areas where the specification is silent or unclear.

## Resolved Ambiguities

| Ambiguity | Resolution | Evidence |
|-----------|------------|----------|
| "Calendar search" in spec | Interpreted as schema discovery/browsing flow | Repository has no calendar feature; closest analogue is schema catalogue search |
| "Detail mutation" in spec | Interpreted as generation with parameter adjustment | Repository has no CRUD; closest analogue is schema detail view with mutable params |
| "Controllers" in spec | Interpreted as schema catalogue entries | Repository has no HTTP controllers; schema catalogue is the "API" |
| "Presentation layer" | Interpreted as data packaging/formatting | No runtime UI; presentation = data formats and access patterns |
| "Browser state" | Interpreted as consumer-side state management | No runtime; state concerns apply to web consumers |
| "Logging framework" | Documented as N/A (no runtime) | Repository has no code; logging is consumer responsibility |

## Open Questions

### Data Quality

| Question | Impact | Status |
|----------|--------|--------|
| Are duplicate entries within a file intentional? | Data quality, generation diversity | Open -- no duplicate detection exists |
| Are cross-file duplicates intentional? | Data quality, storage efficiency | Open -- no cross-reference validation |
| Are entries sorted or insertion-ordered? | Markov training sequence quality | Open -- order matters for markov, unclear if intentional |
| Are there encoding edge cases (BOM, CRLF)? | Parser robustness | Open -- assumed UTF-8 LF, not verified per-file |
| Are offensive/profane entries present? | Consumer filtering needs | Open -- no content audit documented |

### Schema Catalogue

| Question | Impact | Status |
|----------|--------|--------|
| Are all 328 schemas actively used by consumers? | Dead schema cleanup | Open -- no usage telemetry |
| Are schema IDs documented anywhere besides XML? | Consumer discoverability | Open -- no separate ID registry |
| Is the reserved empty field in randomiser (`{randomiser, ,...}`) intentional? | Format clarity | Open -- appears to be placeholder |
| Are there schemas with no wordlist references? | Referential integrity | Open -- not systematically checked |
| Is `FilterlistPath` relative to `wordlists/` or repository root? | Path resolution | Open -- assumed `wordlists/`-relative, unverified |

### Distribution

| Question | Impact | Status |
|----------|--------|----------|
| Are release assets checksum-verified? | Integrity | Open -- no checksums in workflow |
| Is there a latest/stable release pointer? | Consumer update logic | Open -- consumers must query API |
| Are pre-releases used for testing? | Release process | Open -- no pre-release workflow documented |
| Is there a minimum supported consumer version? | Compatibility | Open -- no compatibility matrix |

### Governance

| Question | Impact | Status |
|----------|--------|----------|
| Who reviews PRs? | Contribution process | Open -- CONTRIBUTING.md may specify |
| What is the release cadence? | Consumer update planning | Open -- no schedule documented |
| Are there deprecation policies? | Breaking change management | Open -- no formal policy |
| Is there a security contact? | Vulnerability reporting | Open -- SECURITY.md may specify |

### Consumer Integration

| Question | Impact | Status |
|----------|--------|----------|
| Do consumers validate schema strings before parsing? | Error handling | Open -- consumer implementation detail |
| Do consumers cache wordlists across sessions? | Performance | Open -- consumer implementation detail |
| Do consumers handle missing filter files gracefully? | Robustness | Open -- consumer implementation detail |
| Is there a reference consumer implementation? | Integration testing | Open -- not in this repository |

## Specification Gaps

### Documentation Spec vs Repository Reality

| Spec Requirement | Repository Reality | Handling |
|------------------|-------------------|----------|
| API reference with per-endpoint docs | No HTTP API | Documented schema catalogue as "API" |
| Controller files | No controllers | Documented schema entries as "controllers" |
| Behaviour docs (browse/search, inspect/edit/delete) | No runtime behaviour | Documented contributor/consumer interaction patterns |
| Component deep-dives (presentation, host, services, browser state, integration) | No runtime components | Documented data packaging, distribution, consumer services |
| Flows (startup/rendering, calendar-search, detail-mutation) | No runtime flows | Documented consumer-side flows with closest analogues |
| Logging framework documentation | No runtime, no logging | Documented as N/A with consumer guidance |
| State and persistence | No runtime state | Documented consumer-side state patterns |

### Assumptions Made

| Assumption | Rationale | Risk |
|------------|-----------|------|
| Schema paths are `wordlists/`-relative | Consistent with observed paths | Low -- verify with consumer |
| Filter paths follow same convention | Consistent with schema paths | Low -- verify with consumer |
| Markov order is typically 4 | Observed in schema strings | Low -- varies per schema |
| Randomiser has empty reserved field | Observed in all randomiser schemas | Low -- format is fixed |
| Archive contains all 388 files | Workflow zips entire `wordlists/` | Low -- verify post-release |
| Consumers implement algorithms | No algorithm code in repo | None -- by design |

## Investigation Needed

### High Priority

1. **FilterlistPath resolution**: Verify whether paths are relative to `wordlists/` or repository root
   - Method: Check consumer source code or test with known filter path
   - Impact: Path resolution correctness

2. **Schema string edge cases**: Verify all 328 schemas parse correctly
   - Method: Run parser against all schemas, log failures
   - Impact: Consumer robustness

3. **Wordlist path existence**: Verify all schema paths reference existing files
   - Method: Automated check (see api-usage-examples.md Example 10)
   - Impact: Referential integrity

### Medium Priority

4. **Duplicate entry audit**: Count intra-file and cross-file duplicates
   - Method: `sort | uniq -d` per file and across files
   - Impact: Data quality assessment

5. **Encoding audit**: Verify all files are UTF-8 without BOM
   - Method: `file -i` on all `.lst` files
   - Impact: Parser robustness

6. **Entry count per schema**: Calculate total entries per schema
   - Method: Sum wordlist file line counts per schema paths
   - Impact: Consumer performance planning

### Low Priority

7. **Schema usage telemetry**: Determine which schemas consumers actually use
   - Method: Consumer analytics (out of scope for this repo)
   - Impact: Dead schema cleanup

8. **Release asset verification**: Verify archive structure matches invariant
   - Method: Download and inspect latest release
   - Impact: Distribution correctness

9. **Consumer compatibility matrix**: Document which consumer versions support which data versions
   - Method: Consumer release notes analysis
   - Impact: Version management

## How to Resolve

### For Repository Maintainers

1. **Add CI validation** (per ROADMAP.md):
   - XML well-formedness check
   - Schema path existence check
   - Duplicate detection
   - Encoding validation

2. **Document conventions**:
   - FilterlistPath resolution base
   - Schema string reserved field purpose
   - Entry ordering requirements

3. **Add metadata**:
   - Schema usage documentation
   - Deprecation policy
   - Release cadence

### For Consumers

1. **Verify assumptions**:
   - Test path resolution with known schemas
   - Validate all schemas parse
   - Check archive structure

2. **Report issues**:
   - File GitHub issues for broken references
   - Report parsing failures
   - Request missing metadata

### For Documentation Maintainers

1. **Update this document** as questions are resolved
2. **Move resolved items** to appropriate documentation
3. **Remove resolved items** from open questions
4. **Add new questions** as discovered

## Related Documentation

- Architecture: `./architecture.md` (validation gaps)
- Testing: `./testing.md` (coverage gaps)
- Invariants: `./invariants.md` (verification methods)
- Change guide: `./change-guide.md` (procedures)
- Roadmap: `../ROADMAP.md` (planned enhancements)
