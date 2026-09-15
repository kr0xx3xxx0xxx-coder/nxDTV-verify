```text
작업명 : NON-PG-CONNECTION-POOLING-ORACLE-MSSQL-EXTEND-M371
✅ 작업 완료 - Oracle/MSSQL 커넥션 풀링을 PostgreSQL과 동일 원칙으로 확장(MySQL/MariaDB는 범위 밖, 신규 백로그 M383 등록)

## 목적
배치 공식 병렬 경로(services/batch/wrapper_parallel_runner.py)에서 Oracle/MSSQL
연결이 row(테이블)마다 신규 물리 연결을 반복 생성하는 구조적 비효율(M371, 기존
NON-PG-CONNECTION-POOLING-GAP-CONFIRM-M371 재조사에서 "조치 불필요"로 닫지 않고
"우선순위 중"으로 승격된 항목)을 해소한다. PostgreSQL은 이미 프로세스 단위 idle
풀(services/connection_pool.py)로 warm 재사용 중이나 Oracle/MSSQL은 이 풀에서
제외되어 있었다. MySQL/MariaDB는 별개 문제(connect() 자체 미구현)로 이번 범위에서
명시적으로 제외한다.

## 현상 (파트A — 설계 확인)
1. `services/connection_pool.py`의 `_is_pg(key)`가 `key[0] == "postgresql"`만
   True를 반환해 checkout()/return_conn() 양쪽에서 Oracle/MSSQL을 항상 풀 우회
   경로(매 checkout 신규 연결 → return 시 즉시 close)로 보냈다.
2. Oracle(`services/db_adapters/oracle.py`)·MSSQL(`services/db_adapters/mssql.py`)
   양쪽 다 실제 `connect()`가 이미 구현되어 있음을 확인했다(Oracle: oracledb thin
   우선/cx_Oracle fallback, DSN 구성 + 세션 NLS 고정 + NUMBER→Decimal
   outputtypehandler / MSSQL: pyodbc, ODBC Driver 17 DSN 문자열). 즉 "드라이버가
   지원하지 않아서"가 아니라 풀링 조건(`_is_pg`)이 PostgreSQL 하나로만 좁게
   설정돼 있던 것이 원인이었다(기존 조사 결론과 일치).
3. MySQL(`services/db_adapters/mysql.py`)·MariaDB(`mariadb.py`)는 `connect()`
   override 자체가 없어 `BaseDbmsAdapter.connect()`의 기본 `RuntimeError`로
   떨어진다 — 이 두 DBMS는 커넥션 풀링 이전에 라이브 접속 자체가 불가능한 별개
   문제이며, 지침에 따라 이번 작업에서 손대지 않았다.
4. DBMS별 재사용 안전성 검토 결과, 새로운 cleanup 로직이 필요하지 않다는 결론을
   내렸다(근거는 아래 "실제조치" 참고) — `_is_pg`를 `_is_pooled_dbms`로 조건만
   넓히는 것으로 충분하다.
5. 배치 병렬 경로(`wrapper_parallel_runner.py`)는 각 worker 스레드가
   `run_batch_row` 내부에서 `request_connection_scope()`(thread-local)를 열고
   `_cmn_db_connect` → `connection_pool.checkout/return_conn`을 그대로 타는
   구조임을 재확인했다 — 이 모듈 자체는 무수정으로 풀 확장 혜택을 자동으로 받는다.

## 실제조치 (파트B — 구현)
- `services/connection_pool.py`
  - `_is_pg(key)` → `_is_pooled_dbms(key)`로 일반화, `_POOLED_DB_TYPES =
    frozenset({"postgresql", "oracle", "mssql"})`로 대상 확장. checkout()/
    return_conn() 양쪽 호출부 갱신.
  - 풀 크기/재사용 정책(`POOL_MAX_IDLE_PER_KEY=4`, `POOL_MAX_LIFETIME_S=300`,
    `POOL_IDLE_TIMEOUT_S=120`, `POOL_PING_AFTER_S=5`)은 기존 PostgreSQL 설정
    그대로 재사용했다 — DBMS별 별도 정책을 새로 만들지 않았다.
  - 모듈 docstring에 DBMS별 재사용 안전성 근거를 명시:
    - checkout마다 이미 적용되던 `autocommit=True`·`apply_query_timeout`
      재적용(db_query_service._cmn_db_connect)이 DBMS 중립이라 Oracle
      (`connection.call_timeout`)/MSSQL(`connection.timeout`) 모두 추가
      분기 없이 그대로 재사용된다.
    - `return_conn()`의 `rollback()`은 psycopg2/oracledb·cx_Oracle/pyodbc
      공통 DB-API 메서드라 그대로 재사용된다.
    - Oracle 세션 파라미터 고정(ALTER SESSION NLS_NUMERIC_CHARACTERS/
      NLS_COMP, NUMBER→Decimal outputtypehandler)은 물리 연결(=세션) 생성
      1회만 적용하면 된다 — 이 풀은 같은 물리 커넥션 객체를 여러 논리
      checkout에 걸쳐 재사용할 뿐 서버측 세션을 새로 만들지 않으므로(DRCP
      아님) 재적용이 불필요하다.
    - `_alive()`의 `getattr(real, "closed", 0)` 검사는 psycopg2 전용이라
      Oracle/MSSQL은 사실상 idle 기반 SELECT 1 ping에만 의존해 끊긴 연결을
      감지한다 — PostgreSQL 대비 알려진 비대칭으로 문서화만 하고(구조적
      문제점 참고) 별도 드라이버별 분기는 이번 범위에서 추가하지 않았다.
- `services/db_adapters/oracle.py`
  - `_pin_session_nls_numeric()` docstring의 "훗날 오라클 풀링을 켠다면 checkout
    경로에 재적용을 걸어야 한다"는 예측성 주석이 이제 stale해져, 풀링이 실제로
    켜진 뒤에도 재적용이 불필요한 이유(같은 물리 연결 재사용, 서버측 세션은
    다시 만들지 않음)로 정정했다.
- `tests/test_connection_pool.py`
  - 기존 `test_non_pg_bypasses_pool`(Oracle 대상)을 `test_mysql_bypasses_pool`로
    교체(Oracle은 이제 풀링 대상이므로 우회 테스트 대상에서 제외).
  - `test_oracle_warm_reuse`, `test_mssql_warm_reuse`,
    `test_oracle_mssql_postgresql_pools_separated` 신규 추가.

### 구조적 문제점 / 운영상 위험 (비판적 검토)
- **closed-속성 비대칭**: `_alive()`의 closed 체크가 Oracle/MSSQL 커넥션 객체엔
  해당 속성이 없어 사실상 no-op이고, idle 5초 미만 구간에 서버가 끊은 연결은
  다음 실제 쿼리 실행 시점에야 오류로 드러난다. 이번 범위에선 추가 조치하지
  않았고(directive가 요구한 최소 변경 지점에는 포함되지 않음), 실제 운영에서
  이 경로로 인한 오탐이 관측되면 후속 이슈로 판단이 필요하다.
- **MSSQL은 코드만 확장·실측은 단위테스트로 대체**: 아래 검증 결과 참고.
- **explainability**: 새 heuristic/scoring 요소는 도입하지 않았다(순수 조건
  확장 + 기존 정책 재사용) — 이 항목은 해당 없음.

## 기대효과
- 배치 공식 병렬 경로에서 Oracle/MSSQL 대상 테이블이 많을수록 누적되던 접속
  협상(TNS/TDS) 비용이 제거된다 — 실측(아래)에서 물리 연결 생성 수가 처리
  row 수보다 적음을 확인했다.
- PostgreSQL 기존 동작·정책은 완전히 무변경이며 무회귀 확인했다.
- MySQL/MariaDB는 이번 변경으로 인한 영향이 전혀 없다(조건 미해당 → 기존
  분기 그대로 유지).

## 검증 결과
### 회귀 테스트
- `tests/test_connection_pool.py`: 12 passed (기존 8건 + 신규 4건, 0 failed)
  - `test_mysql_bypasses_pool`, `test_oracle_warm_reuse`, `test_mssql_warm_reuse`,
    `test_oracle_mssql_postgresql_pools_separated` 신규.
- `python samples/test_virtual_cases.py` → 8/8 통과(회귀 없음, 본 작업과 무관한
  파서/분류기 영역이라 영향 없을 것으로 예상했고 실측으로 확인).
- `python samples/test_complex_cases.py` → 5/5 통과(회귀 없음).

### 실측 E2E (11번 규칙)
- **Oracle(라이브, Oracle_asis 프리셋 192.168.0.151:1523, oracledb 4.0.2 thin
  모드)**:
  - 순차 시나리오: 같은 profile로 5회 checkout/return →
    `physical_open=1, reuse=4`(5개 row가 물리 연결 1개를 공유).
  - 스레드 병렬 시나리오(`request_connection_scope` 기반, wrapper_parallel_runner
    와 동일 패턴, 동시성 3·row 6개): `physical_open=3, reuse=3` — 테이블 수(6)보다
    적은 물리 연결(3)로 처리됨을 확인.
  ```
  [순차] {'physical_open': 1, 'checkout': 5, 'reuse': 4, 'returned': 5, 'closed': 0, ...}
  [병렬] {'physical_open': 3, 'checkout': 6, 'reuse': 3, 'returned': 6, 'closed': 0, ...}
  ```
- **PostgreSQL 무회귀(라이브, PostgreSQL_asis / Neon)**: 순차 4회 checkout/return
  → `physical_open=1, reuse=3` — 기존 동작과 동일(회귀 없음).
- **MSSQL — 실측 상태 아님(사유 명시)**: 이 환경에 `pyodbc` 패키지가 설치돼
  있지 않고(`ModuleNotFoundError`), DB 프리셋 저장소(`dnp_db_preset.db`)에도
  MSSQL 프리셋이 등록돼 있지 않아 실 SQL Server 인스턴스 접속 자체가
  불가능했다. 대체수단으로 `test_mssql_warm_reuse`(fake 커넥션 객체 기반
  단위테스트)를 사용해 코드 경로(`_is_pooled_dbms`가 mssql을 포함해 동일
  풀 메커니즘을 타는지)만 검증했다 — **물리 pyodbc/ODBC 드라이버를 통한
  실제 SQL Server 접속·세션 재사용은 검증하지 못했다.** MSSQL 실 인스턴스가
  확보되면 재검증을 권장한다(백로그에 별도 기록하지 않고 이 보고서에만 명시
  — 코드는 이미 Oracle과 동일 조건으로 확장 완료 상태).

### 코드 diff 증적
```diff
diff --git a/services/connection_pool.py b/services/connection_pool.py
index bf6ec90f..8d2da708 100644
--- a/services/connection_pool.py
+++ b/services/connection_pool.py
@@ -2,17 +2,38 @@
 """
 services/connection_pool.py
 INDIVIDUAL-VALIDATION-CROSS-REQUEST-CONNECTION-POOL +
-CONNECTION-POOL-PROFILE-IDENTITY-ISOLATION — 프로세스 단위 PostgreSQL 연결 풀(매니저).
+CONNECTION-POOL-PROFILE-IDENTITY-ISOLATION +
+NON-PG-CONNECTION-POOLING-ORACLE-MSSQL-EXTEND-M371 — 프로세스 단위 연결 풀(매니저).
 
 파이프라인상 위치: db_query_service 의 물리 연결 생성 경계 아래.
 역할:
   - request_connection_scope 가 요청 종료 시 실제 연결을 close 하던 것을, 풀에 '반납'하도록 바꿔
-    다음 요청(warm)이 SSL 핸드셰이크 없이 기존 물리 연결을 재사용하게 한다(Neon 원격 연결 비용 제거).
+    다음 요청(warm)이 SSL 핸드셰이크/TNS·TDS 접속 협상 없이 기존 물리 연결을 재사용하게 한다
+    (Neon 원격 연결 비용 제거 — Oracle/MSSQL 도 동일 이유로 M371 에서 풀링 대상에 포함).
   - 풀 key = db_query_service._conn_cache_key = (db_type, host, port, dbname, user, fingerprint).
     fingerprint 는 비밀번호·profile 식별자를 단방향 해시로 반영하므로, 같은 host/db/user 라도 비밀번호가
     다르면(=다른 profile) 별도 연결로 격리된다(잘못된 암호 profile 이 다른 profile 의 warm 연결을
     재사용하지 못한다). 비밀번호 원문은 key 에 없고 별도 복제·로그도 하지 않는다.
-  - PostgreSQL 만 풀링한다. 그 외 DB(MySQL/Oracle/MSSQL)는 기존처럼 매번 생성·종료(HOLD).
+  - PostgreSQL/Oracle/MSSQL 을 풀링한다(_is_pooled_dbms). MySQL/MariaDB 는 db_adapters 에 connect() 가
+    구현되어 있지 않아(별도 문제, 이번 범위 밖) 애초에 라이브 실행이 불가능하므로 손대지 않고 기존처럼
+    매번 생성·종료(HOLD) 경로를 그대로 둔다 — 풀 확장은 순수 조건 확장이며 그 외 DB 는 무영향이다.
+
+DBMS 별 재사용 안전성(M371 파트A 설계 확인 — 새 cleanup 로직 불필요, 근거):
+  (... 상세 근거는 파일 docstring 전문 참고 ...)
 
 profile 무효화(generation):
   - invalidate_profile(db_info)는 해당 key 의 idle 연결을 close 하고 key generation 을 증가시킨다.
@@ -91,8 +112,14 @@ def _key(db_info: dict) -> tuple:
     return _conn_cache_key(db_info)
 
 
-def _is_pg(key: tuple) -> bool:
-    return bool(key) and key[0] == "postgresql"
+_POOLED_DB_TYPES = frozenset({"postgresql", "oracle", "mssql"})
+
+
+def _is_pooled_dbms(key: tuple) -> bool:
+    """이 key(db_type 포함)가 풀링 대상 DBMS 인지 — PostgreSQL/Oracle/MSSQL 만 True(M371)."""
+    return bool(key) and key[0] in _POOLED_DB_TYPES
 
 
 def _safe_close(real) -> None:
@@ -134,11 +161,12 @@ def _alive(p: _Pooled, key: tuple) -> bool:
 def checkout(db_info: dict):
     key = _key(db_info)
-    if not _is_pg(key):
+    if not _is_pooled_dbms(key):
         from services.db_query_service import _open_real_connection
@@ -178,7 +206,7 @@ def checkout(db_info: dict):
 def return_conn(key_or_db_info, real, broken: bool = False) -> None:
     ...
-    if broken or not _is_pg(key):
+    if broken or not _is_pooled_dbms(key):
         _safe_close(real)

diff --git a/services/db_adapters/oracle.py b/services/db_adapters/oracle.py
index 148b7a68..7c0889ff 100644
--- a/services/db_adapters/oracle.py
+++ b/services/db_adapters/oracle.py
@@ -261,13 +261,17 @@ class OracleAdapter(BaseDbmsAdapter):
-        연결 시점 1회만 실행하는 이유(풀 재사용 검토 결과):
-          services/connection_pool.py 는 `_is_pg(key)` 로 **PostgreSQL 만** 풀링하고 오라클은 checkout
-          마다 새 물리 연결을 만들고 반납 시 close 한다(비-PG 는 풀 우회). ...
-          여기가 아니라 checkout 경로에 재적용을 걸어야 한다(DRCP 세션 purity 도 함께 검토 필요).
+        연결 시점 1회만 실행하는 이유(NON-PG-CONNECTION-POOLING-ORACLE-MSSQL-EXTEND-M371 재검토 결과):
+          services/connection_pool.py 는 `_is_pooled_dbms(key)` 로 PostgreSQL/Oracle/MSSQL 을 풀링한다
+          (... 재사용 시점(connection_pool.checkout)에 재적용할 필요가 없다는 근거 상세 ...)

diff --git a/tests/test_connection_pool.py b/tests/test_connection_pool.py
index 63f53f37..a51fa4f7 100644
(신규 헬퍼 _oracle/_mssql/_mysql, test_mysql_bypasses_pool 로 교체,
 test_oracle_warm_reuse / test_mssql_warm_reuse /
 test_oracle_mssql_postgresql_pools_separated 신규 3건 추가 — 총 +63줄/-9줄)
```
(전체 diff 106 insertions / 21 deletions, 3파일 — 위는 요약 발췌. 전문은 코드
저장소 커밋 `fb9d9d93`에서 `git show fb9d9d93` 로 확인 가능)

### 코드 저장소 커밋
- `fb9d9d93` — `feat(connection-pool): Oracle/MSSQL 커넥션 풀링 확장(M371)`
  (nxDTV-src, origin/main 반영 확인)

### 백로그 갱신
- `BACKLOG.md` M371: "조사완료(승격 - 우선순위 중)" → "✅ 해결 완료(2026-09-15,
  Oracle/MSSQL만 · MySQL/MariaDB는 별도 잔여 M383)"로 갱신, 해결 요약/실측/잔여
  사항 본문 추가.
- `BACKLOG.md` M383 신규 등록: "MYSQL-MARIADB-CONNECT-NOT-IMPLEMENTED" —
  MySQL/MariaDB connect() 미구현(F23 잔여 노트와 동일 사실, 가시성을 위한
  재등록). 코드 변경 없음(지침 명시대로 손대지 않음).
- nxDTV-verify 저장소 커밋+push 완료.

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 (순수 백엔드 커넥션
  풀링 로직 — 화면 요소(버튼/체크박스/텍스트/배지 등)를 코드로 건드린
  부분 없음)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(UI 변경
  없음 — 단, 위 "검증 결과"에 MSSQL 라이브 E2E 불가 사유와 대체수단은
  별도로 명시함)

작업명 : NON-PG-CONNECTION-POOLING-ORACLE-MSSQL-EXTEND-M371
✅ 작업 완료 - Oracle/MSSQL 커넥션 풀링을 PostgreSQL과 동일 원칙으로 확장(MySQL/MariaDB는 범위 밖, 신규 백로그 M383 등록)
```
