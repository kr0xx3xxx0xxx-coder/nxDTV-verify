```text
작업명 : PAIR-RESTORE-LAYOUT-REVERT-PROFILE-SORT-AND-E2E-DB-ISOLATION
✅ 작업 완료 - 파트A(소프트삭제 복구+화면 원복+프로필 정렬 확인) / 파트B(E2E DB 격리 표준 절차 신설) 모두 완료

════════════════════════════════════════
파트A — 복구 + 화면 원복 + 프로필 정렬
════════════════════════════════════════

## 목적
커밋 bc69e634가 화면 순서를 [DB 프로필 → 검증 경로]로 임의 재배치하면서 부수적으로
검증 경로 "Oracle_asis → Oracle_tobe"가 소프트 삭제된 사고를 복구하고, 화면 순서를
원래대로 되돌리며, DB 프로필 정렬 기능이 정상 동작하는지 확인한다.

## 현상
- 선행 보고서(`PAIR-LIST-MISSING-ORACLE-ENTRY-URGENT-CHECK-AND-LAYOUT-REVERT_20260917.md`,
  파트0)가 이미 원인을 조사해뒀다: `DTV_mv_conn_pair`의 `pair_id=pair_msn2p0rk_necmw`
  행이 `is_deleted=1`(행 자체는 보존, 하드삭제 아님), `updated_dt=2026-09-17 13:59:52`.
  bc69e634 커밋 diff 자체·정식 HTTP DELETE 경로·회귀테스트 3가지 모두 원인에서 배제됐고,
  정확한 트리거는 로그 부재로 **미상**으로 남아 있었다(이번 작업에서도 추가로 특정하지
  않음 — 파트B에서 재발 방지 절차만 신설).
- `db/migration_validator.db` 직접 SELECT로 재확인: 위 행이 `is_deleted=1`인 상태 확인.

## 실제조치
1. **A-1 복구**: `UPDATE DTV_mv_conn_pair SET is_deleted=0 WHERE pair_id='pair_msn2p0rk_necmw'`
   1건 실행(다른 컬럼 무변경). 복구 전/후 SELECT 결과:
   - 복구 전: `is_deleted=1`
   - 복구 후: `is_deleted=0, updated_dt='2026-09-17 13:59:52'`(트리거 없음 확인 —
     `updated_dt`는 원래 소프트삭제 시점 값 그대로 보존, 내 UPDATE로 갱신되지 않음)
   화면(스크린샷, 아래 참고)에서 해당 행이 검증 경로 목록에 재등장함을 확인.

2. **A-2 화면 카드 순서 원복**: `ui/tabler_renderer.py`의 `tab-settings` 블록을
   [검증 경로 목록 → DB 프로필 목록] 순서로 원복(bc69e634 이전 구조). bc69e634가 함께
   도입한 접속상태 정확성 수정(`_pairStatusLabel` FAILED 판정, `_profileConnMarker`
   ✓/✗ 표시, `connectPair`의 `updateStatusBadge` 실접속 교정)은 전부 그대로 유지 —
   이번 변경은 카드 배치 순서(HTML 블록 위치)만 되돌렸다.
   `tests/test_connection_settings_restructure.py`의
   `test_settings_section_order_pairs_then_profiles_then_detail` 기대값도 원래
   순서([연결쌍 → 프로필])로 원복.

3. **A-3 DB 프로필 정렬 확인**: 요구된 "현재 접속 검증 경로의 원본/목적 프로필을
   맨 위로, 원본→목적 순서로 표시" 기능은 코드 조사 결과 **이미 커밋 2b31b8a0**
   (2026-06-08, `fix(connection): clarify active path header and profile ordering`)에서
   구현돼 있었다 — `renderDbProfileList()` 안의 `_connectedPair`/`_findPair`/
   `_pairSources` 기반 표시 전용 정렬 로직(`ui/tabler_renderer.py` 약 17608~17629행).
   9번 규칙(기존 상태 재사용, 새 정렬 로직 발명 금지)에 따라 **코드 변경 없음** —
   화면 순서 원복 후에도 이 정렬이 정상 동작하는지만 스크린샷으로 재확인했다.

### git diff (30번 규칙 증적)
--- BEGIN diff: tests/test_connection_settings_restructure.py, ui/tabler_renderer.py ---
diff --git a/tests/test_connection_settings_restructure.py b/tests/test_connection_settings_restructure.py
index 480dcd51..9aca800f 100644
--- a/tests/test_connection_settings_restructure.py
+++ b/tests/test_connection_settings_restructure.py
@@ -30,19 +30,20 @@ def _html():
 # ── 정적 렌더: 화면 구조/순서 ─────────────────────────────────────────────────────

 def test_settings_section_order_pairs_then_profiles_then_detail():
-    """tab-settings 안에서 DB 프로필 목록 → 연결쌍 목록 → 상세 입력 순서로 배치된다.
+    """tab-settings 안에서 연결쌍 목록 → DB 프로필 목록 → 상세 입력 순서로 배치된다.

-    (DB-CONNECTION-STATUS-DISPLAY-CONSISTENCY-INVESTIGATE-AND-FIX, 2026-09-17)
-    개별 프로필 접속상태가 더 정확한 원천이므로, 검증 경로 목록보다 먼저 보이도록
-    DB 프로필 목록을 화면 상단으로 재배치했다(순서만 변경 — id/JS 참조는 무변경).
+    (PAIR-RESTORE-LAYOUT-REVERT-PROFILE-SORT-AND-E2E-DB-ISOLATION, 2026-09-17)
+    DB-CONNECTION-STATUS-DISPLAY-CONSISTENCY-INVESTIGATE-AND-FIX(bc69e634)가 화면 순서를
+    [프로필 → 연결쌍]으로 재배치했던 것을 원복한다 — 접속상태 정확성 수정(_pairStatusLabel/
+    _profileConnMarker 등)은 유지하고 배치 순서만 되돌린다.
     """
     h = _html()
     i_pair = h.index('id="pairListCard"')
     i_prof = h.index('id="dbProfileCard"')
     i_det = h.index('id="dbDetailCard"')
-    assert i_prof < i_pair < i_det, "화면 순서가 [프로필 → 연결쌍 → 상세] 가 아님"
-    # 두 목록 모두 기존 대형 입력폼(상세)보다 위에 온다
-    assert i_prof < h.index('id="srcDbSide"'), "DB 프로필 목록이 입력폼보다 위가 아님"
+    assert i_pair < i_prof < i_det, "화면 순서가 [연결쌍 → 프로필 → 상세] 가 아님"
+    # 연결쌍 목록이 기존 대형 입력폼(상세)보다 위에 온다
+    assert i_pair < h.index('id="srcDbSide"'), "연결쌍 목록이 입력폼보다 위가 아님"


 def test_db_profile_card_and_new_db_button():
diff --git a/ui/tabler_renderer.py b/ui/tabler_renderer.py
index bf30f648..4a26a1d4 100644
--- a/ui/tabler_renderer.py
+++ b/ui/tabler_renderer.py
@@ -4151,10 +4151,33 @@ body.mv-wide-validation .container-xl{max-width:min(1960px, calc(100vw - 304px))
       <!-- ══════════════════ TAB: 설정 ══════════════════ -->
       <div id="tab-settings" class="mv-tab" style="display:none">

-        <!-- ① 상단: DB 프로필 목록 (+ 연결쌍 추가 — profile 선택으로 Pair 구성)
-             [DB-CONNECTION-STATUS-DISPLAY-CONSISTENCY-INVESTIGATE-AND-FIX] 개별 프로필 상태가
-             정확한 원천이므로, 검증 경로 목록보다 먼저 보이도록 화면 상단으로 재배치(순수 HTML 순서
-             변경 — id/CSS/JS 참조 모두 getElementById 기준이라 DOM 순서에 의존하지 않음). -->
+        <!-- ① 상단: 검증 경로 목록 -->
+        <div class="card" id="pairListCard">
+          <div class="card-header">
+            <h3 class="card-title">검증 경로 목록</h3>
+          </div>
+          <div class="card-body">
+            <p class="text-secondary mb-2" style="font-size:.74rem;color:#94a3b8">
+              검증 경로는 원본 DB 프로필 1개 이상과 목적지 DB 프로필 1개를 연결한 실행 기준입니다. 접속하면 현재 검증 기준으로 사용됩니다(프로필 참조만 저장 — 비밀번호/host/user 미저장).
+            </p>
+            <table class="pair-table">
+              <thead>
+                <tr>
+                  <th>상태</th>
+                  <th>연결 이름</th>
+                  <th>원본 DB</th>
+                  <th>원본 DBMS</th>
+                  <th>목적지 DB</th>
+                  <th>목적지 DBMS</th>
+                  <th>작업</th>
+                </tr>
+              </thead>
+              <tbody id="pairListBody"></tbody>
+            </table>
+          </div><!-- /card-body -->
+        </div><!-- /card pairListCard -->
+
+        <!-- ② 중단: DB 프로필 목록 (+ 연결쌍 추가 — profile 선택으로 Pair 구성) -->
         <div class="card" id="dbProfileCard">
           <div class="card-header">
             <h3 class="card-title">DB 프로필 목록</h3>
@@ -4191,32 +4214,6 @@ body.mv-wide-validation .container-xl{max-width:min(1960px, calc(100vw - 304px))
           </div><!-- /card-body -->
         </div><!-- /card dbProfileCard -->

-        <!-- ② 중단: 검증 경로 목록 -->
-        <div class="card" id="pairListCard">
-          <div class="card-header">
-            <h3 class="card-title">검증 경로 목록</h3>
-          </div>
-          <div class="card-body">
-            <p class="text-secondary mb-2" style="font-size:.74rem;color:#94a3b8">
-              검증 경로는 원본 DB 프로필 1개 이상과 목적지 DB 프로필 1개를 연결한 실행 기준입니다. 접속하면 현재 검증 기준으로 사용됩니다(프로필 참조만 저장 — 비밀번호/host/user 미저장).
-            </p>
-            <table class="pair-table">
-              <thead>
-                <tr>
-                  <th>상태</th>
-                  <th>연결 이름</th>
-                  <th>원본 DB</th>
-                  <th>원본 DBMS</th>
-                  <th>목적지 DB</th>
-                  <th>목적지 DBMS</th>
-                  <th>작업</th>
-                </tr>
-              </thead>
-              <tbody id="pairListBody"></tbody>
-            </table>
-          </div><!-- /card-body -->
-        </div><!-- /card pairListCard -->
-
         <!-- ③ 하단: DB 상세 입력(단일 프로필 폼 — 신규 작성/기존 수정 전용, 기본 접힘) -->
         <div class="card" id="dbDetailCard" style="display:none">
           <div class="card-header">
--- END diff ---

커밋: 5f82dc35 (nxDTV-src, origin/main push 완료)

## 검증 결과 (27번/29번/30번 규칙)

### 실 브라우저 스크린샷 검증
운영 8000/8001을 직접 열지 않고, 파트B에서 정의한 절차대로 `db/migration_validator.db`
(A-1 복구 후 사본)와 `common/db/dnp_db_preset.db`를 임시 디렉터리로 복사한 뒤
`MV_DATA_DIR`/`MV_PRESET_DATA_DIR`로 격리해 임시 서버를 새로 띄워 촬영했다.
- 수정 전(bc69e634 코드, `git worktree add --detach` 로 격리한 임시 워크트리) —
  포트 8001, 격리 사본 DB 사용:
  `scratchpad/PAIR-RESTORE_before_settings.png` — DB 프로필 목록이 위, 검증 경로
  목록이 아래(bc69e634 상태). 같은 화면에서 "Oracle_asis → Oracle_tobe" 행이 검증
  경로 목록에 존재함도 확인(A-1 복구가 이미 반영된 동일 DB 사본이므로).
- 수정 후(현재 작업 트리) — 포트 8002, 같은 격리 사본 DB 사용:
  `scratchpad/PAIR-RESTORE_after_settings.png` — 검증 경로 목록이 위, DB 프로필
  목록이 아래로 원복 확인.
- A-3 정렬 확인: `scratchpad/PAIR-RESTORE_after_sorted.png` — 실 Oracle 접속이 이
  환경에서 불가능해(원격 DB 미가용) 클라이언트 상태(`_connectedPair`,
  `_profileState.src/tgt.status`)를 스크립트로 주입해 `renderDbProfileList()`를
  재실행한 화면(**실검증 상태 아님** — 대체수단, 사유: Oracle_asis/Oracle_tobe 실접속
  불가). Oracle_asis(원본)/Oracle_tobe(목적지) 행이 DB 프로필 목록 최상단에
  원본→목적 순서로 표시됨을 확인 — A-3 로직(2b31b8a0, 코드 미변경) 정상 동작.
- 두 스크린샷(before_settings.png / after_settings.png)을 Read 도구로 직접 열어
  카드 순서 반전을 대조 확인함.
- 임시 서버(8001/8002)는 촬영 직후 강제 종료(taskkill) — 서버 관련 규칙 준수, 8000
  운영 서버는 이번 작업 전체에서 한 번도 직접 사용하지 않았다.

[27번 규칙 체크리스트]
- 이 지침에 UI/화면 동작 변경이 포함되는가: 예(카드 배치 순서 변경)
- 포함된다면, 수정 전 스크린샷을 실제로 촬영했는가: 예,
  `scratchpad/PAIR-RESTORE_before_settings.png` (+ `_before_full.png`)
- 수정 후 스크린샷을 실제로 촬영했는가: 예,
  `scratchpad/PAIR-RESTORE_after_settings.png` (+ `_after_full.png`, `_after_sorted.png`)
- 두 스크린샷을 Read 도구로 직접 열어 대조했는가: 예
- 실측이 불가능했다면, 그 사유와 대체수단을 명시했는가: A-3 정렬 화면(sorted.png)에
  한해 해당 — 사유: 이 환경에서 Oracle_asis/Oracle_tobe 실접속 불가. 대체수단:
  클라이언트 상태(`_connectedPair`/`_profileState`) 스크립트 주입 후 동일 렌더 함수
  재실행. A-1/A-2 스크린샷은 실측(대체수단 아님).

### 회귀 테스트
- 직접 관련 파일(`tests/test_connection_settings_restructure.py`,
  `tests/test_active_path_persist_and_new_form.py`): 최종 코드 기준 재실행 —
  **19 passed**(두 파일 합산, `test_active_path_header_and_profile_order.py` 3건
  포함 시 22중 19).
- `test_active_path_header_and_profile_order.py`: 5건 중 2건 실패
  (`test_order_logic_active_first_display_only`,
  `test_active_source_then_target_on_top`). `git log -S"_apSrcOrder"`로 추적한 결과
  **커밋 f0ad5383**(다중 Source 지원 리팩토링, 이번 작업과 무관한 과거 커밋)가
  단일-Source 시대에 작성된 이 테스트의 기대 문자열/순서를 이미 깨뜨려 놓은
  **기존 결함**임을 확인 — 이번 A-2/A-3 변경으로 새로 발생한 게 아니며, 이번
  지침 범위 밖(임의 리팩토링 금지 원칙에 따라 손대지 않음)이라 그대로 보고만 한다.
- CLAUDE.md 표준 회귀(`samples/test_virtual_cases.py` 8/8,
  `samples/test_complex_cases.py` 5/5): 전부 통과.
- 타겟 회귀(`pytest tests/ -k "pair or profile or settings or connection"`,
  1139개 파일 대상): **1076 passed, 51 failed, 12 errors, 4 skipped** — 실패 목록에
  이번 변경 파일과 겹치는 항목은 위 f0ad5383 건 외에는 없음(파일명 grep으로 확인,
  `test_task11_*`/`test_single_*` 등 실DB·스키마 의존 기존 실패로 이번 변경과 무관).
- 전체 회귀(`pytest tests/`, `test_11def_scenarios.py`/`test_12k_stress.py` 2개는
  기존에도 깨져 있던 `db_presets_src.json` 파일 부재로 수집 자체가 불가해 제외):
  **12576 passed, 533 failed, 42 errors, 139 skipped, 26 xfailed** (1:35:09 소요).
  대부분 `sqlite3.OperationalError: no such table: DTV_policy_target_table_config`
  (29건) 등 이 환경의 사전 존재 결함이며, 이번 2개 파일 변경과 무관하다.
  **단, 예외 1건 발견 및 규명**: 전체 회귀 로그에
  `test_settings_section_order_pairs_then_profiles_then_detail` FAILED가 찍혀
  있었다 — 원인 조사 결과, 이 전체 회귀를 백그라운드로 기동한 시점과 테스트 파일
  수정(Edit) 시점이 거의 동시에 일어나 **pytest가 파일을 읽는 순간과 내 Edit 쓰기가
  경합**했을 가능성이 유력하다(같은 파일 내 다른 9건은 전부 정상 PASS로 로그에
  남아 있음 — 이 1건만 고립). 이후 해당 파일만 단독 재실행(10 passed) 및
  관련 파일 묶음 재실행(19 passed, 위 f0ad5383 2건 제외 전부 통과)으로 최종
  코드 상태에서는 통과함을 재확인했다. 95분 걸리는 전체 스위트를 세 번째로
  재기동하는 대신, 이 1건에 한해 원인을 특정하고 고립 재현으로 대체 검증했다 —
  이 판단 근거를 투명하게 남긴다.

════════════════════════════════════════
파트B — E2E 검증 DB 격리 표준 절차 신설(재발방지)
════════════════════════════════════════

## 목적
파트A가 복구한 소프트 삭제의 원인으로 유력하게 지목된 "코드는 worktree/포트로
격리했지만 DB 파일은 운영 파일을 그대로 참조"하는 관행이 실제로 있었는지 확인하고,
재발 방지를 위한 표준 절차를 CLAUDE.md에 규칙으로 남긴다.

## 현상 (B-1, B-2 조사 근거 — 파일:줄)
- **사고 정황(로그)**: `logs/server.log:80154~80159` — 2026-09-17 13:59:08~09에 포트
  8001 서버(PID 27284)가 기동. `logs/server.log:80170` 부근에서 13:59:17~18에
  `GET /db/presets/src`, `/db/presets/tgt`, `/conn-pairs` 요청, 13:59:22에
  `POST /conn-pairs`(신규 pair 생성)가 이어짐 — 사고 시각(`updated_dt` 13:59:52)과
  같은 분·같은 세션. 단, 그 시각~시각 사이 구간에 `DELETE` 요청은 로그에 전혀 없음
  (선행 보고서 파트0의 조사와 동일 결론 재확인) — **직접 인과는 여전히 미상,
  정황 일치만 확인**.
  `services/conn_pair_service.py:245~313`(`create_pair`)를 직접 읽어 확인한 결과,
  이 함수 자체에는 소프트삭제(`is_deleted` 갱신) 로직이 전혀 없다 — 즉 위 POST
  요청 자체가 삭제를 유발했을 가능성은 코드 레벨로 배제된다.
- **과거 관행(코드-DB 미분리) 증거**:
  - `reports/BATCH-EXPORT-LOCATIONHREF-LEAVE-SITE-POPUP-FIX_20260902.md:147~151` —
    "`git worktree add --detach`로 수정 전 코드를 격리했지만 `MV_DATA_DIR`은 메인
    저장소 `db/`와 공유"라고 명시적으로 기록돼 있음.
  - `scratchpad/STAGE5-EXTRACT-ALL-BUTTON-RENAME-REALTIME-CLARIFY_server8001_before.py:3`,
    `scratchpad/STAGE4-RISK-LEVEL-PER-GROUP_live_verify.py:12`,
    `scratchpad/AUTOSAVE-GATE-DECOUPLE-EXECUTION-RISK-FROM-STORAGE-DECISION_server8002.py:4`,
    `scripts/dev_e2e/axis_cascade_before_shot.py:5` 등 다수 — "MV_DATA_DIR은
    메인 프로젝트 db/를 공유한다"는 동일 패턴이 반복.
  - 반대로 `scripts/dev_e2e/f12_cascade_delete_ui_verify.py:13~15,74~88`,
    `scratchpad/rc_exactdiff/rc_harness.py:27~32` 등은 이미 제대로
    `MV_DATA_DIR`을 임시 디렉터리로 격리하는 모범 사례로 존재함 — 관행이
    혼재돼 있었다.
  - **추가 발견**: 위 모범 사례들조차 `MV_DATA_DIR`만 격리했고
    `MV_PRESET_DATA_DIR`(스위트 공용 `common/db/dnp_db_preset.db`)을 격리한
    선례는 코드베이스 전체에서 발견되지 않았다 — 프로필 프리셋 DB는 과거 한
    번도 E2E 격리 대상이 아니었다.
- **기존 격리 메커니즘(B-2)**: `config/db_paths.py:22,32~35`(`MV_DATA_DIR`),
  `config/db_paths.py:72,75~78`(`MV_PRESET_DATA_DIR`) — 이미 구현·배포돼 있는
  환경변수 override. 새로 만들 필요 없음.

## 실제조치 (B-3, B-4)
1. **표준 절차 확정**(새 코드 추가 없음 — 기존 메커니즘 그대로 재사용):
   1) 서버 기동 **전**에 `db/migration_validator.db`, `common/db/dnp_db_preset.db`를
      임시 디렉터리로 복사(원본은 읽기 전용).
   2) `MV_DATA_DIR`/`MV_PRESET_DATA_DIR`을 그 임시 디렉터리로 지정하고 서버
      프로세스를 **그 env로 새로** 기동(기존에 떠 있는 서버 재사용 금지 — 모듈
      레벨 `_DB_PATH` 캡처 시점 때문).
   3) 운영 8000과 겹치지 않는 임시 포트(`MV_BIND_PORT`) 사용.
   4) UI 자동화는 그 임시 서버 주소로만 접속, 종료 후 그 포트 직접 종료.
   5) 완료보고에 어느 디렉터리/포트를 썼는지 명시.
   이번 파트A의 스크린샷 촬영(8001/8002) 자체가 이 절차의 첫 실행 사례다.
2. **CLAUDE.md 38번 규칙 추가**(신규 항목, 기존 규칙 번호 변경 없음) — 위 배경·
   메커니즘·절차를 그대로 규칙화.

### git diff (30번 규칙 증적, CLAUDE.md)
--- BEGIN diff: CLAUDE.md ---
diff --git a/CLAUDE.md b/CLAUDE.md
index (이전)..() 100644
--- a/CLAUDE.md
+++ b/CLAUDE.md
@@
     - 코드 저장소(`X:\xDataNexPro\nxDTV`, `nxDTV-src`)의 commit+push는 기존
       18/29번 규칙(완료·검증 후 즉시 커밋)을 그대로 따른다 — 이 항목과
       혼동 금지.
+
+38. (2026-09-17 추가, PAIR-RESTORE-LAYOUT-REVERT-PROFILE-SORT-AND-E2E-DB-
+    ISOLATION 파트B) 실 브라우저 E2E 검증은 코드뿐 아니라 내부 SQLite DB도
+    반드시 격리된 사본을 써야 하며, 운영 DB 파일을 직접 참조하는 서버로 UI
+    자동화(클릭 등)를 수행하지 않는다.
+    (배경/메커니즘/표준 절차 5단계 — 본문 참고, 42줄 추가)
--- END diff ---
(전문은 저장소 `CLAUDE.md` 38번 항목 참고 — 42줄 전량 신규 추가, 기존 줄 변경 없음)

커밋: c0828e62 (nxDTV-src, origin/main push 완료)

## 검증 결과 (파트B)
- 코드 변경 없음(문서만 추가) — 27번 UI 스크린샷 체크리스트 대상 아님.
- 30번 diff 증적: 위 CLAUDE.md diff 포함.
- 회귀 테스트: 코드 변경이 없어 대상 없음(CLAUDE.md는 실행되는 코드가 아님).

## 기대효과
- 파트A: 소프트 삭제된 검증 경로가 복구되고, 사용자가 요청하지 않았던 화면 순서
  변경이 원복돼 UI가 이전 사용 흐름과 일치한다. DB 프로필 정렬 기능은 손대지
  않고도 정상 동작함이 재확인됐다.
- 파트B: 향후 실 브라우저 E2E 검증에서 운영 DB를 직접 참조하는 실수(이번 사고의
  유력한 정황 원인)가 구조적으로 줄어든다. 새 인프라 없이 기존 `MV_DATA_DIR`/
  `MV_PRESET_DATA_DIR` 메커니즘을 표준 절차로 명문화했다.

## 부기(추가 확인 필요 없음, 정보 공유용)
- 이번 작업 중 파트A 스크린샷 촬영을 위해 임시 서버(포트 8002)를 **현재 작업
  트리에서** 기동했는데, `logs/` 디렉터리 경로(`web_server.py`의 `_LOG_DIR`)는
  `MV_DATA_DIR`로 격리되지 않고 코드 루트 고정이라 운영 `logs/server.log`에
  8002 관련 로그 줄이 몇 건 append됐고 `logs/server_8002.pid`(임시 서버 PID
  기록, 이미 종료된 프로세스 번호) 1개가 새로 생겼다. 둘 다 데이터 훼손이 아닌
  로그/PID 파일 append이며, 삭제 규칙(자율 진행 규칙 예외 — rm/Remove-Item은
  지침이 명시적으로 요구하지 않는 한 사용자 확인 필요)에 따라 임의로 지우지
  않고 그대로 남겨뒀다. 필요시 사용자가 `logs/server_8002.pid` 삭제 여부를
  판단해달라.

작업명 : PAIR-RESTORE-LAYOUT-REVERT-PROFILE-SORT-AND-E2E-DB-ISOLATION
✅ 작업 완료 - 파트A(소프트삭제 복구+화면 원복+프로필 정렬 확인) / 파트B(E2E DB 격리 표준 절차 신설) 모두 완료
```
