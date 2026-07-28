# spec.md — 모바일 반응형 오류 수정

## 배경
`index.html`은 40개 이상의 화면을 JS 토글로 전환하는 단일 파일 프로토타입이다. 사용자가 모바일에서 반응형이 깨진다고 보고했다. 실제 브라우저(375×812)로 주요 화면을 순회 확인한 결과, 헤더/GNB/역할 탭/그리드 컴포넌트(`.grid-2/3/4`, `.process-row`)는 이미 767px·1023px 미디어쿼리로 정상 대응되어 있었다. 깨지는 곳은 **인라인 스타일로 그리드 컬럼 폭을 고정한 2곳**뿐이다.

## 버그 1 — `.form-col-group` 2열 폼이 모바일에서 붕괴하지 않음
- 위치: `index.html:1175` (`program-diagnosis` 화면, 재무정보 입력), `index.html:1504` (`apply` Step 1, 기업정보 입력)
- 원인: `<div class="form-col-group" style="...grid-template-columns:1fr 1fr;...">` — 인라인 스타일이라 `.grid-2/.grid-3`용 767px 미디어쿼리(`index.html:50`)의 적용을 받지 못함
- 실측: `program-diagnosis`에서 375px 기준 `scrollWidth 413px` (38px 가로 오버플로우), 입력 필드·체크박스 라벨 잘림. `apply` Step 1은 오버플로우는 없으나 입력값(사업자번호, 연락처)이 좁은 폭에 시각적으로 잘림.
- 수정: 기존 767px 미디어쿼리 블록(46~50행 부근)에 규칙 추가
  ```css
  @media (max-width:767px){ .form-col-group{grid-template-columns:1fr !important;} }
  ```

## 버그 2 — VDR 뷰어 고정 26rem 사이드바가 콘텐츠를 짓눌러 텍스트가 세로로 쪼개짐
- 위치: `index.html:2022` (`vdr-viewer` 화면)
- 원인: `<div class="box box-pad" style="display:grid;grid-template-columns:26rem 1fr;...">` — 모바일 오버라이드 없음
- 실측: 375px에서 사이드바(260px)+gap(32px)=292px 차지, 콘텐츠 컬럼이 50~80px로 압축되어 안내 문구가 한 줄에 몇 글자만 남는 형태로 깨짐
- 수정: 해당 div에 `vdr-layout` 클래스 추가 후
  ```css
  @media (max-width:767px){ .vdr-layout{grid-template-columns:1fr !important;} }
  ```

## 제외 항목
게시판/목록 테이블의 가로 스크롤(`.krds-table-wrap`)은 KRDS 벤더 컴포넌트의 의도된 동작이며 페이지 레벨 오버플로우를 유발하지 않으므로 버그 아님 — 수정 범위 제외.

## Work 분할 (독립 서브에이전트 2개, 겹치는 코드 영역 없음)
- **서브에이전트 A**: 버그 1만 수정. `<style>` 블록에 미디어쿼리 규칙 1줄 추가. 마크업 변경 없음.
- **서브에이전트 B**: 버그 2만 수정. `index.html:2022` div에 클래스 추가 + `<style>` 블록에 미디어쿼리 규칙 1줄 추가.

## 검증 방법
Browser 도구로 375px 뷰포트에서 `program-diagnosis`, `apply` Step 1, `vdr-viewer` 확인 — 가로 오버플로우 없음(`scrollWidth === innerWidth`), 텍스트/입력값 잘림 없음. 1023px·1280px에서 회귀 없는지 재확인.

## 버그 3 — `.content-layout` 그리드 트랙이 버튼의 min-content로 인해 뷰포트보다 넓어짐 (메인, 온라인신청 3단계)
- 근본 원인: `.content-layout{display:grid;grid-template-columns:1fr;...}` (`index.html:30`) — CSS 그리드는 `1fr` 트랙도 기본적으로 자식의 `min-content` 이하로 줄어들지 않는다(`minmax(auto,1fr)`과 동일). `.krds-btn`(벤더 컴포넌트)은 기본 `white-space:nowrap`이라 긴 한글 버튼 텍스트의 min-content 폭이 매우 커지고, 이 값이 그리드 트랙 자체를, 나아가 `.content-layout`(및 그 안의 모든 형제 블록)을 뷰포트보다 넓게 밀어낸다.
- 실측 1 (메인 홈, 기본 글자크기): 375px 뷰포트에서 히어로의 "1분 만에 끝내는 기업 지원 자가진단 시작하기" CTA 버튼(`.hero .cta-row .krds-btn`, `index.html:369` 부근)이 그리드를 421.375px로 밀어 `scrollWidth 437 / clientWidth 375` = 62px 가로 오버플로우. `document.getElementById('content-layout')`의 computed `grid-template-columns`가 실제로 `"421.375px"`로 확인됨.
- 실측 2 (온라인신청 3단계, 헤더 "글자크기: 크게" 접근성 옵션 적용 시): "법인 인증 및 공공 서류 한 번에 가져오기" 버튼(`#btn-mydata-collect`, `index.html:1566`, `style="width:fit-content;"` + nowrap)이 같은 방식으로 그리드를 밀어 `scrollWidth 450 / clientWidth 375` = 32~75px 오버플로우(그리드 수정만 적용 시에도 407로 남음 — 버튼 자체의 nowrap이 별도 원인이라 그리드 수정만으로는 해결되지 않음).
- 수정 (브라우저에서 조합 테스트로 실제 해결 확인, `scrollWidth === clientWidth`):
  1. 그리드 트랙에 `minmax(0,1fr)` 적용 — 구조적 근본 수정, 그리드가 콘텐츠 때문에 뷰포트보다 커지는 것을 원천 차단
     - `index.html:30` `.content-layout{grid-template-columns:1fr;...}` → `grid-template-columns:minmax(0,1fr);`
     - `index.html:31` `.content-layout.has-lnb{grid-template-columns:24rem 1fr;...}` → `grid-template-columns:24rem minmax(0,1fr);`
     - `index.html:37` (1023px 미디어쿼리 안) `.content-layout.has-lnb{grid-template-columns:1fr;}` → `grid-template-columns:minmax(0,1fr);`
  2. 그리드 트랙이 줄어들면 내부의 nowrap 버튼 자체가 자기 컨테이너보다 넓어지는 문제가 남으므로, 767px 이하에서 버튼 두 곳에 `white-space:normal` 추가
     ```css
     @media (max-width:767px){
       .hero .cta-row .krds-btn{white-space:normal;}
       #btn-mydata-collect{white-space:normal;}
     }
     ```
- 참고: `.hero .cta-row`는 홈 화면에서 게스트/기업/투자자/협약기관 대시보드 4곳 모두 동일 클래스를 공유하므로(`index.html:369,424,490,543`) 이 CSS 규칙 하나로 4곳 모두 커버된다.

## Work 분할 (버그 3, 단일 서브에이전트 — 그리드 트랙 수정과 버튼 수정이 서로 의존적이라 분리 시 이점 없음)
- **서브에이전트 C**: `index.html:30,31,37`의 `grid-template-columns` 3곳 수정 + 새 767px 미디어쿼리 규칙(버튼 2곳 `white-space:normal`) 추가. 마크업 변경 없음, `<style>` 블록만 수정.

## 검증 방법 (버그 3)
Browser 도구로 375px에서 메인 홈 확인(`scrollWidth === clientWidth`, CTA 버튼 텍스트 줄바꿈되어 잘리지 않음), 온라인신청 3단계에서 기본 글자크기 및 "글자크기: 크게" 상태 모두 확인(마이데이터 버튼 텍스트 줄바꿈, 오버플로우 없음). 1280px 데스크톱에서 기존 레이아웃(1줄 버튼, 넓은 그리드) 회귀 없는지 재확인.

## 기능 4 — 모바일 햄버거 메뉴: 사이트맵 제거 + 마이메뉴 탭 신설
사용자 요청(버그 아님, 기능 변경): (1) 모바일 전체메뉴(`#mobile-nav`) 상단의 "사이트맵" 버튼 제거, (2) 로그인 시 사용자명 아래 가로 스크롤 링크 줄(`#mobile-mygov-links`)로 노출되던 마이페이지/관리자 메뉴를 없애고, 좌측 1depth 탭 목록(기업지원 프로그램/투자자 라운지/국유증권 가상데이터룸/고객마당/센터소개) 맨 아래에 "마이메뉴" 탭을 새로 만들어 그 안으로 이동.

### 1) 사이트맵 제거
- 위치: `index.html:320-324` (`.gnb-header` 안 `.gnb-utils` 블록, "사이트맵" 버튼)
- `.gnb-utils` 클래스는 이 한 곳에서만 쓰이므로(데스크톱 유틸리티 바 사이트맵 버튼(`index.html:255`)·푸터 사이트맵 링크(`index.html:2393`)·사이트맵 모달(`#sitemap-overlay`) 자체는 별개 요소라 영향 없음) 이 블록 전체 삭제.

### 2) 마이메뉴 탭
- 근본 제약: 벤더 스크립트 `krds_mainMenuMobile.init()`(페이지 로드 시 1회 실행, `krds.min.js`)이 `.gnb-main-trigger` 탭 각각에 클릭 이벤트를 직접 바인딩한다. 로그인 후 탭 목록 HTML을 새로 갈아끼우면(innerHTML 재생성) 새 탭에는 이벤트가 안 걸리고, `init()`을 재호출하면 "전체메뉴" 열기 버튼 등 기존 리스너가 중복 바인딩되는 부작용이 생긴다.
- 해결: 페이지 로드 시 정적 IA 5개 탭 뒤에 "마이메뉴" 탭(`<li id="mobile-mymenu-tab" style="display:none;">`)과 패널(`<div id="mGnb-anchor-mymenu" style="display:none;">`)을 처음부터 DOM에 함께 만들어 두되 둘 다 숨겨두고, 로그인/로그아웃 시 두 요소의 `display`만 토글한다(기존 `#mobile-mygov`/`#mobile-guest-login` 토글 패턴과 동일).
- 수정 대상:
  - `index.html:2778-2787` 부근 (`gnbMobileTabs.innerHTML = ...`, `gnbMobilePanels.innerHTML = ...`): IA 기반 목록 뒤에 마이메뉴 탭/패널 1개씩 추가로 이어붙인다.
  - `index.html:3476-3484` 부근 `renderMyGovMenu(role)`: 기존 `#mygov-menu-list`(PC 드롭다운) 채우던 것과 같은 데이터로 `#mobile-mymenu-list`(신규 `<ul>`, 마이메뉴 패널 안)도 채운다. `#mobile-mygov-links`를 채우던 줄은 제거.
  - `index.html:3494-3499` `loginAs()`: `#mobile-mygov-links`를 보이던 줄 → `#mobile-mymenu-tab`, `#mGnb-anchor-mymenu` 보이기로 교체.
  - `index.html:3518-3519` `logout()`: `#mobile-mygov-links`를 숨기던 줄 → `#mobile-mymenu-tab`, `#mGnb-anchor-mymenu` 숨기기로 교체.
  - `index.html:333` 마크업의 `<div class="row" id="mobile-mygov-links" ...></div>` 삭제(더 이상 사용 안 함).
  - `index.html:133` 근처 `#mobile-mygov-links a{...}` CSS 규칙 삭제(대상 요소가 없어지므로 죽은 코드).

## Work 분할 (기능 4, 단일 서브에이전트 — 마크업/JS 변경이 서로 강하게 연결되어 있어 분리 시 이점 없음)
- **서브에이전트 E**: 위 사이트맵 제거 1곳 + 마이메뉴 탭 관련 markup 2곳/JS 4곳/CSS 1곳 수정.

## 검증 방법 (기능 4)
Browser 도구로 375px에서 전체메뉴 열어 사이트맵 버튼이 없는지 확인. 로그인 전: 좌측 탭에 마이메뉴가 안 보이는지, 스크롤해도 빈 마이메뉴 섹션이 노출되지 않는지 확인. 기업/투자자/협약기관/관리자 각 역할로 로그인 후: 좌측 탭 맨 아래 마이메뉴 탭이 나타나는지, 클릭 시 해당 역할의 메뉴(마이페이지 3개 또는 관리자 2개)로 정상 스크롤/이동되는지, 로그아웃 시 다시 숨겨지는지 확인. PC 화면(MyGOV 드롭다운)에 회귀 없는지 확인.

## 기능 5 — 회원가입 프로세스 PRD 반영 (마이데이터·행정망 연계 간소화)

배경: `file/온기업_마이데이터연계_회원가입_PRD.md`를 근거로 현재 2단계(약관동의→회원정보입력→완료) 회원가입을 PRD의 6단계 흐름으로 재설계. 상세 설계는 `C:\Users\tagi7\.claude\plans\sleepy-pondering-boot.md`에 기록됨 (Context/새 흐름 설계/JS 상태 머신 변경 섹션 참조). 정적 프로토타입이므로 실제 API 대신 온라인신청 3단계(`#btn-mydata-collect`)의 지연 시뮬레이션 패턴을 재사용.

사용자 확정 범위: 3개 탭(기업회원/투자자회원/협약기관) 모두 적용. 정산계좌는 기업회원/협약기관에만 포함(투자자는 자금을 받는 주체가 아니므로 제외).

### Work 분할 (순차 2단계, CLAUDE.md 3+ 화면 분할 규칙 적용)
1. 서브에이전트 J: 공유 JS 상태 머신 + 제출확인 모달 + 배지 CSS (마크업은 모달 1개 추가만)
2. J 완료 후 병렬: 서브에이전트 F(기업 패널), I(투자자 패널), P(협약기관 패널) — 각자 자기 `<section id="panel-signup-XX">` 안쪽만 수정

각 서브에이전트에게 정확한 element id 컨트랙트가 담긴 전용 지침 파일을 전달함 (task-J/F/I/P-*.md, 스크래치패드).

### 검증 방법
Browser 도구로 3개 탭 각각 처음부터 끝까지(약관동의 → 본인/기업인증 → 자동조회 → 부가정보 → 최종확인 → 제출확인모달 → 완료) 클릭 진행. 기업/협약기관은 정상 사업자번호로 성공 + `999-99-99999`로 컷오프 시나리오 확인. 탭 전환 시 리셋 확인. 375px/1280px 레이아웃 확인.
