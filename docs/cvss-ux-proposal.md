# CVSS Score Display - UX Analysis and Proposal

## Executive Summary

This document analyzes best practices for displaying CVSS vulnerability scores and proposes UI improvements for Trustify. The key challenges are:
1. Handling multiple CVSS versions (2.0, 3.0, 3.1, 4.0)
2. Dealing with conflicting scores from different advisory sources
3. Choosing a "primary" score for list views while showing full context in details
4. Removing the flawed "average score" approach currently used

### Visual Mockups

Interactive PatternFly mockups are available in [`docs/mockups/`](mockups/index.html):
- [Vulnerability List](mockups/vulnerability-list.html) - Table view with variance indicators
- [Vulnerability Details](mockups/vulnerability-details.html) - Score comparison panel
- [SBOM Vulnerabilities](mockups/sbom-vulnerabilities.html) - Multi-source score display
- [Advisory List](mockups/advisory-list.html) - Aggregate severity and vulnerability gallery
- [Advisory Details](mockups/advisory-details.html) - Advisory's scores vs CVE Record comparison

Open these HTML files in a browser to see the proposed UI.

---

## Part 1: Industry Best Practices Analysis

### 1.1 Multiple CVSS Versions

The industry consensus is to display all available CVSS versions while prioritizing the most current:

| Source | Approach |
|--------|----------|
| **NVD** | Shows all available versions (v2, v3.x, v4.0) in separate sections. v4.0 is prioritized when available. |
| **Red Hat** | Displays side-by-side comparison of NVD vs Red Hat scores with visual diff indicators |
| **Snyk** | Uses proprietary scoring that considers CVSS but adds exploit maturity and reachability |
| **Dependency-Track** | Aggregates from multiple sources (NVD, OSS Index, GitHub, Snyk) with EPSS integration |

**Best Practice:** Use CVSS 4.0 as the primary display version when available. It provides more precise scoring with additional metrics (Threat, Environmental, Supplemental groups). Fall back to 3.1 when 4.0 is not available.

### 1.2 Conflicting Scores from Different Sources

This is a critical challenge. Research shows:

- **NVD scores** use a "worst case scenario" approach - if details are missing, they may rate as 10.0
- **Vendor scores** (e.g., Red Hat) are often more context-specific and account for how software is built/deployed
- **Neither is universally "correct"** - it depends on the user's context

**Example from Current Trustify Data:**
```
CVE-2023-1436:  CVE Record: 5.9  |  Red Hat: 7.5
CVE-2023-34455: CVE Record: 7.5  |  Red Hat: 5.9
CVE-2023-2976:  CVE Record: 5.5  |  Red Hat: 4.4
```

**Industry Approaches:**
1. **Red Hat Portal** - Shows both scores side-by-side with explanation of differences
2. **Snyk** - Uses independent analysis, acknowledges differences are "normal and expected"
3. **Tenable** - Adds VPR (Vulnerability Priority Rating) that considers exploit likelihood

### 1.3 Severity Color Coding (Industry Standard)

| Severity | Score Range | Color |
|----------|-------------|-------|
| Critical | 9.0 - 10.0 | Red (#C9190B) |
| High | 7.0 - 8.9 | Orange (#EC7A08) |
| Medium | 4.0 - 6.9 | Yellow/Gold (#F4A460) |
| Low | 0.1 - 3.9 | Blue (#0066CC) |
| None | 0.0 | Green (#3F8600) |

Trustify already follows this standard correctly.

---

## Part 2: Current Trustify UI State

### 2.1 What Works Well

1. **SBOM Scan/Analyze Page** - Already shows multiple scores per advisory with source labels
   - Format: `Critical (9.8): Red Hat | High (7.5): NVD`
   - Uses proper severity icons and colors

2. **Score Priority Logic** - Currently prioritizes CVSS 3.1 > 3.0 > 4.0 > 2.0 (needs update to prefer 4.0)

3. **Consistent Severity Visualization** - Same icons/colors across all pages

### 2.2 What Needs Improvement

| Issue | Current Behavior | Problem |
|-------|------------------|---------|
| **Average Scores** | Many pages show `average_score` | Averaging different sources is mathematically incorrect and misleading |
| **Single Score Display** | List pages show one score | Users can't see score variance or source |
| **No Version Indicator** | Score shown without CVSS version | User doesn't know if it's v3.1 or v4.0 |
| **No Source Attribution** | Vulnerability list doesn't show which advisory the score came from | Lacks transparency |
| **No Comparison View** | Details pages don't compare scores | Users can't evaluate discrepancies |

### 2.3 Pages Requiring Updates

| Page | Current | Proposed |
|------|---------|----------|
| Vulnerability List | Single average score | Primary score + indicator if scores vary |
| Vulnerability Details | Single score in header | Full score breakdown by source |
| SBOM Vulnerabilities Tab | Average score | Primary score + tooltip with alternatives |
| Package Vulnerabilities Tab | Average score | Primary score |
| Advisory Details | Single score | All scores from this advisory |

---

## Part 3: Proposed "Primary Score" Selection Algorithm

### 3.1 The Problem with Averaging

Currently: `(7.5 + 5.9) / 2 = 6.7` - This is meaningless. A vulnerability is not "6.7 severity" - it's either 7.5 (High) or 5.9 (Medium) depending on context.

### 3.2 Proposed Algorithm: "Most Relevant Score"

Instead of averaging, select a single "primary" score using this priority:

```
1. CVSS Version Priority (within same source):
   4.0 > 3.1 > 3.0 > 2.0

   Rationale: CVSS 4.0 is more precise with additional metric groups
   (Threat, Environmental, Supplemental) and is being rapidly adopted.

2. Source Priority (user-configurable, with sensible default):
   Default order:
   - Vendor advisory (Red Hat, etc.) - most context-specific
   - CVE Record (from CNA) - official vulnerability record
   - NVD enrichment - if no other source available

3. Multiple Vendors Tie-breaker: Latest advisory wins
   When multiple vendor advisories exist for the same vulnerability,
   use the one with the most recent modification/publication date.
   This ensures we use the most up-to-date analysis.

   Future: Allow admin configuration of preferred vendor priority.

4. Same-source Tie-breaker: Higher score wins (conservative approach)
```

### 3.3 Why This Order?

| Source | Reasoning |
|--------|-----------|
| **Vendor (e.g., Red Hat)** | Deeply analyzed for specific product context. Red Hat states their scoring is "more accurate than NVD" for their products. |
| **CVE Record** | Official record from the CVE Numbering Authority who discovered/reported the vulnerability |
| **NVD** | Good baseline but uses worst-case assumptions; may score 10.0 when details are missing |

### 3.4 Handling Multiple Vendor Advisories

**Scenario:** A vulnerability (e.g., CVE-2023-1234) has scores from:
- Red Hat advisory (modified: 2024-03-15) - Score: 7.5
- Ubuntu advisory (modified: 2024-02-10) - Score: 6.8
- SUSE advisory (modified: 2024-01-20) - Score: 7.2
- CVE Record - Score: 6.5

**Selection Logic:**
1. All three vendor advisories are preferred over CVE Record
2. Among vendors, Red Hat wins (most recent modification date: 2024-03-15)
3. Primary score displayed: **7.5 (High) - Red Hat**

**UI Indication:**
- Show variance indicator (⚠️) since scores differ
- Tooltip/expansion shows all scores sorted by date:
  ```
  Red Hat: 7.5 (Mar 15, 2024)
  SUSE: 7.2 (Jan 20, 2024)
  Ubuntu: 6.8 (Feb 10, 2024)
  CVE Record: 6.5
  ```

### 3.5 Future: Configurable Vendor Priority

Allow admins to configure preferred vendor order:
- Some organizations may trust NVD more (conservative, regulatory compliance)
- Others may prefer specific vendor scores (operational accuracy)
- Example config: `["Red Hat", "Ubuntu", "SUSE", "CVE Record", "NVD"]`

Until this is implemented, the "latest advisory" approach provides a sensible default.

### 3.6 Complete Algorithm Summary

```pseudocode
function selectPrimaryScore(vulnerability):
    scores = getAllScoresForVulnerability(vulnerability)

    // Step 1: Group by CVSS version, prefer highest version
    versionPriority = [4.0, 3.1, 3.0, 2.0]
    for version in versionPriority:
        scoresForVersion = scores.filter(s => s.cvssVersion == version)
        if scoresForVersion.length > 0:
            scores = scoresForVersion
            break

    // Step 2: Separate by source type
    vendorScores = scores.filter(s => s.source.type == "vendor_advisory")
    cveRecordScores = scores.filter(s => s.source.type == "cve_record")
    nvdScores = scores.filter(s => s.source.type == "nvd")

    // Step 3: Select source category (vendor preferred)
    if vendorScores.length > 0:
        candidateScores = vendorScores
    else if cveRecordScores.length > 0:
        candidateScores = cveRecordScores
    else:
        candidateScores = nvdScores

    // Step 4: Among same category, prefer latest modified
    candidateScores.sortByDescending(s => s.advisory.modifiedDate)

    // Step 5: If same date, prefer higher score (conservative)
    primaryScore = candidateScores[0]

    // Step 6: Calculate variance flag
    scoresVary = (scores.distinctByScore().length > 1)

    return { primaryScore, scoresVary, allScores: scores }
```

---

## Part 4: Detailed UI Proposals

### 4.1 List Pages: Vulnerability List, SBOM Vulnerabilities, Package Vulnerabilities

**Current:**
```
| ID           | Severity        |
|--------------|-----------------|
| CVE-2023-1436| Medium (6.7)    |  ← Average, misleading
```

**Proposed Option A: Primary Score with Variance Indicator**
```
| ID            | CVSS Score              |
|---------------|-------------------------|
| CVE-2023-1436 | 7.5 High ⚠️             |  ← Primary score
                  └── ⚠️ indicates scores differ between sources
```
Clicking ⚠️ or row shows tooltip: "Scores vary: Red Hat 7.5, CVE Record 5.9"

**Proposed Option B: Score Range Display**
```
| ID            | CVSS Score              |
|---------------|-------------------------|
| CVE-2023-1436 | 5.9 - 7.5 High          |  ← Shows range
```
This immediately communicates uncertainty.

**Proposed Option C: Primary Score with Source**
```
| ID            | CVSS Score       | Source   |
|---------------|------------------|----------|
| CVE-2023-1436 | 7.5 High         | Red Hat  |
```
Adds source column for transparency.

**Recommendation:** Option A or C. Option A is more compact; Option C is more transparent.

### 4.2 Detail Pages: Vulnerability Details

**Current:**
- Header shows single score badge
- No breakdown by source or version

**Proposed: Score Comparison Panel (Inspired by Red Hat Portal)**

```
┌─────────────────────────────────────────────────────────────────┐
│ CVE-2023-1436                                                   │
├─────────────────────────────────────────────────────────────────┤
│ CVSS Scores                                                     │
│                                                                 │
│ ┌─────────────────────┐    ┌─────────────────────┐              │
│ │ Red Hat Advisory    │    │ CVE Record          │              │
│ │ ━━━━━━━━━━━━━━━━━━━ │    │ ━━━━━━━━━━━━━━━━━━━ │              │
│ │ Score: 7.5 (High)   │    │ Score: 5.9 (Medium) │              │
│ │ Version: CVSS 3.1   │    │ Version: CVSS 3.1   │              │
│ │                     │    │                     │              │
│ │ [View Vector ▼]     │    │ [View Vector ▼]     │              │
│ └─────────────────────┘    └─────────────────────┘              │
│                                                                 │
│ 📊 Why the difference?                                          │
│ Vendor scores may differ from CVE records due to product-       │
│ specific analysis of how the vulnerability affects their        │
│ particular implementation and build configuration.              │
└─────────────────────────────────────────────────────────────────┘
```

**Vector Expansion (when clicked):**
```
┌─────────────────────────────────────────────────────────────────┐
│ CVSS 3.1 Vector: CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H   │
│                                                                 │
│ ┌────────────────┬─────────────┬─────────────┐                  │
│ │ Metric         │ Red Hat     │ CVE Record  │                  │
│ ├────────────────┼─────────────┼─────────────┤                  │
│ │ Attack Vector  │ Network     │ Network     │                  │
│ │ Attack Complexity│ Low       │ High        │  ← Difference!   │
│ │ Privileges Req │ None        │ None        │                  │
│ │ User Interaction│ None       │ None        │                  │
│ │ Scope          │ Unchanged   │ Unchanged   │                  │
│ │ Confidentiality│ None        │ None        │                  │
│ │ Integrity      │ None        │ None        │                  │
│ │ Availability   │ High        │ High        │                  │
│ └────────────────┴─────────────┴─────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 SBOM Scan/Analyze Page (Already Good, Minor Improvements)

**Current:** Already shows multiple scores with sources - this is the best page currently.

**Proposed Improvements:**
1. Add CVSS version indicator: `7.5 High (v3.1): Red Hat`
2. Add expandable vector comparison (like detail pages)
3. Consider showing score delta: `7.5 High ↑1.6` (compared to baseline)

### 4.4 Advisory Details Page

**Proposed:**
- Show the advisory's aggregate severity prominently
- List all vulnerabilities with THIS advisory's scores (not mixed with others)
- For each vulnerability, option to expand and see comparison with CVE record

---

## Part 5: CVSS Version Display Strategy

### 5.1 When Multiple CVSS Versions Exist

Example: A vulnerability has both CVSS 3.1 (7.5) and CVSS 4.0 (8.1) scores.

**Proposed Approach:**

**List Views:**
- Show CVSS 4.0 as primary when available (more precise scoring)
- Fall back to 3.1 if 4.0 not available
- Always show version badge: `8.1 High (v4.0)` or `7.5 High (v3.1)`

**Detail Views:**
- Tab or toggle to switch between versions
- Default to highest available version (4.0 > 3.1 > 3.0 > 2.0)
- Show all available versions for comparison

```
┌──────────────────────────────────────────────┐
│ CVSS Scores by Version                       │
│                                              │
│ [v4.0 ✓] [v3.1]                              │  ← v4.0 selected by default
│                                              │
│ Currently showing: CVSS 4.0                  │
│ ─────────────────────────────────────────    │
│ Red Hat: 8.1 (High)                          │
│ CVE Record: 7.2 (High)                       │
└──────────────────────────────────────────────┘
```

### 5.2 CVSS 4.0 Transition (Now Active)

CVSS 4.0 was released in late 2024 and adoption is accelerating:
1. **Default to v4.0** when available - it's more precise
2. Always show version indicator to maintain clarity
3. Future: Add user setting to override version preference if needed

---

## Part 6: Implementation Recommendations

### 6.1 Priority Order

| Priority | Item | Effort | Impact |
|----------|------|--------|--------|
| 1 | Remove average_score from display | Low | High - Fixes misleading data |
| 2 | Add primary score selection algorithm | Medium | High - Correct single-score display |
| 3 | Add variance indicator to lists | Low | Medium - Transparency |
| 4 | Add score comparison panel to details | Medium | High - User insight |
| 5 | Add CVSS version badges | Low | Medium - Future-proofing |
| 6 | Add vector comparison view | High | Medium - Power users |
| 7 | Add user-configurable source priority | High | Low - Niche use case |

### 6.2 API Changes Required

Per ADR-00004, the API is being updated to return:
- `VulnerabilityHead.baseScore` - single CVSSScore for lists
- `VulnerabilityDetails.advisories[].CVSSVectors[]` - full vectors for details
- `AdvisoryHead.aggregateSeverity` - for advisory-level display

**Additional Recommendation:**
- Add a `primaryScore` field that uses the selection algorithm
- Add a `scoresVary: boolean` flag for quick UI indication
- Include `cvssVersion` in all score responses

### 6.3 UX Research Questions

Before finalizing, consider user research on:
1. Do users prefer seeing score ranges or single primary scores in lists?
2. How important is vector-level comparison vs. just seeing the scores?
3. Should source priority be admin-configurable or user-configurable?

---

## Part 7: Examples Reference

### 7.1 Red Hat CVE Page (Best Reference)
https://access.redhat.com/security/cve/cve-2023-0044#cve-cvss

Key patterns:
- Side-by-side NVD vs Red Hat scores
- Visual highlighting of differences
- Explanation of why scores differ
- Vector string display

### 7.2 Trustify SBOM Scan (Current Best)
The existing analyze/scan page shows the right pattern:
- Multiple scores per vulnerability
- Source attribution
- Sorted by severity

This pattern should be extended to other pages.

---

## Appendix A: Data from Current Dataset

Vulnerabilities with differing scores in current ds3 dataset:

| CVE | CVE Record Score | Red Hat Score | Delta |
|-----|------------------|---------------|-------|
| CVE-2023-1436 | 5.9 (Medium) | 7.5 (High) | +1.6 |
| CVE-2023-24815 | 4.8 (Medium) | 5.3 (Medium) | +0.5 |
| CVE-2023-2976 | 5.5 (Medium) | 4.4 (Medium) | -1.1 |
| CVE-2023-34455 | 7.5 (High) | 5.9 (Medium) | -1.6 |

This demonstrates real-world score variance and the need for transparent display.

---

## Appendix B: Sources

- [NVD Vulnerability Metrics](https://nvd.nist.gov/vuln-metrics/cvss)
- [Red Hat: Security flaws and CVSS rescore process](https://www.redhat.com/en/blog/security-flaws-and-cvss-rescore-process-nvd)
- [FIRST CVSS v4.0 Specification](https://www.first.org/cvss/specification-document)
- [Snyk Severity Levels Documentation](https://docs.snyk.io/manage-risk/prioritize-issues-for-fixing/severity-levels)
- [OWASP Dependency-Track](https://dependencytrack.org/)
- [Vulnerability Management Dashboard Design (ACM)](https://dl.acm.org/doi/fullHtml/10.1145/3491418.3535176)
