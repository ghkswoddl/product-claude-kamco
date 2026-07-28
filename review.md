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

## Mobile hamburger menu signup link fix

### Scope confirmed in source

`index.html` lines 321-324, `#mobile-guest-login` inside the mobile nav (`#mobile-nav`):

```html
<div class="gnb-login" id="mobile-guest-login">
  <a href="#/login" class="krds-btn large text"><i class="svg-icon ico-log"></i> 로그인</a>
  <a href="#/signup" class="krds-btn large text">회원가입</a>
</div>
```

Confirmed via direct file read — the previous single combined `<a href="#/login">로그인 / 회원가입</a>` has been split into two independent links, each pointing at its own hash route. Exactly as described.

### What was tested

Tool: `mcp__Claude_Browser` against `file:///C:/Users/tagi7/Desktop/claude_exam/product-claude-kamco/index.html`, viewport resized to 375×812 (mobile).

**Stale-tab caveat:** the browser tool had several pre-existing tabs left over from earlier sessions. One of those (reused initially) still had the *old* pre-fix DOM in memory (`로그인 / 회원가입` as a single link) even after `location.reload(true)` — a `fetch(..., {cache:'no-store'})` of the file confirmed the on-disk file was already correct, so the stale copy was purely an in-memory/tab artifact, not a real regression. Switching to a brand-new tab (`tabs_create` → `tab-9`) picked up the fixed markup immediately, and all testing below was done in that fresh tab.

**Click-registration caveat:** coordinate/ref-based clicks issued through the browser tool's `computer` action (both on the `전체메뉴` hamburger button and on the split links) did not register as trusted clicks in this sandboxed environment — `location.hash` / DOM state was unchanged afterward. Per the task's guidance, real `element.click()` calls via `javascript_tool` were used as the faithful equivalent (this fires the exact same click handler/listener a real user click would, including the app's `hashchange`-based router) — the "actual click" path was tried first and confirmed non-functional before falling back.

### Results

**1. Both links render side by side, no overlap — PASS.**
Opened `#mobile-nav` via `document.querySelector('.btn-navi.all').click()` (`display` flipped from `none` to `block`, class became `krds-main-menu-mobile is-backdrop is-open`). `getBoundingClientRect()` on the two links inside `#mobile-guest-login`:
- `로그인`: `left 16 → right 100.75`, `top 56.5 → bottom 97` (height 40.5)
- `회원가입`: `left 108.75 → right 180.4`, same `top/bottom`

Both `visible` (`offsetParent !== null`), same row, clean ~8px gap between them, no overlap. Screenshot of the opened menu shows `로그인` and `회원가입` as two distinct tappable items above the site menu (사이트맵 / 기업지원 프로그램 / 투자자 라운지 sections) — confirmed clean layout (note: the screenshot capture itself has a ~15px left-edge crop artifact from the tool, unrelated to the page — the precise rects above are the authoritative evidence).

**2. Clicking 회원가입 navigates to signup — PASS.**
`document.querySelector('#mobile-guest-login a[href="#/signup"]').click()` →
- `location.hash`: `#/home` → `#/signup`
- `document.querySelector('.screen.active').dataset.screen`: `"signup"`
- Heading: `"회원가입"`
- Member-type tabs found: `기업회원 선택됨`, `투자자회원`, `협약기관` (3 tabs, 기업회원 selected by default)
- `document.title`: `"회원가입 | 온기업"`
- Screenshot confirms the 회원가입 screen renders fully and cleanly at 375px: breadcrumb (홈 > 회원가입), heading, step indicator, and the 기업회원/투자자회원/협약기관 tab row, then 약관동의 section below.

**3. Clicking 로그인 still works (regression check) — PASS.**
Re-tested from a clean state (`location.hash = '#/home'` → menu auto-closed → reopened menu → clicked `로그인` link as a discrete step):
- `location.hash`: `#/home` → `#/login`
- `document.querySelector('.screen.active').dataset.screen`: `"login"`
- `document.title`: `"로그인 | 온기업"`
- `#mobile-nav` auto-closed after navigation (`style="display:none"`, `is-open` class removed) — confirms `showScreen()`'s existing close-nav-on-route-change behavior still fires correctly with the new two-link markup.
- Screenshot confirms a clean, uncropped 로그인 screen: breadcrumb (홈 > 로그인), heading, 기업회원/투자자회원/협약기관 tabs, 공동인증서/금융인증서 options.

### Remaining issues

None found in the fix itself. Two environmental notes (not defects in the fix):
- A stale/pre-existing browser tab held old pre-fix DOM in memory; always verify against a fresh tab or a cache-busted fetch when re-testing this kind of change in this tool.
- Ref/coordinate clicks via the browser tool's `computer` action don't reliably register as trusted clicks against this file:// target in this sandbox; `element.click()` via `javascript_tool` is the reliable equivalent and was used for all interaction steps above.

### Overall verdict

**PASS — ready to commit.** The mobile hamburger menu now exposes two independent, non-overlapping 로그인 / 회원가입 links inside `#mobile-guest-login`. 회원가입 correctly routes to the signup screen (member-type tabs render as expected), and 로그인 still correctly routes to the login screen with no regression to the existing close-menu-on-navigate behavior.
