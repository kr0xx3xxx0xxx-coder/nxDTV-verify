```text
작업명 : BATCH-UI-STRUCTURE-PARITY-WITH-SINGLE-INVESTIGATE-AND-PLAN
✅ 작업 완료 - 개별/일괄검증 화면 정보구조 조사 및 단계별 대응관계·구현계획 수립(조사+계획 전용, 코드 미수정)

## 목적

일괄검증(배치) 화면 전체 구조를 개별검증과 "정보구조 패턴" 차원에서 맞추기 위한
선행 조사. "똑같이 만들라"가 아니라 "개별의 탭별 정보구조 패턴을 단건→다건,
테이블중심→배치중심으로 치환한 대응 구조"를 도출하는 것이 목표이며, 이번
지침은 조사+계획 수립까지만 수행한다(실제 구현은 별도 지침으로 단계마다 진행).

오늘 이미 대기 중이던 BATCH-VALIDATION-SCREEN-SINGLE-UPLOAD-STRICT-DUPLICATE-BLOCK
(1단계 업로드 중복차단 지침)은 지시대로 본 조사·계획 수립이 끝날 때까지 착수하지
않았다(같은 파일 tabler_renderer.py를 다루므로 순서 충돌 방지).

## 현상 (조사 결과)

### 0. 전체 구조 — CLAUDE.md 서술과 실제 코드의 단계 수 불일치

CLAUDE.md·README는 "Web UI 4단계"로 서술하지만, 실제 코드는 개별·배치 모두
**5단계** 체계다.

- 개별: `ui/tabler_renderer.py:6985-7025`의 `SINGLE_STEP_CARDS` — `query`(1)→
  `count`(2)→`candidate`(3)→`validation`(4, SQL생성+실행 트리거)→`result`(5, 결과확인).
  즉 문서상 "4단계=SQL생성+실행+결과표시"가 코드에서는 4단계(트리거)와
  5단계(결과)로 분리되어 있다. pane 전환은 `_applySinglePane()`(`:7040`).
- 배치: `ui/tabler_renderer.py:11089-11095`의 `BATCH_STEP_DEFS` — 1.그룹/업로드,
  2.검증대상/COUNT, 3.후보 컬럼 선정, 4.전체 통계검증, 5.결과 요약. 개별의
  "2단계 COUNT비교"에 해당하는 것을 "검증대상 확인+COUNT 사전검증"으로 한
  단계에 통합했다고 코드 주석(`:11088`)에 명시되어 있다.

이하 1~5단계 표기는 이 **실제 코드 기준 5단계**를 따른다(CLAUDE.md의 "4단계"
표기와는 번호가 다르다는 점 주의 — 4단계/5단계 구분 자체는 이번 지침에서 수정할
대상이 아니므로 그대로 둔다).

### 1. 개별검증 1~5단계 정보구조 (파일:줄 근거)

**1단계 — 쿼리 검토(분석 실행)**
- SQL 입력 `#sqlInput`, 목적지 WHERE 수동입력 `#tgtWhere` (`tabler_renderer.py:2842-2886`)
- "쿼리 검토 결과" 카드 `#queryReviewCard`(`:2797-2826`) → `#qrSummary`에 8칸
  div 타일: 목적테이블/원본테이블/입력컬럼수/추출컬럼수/조건문/조인/실행시간/상태
  — `ui/grid_helpers.py:563-573`(`_mvQueryReviewColumns`), 렌더 `tabler_renderer.py:23910-23924, 23813-23869`
- 과거 Tabulator 그리드였으나 "SQL 1건 요약"이라는 이유로 div 타일로 전환됨
  (`grid_helpers.py:579` 주석 "1 SQL = 1 Row")
- 컬럼 분류 결과는 `#unifiedColOut`에 `display:none`으로 숨김(`:2817-2824`,
  CLAUDE.md "1단계 숨김/3단계 표시" 서술과 일치)
- 매칭 신뢰도(HIGH/LOW/FAIL) 배지는 1단계 화면에 없음(Provider 배지/SQL Shape
  카드도 그리드로 흡수되어 별도 렌더 없음, `:23018` 주석)

**2단계 — COUNT 사전검증**
- `#countCard`(`:3091-3131`) → `#countResult`에 HTML table: 원본테이블/목적테이블/
  검증항목/원본건수/대상건수/차이/처리시간/상태 — `grid_helpers.py:630-644`
  (`_mvCountColumns`), 값 계산 `:596-627`
- COUNT 불일치는 **비차단**(`tabler_renderer.py:7337` 주석, 안내문 "COUNT
  불일치는 검증을 중단하지 않습니다" `:3097-3099`) — 진입 즉시 자동 실 COUNT
  조회(`:3120-3125`)

**3단계 — 후보 컬럼 선정**
- 상단 9칸 컨텍스트 타일: 목적테이블/원본테이블/통계검증규모/목적그룹/원본그룹/
  목적집계/원본집계/실행시간/상태 — `grid_helpers.py:707-718`(`_mvCandidateColumns`)
- 후보 선택표(`#colSelectOut`, `renderColSelect:25146-`, `buildRow:25892-26050`,
  HTML 커스텀 표) — 컬럼별 역할배지(그룹기준/합계대상), 상태배지(기본추천/선택가능/
  수동선택필요/정책상제외/참고후보/보조확인후보/실행차단), 타입, 점수, 경고태그
  (NULL주의/타입확인/카디널리티주의). GROUP BY/SUM 각 최대 3개 체크박스
- **"상세보기" decision-trail 패널**(`:25708`, `_candDetailFieldsFor`) —
  컬럼별 타입/scale, 카디널리티(n_distinct+등급), NULL비율(%), 의미유형+근거,
  PK/UNIQUE 여부, 판정경로(STEP1_SQL_LINEAGE/STEP2_METADATA/STEP3_SEMANTIC/
  LAST_RESORT), 점수구성 상세, 사유 전문 — 5개 근거축(의미/타입/통계/키·제약/
  호환성) 기반 explainability. **개별검증에서 단건 상세 밀도가 가장 높은 지점.**

**4단계 — 통계검증 SQL 생성 + 실행**
- `stage4CtxCard`(`:3290-3312`) 8칸 타일 — `grid_helpers.py:904-936`
- `sqlCard`/`#sqlOut`(`:3315-3322`, `ui/js_sql_preview.py:31-198`) — 원본/목적
  SQL 코드박스 좌우 2열, 복사버튼, 통계전략 정보(전략명/신뢰도/규모등급/예상소요시간,
  `js_sql_preview.py:235-260+`), 조합검증 캐스케이드 안내
- 자동저장 체크박스(`:3344-3362`), 실행버튼 `#execBtn`→`runExecuteAsync()`(`:3363`)
- 실제 비교결과 그리드는 4단계가 아니라 5단계 소속(`:3492` 확인) — 4단계는
  SQL/전략/실행 제어만 담당

**5단계 — 통계검증 실행 결과(드릴다운)**
- `execResultCard`(`:3441-3529`) — stale 경고 배너 → 8칸 컨텍스트 타일(목적/원본
  테이블, 전체그룹, 불일치그룹, 재이관대상, 재이관PK, 실행시간, 상태,
  `grid_helpers.py:1249-1270`) → "[고정] 1~5단계 소요시간" 카드
  (`_mvStage5PipelineCardHtml:30845`) → 불일치 추출전략 정보 → 요약 → 불일치
  항목 축약
- **그룹별 비교표(`#executeOut`, `execute_result_renderer.py:903-`,
  `_renderRow:1323-1429`)** — 네이티브 HTML `<table>`: 트리형 그룹키 열,
  GROUP BY 축 목적/원본 값 병렬열, SUM·COUNT 값 목적/원본 병렬+Δ차이, 판정배지
  (일치/집계값 불일치/건수차이/의심(지문불일치)/원본만). diff/src_only 2분류만
  판정대상(CLAUDE.md 2026-09-01 정정과 일치, tgt_only는 legacy fallback으로만
  잔존, `:1355-1368`)
- **행 클릭 시 PK 레코드 드릴다운**(`_mvToggleRowAggDiff:20198-20463`,
  `/agg-diff/pk-records`) — PK+선정 GROUP BY 전체+선택 SUM 전체 컬럼을 원본/목적
  대조, 5건 페이징
- 별도 EXACT-DIFF 경로(`_mvToggleRowExactDiff:20232-20384`) — 누락/과잉/값불일치/
  중복키, 유형/MATCH KEY/불일치컬럼/원본값/목적값, 100건 페이지네이션, CSV 다운로드

**"테이블 중심"(단건 상세) 요소 요약**: 1~4단계는 모두 "SQL/테이블 1건"을
요약하는 8~9칸 div 타일이며 다중 행 그리드가 아니다. 단건 상세 밀도가 가장
높은 두 지점은 (1) 3단계 후보 "상세보기" decision-trail(컬럼 하나의 전체
판정근거)과 (2) 5단계 그룹 클릭 시 PK 레코드 드릴다운(레코드 하나의 전체
비교키 대조)이며, 둘 다 "하나의 대상을 깊게 들여다보는" 패널 구조다.

### 2. 일괄검증 1~5단계 정보구조 (파일:줄 근거)

단계 정의 단일 진실 소스: `BATCH_STEP_DEFS`(`tabler_renderer.py:11089-11095`),
표시 카드 매핑 `BATCH_STEP_SHOW`(`:11118-11126`), 전환 `showBatchStep()`
(`:11239-11330` 부근, `mv-pane-hidden` 클래스 토글 방식 — DOM 재배치 없는
"가상 패널"). 게이트 판정은 개별과 공용 `MvStageGate.computeTabView`
(`:11104-11196`)가 담당.

**1단계 — 그룹/업로드**
- 카드: `batchStage1CtxCard`, `batchGroupCard`, `batchUploadCard`,
  `batchRunListCard`, `batchUploadAuditCard`, `batchResultCard`, `batchDetailCard`
  (`:5404-5688`)
- 요약 타일그리드(`batchStage1CtxGrid`, `:5404-5407`, `_mvBatchStage1Refresh()`
  `:11050-11069`가 채움): 그룹명/업로드건수/검증대상건수/최근업로드시각/DB연결
- 검증 작업 그룹 드롭다운 + 새 그룹 생성 + **COUNT 불일치 처리 정책(사전설정)**
  드롭다운(`batchStage1CountMismatchPolicy`, `:5427-5436`, 3옵션) + **COUNT
  불일치 확인강제** 드롭다운(`batchStage1CountMismatchConfirmMode`, `:5445-5451`)
- 이관쿼리 업로드 안내(실제 업로드는 "데이터 준비 > 이관쿼리 업로드"로 이관,
  `:5527-5536`)
- 업로드 원본 row 전체 목록(`batchUploadAuditCard`, `services/upload_audit_store.py`
  기반, `:5557-5563`)
- 업로드 배치 이력(`batchRunListCard`), 업로드 결과 HTML table(순번/원본테이블/
  목적테이블/합계대상컬럼/그룹기준/MIN-MAX/제외/파싱/DB검증/중복상태,
  `:5606-5623`)

**2단계 — 검증대상/COUNT**
- 카드: `batchStage2CtxCard`, `batchLatestCard`(`:5690-5819`)
- "⚡ COUNT 사전검증 실행" 버튼(`batchRunCountOnlyBtn`, `runBatchCountOnly()`,
  `:5703-5705`), 건수/실행상태 배지, 검증대상 요약 배지(상태조합 다수)
- **강제 재실행 패널(Path A, "정식 경로")**(`:5738-5760`) — 검색창(`mvFrSearch`)
  +체크박스 다중선택+`mvFrRunForce` 버튼. `EXECUTION-REUSE-PART5` 주석 근거,
  `services/execution_reuse_lookup.py` 연동
- 상세 보기 토글(`batchLatestDetailWrap`) — 테이블 검색, 상태 필터(13종),
  페이지당 10/30/50/100건 게시판형(`batchLatestBoard`)

**3단계 — 후보 컬럼 선정**
- 카드: `batchStage3CtxCard`, `batchColumnEntryCard`,
  `batchColumnCandidateSnapshotCard`(`:5821-6038`)
- 진입 카드 — "후보 컬럼 선정 시작" 버튼, 진행 정책 드롭다운(1단계와 동일
  3옵션 재사용), 고위험 경고 배너, 진행/제외 건수 요약
- profile 수집 상태 — coverage, "미수집/실패 재수집", "연결정보 없음 대상 보기",
  "connection 보정" 버튼(`:5864-5883`)
- **후보 컬럼 선정 결과** — `_batchRenderSnapshotList()`(`ui/js_batch_display.py:143-333`)
  가 렌더: 필터 칩(`_BATCH_CAND_FILTERS`, `:136-141`, 전체/GROUP BY 없음/SUM
  없음/COUNT only/GB·SUM 모두없음/보조 SUM 후보 있음/profile 부족/고유값 수
  미수집/수동확인 필요/재선정 필요/오류), 항목별 접이식 헤더(목적테이블+상태
  아이콘+예상 최대 그룹수+`GROUP BY N·SUM N·대표:xxx·COUNT only` **요약
  문자열만**), 페이지네이션(`:195-332`)
- **개별 3단계의 decision-trail(컬럼별 카디널리티/NULL율/의미유형/판정경로/
  점수구성/사유)에 해당하는 정보는 배치 3단계 화면에 노출되지 않는다** — 접이식
  헤더를 펼쳐도 요약 문자열 수준에 그침(핵심 gap, 아래 3·5번 참고)

**4단계 — 전체 통계검증**
- 카드: `batchStage4CtxCard`, `batchFullValidationCard`, `batchFullResultCard`,
  `batchStatsPlanCard`, `batchValidCard`(`:5947-6162`)
- **이중 실행 경로가 화면에 명시적으로 구분 표시됨**:
  - 정식 경로 `batchFullValidationCard` — "▶ 전체 통계검증 실행"
    (`batchRunWrapperBtn`, `runWrapperValidation()`, `:5959-5961`), 강제 재실행
    패널 복제(id `2` 접미사, `:5982-6005`)
  - 보조 경로 `batchStatsPlanCard` 내 "▶ 안전 계획 실행(LOW/MEDIUM)"
    (`batchStatsExecuteBtn`), "보조 경로" 배지(`:6085-6088`, 근거 주석
    `BATCH-STATS-EXECUTE-SERVICE-VS-SHARED-FACADE-ARCHITECTURE-VERIFY_20260915`),
    강제재실행 체크박스(`batchForceRerunToggle`, `:6089-6095`)
- 통계검증계획 리스트 — `_batchRenderStatsPlanList()`(`js_batch_display.py:523-715`)
  가 계획별 카드 렌더: 상태아이콘, `plan_type_label`(COUNT만/COUNT+SUM/GROUP
  BY별), 테이블규모배지(HUGE/LARGE/MEDIUM/SMALL), 위험도배지(VERY_HIGH~LOW),
  GROUP BY/SUM 컬럼명, `effective_row_count`, "SQL 보기" 토글(Source/Target
  SQL preview), 위험도별 권장조치, 경고 목록
- 진행률(`batchExecProgressArea`, `_batchExecSetProgress()`,
  `ui/js_batch_dom.py:26-35`), 결과 board(`batchFullResultCard`), 실행 결과
  요약(`_batchExecShowStatusSummary()`, `js_batch_dom.py:36-98` — 상태/건수/
  불일치 자동저장 요약/run_id)

**5단계 — 결과 요약**
- 카드: `batchStage5CtxCard`, `batchWrapperResultCard`, `batchExecHistoryCard`,
  `batchSummaryCard`, `batchFailureSummaryCard`, `batchItemsCard`,
  `batchItemDetailWrap`, `batchViewerCard`(`:5566-6347`)
- 저장 결과 조회(정식 경로, `batchWrapperResultCard`) — 고급조회+CSV/XLSX
- 통계검증 실행 이력(보조 경로, `batchExecHistoryCard`, "보조 경로" 배지
  `:6168-6174`) — run별 상태배지, 요약카운트, `_batchRenderHistoryList()`
  (`js_batch_display.py:24-66`)
  - **재사용 배지**: `_batchRenderHistoryDetail()`(`js_batch_display.py:343-432`)
    의 오류/메모 열에 `is_reused_result===true`면 `🔁 재사용(일시)` 표시
    (`:397-404`, 근거 M380/M366, `services/execution_reuse_lookup.py`,
    `tests/test_batch_official_reuse_gate.py:160-188`)
  - "재사용된 항목만 보기" 토글 + "선택 항목만 강제 재실행"(따라잡기,
    `:6211-6223`, `BATCH-STAGE4-FORCE-REEXECUTE-AND-CATCHUP-IMPLEMENT 파트C`)
  - 결과 5분류(PASS/FAIL/BLOCKED/ERROR/UNSUPPORTED), 일치/차이(diff+src_only
    2분류, tgt_only 제외) 컬럼
- 검증 결과 조회(`batchViewerCard`), 검증 실행 요약(`batchSummaryCard`),
  실패 원인 요약(`batchFailureSummaryCard`, 0건/LLM비활성 시 숨김)
- **Item 목록**(`batchItemsCard`) — 순번/Query ID/목적테이블/상태/전체그룹/
  차이그룹/History ID HTML table, 클릭 시 상세(`batchItemDetailWrap`)
- **행 상세 모달**(`batchRowDetailModal`) — 목적/원본 테이블, 케이스명, 파싱/
  DB검증 결과, 원본·목적지 검증 상세, SUM/GROUP BY/MIN-MAX/제외 컬럼, 중복상태
  — **개별 5단계의 "그룹별 비교표+PK 레코드 드릴다운+EXACT-DIFF"에 해당하는
  단건 심층 드릴다운까지 내려가는지는 이번 조사에서 확인되지 않음(조사 갭,
  아래 5번 계획에 후속 확인 항목으로 반영)**

**그리드 렌더링 방식**: `_batchRenderStatsExecuteResults()`(`js_batch_display.py:433-522`)
만 Tabulator+HTML table fallback을 명시적으로 사용(`_mvRenderGrid` 존재 여부
분기). 그 외 대부분(후보목록/계획목록/이력목록/item목록/업로드결과)은 순수
HTML table(`mtbl` 클래스) 또는 DOM 직접 생성이며 Tabulator 미사용.

**종합 특성**: 배치는 "여러 테이블 상세 나열"이 아니라 "그룹 단위 요약 타일
(`batchStageNCtxGrid`) + 접이식 개별 항목 카드 + 필터/페이지네이션" 조합이다.
개별 테이블 단위 상세는 기본 접혀 있다가 펼치거나 필터를 적용해야 노출된다.
`ui/js_batch_overview.py`의 `mvBatchSummaryCounts`/`mvBatchRenderSummaryCard`
(row 상태 8종 집계)가 여러 단계에서 공유되는 표준 요약 카드 계약이다.

## 제안해결안

### 3. 단계별 대응관계표(단건→다건 치환 원칙)

| 단계 | 개별(단건·테이블중심) | 일괄(다건·배치중심) 현재 상태 | 치환 원칙 적용 판단 |
|---|---|---|---|
| 1 | SQL 1건 파싱요약 8칸 타일 | 업로드 N건 결과 HTML table(행당 1건 요약) + 원본row 전체목록 + 이력 | **이미 잘 대응됨.** 개별 1단계 타일의 8개 항목(목적/원본테이블·컬럼수·조건·조인·상태)이 배치 업로드결과 표의 컬럼과 대체로 일치. 추가 구조변경보다 세부 컬럼 매핑 재확인 수준으로 충분 |
| 2 | COUNT 1행 비교표 | 상태배지 요약 + 상세보기 목록(13종 상태필터) + 강제재실행 패널 | **이미 대응, 오히려 배치가 더 정교함**(정책 3종+확인강제). 단, 개별은 COUNT 불일치 비차단인데 배치는 정책 선택형이라 두 화면의 "기본 동작"이 서로 다름 — 정책 정합성은 별도 판단 필요(구조 문제 아님) |
| 3 | 컬럼별 후보표 + **컬럼 단위 decision-trail**(카디널리티/NULL율/의미유형/판정경로/점수구성/사유) | 테이블별 접이식 카드의 **요약 문자열만**(`GROUP BY N·SUM N·대표:xxx`) | **가장 큰 gap.** "단건→다건 치환 원칙"에 따르면 배치 3단계는 "테이블별 접힌 카드를 펼치면 그 테이블에 대한 개별 3단계 후보표+decision-trail 전체가 나와야" 하나, 현재는 그 안쪽 계층이 비어 있음. 우선순위가 가장 높은 구조 보완 지점 |
| 4 | SQL 프리뷰 2열 + 통계전략(신뢰도/규모등급/예상소요시간) | 계획별 카드(위험도배지+규모배지+SQL 보기 토글) + 이중경로(정식/보조) | **이미 대체로 대응됨.** 배치 계획카드의 위험도/규모 배지가 개별 통계전략 정보의 배치판 역할을 하고 있음. 다만 "SQL 보기" 토글을 펼쳤을 때 개별 4단계 수준의 신뢰도 문구까지 나오는지는 세부 확인 필요 |
| 5 | 그룹별 비교표(GROUP BY 축 병렬열+Δ차이+판정배지) + **PK 레코드 드릴다운** + EXACT-DIFF | Item 목록(요약 컬럼) + 행 상세 모달(파싱/DB검증/원본목적 상세 수준) | **두 번째로 큰 gap 후보.** 행 상세 모달이 개별 5단계의 "그룹별 비교표+PK 드릴다운" 수준까지 내려가는지 이번 조사에서 미확인 — 구현 착수 전 재조사 필요 |

### 4. 오늘 기존 작업과의 관계 (겹치는 부분은 재작업하지 않고 편입)

오늘(2026-09-17) 이미 다뤄진 배치 UX 작업 9건을 확인했다:

1. **BATCH-AUTORUN-FIXES-AND-UX-ALL-IN-ONE** — 레이스컨디션·정책불일치 수정 +
   COUNT 불일치 확인강제 + UX개선 8건(C-9 보류). `tabler_renderer.py` 3파트
   diff(총 468줄).
2. **BATCH-COUNT-MISMATCH-BLOCK-AND-MANUAL-PROCEED-DESIGN** — 개별의 "COUNT
   불일치 차단+수동진행" 정책을 배치에 포팅할지 조사·설계(코드 미수정).
3. **BATCH-STAGE1-COUNT-MISMATCH-POLICY-PRESELECT-IMPLEMENT** — 2번 설계(A안)
   구현. `batchStage1CountMismatchPolicy` select 신규(`:5402` 등). 커밋 b49abd10.
4. **BATCH-STAGE1-COUNT-MISMATCH-PROCEED-OR-HALT-CHECKBOX-DESIGN** — 체크박스안
   조사, A안(select) 채택 근거.
5. **BATCH-STAGE4-FORCE-REEXECUTE-AND-CATCHUP-DESIGN** — 4단계 강제재실행/
   따라잡기 설계, Path A/B 이중경로 확인.
6. **BATCH-STAGE4-FORCE-REEXECUTE-AND-CATCHUP-IMPLEMENT** — 5번 구현. 커밋
   337f9e3d/ea31f6ea/d0dd9788, 총 1001줄 diff.
7. **BATCH-UI-PARITY-WITH-SINGLE-TABS-AND-BUTTON-STATES-INVESTIGATE** — **축이
   다른 조사**: "탭 이동/버튼 활성화가 언제 멈추는가"(상태전이·게이팅)를 다룸.
   이번 신규 조사("각 단계가 무엇을 보여주는가")와 직접 중복 없음. 단, 그 보고서
   4절(완료/진행중/대기/차단 상태 표시 대조)은 "무엇을 보여주는가"와 일부 겹쳐
   재인용 가치 있음.
8. **M390-BATCH-OFFICIAL-PATH-REUSE-GATE-FIX** — 정식 경로 재사용 게이트
   `group_id` 미배선 결함 수정(백엔드, UI 변경 없음). 커밋 df04b82b(로컬만,
   push 미수행으로 보고서에 기록됨).
9. **EXECUTION-REUSE-WHICH-PRIOR-RESULT-SELECTION-CORRECTNESS-VERIFY** — 재사용
   대상 선택 기준 검증. 배치는 문제없음, 개별 경로에 동률 2차기준(UUID) 구조적
   결함 발견(발생확률 낮음, 별도 지침 권장).

**편입 판단**: 1~9번 모두 "정보 표시 내용 자체"의 구조 정리가 아니라 정책/게이팅/
재사용로직 쪽 작업이므로, 이번 조사의 대응관계표(3번)와 충돌하지 않는다. 다만
아래 항목은 이번 계획에 직접 편입한다.
- **P1(1단계)**: `BATCH-VALIDATION-SCREEN-SINGLE-UPLOAD-STRICT-DUPLICATE-BLOCK`
  (업로드 중복차단)이 여기 해당 — 1단계 정보구조 자체는 이미 개별과 잘 대응되어
  있으므로, 그 지침은 "구조 정리"가 아니라 "업로드 검증 강화"라는 별개 축이다.
  이번 지침 완료로 순서 충돌이 해소됐으니 이어서 착수 가능.
- **후속보류 3건**을 이번 계획의 위험요소로 반영: C-9(배치 후보 재선택 draft
  개념 부재), 4단계 위험도 확인 지점의 정책 하드코딩 잔존, 개별 재사용 동률
  2차기준(UUID) 구조적 결함.

### 5. 단계별 구현 계획 (실행하지 않음 — 사용자 확인 후 별도 지침으로 진행)

| 순번 | 대상 | 작업 내용 | 규모 | 위험도 | 근거 |
|---|---|---|---|---|---|
| P1 | 1단계 | 정보구조 자체는 대응 완료로 판단 — 구조변경 불필요. `BATCH-VALIDATION-SCREEN-SINGLE-UPLOAD-STRICT-DUPLICATE-BLOCK`(업로드 중복차단)만 별개로 진행 | 소 | 낮음 | 대응관계표 1행 |
| P2 | 2단계 | 정보구조 자체는 대응 완료(오히려 배치가 더 정교) — 구조변경 불필요. COUNT 비차단(개별) vs 정책선택형(배치) 정합성 여부만 사용자 판단 필요(구조 문제 아니므로 선택지 제시 후 확인) | 소 | 낮음 | 대응관계표 2행 |
| P3 | 5단계 재조사 | 배치 행 상세 모달이 개별 5단계 수준(그룹별 비교표+PK드릴다운+EXACT-DIFF)까지 내려가는지 코드 재확인. **구현 아님, 순수 조사** — 대용량 배치에서 N개 테이블 각각 PK드릴다운을 지원할 경우 성능/저장 부담이 커질 수 있어 설계 전 현황 파악이 먼저 필요 | 소 | 낮음(조사만) | 대응관계표 5행 |
| P4 | 3단계 | **핵심 gap 보완**: 배치 후보 스냅샷 접이식 카드를 펼쳤을 때, 개별 3단계 후보표+decision-trail(카디널리티/NULL율/의미유형/판정경로/점수구성/사유)에 상당하는 정보를 테이블별로 노출. UI 카드 구조 변경 + 컬럼 단위 상세 데이터 추가 조회 필요 | 중~대 | 중간(N테이블×M컬럼 렌더 시 성능/응답크기 고려 필요, heuristic 노출 방식 설계 필요) | 대응관계표 3행 |
| P5 | 4단계 보완 | 배치 계획카드 "SQL 보기" 토글 펼침 시 개별 4단계 수준 신뢰도/전략 문구 노출 여부 확인 후 필요시 보강 | 소 | 낮음 | 대응관계표 4행 |
| P6 | 5단계 구현 | P3 재조사 결과에 따라 필요시 행 상세 모달에 그룹별 비교표+드릴다운 보강 | P3 결과에 따라 중~대 | P3 결과에 따라 중~높음(대용량 배치 성능 우려) | 대응관계표 5행 |

**권장 순서**: P1(즉시 진행 가능, 이미 대기 중) → P3(경량 재조사 우선, P6 설계의
전제조건) → P4(핵심 gap, 설계안 먼저 제시 후 사용자 승인) → P5(경량 보완) →
P6(P3 결과·P4 설계 패턴 재사용해 진행) → P2는 구조변경이 아니라 정책 방향
확인만 필요하므로 아무 때나 짧게 처리 가능.

각 구현형 지침(P1, P4, P6, P5)은 착수 전 CLAUDE.md "기능/구조 변경 요청 응답
형식"(요청요약/긍정적효과/구조적문제점/운영상위험/heuristic·scoring·
explainability영향/권장대응책/지금구현여부)을 먼저 제시하고 사용자 확인 후
진행해야 한다.

## 기대효과

- 개별↔배치 화면의 "정보 표시" 차이가 어디서 오는지(구조적 누락 vs 의도된
  요약화) 구분되어, 향후 지침에서 불필요한 전면 재설계 없이 표적화된 보완만
  가능해짐
- 오늘 진행된 9건의 배치 UX 작업과 겹치지 않는 것을 확인해 재작업 낭비 방지
- P1(업로드 중복차단) 착수 재개 가능, P3~P6 우선순위와 위험도가 명확해져
  사용자가 다음 지침을 순서대로 발행하기 용이

## 비고 — 조사 중 발견된 CLAUDE.md/README 문서-코드 불일치(수정하지 않음, 보고만)

- CLAUDE.md "Web UI 4단계 화면 흐름"은 실제로 5단계(개별 `SINGLE_STEP_CARDS`,
  배치 `BATCH_STEP_DEFS` 기준)이며, "4단계=SQL생성+실행+결과표시"가 코드에서는
  4/5단계로 분리되어 있음
- CLAUDE.md는 2단계 COUNT 불일치 시 "기본 차단, 수동 진행 버튼"을 명시하나,
  개별검증 코드는 현재 비차단으로 동작(`tabler_renderer.py:7337`, `:3097-3099`)
  — 배치는 오늘 정책선택형으로 구현됨(위 4번 참고)
- CLAUDE.md는 Tabulator가 "개별검증 탭별 Summary Grid(쿼리검토/COUNT/후보추천/
  통계검증/결과확인)"에 쓰인다고 명시하나, 조사 결과 개별 1~5단계 요약 타일/표는
  모두 순수 HTML(div 타일 또는 `<table>`)이며 Tabulator 미사용으로 확인됨.
  배치도 `_batchRenderStatsExecuteResults()` 1곳만 Tabulator+fallback 사용
- 위 3건은 README.md가 아닌 CLAUDE.md 서술이므로 "README.md 임의수정 금지"
  규칙 대상은 아니나, CLAUDE.md 자체도 사용자 확인 없이 임의 수정하지 않았음
  (이번 지침은 조사·계획 전용). 문서 정정 필요 여부는 별도 판단 요청

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 (코드 수정 없이 순수 조사·
  계획 수립만 수행)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(코드 미변경이므로
  27/29/30번 증적 규칙 대상 자체가 아님)

작업명 : BATCH-UI-STRUCTURE-PARITY-WITH-SINGLE-INVESTIGATE-AND-PLAN
✅ 작업 완료 - 개별/일괄검증 화면 정보구조 조사 및 단계별 대응관계·구현계획 수립(조사+계획 전용, 코드 미수정)
```
