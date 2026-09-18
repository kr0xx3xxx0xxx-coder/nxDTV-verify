```text
작업명 : TEST-FRESH-PROJECT-VS-REUSE-STANDARD-FIXTURE-RATIONALE-CHECK

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 (코드 수정 없이 기존 파일·git 이력만
  읽어서 대조한 순수 조사 지침)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(UI 변경 자체가 없음)

(30번 규칙 관련) 코드 수정 없음(조사 전용 지침) — git diff 증적 대상 아님.

■ 목적
"표준 픽스처(TESTONLY_REG / __TEST_PROJECT__)가 이미 있는데, 왜 많은 검증
스크립트가 재사용하지 않고 매번 새 프로젝트를 만들었다가 방치했는가 — 그냥
기존 걸 재사용 후 나중에 지우면 되는 것 아니냐"는 사용자 판단을, 짐작 없이
코드 근거로 확인한다.

■ 현상 (조사 결과 — 근거)

1. 표준 픽스처의 격리 메커니즘 확인
   `tests/_group_test_project.py`(파일 전체 읽음):
   - `test_project_id()`(21~38행): `__TEST_PROJECT__`라는 프로젝트를
     `list_projects()`로 이름 검색 → 있으면 재사용, 없으면 `is_test=True`로
     1회만 생성(get-or-create). **프로젝트는 공유**한다.
   - `make_test_group()`(41~49행): 매 호출마다 `create_group()`으로 **새
     그룹**을 만들고 `is_test=True`를 강제한다. 실제 호출부(예:
     `tests/test_single_group_registration.py:84`,
     `tests/test_single_official_register.py:87` 등 13개 파일)는
     `group_name="REG-TEST-" + os.urandom(4).hex()`처럼 매번 랜덤 접미사를
     붙여 그룹명을 유일하게 만든다.
   → 격리 단위는 "프로젝트"가 아니라 "그룹"이다. 프로젝트를 공유해도 각
     테스트가 자기 그룹만 보므로 서로 안 헷갈리고, `is_test=1` 플래그 덕에
     잔재가 남아도 `scripts/cleanup_test_projects.py`/`cleanup_test_groups.py`
     같은 자동 회수 도구가 식별할 수 있게 설계돼 있다(주석에 명시:
     "cleanup_test_projects 가 이름 패턴 추측 없이 플래그만으로 안전하게
     회수할 수 있게").
   - `tests/test_project_scope_guard.py`(1~40행)류는 이것과 **또 다른**
     정당한 격리 방식을 쓴다: `setUp()`에서 `ps._DB_PATH`/`bgs._DB_PATH`를
     `_tmp_scope_<timestamp>.db`라는 **완전히 새 임시 SQLite 파일**로
     바꿔치기하고, `tearDown()`에서 그 파일을 `unlink()`로 지운다(24~36행).
     즉 운영 `db/migration_validator.db`를 아예 건드리지 않고 매 테스트
     클래스마다 프로젝트를 자유롭게 새로 만들어도(`_pid()`, 39행) 잔재가
     남지 않는다.

2. "매번 새 프로젝트를 만드는" 스크립트들의 코드 근거 확인
   - 사용자가 예로 든 `MV_VERIFY_*_TMP`, `SIZE-SCORE-*`,
     `M337_COMPOSITE_PK_VERIFY`, `BATCH_AUTOSAVE_VERIFY_TMP` 류의 원본
     스크립트는 **이미 오늘 정리 대상으로 삭제됐고 git에도 한 번도 커밋된
     적이 없어(`git log --all --diff-filter=D` 조회 결과 0건)** 원문을 직접
     열어볼 수 없었다. 대신 같은 성격(1회성 자동 프로젝트 생성)이면서
     여전히 저장소에 남아있는 형제 스크립트들로 대체 확인했다.
   - `scripts/dev_e2e/m34_disabled_button_title_accessibility_verify.py:258`
     — `project_store.create_project("MV-M34-DISABLED-TITLE-VERIFY", ...,
     is_test=True)`. 이유 설명 주석 없음. 그냥 매 지침마다 고유한 프로젝트명을
     짓고 `create_project`를 1회 호출하는 패턴.
   - `scripts/dev_e2e/admin_column_override_project_scope_ui_verify.py:271-282`
     (`_ensure_project()`) — `__TEST_PROJECT__`를 안 쓰고 `MV-ADMOVR-PROJSCOPE`
     라는 **전용** 프로젝트를 이름으로 get-or-create. 같은 파일 309행
     `for row in store.list_overrides(project_id=pid): store.delete_override(...)`
     로 시작 시 그 프로젝트의 관리컬럼 override를 매번 초기화한다. 관리컬럼
     override는 **프로젝트 전역 설정**(그룹이 아니라 project_id 단위)이라서,
     공유 프로젝트(`__TEST_PROJECT__`)를 썼다면 이 스크립트가 override를
     지웠다 켰다 하는 동안 **다른 검증 스크립트가 그 시점에 남겨둔
     override 상태와 충돌**할 위험이 있다 — 이건 그룹 단위 격리(표준
     픽스처 방식)로는 못 막는, project_id 단위로만 막을 수 있는 간섭이다.
   - `scripts/dev_e2e/sample_basis_evidence_batch_fixture_setup.py:15`
     — 주석 원문: "기존 데이터는 건드리지 않는다(새 프로젝트/그룹/검증대상만
     추가)". 이건 "프로젝트 레벨 격리가 구조적으로 필요하다"는 근거라기보다,
     "실 DB의 기존 등록/후보 데이터를 실수로 건드리지 않겠다"는 **일회성
     브라우저 실측 편의** 목적의 소극적 이유에 가깝다 — `make_test_group()`
     +새 그룹만으로도 동일하게 달성 가능했던 목적이다.
   - `tests/test_single_save_source_binding.py:43,109` — 오히려 반대 극단
     사례: 새 프로젝트를 만드는 대신 **운영 프로젝트 `project_id="2"`
     ("2026 차세대 시스템 데이터 검증")를 직접 하드코딩**해 그 안에
     `SAVE-SRCBIND-` 접두어 그룹을 매 실행마다 새로 만든다. 왜 `_group_test_project`
     헬퍼(같은 tests/ 디렉터리에 이미 존재)를 안 쓰는지 설명 주석 없음.
     이번 조사와 별도로 존재했던 오늘자 인벤토리 보고서
     (`STALE-TEST-PROJECTS-AND-GROUPS-INVENTORY-FOR-USER-CONFIRM_20260918.md`
     125~131행)가 이미 지적한 사례로, 이건 "새 프로젝트를 만든" 문제가
     아니라 "표준 픽스처를 안 쓰고 운영 프로젝트에 잔재를 남긴" 문제다.

3. 표준 픽스처로 충분했는데 안 쓴 사례 vs 프로젝트 격리가 실제로 필요했던
   사례 구분
   - 오늘자 인벤토리 보고서(`...INVENTORY-FOR-USER-CONFIRM_20260918.md`
     196~206행)가 이미 `scripts/cleanup_test_projects.py`를 dry-run으로
     돌려 확인한 사실: `MV_VERIFY_*_TMP`·`M337_COMPOSITE_PK_VERIFY`·
     `BATCH_AUTOSAVE_VERIFY_TMP` 등 **약 40여 건은 `is_test=0`으로
     생성**돼 있다 — 즉 이 스크립트들은 "테스트 전용" 표시조차 안 남겼다.
     이는 프로젝트 격리가 필요해서가 아니라, 애초에 `_group_test_project.py`
     헬퍼(2026년 8월경 도입, 주석에 "GROUP-CREATION-GUARD-AND-TEST-RESIDUE-
     CLEANUP" 배경 명시)가 나오기 **이전 시기부터 굳어진 각자 편의 패턴**이
     그대로 반복된 것으로 판단된다(1000번대 초반 project_id들의 생성일시가
     2026-08-28~29에 몰려 있고, 헬퍼 파일 자체가 이 시기 전후 도입된 정황과
     맞물림).
   - 반대로 `test_project_scope_guard.py`류(프로젝트 자체의 상태 — 삭제됨/
     보호됨/타 프로젝트 불일치 — 를 조합 검증해야 함)와
     `admin_column_override_project_scope_ui_verify.py`류(프로젝트 전역
     설정을 다루므로 공유 시 교차 오염 위험)는 "새 프로젝트를 만드는 것"
     자체에 구조적 근거가 있다. 다만 이 두 부류는 이미 각자 방식으로
     "잔재를 안 남기는" 처리가 돼 있다(전자는 임시 DB 파일 자체를 삭제,
     후자는 이름 기반 get-or-create로 재실행 시 중복 생성은 안 됨 — 다만
     최초 1회 생성한 뒤로는 삭제 로직이 없어 프로젝트 자체는 영구 잔존).

4. 결론 — 사용자 판단 구분
   (a) 이미 일부는 그렇게 하고 있다 — 참(YES).
       `tests/_group_test_project.py` 경유 pytest 테스트(13개+ 파일)는
       "프로젝트 공유 + 그룹만 매번 새로 + is_test 플래그로 회수 가능"
       방식을 이미 쓰고 있고, `test_project_scope_guard.py`류는 "완전
       격리된 임시 DB + 자동 삭제"라는 더 강한 방식으로 이미 잔재를 안
       남긴다.
   (b) 나머지 대부분은 그렇게 바꿀 수 있는데 안 바꾼 것뿐이다 — 이번
       조사에서 확인된 증거로는 **참(YES)에 가깝다.**
       사용자가 예로 든 `MV_VERIFY_*_TMP`/`SIZE-SCORE-*`/
       `M337_COMPOSITE_PK_VERIFY`/`BATCH_AUTOSAVE_VERIFY_TMP` 등 오늘자
       인벤토리 [C](삭제후보) 목록 67건 대부분, 그리고 여전히 남아있는
       형제 스크립트(`m34_disabled_button_title_accessibility_verify.py`
       등)에서 "프로젝트를 새로 만들어야만 하는" 구조적 필요성을 코드
       주석·로직 어디서도 찾지 못했다. `is_test=0`으로 남겨져 회수 도구의
       사각지대에 있다는 사실 자체가,애초에 재사용/정리를 염두에 두고
       설계된 스크립트가 아니라 각자 편의로 짠 1회성 산출물임을 뒷받침한다.
       표준 픽스처(공유 프로젝트+매번 새 그룹, 또는 임시 DB) 방식으로
       충분히 커버 가능했을 사례로 판단된다.
   (c) 일부는 프로젝트 격리가 실제로 필요해서 그런 것이다 — 부분적으로
       참(YES), 단 소수.
       `test_project_scope_guard.py`류(프로젝트 상태 조합 자체가 검증
       대상)와 `admin_column_override_project_scope_ui_verify.py`류
       (프로젝트 전역 설정 충돌 회피)는 "왜 새 프로젝트가 필요한지"에
       구조적 근거가 있다. 다만 후자 계열(MV-ADMOVR-*, MV-SBE-*, M34-* 등)
       조차도 "새로 만드는 것"의 필요성과 "만든 뒤 안 지우는 것"은 별개
       문제다 — 필요성이 있는 스크립트들도 완료 후 프로젝트 자체를 정리하는
       로직은 없어서, 필요성 유무와 무관하게 전부 방치되는 결과는 동일했다.

   요약: 사용자 판단은 대체로 타당하다. "격리가 필요해서"라는 정당한
   이유가 있는 경우는 소수(프로젝트 상태 자체를 다루는 테스트, 프로젝트
   전역 설정을 다루는 테스트)이고, 나머지 다수는 표준 픽스처를 재사용한
   뒤 정리했으면 됐을 것을 각자 편의로 새로 만들고 방치한 결과로 확인된다.
   다만 "격리가 필요한 소수"조차도 "생성 필요성"과 "사후 미정리"는 서로
   다른 문제이며, 이번 조사는 그 둘을 구분해 확인하는 데 그친다(조사만
   수행, 코드 수정·삭제는 이 지침 범위 밖).

■ 기대효과
- 사용자가 가진 "그냥 재사용하고 나중에 지우면 된다"는 판단이 실제로
  타당한 범위(대다수)와, 구조적으로 별도 프로젝트가 필요한 예외 범위
  (소수)를 코드 근거로 구분해, 향후 "표준 픽스처 사용 의무화" 같은 규칙을
  만들 때 오탐 없이(즉 정말 필요한 소수 사례까지 강제로 막지 않고) 설계할
  수 있는 근거자료가 된다.
- 오늘자 `STALE-TEST-PROJECTS-AND-GROUPS-INVENTORY-FOR-USER-CONFIRM_20260918.md`
  보고서가 지적한 "사각지대(is_test=0 자동생성 잔재 약 42건)"가 왜
  생겼는지(헬퍼 도입 이전 관행이 계속 반복됨)에 대한 원인 설명을 보탠다.

■ 검증 결과
- 코드 수정 없이 아래만 수행: `tests/_group_test_project.py` 전체 읽기,
  `make_test_group(` 호출부 13개 파일 grep, `tests/test_project_scope_guard.py`
  1~45행 읽기, `create_project(` 호출부 30개 파일 목록 확인 후 대표 사례
  3개(`m34_disabled_button_title_accessibility_verify.py`,
  `admin_column_override_project_scope_ui_verify.py`,
  `sample_basis_evidence_batch_fixture_setup.py`) 및
  `tests/test_single_save_source_binding.py` 원문 확인.
- `git log --all --diff-filter=D --name-only`으로 사용자가 예로 든
  `MV_VERIFY_*_TMP` 등 원본 스크립트가 git 이력에 전혀 없음(커밋된 적
  없는 scratchpad 산출물)을 확인 — 이 때문에 그 스크립트들 자체의
  주석은 직접 인용할 수 없었고, 같은 시기·같은 패턴의 현존 형제
  스크립트로 대체 확인했음을 위 2번 항목에 명시함(한계로 밝힘).
- HEAD 커밋: 6f16d910 (조사 시점 기준, 코드 변경 없음).

작업명 : TEST-FRESH-PROJECT-VS-REUSE-STANDARD-FIXTURE-RATIONALE-CHECK
✅ 작업 완료 - 표준 픽스처 재사용 vs 매번 새 프로젝트 생성 관행을 코드 근거로 대조해, 사용자 판단(대부분 불필요했음)이 타당함과 예외(프로젝트 상태/전역설정 검증)를 구분해 확인
```
