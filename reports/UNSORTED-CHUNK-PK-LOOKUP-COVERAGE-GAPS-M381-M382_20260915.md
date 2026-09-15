```text
작업명 : UNSORTED-CHUNK-PK-LOOKUP-COVERAGE-GAPS-M381-M382
✅ 작업 완료 - progress_cb/basis 회귀 테스트 신설(M381) + tracemalloc 계측 배선(M382) 완료

**목적**
MERGE-WALK-PK-RANGE-CHUNK-PERMANENT-REMOVAL(M364)로 불일치 레코드 추출이
run_unsorted_chunk_pk_lookup_compare(UNSORTED_CHUNK_PK_LOOKUP) 엔진 하나로
단일화된 뒤 확인된 커버리지 공백 2건을 메운다.
- M381: 이 엔진의 progress_cb 단조증가·basis(기준값) 초기화 자체를 검증하는
  회귀 테스트 부재.
- M382: 이 엔진에 tracemalloc 메모리 계측이 애초에 배선돼 있지 않음
  (DIRECT_STREAM 엔진만 계측 대상이었음).

**현상**
- M381: 과거 merge-walk 전용 progress 회귀 테스트
  (tests/test_d7_6_progress_earlystop.py::test_progress_monotonic_nonzero)는
  M364 삭제 대상에 포함돼 함께 제거됐고, 생존 엔진(UNSORTED_CHUNK_PK_LOOKUP)
  기준 동등 테스트가 작성되지 않은 채 남아 있었다.
- M382: tests/test_tracemalloc_toggle_wiring.py 상단 코멘트에 "생존
  UNSORTED_CHUNK_PK_LOOKUP 엔진은 애초에 tracemalloc 계측이 배선돼 있지
  않아 대체 기준선을 추가하지 않았다(커버리지 공백, 후속 지침 권고)"라고
  명시돼 있었다 — services/exact_diff/pk_range_chunk.py 확인 결과 실제로
  tracemalloc import/start/stop 코드가 전혀 없었다(DIRECT_STREAM 엔진 —
  agg_contribution.prepare_reimport_pk_index_stream — 에만 존재).

**실제조치**

파트A(M381) — tests/test_unsorted_chunk_progress_basis.py 신설(3개 테스트):
1. test_progress_cb_monotonic_and_matches_final_counts — 5배치 × 4행을
   흘려 progress_cb 가 배치마다 정확히 1회(force=True — rate limit
   미적용) 호출되고, processed_src/available 이 단조증가하며, 마지막
   호출값이 최종 반환값(out["reimport"], out["metrics"]["src_rows"])과
   정확히 일치함을 검증. target_only 필드가 항상 UNKNOWN(None/False)인
   계약도 함께 확인.
2. test_progress_cb_not_called_when_omitted — progress_cb=None(기본)이어도
   정상 완주.
3. test_basis_initialized_with_source_target_count_and_engine_fields —
   store.set_basis 로 초기화되는 basis 가 source_count/target_count/
   gb_cols/sum_labels/strategy/target_only_measured/target_only_reason을
   호출측 인자·계약 그대로 담는지 검증.

파트B(M382) — DIRECT_STREAM 엔진의 TRACEMALLOC-ALWAYS-ON-COST-REDUCTION
계측 토글 패턴(services/exact_diff/agg_contribution.py:969~1299,
is_memory_profiling_enabled() 기반 start/stop + metrics.peak_python_mb)을
확인 후 run_unsorted_chunk_pk_lookup_compare 에 동일하게 적용 가능함을
확인하고 배선:
- 함수 진입 시 `_mem_prof = is_memory_profiling_enabled()`; ON 이면
  tracemalloc.start().
- 이 함수는 DIRECT_STREAM 처럼 CANCELLED/예외용 반환 분기가 따로 나뉘어
  있지 않고 fetch_source_closer 정리용 finally 블록 하나뿐이라, 그
  finally 에 stop/peek(tracemalloc.get_traced_memory() + stop())를 함께
  묻어 정상 완주·cancel_check break·예외 전파 세 경로를 중복 삽입 없이
  하나의 지점에서 커버(기존 구조 임의 리팩토링 금지 원칙 준수).
- metrics 딕셔너리에 peak_python_mb 필드 추가(OFF 면 None — DIRECT_STREAM
  과 동일 필드명·계약).
- tests/test_tracemalloc_toggle_wiring.py 의 기존 3개 테스트(OFF 미시작 /
  ON 계측 / 토글 무관 판정 동일)를 `_run_lookup()` 헬퍼로 이 엔진까지
  확장하고, 상단 코멘트의 "커버리지 공백" 서술을 해소 완료로 갱신.
- 억지 적용 사례 없음 — 구조가 근본적으로 달라 적용 불가능한 부분은
  발견되지 않았다(finally 재사용만으로 충분).

**검증 결과**
- 파트A 탐지력 실측: metrics.py `progress_cb({"processed_src": m["src_rows"], ...})`
  의 processed_src 를 일부러 `0`으로 고정 → 신규 테스트
  test_progress_cb_monotonic_and_matches_final_counts 즉시 실패
  (`assert [0, 0, 0, 0, 0] == [4, 8, 12, 16, 20]`) 확인 후 원복.
  basis 쪽도 `"source_count": source_count` → `"source_count": None` 으로
  일부러 깨뜨려 test_basis_initialized... 가 즉시 실패
  (`assert None == 1234`) 확인 후 원복.
- 파트B 실측: test_on_measures_again 에서 계측 ON 시
  out_l["metrics"]["peak_python_mb"] > 0 확인, OFF 시 None 확인,
  tracemalloc.is_tracing() 이 계측 종료 후 False 로 복귀함도 확인.
- 신규/확장 테스트 24개 전체 통과:
  tests/test_unsorted_chunk_progress_basis.py(3) +
  tests/test_tracemalloc_toggle_wiring.py(4) +
  tests/test_unsorted_chunk_pk_lookup.py(9) +
  tests/test_unsorted_chunk_multi_tag_pk_lookup.py(7) [멀티태그 변형은
  이번 변경 대상이 아니므로 무회귀만 확인] = 24 passed.
- 전체 회귀: samples/test_virtual_cases.py 8/8 통과,
  samples/test_complex_cases.py 5/5 통과(무회귀).

git diff (services/exact_diff/pk_range_chunk.py):
--------------------------------------------------
diff --git a/services/exact_diff/pk_range_chunk.py b/services/exact_diff/pk_range_chunk.py
index e575fd63..65a95e0d 100644
--- a/services/exact_diff/pk_range_chunk.py
+++ b/services/exact_diff/pk_range_chunk.py
@@ -19,6 +19,7 @@ from __future__ import annotations
 import time

 from services.exact_diff.agg_contribution import REIMPORT_KIND
+from services.exact_diff.execution_settings import is_memory_profiling_enabled as _es_memory_profiling_enabled
 from services.exact_diff.store import default_exact_diff_store


@@ -192,7 +193,18 @@ def run_unsorted_chunk_pk_lookup_compare(*, fetch_source_batches, fetch_source_c
     스캔 순서상 뒤쪽에 몰려 있으면 mismatch_limit_count 조건만으로는 대규모 원본에서 스캔이 오래도록
     끝나지 않을 수 있어, 이 상한 도달 시 mismatch_limit_count 도달과 동일하게 즉시 중단한다(화면 표시도
     통일 — 사용자 확정 방침)."""
+    import tracemalloc
     t0 = time.perf_counter()
+    # [UNSORTED-CHUNK-PK-LOOKUP-COVERAGE-GAPS-M381-M382, 파트B(M382)] DIRECT_STREAM 엔진
+    # (agg_contribution.prepare_reimport_pk_index_stream)이 이미 쓰는 TRACEMALLOC-ALWAYS-ON-COST-REDUCTION
+    # 계측 토글 패턴을 이 엔진에도 동일하게 적용한다 — 기본 OFF, ON 일 때만 start/stop. 이 함수는 DIRECT_STREAM
+    # 처럼 CANCELLED/예외 반환 분기가 별도로 나뉘어 있지 않고 finally 블록 하나(fetch_source_closer 정리)뿐이라,
+    # 그 finally 에 stop/peek 를 같이 묻어 정상 완주·cancel_check break·예외 전파 세 경로 모두를 하나의
+    # 지점에서 커버한다(경로별 중복 삽입 없이 기존 finally 재사용 — 임의 리팩토링 금지 원칙 준수).
+    _mem_prof = _es_memory_profiling_enabled()
+    if _mem_prof:
+        tracemalloc.start()
+    _peak = None
     # [REPLACE-MERGEWALK-WITH-UNSORTED-ENGINE-UNCONDITIONAL] 과거엔 cap 없는 호출(mismatch_limit_count
     # None/0)을 HOLD 로 거부하고 "기존 merge-walk 경로를 사용하십시오"라고 안내했다 — merge-walk 가
     # 조건부 대안으로 남아 있던 설계 전제였다. 이 지침으로 merge-walk(DIRECT_STREAM_COMPARE/
@@ -300,6 +312,9 @@ def run_unsorted_chunk_pk_lookup_compare(*, fetch_source_batches, fetch_source_c
             fetch_source_closer()
         except Exception:  # noqa: BLE001
             pass
+        if _mem_prof:
+            _cur, _peak = tracemalloc.get_traced_memory()
+            tracemalloc.stop()
     _flush()
     _stopped = limit_hit or scan_rows_capped
     if _stopped:
@@ -327,7 +342,10 @@ def run_unsorted_chunk_pk_lookup_compare(*, fetch_source_batches, fetch_source_c
                "src_rows": m["src_rows"], "src_batches": m["src_batches"],
                "tgt_query_count": m["tgt_query_count"], "strategy": "UNSORTED_CHUNK_PK_LOOKUP",
                "total_wall_sec": round(time.perf_counter() - t0, 2),
-               "gate_evidence": gate_evidence or {}, "max_scan_rows": _max_scan_rows}
+               "gate_evidence": gate_evidence or {}, "max_scan_rows": _max_scan_rows,
+               # [UNSORTED-CHUNK-PK-LOOKUP-COVERAGE-GAPS-M381-M382] 계측 OFF(기본)면 None(미측정) — DIRECT_STREAM
+               # 엔진과 동일 필드명·계약(agg_contribution.py m["peak_python_mb"]).
+               "peak_python_mb": (round(_peak / (1024 * 1024), 2) if _peak is not None else None)}
 limit_evidence = {"mismatch_limit_count": limit, ...}

git diff (tests/test_tracemalloc_toggle_wiring.py):
----------------------------------------------------
diff --git a/tests/test_tracemalloc_toggle_wiring.py b/tests/test_tracemalloc_toggle_wiring.py
index 616bab5c..3b257967 100644
--- a/tests/test_tracemalloc_toggle_wiring.py
+++ b/tests/test_tracemalloc_toggle_wiring.py
@@ -20,6 +20,7 @@
 from services.exact_diff import agg_contribution as ac
 from services.exact_diff import execution_settings as es
+from services.exact_diff.pk_range_chunk import run_unsorted_chunk_pk_lookup_compare
 from services.exact_diff.store import ExactDiffRunStore
@@ -31,11 +32,12 @@
-# 기준선)는 run_pk_range_chunk_compare 삭제와 함께 제거했다. 아래 세 테스트는 이제 DIRECT_STREAM
-# 엔진(_run_stream/prepare_reimport_pk_index_stream, merge-walk 삭제와 무관하게 그대로 생존) 기준
-# 하나로만 계측 배선을 검증한다 — 생존 UNSORTED_CHUNK_PK_LOOKUP 엔진은 애초에 tracemalloc 계측이
-# 배선돼 있지 않아(services/exact_diff/pk_range_chunk.py 확인) 대체 기준선을 추가하지 않았다
-# (커버리지 공백, 후속 지침 권고).
+# 기준선)는 run_pk_range_chunk_compare 삭제와 함께 제거했다.
+# [UNSORTED-CHUNK-PK-LOOKUP-COVERAGE-GAPS-M381-M382, 파트B(M382)] 위 코멘트가 남겼던 커버리지 공백
+# (생존 UNSORTED_CHUNK_PK_LOOKUP 엔진에 tracemalloc 계측이 배선돼 있지 않던 문제)을 이 지침으로
+# 메웠다 — 아래 세 테스트는 이제 DIRECT_STREAM 엔진과 UNSORTED_CHUNK_PK_LOOKUP 엔진 둘 다를 기준으로
+# 계측 배선을 검증한다.
@@ (신규) _run_lookup(run_id) 헬퍼 추가
@@ test_off_does_not_start_tracemalloc: out_l = _run_lookup(...) 추가, for out in (out_s, out_l)
@@ test_on_measures_again: out_l = _run_lookup(...) 추가, for out in (out_s, out_l)
@@ test_toggle_does_not_change_verdict: off_l/on_l 추가 + reimport/status 동일성 assert 추가

신규 파일: tests/test_unsorted_chunk_progress_basis.py (88 lines, 3 tests) — 위 "실제조치" 참고.

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오(순수 백엔드 엔진 계측 배선 + 테스트, 화면 요소 무변경)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(UI 변경 자체가 없어 이 규칙 대상 아님)

**기대효과**
- M381: UNSORTED_CHUNK_PK_LOOKUP 엔진의 진행률 표시(processed_src/available)가
  역행하거나 basis 초기값이 잘못 저장되는 회귀를 이후 CI/자체테스트에서
  즉시 탐지(탐지력 실측으로 확인).
- M382: 대량 chunk 처리 시 메모리 폭주 등 이상을 이 엔진에서도 tracemalloc
  으로 조기 탐지 가능해짐 — DIRECT_STREAM 엔진과 동일한 관측 가능성 확보.
- BACKLOG.md M381/M382 두 항목 모두 "✅ 해결 완료"로 갱신, verify 저장소
  커밋 d3786d1 로 push 완료.

관련 커밋:
- 코드 저장소(X:\xDataNexPro\nxDTV): 30aa263e (commit, push 불필요 — 개별
  작업 단위, 21번 규칙 예외 아님)
- verify 저장소(migration-validator-verify, X:\Verify\nxDTV\_rpt_push):
  d3786d1 (commit+push 완료, fd79726..d3786d1)

작업명 : UNSORTED-CHUNK-PK-LOOKUP-COVERAGE-GAPS-M381-M382
✅ 작업 완료 - progress_cb/basis 회귀 테스트 신설(M381) + tracemalloc 계측 배선(M382) 완료
```
