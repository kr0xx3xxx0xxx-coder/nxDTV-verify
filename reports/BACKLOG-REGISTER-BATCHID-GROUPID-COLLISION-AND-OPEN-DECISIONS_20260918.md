```text
작업명 : BACKLOG-REGISTER-BATCHID-GROUPID-COLLISION-AND-OPEN-DECISIONS
✅ 작업 완료 - BACKLOG-REGISTER-BATCHID-GROUPID-COLLISION-AND-OPEN-DECISIONS

**목적**
최근 조사(QUERY-UNIQUENESS-AXIS-VERIFY-PROJECT-GROUP-BATCH-ROWNUM,
GROUP-ID-COLLISION-RISK-PARITY-WITH-BATCH-ID-VERIFY,
PROJECT-SOFT-DELETE-CHILD-GROUP-CASCADE-CONSISTENCY-VERIFY) 및
2026-09-18 사용자 대화에서 확인됐으나 아직 실행/결정되지 않은 항목
4건을 nxDTV-verify BACKLOG.md에 정식 등록해 기록 누락을 방지한다.

**현상**
아래 4건이 백로그에 등록되지 않은 채 조사 보고서/대화 기록에만
흩어져 있었다.
1. batch_id 생성(datetime 마이크로초, 락/재시도 없음)이 `INSERT OR
   REPLACE`와 결합돼 충돌 시 이전 배치 이력이 조용히 덮어써질 위험
   (우선순위 높음 — 업로드마다 자동 발급).
2. group_id도 동일 클래스 결함이나 PK+일반 INSERT라 충돌 시 명시적
   500 실패이고 생성 빈도도 낮음(우선순위 낮음, 참고).
3. project_id/group_id/batch_id를 3자리 텍스트로 통일하자는 사용자
   제안 — 검토 결과 반대 근거 3가지 확인, 사용자도 보류에 동의.
4. 프로젝트 소프트 삭제 시 하위 그룹의 `is_deleted` 값 자체는 전파
   안 됨(조회/실행 시점 재확인 방식으로 안전은 이미 확인됨) — 값
   비전파 자체는 범위 밖 참고사항.

**실제조치**
- 공유 워킹트리(`X:\xDataNexPro\nxDTV-verify-clone`)에서 `git status`로
  다른 세션의 미커밋 변경이 없음을 확인(clean, origin/main과 동기화
  상태) — 임시 worktree 없이 바로 작업.
- BACKLOG.md 최고 번호(M399) 다음으로 4건을 M400~M403으로 등록:
  - M400. 아이디어(미착수, 우선순위 높음) - BATCH-ID-COLLISION-RISK-FIX
  - M401. 아이디어(미착수, 우선순위 낮음, 참고) - GROUP-ID-COLLISION-RISK-FIX
  - M402. 아이디어(보류, 사용자 판단 필요) - PROJECT-GROUP-BATCH-ID-SCHEME-UNIFY-3DIGIT-TEXT-PROPOSAL
  - M403. 참고(범위 밖, 기능 안전은 이미 확인됨) - PROJECT-SOFT-DELETE-CHILD-GROUP-DB-VALUE-NOT-PROPAGATED
- `git diff --stat`으로 BACKLOG.md 1개 파일만 변경됐음을 확인.
- add → commit → `git fetch origin main`(선행 커밋과 동일, 그 사이
  다른 push 없음 확인) → `git push origin HEAD:main`.

**검증 결과**
- git diff --stat: `BACKLOG.md | 55 +++++++++++++++++++++++++++++++++++++++++++++++++++++++` (1 file changed, 55 insertions(+))
- 커밋 해시: 10279ba08dc3b4e37d192b50039f71b702293759
- push 로그: `473ebdd..10279ba  HEAD -> main` (github.com:kr0xx3xxx0xxx-coder/nxDTV-verify.git)
- 신규 부여 백로그 번호: M400, M401, M402, M403

**기대효과**
4건의 미실행/미결정 항목이 백로그에 정식 등록되어, 추후 세션에서
근거 보고서 없이도 우선순위와 배경을 즉시 파악하고 착수 여부를
판단할 수 있다. 특히 M400(BATCH-ID-COLLISION)은 발생 빈도가 높은
실사용 리스크로 후속 착수 대상 후보로 바로 추적 가능하다.

작업명 : BACKLOG-REGISTER-BATCHID-GROUPID-COLLISION-AND-OPEN-DECISIONS
✅ 작업 완료 - BACKLOG-REGISTER-BATCHID-GROUPID-COLLISION-AND-OPEN-DECISIONS
```
