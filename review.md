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

## content-layout grid overflow fix (home / apply step 3)

### Scope confirmed in source

All four parts of the fix are present exactly as described:

1. `index.html` line 30: `.content-layout{display:grid;grid-template-columns:minmax(0,1fr);max-width:1248px;margin:0 auto;padding:0 24px;box-sizing:border-box;}`
2. `index.html` line 31: `.content-layout.has-lnb{grid-template-columns:24rem minmax(0,1fr);gap:3.6rem;align-items:start;}`
3. `index.html` line 37, inside the `@media (max-width:1023px)` block: `.content-layout.has-lnb{grid-template-columns:minmax(0,1fr);}`
4. `index.html` lines 53-56, new `@media (max-width:767px)` block: `.hero .cta-row .krds-btn{white-space:normal;}` and `#btn-mydata-collect{white-space:normal;}`

`#btn-mydata-collect` (line 1570) is the Step 3 "법인 인증 및 공공 서류 한 번에 가져오기" button. `.hero .cta-row` appears 4 times (lines 370, 425, 491, 544) — guest home plus three logged-in dashboard variants (company/investor/partner), all sharing the same markup/button text for the diagnosis CTA.

### What was tested

Tool: `mcp__Claude_Browser` against `file:///C:/Users/tagi7/Desktop/claude_exam/product-claude-kamco/index.html`.

**Stale-tab caveat (recurring from prior review):** the browser tool had 8 leftover tabs from earlier sessions, some holding pre-fix DOM/state in memory. All were closed and a brand-new tab (`tab-12`) was created and force-navigated to the file before any testing, consistent with the lesson from the previous review round.

**Fake-SPA routing:** `location.hash = '#/home'`, `'#/apply/1'`, `'#/apply/3'`, `'#/program-diagnosis'` were set directly via `javascript_tool` to drive the hash router — clicking `#btn-next` on the apply screen turned out to be gated by `docsComplete()` validation logic (`updateNextState()` disables the button until required step-2 docs are attached), so rather than fighting that unrelated validation flow, step 3 was reached directly via `location.hash = '#/apply/3'` (this is the same code path the router uses regardless of how the hash changed, and was independently verified as correct by a stale tab that had organically reached `#/apply/3` in an earlier session).

**xlg font-scale mechanism discovered:** the "글자·화면 설정 → 크게" control (`[data-adjust-scale="xlg"]`) lives in the desktop-only `.header-utility` bar and is not reachable by a real tap at 375px (it's hidden off mobile layout, with no mobile-nav equivalent in this prototype). Its handler is wired by the vendor `assets/krds/js/krds.min.js` (not app code) and works by setting `document.body.style.zoom` (not root font-size or a class) — `scaleValue(e){this.scaleLevel=e,this.body.style.zoom=this.scaleLevel}`, with xlg resolving to `--krds-zoom-xlarge` (measured as zoom `1.3`). Calling `.click()` on the button element directly via `javascript_tool` fires this real handler regardless of visibility, which is a faithful test of the CSS regression even though a mobile user couldn't physically reach the control through this prototype's current responsive nav.

### Mobile (375×812) results

**1. Home screen — PASS.**
`scrollWidth` (375) === `clientWidth` (375) === `innerWidth` (375) — no horizontal overflow.
Hero CTA button `1분 만에 끝내는 기업 지원 자가진단 시작하기`: `white-space: normal`, rendered rect `width 303 × height 56` (two lines), `left 36 → right 339`, fully inside the 375px viewport, `offsetParent !== null` (visible/tappable). Screenshot confirms the button wraps cleanly onto two lines with no right-edge clipping and no horizontal scrollbar.

**2. Apply Step 3 at default font scale (regression check) — PASS.**
`scrollWidth` (375) === `clientWidth` (375). `#btn-mydata-collect` rect: `width 263.7 × height 40` (single line at default size), `white-space: normal` (rule applies but text fits on one line at default scale, so it's a no-op visually — expected).

**3. Apply Step 3 at xlg font scale — PASS, matches the fully-fixed case from the bug report.**
After firing the xlg click (`body.style.zoom` confirmed `1.3`): `scrollWidth` (375) === `clientWidth` (375) === `innerWidth` (375) — this matches the task's documented "375, i.e. fully fixed" measurement (vs. the pre-fix ~450 and grid-only-fix ~407 regressions noted in the task). `#btn-mydata-collect` rect: `width 246.2 × height 52` (wraps to two lines), `white-space: normal`. Screenshot (after `scrollIntoView`) confirms the button text `법인 인증 및 공공 서류 한 번에 가져오기` wraps onto two lines fully inside its card, no clipping, no horizontal scrollbar. Font scale was then reset via `[data-adjust-scale="md"]` (`body.style.zoom` back to `"1"`), confirmed via JS return value.

**4. Additional spot-check — `home-dashboard-company` hero (one of the 3 logged-in dashboard variants) — PASS.**
This dashboard is normally gated behind `loginAs(role)` / `isLoggedIn` state; reaching it via a full mock-login flow was out of scope, so it was surfaced directly by toggling the same `display` properties the app's own `renderHomeDashboard()` function toggles (`#home-content` → `none`, `#home-dashboard-company` → `flex`) — this exercises the identical shared `.hero .cta-row .krds-btn` markup/CSS as the guest home, just without simulating full auth state. Result: `scrollWidth` (375) === `clientWidth` (375), button rect `width 303 × height 56` (wraps to two lines), `white-space: normal`. Screenshot confirms clean two-line rendering, no overflow — same behavior as the guest-home hero. (The remaining two dashboard variants — investor, partner — share byte-for-byte identical `.hero .cta-row` markup per the source, so this one spot-check is considered representative of all four; not independently re-screenshotted.)

### Desktop (1280×900) regression check

All re-verified after resizing the same tab back up:

- **Home hero CTA:** `white-space: nowrap` (media query correctly scoped out at this width), screenshot confirms single-line rendering, matching original design — no awkward wrap introduced by the fix.
- **Apply Step 3 `#btn-mydata-collect`:** `white-space: nowrap`, rect `height 40` (single line), unaffected by the mobile-only rule.
- **`.content-layout.has-lnb` (tested on `program-diagnosis`):** `getComputedStyle(...).gridTemplateColumns` = `"240px 924px"` — sidebar (24rem = 240px at 10px root) and content (`minmax(0,1fr)` resolving to the remaining 924px) render side by side exactly as before; screenshot confirms the left LNB nav and the 재무정보 입력 form panel sit side by side, unchanged from the pre-fix design. `scrollWidth` (1265) vs `clientWidth` (1265) — no overflow, matching the 1265/1280 pattern seen in the prior review's desktop checks (24px page padding not counted in `clientWidth` measurement point).

**PASS** — `minmax(0,1fr)` and the mobile-only `white-space:normal` rule are correctly scoped; desktop layout and single-line button rendering are unaffected.

### Remaining issues

None found in the fix itself. Two notes, not defects:
- The xlg font-scale control is only reachable via the desktop header-utility bar in this prototype (no mobile-nav equivalent), so a real mobile user cannot currently trigger it directly at 375px — this is a pre-existing prototype/nav-completeness gap, unrelated to this CSS fix, and firing the click programmatically via `javascript_tool` is a faithful proxy for testing the CSS regression itself.
- `#btn-next` on the apply flow is gated by document-upload validation (`docsComplete()`), so step navigation was done via direct hash change (`#/apply/3`) rather than sequential button clicks — this is expected app behavior, not a bug.

### Overall verdict

**Ready to commit.** All 4 parts of the `.content-layout` / `.krds-btn` white-space fix are present in source and verified working: home hero CTA and the apply Step 3 mydata-collect button no longer overflow at 375px (confirmed both at default and at xlg/zoom-1.3 font scale, matching the task's documented "375 = fully fixed" target), and desktop (1280px) rendering — single-line buttons and the `has-lnb` two-column sidebar layout — is unchanged. A second `.hero .cta-row` instance (company dashboard) was spot-checked and behaves identically to the guest home.

## Mobile hamburger menu: sitemap removal + 마이메뉴 tab

### Scope confirmed in source

- `index.html` line 252, desktop `.header-utility` sitemap button (`<button ... data-target="sitemap-overlay">사이트맵</button>`) — **untouched**, still present outside `#mobile-nav`.
- `index.html` line 2384, footer sitemap link (`<button ... data-target="sitemap-overlay">사이트맵 <i class="svg-icon ico-angle right"></i></button>`) — **untouched**.
- `index.html` lines 2406-2419+, `#sitemap-overlay` modal itself and its `#sitemap-grid` population logic (lines 2782-2792) — **untouched**.
- `index.html` lines 314-329, `#mobile-nav` → `.gnb-header`: no `사이트맵`/`open-modal`/`data-target="sitemap-overlay"` element remains inside it; `.gnb-login` blocks are only `#mobile-guest-login` (로그인/회원가입 links) and `#mobile-mygov` (username + 로그아웃), confirming the old `#mobile-mygov-links` quick-link row is gone.
- `index.html` line 2772-2773: `gnbMobileTabs.innerHTML = IA.map(...) + '<li id="mobile-mymenu-tab" style="display:none;"><a href="#mGnb-anchor-mymenu" class="gnb-main-trigger">마이메뉴</a></li>'` — 6th `<li>` appended after the 5 IA-driven tabs.
- `index.html` line 2780: matching hidden panel `<div class="gnb-sub-list" id="mGnb-anchor-mymenu" style="display:none;"><h2 class="sub-title">마이메뉴</h2><ul id="mobile-mymenu-list"></ul></div>` appended after the 5 IA panels.
- `index.html` lines 3469-3477, `renderMyGovMenu(role)`: populates both `#mygov-menu-list` (desktop) and `#mobile-mymenu-list` (mobile) from the same `group.items` (looked up via `profile.mygovGroup` in `ROLE_PROFILES`, lines 3443-3449) — single shared data source, as intended.
- `index.html` lines 3479-3502, `loginAs(role)`: sets `#mobile-mymenu-tab` and `#mGnb-anchor-mymenu` `style.display = ''` on login.
- `index.html` lines 3504-3518, `logout()`: sets both back to `style.display = 'none'`.

All parts of the described change are present exactly as described.

### What was tested

Tool: `mcp__Claude_Browser` against `file:///C:/Users/tagi7/Desktop/claude_exam/product-claude-kamco/index.html`.

**Tab-cap / stale-tab environment note:** the browser tool session already had 9 tabs open (the cap), so a brand-new tab could not be created this round (`tabs_create` errored with "tab cap reached"). `navigate()` calls to the file:// URL on both the active tab and an inactive one also errored ("may be missing, unreadable, or the user declined access") despite the file existing on disk. Falling back to `location.reload()` via `javascript_tool` on the existing active tab (`tab-19`) worked, and a DOM check both before and immediately after the reload confirmed the tab already reflected the post-edit markup (`#mobile-mymenu-tab` and `#mGnb-anchor-mymenu` present, `#mobile-mygov-links` absent) — so the reload was a safety measure to guarantee fresh JS state (fresh `isLoggedIn=false`), not a fix for stale markup. All testing below was done in this reloaded tab.

**Click-registration:** per the guidance and prior reviews' findings, all interactions (hamburger open, login-role-card clicks, tab clicks, mymenu link clicks, logout, desktop MyGOV dropdown) were driven via `element.click()` calls through `javascript_tool`, which fires the real listener a trusted click would (including `hashchange`-router navigation), rather than the browser tool's coordinate/ref `computer` clicks.

**Async note:** immediately after setting `location.hash` or calling `loginAs()`/`logout()` in one `javascript_tool` call, `document.querySelector('.screen.active').dataset.screen` sometimes still reported the *previous* screen (one-tick lag before the `hashchange` handler ran). A follow-up read in a separate call always showed the correct, updated screen — this is a tool-round-trip-timing artifact, not an app bug (confirmed by re-reading the same expression a moment later and seeing it update).

### Results

**1. Guest state — PASS.**
Opened `#mobile-nav` via `document.querySelector('.btn-navi.all').click()` (confirmed open: `className` became `krds-main-menu-mobile is-backdrop is-open`, `style="display: block;"`).
- `header.querySelectorAll('[data-target="sitemap-overlay"], .open-modal')` inside `.gnb-header` → **0 matches**. A broader sweep of every `button`/`a` inside `#mobile-nav` whose text includes `사이트맵` → **0 matches**. No sitemap trigger anywhere in the mobile nav.
- `#gnb-mobile-tabs > li` → 6 `<li>` in the DOM (5 IA tabs + the hidden 마이메뉴 `<li>`), but only **5 visible**: `getComputedStyle(#mobile-mymenu-tab).display === 'none'` confirmed, and filtering the tab list by `display !== 'none'` yields exactly `기업지원 프로그램 / 투자자 라운지 / 국유증권 가상데이터룸 / 고객마당 / 센터소개`.
- `getComputedStyle(#mGnb-anchor-mymenu).display === 'none'` confirmed — no empty 마이메뉴 section renders while scrolling the panel list (since it's `display:none`, no content of it can appear regardless of scroll position).
- `document.getElementById('mobile-mygov-links')` → `null` (old quick-link row is gone).

**2. Login as company — PASS.**
Navigated to `#/login` (`location.hash = '#/login'`), clicked `document.querySelector('[data-role-login="company"]')`. Result: `isLoggedIn` state applied — `#mygov-username` / `#mobile-mygov-username` both show `이수민님`, hash ended at `#/home`, `.screen.active.dataset.screen === 'home'`.

Reopened the hamburger menu (`.btn-navi.all.click()`). `#gnb-mobile-tabs > li` now shows the 6th tab **visible** (`display: 'list-item'`, `id: 'mobile-mymenu-tab'`), positioned last, after 센터소개: `[기업지원 프로그램, 투자자 라운지, 국유증권 가상데이터룸, 고객마당, 센터소개, 마이메뉴]`. `#mGnb-anchor-mymenu` computed display is `'block'`.

`#mobile-mymenu-list` innerHTML:
```html
<li><a href="#/mypage-status" class="gnb-sub-trigger">신청 및 진행 현황</a></li>
<li><a href="#/mypage-docs" class="gnb-sub-trigger">제출 서류/보완 관리</a></li>
<li><a href="#/mypage-account" class="gnb-sub-trigger">회원정보 및 권한 관리</a></li>
```
Exactly the 3 expected links with the expected `#/mypage-...` hrefs.

Clicked `#mobile-mymenu-tab .gnb-main-trigger` (the tab itself): the target panel (`#mGnb-anchor-mymenu`, `getBoundingClientRect().top === 562` at that scroll position) was brought into the 812px-tall viewport — same scroll-into-view behavior as the original tabs (see check 6). Then clicked the `#/mypage-status` link directly: `location.hash` became `#/mypage-status`, and on a follow-up read `.screen.active.dataset.screen === 'mypage-status'` (document title updated to `"신청 및 진행 현황 | 온기업"`), and `#mobile-nav` auto-closed (`style="display: none;"`) — confirming the existing close-nav-on-navigate behavior still fires for the new tab's links too.

**3. Logout — PASS.**
Clicked `#mobile-logout-btn` directly (`hash` → `#/home`). Reopened the hamburger menu: `#gnb-mobile-tabs > li` filtered by visible (`display !== 'none'`) → count **5** again (마이메뉴 gone from the visible list). `getComputedStyle(#mobile-mymenu-tab).display === 'none'` and `getComputedStyle(#mGnb-anchor-mymenu).display === 'none'` both confirmed. `#mobile-guest-login` back to `display: flex`, `#mobile-mygov` back to `display: none`. Full round-trip (guest → company login → logout) returns to the exact guest-state markup.

**4. Admin role (via 캠코 직원 로그인 / staff-login screen) — PASS.**
Navigated to `#/staff-login`, filled `#staff-id`/`#staff-pw` with dummy values, clicked `#staff-login-btn` (this calls `loginAs('staff')`, which per `ROLE_PROFILES` has `name:'김민준'`, `mygovGroup:'캠코 관리자'` — this is the role the task's "admin" check refers to; there's a separate top-tier `'admin'`/최고 관리자 role behind an OTP screen with the identical `mygovGroup`, so both resolve to the same menu). Result: `hash → '#/admin-dashboard'`, `#mobile-mygov-username` shows `김민준님`.
Reopened hamburger menu: `#mobile-mymenu-tab` display `'list-item'` (visible), `#mobile-mymenu-list` innerHTML:
```html
<li><a href="#/admin-dashboard" class="gnb-sub-trigger">관리자 대시보드</a></li>
<li><a href="#/admin-applications" class="gnb-sub-trigger">신청 접수 목록</a></li>
```
Exactly 관리자 대시보드 / 신청 접수 목록, as expected, replacing the 3-item 마이페이지 set. Screenshot (375×812, admin logged in, menu open) visually confirms: header shows `김민준님 ⏎ 로그아웃` (no sitemap button), tab list ends in a highlighted `마이메뉴` tab, and scrolling to its section shows a `마이메뉴` heading with `관리자 대시보드` / `신청 접수 목록` links.

**5. Desktop MyGOV dropdown regression (1280×900, still logged in as staff/admin from check 4) — PASS.**
Resized to 1280×900. `#mygov-menu-list` innerHTML:
```html
<li><a href="#/admin-dashboard" class="item-link">관리자 대시보드</a></li>
<li><a href="#/admin-applications" class="item-link">신청 접수 목록</a></li>
```
Identical items/hrefs to the mobile 마이메뉴 list (both are populated by the same `renderMyGovMenu(role)` call from the same `group.items`, as expected — this data source was not supposed to change). `#mygov-username` shows `김민준님`. Clicked the drop-toggle button (`#nav-mygov-wrap .drop-btn`): `.drop-menu` computed display became `'block'` (dropdown opens). Clicked the `#/admin-applications` link inside it: `location.hash` → `#/admin-applications`, and on follow-up read `.screen.active.dataset.screen === 'admin-applications'` (title `"신청 접수 목록 | 온기업"`) — the desktop MyGOV dropdown is fully functional and unchanged.

**6. Tab-click scroll regression (back at 375×812) — PASS.**
Reopened the hamburger menu, located the `고객마당` `<li>` among the now-6-item `#gnb-mobile-tabs` list, and clicked its `.gnb-main-trigger` (`href="#mGnb-anchor3"`). The resolved target panel's `<h2 class="sub-title">` text was `"고객마당"` (correct target, unaffected by the new 6th tab's presence), and its `getBoundingClientRect()` (`top: -36.5`, `height: 398.5`) showed it scrolled to align near the top of the viewport (`panelInViewport: true`) — the same scroll-to-anchor behavior observed for the new 마이메뉴 tab in check 2. The vendor's `krds_mainMenuMobile` click-binding is unaffected by adding a 6th static `<li>` at page load.

### Remaining issues

None found in the fix itself. One pre-existing-behavior note, not a defect introduced by this change: `ROLE_PROFILES.investor.mygovGroup` is `'투자자 라운지'` (not `'마이페이지'`), so logging in as **investor** specifically would show `매칭대상 기업정보 조회 / 투자제안 및 의향서 제출 / 매칭 관리` in both the mobile 마이메뉴 and the desktop MyGOV dropdown, not the `신청 및 진행 현황 / 제출 서류·보완 관리 / 회원정보 및 권한 관리` set the task description generalized across company/investor/partner. This is existing `ROLE_PROFILES`/`renderMyGovMenu` mapping behavior that predates this change (untouched by it, and correctly mirrored between mobile and desktop) — company and partner roles (both `mygovGroup:'마이페이지'`) do show the exact 3 links described, which is what was verified in check 2. Flagging only so the discrepancy in the task's own wording doesn't get mistaken for a bug in this diff.

Two environmental notes (not defects): the browser tool's tab cap was already reached this round (9/9 tabs) and fresh `navigate()` calls to the file:// URL errored, so testing reused an existing tab plus `location.reload()` rather than a brand-new tab as in prior reviews — verified via DOM inspection immediately after reload that this did not reintroduce the stale-DOM risk noted in earlier reviews. Coordinate/ref clicks were not attempted this round (per the task's guidance, `element.click()` via `javascript_tool` was used directly for every interaction).

### Overall verdict

**PASS — ready to commit.** The mobile hamburger nav's 사이트맵 button and the old `#mobile-mygov-links` row are both fully removed from `#mobile-nav`, with the desktop header sitemap button, footer sitemap link, and `#sitemap-overlay` modal all untouched. The new 마이메뉴 tab/panel is hidden for guests, correctly appears (positioned after 센터소개) and populates with the right role-based links on login for company (마이페이지 group) and admin/staff (캠코 관리자 group), correctly disappears on logout, navigates correctly on link click (closing the mobile nav as expected), and does not break the existing tab-click scroll behavior for the 5 original tabs. The desktop MyGOV dropdown regression check also passes — same shared data source, unaffected by this change.

## 회원가입 PRD 반영 (6단계 재설계)

### Scope confirmed in source

`index.html` lines 807-1342 (`data-screen="signup"`) implement the redesigned 6-step flow (0 약관동의 → 1 본인/기업인증 → 2 행정정보자동조회 → 3 부가정보입력 → 4 최종확인 → 5 가입완료) across all 3 member-type tabs/prefixes: `su-co` (기업회원, lines 824-1000), `su-inv` (투자자회원, lines 1002-1151), `su-pt` (협약기관, lines 1153-1338). The controlling JS is at lines 3790-4160 (`SIGNUP_PREFIXES`, `signupStep`, `runBizStatusCheck`, `runPersonAuth`, `runAdminDataFetch`, `runAccountVerify`, `renderSignupFinalRecap`, the `.su-btn-next` step-transition handler, `resetSignupSubForm`, `switchSignupTab`). The submit-confirm modal (`#signup-submit-confirm-modal`, lines 2884-2898) and the tab-switch-confirm modal (`#signup-tab-switch-modal`, lines 2858-2872) are also part of this feature.

### What was tested

Tool: `mcp__Claude_Browser` against `file:///C:/Users/tagi7/Desktop/claude_exam/product-claude-kamco/index.html`, driving every interaction via `element.click()` / direct property assignment through `javascript_tool` (per this sandbox's documented click-registration caveat — coordinate/ref `computer` clicks don't reliably register as trusted here). `location.hash = '#/signup'` was used to enter the screen (fake-SPA hash router).

**Session interruption / redo:** this review's browser session was interrupted partway through (after fully completing 기업회원 and 투자자회원 manually) and the preview pane was lost entirely (`tabs_context` returned "No preview is open"). Per instruction, the entire checklist was **restarted from scratch in a brand-new preview session** rather than trusting the partial pre-interruption results, so everything reported below comes from one continuous, fresh run.

**Stale-tab / stale-reload artifact encountered and resolved:** while doing the mobile (375px) pass, a `location.reload(true)` on the already-used tab did **not** actually reset the page's in-memory JS/DOM state (the tab-switch state from a prior step "leaked" through the reload — `panel-signup-pt` was still the active panel even though the markup/URL looked fresh), which briefly looked like a rendering bug (a status badge with `getClientRects().length === 0`). Re-verified in a genuine brand-new tab (`tabs_create` → fresh `navigate`) and confirmed the badge renders correctly (309×19.5px, fully in-viewport) — this was purely the documented sandbox reload artifact, not an app defect. All results below for the mobile pass come from the clean fresh-tab run.

Viewports tested: 1280×900 (desktop, full 3-tab walkthrough) and 375×812 (mobile, overflow/rendering spot-check through 기업회원's steps 0-4). No console errors (`read_console_messages`) were observed at any point in either tab.

### Results by tab

#### 기업회원 (su-co)

| # | Checklist item | Result |
|---|---|---|
| 1 | 약관동의: 다음 disabled until all 4 required checked; one-missing keeps it disabled; 모두 동의합니다 toggles all 4 required + 2 optional; re-unchecking one required re-disables | **PASS** |
| 2 | 사업자 상태 확인하기 (normal `123-45-67890`) → "checking" badge → after ~1.2s `ok`/"계속사업자로 확인되었습니다"; 본인인증하기 → "checking" → after ~0.8s `ok`/"본인인증 완료"; 다음 disabled with only one of the two done, enabled only after both | **PASS** |
| 2b | Cutoff scenario `999-99-99999` | **FAIL — see Bug #1 below.** Badge shows `ok`/success text, not `fail`, and 다음 stays eligible to enable (i.e. the cutoff never triggers) |
| 3 | 행정정보자동조회: loading → (~1s) result card; `su-co-admin-rep-name` shows the step-1 name (홍길동); 다음 disabled during loading, enabled after | **PASS** |
| 4 | 부가정보입력: empty-required-fields → toast "필수 항목을 모두 입력하세요", no advance; pw≠pw2 → toast "비밀번호가 일치하지 않습니다", no advance; 예금주 확인 (계좌 `110123456789`) → after ~0.8s shows "홍길동 (확인됨)" with class `ok`; without verifying, 다음 blocked with toast "정산계좌 예금주 확인을 완료하세요"; after verifying, 다음 advances | **PASS** |
| 5 | 최종확인: recap fields (`su-co-name`, `su-co-brn`, `su-co-phone`, `su-co-id`, `su-co-account-no`) all show actual entered values, not "-"; button reads "제출하기"; clicking opens confirm modal ("가입 신청을 제출하시겠습니까?"); 취소 closes modal, stays on step 4 (최종확인); re-open + 제출하기 advances to 완료, nav hidden | **PASS** |
| 6 | 완료: "기업회원 가입이 완료되었습니다" + 로그인하러 가기 button present | **PASS** |
| 7 | 수정 link (tested before final submit): returns to step 3 (부가정보입력) with phone/id/pw/account-no/account-holder-verified-state all intact | **PASS** |
| 8 | Tab-switch reset: switching away from su-co (even from its completed 완료 screen) via the confirm modal fully resets it — step-1 visible, step-done hidden, nav visible, agree-all/terms unchecked, 다음→"동의하기"/disabled, biz-status cleared, phone/account fields cleared | **PASS** |

#### 투자자회원 (su-inv)

| # | Checklist item | Result |
|---|---|---|
| — | `#su-inv-brn` / `#su-inv-biz-status` do not exist (`getElementById` returns `null`) — no biz check for this tab | **PASS** |
| 1 | 약관동의: 4 required terms, agree-all works, 다음 disabled until checked | **PASS** |
| 2 | 본인인증: name (김투자) + birth (19900101) + 본인인증하기 → "인증 중..." → after ~0.8s `ok`/"본인인증 완료"; 다음 enabled only after | **PASS** |
| 3 | 행정정보자동조회 (실명확인): loading → result; `su-inv-admin-person-name` = "김투자", `su-inv-admin-person-birth` = "19900101" (both match step-1 input) | **PASS** |
| 4 | 부가정보입력: empty → toast, blocked; pw≠pw2 → toast, blocked; after correction, advances | **PASS** |
| 5 | 최종확인: recap (`su-inv-name`, `su-inv-birth`, `su-inv-type`, `su-inv-id`) shows real values; 제출하기 → modal → 취소 stays put → re-submit → 완료, nav hidden | **PASS** |
| 6 | 완료: "투자자회원 가입이 완료되었습니다" + 로그인하러 가기 | **PASS** |
| 7 | 수정: returns to step 3 with id/pw intact | **PASS** |
| 8 | Tab-switch reset (su-inv → su-pt, from su-inv's 완료 screen): step-1 visible again, 다음 disabled, done-screen hidden | **PASS** |

#### 협약기관 (su-pt)

| # | Checklist item | Result |
|---|---|---|
| 1 | 약관동의: **5** required terms (온기업 이용약관/개인정보/마이데이터/행정정보공동이용/위탁과업수행기관준수사항) + 2 optional; agree-all works, 다음 disabled until all 5 checked | **PASS** |
| 2 | 사업자 상태 확인하기 (`123-45-67890`) → ok after ~1.2s; 본인인증하기 (이감정) → ok after ~0.8s; 다음 disabled with only one done, enabled after both | **PASS** |
| 3 | 행정정보자동조회: `su-pt-admin-rep-name` = "이감정" (matches step-1 name) | **PASS** |
| 4 | 부가정보입력: empty → toast; pw≠pw2 → toast; 예금주 확인 (계좌 `110987654321`) → "이감정 (확인됨)" class `ok`; blocked without it (toast "정산계좌 예금주 확인을 완료하세요") | **PASS** |
| 5 | 최종확인: recap (`su-pt-name`, `su-pt-brn`, `su-pt-org-type`, `su-pt-phone`, `su-pt-id`, `su-pt-account-no`) all real values; submit modal 취소/제출하기 flow correct | **PASS** |
| 6 | 완료: "협약기관 가입이 완료되었습니다" + 로그인하러 가기 | **PASS** |
| 7 | 수정: returns to step 3 with phone/id/account-holder intact | **PASS** |

Note: the cutoff scenario (999-99-99999) was tested once on su-co per the checklist's instruction. It shares the exact same `runBizStatusCheck` function as su-pt (both are `isBizPrefix`), so Bug #1 below applies identically to su-pt — not re-tested a second time there to avoid redundant evidence, but it is the same code path.

### Also checked

- **Desktop (1280px):** `#/signup` loads with `scrollWidth` (1265) ≤ `innerWidth` (1280) — no horizontal overflow. `#signup-stepper-co` renders all 6 steps with correct labels (약관동의/본인인증/정보 자동조회/부가정보입력/최종확인/가입완료).
- **Mobile (375px):** fresh-tab retest confirmed `scrollWidth === innerWidth === 375` at step 0 (stepper, 6 items), step 1 (status badge after 사업자 상태 확인, rect 309×19.5, fully in-viewport), step 2 (두 result cards, each 309px wide, no clipping), and step 3 최종확인 (`signup-stepper-co` width 343 < 375, no overflow). No cards/badges clipped at any tested step.
- **Console errors:** none observed (`read_console_messages` with `onlyErrors:true`) across the entire desktop run (all 3 tabs, full flow, submissions) or the mobile run.

### Bugs found

**Bug #1 — 휴폐업 사업자 cutoff test value never triggers the failure path (su-co and su-pt).**

- **Where:** `index.html` line 3879, inside `runBizStatusCheck(prefix)`:
  ```js
  if(val.replace(/-/g, '') === '999999999'){
  ```
- **Repro:** On su-co (or su-pt) step 1 (본인/기업인증), type exactly `999-99-99999` into the business-registration-number field — this is the **exact value the screen's own on-screen hint text instructs testers to use** ("테스트를 위해 사업자등록번호에 999-99-99999를 입력하면 휴폐업 사업자 시나리오를 확인할 수 있습니다.", visible at both `su-co-step-2` and `su-pt-step-2`). Click 사업자 상태 확인하기, wait ~1.3s.
- **Expected:** badge shows `fail` class + "휴폐업 상태로 확인되어 가입을 진행할 수 없습니다", plus a toast, and 다음 stays blocked.
- **Actual:** badge shows `ok` class + "계속사업자로 확인되었습니다" (the success path) — confirmed twice, independently, in two separate fresh browser sessions.
- **Root cause:** `'999-99-99999'.replace(/-/g, '')` produces `'9999999999'` (10 digits/nines — matching the 10-digit format used everywhere else in this form, e.g. the field's own placeholder `000-00-00000` = 3+2+5 = 10 digits), but the comparison literal in the code, `'999999999'`, is only **9** nines/digits — one digit short. The strings can never be equal, so the failure branch is dead code as currently written; the documented QA/demo scenario for this cutoff is broken.
- **Fix suggestion (not applied — review only):** change the literal on line 3879 to `'9999999999'` (10 nines) to match the 10-digit BRN format and the on-screen instructions.
- **Scope of impact:** affects both `su-co` and `su-pt` (both share `isBizPrefix` / `runBizStatusCheck`); does not affect `su-inv` (no biz check on that tab).

No other functional bugs were found. All other checklist items (agree-all toggle, per-required-checkbox gating, dual-gate on 다음 in step 1, admin-data auto-fill matching step-1 input, step-3 required/password/account validations and their toasts, 최종확인 recap accuracy, 수정 preserving entered values, submit-modal cancel/confirm, completion screen copy + 로그인하러 가기, and tab-switch full-reset) passed on all 3 tabs.

### Overall verdict

**Bug #1 fixed.** `runBizStatusCheck`'s cutoff literal (`index.html:3879`) was corrected from `'999999999'` (9 digits) to `'9999999999'` (10 digits, matching the stripped form of `999-99-99999`). Re-verified directly in a fresh browser session: entering `999-99-99999` now yields `class="auth-status-badge fail"` / "휴·폐업 상태로 확인되어 가입을 진행할 수 없습니다", and a normal number (e.g. `123-45-67890`) still yields `class="auth-status-badge ok"` / "계속사업자로 확인되었습니다" (no regression). Since `su-co` and `su-pt` share the same `runBizStatusCheck` function, the fix applies to both.

**Ready to commit.** Everything in the redesigned 6-step signup flow — across all 3 member-type tabs, all step transitions, all validation/toast paths, the 수정 (edit) round-trip, the submit-confirm modal, the tab-switch-confirm-and-reset flow, mobile/desktop responsive rendering, and now the 휴·폐업 cutoff scenario — passes with no known defects and no console errors.

## 기업지원프로그램 PRD 반영 (맞춤매칭/찜/진단/매칭신청)

### Scope confirmed in source

- `index.html` line 1366: `<section class="screen" data-screen="program-matched">` — new "맞춤 매칭" screen, populated by `renderMatchResults()` (line 4216) from the hardcoded `MATCH_RESULTS` array (line 4211: `program-snlb` 92%, `program-workout` 78%, `program-fund` 65%, each with `reasons` bullets and 제도 안내/온라인 신청/찜하기 buttons).
- `index.html` line 2046: `<section class="screen" data-screen="mypage-bookmarks">` — new "관심 프로그램" screen (`#bookmarks-list` / `#bookmarks-empty`), populated by `renderBookmarksList()` (line 4260).
- `index.html` line 3182: `MYPAGE_SCREENS = ['mypage-status', 'mypage-docs', 'mypage-account', 'apply-cert', 'program-matched', 'mypage-bookmarks']` — both new screens added to this login-gate list, as described.
- Bookmark system confirmed exactly as described: `.bookmark-toggle-btn[data-program-id]` present on 8/10 `program-overview` cards (absent on 자가진단 line 1352 and 투자 매칭 line 1353) and on all 3 `program-matched` cards (line 4222); `toggleBookmark()` (line 4249) and the delegated `document.body` click listener (line 4256-4259) toggle `bookmarkedPrograms`, flip button class/text (line 4240-4245), and call `renderBookmarksList()` on every toggle so any visited `mypage-bookmarks` view re-renders immediately.
- `program-diagnosis`: `#diag-member-banner` (line 1384) hidden by default; `renderDiagnosisMemberBanner()` (line 3217) shows it only when `isLoggedIn && currentRole === 'company'`. Input values (86/214/12/중소기업, lines 1391-1394) are static regardless of login state, matching the "light-touch demo" description.
- `program-investor-match`: `#investor-match-status-banner` (line 1427, default `display:none`) and `#investor-match-apply-btn` (line 1446, `data-prefill="투자자 매칭 지원"`); `renderInvestorMatchStatus()` (line 4280) toggles both based on module-level `hasSubmittedMatchRequest` (line 4279, default `false`). `showDone()` (line 3441-3450) sets it to `true` only when `#confirm-program`'s text is exactly `"투자자 매칭 지원"`.
- Home dashboard: `home-dashboard-company` (the only one of the 4 dashboard variants with a "심사 진행 현황 전체보기" button) now has a second button at line 437, `data-goto="program-matched"`, text "맞춤 매칭 결과 보기" (previously `program-diagnosis`/"맞춤 지원 진단" per the task description). The big hero CTA (`data-goto="program-diagnosis"`) is unchanged and identical across all 4 dashboard variants (lines 371/426/492/544-ish).
- IA placement: line 2952 `{id:'program-matched', label:'맞춤 매칭'}` is the **first** item under "기업지원 프로그램" in the main `IA` array (so it renders in the GNB dropdown, mobile nav, and sitemap for all users, logged in or not). Line 2995 `{id:'mypage-bookmarks', label:'관심 프로그램'}` is the last item under "마이페이지" in `AUTH_IA`, which by design (per the code's own comment, line 2988-2989) only surfaces through the MyGOV dropdown / mobile 마이메뉴 tab post-login, not the GNB/sitemap.

All parts of the described change are present in source exactly as described.

### What was tested

Tool: `mcp__Claude_Browser` against `file:///C:/Users/tagi7/Desktop/claude_exam/product-claude-kamco/index.html`, driving every interaction through `javascript_tool` (`element.click()`, direct `location.hash` assignment) per this sandbox's standing click-registration caveat — coordinate/ref `computer` clicks don't reliably register as trusted here, and `location.hash` assignment is a faithful proxy for `data-goto`/`href="#/..."` navigation since this is a hashchange-driven fake-SPA.

**Tab-cap note:** the session already had 9 tabs open (cap). Closed 7 stale ones, then `tabs_create` succeeded and opened a genuinely fresh tab (`tab-11`), force-navigated to the file. Verified freshness via `fetch(location.pathname, {cache:'no-store'})` before testing — confirmed the on-disk file (containing both `program-matched` and `mypage-bookmarks`) matched what the tab rendered.

**One-tick hashchange lag:** as in prior reviews, a `location.hash` assignment and an immediate read in the *same* `javascript_tool` call sometimes showed the pre-navigation screen; a follow-up read in a separate call always showed the correct, settled state. All results below use the settled reads.

`loginAs('company')` (confirmed to exist on `window` before use) was called directly to simulate company login, per the task's guidance.

No console errors (`read_console_messages`, `onlyErrors:true`) were observed at any point across the entire session (guest pass, login, bookmarking, full 4-step apply submission, viewport resizes).

### Results

**A. Guest (not logged in)**

| # | Item | Result |
|---|---|---|
| 1 | `#/program-matched` direct nav / nav-item click → redirect to login | **FAIL — see Bug #1.** Setting `location.hash = '#/program-matched'` on a fresh, logged-out tab rendered the full screen directly (`document.querySelector('.screen.active').dataset.screen === 'program-matched'`, `document.title === '맞춤 매칭 \| 온기업'`, 3 match cards fully rendered) — no toast, no redirect to `#/login`. Also reproduced by finding and `.click()`-ing the actual GNB link (`document.querySelector('a[href="#/program-matched"]')`, confirmed to be a real `<a href="#/program-matched">맞춤 매칭</a>` in the rendered IA menu) — same bypass, same result. |
| 2 | `#/mypage-bookmarks` direct nav → redirect to login | **FAIL — see Bug #1.** Identical: `location.hash = '#/mypage-bookmarks'` from a logged-out state rendered `data-screen="mypage-bookmarks"` directly (title `"관심 프로그램 \| 온기업"`), no gate. |
| 3 | `program-diagnosis`: banner hidden, manual flow unaffected | **PASS.** `#diag-member-banner` computed `display: 'none'` while logged out. Field values as-shipped: `diag-revenue=86`, `diag-debt=214`, `diag-equity=12`, `diag-size=중소기업`. Clicking `#run-diagnosis` disabled the button and showed the spinner immediately; after ~1.4s, `#diagnosis-result` computed `display` became `'flex'` and the button re-enabled — no regression. |
| 4 | `program-overview`: 8 cards have 찜하기, 자가진단/투자매칭 don't | **PASS.** Programmatic sweep of all 10 `program-overview` cards confirmed `.bookmark-toggle-btn` present on exactly the 8 expected cards (자산매입 후 임대, 선박매입 후 임대, 회생·워크아웃, 펀드 출·투자, 기업구조혁신펀드, PF사업장 정상화, 담보부사채, 유관기관 지원정책) and absent on "내게 맞는 지원 프로그램 찾기" and "투자자 매칭 지원". |
| 5 | Bookmarking as guest — no login gate, no JS error | **PASS.** Clicking `.bookmark-toggle-btn[data-program-id="program-snlb"]` as a guest worked immediately: text flipped `찜하기` → `찜 해제`, class flipped `secondary` → `primary` (no `pointer`/exception thrown), confirmed via `read_console_messages` showing zero errors. Toggled back off afterward to leave state clean. Bookmarking indeed has no login gate by design, as the task anticipated. |

**B. Logged in as company** (`loginAs('company')`, confirmed `#mygov-username` → `이수민님`)

| # | Item | Result |
|---|---|---|
| 6 | Home dashboard small button → "맞춤 매칭 결과 보기" → `program-matched` | **PASS.** `#home-dashboard-company [data-goto="program-matched"]` found with `textContent === '맞춤 매칭 결과 보기'`; clicking it navigated to `hash === '#/program-matched'`, `screen === 'program-matched'`. |
| 7 | `program-matched`: 3 cards, score badges, reason bullets, 3 buttons each | **PASS.** All 3 cards rendered: 92% (자산매입 후 임대(S&LB), 2 reasons), 78% (회생·워크아웃 기업 지원, 1 reason), 65% (펀드 출·투자(기업지원펀드), 1 reason); each card had exactly `[제도 안내, 온라인 신청, 찜하기]` buttons. |
| 8 | `program-diagnosis`: banner visible when logged in as company | **PASS.** `#diag-member-banner` computed `display: 'flex'`, text confirmed "회원님의 재무정보가 자동으로 반영되었습니다...". |
| 9 | Bookmark on `program-matched` + `program-overview`, cross-screen sync, un-bookmark | **PASS, fully.** Bookmarked `program-fund` from `program-matched` and `program-workout` from `program-overview`. Navigated to `mypage-bookmarks`: both appeared (`펀드 출·투자(기업지원펀드)`, `회생·워크아웃 기업 지원`), each with `[자세히 보기, 찜 해제]`. Clicked "찜 해제" on the 펀드 card — list re-rendered immediately in the same call (`count` 2→1, remaining item = 회생·워크아웃). Navigated back to `program-overview`: `program-fund`'s button had reverted to `찜하기`/`secondary`; `program-workout`'s button still showed `찜 해제`/`primary` — cross-screen state sync confirmed both ways. |
| 10 | Empty state after un-bookmarking everything | **PASS.** After removing the remaining `program-workout` bookmark and revisiting `mypage-bookmarks`: `#bookmarks-list` had 0 items, `#bookmarks-empty` computed `display: 'block'`, and its button read exactly "기업지원 프로그램 한눈에 보기". |
| 11 | `program-investor-match` full apply round-trip → status banner flips | **PASS.** Initial state: banner `display:'none'`, `#investor-match-apply-btn` visible. Clicked the button → `#program-select` value and `#confirm-program` text both became exactly `"투자자 매칭 지원"` via the real `data-prefill` mechanism (not manually set), landing on `#/apply/1`. Stepped through: step 1 (already valid) → step 2 → step 3 (clicked `#btn-mydata-collect`, waited ~1.4s, 4/5 required docs auto-attached as designed; manually clicked the 5th doc's own "첨부하기" `.doc-toggle-btn` to satisfy `docsComplete()` — an existing, unrelated document-upload gate, not part of this feature) → step 4 (`#confirm-program` still "투자자 매칭 지원") → clicked `#btn-next` ("신청 최종 제출") → confirm modal → clicked `#btn-confirm-submit` → `hash` became `#/apply/done`, `#step-done` visible. Navigated back to `program-investor-match`: banner now `display:'flex'`, text "이미 매칭 신청을 접수하셨습니다" / "검토중" badge present, and `#investor-match-apply-btn` now `display:'none'`. |
| 12 | "맞춤 매칭" in GNB 기업지원 프로그램 category; "관심 프로그램" under 마이페이지 (MyGOV dropdown / mobile 마이메뉴) | **PASS.** Two `<a href="#/program-matched">맞춤 매칭</a>` links found in the rendered menu (desktop GNB + mobile nav), both inside the 기업지원 프로그램 IA group. After company login, `#mygov-menu-list` (desktop MyGOV dropdown) contained `<a href="#/mypage-bookmarks">관심 프로그램</a>` as the 4th item after 신청 및 진행 현황/제출 서류.../회원정보..., and `#mobile-mymenu-list` contained the identical 4th item. (Sitemap grid does *not* show "관심 프로그램" — expected, since `AUTH_IA` items are intentionally excluded from the sitemap per the app's own existing design, confirmed via source comment at line 2988-2989; the task only requires it under MyGOV dropdown/mobile 마이메뉴, which both pass.) |

**C. Regression / cross-cutting**

| # | Item | Result |
|---|---|---|
| 13 | No console errors throughout | **PASS.** `read_console_messages({onlyErrors:true})` returned empty at multiple checkpoints across the entire guest + logged-in + full-apply-submission session. |
| 14 | No horizontal overflow at 375px / 1280px on `program-matched` / `mypage-bookmarks` | **PASS.** At 375×812: `program-matched` `scrollWidth === clientWidth === innerWidth === 375`; `mypage-bookmarks` (with 1 bookmarked item) same `375/375`. Screenshots confirm clean single-column card rendering, no clipping, no horizontal scrollbar. At 1280×900: both screens `scrollWidth === clientWidth === 1265` (vs `innerWidth` 1280, matching the established 1265/1280 padding pattern from prior reviews) — no overflow. |

### Bugs found

**Bug #1 — `program-matched` and `mypage-bookmarks` are not actually login-gated for direct hash navigation or `<a>`-tag clicks (checklist items 1 & 2 FAIL).**

- **Where:** `index.html` line 3155-3176, `handleRoute()`, and line 3184-3208, the `document.body` click-delegation listener.
- **Root cause:** the login gate for `MYPAGE_SCREENS` (line 3182) is implemented *only* inside the `document.body` click listener (line 3200: `if(!isLoggedIn && MYPAGE_SCREENS.includes(target))`). That listener is reached solely via `[data-goto]` elements, and it explicitly bails out early for anchor tags: line 3198, `if(el.tagName === 'A') return; // href="#/id"가 이미 라우팅을 처리`. `handleRoute()` itself (the actual hashchange handler, which runs for *every* navigation regardless of how the hash changed) contains no `isLoggedIn`/`MYPAGE_SCREENS` check at all — it only validates the hash against `VALID_SCREENS`.
- **Impact:** Because the GNB/사이트맵/MyGOV-dropdown render their program-matched and mypage-bookmarks entries as real `<a href="#/...">` links (confirmed in source and in the live DOM), and because the router itself never checks login state, a guest can reach both "login-gated" screens by (a) typing/pasting the URL hash directly, (b) using browser back/forward to a previously-hashed URL, or (c) clicking the GNB/sitemap `<a>` link itself — all three bypass the gate. Only clicking a `<button data-goto="program-matched">` (the one instance being the company-only home-dashboard button, which a guest never sees in the first place) goes through the gate-checking code path at all.
- **Repro (either):**
  1. Fresh/logged-out tab → `location.hash = '#/program-matched'` (or `'#/mypage-bookmarks'`) → screen renders directly with full content, no toast, no redirect.
  2. Fresh/logged-out tab → click the "맞춤 매칭" link in the GNB (`document.querySelector('a[href="#/program-matched"]').click()`) → same result.
- **Scope note — pre-existing, not a new regression:** this is not a defect introduced by this diff. The same gate-only-via-click-on-non-anchor-elements design already existed for the pre-existing `MYPAGE_SCREENS` entries; re-tested directly: a fresh logged-out tab navigating via `location.hash = '#/mypage-status'` (an already-shipped, previously-reviewed screen, untouched by this change) shows the same bypass (`screen` becomes `'mypage-status'` directly, no gate). So the new `program-matched`/`mypage-bookmarks` entries simply inherit an existing structural gap in the login-gate mechanism rather than introducing a new one — but the task's explicit checklist items 1 and 2 do fail as written, since "confirm it redirects to login with a toast" does not hold for direct navigation.
- **Fix suggestion (not applied — review only):** move the `MYPAGE_SCREENS`/`isLoggedIn` check into `handleRoute()` itself (or into `showScreen()`), so it applies uniformly regardless of navigation method (hash paste, back/forward, `<a>` click, or `<button data-goto>` click), rather than only inside the click-delegation listener with its anchor-tag carve-out.

No other functional bugs were found. All bookmark-state synchronization, the diagnosis member banner, the investor-match status banner/apply-button toggle (including the full real 4-step apply submission path), the home-dashboard button relabel/retarget, and IA/menu placement all behave exactly as designed.

### Overall verdict

**Bug #1 fixed.** The login gate was moved from the `document.body` `[data-goto]` click handler into `handleRoute()` itself (`index.html`, right after the `VALID_SCREENS` check, before `showScreen(id)`): `if(!isLoggedIn && MYPAGE_SCREENS.includes(id)){ pendingRedirect = id; toast(...); navigateTo('login'); return; }`. Since every navigation path (typed hash, `<a href="#/...">` clicks, back/forward, and `data-goto` buttons) ends up going through `handleRoute()` via the hashchange event or `navigateTo`'s direct call, this covers all of them uniformly. The now-redundant duplicate check in the click handler was removed (`navigateTo()` already triggers `handleRoute()`, which re-applies the same gate) to avoid maintaining two copies of the same logic.

Re-verified directly in a fresh browser session: `location.hash = '#/program-matched'` and `'#/mypage-bookmarks'` as a guest both now redirect to `#/login` with the toast. Clicking the actual GNB `<a href="#/program-matched">` link produces the same result. Confirmed no regression: `#/mypage-status` (the pre-existing screen that had the same architectural gap) is now also correctly gated via direct hash nav — previously it wasn't, so this is a net improvement, not just a fix scoped to the two new screens. Logged-in access to `program-matched` continues to work normally (`loginAs('company')` then `location.hash='#/program-matched'` renders the screen correctly). No console errors.

**Ready to commit.** All 14 checklist items now pass: cross-screen bookmark sync, the full apply-to-completion round trip driving the investor-match banner, mobile/desktop responsive rendering, and now the login-gate coverage for both new screens (plus the incidental fix for the older `mypage-status` gap).
