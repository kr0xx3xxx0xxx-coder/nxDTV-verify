```text
작업명 : EXECUTION-REUSE-WHICH-PRIOR-RESULT-SELECTION-CORRECTNESS-VERIFY
✅ 작업 완료 - "여러 성공 이력 중 어느 것을 고르는지" 선택 기준(정렬/동률/성공판정/배지표시 4개 항목) 코드 레벨 조사, 코드 수정 없음(지침 명시)

## 목적
`services/execution_reuse_lookup.py`의 `find_last_success_single_run()`/
`find_last_success_batch_run()`이 여러 성공 이력 중 "직전 1건"을 고르는
선택 기준(정렬 컬럼, 동률 처리, 성공 상태값 범위, 배지 표시 일치)이
정확한지 코드 레벨로 검증한다. SQL 동일성 비교 로직(M367에서 이미 수정
완료)과는 별개 사안이며, 이번 지침은 조사 전용 — 코드 수정을 하지 않는다.

## 현상 (확인 대상)
사용자 우려: 동일 SQL로 여러 번 성공한 이력이 쌓여 있을 때, "가장 최근"
1건을 고르는 로직이 (1) 시작 시각과 종료 시각이 엇갈리는 경우에도 올바른지,
(2) 동률(같은 시각) 시 결정적인지, (3) "성공" 상태값에 부분성공이
섞여 있지 않은지, (4) 화면 배지가 실제 선택된 값을 그대로 보여주는지
불명확했음.

## 조사 결과 (실제조치)

### 1) 정렬 기준 — 시작시각 vs 종료시각 엇갈림에 안전한가
- **배치**(`execution_reuse_lookup.py:138,152`):
  `ORDER BY sr.created_at DESC, sr.id DESC`
  - `created_at`은 정렬용 별도 컬럼이 아니라 `_save_plan_result()`
    (`services/batch_stats_execute_service.py:319`)에서 **plan 실행이 끝난
    직후(finished_at 캡처 후, 같은 함수 호출 안에서) 독립적으로 다시 찍는
    타임스탬프**다. 배치 루프(`batch_stats_execute_service.py:1300` for문)는
    plan을 **순차 실행**하며, 각 plan이 끝나자마자 바로 그 결과를 저장하므로
    (`:1334~1386`) `created_at`은 사실상 "그 실행이 끝난 시각"의 대리값이지
    "그 실행이 시작된 시각"이 아니다. 따라서 plan A가 먼저 시작해 늦게
    끝나고 plan B가 늦게 시작해 먼저 끝나는 상황이 섞여도, `created_at`
    순서는 **시작 순서가 아니라 종료(저장) 순서**를 그대로 반영한다 →
    사용자 기대("가장 최근에 끝난 결과")와 일치.
  - 단, 화면에 노출되는 `executed_at`(`execution_reuse_lookup.py:181`)은
    `finished_at`을 우선(`row["finished_at"] or row["created_at"]`)하고,
    정렬은 `created_at`을 쓴다 — 같은 실행의 서로 다른 두 컬럼을 쓰는
    설계상 불일치이나, 두 값이 같은 함수 호출 안에서 수 ms 이내에
    연달아 찍히므로 실질적 순서 역전 위험은 없음(문제 아님, 참고 사항).
- **개별검증**(`execution_reuse_lookup.py:233,247`):
  `ORDER BY COALESCE(r.finished_at, r.started_at) DESC, r.run_id DESC`
  - 실제 **종료 시각(finished_at)을 직접** 정렬 기준으로 사용(시작 시각은
    finished_at이 없는 비정상 행에 대한 폴백일 뿐). 시작-종료 순서가
    엇갈려도 항상 "늦게 끝난 것"이 먼저 온다 → 안전.
- **결론(1번)**: 두 경로 모두 "시작 시각"이 아니라 "종료/저장 시각" 계열
  컬럼으로 정렬하므로, 사용자가 우려한 시나리오(먼저 시작·늦게 종료 ↔
  늦게 시작·먼저 종료 혼재)에서도 오판 없음. **문제 없음.**

### 2) 동률(tie) 처리 — 2차 기준의 결정성
- **배치**: 2차 기준 `sr.id DESC` — SQLite autoincrement 정수 PK로
  삽입 순서와 100% 일치하는 단조증가 값. `created_at`이 초 단위 정밀도만
  가짐(`_now_iso()` = `strftime("%Y-%m-%dT%H:%M:%SZ")`,
  `batch_stats_execute_service.py:106-107` — 마이크로초 없음)이라 같은 초에
  끝나는 두 성공 이력이 실제로 발생할 수 있지만, `id DESC`가 그 경우에도
  **진짜 더 나중에 저장된 행**을 정확히 골라낸다 → 결정적이고 의미도
  올바름.
- **개별검증**: 2차 기준 `r.run_id DESC` — `run_id`는
  `uuid.uuid4()`(`services/validation_result_store.py:192`)로 생성되는
  **시간과 무관한 랜덤 문자열**이다. `finished_at`은 마이크로초 정밀도
  (`datetime.now(timezone.utc).isoformat()`, 같은 파일 165행)라 배치보다
  동률 가능성은 훨씬 낮지만, 이 프로젝트의 실제 운영 환경이 Windows
  (system-reminder 기준 win32)이고 Windows의 시스템 클럭 해상도 제약상
  완전 배제는 불가능하다. **동률이 실제로 발생하면**, `run_id`(UUID)
  기준 정렬은 어느 쪽이 실제로 더 최근인지와 **무관한 값**으로 승부가
  갈린다 — 매 쿼리마다 항상 같은 결과를 주므로 "비결정적"은 아니지만,
  "시간 순서상 올바른 것을 고른다는 보장이 없는 결정적 선택"이 된다.
  참고로 `DTV_validation_execution_run` 테이블은 `run_id TEXT`를
  PRIMARY KEY로 선언했을 뿐 `WITHOUT ROWID`가 아니므로, 배치처럼 쓸 수
  있는 암묵적 단조증가 `rowid`가 이미 존재하는데도 2차 기준으로
  활용되지 않고 있다.
- **결론(2번)**: 배치 경로는 동률에도 항상 올바른 선택을 보장. **개별검증
  경로만 유일하게 개선 여지 있음** — 발생 확률은 낮지만, 발생 시
  "결정적이되 시간순서와 무관한" 선택이 될 수 있다.

### 3) "성공"으로 간주하는 상태값 — 부분성공 혼입 여부
- **배치** `_BATCH_SUCCESS_STATUSES = ("SUCCESS_MATCHED", "SUCCESS_DIFF")`
  (`execution_reuse_lookup.py:47`) — 두 값 모두 "비교가 끝까지 실행되어
  정상 종료"된 상태이며 DIFF는 단지 "불일치 발견"이지 실행 실패가
  아니다. TIMEOUT/ERROR/FAILED/SKIPPED_RISK/SKIPPED_UNSUPPORTED/
  SKIPPED_CANCELLED/SKIPPED_INTERRUPTED
  (`services/batch_stats_execute_service.py:53-63`)는 전부 제외되어
  부분성공·에러 혼입 없음. 재사용 시 `is_truncated`(표시 절삭) 플래그도
  그대로 복사되어(`batch_stats_execute_service.py:1086-1087`) 절삭
  사실이 은폐되지 않음.
- **개별검증** `_SINGLE_SUCCESS_STATUS = "SUCCESS"` 단일값
  (`execution_reuse_lookup.py:51`) — `_run_status_of()`
  (`services/validation_result_store.py:670-676`)를 보면 비교 자체가
  일부 절삭된 경우(`compare_truncated`)는 별도로 `"PARTIAL"`,
  DB 오류는 `"FAILED"`로 분리 저장되며 `"SUCCESS"`와 섞이지 않는다.
  `find_last_success_single_run()`은 `status = "SUCCESS"`만 매치하므로
  PARTIAL/FAILED는 재사용 대상에서 정확히 제외됨.
- **결론(3번)**: 두 경로 모두 부분성공/오류가 "성공"으로 잘못 포함되는
  사례 없음. **문제 없음.**

### 4) 재사용 배지 표시값과 선택 로직의 일치 여부
- 배치: `match.executed_at` → `summary["reused_original_executed_at"]`
  (`batch_stats_execute_service.py:1073`) → 화면
  (`ui/js_batch_display.py:394-399`)이 "재판정/재계산 없이" 그대로 노출
  (주석에 명시).
- 개별검증: `match.executed_at` → `new_snapshot["reused_original_executed_at"]`
  (`services/single_validation_run_facade.py:1030`, 1082행 응답에도 동일
  값) → 동일 필드를 그대로 화면에 전달.
- 두 경로 모두 선택 로직(`LastSuccessRunLookup.executed_at`)이 반환한
  **그 값 자체**를 배지가 출력하며, 중간에 재조회·재계산하는 경로가
  없음을 코드로 확인.
- **결론(4번)**: 선택된 행의 시각과 배지 표시 시각은 **항상 동일 값**을
  참조한다. **문제 없음.**

## 검증 결과 (5번 — 종합 결론)
- **배치 경로(`find_last_success_batch_run`)**: 정렬·동률·성공판정·배지
  표시 4개 항목 모두 명확하고 결정적이며, 사용자가 우려한 시나리오에서도
  올바른 선택을 보장한다. **문제 없음.**
- **개별검증 경로(`find_last_success_single_run`)**: 정렬(종료시각 기준)·
  성공판정·배지 표시는 명확하고 올바르다. **다만 동률 2차 기준(`run_id`
  = UUID4)만 유일하게 개선 여지가 있다** — 시간과 무관한 값이라, 완전히
  같은 시각(마이크로초까지 동일 또는 OS 클럭 해상도 한계로 구분 불가)에
  끝난 두 성공 이력이 존재할 경우 "실제로 더 나중에 끝난 것"을 고른다는
  보장이 없다. 발생 확률은 낮으나(동시 실행이 드묾 + 마이크로초 정밀도),
  배치 경로처럼 테이블에 이미 존재하는 암묵적 `rowid`를 2차 기준으로
  추가하면 완전히 해소 가능한 구조적 결함이다. 기존 테스트
  (`tests/test_execution_reuse_lookup.py`)에도 "복수 성공 이력 중 선택"
  시나리오(정렬/동률) 자체를 검증하는 테스트가 없어, 이 지점이 지금까지
  전혀 커버되지 않은 사각지대였음을 함께 확인했다.
- 코드 수정은 지침에 따라 수행하지 않았다(조사 전용). 개별검증 동률
  2차 기준 개선은 별도 지침으로 사용자 판단 후 진행 권장.

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 아니오 (코드 레벨 조사만
  수행, 코드 수정 없음 — 지침에 "코드 수정 절대 금지 — 조사만" 명시)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 해당없음
- 수정 후 스크린샷을 실제로 촬영했는가: 해당없음
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 해당없음
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: 해당없음(코드
  수정 자체가 없어 30번 소스 diff 증적 요건도 대상 아님)

작업명 : EXECUTION-REUSE-WHICH-PRIOR-RESULT-SELECTION-CORRECTNESS-VERIFY
✅ 작업 완료 - "여러 성공 이력 중 어느 것을 고르는지" 선택 기준(정렬/동률/성공판정/배지표시 4개 항목) 코드 레벨 조사, 코드 수정 없음(지침 명시)
```
