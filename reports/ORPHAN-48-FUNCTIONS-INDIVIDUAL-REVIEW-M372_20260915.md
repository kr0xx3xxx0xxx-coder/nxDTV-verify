```text
작업명 : ORPHAN-48-FUNCTIONS-INDIVIDUAL-REVIEW-M372
✅ 작업 완료 - 완전 고아 함수/클래스 48건 전부 개별 판단(삭제가능 34건 실삭제 / 삭제보류 14건 / 오판정 0건), 커밋 c2540431

## 목적
CODEBASE-WIDE-DEAD-CODE-AND-ISSUES-AUDIT(2026-09-15)에서 확인된 "완전 고아"
(같은 파일 내에서도 참조가 전혀 없는 함수/클래스 정의) 48건의 삭제 여부를
건별로 개별 판단한다. SILENT-EXCEPTION-12B-16C-FOLLOWUP-M373와 동일한 방법론
(파일 그룹으로 나눠 병렬 서브에이전트가 각자 실제 코드 문맥을 확인한 뒤
개별 판단, 일괄 삭제 절대 금지)을 그대로 재사용했다.

## 현상
- 원 감사는 "식별자가 파일 전체 텍스트에 등장하는지"를 보는 토큰 스캔이라
  실제 AST 호출 그래프 분석이 아니며, 감사 시점 이후 같은 날 다른 지침들
  (merge-walk 엔진 완전삭제 M364, SQL 해시 통합, chunk_pushdown.py 삭제
  M377 등)로 호출관계가 이미 여러 번 바뀐 상태였다. 따라서 48건 전부를
  최신 코드 기준으로 처음부터 재검증해야 했다.
- 48건을 파일 인접성 기준 6개 그룹(그룹1: parser/analyzer/routes 7건,
  그룹2: batch 관련 services 9건, 그룹3: services/dialects 7건, 그룹4:
  diagnosis/exact_diff 등 8건, 그룹5: profile_snapshot 등 8건, 그룹6:
  validation/upload_audit 등 9건)으로 나눠 병렬 서브에이전트(Opus)에게
  배정, 각자 grep 재확인 → 문서/주석 힌트 확인 → 3분류 판정 → 삭제가능
  건만 실제 삭제 → 재grep(0건 확인) → py_compile/import 확인 → 관련
  pytest 실행까지 수행하게 했다.

## 실제조치

### 48건 전체 분류표 (원 감사 파일:줄 기준, 분류 | 근거요약 | 조치)

| # | 파일:줄(감사 당시) | 이름 | 분류 | 조치 |
|---|---|---|---|---|
| 1 | analyzer/base_column_analyzer.py:148 | AnalyzeResult.all_columns | 삭제가능 | 실제삭제됨 |
| 2 | config/db_paths.py:55 | legacy_data_dir | 삭제보류 | 보류(미실행 이관단계 전용, 향후 사용 명시) |
| 3 | parser/sqlglot_parser.py:73 | SqlglotParser.extract_select_from_insert | 삭제가능 | 실제삭제됨 |
| 4 | parser/sqlglot_safe_parse.py:76 | clear_timeout_cache | 삭제보류 | 보류(진단 escape hatch, docstring 명시) |
| 5 | routes/group_upload_view_route.py:671 | CollectJobStart(class) | 삭제가능 | 실제삭제됨 |
| 6 | routes/group_upload_view_route.py:678 | CollectJobFinish(class) | 삭제가능 | 실제삭제됨 |
| 7 | routes/scope_guard_response.py:46 | guard_batch | 삭제보류 | 보류(대칭 helper 5형제 중 하나, 나머지 4개 실사용) |
| 8 | services/batch/resource_budget.py:74 | ResourceBudget.can_launch | 삭제가능 | 실제삭제됨 |
| 9 | services/batch_auto_save_store.py:350 | run_within_time_cap | 삭제보류 | 보류(docstring에 향후 배선 계획 명시) |
| 10 | services/batch_delta_rerun_service.py:59 | count_execution_items | 삭제가능 | 실제삭제됨 |
| 11 | services/batch_execution_plan_service.py:94 | summarize_execution_plan | 삭제가능 | 실제삭제됨 |
| 12 | services/batch_execution_plan_service.py:114 | get_execution_plan_preview_for_group | 삭제가능 | 실제삭제됨 |
| 13 | services/batch_file_service.py:321 | get_file_meta | 삭제가능 | 실제삭제됨 |
| 14 | services/batch_group_service.py:733 | restore_group_status_after_restore | 삭제가능 | 실제삭제됨 |
| 15 | services/batch_group_service.py:1227 | get_group_baseline_filename | 삭제가능 | 실제삭제됨 |
| 16 | services/batch_wrapper_result_store.py:463 | count_wrapper_results | 삭제가능 | 실제삭제됨 |
| 17 | services/candidate_scoring_runner.py:141 | ScoredColumn(class) | 삭제가능 | 실제삭제됨 |
| 18 | services/db_query_service.py:432 | conn_scope_stats | 삭제가능 | 실제삭제됨 |
| 19 | services/diagnosis/contracts.py:74 | DiagnosisBudget.node_budget_left | 삭제가능 | 실제삭제됨 |
| 20 | services/diagnosis/cost_accuracy.py:136 | next_calibration_hint | 삭제가능 | 실제삭제됨 |
| 21 | services/dialects/mssql/runner_capabilities.py:15 | MSSQLRunnerCapabilities(class) | 삭제보류 | 보류(base.py·DIALECT_POLICY.md §5에 "향후 확장용 placeholder" 명시) |
| 22 | services/dialects/mssql_dialect.py:21 | resolve_mssql_dialect | 삭제보류 | 보류(→ M385로 구조적 정리 별도 등록) |
| 23 | services/dialects/mysql/runner_capabilities.py:15 | MySQLRunnerCapabilities(class) | 삭제보류 | 보류(21과 동일 근거) |
| 24 | services/dialects/mysql_dialect.py:21 | resolve_mysql_dialect | 삭제보류 | 보류(→ M385) |
| 25 | services/dialects/oracle/runner_capabilities.py:16 | OracleRunnerCapabilities(class) | 삭제보류 | 보류(21과 동일 근거) |
| 26 | services/dialects/oracle_dialect.py:21 | resolve_oracle_dialect | 삭제보류 | 보류(→ M385) |
| 27 | services/dialects/postgresql/runner_capabilities.py:16 | PostgreSQLRunnerCapabilities(class) | 삭제보류 | 보류(21과 동일 근거, 4방언 대칭 확인) |
| 28 | services/exact_diff/reimport_job.py:373 | clear_recover | 삭제가능 | 실제삭제됨 |
| 29 | services/exact_diff/reimport_job.py:672 | status_dict | 삭제가능 | 실제삭제됨 |
| 30 | services/exact_diff/sampling_preflight.py:268 | is_numeric_pk_type | 삭제가능 | 실제삭제됨 |
| 31 | services/exact_diff/stream_merge.py:244 | outcome_counts | 삭제가능 | 실제삭제됨 |
| 32 | services/execution_profile_service.py:317 | classify_execution_profile | 삭제가능 | 실제삭제됨 |
| 33 | services/exhaustive_candidates/store.py:123 | record_to_dict | 삭제가능 | 실제삭제됨 |
| 34 | services/mysql_metadata_provider.py:30 | is_mysql_family | 삭제가능 | 실제삭제됨 |
| 35 | services/owner_connection_enforcer.py:143 | EvidenceOwner(class) | 삭제가능 | 실제삭제됨 |
| 36 | services/profile_snapshot_service.py:745 | mark_profile_snapshot_stale | 삭제가능 | 실제삭제됨 |
| 37 | services/profile_snapshot_service.py:775 | build_profile_snapshot_from_pg_stats | 삭제가능 | 실제삭제됨 |
| 38 | services/profile_snapshot_service.py:833 | get_snapshot_summary | 삭제가능 | 실제삭제됨 |
| 39 | services/query_column_metadata.py:628 | assert_metadata_usable_for_candidate_generation | 삭제가능 | 실제삭제됨 |
| 40 | services/security_utils.py:32 | has_sensitive_keys | 삭제가능 | 실제삭제됨 |
| 41 | services/sql_validation_service.py:824 | validate_insert_select_sql | 삭제가능 | 실제삭제됨 |
| 42 | services/strategy/strategy_models.py:186 | resource_weight_for | 삭제보류 | 보류(설계문서 Phase 4-A 스펙, 폴백 계약 주석 명시) |
| 43 | services/ui_settings_service.py:58 | ensure_ui_settings_table | 삭제보류 | 보류(2일 전 신설 모듈, 배선 전 단계) |
| 44 | services/upload_audit_store.py:284 | detect_in_batch_duplicate_targets | 삭제가능 | 실제삭제됨 |
| 45 | services/upload_audit_store.py:404 | group_conflict_target_keys | 삭제가능 | 실제삭제됨 |
| 46 | services/validation_policy_service.py:596 | ensure_validation_policy_table | 삭제보류 | 보류(43과 동일 관례 짝) |
| 47 | services/validation_result_store.py:664 | get_run_by_execution_id | 삭제가능 | 실제삭제됨 |
| 48 | services/validation_run/idempotency_store.py:332 | get_row_on_path | 삭제가능 | 실제삭제됨 |

**집계: 삭제가능(실제삭제) 34건 / 삭제보류 14건 / 오판정 0건 / 합계 48건.**

### 그룹별 핵심 근거 요약
- **그룹1(7건, 4삭제/3보류)**: 5·6번은 같은 파일 주석이 "해당 API는 제거됨
  (SIMPLIFY-METADATA-COLLECTION §9)"이라고 명시한 제거된 엔드포인트의
  잔존 요청 모델이었다. 2·4·7번은 각각 "미실행 이관 단계 전용", "진단
  escape hatch(테스트/진단용 명시)", "대칭 helper 5형제 중 나머지 4개
  실사용"으로 확인돼 보류.
- **그룹2(9건, 8삭제/1보류)**: 9번(run_within_time_cap)만 docstring에
  "게이트/prepare 모듈 구현되면 이 안에서 호출하도록 배선" 계획이 명시돼
  보류. 나머지 8건은 전부 동일 기능을 수행하는 다른 함수로 대체돼 있거나
  (예: get_execution_plan_preview_for_batch만 실사용, group판은 미배선)
  단순 집계 래퍼가 호출부 없이 방치된 경우였다.
- **그룹3(7건, 0삭제/7보류)**: `resolve_{oracle,mysql,mssql}_dialect`와
  `RunnerCapabilities` 4개 클래스 전부 문서화된 "향후 확장용/호환 shim"
  근거로 보류. 조사 중 postgresql까지 포함해 4개 방언이 완전 대칭으로
  고아 상태(감사의 postgresql 제외는 docstring 텍스트 매치로 인한 오판)
  이고, `{d}_dialect.py` 4개 shim 파일 자체가 통째로 미사용임을 확인해
  별도 구조적 정리 항목(M385)으로 등록했다(함수 단위 삭제 시 postgresql만
  남는 비대칭이 생기므로 이번 범위에서는 보류).
- **그룹4(8건, 8삭제/0보류)**: exact_diff 3개 파일은 M364/M377 이후에도
  모듈 자체(stream_merge.merge_compare, reimport_job.begin_recover 등)는
  살아있고, 문제의 4개 심볼만 국소적으로 고아임을 개별 확인. 오판정 없음.
- **그룹5(8건, 8삭제/0보류)**: 감사 스냅샷이 전부 정확했다. 특히
  build_profile_snapshot_from_pg_stats는 `services/metadata_provider.py::
  from_pg_stats`로 대체된 중복 구현이었음을 확인(삭제로 중복 해소).
- **그룹6(9건, 6삭제/3보류)**: 42·43·46번은 전부 "미배선 공개 API"
  성격(설계문서 스펙 대기 / 2일 전 신설 모듈 / 형제 관례)으로 보류.
  41번 삭제로 2차 고아가 된 `validate_parsed_sql()`(~350줄)은 이번
  지정 48건 범위를 벗어나 손대지 않고 M386으로 별도 등록.

### 코드 저장소 변경 (커밋 c2540431, 27개 파일, 579줄 삭제/3줄 추가)
```
git show --stat c2540431
 analyzer/base_column_analyzer.py             |  10 --
 parser/sqlglot_parser.py                     |  50 ----------
 routes/group_upload_view_route.py            |  16 ---
 services/batch/resource_budget.py            |   5 -
 services/batch_delta_rerun_service.py        |   5 -
 services/batch_execution_plan_service.py     |  38 --------
 services/batch_file_service.py               |  18 ----
 services/batch_group_service.py              |  32 ------
 services/batch_wrapper_result_store.py       |   5 -
 services/candidate_scoring_runner.py         |  37 -------
 services/db_query_service.py                 |   6 --
 services/diagnosis/contracts.py              |   3 -
 services/diagnosis/cost_accuracy.py          |  17 ----
 services/exact_diff/reimport_job.py          |  12 ---
 services/exact_diff/sampling_preflight.py    |  13 ---
 services/exact_diff/stream_merge.py          |  11 ---
 services/execution_profile_service.py        |   5 -
 services/exhaustive_candidates/store.py      |   6 +-
 services/mysql_metadata_provider.py          |   5 -
 services/owner_connection_enforcer.py        |  12 +--
 services/profile_snapshot_service.py         |  90 -----------------
 services/query_column_metadata.py            |   8 --
 services/security_utils.py                   |   5 -
 services/sql_validation_service.py           | 116 ----------------------
 services/upload_audit_store.py               |  14 ---
 services/validation_result_store.py          |  17 +---
 services/validation_run/idempotency_store.py |  26 -----
 27 files changed, 3 insertions(+), 579 deletions(-)
```
전부 함수/클래스 정의 삭제 + 그로 인해 유일 사용처를 잃은 import 정리
(예: `services/exhaustive_candidates/store.py`의 `asdict`,
`services/owner_connection_enforcer.py`의 `field`,
`services/batch_execution_plan_service.py`의 `logging`/`_log`)뿐이며,
그 외 다른 코드는 손대지 않았다(6개 서브에이전트가 각자 `git diff`로
순수 삭제만 확인). 전체 diff는 `git show c2540431`로 재확인 가능
(코드 저장소 X:\xDataNexPro\nxDTV, main 브랜치).

같은 세션 동안 무관한 다른 작업(routes/batch_route.py의 별도 미커밋
변경, `services/batch_stats_execute_service.py`의 다른 세션 미커밋
작업)은 커밋에서 명시적으로 제외했다(git status로 확인 후 27개 파일만
개별 staging).

※ 커밋 메시지 첫 줄 요약에 "22건"이라고 잘못 적었다(실제 34건, 본문
목록에는 34건 전부 정확히 나열돼 있음 — 요약 숫자만 집계 오기). 이미
푸시된 커밋이라 히스토리를 되돌려쓰지 않고, 이 보고서에 정정 사실을
명시한다.

### BACKLOG.md 갱신(verify 저장소)
- M372: 아이디어(미착수) → **✅ 해결 완료**로 갱신(48건 전부 개별 판단
  완료했으므로 부분 해결이 아님 — 14건 보류는 "미처리"가 아니라 근거
  있는 명시적 유지 결정).
- 신규 등록: **M385**(dialect shim 4파일 구조적 삭제 필요, 아이디어/
  미착수), **M386**(M372로 파생된 2차 고아 3건 — validate_parsed_sql
  ~350줄 + 미사용 상수 2건, 아이디어/미착수).

## 검증 결과
- **전체 회귀**: `samples/test_virtual_cases.py` 8/8 통과,
  `samples/test_complex_cases.py` 5/5 통과(수정 후 1회 실행, 전부 통과).
- **영향받는 모듈별 pytest**(6개 그룹 합산, PYTHONIOENCODING=utf-8):
  - 그룹1: `test_project_scope_guard.py` + `test_sqlglot_pre_parse_block.py` 30 passed/37 subtests
  - 그룹2: 관련 4개 테스트 파일 146 passed
  - 그룹4: `test_candidate_scoring_runner.py` 등 95 passed
  - 그룹5: 관련 6개 테스트 파일 124 passed(+ 실패 8건은 `tests/conftest.py`의
    `[PROD-DB-WRITE-BLOCKED]` 가드로 인한 기존 환경 이슈, 삭제 대상 함수와
    무관함을 grep 0건으로 확인)
  - 그룹6: `test_validation_result_store.py` 8/8, 그 외 실패 건은 HEAD
    원본으로 되돌린 상태에서도 동일하게 재현되는 기존 실패임을 직접 대조
    (임시 교체 후 원복, `rm` 미사용)로 확인
  - 그룹3: 코드 변경 없음(전건 보류) — 4방언 dialect/capabilities 모듈
    import 및 `get_db_dialect(d).capabilities().runner` 4방언 정상
    반환만 건전성 확인
  - 합계: **400건 이상 pass**, 실패는 전부 본 변경과 무관한 기존 이슈임을
    HEAD 대조로 확인
- **삭제 후 grep 재확인**: 34개 식별자 전체 저장소 참조 0건(scratchpad
  제외). 아카이브 문서 2곳(`docs/archive/...md`,
  `SOURCE_STRUCTURE_REVIEW_FOR_*.md`)에만 과거 스냅샷 서술로 이름이
  남아있으나 작업 범위 밖(문서 수정 아님)이라 손대지 않음.
- **py_compile/import**: 수정된 27개 파일 전부 통과.
- **코드 저장소 push**: 커밋 c2540431, `git push origin main` → 이미
  최신(post-commit 훅으로 자동 반영) 확인.

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 (services/routes/
  parser/analyzer 계층의 순수 백엔드 함수·클래스 정의 삭제만 수행 — 버튼/
  체크박스/문구/배지 등 화면 요소는 전혀 건드리지 않음. 삭제된
  CollectJobStart/CollectJobFinish는 이미 제거된 API의 요청 body 모델일
  뿐 화면과 무관)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(UI 변경
  자체가 없는 순수 백엔드 삭제 지침이므로 27번 규칙 대상 아님)

※ 30번 규칙(소스 diff 증적): 위 "코드 저장소 변경" 섹션에 `git show
--stat` 결과와 커밋 해시(c2540431)를 포함했으며, 전체 상세 diff는
`git show c2540431`로 재확인 가능.

## 기대효과
- 죽은 코드 34건(합계 579줄) 제거로 실제 호출 그래프와 코드베이스가
  더 가까워지고, 향후 유지보수 시 "이 함수가 정말 쓰이는지" 재확인하는
  비용이 그만큼 줄어든다.
- 14건 보류는 전부 근거가 코드/커밋/보고서에 남아, 다음 세션이 같은
  조사를 반복하지 않도록 했다.
- 그룹3·그룹6 조사 과정에서 이번 범위 밖의 구조적 문제(dialect shim
  파일 전체 사장, 2차 고아 ~350줄)를 추가로 발견해 M385/M386으로 등록,
  후속 지침의 출발점을 명확히 했다.

작업명 : ORPHAN-48-FUNCTIONS-INDIVIDUAL-REVIEW-M372
✅ 작업 완료 - 완전 고아 함수/클래스 48건 전부 개별 판단(삭제가능 34건 실삭제 / 삭제보류 14건 / 오판정 0건), 커밋 c2540431
```
