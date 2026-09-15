```text
작업명 : CODEBASE-AUDIT-SAFE-CONSOLIDATION-FIXES-IMPLEMENT
✅ 작업 완료 - SQL 해시/GROUP BY·SUM 상한/Oracle LOB 타입 목록 3건 안전 통합(값 변경 없는 리터럴 단일 출처화), 커밋 3건(파트별) 완료

**목적**
CODEBASE-WIDE-DEAD-CODE-AND-ISSUES-AUDIT 에서 확인된 항목 중 개별 판단 없이 안전하게
바로 처리 가능한 3가지를 구현한다.
- 파트1: SQL 해시 잔여 지점을 canonical_sql_hash 기반으로 통합
- 파트2: GROUP BY/SUM "최대 3개" 정책 상수를 리터럴 반복에서 공유 상수로 통합
- 파트3: Oracle LOB 타입 목록을 공유 상수로 통합
48개 고아 함수 삭제 여부·172건 무주석 예외처리 검토는 지침 범위 밖(미착수, 요청대로).

**현상**
- SQL 해시 로직이 파일마다 각자 정규화 규칙(sha1/sha256, 정규식 기반)으로 중복 구현돼
  있어 포맷 차이(공백/대소문자)에 취약하고, canonical_sql_hash(sqlglot 기반) 도입 이후
  일부 지점만 전환되지 않은 채 남아 있었다.
- GROUP BY/SUM "자동선정 최대 3개" 라는 CLAUDE.md 절대 규칙이 코드에는 서비스 모듈
  10곳 안팎에 리터럴 `= 3` 으로 반복 정의돼 있어, 향후 정책이 바뀔 때 누락 위험이 있었다.
- Oracle LOB 타입 목록(CLOB/NCLOB/BLOB/LONG/LONG RAW)이 db/meta_collector.py 와
  services/exact_diff/dialects/oracle.py 에 각자 독립 정의돼 있어 한쪽만 바뀌면
  판정이 어긋날 위험이 있었다.

**실제조치**

━━━ 파트1 — SQL 해시 잔여 지점 canonical_sql_hash 통합 (커밋 a02cbed4) ━━━
지침이 지목한 3개 지점을 개별 조사한 결과, 안전하게 전환 가능한 곳은 1곳뿐이었다
(나머지 2곳은 조사 결과 "전환 대상 아님"으로 판단 — 근거 아래 기재, 임의 판단이
아니라 조사 후 도출한 결론).

1. services/validation_sql_parse_service.py:_normalized_sql_hash — ✅ 전환함.
   - 기존: 정규식 정규화(공백 단일화+대문자) 후 sha1.
   - 신규: services/single_run_condition.py::query_hash_or_raw_fallback() 재사용
     (canonical_sql_hash 우선, 파싱 실패 시 원문 sha256 폴백).
   - 안전성 근거: 이 값(SqlParseResult.normalized_sql_hash)의 실제 소비처를 추적한 결과,
     routes/batch_route.py 에서 API 응답 표시용(entry["normalized_sql_hash"])으로만
     쓰이고, 실제 영속 dedup key(DTV_policy_target_table_config.normalized_sql_hash)는
     policy_target_table_service.py 가 이 필드를 전혀 참조하지 않고 자체적으로 이미
     query_hash_or_raw_fallback() 을 호출해 별도 계산하고 있었다(선행 오늘자 지침들의
     결과물). 값의 길이/hex 형식을 고정하는 테스트도 없어 형식 변경 위험이 없다.

2. services/profile_snapshot_service.py:_make_hash/_make_sql_hash/_make_schema_hash
   — ❌ 전환 안 함(조사 결과 대상 아님).
   - _make_sql_hash 는 프로덕션 코드 어디서도 호출되지 않는 고아 함수(테스트에서만
     참조) — 본 지침이 명시적으로 제외한 "48개 고아 함수" 범주에 속해 손대지 않음.
   - _make_hash/_make_schema_hash 는 SQL 문자열이 아니라 커넥션 식별자(conn_id)와
     컬럼명 집합(schema fingerprint)을 해시하는 함수로,애초에 "SQL 해시"가 아니다
     (canonical_sql_hash 를 적용하면 SQL 이 아닌 입력을 sqlglot 파서에 넣게 돼 오히려
     항상 실패/None 처리되는 회귀가 생긴다). 전환 대상에서 제외.

3. services/validation_history_service.py:normalize_sql/build_sql_hash
   — ❌ 전환 안 함(조사 결과 대상 아님).
   - normalize_sql 의 출력(정규화된 SQL "텍스트")은 build_set_fingerprint() 안에서
     다른 필드들과 "|" 로 join 되는 한 조각으로 쓰이며, samples/test_validation_
     history_service.py TC-7/TC-8 이 "SELECT 대문자 포함", "연속공백 없음" 등 텍스트
     형태 자체를 직접 검증한다 — canonical 해시(hex 32자)로 바꾸면 이 텍스트 계약이
     깨진다.
   - build_sql_hash 는 TC-9 가 "길이 64, hex 문자만"을 명시적으로 검증하는데,
     canonical_sql_hash 는 32자, query_hash_or_raw_fallback 폴백은 "raw_" 접두사가
     붙은 28자로 — 길이/문자셋 계약이 근본적으로 다르다. 또한 job_id 로 영속 저장되는
     값이라(services/validation_history_service.py:470) 알고리즘을 바꾸면 과거 이력과
     새 실행이 같은 job으로 병합되지 않는 연속성 손실이 생긴다.
   - 결론: 기존 정규화+sha1/sha256 계약이 테스트·저장 형식 양쪽에 강하게 결합돼 있어
     "안전한 기계적 치환"이 아니라 별도 개별 판단이 필요한 사안으로 판단, 이번 지침
     범위에서는 보류.

E2E 검증(파싱 성공/실패 케이스):
  - 파싱 성공(동치 SQL): "SELECT a FROM t" vs "select   a  from   t" → 동일 해시.
  - 파싱 성공(다른 SQL): "SELECT a FROM t" vs "SELECT b FROM t" → 다른 해시.
  - 파싱 실패(원문만 다른 비SQL): "...@@@" vs "...###" → raw_ 폴백이지만 서로 다른 해시
    (오늘 확립된 안전 원칙 — 파싱 실패해도 다른 원문은 다른 값 — 충족 확인).
  - 빈 SQL: 빈 문자열 반환(EMPTY_SQL 에러 경로로 바로 빠져 하위 로직에 영향 없음).

━━━ 파트2 — GROUP BY/SUM "최대 3개" 공유 상수화 (커밋 6ca9cc58) ━━━
config/model_config.py 에 시스템 절대 상한 상수 2개 신설(값 변경 없음):
  MAX_GROUPBY_COLS = 3, MAX_SUM_COLS = 3
화면에서 조정 가능한 정책DB 값(GENERAL_COLUMN_MAX_GROUPS = 그룹 "결과 행 수" 상한,
INTERACTIVE_GROUPBY_MAX_GROUPS = 실행 안전 게이트의 "행 수" 상한)과는 "선택 컬럼
개수" vs "그 선택으로 나오는 그룹/행 수" 로 축이 다르다는 점을 주석으로 명확히
구분(요청 2번 항목 반영).

9개 서비스 모듈의 리터럴 `= 3` 을 이 상수 참조로 교체(지침이 나열한 10개 파일 중
후보_context_adapter 포함 — 실제로는 9개 물리 파일):
  candidate_context_adapter.py, candidate_engine.py, candidate_recommendation_policy.py,
  candidate_scoring_runner.py, column_profile_service.py, date_bucket_evidence.py,
  fallback_recommendation.py, table_effective_policy_service.py, validation_set_builder.py

세부 판단(값이 같다고 무조건 치환하지 않고 개별 의미를 확인):
  - candidate_scoring_runner.py 의 _apply_top_n/_apply_top_n_by_table, fallback_
    recommendation.py 의 promote_general_fallback_default 는 GB/SUM 공용 제네릭
    Top-N 헬퍼(파라미터명도 role-중립)라 실제로는 모든 프로덕션 호출부가 max_n 을
    명시 전달하고 기본값은 쓰이지 않는다 — 기본값은 대표로 MAX_GROUPBY_COLS 를
    사용하고 주석으로 명시.
  - candidate_engine.py 의 apply_limit(line 106, max_n=3 기본값)은 지침이 열거한
    "candidate_engine.py(2곳)"에 포함되지 않는 별개 함수로, docstring 에 "legacy/
    test-compat alias — 라이브 선정에는 쓰이지 않음"이라 명시돼 있어 대상에서 제외.
  - column_profile_service.py 의 max_pairs=3(collect_pair_distinct_sample)은 값은
    같지만 의미가 다르다 — 기존 docstring 이 이미 "GROUP BY 상한 3개에서 나오는 최대
    조합 수(3choose2=3)"라고 명시한 파생값이다. 그대로 상수를 대입하면 향후
    MAX_GROUPBY_COLS 가 바뀔 때 조합 수가 안 맞게 되므로, 리터럴 대입 대신
    `(MAX_GROUPBY_COLS * (MAX_GROUPBY_COLS - 1)) // 2` 조합 공식으로 파생시켜
    상한이 바뀌어도 자동으로 맞게 따라가도록 처리(config/size_threshold_registry.py
    가 이미 경고한 "값이 같아 보여도 다른 개념은 합치지 않는다" 원칙 적용).
  - validation_set_builder.py 의 sum_chunk_size(UNKNOWN=3/LARGE=3, 79-103행)는
    "row_count 기준 SUM chunk 크기"라는 별개의 adaptive 정책값(SMALL=10/MEDIUM=5로
    서로 다름)이라 지침 대상(default_sum_limit 파라미터)과 구분해 손대지 않음.
  - routes/batch_route.py 는 세션 시작 시점부터 이미 이 지침과 무관한 별도 작업
    (PERFORMANCE-AUDIT-SAFE-FIXES-IMPLEMENT)으로 미커밋 상태였음을 확인, 이번 커밋에
    포함하지 않음(건드리지 않음 — git status 로 확인 후 분리 staging).

━━━ 파트3 — Oracle LOB 타입 목록 공유 상수화 (커밋 97fb7345) ━━━
신규 config/oracle_lob_types.py 생성 — ORACLE_LOB_TYPE_NAMES = frozenset({"CLOB",
"NCLOB", "BLOB", "LONG", "LONG RAW"}).

- db/meta_collector.py:_EXCLUDE_TYPES['oracle'] — 이 상수 + RAW(LOB 이 아니라 "Long/
  Binary" 라는 별도 사유로 함께 EXCLUDE되던 것이라 그 자리에 유지)로 재구성.
  변환 전/후 set 이 완전히 동일함을 런타임에서 직접 비교해 확인(equal: True).
- services/exact_diff/dialects/oracle.py:_ORACLE_TYPE_CATEGORY — CAT_LOB 매핑 5개
  항목을 이 상수에서 "CLOB"→"DB_TYPE_CLOB", "LONG RAW"→"DB_TYPE_LONG_RAW" 식으로
  파생. 변환 전/후 dict 가 완전히 동일함을 런타임에서 직접 비교해 확인(equal: True,
  실 diff 실행 경로의 해시 판정에 영향 없음 검증).
- services/table_size_scoring.py:_LOB_TYPES_BY_DIALECT['oracle'] — 목적이 다름
  (바이트 합산 배제용, BFILE 포함하는 상위집합)이라 완전 통합은 하지 않고, 공유
  상수 + BFILE 추가로 부분 통합(요청 3번의 "완전히 합치기 어려우면 최소한 목록
  자체만이라도 공유" 지침 반영). 변환 전/후 set 이 완전히 동일함을 확인(equal: True).

**검증 결과**
파트1: tests/test_validation_sql_parse_service.py 11 passed / samples/
  test_virtual_cases.py 8/8 / samples/test_complex_cases.py 5/5.
파트2: 후보추천/fallback/date_bucket/validation_set_builder/table_effective_policy
  관련 pytest 330 passed, 1 skipped(무관) / samples 두 회귀 스위트 전부 통과.
  실패 6~9건 발견했으나 전부 git worktree 로 변경 전(HEAD) 코드에서도 동일하게
  재현되는 기존 실패(주로 한글 인코딩 관련 JS/HTML 문자열 검색 테스트)임을 확인 —
  이번 변경과 무관.
파트3: tests/test_table_size_scoring.py + test_groupby_lob_hard_exclusion.py 27
  passed / oracle 관련 pytest 223 passed, 1 skipped(무관), 1건 기존 실패(worktree
  대조로 무관 확인) / samples 두 회귀 스위트 전부 통과.
공통: 3개 커밋 전부 git diff 확보(본 보고서 하단 첨부, 30번 규칙 증적).

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 — services/db/config 계층의
  순수 백엔드 로직(해시 계산, 정책 상수, 타입 판별)만 수정했고 버튼/체크박스/문구/
  배지 등 화면 요소는 전혀 건드리지 않음.
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(UI 변경 자체가 없음)

**기대효과**
- SQL 동일성 판정이 canonical_sql_hash(sqlglot 기반)로 더 일관되게 수렴(불필요한
  신규 실행/캐시 미스 감소).
- GROUP BY/SUM 상한 정책이 향후 바뀌어도 config/model_config.py 한 곳만 수정하면
  9개 모듈에 자동 반영 — 누락 위험 제거. 단, 정책값 자체는 여전히 3(값 변경 없음).
- Oracle LOB 타입 판별이 두 소비처(meta_collector/exact_diff)에서 항상 같은 목록을
  참조하게 돼, 한쪽만 갱신해 판정이 어긋나는 회귀를 구조적으로 방지.
- 전환하지 않기로 한 2건(profile_snapshot_service, validation_history_service)은
  조사 근거를 코드 주석과 커밋 메시지에 남겨, 다음 세션이 같은 조사를 반복하지
  않도록 함.

━━━ git diff 증적(30번 규칙) ━━━

[커밋 a02cbed4 — 파트1] services/validation_sql_parse_service.py
--- a/services/validation_sql_parse_service.py
+++ b/services/validation_sql_parse_service.py
@@ -28,7 +28,6 @@ parser dialect:
 
 from __future__ import annotations
 
-import hashlib
 import re
 from concurrent.futures import ThreadPoolExecutor
 from dataclasses import dataclass, field
@@ -39,6 +38,7 @@ from services.sql_parser_adapter import (
     _DIALECT_TO_PARSER,
     ParsedMigrationModel,
 )
+from services.single_run_condition import query_hash_or_raw_fallback
 
 # INSERT INTO ... 형태 statement 인식 (txt 자유텍스트에서 SQL 만 추출용)
 _RE_INSERT_INTO = re.compile(r"\bINSERT\s+INTO\b", re.IGNORECASE)
@@ -164,10 +164,18 @@ def _strip_sql_comments(sql: str) -> str:
     return out
 
 
-def _normalized_sql_hash(sql: str) -> str:
-    """SQL 정규화(공백 단일화·대문자) 후 sha1 해시."""
-    norm = re.sub(r"\s+", " ", (sql or "").strip()).upper()
-    return hashlib.sha1(norm.encode("utf-8")).hexdigest()
+def _normalized_sql_hash(sql: str, dbms: str = "postgresql") -> str:
+    """SQL 동일성 판정용 해시 — canonical_sql_hash(sqlglot) 우선, 파싱 실패 시 원문 sha256 폴백.
+    ...(중략, 전체는 커밋 참고)...
+    """
+    return query_hash_or_raw_fallback(sql or "", dbms)
 
 
 def parse_validation_sql_item(...):
     res = SqlParseResult(
         item_id=item.item_id, mode=mode, success=False,
-        normalized_sql_hash=_normalized_sql_hash(item.migration_sql),
+        normalized_sql_hash=_normalized_sql_hash(item.migration_sql, dialect or "postgresql"),
         ...
     )

[커밋 6ca9cc58 — 파트2] config/model_config.py (신규 상수) + 9개 서비스 모듈
config/model_config.py:
+MAX_GROUPBY_COLS = 3   # GROUP BY 자동선정 최대 컬럼 개수(시스템 절대 상한)
+MAX_SUM_COLS = 3        # SUM 자동선정 최대 컬럼 개수(시스템 절대 상한)

예) services/candidate_engine.py:
-    effective_max_group_by: int = 3,
-    effective_max_sum: int = 3,
+    effective_max_group_by: int = MAX_GROUPBY_COLS,
+    effective_max_sum: int = MAX_SUM_COLS,

예) services/column_profile_service.py(파생값 처리):
+_MAX_GROUPBY_PAIR_COMBINATIONS = (MAX_GROUPBY_COLS * (MAX_GROUPBY_COLS - 1)) // 2
-    max_pairs:    int = 3,
+    max_pairs:    int = _MAX_GROUPBY_PAIR_COMBINATIONS,
(나머지 7개 파일 동일 패턴 — 전체 diff는 로컬 커밋 6ca9cc58 참고)

[커밋 97fb7345 — 파트3] config/oracle_lob_types.py(신규) + 3개 파일
config/oracle_lob_types.py(신규):
+ORACLE_LOB_TYPE_NAMES: frozenset[str] = frozenset({
+    "CLOB", "NCLOB", "BLOB", "LONG", "LONG RAW",
+})

db/meta_collector.py:
     'oracle': {
         'TIMESTAMP', 'TIMESTAMP WITH TIME ZONE', 'TIMESTAMP WITH LOCAL TIME ZONE',
-        'CLOB', 'NCLOB', 'BLOB', 'LONG', 'RAW', 'LONG RAW',
-    },
+        'RAW',  # LOB 은 아니나(고정폭 최대 2000byte) "Long/Binary" 사유로 함께 EXCLUDE(동작 불변)
+    } | ORACLE_LOB_TYPE_NAMES,

services/exact_diff/dialects/oracle.py:
-    "DB_TYPE_CLOB": CAT_LOB, "DB_TYPE_NCLOB": CAT_LOB,      # 해시 입력 불가 — 비교 제외
-    "DB_TYPE_BLOB": CAT_LOB, "DB_TYPE_LONG": CAT_LOB, "DB_TYPE_LONG_RAW": CAT_LOB,
+    **{"DB_TYPE_" + _n.replace(" ", "_"): CAT_LOB for _n in ORACLE_LOB_TYPE_NAMES},

services/table_size_scoring.py:
-    "oracle": frozenset({"clob", "nclob", "blob", "bfile", "long", "long raw"}),
+    "oracle": frozenset({n.lower() for n in ORACLE_LOB_TYPE_NAMES}) | frozenset({"bfile"}),

(전체 diff는 git show a02cbed4 / 6ca9cc58 / 97fb7345 로 재확인 가능 — 코드 저장소
X:\xDataNexPro\nxDTV, main 브랜치)

작업명 : CODEBASE-AUDIT-SAFE-CONSOLIDATION-FIXES-IMPLEMENT
✅ 작업 완료 - SQL 해시/GROUP BY·SUM 상한/Oracle LOB 타입 목록 3건 안전 통합(값 변경 없는 리터럴 단일 출처화), 커밋 3건(파트별) 완료
```
