# Test Plan: Improve Loom Skills to Follow Best Practices

## Harness Requirements

No custom harness needs to be built. All tests use shell commands (`wc -l`, `grep`, `diff`, word count scripts) run against the markdown files in the worktree. The "harness" is bash verification — appropriate for a markdown-only refactoring task where the observation surface is file structure, line counts, content presence, and prose patterns.

**Harness: bash-verify** — Run shell commands against files in the worktree. Exposes: line counts, pattern matching, file existence checks, word counts, heading extraction. All tests below depend on this.

---

## Test Plan

### Test 1: Full refactoring preserves all loom skill content (scenario)

- **Name**: Loom skill content is relocated to reference files, not deleted
- **Type**: scenario
- **Harness**: bash-verify
- **Preconditions**: Tasks 1, 1b, and 2 are complete. `skills/loom/SKILL.md` has been rewritten and `references/server-patterns.md` and `references/advanced-patterns.md` have been created.
- **Actions**:
  1. Count total lines across all loom skill files: `wc -l skills/loom/SKILL.md skills/loom/references/*.md`
  2. Extract all `##` and `###` headings from the original SKILL.md (use git to retrieve the pre-refactoring version): `git show HEAD~N:skills/loom/SKILL.md | grep '^##'` (where N is the commit before Task 2)
  3. For each original heading, verify it appears either as a heading in the new SKILL.md OR as a heading in one of the reference files OR as a cross-reference (table row or inline `references/*.md#section-anchor`) in the new SKILL.md.
- **Expected outcome**:
  - Total line count across `SKILL.md` + all `references/*.md` is within 15% of the original total (~2804 lines for loom: 1632 SKILL + 479 cli-ref + 693 oauth-ref). New total should be ~2600-3100 lines. Source of truth: implementation plan verification criteria item 5 ("Total line count across all files per skill stays roughly the same").
  - Every original heading from the list (39 headings extracted above) is accounted for — present in SKILL.md, present in a reference file, or cross-referenced. Source of truth: implementation plan verification criteria item 4 ("No major sections silently dropped").
- **Interactions**: Git history (to retrieve original file).

---

### Test 2: Full refactoring preserves all loom-desktop skill content (scenario)

- **Name**: Loom-desktop skill content is relocated to reference files, not deleted
- **Type**: scenario
- **Harness**: bash-verify
- **Preconditions**: Tasks 3-7 are complete. `skills/loom-desktop/SKILL.md` has been rewritten and four new reference files exist.
- **Actions**:
  1. Count total lines across all loom-desktop skill files: `wc -l skills/loom-desktop/SKILL.md skills/loom-desktop/references/*.md`
  2. Extract all `##` and `###` headings from the original loom-desktop SKILL.md (from git history).
  3. For each original heading, verify it appears either in the new SKILL.md, in a reference file, or as a cross-reference in the new SKILL.md.
- **Expected outcome**:
  - Total line count across all files is within 15% of original total (~3162 lines: 2026 SKILL + 460 cli-ref + 280 electrobun + 396 rpc-schema). New total should be ~2700-3600 lines. Source of truth: implementation plan verification criteria item 5.
  - Every original heading (47 headings) is accounted for. Source of truth: implementation plan verification criteria item 4.
- **Interactions**: Git history.

---

### Test 3: Loom SKILL.md is under 500 lines (invariant)

- **Name**: Loom SKILL.md line count stays under the 500-line limit
- **Type**: invariant
- **Harness**: bash-verify
- **Preconditions**: Task 2 is complete.
- **Actions**:
  1. Run: `wc -l skills/loom/SKILL.md`
- **Expected outcome**: Line count is strictly less than 500. Source of truth: implementation plan verification criteria item 1 ("Both SKILL.md under 500 lines"), testing strategy item 1.
- **Interactions**: None.

---

### Test 4: Loom-desktop SKILL.md is under 500 lines (invariant)

- **Name**: Loom-desktop SKILL.md line count stays under the 500-line limit
- **Type**: invariant
- **Harness**: bash-verify
- **Preconditions**: Task 7 is complete.
- **Actions**:
  1. Run: `wc -l skills/loom-desktop/SKILL.md`
- **Expected outcome**: Line count is strictly less than 500. Source of truth: implementation plan verification criteria item 1, testing strategy item 1.
- **Interactions**: None.

---

### Test 5: Loom description uses "Use when..." format and is under 120 words (integration)

- **Name**: Loom skill metadata description follows trigger-optimized format
- **Type**: integration
- **Harness**: bash-verify
- **Preconditions**: Task 2 is complete.
- **Actions**:
  1. Extract the YAML description from `skills/loom/SKILL.md` (lines between `---` markers, the `description:` field).
  2. Verify first two words after `description:` content begins are "Use when" (case-sensitive).
  3. Count words in the description field.
  4. Verify the description contains at least two trigger phrases (strings like "Claude-powered web app", "streaming Claude output", "build an app that uses Claude", etc.).
  5. Verify the description contains a "NOT for" exclusion clause.
- **Expected outcome**:
  - Description starts with "Use when". Source of truth: implementation plan Task 2 step 1 ("Descriptions should start with 'Use when...'"), testing strategy item 2.
  - Word count is under 120. Source of truth: implementation plan verification criteria item 2, testing strategy item 9.
  - Contains trigger phrases (at least "Claude" and "web app" appear). Source of truth: implementation plan Task 2 revised description.
  - Contains "NOT for" or equivalent exclusion. Source of truth: implementation plan Task 2 revised description.
- **Interactions**: YAML parsing.

---

### Test 6: Loom-desktop description uses "Use when..." format and is under 120 words (integration)

- **Name**: Loom-desktop skill metadata description follows trigger-optimized format
- **Type**: integration
- **Harness**: bash-verify
- **Preconditions**: Task 7 is complete.
- **Actions**:
  1. Extract the YAML description from `skills/loom-desktop/SKILL.md`.
  2. Verify description starts with "Use when".
  3. Count words in the description.
  4. Verify contains trigger phrases (at least "desktop" and "Claude").
  5. Verify contains "NOT for" exclusion.
- **Expected outcome**:
  - Description starts with "Use when". Source of truth: implementation plan Task 7 step 1, testing strategy item 2.
  - Word count under 120. Source of truth: implementation plan verification criteria item 2, testing strategy item 9.
  - Contains trigger phrases and exclusion. Source of truth: implementation plan Task 7 revised description.
- **Interactions**: YAML parsing.

---

### Test 7: All reference file paths in loom SKILL.md resolve (integration)

- **Name**: Every reference path in loom SKILL.md points to an existing file
- **Type**: integration
- **Harness**: bash-verify
- **Preconditions**: Tasks 1, 1b, and 2 are complete.
- **Actions**:
  1. Extract all `references/*.md` paths from `skills/loom/SKILL.md`: `grep -oE 'references/[a-z-]+\.md' skills/loom/SKILL.md | sort -u`
  2. For each path, check file exists: `test -f "skills/loom/$path"`
  3. Check expected files are present: `cli-runtime-reference.md`, `oauth-reference.md`, `server-patterns.md`, `advanced-patterns.md`
- **Expected outcome**:
  - All extracted paths resolve to existing files (no MISSING). Source of truth: implementation plan verification criteria item 3.
  - At minimum the four expected files are referenced. Source of truth: implementation plan Task 2 step 3.
- **Interactions**: Filesystem.

---

### Test 8: All reference file paths in loom-desktop SKILL.md resolve (integration)

- **Name**: Every reference path in loom-desktop SKILL.md points to an existing file
- **Type**: integration
- **Harness**: bash-verify
- **Preconditions**: Tasks 3-7 are complete.
- **Actions**:
  1. Extract all `references/*.md` paths from `skills/loom-desktop/SKILL.md`: `grep -oE 'references/[a-z-]+\.md' skills/loom-desktop/SKILL.md | sort -u`
  2. For each path, check file exists: `test -f "skills/loom-desktop/$path"`
  3. Check expected files: `cli-runtime-reference.md`, `electrobun-setup.md`, `rpc-schema-reference.md`, `desktop-patterns.md`, `desktop-features.md`, `distribution.md`, `gotchas.md`
- **Expected outcome**:
  - All extracted paths resolve (no MISSING). Source of truth: implementation plan verification criteria item 3.
  - At minimum the seven expected files are referenced. Source of truth: implementation plan Task 7 step 3.
- **Interactions**: Filesystem.

---

### Test 9: No orphan reference files in loom skill (integration)

- **Name**: Every loom reference file is cross-referenced from SKILL.md
- **Type**: integration
- **Harness**: bash-verify
- **Preconditions**: Tasks 1, 1b, and 2 are complete.
- **Actions**:
  1. List all files in `skills/loom/references/`: `ls skills/loom/references/*.md`
  2. For each file, check its basename appears in `skills/loom/SKILL.md`: `grep -q "$(basename $f)" skills/loom/SKILL.md`
- **Expected outcome**: Every reference file is referenced from SKILL.md (no orphans). Source of truth: implementation plan verification criteria item 3, testing strategy item 3.
- **Interactions**: Filesystem.

---

### Test 10: No orphan reference files in loom-desktop skill (integration)

- **Name**: Every loom-desktop reference file is cross-referenced from SKILL.md
- **Type**: integration
- **Harness**: bash-verify
- **Preconditions**: Tasks 3-7 are complete.
- **Actions**:
  1. List all files in `skills/loom-desktop/references/`: `ls skills/loom-desktop/references/*.md`
  2. For each file, check its basename appears in `skills/loom-desktop/SKILL.md`.
- **Expected outcome**: Every reference file is referenced from SKILL.md (no orphans). Source of truth: implementation plan verification criteria item 3, testing strategy item 3.
- **Interactions**: Filesystem.

---

### Test 11: No MUST/MUST NOT in any loom skill file (invariant)

- **Name**: Loom skill files use imperative form, not rigid MUST language
- **Type**: invariant
- **Harness**: bash-verify
- **Preconditions**: All loom tasks (1, 1b, 2) are complete.
- **Actions**:
  1. Run: `grep -rn 'MUST' skills/loom/SKILL.md skills/loom/references/server-patterns.md skills/loom/references/advanced-patterns.md`
- **Expected outcome**: Zero matches. Source of truth: implementation plan verification criteria item 9, testing strategy item 6. The original SKILL.md has one known instance at line 223 ("you MUST pair") which the plan specifies fixing in server-patterns.md during extraction.
- **Interactions**: None.

---

### Test 12: No MUST/MUST NOT in any loom-desktop skill file (invariant)

- **Name**: Loom-desktop skill files use imperative form, not rigid MUST language
- **Type**: invariant
- **Harness**: bash-verify
- **Preconditions**: All loom-desktop tasks (3-7) are complete.
- **Actions**:
  1. Run: `grep -rn 'MUST' skills/loom-desktop/SKILL.md skills/loom-desktop/references/desktop-patterns.md skills/loom-desktop/references/desktop-features.md skills/loom-desktop/references/distribution.md skills/loom-desktop/references/gotchas.md`
- **Expected outcome**: Zero matches. Source of truth: implementation plan verification criteria item 9, testing strategy item 6. The original SKILL.md has zero MUST instances, but gotchas.md extraction must also fix "Must remove:" (line 1668) and "Must NEVER remove:" (line 1672) from the original.
- **Interactions**: None.

---

### Test 13: No stale "above"/"below" references in loom SKILL.md (invariant)

- **Name**: Loom SKILL.md has no stale internal references pointing to moved content
- **Type**: invariant
- **Harness**: bash-verify
- **Preconditions**: Task 2 is complete.
- **Actions**:
  1. Run: `grep -n '\babove\b\|\bbelow\b' skills/loom/SKILL.md`
  2. For each match, verify the referenced content is still in SKILL.md (not moved to a reference file).
- **Expected outcome**: Every "above"/"below" reference points to content that remains in SKILL.md. The implementation plan identifies these specific stale references to fix:
  - Line 80: "HTTP Hooks section below" (HTTP Hooks moved to advanced-patterns.md)
  - Line 122: "Handling File Uploads below" (moved to server-patterns.md)
  - Line 332: "shared utilities above" (moved to server-patterns.md — but this line itself moves, so it's fine in reference)
  - Line 363: "Server Setup block above" (same — moves with the code)
  - Line 962: "Every pattern above" (patterns moved to server-patterns.md)
  - Line 1560: "Server Setup block above" (stays in What to Generate — needs cross-ref update)
  - Line 1587: "inline patterns above" (stays in Checklist — needs cross-ref update)

  After refactoring, any remaining "above"/"below" in the new SKILL.md should reference only content still in that file. Source of truth: implementation plan verification criteria item 8, testing strategy item 7.
- **Interactions**: Requires human judgment on each match to determine if the reference is valid. The test script flags matches; the executor reviews them.

---

### Test 14: No stale "above"/"below" references in loom-desktop SKILL.md (invariant)

- **Name**: Loom-desktop SKILL.md has no stale internal references pointing to moved content
- **Type**: invariant
- **Harness**: bash-verify
- **Preconditions**: Task 7 is complete.
- **Actions**:
  1. Run: `grep -n '\babove\b\|\bbelow\b' skills/loom-desktop/SKILL.md`
  2. For each match, verify the referenced content is still in SKILL.md.
- **Expected outcome**: Every "above"/"below" reference points to content still in SKILL.md. The original has these references that will move:
  - Line 1391: "Pattern 4 above" (moves to desktop-patterns.md)
  - Line 1738: "deriveAndSendRPC function above" (moves to gotchas.md — needs cross-ref update to desktop-patterns.md)

  Source of truth: implementation plan verification criteria item 8, testing strategy item 7.
- **Interactions**: Requires judgment per match.

---

### Test 15: TOC present on reference files over 300 lines (invariant)

- **Name**: Large reference files include a table of contents
- **Type**: invariant
- **Harness**: bash-verify
- **Preconditions**: All tasks complete.
- **Actions**:
  1. For each new reference file, check line count: `wc -l <file>`
  2. For files over 300 lines, check that the first 30 lines contain either "## Table of Contents", "## Contents", or a markdown link list (lines matching `- [`): `head -30 <file> | grep -c '\- \['`
  3. Expected files needing TOC per plan: `server-patterns.md` (~650 lines), `advanced-patterns.md` (~380 lines), `desktop-patterns.md` (~750 lines), `gotchas.md` (~350 lines).
- **Expected outcome**:
  - Every reference file over 300 lines has a TOC in its first 30 lines. Source of truth: implementation plan verification criteria item 6, testing strategy item 8.
  - Files under 300 lines (desktop-features.md ~250, distribution.md ~250) are not required to have a TOC.
- **Interactions**: None.

---

### Test 16: Content balance — loom server-patterns.md is substantial (boundary)

- **Name**: Extracted server-patterns.md has the expected volume of content
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 1 is complete.
- **Actions**:
  1. Run: `wc -l skills/loom/references/server-patterns.md`
- **Expected outcome**: Line count is between 500 and 800. The implementation plan estimates ~600-700. If the file is under 400 lines, content was likely deleted rather than moved. If over 900, extraneous content was added. Source of truth: implementation plan Task 1 step 2, testing strategy item 5.
- **Interactions**: None.

---

### Test 17: Content balance — loom advanced-patterns.md is substantial (boundary)

- **Name**: Extracted advanced-patterns.md has the expected volume of content
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 1b is complete.
- **Actions**:
  1. Run: `wc -l skills/loom/references/advanced-patterns.md`
- **Expected outcome**: Line count is between 280 and 500. The implementation plan estimates ~350-420. Source of truth: implementation plan Task 1b step 2, testing strategy item 5.
- **Interactions**: None.

---

### Test 18: Content balance — loom-desktop desktop-patterns.md is substantial (boundary)

- **Name**: Extracted desktop-patterns.md has the expected volume of content
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 3 is complete.
- **Actions**:
  1. Run: `wc -l skills/loom-desktop/references/desktop-patterns.md`
- **Expected outcome**: Line count is between 550 and 900. The implementation plan estimates ~700-800. Source of truth: implementation plan Task 3 step 2, testing strategy item 5.
- **Interactions**: None.

---

### Test 19: Content balance — loom-desktop desktop-features.md is substantial (boundary)

- **Name**: Extracted desktop-features.md has the expected volume of content
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 4 is complete.
- **Actions**:
  1. Run: `wc -l skills/loom-desktop/references/desktop-features.md`
- **Expected outcome**: Line count is between 180 and 350. The implementation plan estimates ~250-300. Source of truth: implementation plan Task 4 step 2, testing strategy item 5.
- **Interactions**: None.

---

### Test 20: Content balance — loom-desktop distribution.md is substantial (boundary)

- **Name**: Extracted distribution.md has the expected volume of content
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 5 is complete.
- **Actions**:
  1. Run: `wc -l skills/loom-desktop/references/distribution.md`
- **Expected outcome**: Line count is between 150 and 300. The implementation plan estimates ~200-250. Source of truth: implementation plan Task 5 step 2, testing strategy item 5.
- **Interactions**: None.

---

### Test 21: Content balance — loom-desktop gotchas.md is substantial (boundary)

- **Name**: Extracted gotchas.md has the expected volume of content
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 6 is complete.
- **Actions**:
  1. Run: `wc -l skills/loom-desktop/references/gotchas.md`
- **Expected outcome**: Line count is between 260 and 430. The implementation plan estimates ~330-370. Source of truth: implementation plan Task 6 step 2, testing strategy item 5.
- **Interactions**: None.

---

### Test 22: Cross-references include "when to read" guidance (invariant)

- **Name**: Reference cross-links in SKILL.md include contextual guidance
- **Type**: invariant
- **Harness**: bash-verify
- **Preconditions**: Tasks 2 and 7 are complete.
- **Actions**:
  1. For each reference file mentioned in `skills/loom/SKILL.md`, verify at least one cross-reference includes guidance text (phrases like "Read this when", "See ... for", "when you need", or similar contextual framing near the reference path).
  2. Same for `skills/loom-desktop/SKILL.md`.
- **Expected outcome**: Each reference file path in both SKILL.md files appears at least once with contextual guidance (not bare path-only references). Source of truth: implementation plan "Remember" section ("Every reference cross-link includes guidance on WHEN to read it"), implementation plan verification criteria item 7.
- **Interactions**: Requires judgment on whether guidance is sufficient.

---

### Test 23: Gotchas section in loom-desktop SKILL.md lists all 16 gotchas (boundary)

- **Name**: Condensed gotchas section in loom-desktop SKILL.md accounts for all 16 original gotchas
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 7 is complete.
- **Actions**:
  1. Count the number of gotcha entries in the condensed SKILL.md gotchas section (look for numbered items or list entries).
  2. Count the number of gotcha headings in `references/gotchas.md` (look for `### N.` patterns).
- **Expected outcome**:
  - SKILL.md gotchas section contains references to 16 gotchas (numbered 1-16). Source of truth: implementation plan Task 7 item 12 ("16 entries"), original SKILL.md has 16 `### N.` headings from lines 1666-1979.
  - `references/gotchas.md` contains 16 gotcha headings. Source of truth: implementation plan Task 6 ("all 16 gotchas").
- **Interactions**: None.

---

### Test 24: Patterns at a Glance table in loom SKILL.md covers all patterns (boundary)

- **Name**: Loom SKILL.md pattern table includes all 9 communication patterns
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 2 is complete.
- **Actions**:
  1. Extract table rows from the "Patterns at a Glance" section in `skills/loom/SKILL.md`.
  2. Verify these 9 patterns appear: REST, SSE, WebSocket, Background Job, Parallel, Structured Extraction, Persistent Session, Action Markers, HTTP Hooks.
- **Expected outcome**: All 9 patterns from the original SKILL.md have a row in the summary table. Source of truth: implementation plan Task 2 item 6 (the table listing all 9), original SKILL.md headings at lines 358, 417, 527, 655, 722, 1170, 1220, 1303, 1335.
- **Interactions**: None.

---

### Test 25: Patterns table in loom-desktop SKILL.md covers all 4 patterns (boundary)

- **Name**: Loom-desktop SKILL.md pattern table includes all 4 interaction patterns
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 7 is complete.
- **Actions**:
  1. Extract table rows from the "Patterns at a Glance" section in `skills/loom-desktop/SKILL.md`.
  2. Verify these 4 patterns appear: Synchronous, Streaming, Conversational, Background.
- **Expected outcome**: All 4 patterns have a row. Source of truth: implementation plan Task 7 item 9, original SKILL.md headings at lines 427, 508, 886, 1092.
- **Interactions**: None.

---

### Test 26: Desktop features table covers all 5 features (boundary)

- **Name**: Loom-desktop SKILL.md features table includes all 5 desktop features
- **Type**: boundary
- **Harness**: bash-verify
- **Preconditions**: Task 7 is complete.
- **Actions**:
  1. Extract table rows from the "Desktop Features" section in `skills/loom-desktop/SKILL.md`.
  2. Verify these 5 features appear: File Drag-and-Drop, Native File Dialogs, System Tray, Native Menus, Local File Access (or File Access Config).
- **Expected outcome**: All 5 features have a row. Source of truth: implementation plan Task 7 item 10, original SKILL.md headings at lines 1270, 1338, 1389, 1399, 1468.
- **Interactions**: None.

---

## Coverage Summary

### Covered areas

| Area | Tests | Coverage |
|------|-------|----------|
| Line counts (under 500) | 3, 4 | Both SKILL.md files verified |
| Metadata descriptions | 5, 6 | Format, word count, triggers, exclusions |
| Reference integrity (paths resolve) | 7, 8 | All reference paths in both skills |
| No orphan files | 9, 10 | All reference dirs in both skills |
| Section heading preservation | 1, 2, 24, 25, 26 | Full heading diff + pattern/feature tables |
| Content balance (moved not deleted) | 1, 2, 16-21 | Total line counts + per-file ranges |
| No MUST/MUST NOT | 11, 12 | Both skills including all new reference files |
| No stale above/below | 13, 14 | Both SKILL.md files |
| TOC on large files | 15 | All reference files over 300 lines |
| Description word count | 5, 6 | Both skills |
| Cross-reference guidance | 22 | Both skills |
| Gotcha completeness | 23 | All 16 gotchas preserved |

### Explicitly excluded per agreed strategy

| Excluded | Risk |
|----------|------|
| Automated writing quality linting | Low — prose style is manually reviewed during extraction. MUST/MUST NOT check covers the key mechanical violation. |
| Live triggering tests | Medium — description changes could affect Claude's skill selection. Mitigated by following the "Use when..." format from best practices docs. |
| End-to-end app generation tests | Low — no functional code changes; this is a documentation refactoring. Skills would need to be tested in a live Claude Code session to verify they still produce working apps. |
| Cross-references with `#section-anchor` validation | Medium — anchor links (e.g., `server-patterns.md#pattern-sse-streaming`) could be wrong if the heading text doesn't match the anchor. Manual review during implementation is the mitigation. |
