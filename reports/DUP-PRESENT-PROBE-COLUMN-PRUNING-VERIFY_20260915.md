```text
작업명 : DUP-PRESENT-PROBE-COLUMN-PRUNING-VERIFY
✅ 작업 완료 - _dup_present() 중복 probe SQL은 실측상 불필요 컬럼 계산이 전혀 발생하지 않음을 확인(코드 자체가 서브쿼리도 안 만들고, DB 옵티마이저도 100% 가지치기)

## 목적
services/diagnosis/match_key_evidence.py 의 _dup_present() 가 원본 이관 SQL(JOIN
포함, 여러 컬럼 SELECT)을 COUNT(*)/GROUP BY/HAVING 로 감싸 중복 여부를 확인할 때,
원본 SQL의 SELECT 목록 중 실제로 안 쓰는 컬럼(plain 컬럼, 무거운 표현식, LOB 등)
까지 DB가 읽거나 계산하는지 — DB 옵티마이저의 컬럼 가지치기(column pruning)가
실제로 일어나는지를 코드 레벨 분석 + Oracle/PostgreSQL 실 DB EXPLAIN 실측으로
확인한다. 코드 수정은 지침에 의해 금지되어 있으며, 본 작업은 조사·실측만 수행했다.

## 현상 (분석 결과)

### A. 코드 레벨 확인 — 애초에 서브쿼리조차 만들어지지 않는다
_dup_present() 의 실제 동작을 그대로 재현해보면, 사용자가 우려한 "SELECT COUNT(*)
FROM (원본SQL) GROUP BY ..." 형태의 서브쿼리 wrapping은 실제 코드에서 일어나지
않는다. 코드는 sqlglot으로 파싱한 base_sql의 최상위 SELECT AST를 그대로 대상으로
tree.set("expressions", [COUNT(*) AS C]) 를 호출해 SELECT 목록 자체를 치환하고,
group/having/limit 만 덧붙인다(match_key_evidence.py 24~42줄). 즉 FROM/JOIN/WHERE
는 유지되지만 원본 SELECT 목록(plain_col, heavy_col 등)은 최종적으로 DB에 보내지는
SQL 텍스트에서 통째로 사라진다. 실제 재현 결과:

  입력(원본 이관 SQL 예시):
    SELECT t1.pk_col, t1.plain_col, UPPER(t1.heavy_lob_col) AS disp, t2.other_col
    FROM t1 JOIN t2 ON t1.fk_col = t2.fk_col WHERE t1.plain_col > 10

  _dup_present() 가 실제로 만들어 실행하는 SQL:
    SELECT COUNT(*) AS C FROM t1 JOIN t2 ON t1.fk_col = t2.fk_col
    WHERE t1.plain_col > 10 GROUP BY t1.pk_col HAVING COUNT(*) > 1 LIMIT 1

  plain_col(SELECT 목록으로서의 값), UPPER(heavy_lob_col), t2.other_col 은 SQL
  텍스트 단계에서 이미 제거되어 DB로 전달조차 되지 않는다. 옵티마이저 가지치기에
  기댈 필요가 없는 구조다.

### B. "만약 서브쿼리로 감쌌다면" 최악 가정을 실 DB로 실측(지침 요청 원형)
지침이 요청한 정확한 형태(SELECT COUNT(*) FROM (안쓰는 컬럼 포함 원본SELECT) ...
GROUP BY ... HAVING ...)를 A항목보다 불리한 최악 가정으로 별도 실측했다. Oracle AI
Database 26ai Free(23.26.1.0.0) / PostgreSQL 17.11(Neon, ap-southeast-1)에 ZZPRB_
접두 임시 테이블(PostgreSQL 30만건·PK 25%중복=중복그룹75,000건, Oracle 10만건·
중복그룹25,000건)을 만들어 EXPLAIN(PostgreSQL: ANALYZE,VERBOSE,BUFFERS / Oracle:
DBMS_XPLAN.DISPLAY format=ALL, Column Projection Information 포함)으로 3케이스
(단순컬럼참조/무거운표현식/LOB)를 각각 "베이스라인(안쓰는 컬럼 포함 SELECT)" vs
"가지치기판(PK만 SELECT)"으로 비교했다(각 3회 실행 median).

[PostgreSQL — cost/rows/buffers 실측 요약]

  케이스             베이스라인 buffers  가지치기판 buffers  실행시간(median, 베이스라인/가지치기)
  (a)단순컬럼        shared hit=7335     shared hit=7335     304.8ms / 266.2ms (노이즈 수준)
  (b)무거운표현식    shared hit=7335     shared hit=7335     Execution Time 약 240ms대, 동일 수준
  (c)LOB(TOAST106MB) shared hit=633      shared hit=633      26.3ms / 25.5ms

  (c) 핵심 증거: HEAVY_COL을 STORAGE EXTERNAL로 강제해 106MB TOAST 세그먼트를
  실제로 만들었음에도, 베이스라인/가지치기판 모두 buffers=633 블록(≈5MB, heap+t2
  분량)만 읽힘 — TOAST 블록은 단 1개도 읽지 않음.
  대조군(같은 컬럼을 실제로 SELECT/COUNT에 사용): 무거운 표현식(md5(repeat(...)))
  사용 시 99.0배, LOB length() 사용 시 20.0배 실행시간 증가 — 측정 감도는 충분히
  확보되어 있고, (a)(b)(c)에서 차이가 안 난 것은 "정말로 안 읽었기 때문"임을 뒷받침.

[Oracle — Column Projection Information 실측 요약]

  케이스           베이스라인 Plan hash  가지치기판 Plan hash  Bytes(FULL SCAN)          실행시간(median)
  (a)단순컬럼      2676681376            동일                  878K(동일)                34.8ms/17.6ms(LAN노이즈)
  (b)무거운표현식  2676681376            동일                  878K(동일)                37.5ms/22.7ms
  (c)LOB(112.3MB)  2702024361            동일                  96000(동일,LOB제외크기)   6.7ms/4.7ms

  Column Projection Information(각 연산 단계에서 실제 프로젝션하는 컬럼 목록)이
  베이스라인/가지치기판 사이에 "T1"."PK_COL","T1"."FK_COL" 두 개로 완전히 동일 —
  PLAIN_COL/무거운표현식/HEAVY_COL(LOB)이 어느 단계에도 등장하지 않음.
  대조군: 같은 무거운 표현식(RAWTOHEX(STANDARD_HASH(...,'SHA512')))을 실제 COUNT
  대상으로 쓰면 412배, LOB DBMS_LOB.GETLENGTH() 사용 시 9.3배 실행시간 증가 — 역시
  측정 감도 충분.

## 실제조치
코드 수정 없음(지침 명시 사항 — 조사·실측만 수행). 실험에 사용한 임시 테이블
(PostgreSQL zzprb_t1/t2/t4, Oracle ZZPRB_T1/T2/T3)은 실측 종료 직후 DROP(Oracle은
PURGE 포함)하여 정리했고, 재조회로 양쪽 DB 모두 잔존 0건을 확인했다(Oracle은
USERS 테이블스페이스 여유 345MB로 원복 확인). 프로젝트 소스 파일 변경도 git status
--porcelain 기준 0건임을 확인했다(30번 규칙 — 애초에 diff 증적 대상 자체가 없음).

## 검증 결과
1~4번 근거: 위 "현상" A/B 항목의 코드 재현 결과·EXPLAIN 발췌·수치가 곧 근거다.

5번 결론: "옵티마이저가 이미 알아서 잘 처리해서 걱정할 필요 없다" 쪽으로 명확히
판정한다. 근거:
  - (더 근본적으로) 실제 코드는 서브쿼리조차 만들지 않고 SELECT 목록을 텍스트
    단계에서 통째로 치환하므로, 애초에 옵티마이저 가지치기에 기댈 필요가 없다.
  - 지침이 요청한 "서브쿼리로 감싸는 최악 가정" 하에서도, Oracle 23ai/PostgreSQL 17
    모두 단순컬럼·무거운 표현식(해시/문자열 연산)·out-of-line LOB 세 경우 전부에서
    Plan hash(Oracle)/cost·buffers(PostgreSQL)가 베이스라인과 100% 동일했고, 대조군
    (실제 그 컬럼을 사용)과 비교해 9~412배 차이가 나는 것으로 측정 감도가 충분함을
    함께 확인했다(=차이가 안 난 것이 측정 한계 때문이 아니라는 근거).
  - 3번 질문(PK만 SELECT하도록 sqlglot으로 재작성하는 방법의 기술적 가능성)은
    tree.set("expressions", [k.copy() for k in keys]) 한 줄로 trivial하게 가능함을
    코드 레벨로 확인했으나(_dup_present 자체가 이미 이 패턴을 COUNT(*) 치환에 쓰고
    있음), 실측 결과 옵티마이저가 이미 완전 가지치기하므로 추가 구현의 실익은 없다.
  - 단, 본 결론은 sqlglot이 base_sql을 정상 파싱해 GROUP BY/HAVING 형태로 변환에
    성공하고, 서브쿼리 평탄화(view merging/subquery pull-up)를 막는 요소(DISTINCT,
    집계, 윈도우함수, ROWNUM 등)가 원본 SQL에 없는 경우에 한정된다 — 이는 실측 범위
    밖이며, 그런 요소가 섞인 원본 SQL까지 보장하려면 별도 확인이 필요하다.

## 기대효과
_dup_present()의 현재 구현(AST 레벨 SELECT 목록 치환)이 이미 최선에 가까운 구조
임을 코드 분석 + 실 DB 실측 양쪽으로 확인했으므로, 추가 최적화(PK-only rewrite)
작업 없이 현재 구조를 그대로 유지해도 된다는 근거를 확보했다 — 불필요한 코드 변경
과 그에 따르는 리스크(1~6단계 완료 모듈 임의 수정 금지 원칙과도 상충)를 예방했다.

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 (순수 DB EXPLAIN/실행시간 실측
  조사이며, 화면 요소를 코드로 수정한 바 없음)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(실 DB 2종 모두 실측 성공)

[30번 규칙 — 소스 diff 증적]
코드 수정이 전혀 없는 순수 조사 지침이므로 diff 증적 대상 아님. git status
--porcelain 기준 프로젝트 소스(services/parser/analyzer/validator/generator/
checker/ui/config/routes/web_server.py/samples) 변경 0건.

부록 — 실측에 사용한 임시 스크립트/원문 로그(에이전트 세션 scratchpad, 로컬 보관):
probe_pg.py, probe_pg2.py, probe_ora.py, probe_ora2.py, ora_space.py / 결과 원문
pg_result.txt(693줄), pg_result2.txt(237줄), ora_result3.txt(730줄)

작업명 : DUP-PRESENT-PROBE-COLUMN-PRUNING-VERIFY
✅ 작업 완료 - _dup_present() 중복 probe SQL은 실측상 불필요 컬럼 계산이 전혀 발생하지 않음을 확인(코드 자체가 서브쿼리도 안 만들고, DB 옵티마이저도 100% 가지치기)
```
