```text
작업명 : PROJECT1013-IS-TEST-GROUP-8-CLEANUP
✅ 작업 완료 - 프로젝트 1013의 is_test=True 그룹 8건 + 하위데이터(등록 대상/size_score/메타데이터 등) 앱 내부 SQLite 관리DB에서만 완전 삭제, 실 Oracle/Postgres는 접속하지 않음

**목적**
사용자 요청에 따라 프로젝트 1013 하위 is_test=True 그룹 8건과 그에 딸린 모든 하위 추적 데이터
(등록 대상, size_score, 메타데이터 등)를 앱 내부 관리 DB(SQLite, db/migration_validator.db)에서만
완전 삭제한다. 실 Oracle/Postgres 등 외부 검증 대상 DB는 절대 접속하지 않는다.

대상 group_id 8건:
GRP_202609140919054137, GRP_202609140924061714, GRP_202609140925344540,
GRP_202609140929084295, GRP_202609140932038208, GRP_202609140933273582,
GRP_202609140936124120, GRP_202609140937364490

**현상**
- 8건 모두 사전 조회로 project_id='1013', is_test=1, is_deleted=0 확인(전량 일치, 누락/오류 없음).
- 기존 서비스 `services/group_hard_reset_service.delete_group()`(테스트 그룹 전용 hard delete cascade,
  `services/batch_group_service.hard_delete_group()`이 위임하는 공통 core)이 다루는 테이블은
  group_id 참조 테이블 전체(28개) 중 5개(DTV_metadata_collection_run, DTV_metadata_table_snapshot,
  DTV_policy_stats_validation_plan_snapshot, DTV_policy_target_table_config,
  DTV_validation_batch_group)뿐이었고, `DTV_task_size_score`(size_score 캐시)를 포함해 아래 7개
  테이블(총 51건)이 해당 cascade 목록에 빠져 있음을 사전 스캔으로 확인함:
  DTV_incomplete_attempt_history(4), DTV_official_register_candidate_snapshot(8),
  DTV_source_owner_binding(8), DTV_task_size_score(7), DTV_validation_execution_run(8),
  DTV_validation_history_run(8), DTV_validation_persistence_idempotency(8).
- 완성 모듈(group_hard_reset_service.py)은 지침상 임의 리팩토링 대상이 아니므로 코드는 수정하지 않고,
  이번 삭제 작업 범위 한정으로 별도 1회성 스크립트를 통해 누락분까지 함께 정리함.

**실제조치**
1. DB 파일 백업: db/migration_validator.db.bak_20260914_184810_pre_test_group_1013_cleanup 생성(원본 유지, 되돌리기 가능).
2. 8개 group_id 전량에 대해 기존 서비스 `delete_group(project_id='1013', group_id=gid)` 호출 —
   wired cascade(업로드/메타데이터/그룹 마스터 row 등)를 트랜잭션으로 삭제(8건 모두 success=True).
3. cascade 밖 7개 테이블(위 목록)을 동일 8개 group_id 범위로 한정해 직접 DELETE 실행(스크립트는
   프로젝트 코드가 아닌 세션 scratchpad에서만 실행, 저장소 코드 변경 없음).
4. 삭제 전 개수와 4)의 삭제 건수가 정확히 일치함을 로그로 확인.

**검증 결과**
- 삭제 전 group_id 참조 28개 테이블 전수 스캔: nonzero 12개 테이블, 총 91건(그룹 마스터 8건 포함).
- 삭제 후 동일 28개 테이블 재스캔: 전부 0건(잔존 없음).
- DTV_validation_batch_group에서 8개 group_id 조회 결과: 0행(마스터 row 완전 삭제 확인).
- 프로젝트 1013의 나머지 그룹(비대상, is_test=0 또는 다른 그룹) 12건은 영향 없음(건수 변동 없음) —
  요청 범위(8건) 외 그룹은 손대지 않았음을 확인.
- 실 Oracle/Postgres 등 외부 DB 접속 없음(로컬 SQLite 관리 DB만 조작) — 코드상 외부 커넥션 호출
  경로를 전혀 거치지 않았으므로 확인됨.

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 (화면 요소를 전혀 건드리지 않은 순수 백엔드 관리DB row 삭제 작업)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(UI 변경 자체가 없어 스크린샷 불필요, 대신 위 "검증 결과"의 DB row count 전/후 비교로 실측 증적 대체)

※ 30번 규칙(소스 diff 증적) 관련: 이번 작업은 저장소 내 소스 코드 파일을 전혀 수정하지 않음
(기존 검증된 서비스 함수 호출 + 세션 scratchpad 1회성 스크립트로 직접 DELETE만 수행). 따라서
"코드 수정이 전혀 없는 지침"에 해당해 30번 규칙 대상이 아님.

**기대효과**
- 요청받은 8개 테스트 그룹과 그 하위 데이터(등록 대상/size_score/메타데이터 등)가 앱 추적 데이터에서
  완전히 제거되어 그룹 목록/집계 화면 등에 더 이상 노출되지 않음.
- 기존 delete_group() cascade의 사각지대(특히 size_score, official-register 후보 스냅샷, 실행이력 등
  7개 테이블)를 실측으로 발견 — 향후 일반적인 테스트 그룹 정리 시에도 이 사각지대가 반복될 수 있으므로,
  해당 cascade 목록 보강 여부는 별도 지침(사용자 확인 후)으로 판단 필요.

작업명 : PROJECT1013-IS-TEST-GROUP-8-CLEANUP
✅ 작업 완료 - 프로젝트 1013의 is_test=True 그룹 8건 + 하위데이터(등록 대상/size_score/메타데이터 등) 앱 내부 SQLite 관리DB에서만 완전 삭제, 실 Oracle/Postgres는 접속하지 않음
```
