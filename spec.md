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
