---
name: gn-dd2-orchestrator
description: GameNative-DD2 fork 의 Workshop 브라우저 기능 작업을 도메인별 전문 에이전트에 분배·조율한다. Workshop·Steam Workshop·창작마당·Subscribe·QueryFiles·GetDetails·PublishedFile·browser UI·upstream sync·Fold 7 검증·release 빌드 — GameNative-DD2 저장소 관련 모든 요청에서 반드시 이 스킬을 사용하라. 후속 작업("다시 실행", "수정", "보완", "이전 결과 기반", "{부분작업}만 다시")도 동일하게 트리거. D:/git/GameNative-DD2 작업이면 단순 정보 질문이 아닌 한 모두 이 스킬로.
---

# GameNative-DD2 Orchestrator

3명의 전문가(`gamenative-fork-maintainer`, `workshop-integrator`, `device-tester`) 중 작업 성격에 맞는 1~2명을 호출한다. 이전 DD2-Launcher 풀-스크래치 빌드 하네스(7-에이전트)는 폐기됨 — 본 하네스는 **fork 한정 + Workshop 브라우저 추가** 작업에 특화.

**실행 모드:** 하이브리드 — 단일 도메인이면 서브 에이전트, 2개 이상이면 팀.

## Phase 0: 컨텍스트 확인

1. `D:/git/GameNative-DD2/_workspace/` 존재 여부 확인
2. 사용자 요청 형태 분석:
   - "이전 결과", "수정", "보완", "다시" → **부분 재실행** 모드
   - 새 도메인 + 기존 산출물 존재 → **새 실행** (기존 `_workspace/` → `_workspace_prev_<date>/`)
   - `_workspace/` 미존재 → **초기 실행** (디렉토리 생성)
3. 자매 메모리 읽기:
   - `reference_gamenative_dd2_validation_2026_05_04.md` (검증 결과)
   - `reference_gamenative_emulator.md`, `reference_dd2_engine_findings.md`
   - `project_sts2_launcher_workflow.md` (APK 빌드·키스토어·adb 패턴)

## Phase 1: 도메인 분류

| 도메인 | 키워드·신호 | 담당 |
|---|---|---|
| Workshop 기능 | "Workshop", "창작마당", "Subscribe", "QueryFiles", "GetDetails", "PublishedFile", "browser", "browse", "검색", "구독", "폴더 아이콘" | `workshop-integrator` |
| Fork 운영 | "upstream", "sync", "rebase", "merge", "v0.9.0", "릴리스", "키스토어", "버전 핀" | `gamenative-fork-maintainer` |
| 기기 검증 | "Fold 7", "8 Elite", "adb", "logcat", "FPS", "발열", "스크린샷", "재현", "회귀" | `device-tester` |

**다중 도메인** (현실에서 자주 발생):
- "Workshop 화면 fold 7 에서 깨짐" → workshop + device (2명)
- "GameNative 새 v0.9.2 sync 후 Workshop 회귀 검증" → fork + workshop + device (3명)
- "release 빌드해서 fold 7 install" → fork + device (2명)

## Phase 2: 실행 모드 선택

| 조건 | 선택 |
|---|---|
| 단순 질문 (코드 변경 미발생) | **메인이 직접 응답** (에이전트 호출 X) |
| 단일 도메인 + 코드 변경 | **단일 서브 에이전트** (Agent + model: opus) |
| 매핑 도메인 2~3개 | **에이전트 팀** (TeamCreate) |

모든 에이전트 호출 시 `model: "opus"` 강제.

## Phase 3: 작업 분배

### 팀 모드
1. `TeamCreate(team_name="dd2ws-<task>", members=[...])`
2. `TaskCreate` 로 단계별 작업 분해. 일반 의존: 정찰 → 구현 → 빌드 → 검증.
3. 산출물 컨벤션: `_workspace/<topic>_*.md` (행 추가만, 덮어쓰기 금지).

### 서브 에이전트 모드
`Agent(subagent_type="general-purpose", model="opus", prompt=...)` 직접 호출.

## Phase 4: 결과 통합

1. `_workspace/` 산출물 종합해서 사용자에게 결론.
2. 후속 작업 가능성 안내.
3. 결정·발견 사항이 메모리화 가치 있으면 `~/.claude/projects/D--git/memory/` 에 추가 제안.

## Phase 5: 피드백 루프

매 작업 종료 후 1회 묻기. 피드백 없으면 넘어감. 수정 대상 결정:
- 결과 품질 → 해당 에이전트 정의·스킬
- 워크플로우 순서 → 본 오케스트레이터
- 트리거 누락 → description 확장

모든 변경은 `D:/git/GameNative-DD2/CLAUDE.md` 변경 이력에 기록.

## 데이터 전달 프로토콜

| 전략 | 사용 |
|---|---|
| TaskCreate / TaskUpdate | 작업 조율, 의존 |
| SendMessage | 팀 모드 실시간 질의 |
| 파일 (`_workspace/`) | 영구 산출물 — 모든 정찰·매핑·검증 결과 |
| 반환값 | 서브 에이전트 단일 결과 |

`_workspace/` 컨벤션:
- `_workspace/published_file_rpc_callsites.md` (`workshop-integrator`)
- `_workspace/workshop_ui_state_machine.md` (`workshop-integrator`)
- `_workspace/workshop_manager_existing_vs_new.md` (`workshop-integrator`)
- `_workspace/upstream_sync_log.md` (`gamenative-fork-maintainer`)
- `_workspace/release_history.md` (`gamenative-fork-maintainer`)
- `_workspace/device_test_runs/<date>/` (`device-tester`)

## 에러 핸들링

| 상황 | 대응 |
|---|---|
| 에이전트 1회 실패 | 1회 재시도. 재실패 시 결과 누락 표기 |
| 라이선스 호환 불가 (의존성 추가 시) | 즉시 차단, 사용자 결정 위임 |
| upstream merge 충돌이 우리 영역 | cherry-pick 시도, 안 되면 사용자 결정 |
| Workshop RPC 응답이 throttle/error | retry/backoff, 사용자 메시지 |
| Fold 7 + 신생 드라이버 결함 (메모리 `project_sts2_issue11_fold_findings.md` 패턴) | 코드 수정 보류, 드라이버 행 broken 표기 |
| 데이터 손실 위험 | 메모리 `feedback_reproduce_before_fix.md` 강제 |

## 테스트 시나리오

### 시나리오 1: Workshop QueryFiles RPC 추가
- 입력: "QueryFiles 호출 코드 작성"
- Phase 1: workshop (단일)
- Phase 2: 서브 에이전트 (`workshop-integrator`)
- 산출물: `workshop/WorkshopBrowser.kt` 부분 + `_workspace/published_file_rpc_callsites.md` 갱신

### 시나리오 2: Fork sync 후 회귀 검증
- 입력: "upstream master 신규 변경 sync 하고 Fold 7 회귀 확인"
- Phase 1: fork + device (2 도메인)
- Phase 2: 팀 — `TeamCreate(dd2ws-sync-regression, [gamenative-fork-maintainer, device-tester])`
- TaskCreate: (1) fork-maintainer 가 cherry-pick → (2) build → (3) device-tester 가 회귀 시나리오

### 시나리오 3: Workshop 브라우저 화면 폴드 7 종횡비 확인
- 입력: "Workshop 브라우저가 폴드 7 언폴드에서 깨지는지 확인"
- Phase 1: workshop + device
- Phase 2: 팀 — `TeamCreate(dd2ws-fold7-ui, [workshop-integrator, device-tester])`
- 산출물: `_workspace/device_test_runs/<date>/workshop_browser_fold7.png`

### 시나리오 4: 후속 작업 — 부분 재실행
- 입력: "이전 RPC 호출 패턴에서 Subscribe 만 수정"
- Phase 0: `_workspace/published_file_rpc_callsites.md` 존재 → 부분 재실행
- Phase 2: 서브 에이전트 (`workshop-integrator` 단독)
- 산출물: 해당 섹션만 갱신, 나머지 보존
