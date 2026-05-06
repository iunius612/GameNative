# GameNative-DD2 — Steam Workshop Browser Fork

`utkarshdalal/GameNative` 의 fork. **Darkest Dungeon 2 (Steam AppID 1940340)** 사용자가 모바일 앱 안에서 Steam Workshop 을 직접 브라우즈·검색·구독할 수 있게 하는 기능을 추가한다. PC Steam 클라이언트에서 사전 구독하지 않아도 모바일에서 모드 발견·구독 가능.

## 포크 관계

- **upstream**: https://github.com/utkarshdalal/GameNative (GPL 3.0)
- **origin (this fork)**: https://github.com/iunius612/GameNative (GPL 3.0)
- **베이스라인**: `v0.9.0` 태그 (sha `24508c2aa125870988f2815a7f75b03db97cd0db`) — 사용자 실기 검증 통과 버전, ControllerTab 의 마우스 입력 비활성화/터치스크린 모드가 직접 노출되는 마지막 stable. main HEAD 는 일부 UI 회귀 의심.
- **작업 브랜치**: `feature/workshop-browser` (v0.9.0 분기)

## 핵심 추가 기능

| 기능 | 구현 방식 |
|---|---|
| Workshop **브라우즈** (정렬·태그·날짜) | `PublishedFile.QueryFiles` RPC (JavaSteam 기존 SteamUnifiedMessages) |
| Workshop **상세** (썸네일·설명·평점·작성자) | `PublishedFile.GetDetails` RPC |
| **구독**·구독해제 | `PublishedFile.Subscribe`/`Unsubscribe` RPC |
| 구독 후 다운로드·설치 | **기존 `WorkshopManager.kt` (3,635줄) 재사용** — 변경 없음 |
| UI 진입점 | 게임 디테일 화면, EditContainer 톱니바퀴 좌측에 폴더 아이콘 추가 |

**핵심 결정**: Steam Web API · steamcommunity 쿠키 · Steam-in-Wine 프로토콜 URL 모두 **불필요**. JavaSteam 의 `PublishedFile` 서비스가 29개 RPC 메서드 제공 (Subscribe/Unsubscribe/QueryFiles/GetDetails/Vote/CanSubscribe 등) — GameNative 의 기존 `SteamClient` 인증 세션 그대로 사용.

## 하네스: Workshop Browser

**목표:** Workshop 브라우저 기능 추가를 위한 도메인별 작업 분배.

**트리거:** Workshop·Steam Workshop·창작마당·Subscribe·QueryFiles·GetDetails·PublishedFile·brower UI·upstream sync 관련 작업 시 `gn-dd2-orchestrator` 사용.

**관련 메모리 (자매 프로젝트 + 검증 결과 상속):**
- `reference_gamenative_dd2_validation_2026_05_04.md` — Fold 7 검증 통과 스택, Workshop UI 기존 위치, HEAD↔v0.9.0 차이
- `reference_gamenative_emulator.md` — GameNative 지형 (Pluvia 후신, GPL 3.0)
- `reference_dd2_engine_findings.md` — DD2 엔진 (Mono + Burst + Steamworks)
- `project_sts2_launcher_workflow.md` — APK 빌드·키스토어·adb 패턴

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-05-04 | 초기 구성 | 전체 | fork 셋업 + 3-에이전트 하네스 |
