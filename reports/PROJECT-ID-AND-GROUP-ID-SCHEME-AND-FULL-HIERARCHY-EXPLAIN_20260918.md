작업명 : PROJECT-ID-AND-GROUP-ID-SCHEME-AND-FULL-HIERARCHY-EXPLAIN
✅ 작업 완료 - project_id/group_id 키 체계 근거 확인 + 일괄/개별검증 전체 계층구조를 코드·DB 조회로 규명(코드 수정 없음, 조사 전용)

## 목적
사용자 질문 2가지에 코드/DB 근거로 답한다.
1) 왜 `project_id`가 2부터 시작하는지, `group_id`는 왜 순번이 아니라 긴 문자열인지.
2) 일괄검증(프로젝트→그룹→배치→쿼리)과 개별검증(프로젝트→그룹→?→쿼리)의 실제 계층 구조 —
   특히 개별검증 쪽 "?"에 해당하는 실제 엔티티 규명. 코드 수정은 하지 않고 조사만 수행했다.

## 현상 (조사 대상 원질문)
- `DTV_project.project_id`가 1이 아니라 2부터 존재 — AUTOINCREMENT 시작값 이상 현상으로 보임.
- `group_id`가 `GRP_202609010933000433` 같은 긴 문자열 — 순번 정수가 아닌 이유 불명.
- 일괄검증은 "프로젝트→그룹→배치→쿼리" 4단계로 보이는데, 개별검증도 같은 단계 수인지,
  같다면 "그룹과 쿼리 사이"에 해당하는 엔티티가 무엇인지 불명.

## 실제조치 (조사 방법 및 근거)

### 1. `project_id`가 2부터 시작하는 이유
- 스키마: `DTV_project.project_id INTEGER PRIMARY KEY AUTOINCREMENT`
  (DB 조회 결과 `CREATE TABLE "DTV_project" (project_id INTEGER PRIMARY KEY AUTOINCREMENT, ...)`,
  `services/project_store.py:100`의 `ensure_schema()` 정의와 일치)
- **DB 조회 결과**: 현재 DB(`db/migration_validator.db`)와 이 저장소에서 가장 오래된 백업
  (`db/migration_validator.db.bak_20260914_184810_pre_test_group_1013_cleanup`) 모두에서
  `project_id=1` 행은 활성/소프트삭제 어느 쪽으로도 **한 번도 존재하지 않음**을 확인했다.
  가장 이른 행은 `project_id=2`(`created_at='2026-06-17 02:43:04'`)이고, `sqlite_sequence`의
  현재 카운터는 1044(다음 신규 프로젝트는 1039~번대로 배정될 것).
- 테이블 자체는 2026-06-13 커밋 `374be77b`(feat(ui): project-first left-menu restructure)에서
  테이블명 `project`로 최초 생성됐다(`git show 374be77b -- services/project_store.py`, 당시
  파일은 `services/project_store.py:22`에 해당). **이 최초 버전에는 생성(create)/목록(list)
  API만 있고 삭제 API가 없었다** — soft delete(`is_deleted`/`deleted_at`)는 6일 뒤 커밋
  `84df47f0`(2026-06-19, "feat(project): 프로젝트 목록·전역 작업 프로젝트 선택기 운영
  보강(soft delete·모달·수정)")에서야 추가됐다. `project_id=2`의 생성 시각(6/17)은 정확히
  이 "테이블 신설(6/13)~soft delete 도입(6/19)" 사이 구간이다.
  (참고: 이후 리팩터링(`bd14987f`, SQLITE-TABLE-RENAME-EXECUTE-BY-CATEGORY-SEQUENTIAL)에서
  테이블명이 `project`→`DTV_project`로 변경됐을 뿐 PK 체계는 그대로다.)
- `db/migration_validator.db`는 `.gitignore:43,46`(`*.db`, `db/*.db`)에 의해 버전관리 대상이
  아니므로, `project_id=1` 행이 "무엇이었고 언제 지워졌는지"는 git 이력으로 직접 추적이
  **불가능**하다. 이는 확정 사실의 한계이며, 아래 결론은 정황 근거 기반 추정임을 명시한다.
- **결론(추정, 확정 아님)**: SQLite AUTOINCREMENT는 성공적으로 커밋된 INSERT에서만
  시퀀스를 전진시키며 한 번 소비된 번호는 재사용하지 않는다. `project_id=1`은 테이블이
  막 만들어진 직후(soft delete·백업 체계가 아직 없던 시점, 6/13~6/17 사이) 생성됐던 테스트성
  또는 초기 확인용 1건이었을 가능성이 높고, 당시 앱에는 삭제 API 자체가 없었으므로 DB 파일을
  직접 조작(hard delete)해 지워졌을 것으로 추정된다. "왜/누가 지웠는지"의 직접 증거(로그·git)는
  남아있지 않다.

### 2. `group_id`가 `GRP_{timestamp}` 형식인 이유 + 다른 엔티티와의 일관성
- 생성 코드: `services/batch_group_service.py:462`
  ```
  group_id = f"GRP_{datetime.now(tz=timezone.utc).strftime('%Y%m%d%H%M%S%f')[:18]}"
  ```
  (스키마: `DTV_validation_batch_group.group_id TEXT PRIMARY KEY` — 정수 AUTOINCREMENT가 아니라
  타임스탬프 기반 문자열 PK)
- **직접 근거(명시적 주석)**: 동일한 타임스탬프 문자열 방식이 배치 업로드 ID(`batch_id`,
  `DTV_mv_batch_run` PK)에도 그대로 쓰이며, 그 생성부 바로 위 주석에 설계 의도가 명시돼 있다.
  `routes/batch_route.py:950~953`
  ```
  # batch_id: 초 단위(%Y%m%d%H%M%S)에 마이크로초 6자리를 더해 같은 초 내 다건 업로드 충돌 방지.
  # (DTV_mv_batch_run PK·파일 경로 키. 같은 초 재제출 시 INSERT OR REPLACE 로 이력이
  #  덮어써지던 문제 해소.)
  batch_id = _dt_bid.now().strftime('%Y%m%d%H%M%S%f')
  ```
  즉 "동시/연속 생성 시 충돌(race) 방지"가 명시된 설계 목적이다. `group_id`가 처음 도입된 커밋
  (`32364712`, 2026-05-26, "feat(batch): add validation job groups and batch run browser")
  자체에는 그런 주석이 없었지만, 같은 작성자가 이후 같은 패턴을 `batch_id`에도 적용하면서
  이유를 명문화했으므로 두 ID 모두 동일한 설계 의도(락/시퀀스 테이블 경합 없이 유일성 보장)로
  보는 것이 타당하다.
- **다른 엔티티 일관성 확인 결과** — 키 체계는 실제로 3가지 방식이 혼재한다(무작위가 아니라
  테이블 성격별로 일관됨):
  | 방식 | 대상 | 근거 |
  |---|---|---|
  | ①INTEGER AUTOINCREMENT | `DTV_project.project_id`, `DTV_mv_upload_row_result.id`, `DTV_mv_batch_result.result_id` | 단순 참조용 내부 PK, 동시 생성 빈도가 낮거나 단일 트랜잭션 내 순차 생성 |
  | ②`prefix + %Y%m%d%H%M%S%f` 문자열 | `group_id`(`GRP_`+18자), `batch_id`/`batch_run_id`(prefix 없음, 20자) | 사용자가 브라우저에서 수시로·동시에 생성 트리거 가능(그룹 생성 버튼, 배치 업로드) — 시퀀스 테이블 없이 유일성 확보(근거: `routes/batch_route.py:950~953` 주석) |
  | ③UUID4 | `DTV_validation_execution_run.run_id`, 개별검증 `session_id`(`services/batch_runner.py:996,1319`) | 실행 단위 식별자, 분산/재시도 시나리오에서도 충돌 없는 전역 유일값 필요 |

### 3. 전체 계층 구조 확인
DB 스키마 조회(`sqlite_master`) + 코드 추적 결과, 두 경로는 **이름만 다른 게 아니라 실제
계층 단계 수 자체가 다르다** — 일괄검증 4단계, 개별검증 3단계.

**일괄검증(배치) 경로 — 4단계**
```
DTV_project (project_id: INTEGER)
  └─ DTV_validation_batch_group (group_id: TEXT "GRP_..", project_id 컬럼으로 FK)
       └─ DTV_mv_batch_run (batch_run_id: TEXT, group_id 컬럼으로 FK)   ← 업로드 1회
            ├─ DTV_mv_upload_row_result (id: INT AUTOINCREMENT, batch_id FK,
            │     project_id·group_id 중복 보관)   ← row 1건 = 목적지테이블별 이관쿼리 1건
            └─ DTV_mv_batch_result (result_id: INT AUTOINCREMENT, batch_id FK, row_num)
                  ← 같은 row의 COUNT/SUM 검증 결과
```
- `DTV_validation_batch_group` 스키마: `group_id TEXT PRIMARY KEY ... project_id TEXT ...`(DB 조회)
- `DTV_mv_batch_run` 스키마: `batch_run_id TEXT PRIMARY KEY, group_id TEXT NOT NULL DEFAULT ''`(DB 조회)
- `DTV_mv_upload_row_result` 스키마: `id INTEGER PRIMARY KEY AUTOINCREMENT, batch_id TEXT NOT NULL,
  project_id TEXT, group_id TEXT`(DB 조회) — batch 하위지만 project_id/group_id를 비정규화 보관해
  화면에서 배치를 거치지 않고도 프로젝트/그룹 단위로 바로 필터링 가능하게 함.
- **DB 실측**: `DTV_mv_batch_run` 10,044건(최신 `2026-09-18T07:26:26Z`)로 현재 라이브 경로임을
  확인. 병행 구조로 `DTV_batch_run`(802건, 최신 `2026-09-06`)/`DTV_batch_item`도 존재하나
  `routes/batch_route.py`에서 `services.batch_runner`의 `legacy_batch_exec_disabled`를 명시적으로
  import해 쓰는 것으로 보아 **레거시(비활성)** 경로로 판단된다 — 현재 화면이 쓰는 라이브 4단계는
  위 `DTV_mv_*` 계열이다.

**개별검증 경로 — 3단계 (배치 레이어 자체가 없음)**
```
DTV_project (project_id: INTEGER)
  └─ (선택, None 가능) group_id
       └─ DTV_validation_execution_run (run_id: TEXT uuid4, project_id·group_id 컬럼 직접 보유,
             batch_id 컬럼은 존재하나 항상 NULL)   ← 쿼리 1건의 실행 단위 = 최종 엔티티
```
- `DTV_validation_execution_run` 스키마(DB 조회): `run_id TEXT PRIMARY KEY, execution_id TEXT,
  project_id TEXT, group_id TEXT, ... batch_id TEXT, ...`
- **DB 실측**: 최근 실행 8건 샘플 모두 `run_id`는 uuid4 형식이고 `group_id`는 `GRP_...`(그룹
  생성 시 만든 값 그대로) 값이 채워져 있으며, `batch_id`는 전체 147건 중 **0건**이 비어있지
  않음(=전부 NULL) — 개별검증 경로에는 "배치(업로드 세트)" 레이어가 아예 존재하지 않는다.
- `group_id`가 그룹 없이도 동작 가능함은 코드로도 확인:
  `services/validation_run/execution_context.py:54,149` — `group_id: Optional[str] = None`
  (그룹 미선택 개별검증도 허용된다는 뜻).
- `validation_target_id` 컬럼(같은 테이블)은 계층 엔티티가 아니라 "공식 등록 재사용"
  기능(`services/single_official_register_service.py`, `services/execution_reuse_lookup.py`)에서
  `DTV_policy_target_table_config.id`를 참조하는 별도 목적 FK다 — 사용자가 물은 "그룹과 쿼리
  사이의 ?"에 해당하는 계층 엔티티가 아니다(실측: 147건 중 21건만 값 존재, 나머지는 NULL).

**결론**: 사용자 가설("일괄검증은 프로젝트→그룹→배치→쿼리, 개별검증은 프로젝트→그룹→?→쿼리")은
절반만 맞다. 앞 두 단계(프로젝트→그룹)는 동일하지만, 개별검증에는 "?"에 해당하는 **별도
엔티티가 없다** — `group_id`가 실행 row(`DTV_validation_execution_run`)에 컬럼으로 직접 붙어
그룹이 곧바로 쿼리 실행 단위에 연결되며, 그룹 자체도 선택하지 않아도 된다(Optional). 즉
개별검증은 일괄검증의 "업로드 세트(배치)" 레이어가 통째로 빠진 3단계 구조다.

## 검증 결과
- 파일:줄 근거 — `services/project_store.py:100`, `services/batch_group_service.py:462`,
  `routes/batch_route.py:950-953`, `services/validation_run/execution_context.py:54,149`,
  `.gitignore:43,46`
- DB 조회 근거 — `db/migration_validator.db` 및 `db/migration_validator.db.bak_20260914_184810_*`
  두 시점 모두 `DTV_project` 전수 조회, `sqlite_sequence`, `DTV_mv_batch_run`/`DTV_batch_run`/
  `DTV_validation_execution_run` 스키마·건수·표본 조회로 직접 확인(sqlite3 CLI 결과, 본 보고서
  본문에 원문 반영)
- git 이력 근거 — `374be77b`(테이블 최초 생성, 삭제 API 없음), `84df47f0`(soft delete 도입),
  `32364712`(group_id 최초 도입), `bd14987f`(project→DTV_project 테이블명 변경)
- 코드 수정 없음(지침 명시 "코드 수정 절대 금지" 준수) — 30번 규칙(소스 diff 증적) 대상 아님.
  UI 변경도 없음 — 29번 체크리스트 대상 아님(순수 조사).

## 기대효과
- `project_id=2` 시작, `group_id` 문자열 형식에 대한 "왜"가 추정이 아니라 코드/DB 근거로
  뒷받침되어, 향후 유사 질문(다른 ID 체계 등) 시 동일 방식(스키마 조회 + git blame + 관련
  주석)으로 재현 가능한 조사 절차가 확보됨.
- 일괄/개별검증의 실제 계층 구조 차이(4단계 vs 3단계, "배치" 레이어 유무)가 명확해져, 향후
  두 경로에 공통 기능을 추가할 때 "그룹 다음에 무엇을 걸어야 하는지" 설계 혼선을 줄일 수 있음.

작업명 : PROJECT-ID-AND-GROUP-ID-SCHEME-AND-FULL-HIERARCHY-EXPLAIN
✅ 작업 완료 - project_id/group_id 키 체계 근거 확인 + 일괄/개별검증 전체 계층구조를 코드·DB 조회로 규명(코드 수정 없음, 조사 전용)
