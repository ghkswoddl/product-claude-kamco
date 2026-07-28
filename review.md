# Review: Mobile grid-collapse CSS fix (form-col-group / vdr-layout)

## Scope confirmed in source

- `index.html` line 51: `@media (max-width:767px){ .form-col-group{grid-template-columns:1fr !important;} }`
- `index.html` line 84: `@media (max-width:767px){ .vdr-layout{grid-template-columns:1fr !important;} }`
- `.form-col-group` is applied to the 2-column grids at line 1177 (`program-diagnosis` / 재무정보 입력) and line 1506 (`apply` Step 1 / 기업정보 입력).
- `.vdr-layout` class was added to the `box box-pad` grid at line 2024 (`vdr-viewer`, 26rem sidebar + 1fr content).

Both edits are present exactly as described.

## What was tested

Tool: `mcp__Claude_Browser` (Chromium preview) against the local file
`file:///C:/Users/tagi7/Desktop/claude_exam/product-claude-kamco/index.html`.

This prototype is a fake-SPA with a hash router (`window.addEventListener('hashchange', handleRoute)`), so screens were reached by setting `location.hash` directly (`#/program-diagnosis`, `#/apply/1`, `#/vdr-viewer`), which correctly triggers `handleRoute()` → `showScreen()` (and `initVdrViewer()` for the VDR screen). Direct browser-tool `navigate()` calls with a `#...` suffix did not reliably preserve the hash on this file:// target across the tool's tab handling, so `location.hash = '...'` via `javascript_tool` was used instead — this reproduces exactly what a real user would trigger by clicking `data-goto`/`href="#/..."` links, so it's a faithful test path.

Viewports tested: 375×812 (mobile) and 1280×900 (desktop), for all 3 screens.

## Mobile (375px) results

### 1. `program-diagnosis` — 재무정보 입력 (자가진단)
**PASS.** After navigating to `#/program-diagnosis`, `document.documentElement.scrollWidth` (413) equaled `window.innerWidth` (413) — no horizontal overflow. Screenshot confirms 최근 매출액, 부채비율, 자기자본, 기업 규모 fields are stacked in a single column, full width, nothing clipped.

### 2. `apply` Step 1 — 기업정보 입력
**PASS.** `scrollWidth` (375) equaled `innerWidth` (375) — no overflow. Screenshot confirms 상호, 사업자등록번호, 대표자명 fields (and the 담당자 연락처 field below the fold) stack in a single column; input text ((주)한빛정밀, 214-81-XXXXX, 박정호) is fully visible, not clipped.

### 3. `vdr-viewer` — 국유증권 가상데이터룸 보안 뷰어
**PASS.** `scrollWidth` (375) equaled `innerWidth` (375) — no overflow. Screenshot confirms the document tree nav (01~04 항목 목록, 첨부파일) stacks above the viewer/NDA content panel (full width), rather than being squeezed into a narrow side-by-side column. Text in the content panel renders as normal wrapped lines/paragraphs — no single-character-per-line fragmentation observed.

## Desktop (1280px) regression check

All 3 screens re-checked at 1280px:

- `program-diagnosis`: 2-column grid preserved (매출액/부채비율 side by side, 자기자본/기업 규모 side by side). `scrollWidth` 1265 vs `innerWidth` 1280 — no overflow.
- `apply` Step 1: 2-column grid preserved (상호/사업자등록번호 side by side, 대표자명/담당자 연락처 side by side). `scrollWidth` 1265 vs `innerWidth` 1280 — no overflow.
- `vdr-viewer`: sidebar (26rem) + content (1fr) side-by-side grid preserved, matching original design. `scrollWidth` 1265 vs `innerWidth` 1280 — no overflow.

**PASS** — the `max-width:767px` media queries are correctly scoped and do not affect desktop layout.

## Remaining issues

None found. Both fixes behave as intended at the mobile breakpoint and do not regress the desktop layout.

## Overall verdict

**Ready to commit.** Both CSS fixes (`.form-col-group` and `.vdr-layout` mobile overrides) work correctly on the three target screens at 375px, and desktop (1280px) layouts are unaffected.
