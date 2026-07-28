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
