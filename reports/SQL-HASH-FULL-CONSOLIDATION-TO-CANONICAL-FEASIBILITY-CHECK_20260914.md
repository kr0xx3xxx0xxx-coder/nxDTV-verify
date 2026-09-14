```text
작업명 : SQL-HASH-FULL-CONSOLIDATION-TO-CANONICAL-FEASIBILITY-CHECK
⚠️ 추가 작업 필요 - 10곳 전부 canonical_sql_hash 이전은 "가능"하나 전량 동시 무조건 안전은 아님(3개 지점은 None-처리 로직 변경 선행 필요, 2개 그룹은 여러 파일 동시배포 필수) — 코드 수정 없음(조사만)

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 (조사 전용 지침, 코드·화면 요소를 전혀 건드리지 않음)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음 (UI 변경 자체가 없어 실측 대상 아님)

※ 30번 규칙(소스 diff 증적): 본 지침은 코드 수정이 전혀 없는 순수 조사이므로 대상 아님.

## 목적

선행 조사(SQL-HASH-SQLGLOT-NORMALIZATION-MIGRATION-IMPACT-INVESTIGATE-ONLY, 2026-09-14)가
"위험한 2곳(policy_target_table_service.py / validation_target_registration_service.py)만
canonical_sql_hash로 옮기고 나머지는 그대로 두자"를 권고했으나, 사용자가 "비슷한 두 함수를
계속 유지하지 말고 나머지도 전부 canonical_sql_hash로 옮기고 services/common/sql_hash.py
자체를 완전히 제거하면 안 되냐"고 물었다. 9번 규칙(중복 제거)과 방향은 일치하므로, 실제로
안전한지 나머지 호출부 각각을 코드 레벨로 확인한다. 코드 수정 없이 조사·실현가능성 판단만
수행한다.

## 현상 (조사 근거)

### 0) 두 함수의 근본적 계약 차이(핵심 걸림돌)

| 항목 | `services/common/sql_hash.py` (`compute_sql_hash`) | `services/sql_canonical.py` (`canonical_sql_hash`) |
|---|---|---|
| 반환 타입 | `str` 단일값(항상 반환) | `tuple[str\|None, str]` (해시, 사유코드) |
| 빈/파싱불가 입력 | 빈 문자열의 sha256을 반환(예외 없음) | `(None, 사유코드)` 반환(설계원칙 §12) |
| 정규화 방식 | 정규식(공백/세미콜론만 흡수, 대소문자 보존) | sqlglot AST 재생성(리터럴/식별자/hint/순서 보존, 키워드 대소문자·공백만 정규화) |

이 계약 차이 때문에 단순 함수 치환(find & replace)은 어느 호출부에서도 그대로 동작하지
않는다 — 모든 호출부가 최소 "tuple 언패킹"을, 상당수가 "None 의미를 그 호출부 맥락에
맞게 처리하는 코드"를 추가로 요구한다.

### 1) 10곳 개별 확인 — None 처리 가능/필요 여부 (실제 코드 근거)

**A. `services/workflow_stage_guard.py:127-134` (`_norm_sql`/`_sql_hash`)**
- 용도: `/analyze` 성공 시 서버가 발급하는 workflow_token에 SQL hash를 바인딩하고, 후속
  단계(`/count`~`/single/save`) 요청마다 재계산한 hash를 저장된 hash와 비교해 "분석 시점과
  SQL이 달라졌는가"를 검증한다(`validate()`, `workflow_stage_guard.py:355-357`):
  ```
  sql_hash, src_fp, tgt_fp, project = _extract(stage, req)
  if sql_hash and sql_hash != ctx.sql_hash:
      return _block("SQL_CHANGED", stage, _PREREQ.get(stage))
  ```
- **None이 오면**: `canonical_sql_hash`로 교체 시 파싱 실패 SQL은 `sql_hash=None`이 되고,
  `if sql_hash and ...`의 `if sql_hash`(진위평가)가 `False`가 되어 **SQL_CHANGED 검사 자체가
  통째로 스킵**된다. 즉 sqlglot이 파싱하지 못하는 이관 SQL(오라클 PL/SQL 블록, 일부 MERGE,
  미지원 hint 등 — 이 프로젝트 이관 SQL에서 드물지 않은 유형)에 대해서는, 단계 사이에 실제로
  다른 SQL을 제출해도 서버가 이를 탐지하지 못한다. 이 가드는 "클라이언트가 성공 여부·문맥을
  임의로 정하지 못하게" 만든 보안성 토큰 바인딩(파일 헤더 주석, `workflow_stage_guard.py:18-20`)
  이므로, 이는 **추측이 아니라 코드로 확인되는 구체적 회귀**다.
- 성능: 걱정할 수준 아님 — `parser/sqlglot_safe_parse.py:38-40` 자체 실측 주석에 "워커 재사용
  상태에서 정상 SQL 파싱 오버헤드는 호출당 +0.37~0.54ms", 상단 주석(`같은 파일:52-53`)에
  "이관 SQL 1건 ≈ 1~10ms"로 기록돼 있다. 단계당 1회 호출(요청당 1회)이므로 DB 왕복(수초)에
  비해 무시 가능.
- **판정: 이전하려면 이 부분을 같이 고쳐야 함** — `if sql_hash and ...`를 "None이면 오히려
  보수적으로 SQL_CHANGED 처리"(또는 별도 UNVERIFIABLE 코드로 차단)하도록 로직 자체를 바꿔야
  안전하다. 단순 함수 교체로는 보안 가드가 조용히 약화된다.

**B. `services/batch_wrapper_result_store.py:80-90` (`sql_hash()`) + C. `services/group_current_freshness_service.py:255,337` (동일 함수 재사용, freshness 배지)**
- 용도: `DTV_batch_wrapper_result.sql_hash`(컬럼 `NOT NULL DEFAULT ''`, `batch_wrapper_result_store.py:50`)에
  저장, `group_current_freshness_service.py`가 "현재 등록된 SQL의 hash"와 "마지막 저장된 결과의
  hash"를 비교해 fresh/stale 배지를 판정(`group_current_freshness_service.py:255-256,337-346`):
  ```
  cur_hash = _rstore_hash(m["migration_sql"] or "")
  fresh = "fresh" if (res.get("sql_hash") or "") == cur_hash else "stale"
  ```
- **None이 오면**: 저장측(`_extract_record`, `batch_wrapper_result_store.py:173`)은
  `str(r.get("sql_hash") or "").strip()`이라 None을 이미 방어적으로 ""로 흡수하므로 DB
  INSERT 자체는 깨지지 않는다(NOT NULL 위반 없음). 문제는 비교 로직 쪽이다 — 두 SQL이 서로
  달라도 **둘 다 sqlglot 파싱에 실패**하면 `cur_hash=""`, `res.get("sql_hash")=""`가 되어
  `"" == ""`로 **"fresh"(최신)로 오판**한다. 실제로는 "판정 불가"인데 "동일함"으로 표시되는
  것이므로 설계 원칙 §12(무리한 동일 판정 금지)를 정확히 위반하는 방향이다. 현재 정규식
  방식은 이런 충돌이 사실상 없다(파싱 성패와 무관하게 항상 실제 SQL 텍스트 기반 해시를 냄).
- **판정: 이전하려면 이 부분을 같이 고쳐야 함(두 파일 동시)** — `sql_hash()`가 파싱 실패를
  구분 가능한 값(예: 빈 문자열이 아닌 별도 sentinel, 혹은 "판정불가"를 의미하는 명시적 플래그)
  으로 반환하고, freshness 비교 쪽이 "hash가 없으면(빈 문자열이 아니라 '판정불가') stale도
  fresh도 아닌 별도 상태로 표시"하도록 함께 고쳐야 한다.

**D. `services/single_run_condition.py:27-45` (`single_condition_fingerprint`) + E. `routes/single_completed_history_route.py:39-51` (`single_completed_history`, 소비측)**
- 용도: "동일 완료이력" 판정용 조건 지문을 만들어(`query_hash` 포함) 저장·조회 양쪽에서 같은
  방식으로 계산해 비교한다(`single_run_condition.py:32-35`):
  ```
  from services.sql_normalizer import sql_hash
  qh = sql_hash(query_full or "") if (query_full or "").strip() else ""
  ...
  payload = {"query_hash": qh, ...}
  blob = json.dumps(payload, sort_keys=True, ensure_ascii=False)
  return "cond_" + hashlib.sha256(blob.encode("utf-8")).hexdigest()[:24]
  ```
- **None이 오면**: 현재 `try/except`는 "예외가 나면 ''"만 방어한다. `canonical_sql_hash`는
  파싱 실패를 예외가 아니라 정상 반환값 `(None, reason)`으로 주므로, 언패킹하지 않고 그대로
  `qh = sql_hash(...)`로 대입하면 `qh`가 tuple이 되어 `json.dumps`가 `[null, "reason"]`
  형태로 직렬화된다(동작은 하지만 지문 스키마가 바뀜). 제대로 언패킹해도 더 근본적인 문제가
  남는다 — **서로 다른 두 SQL이 같은 이유(예: 둘 다 동일한 미지원 구문 패턴)로 파싱에
  실패하면 `qh`가 똑같이 None(또는 동일 reason 문자열)이 되어, 두 SQL이 완전히 달라도 같은
  조건 지문을 만들어낼 수 있다.** 선행 조사(§2)는 이 계열을 "불일치 시 단순 cache-miss(안전)"
  로 분류했는데, 그건 **서로 다른 값이 서로 다른 해시로 갈라지는 것을 전제**로 한 판단이다.
  canonical 방식은 파싱 실패 케이스에서 "서로 다른 SQL이 같은 해시로 뭉쳐질" 수 있으므로,
  `single_completed_history_route.py:51`의 `_rj.find_completed_by_condition(...)`가 **실제로는
  다른 쿼리의 완료 이력을 "동일 조건"으로 잘못 매칭해 반환**할 위험이 새로 생긴다(cache-miss가
  아니라 false cache-hit). 이는 표시용 "재실행/기존결과보기" 선택창의 근거 데이터가 틀어지는
  것이라 §12 원칙상 허용하기 어렵다.
- **판정: 이전하려면 이 부분을 같이 고쳐야 함(두 파일 동시 배포 필수)** — `query_hash`
  구성요소가 파싱 실패 시에도 SQL 원문에서 유래한 구분 가능한 값(예: 원문 sha256을 폴백으로
  섞는 등)을 유지하도록 고친 뒤, `single_run_condition.py`와 `single_completed_history_route.py`
  양쪽을 같은 배포에서 함께 바꿔야 한다(한쪽만 바뀌면 저장 시점 지문과 조회 시점 지문의
  계산 방식이 서로 달라 상시 불일치가 나거나, 위 충돌 문제가 그대로 남는다).

**F. `services/single_validation_analyze_service.py:2243-2254` (chunk key evidence `query_hash`)**
- 용도: `build_chunk_key_evidence_snapshot(..., query_hash=_qh_a, ...)`로 snapshot에 기록하고,
  `services/diagnosis/key_evidence.py:433-456`의 `resolve_trusted_chunk_key`가 재사용 여부를
  판단한다:
  ```
  and bool(query_hash) and reuse_snapshot.get("query_hash") == query_hash
  ```
- **None이 오면**: 이 소비측은 **이미 `bool(query_hash)` 가드**를 갖고 있어, 새로 계산한
  `query_hash`가 None(또는 빈 문자열)이면 그 자체로 재사용 조건이 거짓이 되어 **항상 안전하게
  "재사용 안 함"(metadata 재조회로 폴백)** 으로 떨어진다 — 파일 자체 주석 "실패는 무시(비파괴)"
  (`single_validation_analyze_service.py:2237`)가 이미 이 상황을 전제로 설계돼 있다.
  `build_chunk_key_evidence_snapshot`(`key_evidence.py:370`) 저장 시에도
  `"query_hash": query_hash or ""`로 None을 방어적으로 흡수한다.
- **판정: 안전하게 이전 가능** — 단, tuple을 그대로 넘기지 말고 `hash_or_none, _reason =
  canonical_sql_hash(sql, dbms)`로 정확히 언패킹해야 한다(언패킹만 지키면 하위 로직 변경 불필요).

**G. `services/single_official_register_service.py:429-496`**
- 용도: `new_hash = sql_hash(sql or "") or ""`(regex 해시)를 계산하지만, **실제 "동일본" 판정은
  이미 canonical_sql_hash로 하고 있다** — `single_official_register_service.py:184-204`:
  ```
  ex_canon = cur.get("_canonical_hash") or None
  if not ex_canon:
      ex_canon, _r = canonical_sql_hash(cur.get("_raw_sql") or "", src_db)
  ...
  sql_match = (new_canon is not None and ex_canon is not None and new_canon == ex_canon)
  ```
  `new_hash`(regex)는 no-op 응답의 `migration_sql_hash` 표시 필드에만 쓰인다
  (`single_official_register_service.py:111,459`) — 동일성 판정에는 전혀 관여하지 않는다.
  같은 파일 496행에 이미 None-안전 관용구도 있다: `sql_identity=(new_canon or
  ("rawfallback:" + new_hash))`.
- **None이 오면**: 표시 필드가 `null`이 될 뿐 판정 로직에는 영향 없음. 이미 이 파일 안에
  "canon이 없으면 rawfallback 문자열" 관용구가 존재하므로 그대로 재사용하면 된다.
- **판정: 안전하게 이전 가능** — 실질적으로 이미 canonical_sql_hash 기반으로 판정 중이며,
  regex 해시는 표시용 잔재에 가깝다.

**H-1. `routes/agg_diff_route.py:1555-1564` (chunk key evidence `_qh`)**
- F와 완전히 동일한 소비 패턴(`resolve_trusted_chunk_key` + `bool(query_hash)` 가드) —
  **판정: 안전하게 이전 가능**(tuple 언패킹만 주의).

**H-2. `routes/agg_diff_route.py:2008-2032` (`_qhash` → job_registry 메타 기록) + I. `routes/single_active_run_route.py:56-63`(`_same_query`) + J. `routes/single_exec_gate_route.py:40-48`(`_same_query`)**
- 용도: 재이관(exact_diff) 실행 **시작 시점**에 `agg_diff_route.py:2011-2012`가 `sql_hash(query)`
  값을 job_registry의 run 메타 `query_hash`로 저장하고(`services/job_registry.py:541-545` 저장,
  `services/exact_diff/reimport_job.py:345` 유사 경로), **조회 시점**에
  `single_active_run_route.py:59-60`/`single_exec_gate_route.py:45-46`가 **다시 `sql_hash()`를
  호출해 방금 계산한 값과 저장된 `run.get("query_hash")`를 비교**한다:
  ```
  same = (sql_hash(query) == run.get("query_hash"))
  ```
- **None(및 tuple)이 오면 — 이 그룹이 10곳 중 가장 위험도가 높다.** 두 가지 문제가 겹친다.
  1. tuple 문제: `single_active_run_route.py`/`single_exec_gate_route.py` 쪽만 먼저
     `canonical_sql_hash`로 바꾸고 언패킹을 잊으면 `sql_hash(query)`가 tuple이 되어
     문자열(`run.get("query_hash")`)과 **항상 다름** — `same`이 항상 `False`가 된다.
  2. **더 근본적으로, 쓰기(`agg_diff_route.py`)와 읽기(`single_active_run_route.py`,
     `single_exec_gate_route.py`) 두 지점이 서로 다른 파일에서 독립적으로 `sql_hash`를
     호출한다.** 한쪽만 canonical로 옮기고 다른 쪽을 regex로 남겨두면(예: 부분 배포,
     롤링 배포 중 신·구 버전 혼재, 혹은 이번 조사에서 실수로 한 파일만 고치는 경우),
     저장된 값과 재계산 값이 **서로 다른 해시 알고리즘의 결과물이라 동일 SQL이어도 절대
     일치하지 않는다.** 이는 테스트 파일 존재로도 뒷받침된다
     (`tests/test_active_run_query_notice.py`,
     `tests/test_urgent_target_only_and_reuse.py`,
     `tests/test_rerun_exec_gate_reuse.py`) — "동일 쿼리로 재진입했는가"를 안내하는
     UI 배지 기능이 전환 도중 상시 오답을 낼 수 있다.
- **판정: 이전하려면 이 부분을 같이 고쳐야 함(3개 파일 — `agg_diff_route.py` 2011행,
  `single_active_run_route.py`, `single_exec_gate_route.py` — 동일 배포에서 원자적으로
  동시 이관 필수)**. 부분 이관은 "안전한 실패(cache-miss)"가 아니라 **기능이 상시 오답을
  내는 조용한 회귀**로 이어진다는 점에서, 10곳 중 유일하게 "이전하면 안 되는 이유가 있음"에
  가장 근접한 항목이지만, 3개 파일을 같은 배포로 묶으면 실제로는 이전 가능하므로 최종
  판정은 "같이 고쳐야 함"으로 분류한다(완전 금지 사유는 아님).

### 2) 참고 — 10곳 목록 밖이지만 관련된 지점(directive의 "나머지 호출부들"에 해당할 수 있음)

`services/validation_result_store.py:168-179`(`_sql_hash`) + `services/validation_run/persistence_verifier.py:82-128`
— `DTV_validation_execution_run.source_sql_hash`/`target_sql_hash`를 resume(재시도) 검증에
사용한다. `_STORE_SNAPSHOT=False`(`validation_result_store.py:32`)라 원문 자체를 저장하지
않으므로, 저장 시점과 재계산 시점의 알고리즘이 배포 전후로 달라지면(예: 오래 대기 중인 run이
배포 경계를 넘어 resume) CONFLICT로 판정될 수 있다. 다만 이 모듈 자체 설계 원칙이 "재사용
금지를 안전한 기본값"으로 명시하므로(선행 조사 §2 인용) 방향은 안전하다(데이터 훼손이 아니라
신규 재실행 강제). directive가 열거한 10곳에는 포함돼 있지 않아 이번 판정표에서는 참고
항목으로만 다룬다.

### 3) 판정표 (10곳 전부)

| # | 파일 | 무엇을 비교/저장하는가 | None 처리 위험 | 판정 |
|---|---|---|---|---|
| 1 | services/workflow_stage_guard.py | 단계 토큰의 SQL hash(인메모리) — SQL_CHANGED 위조탐지 | `if sql_hash and ...`가 None에서 falsy-skip → 위조탐지 가드 무력화 | 이전하려면 같이 고쳐야 함(로직 변경 필요) |
| 2 | services/batch_wrapper_result_store.py | DTV_batch_wrapper_result.sql_hash(NOT NULL) | None→"" 저장은 안전, 단 freshness 비교에서 "둘 다 파싱실패=동일"로 오판 가능 | 이전하려면 같이 고쳐야 함(#3과 동시) |
| 3 | services/group_current_freshness_service.py | #2 hash를 현재 SQL과 비교(freshness 배지) | 상동(#2와 동일 원인) | 이전하려면 같이 고쳐야 함(#2와 동시) |
| 4 | services/single_run_condition.py | 완료이력 매칭용 조건 지문(query_hash 포함) | 서로 다른 SQL이 같은 파싱실패 사유로 지문 충돌(false cache-hit) 가능 | 이전하려면 같이 고쳐야 함(#5와 동시) |
| 5 | routes/single_completed_history_route.py | #4 지문으로 완료 run 조회 | 상동(#4와 동일 원인) | 이전하려면 같이 고쳐야 함(#4와 동시) |
| 6 | services/single_validation_analyze_service.py(chunk key evidence) | PK 증거 snapshot 재사용 판단용 query_hash | 소비측이 이미 bool() None-가드 보유(안전) | 안전하게 이전 가능(tuple 언패킹만 주의) |
| 7 | services/single_official_register_service.py | 표시용 migration_sql_hash(동일성 판정과 무관) | 표시 필드만 영향, 판정 로직은 이미 canonical_sql_hash 사용 중 | 안전하게 이전 가능 |
| 8 | routes/agg_diff_route.py:1557 (chunk key evidence) | #6과 동일 패턴 | 소비측 bool() 가드(안전) | 안전하게 이전 가능 |
| 9 | routes/agg_diff_route.py:2011-2032 (job meta 기록) | job_registry query_hash 저장(쓰기측) | #10·#11과 알고리즘 불일치 시 "동일 쿼리" 배지 상시 오답 | 이전하려면 같이 고쳐야 함(#10·#11과 3파일 동시) |
| 10 | routes/single_active_run_route.py | #9 저장값과 재계산값 비교(읽기측) | 상동(#9와 동일 원인, tuple 비교 버그 포함) | 이전하려면 같이 고쳐야 함(#9·#11과 동시) |
| — | routes/single_exec_gate_route.py(같은 그룹, directive 원문에도 포함) | #9와 동일 소비 패턴 | 상동 | 이전하려면 같이 고쳐야 함(#9·#10과 동시) |

※ 10곳 중 "이전하면 안 되는 이유가 있음(=완전 금지)"으로 확정 판정된 항목은 없다.
다만 #9/#10/#11 그룹은 "부분 이관 시 기능이 조용히 상시 오답을 낸다"는 점에서 사실상 가장
엄격한 제약(여러 파일 원자적 동시배포)을 지닌다.

### 4) 기존 sqlglot 인프라 재사용 가능성 재확인

선행 조사가 확인한 대로 `services/sql_canonical.py`(canonical_sql/canonical_sql_hash)와
`parser/sqlglot_safe_parse.py`(parse_one_guarded, hang 방어)가 이미 완성돼 있고,
`services/single_official_register_txn.py:337-340,415-417`이 이미 두 해시를 **동시에**
계산해 각각 다른 컬럼(`normalized_sql_hash`/`canonical_sql_hash`)에 저장하는 실전 패턴을
보유하고 있다(§18). 이번 10곳 검토에서도 이 패턴(None→"" 저장, `or None`으로 재해석해
비교는 None-safe하게)을 그대로 재사용하면 되는 지점이 다수(#6,#7,#8)였다 — 신규 유틸
설계는 불필요하다.

## 제안 해결안(설계 제안 — 코드 미작성, 실행하지 않음)

### 5) 종합 결론

**10곳 + 2곳(선행 권고) 전부를 canonical_sql_hash로 옮기고 `services/common/sql_hash.py`를
완전히 제거하는 것은 "가능"하다.** 단, 사용자가 상정한 "단순 일괄 치환"은 아니며, 아래 3개
그룹으로 나눠 순서대로 진행해야 안전하다.

**Phase 0(이번 조사 범위 밖, 선행 지침이 이미 권고 — 별도 승인 필요)**
- `services/policy_target_table_service.py`, `services/validation_target_registration_service.py`
  — 선행 조사(2026-09-14) 권고안 그대로.

**Phase 1 — 독립적·저위험, 순서 무관하게 개별 이관 가능(기존 None-safe 소비 패턴 존재)**
1. `services/single_validation_analyze_service.py` chunk key evidence(#6)
2. `services/single_official_register_service.py`의 표시용 hash(#7)
3. `routes/agg_diff_route.py:1557` chunk key evidence(#8)

각각 `hash_or_none, reason = canonical_sql_hash(sql, dbms)`로 정확히 언패킹만 하면 되고,
하위 로직 변경은 불필요(이미 `bool()` 가드/`or None`/`rawfallback:` 관용구가 존재).

**Phase 2 — 쌍/그룹 단위로 같은 배포에서 원자적으로 동시 이관 필수**
1. `services/single_run_condition.py` + `routes/single_completed_history_route.py`
   — 먼저 "파싱 실패 시에도 SQL 원문에서 유래한 구분 가능값을 유지"하도록 지문 계산 로직을
   바꾼 뒤(예: canonical 실패 시 원문 sha256을 지문 구성요소에 포함), 두 파일을 함께 배포.
2. `routes/agg_diff_route.py`(2011행) + `routes/single_active_run_route.py` +
   `routes/single_exec_gate_route.py` — 3개 파일을 같은 배포로 묶는다(부분 배포 금지).

**Phase 3 — 로직 자체 변경이 선행돼야 안전(가장 신중해야 함)**
1. `services/workflow_stage_guard.py` — `if sql_hash and sql_hash != ctx.sql_hash` 를
   "None이면 SQL_CHANGED(또는 별도 UNVERIFIABLE 차단코드)"로 명시 처리하도록 먼저 바꾼다
   (falsy-skip 제거). 이 변경 자체가 워크플로 가드의 보안 성격상 회귀 테스트 비중이 커야 한다.
2. `services/batch_wrapper_result_store.py` + `services/group_current_freshness_service.py`
   — `sql_hash()`가 "파싱 실패"를 빈 문자열이 아닌 구분 가능한 sentinel로 반환하도록 먼저
   바꾸고, freshness 비교 쪽이 그 sentinel을 "판정불가"로 별도 처리하도록 함께 고친다.

**Phase 4 — 이번 지침 범위 밖, 우선순위 낮음(보류 권장)**
- `services/validation_result_store.py` / `services/validation_run/persistence_verifier.py`
  — 원문 미저장 구조상 이관 이득이 적고, 현재도 안전한 방향(CONFLICT→재실행)이라 급하지 않음.

**마지막 단계**: Phase 0~3 전부 이관 완료 + 회귀 테스트
(`samples/test_virtual_cases.py`, `samples/test_complex_cases.py`,
`tests/test_d9_1_sql_hash_single_source.py`, `tests/test_sql_duplicate_policy.py`,
`tests/test_batch_delta_rerun_freshness.py`, `tests/test_active_run_query_notice.py`,
`tests/test_rerun_exec_gate_reuse.py`, `tests/test_urgent_target_only_and_reuse.py`,
`tests/test_d7_12_trusted_pk.py`, `tests/test_d7_13_perf_decomp.py` 등 이번 조사에서 식별된
관련 테스트 전량) 통과 확인 후에만 `services/common/sql_hash.py`와 `services/sql_normalizer.py`
삭제. (`services/validation_run/sql_hash.py`는 계약이 다른 별개 모듈이라 이번 정리 대상이
아님 — 혼동 주의, 선행 조사 §1 각주와 동일.)

## 검증 결과

- 코드 수정 없음(지침 요구사항대로 조사만 수행) — 실행/테스트 대상 코드 변경이 없으므로
  자체 테스트(회귀 스위트) 실행 대상 아님.
- 위 10곳 각각의 판정은 실제 파일:행 코드를 직접 읽고 도출했으며(추측 없음), 성능 판단은
  이 저장소 자체에 이미 기록된 실측치(`parser/sqlglot_safe_parse.py:38-40,52-53`)를 근거로
  삼았다.

## 기대효과

- "완전 제거가 가능한가"에 대한 예/아니오를 넘어, 10곳 각각을 **독립 이관 가능(3곳)** /
  **쌍·그룹 동시배포 필수(5곳: 2개 그룹)** / **로직 변경 선행 필요(3곳: 사실상 2개 그룹)**
  로 세분화해, 다음 단계 실행 지침을 "한 번에 전부"가 아니라 리스크 낮은 순서(Phase
  1→2→3)로 쪼갤 근거를 확보함.
- `workflow_stage_guard.py`의 SQL_CHANGED 가드가 canonical_sql_hash로 단순 치환될 경우
  **조용히 무력화**된다는 구체적 회귀를, 실행 전에 코드 근거와 함께 확인해 사고를 예방함.
- `single_run_condition.py`/`agg_diff_route.py`+`single_active_run_route.py`+
  `single_exec_gate_route.py` 두 그룹이 "쓰기/읽기 알고리즘 불일치" 위험을 안고 있어
  부분 이관이 금지돼야 함을 확인해, 향후 실행 지침에서 "파일 단위"가 아니라 "그룹 단위"로
  배포 범위를 정하도록 하는 근거를 제공함.
- Phase 1(3곳)은 이미 이 저장소에 존재하는 None-safe 소비 관용구(`bool()` 가드,
  `or None`, `rawfallback:` 패턴)를 그대로 재사용하면 되어 **신규 설계 없이 즉시 착수
  가능한 저위험 착수점**임을 확인함.

작업명 : SQL-HASH-FULL-CONSOLIDATION-TO-CANONICAL-FEASIBILITY-CHECK
⚠️ 추가 작업 필요 - 10곳 전부 canonical_sql_hash 이전은 "가능"하나 전량 동시 무조건 안전은 아님(3개 지점은 None-처리 로직 변경 선행 필요, 2개 그룹은 여러 파일 동시배포 필수) — 코드 수정 없음(조사만)
```
