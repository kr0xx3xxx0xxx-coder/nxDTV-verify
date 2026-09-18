```text
작업명 : BATCH-WORKFLOW-LIST-GRID-SYNTAXCHECK-AUTORUN-GAP-ASSESS-RERUN
✅ 작업 완료 - ①②③④ 4개 항목 코드+실제 브라우저 스크린샷 재조사 완료(코드 수정 없음), 절충안 제안까지 완료

## 목적
사용자가 원하는 전체 워크플로(①업로드 후 배치ID+파일명 목록 → ②클릭 시 행별
데이터 그리드 → ③가벼운 문법체크 버튼 → ④문제없으면 실행까지 자동 진행)를
코드 레벨 + 실제 브라우저 스크린샷으로 짐작 없이 재확인한다. 이전 실행(RERUN이
아닌 원본 지침)의 완료보고서가 "Drive에서 비어 읽힌다"는 문제 제기로 재실행이
지시됐다. 코드 수정은 전혀 하지 않았다(Read/Grep/실제 서버 기동+Playwright 조사만).

## 현상 (조사 전 상태 확인)
- 이전 완료보고서(`BATCH-WORKFLOW-LIST-GRID-SYNTAXCHECK-AUTORUN-GAP-ASSESS_20260918.md`,
  Drive file id `1qB4YJtoPIuekcdwRDYPPH-JtUC0vfpIV`)를 `get_file_metadata`/
  `download_file_content`로 직접 재확인한 결과 **파일 크기 9,737바이트, base64
  디코드 시 완전한 한글 보고서 본문**이 확인됐다 — "비어 읽힌다"는 재실행 지침의
  전제와 달리 실제로는 비어있지 않았다(원인은 불명 — 조회 시점/클라이언트 캐시
  문제였을 가능성). 다만 그 이전 보고서의 일부 결론(②의 "migration_sql 컬럼
  자체가 없다")이 코드 직접 확인 결과 부정확했음을 이번 재조사에서 발견해 아래
  ②에서 정정했다.
- 이번 재실행은 실제 도구 호출 없이 계획만 재진술했던 1차 시도(0회 도구 호출,
  2초 종료)를 사용자가 직접 지적해 재작업한 결과다. 아래 근거는 모두 실제
  Read/Grep/git log/Drive API/Playwright 스크린샷 실행 결과다.

## 검증 결과 (①~④ 항목별, 파일:줄 근거 + 실제 스크린샷)

### ① 배치ID+파일명 목록 — 충족, 새로 만들 필요 없음
- `ui/tabler_renderer.py:12259-12271` "업로드 배치 이력" 표 헤더가 작업그룹 ID·
  파일명·업로드시각(KST)·행수·파싱OK·DB검증OK·DB실패·후보·상태·작업(상세/결과XLS/
  원본XLS/배치제외)을 이미 제공.
- `routes/batch_route.py:1575-1582`가 `total_rows`/`parse_ok_count`/`db_ok_count`/
  `db_fail_count`/`execution_status` 필드로 이 표를 공급.
- **실측**: 운영 DB(`db/migration_validator.db`, 오늘 FULL-DATA-RESET 이후라 활성
  배치 0건)를 격리 사본으로 복사한 뒤(→ 아래 "격리 환경" 참고) group `GRP_001`
  (인사급여1)에 테스트용 row 1건만 격리 사본에 직접 삽입해 표 렌더링을 실제
  화면으로 확인. 스크린샷: `02_history_area.png` — "업로드 배치 이력" 표에
  작업그룹 ID(GRP_001)·파일명(e2e_verify_sample.xlsx)·시각·행수(8)·파싱OK(8)·
  DB검증OK(6)·DB실패(2)·후보(5)·상태(업로드 완료)·작업(상세/결과XLS/원본XLS/
  배치제외) 모두 정상 렌더 확인.

### ② 클릭 시 그리드로 행별 데이터 — 틀은 충족, "SQL 원문 미표시"만 정정 필요(이전 보고서보다 더 작은 작업)
- "상세" 버튼(`button[onclick^='batchSelectRunFromList']`, `ui/tabler_renderer.py:12297-12298`)
  → `batchSelectRunFromList()`(12823) → `batchLoadBatchDetail()`(12846) →
  `GET /api/batch-runs/{id}/detail`(`routes/batch_group_route.py:318-343`) →
  `batchDetailBoard`(12901-12942)가 행번호·목적지테이블·원본테이블·파싱(OK/실패)·
  DB검증(PASSED/FAILED/SKIP)·최신·중복상태를 HTML `<table>`로 렌더.
- **실측**: 스크린샷 `03_after_detail_click.png` — "상세" 클릭 시 "배치 상세"
  패널이 실제로 펼쳐지고 요약 필드(작업그룹 ID/그룹명/업로드 ID/파일명/시각/총/
  파싱OK/DB검증OK/DB검증실패)가 정상 렌더됨을 확인. row 상세 표는 "이 배치에
  row가 없습니다"로 표시됐는데, 이는 이번 격리 사본 테스트가 배치런 요약(1행)만
  심고 row 상세 테이블(`DTV_policy_target_table_config`)은 심지 않았기 때문이며
  ─ 프레임(패널 오픈→요약 렌더) 자체의 동작은 정상 확인됨.
- **이전 보고서 정정(중요)**: 이전 보고서는 "이관 SQL 원문(migration_sql) 컬럼
  자체가 없다"고 결론지었으나, 코드를 직접 재확인한 결과 이는 부정확하다.
  - `/api/batch-runs/{id}/detail`가 호출하는 `services/policy_target_table_service.py:988-1017`의
    `get_items_by_source_batch()`는 `SELECT ... migration_sql, ... FROM
    DTV_policy_target_table_config`로 **migration_sql 컬럼을 이미 조회해서
    API 응답(`items[].migration_sql`)에 포함**시키고 있다(`services/policy_target_table_service.py:1006`).
  - 다만 프론트엔드 렌더 함수 `batchDetailBoard`(`ui/tabler_renderer.py:12911-12942`)가
    그 응답에서 `migration_sql` 필드를 테이블 컬럼으로 그리지 않을 뿐이다(현재
    컬럼: 행/목적지테이블/원본테이블/파싱/DB검증/최신/중복상태 — SQL 컬럼 없음).
  - 결론: "행별 SQL 내용을 그리드에서 바로 확인"이 필요하면, **백엔드 변경 없이
    프론트엔드 렌더(`ui/tabler_renderer.py:12911` 부근 `<th>`/`<td>` 한 줄 추가)만으로
    해결 가능** — 이전 보고서가 예상한 것보다 더 작은 작업.
  - Tabulator는 이 화면에서 미사용(일반 HTML `<table>`). Tabulator는 통계검증계획
    등 다른 화면(`ui/grid_helpers.py`)에서만 쓰인다(CLAUDE.md 예외 목록과 일치,
    사용처 확대 아님).

### ③ 문법체크(품질점검) 버튼 범위 — 순수 문법체크 아님, DB접속과 결합됨
- `services/upload_quality_check_service.py:394-428` `_run_quality_check_locked()`가
  `enforce_connections: bool = True` 기본값으로 진입 시점부터
  `preflight_quality_connections()`(206-231)를 강제 실행 — Source/Target DB
  접속 실패 시 `blocked: True`로 즉시 중단하고 파싱조차 실행하지 않는다(423-428행
  직접 확인).
- row 루프 내부에서는 파싱(`parse_single_sql()` 565, DB 불필요)과 DB검증
  (`_check_src_tables()` 623, `validate_sql_db_objects()` 628, DB 필요)이 순서상
  분리돼 있으나, 진입점의 preflight 게이트 때문에 이 서비스를 그대로 재사용해
  "순수 문법체크 버튼"을 만들 수는 없다 — (1) preflight 우회 분기, (2) 622-641
  DB검증 블록 스킵 분기, 최소 두 곳을 새로 나눠야 한다.
- 버튼 위치: `ui/tabler_renderer.py:3839-3841` — `id="quQualityBtn"`,
  "업로드 관리 · 품질점검" 카드(`quHistoryCard`) 내부, **"이관쿼리 업로드"
  화면(사이드바 "데이터 준비 › 이관쿼리 업로드")** — 일괄검증 1번탭과는 다른 화면.
- **실측**: 스크린샷 `04_query_upload_tab.png` — "이관쿼리 업로드" 화면 진입
  확인(사이드바 라벨 "업로드 이력·품질점검 포함"과 일치). 프로젝트 미선택
  상태라 `quQualityBtn`이 포함된 `quHistoryCard`는 `display:none`으로 화면에
  보이지 않았으나(정상 동작 — 프로젝트 선택 후 노출되는 조건부 카드), 페이지
  DOM 내 "품질점검" 텍스트 매치 7건으로 버튼 자체의 존재는 확인됨.
- 반대로 일괄검증 1번탭(`routes/batch_route.py`) 업로드 흐름은 이미 파싱과
  DB검증을 분리된 카운터로 갖고 있다: `parse_ok_count`/`parse_fail_count`
  (1578-1579), `db_ok_count`/`db_fail_count`/`db_pending_count`(1580-1582) —
  화면(`02_history_area.png`)에서도 파싱OK(8)/DB검증OK(6)/DB실패(2)가 별도
  컬럼으로 이미 분리 표시됨을 실측 확인. 순수 문법체크 버튼을 새로 만든다면
  "품질점검" 서비스를 억지로 재사용하기보다, 일괄검증 1번탭에 이미 있는
  parse_ok_count 산출 경로(업로드 시 자동 실행되는 파싱)를 그대로 노출하는
  방향이 구조적으로 더 가깝다.

### ④ "문제없으면 실행 눌러서 자동으로 쭉" vs 오늘 확정된 안전설계 — 충돌 지점 명확히 확인
- 오늘(2026-09-18) 커밋 `dc01d4f5`(멀티프로젝트 배치 워크플로 사용성 3건 수정,
  MULTI-PROJECT-BATCH-WORKFLOW-USABILITY-FIX-3ITEMS) 및 `80f99f65`(일괄검증
  1번탭 자동진행 토글 제거, BATCH-STAGE1-UPLOAD-DRAGDROP-REPOSITION-AND-AUTORUN-TOGGLE-SIMPLIFY)로
  "자동 진행" 토글이 제거되고 **상시 자동화**가 기본 동작이 됐다(git log 직접 확인).
- 안전설계 본체(코드 직접 확인, `ui/tabler_renderer.py`):
  - `_batchAutoRunAfterCandidate(hasCandidates)`(11628-11645): 후보 컬럼 선정
    완료 시 무조건 `batchAutoRunConfirmBanner`를 노출하고 멈춘다("GROUP BY/SUM
    선정 결과를 확인한 후 전체 통계검증으로 자동 진행을 계속하시겠습니까?").
  - `_batchAutoRunConfirmProceed()`(11650-11678): "▶ 그대로 실행" 클릭 시에만
    `batchGenerateStatsPlans()`로 위험도(카디널리티)를 재계산하고,
    `execution_risk_level`이 `HIGH`/`VERY_HIGH`인 계획이 하나라도 있으면
    (11666-11671) **alert로 사용자에게 알리고 자동 진행을 다시 멈춘다** — 4단계
    (전체 통계검증) 실행은 그 이후에만 진행된다.
  - **실측(화면 문구, `01_batch_tab.png`/`02_history_area.png`에 그대로 노출)**:
    "자동 진행 : COUNT 완료 → 후보 컬럼 선정은 항상 자동 실행됩니다. 후보선정→
    전체 통계검증은 자동 실행되지 않고 확인 배너를 거칩니다 · COUNT 불일치 시
    동작은 위 '검증 작업 그룹' 카드의 'COUNT 불일치 시' 설정을 따릅니다." — 코드
    분석과 실제 화면 문구가 정확히 일치.
- **충돌 여부**: ④("문제없으면 실행 눌러서 자동으로 쭉")는 이 안전설계와
  **정확히 3→4단계 경계 1곳에서 충돌**한다. 1→2(COUNT)→3(후보선정)까지는 이미
  사람 개입 없이 자동 진행되므로 ④ 요구와 충돌이 없다. 3→4 경계에서만 (a)
  확인배너 클릭이 강제되고, (b) 위험도가 HIGH/VERY_HIGH면 그 이후에도 한 번 더
  멈춘다 — 이 두 지점이 "문제없으면 자동으로 끝까지"라는 요구와 정면으로 부딪힌다.
- **절충안 제안(실행하지 않음, 제안까지만)**:
  1. (a) 확인배너: "문제없음"의 판정 기준을 명시적으로 좁히면(예: 후보 0건이
     아니고 COUNT 정책이 이미 사전설정된 경우) 배너를 자동으로 "그대로 실행"
     처리하는 옵션을 1번 탭 사전설정에 추가하는 방향이 가능하나, 이는 이미
     이번 요청과 유사한 취지로 검토된 바 있는 `BATCH-COUNT-MISMATCH-BLOCK-AND-MANUAL-PROCEED-DESIGN`
     (2026-09-17, 코드 수정 없는 설계안)의 연장선이며 구현은 별도 승인이 필요.
  2. (b) 위험도 HIGH/VERY_HIGH 자동중단은 카디널리티 기반 안전장치이므로 이
     부분은 절충 대상에서 제외하고 그대로 유지하는 것을 권장한다(사용자가
     "문제없으면"이라고 표현한 전제 자체가 위험도 높은 계획은 "문제 있음"에
     해당한다고 해석하는 것이 안전).
  3. 즉 실현 가능한 절충은 "(a)만 사전설정으로 완화 + (b)는 그대로 유지"이며,
     이는 코드 수정이 필요한 별도 지침으로 분리해 진행 여부를 사용자에게 확인
     받아야 한다(이번 지침 범위는 조사+절충안 제안까지).

## 격리 환경(CLAUDE.md 38번 규칙 준수)
- MV_DATA_DIR: `...scratchpad\batch_gap_verify_8031`(운영
  `db/migration_validator.db`를 서버 기동 전 복사, 원본은 읽기 전용 — 격리 사본에만
  테스트 row 1건 INSERT, 운영 DB는 전혀 건드리지 않음)
- MV_PRESET_DATA_DIR: `...scratchpad\batch_gap_verify_preset`(운영
  `common/db/dnp_db_preset.db` 복사본)
- MV_BIND_PORT: 8031(운영 8000과 별도), MV_AUTH_DISABLED=1(테스트 전용 우회,
  web_server.py:125 주석에 명시된 용도 그대로 사용)
- 서버 프로세스는 위 env로 새로 기동(기존 실행 중 서버 재사용 안 함) → Playwright로
  스크린샷 4장 캡처 → **확인 직후 포트 8031 프로세스 직접 종료, 재확인 완료**.

## [27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 (코드 수정 없는 순수 조사 —
  버튼/체크박스/텍스트/배지 등 화면 요소를 코드로 바꾼 사실이 없음. 다만 지침
  원문이 명시적으로 요구한 실측 증적 확보를 위해 브라우저 스크린샷은 실제로
  촬영했다.)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음(변경 없음)
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음(변경 없음) — 대신 "조사 대상
  현재 상태" 스크린샷 4장을 촬영함: `01_batch_tab.png`, `02_history_area.png`,
  `03_after_detail_click.png`, `04_query_upload_tab.png`
  (경로: 세션 스크래치패드 `batch_gap_verify_8031\` 하위)
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 4장 모두 Read 도구로 직접
  열어 내용을 확인함(예/코드 근거와 화면 문구 일치 확인)
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(실측 가능,
  격리 서버로 전부 실측함). 단 ②의 row 상세 표는 운영 DB에 활성 배치가 0건이라
  격리 사본에 최소 테스트 row 1건을 직접 삽입해 실측했음을 명시.

## 기대효과
①②③④ 각각의 실제 코드 위치·동작 범위·화면 증적이 모두 확보되어, 후속 구현
지침(SQL 컬럼 추가/순수 문법체크 분리/3→4 확인배너 완화)을 짐작 없이 바로
설계할 수 있는 상태가 됐다. 특히 ②는 이전 보고서보다 작업 범위가 더 작다는
것이 새로 확인돼(백엔드 변경 불필요, 프론트 렌더 한 줄), 우선순위 판단에 반영
가능하다.

저장 검증: 이 파일을 Drive에 저장한 직후 `get_file_metadata`/`download_file_content`로
다시 열어 파일 크기와 base64 디코드 본문이 방금 작성한 내용과 일치함을 확인함
(정확한 바이트 수는 저장 직후 재조회 결과를 아래에 기록). 저장 검증: 파일 크기 14821바이트,
내용 정상 확인됨(비어있지 않음, 본문 한글 정상 디코드 확인).

작업명 : BATCH-WORKFLOW-LIST-GRID-SYNTAXCHECK-AUTORUN-GAP-ASSESS-RERUN
✅ 작업 완료 - ①②③④ 4개 항목 코드+실제 브라우저 스크린샷 재조사 완료(코드 수정 없음), 절충안 제안까지 완료
```
