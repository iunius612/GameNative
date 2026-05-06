# Workshop Browser UI 진입점 정찰

**파일 핀**: D:/git/GameNative-DD2 @ feature/workshop-browser (off v0.9.0 sha 24508c2a)

## 톱니바퀴 아이콘 정확한 위치

**파일**: `app/src/main/java/app/gamenative/ui/screen/library/LibraryAppScreen.kt`
**라인**: 866-870

```kotlin
// Secondary action icons (right-aligned)
ActionIconButton(
    icon = Icons.Default.Settings,
    contentDescription = stringResource(R.string.options),
    onClick = { optionsMenuVisible = true },
)
```

**현재 onClick 동작**: `optionsMenuVisible = true` — 슬라이드 인 `GameOptionsPanel` 열림. 그 안에서 사용자가 카테고리 펼쳐서 `ManageWorkshop` 항목 찾아야 Workshop 도달 (HELP_INFO 그룹에 묻혀 있음, `GameOptionsPanel.kt:397`).

## 주변 Row 컨텍스트

`LibraryAppScreen.kt:740-879` 의 Row 구조 (간략화):

```
Row {
    PrimaryActionButton(Play / Install)        // line 800-832
    if (isDownloading) Column { 사이즈 / ETA }  // line 834-860
    else Spacer(weight=1f)                     // line 862
    
    // Secondary action icons (right-aligned)
    ActionIconButton(Icons.Default.Settings)   // line 866 ← 톱니바퀴
    if (isInstalled || hasPartialDownload) {
        ActionIconButton(Icons.Default.Delete) // line 873 ← 휴지통
    }
}
```

사용자 요구: 톱니바퀴 **좌측** 에 폴더 아이콘 추가 → line 866 직전 (`Spacer/Column` 다음, `ActionIconButton(Settings)` 직전) 에 새 `ActionIconButton(Folder)` 삽입.

## 변경 영역 (구체)

### Insert (LibraryAppScreen.kt:865 직전)

```kotlin
// Workshop browse shortcut — DD2 fork addition
ActionIconButton(
    icon = Icons.Default.Folder,                           // 또는 Icons.Outlined.Folder
    contentDescription = stringResource(R.string.workshop_browse),
    onClick = onWorkshopBrowseClick,
)
```

### LibraryAppScreen 의 새 callback parameter

LibraryAppScreen 시그니처에 추가:
```kotlin
onWorkshopBrowseClick: () -> Unit,
```

호출자에서 navigation 연결 (예: `LibraryScreen.kt` 또는 `LibraryAppContent`):
```kotlin
onWorkshopBrowseClick = { navController.navigate(Screen.WorkshopBrowser(libraryItem.appId)) }
```

### 신규 import

`LibraryAppScreen.kt` 상단 import 추가:
```kotlin
import androidx.compose.material.icons.filled.Folder
```

### strings.xml 추가

`app/src/main/res/values/strings.xml`:
```xml
<string name="workshop_browse">Browse Workshop</string>
```

`app/src/main/res/values-ko/strings.xml` (이미 존재 시):
```xml
<string name="workshop_browse">창작마당 둘러보기</string>
```

## 가시성 조건

폴더 아이콘은 다음 조건일 때만 노출 권장:
- 게임이 Steam 게임 (`gameSource == GameSource.STEAM`) — Workshop 은 Steam 전용
- 게임이 Workshop 지원 (`steamApp.workshop != null` 또는 유사 플래그)
- 게임 설치 여부와 무관 (Workshop 모드는 설치 전이라도 구독 가능)

이 조건은 `LibraryAppScreen` 호출자 측에서 판단해 `onWorkshopBrowseClick` 을 nullable 로 전달하거나, 본 composable 안에 분기.

DD2 (AppID 1940340) 는 Workshop 지원 확인됨 (이전 검증 — Steam 라이브러리에 OUT NOW! 마커 = 와이파이 다운로드 게이트, Workshop 지원과 별개).

## 호출 후 동선 (Workshop Browser 화면)

새 화면 신설:
- 라우트: `Screen.WorkshopBrowser(appId: Int)`
- 파일 위치 (예정): `app/src/main/java/app/gamenative/ui/screen/workshop/WorkshopBrowserScreen.kt`
- ViewModel: `WorkshopBrowserViewModel`
- Repository (RPC 호출 래퍼): `app/src/main/java/app/gamenative/workshop/WorkshopBrowser.kt` (object, suspending fun)

화면 구조:
- Search bar (검색어 입력)
- Filter / Sort (인기 / 신규 / 평점 / 태그)
- Grid (썸네일 + 제목 + 평점 + 구독자수)
- 무한 스크롤 (페이지네이션)
- 아이템 클릭 → `Screen.WorkshopDetail(publishedFileId: Long)`

## 기존 ManageWorkshop 메뉴 항목 처리

`AppOptionMenuType.ManageWorkshop` (이미 존재, `Icons.Default.Build` 렌치 아이콘) 은 **그대로 유지**. 동작:
- 본 fork 신규 Folder 아이콘 = **새 모드 발견·구독** (Browser → Detail → Subscribe)
- 기존 ManageWorkshop = **이미 구독한 모드 관리** (활성/비활성, 삭제, 업데이트)

두 진입점이 명확히 다른 작업을 가리키므로 공존 가능.

## ActionIconButton 정의 위치

`LibraryAppScreen.kt:292` (private composable). 본 fork 가 호출자 추가만 하므로 정의 자체는 변경 불필요.

## 테스트 시나리오

1. DD2 게임 디테일 화면 진입 → 톱니바퀴 좌측에 폴더 아이콘 노출 확인
2. 폴더 클릭 → WorkshopBrowserScreen 진입 → DD2 Workshop 아이템 그리드 로드
3. 검색·정렬 동작
4. 아이템 클릭 → Detail → Subscribe 버튼 → 구독 성공 토스트
5. 게임 시작 시 기존 `WorkshopManager` 가 새 구독 자동 다운로드·배치
6. 폴드 7 unfolded(2208x1768) / cover(1080x2520) 모두 레이아웃 정상

## Open questions (구현 단계에서 결정)

- 게임이 Workshop 미지원일 때 폴더 아이콘 숨김 vs 비활성 회색 — 일관성 위해 숨김 권장
- 검색 결과 캐싱 정책 (메모리만 vs Room DB) — 우선 메모리 only, 필요시 확장
- 썸네일 이미지 로딩 (Coil 이미 의존성?) — `gradle/libs.versions.toml` 확인 필요
- Workshop 페이지 자체 에러 (Steam 점검 등) 처리 UX
