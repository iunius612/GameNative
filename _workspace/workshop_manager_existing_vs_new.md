# WorkshopManager.kt 기능 매핑 — 기존 vs 신규

**파일 핀**: D:/git/GameNative-DD2 @ feature/workshop-browser (off v0.9.0 sha 24508c2a)
**대상 파일** (전부 절대경로):
- `app/src/main/java/app/gamenative/workshop/WorkshopManager.kt` (3,279줄 — v0.9.0 분기 시점 기준)
- `app/src/main/java/app/gamenative/workshop/WorkshopItem.kt`
- `app/src/main/java/app/gamenative/ui/component/dialog/WorkshopManagerDialog.kt`
- `app/src/main/java/app/gamenative/ui/screen/library/appscreen/SteamAppScreen.kt`
- `app/src/main/java/app/gamenative/ui/PluviaMain.kt`
- `app/src/main/java/app/gamenative/service/SteamService.kt`

## ⚠️ 결정적 의미 발견

기존 `WorkshopManagerDialog.kt` 는 **신규 구독 추가 UI 가 아니다** — 이미 PC Steam 클라이언트에서 구독한 항목 중 *어떤 것을 모바일 컨테이너에 설치/심볼릭링크할지* 선택하는 토글 UI (구독자 selector). 따라서 fork 의 "신규 브라우저 + Subscribe RPC" 는 **진정한 추가 기능** 이며, 기존 UI 와 직접 충돌하지 않는다.

## 섹션 1 — 기존 함수 inventory (호출자 + 종속)

| Symbol (파일:줄) | 역할 | 호출자 | 종속/외부 |
|---|---|---|---|
| `WorkshopManager` (object) `:63` | 단일 싱글턴 | 전역 | SteamClient, DepotDownloader, PubFileItem, ImageFs, SteamService |
| `getSubscribedItems(appId, steamClient, steamId): WorkshopFetchResult` `:84-132` | `PublishedFile.GetUserFiles type=mysubscriptions` 페이지네이션 (page≤50, perPage=100) | `WorkshopManagerDialog.kt:106`, `:3056`, `:3193` | SteamUnifiedMessages, PublishedFile, CPublishedFile_GetUserFiles_Request |
| `fetchSubscribedFilesViaRPC(...)` `:143-219` | private — 단일 페이지 RPC + 30s timeout + EResult.OK 검사 | `getSubscribedItems` only | withTimeoutOrNull, toFuture().await() |
| `cleanupUnsubscribedItems(items, dir)` `:226-252` | 구독 목록 외 on-disk 디렉토리(+`.partial`) 삭제 | `:3074, :3214`, 테스트 | File.deleteRecursively |
| `getItemsNeedingSync(items, dir)` `:264-322` | `.workshop_complete` 마커 + `timeUpdated` 비교로 sync 필요 항목 필터 | `:3077, :3216` | — |
| `updateMarkerTimestamps(items, dir)` `:329-348` | 재다운로드 방지 마커 갱신 | `:2972, :3253, :3266` | — |
| `fixItemFileNames`, `extractCkmFiles`, `fixFileExtensions`, `decompressLzmaFiles`, `detectExtension` `:359-657` | 다운로드 후 파일 정규화 (확장자 복구, CKM 분해, LZMA 해제) | `runPostProcessing` | LZMAInputStream, magic-byte detection |
| **`downloadItems(...)`** `:674-959` | **핵심 다운로드** — HTTP / Depot 분기, Semaphore 동시성, prefetch, `.workshop_complete` 마커 | `:3104, PluviaMain.kt:1959` | DepotDownloader, IDownloadListener, PubFileItem, OkHttp |
| `computeDownloadTimeout`, `computeDownloadThreads` `:963-987` | 파일 크기·`PrefManager.downloadSpeed` 기반 튜닝 | `downloadItems` | PrefManager |
| `downloadPreviewImage`, `downloadViaHttp` `:994-1124` | 프리뷰 이미지 + HTTP fallback (Range resume) | `downloadItems` | okhttp3.Request |
| `getWorkshopContentDir(winePrefix, appId)` `:1135-1140` | gbe_fork 경로: `<prefix>/drive_c/Program Files (x86)/Steam/steamapps/workshop/content/<appId>/` | 곳곳 | — |
| `getContainerWinePrefix(context, appId)` `:1149-1154` | private — `xuser` symlink 우회, 컨테이너 고유 prefix | 내부 | ImageFs |
| `deleteWorkshopMods(...)` `:1166-1204` | 컨테이너 워크샵 + `mods.json`, `mod_images/` 일괄 삭제 | `SteamAppScreen.kt:1276` | cleanupInstalledModEntries, ContainerUtils |
| `cleanupInstalledModEntries`, `revertGameInfoPatch` `:1211-1366` | 게임 트리 symlink/copy 회수 | deleteWorkshopMods | walkTopDown, getOrDetectStrategy |
| `routeItemsToUnityTargets`, `detectUnityModTargets`, `syncSkyrimWorkshopMods` `:1593-1795` | 게임 엔진별 (Unity/Skyrim) 모드 routing | configureModSymlinks 내부 | WorkshopModPathStrategy |
| `strategyCacheFile`, `saveStrategyCache`, `loadCachedStrategy`, `getOrDetectStrategy` `:1796-1933` | 게임별 mod-path 전략 캐시 | cleanupInstalledModEntries, configureModSymlinks | WorkshopModPathStrategy/Detector |
| **`configureModSymlinks(...)`** `:1934-2899` (≈960줄) | 게임 폴더 symlink/copy + gbe_fork mods.json/mod_images. **가장 큰 함수**. Source/Skyrim/Unity 분기 | configureSymlinksForApp:2985 | 다수 |
| `patchSupportedWorkshopFileTypes()` `:2900` | `EWorkshopFileType` enum 확장 (1회) | `downloadItems` (depot 경로) | reflection on enum |
| `removeSymlinksIn`, `clearModEntries` `:2930-2948` | private — symlink 청소 헬퍼 | 내부 | Files.isSymbolicLink |
| `parseEnabledIds(idsString): Set<Long>` `:2951-2952` | DB CSV → Set | `SteamService.kt:3474, PluviaMain.kt:1873, SteamAppScreen.kt:471/1251` | — |
| **`runPostProcessing(...)`** `:2955-2973` | 다운로드 후: rename → CKM extract → LZMA decompress → ext fix → marker | `:3130, PluviaMain.kt:1999, :3219/3254` | 위 헬퍼들 |
| `configureSymlinksForApp(context, appId, ...)` `:2976-2992` | configureModSymlinks 의 호출자 친화 wrapper | `:3081/3134, PluviaMain.kt:2005, :3208/3220/3255` | SteamService.getAppDirPath/getAppInfoOf |
| `checkDiskSpace(dir, requiredBytes)` `:2998-3007` | 디스크 부족 시 에러 메시지 (`null`=OK) | `:3091, PluviaMain.kt:1945` | — |
| **`startWorkshopDownload(appId, enabledIds, context): DownloadInfo?`** `:3024-3170` | Save 직후 백그라운드 다운로드 시작. DownloadInfo 즉시 반환 → SteamService.downloadJobs 등록 → 진행률 UI 자동 연동 | `SteamAppScreen.kt:475/1271, SteamService.kt:3482` (재시작 후 pending 재개) | DownloadInfo, SteamService.set/notifyDownloadStarted, appDao.setWorkshopDownloadPending |
| `checkForWorkshopUpdates(...)` `:3185-3276` | **게임 launch 직전** 호출. fresh fetch + sync 필요 + `trulyMissing` 검증 (timestamp drift 흡수) | `PluviaMain.kt:1904` | getSubscribedItems, getItemsNeedingSync, runPostProcessing |
| `getUpdateThresholdBytes()` `:3278` | 100MB 임계 (이상이면 사용자 confirm) | `PluviaMain.kt:1912` | const |

## 섹션 2 — fork 가 신규 추가할 함수 (제안)

> 신규 코드는 **별도 파일 `WorkshopBrowser.kt`** 권장 (기존과 다른 책임). 또는 `WorkshopManager` object 에 추가도 가능.

| 신규 함수 | 역할 | 기존 인프라 의존도 |
|---|---|---|
| `queryWorkshopItems(appId, query, sortBy, page, perPage): WorkshopQueryResult` | `PublishedFile.QueryFiles` (검색·정렬·태그) — 신규 모드 발견 | `SteamUnifiedMessages.createService<PublishedFile>()` 그대로. **`SteamID` 불필요** (검색은 anonymous 가능) |
| `getWorkshopItemDetails(publishedFileIds: List<Long>)` | `PublishedFile.GetDetails` — 썸네일·설명·평점·작성자 | 동일 RPC 템플릿. `WorkshopItemDetail` 신규 데이터 클래스 |
| `subscribeToWorkshopItem(appId, publishedFileId): EResult` | `PublishedFile.Subscribe` | 동일 템플릿. **검증됨: Subscribe 만으로는 자동 다운로드 X** (섹션 4) |
| `unsubscribeFromWorkshopItem(appId, publishedFileId): EResult` | `PublishedFile.Unsubscribe` | 동일. 호출 후 `cleanupUnsubscribedItems` + `configureSymlinksForApp(items=newList)` 트리거 권장 |
| `triggerDownloadAfterSubscribe(appId, publishedFileId, context)` | Subscribe 직후 단일 항목 다운로드 시작 | **`startWorkshopDownload` 패턴 모방** (또는 enabledIds 갱신 후 그대로 호출 — 코드 중복 X) |
| UI: `WorkshopBrowserDialog` (또는 Screen) | 검색·필터·페이지네이션·상세 | `WorkshopManagerDialog.kt` 의 LaunchedEffect/Coil/list 패턴 참고 |
| UI 진입점 | EditContainer 톱니바퀴 좌측 폴더 아이콘 — `LibraryAppScreen.kt:866-870` 직전 (`AppOptionMenuType` 변경 X — 별도 진입점) | `displayInfo.gameId, SteamService.instance?.steamClient` |

## 섹션 3 — 절대 손대지 말 영역 ("Don't Break" 핫스팟)

깨질 경우 다운로드 파이프라인 전체 무력화. **읽기 전용으로만 참조**:

1. **`downloadItems` `:674-959`** — HTTP/Depot 분기, semaphore, 파이프라인 pre-start (`pending: Triple<...>`), DepotDownloader close 정책 (`openedDDs` deferred close), `.workshop_complete` 마커 timing. 신규 코드는 **`startWorkshopDownload` 호출만** — `downloadItems` 직접 호출 절대 금지.
2. **`configureModSymlinks` `:1934-2899`** — 게임별 분기 (Source/Skyrim/Unity/Insurgency/L4D2/RimWorld 등). DD2 는 Unity → `routeItemsToUnityTargets` 경로 가능성. 변경 시 다른 게임 회귀 다발.
3. **`patchSupportedWorkshopFileTypes` `:2900`** — enum 확장 reflection. 한 번만 실행. 호출 추가/제거 X.
4. **`startWorkshopDownload` 의 finally `:3149-3164`** — `removeDownloadJob` + `setWorkshopDownloadPending(false)` (NonCancellable) — UI stuck 방지 핵심. 신규 코드 동일 패턴 모방 필수.
5. **`SteamService.workshopPausedApps` / `setAppDownloadInfo` / `notifyDownloadStarted` 트리오** — 신규 다운로드도 반드시 같은 순서. 미호출 시 진행률 UI 미표시 + pause/resume 깨짐.
6. **`getItemsNeedingSync` 의 `trulyMissing` `:3232-3267`** — Steam API timestamp drift 흡수. 단순화하면 매번 재다운로드.

## 섹션 4 — 구독 후 다운로드 자동 트리거 여부 (검증)

### **결론: 자동 트리거되지 않음. 명시 호출 필수.**

근거:
- (a) `Subscribe` RPC 자체는 Steam 서버 구독 목록만 갱신. 다운로드 행위와 별개.
- (b) 다운로드는 항상 `startWorkshopDownload` 또는 `checkForWorkshopUpdates` → `downloadItems` 명시 체인.
- (c) 두 진입점 모두 `enabledIds` (DB CSV) 기반 필터. Subscribe 만으로 DB 미갱신 → 다음 launch 에도 다운로드 X.
- (d) `getSubscribedItems` 는 Steam 서버 응답 단순 fetch. caching 하지 않으므로 Subscribe 후 즉시 재호출 시 신규 ID 응답 포함 (서버 latency ~수초).

### 신규 fork 가 Subscribe 직후 해야 할 일

```kotlin
// 1. RPC 호출
val result = subscribeToWorkshopItem(appId, publishedFileId)
if (result != EResult.OK) { /* 토스트 */ return }

// 2. DB 의 enabledIds 에 신규 ID 추가
val current = WorkshopManager.parseEnabledIds(appDao.getEnabledWorkshopItemIds(gameId))
val updated = current + publishedFileId
appDao.updateWorkshopState(gameId, true, updated.joinToString(","))

// 3. 다운로드 트리거 (게임 설치돼 있고 인터넷 있을 때만)
if (SteamService.isAppInstalled(gameId) && NetworkMonitor.hasInternet.value) {
    WorkshopManager.startWorkshopDownload(gameId, updated, context)
}
```

이 패턴은 `SteamAppScreen.kt:1262-1283` 의 기존 Save 핸들러와 동일 → **재사용 권장**.

### gbe_fork 경로 자동 배치 검증

`startWorkshopDownload:3070` → `getWorkshopContentDir(winePrefix, appId)` → `:1135-1140`:
```kotlin
return File(winePrefix, "drive_c/Program Files (x86)/Steam/steamapps/workshop/content/$appId")
```
→ `downloadItems` 가 `File(workshopContentDir, item.publishedFileId.toString())` 에 저장 (`:725, :798`).
→ **신규 구독은 자동으로 정확한 경로에 배치됨**, `startWorkshopDownload` 만 호출하면 OK.

## 섹션 6 — 호출자 / Race condition 분석

### 기존 `getSubscribedItems` 호출자 3곳

1. `WorkshopManagerDialog.kt:106` — UI 다이얼로그 열 때 (LaunchedEffect, IO).
2. `WorkshopManager.kt:3056` (`startWorkshopDownload` 내부) — Save 직후 background.
3. `WorkshopManager.kt:3193` (`checkForWorkshopUpdates` 내부) — 게임 launch 직전 (`PluviaMain.kt:1904`).

### Race 우려

- `SteamService.getAppDownloadInfo(appId)?.cancel(...)` (`:3034`) — 이미 진행 중인 다운로드 강제 취소 후 새로 시작. Subscribe 직후 호출 시 기존 launch-time 다운로드 중단 가능성. **신규 코드에서 `awaitCompletion(timeoutMs=...)` (`PluviaMain.kt:1897`) 으로 기다리는 옵션 고려**.
- 동일 `SteamUnifiedMessages` 핸들러 thread-safe 명시 보장 X — 그러나 기존 코드 다중 호출 사례 있어 사실상 안전.

### 신규 진입점 (브라우저) 안전한 호출 위치

- 게임 디테일 화면 헤더 폴더 아이콘 클릭 → `WorkshopBrowserDialog` 열기 → 내부 검색 RPC.
- 다이얼로그 lifecycle (`DisposableEffect` 또는 ViewModel) 에서 RPC scope 관리 → 다이얼로그 닫힘 시 자동 cancel.
- Subscribe 클릭 → 위 3단계 (RPC → DB → `startWorkshopDownload`) 를 `CoroutineScope(Dispatchers.IO).launch`. `SteamAppScreen.kt:1265` 패턴 그대로.

## 부록 — fork 데이터 모델 추정

`WorkshopItem.kt:6-26` 의 `WorkshopItem` 은 구독 항목용 (`manifestId, fileUrl, fileSizeBytes`). 브라우저는 더 풍부한 메타데이터 필요. 분리 권장:

```kotlin
data class WorkshopItemDetail(
    val item: WorkshopItem,
    val description: String,
    val shortDescription: String,
    val tags: List<String>,
    val voteScore: Float,             // 0.0 ~ 1.0
    val voteUp: Long,
    val voteDown: Long,
    val creatorSteamId: Long,
    val creatorName: String,          // PublishedFile.GetDetails 응답에 미포함 → ISteamUser 추가 RPC 필요할 수 있음
    val timeCreated: Long,
    val viewCount: Long,
    val subscriberCount: Long,
)
```

## 200단어 요약

기존 `WorkshopManager.kt` 는 **이미 구독한 항목의 다운로드/설치 파이프라인** 이며 fork 가 추가할 **브라우저+Subscribe RPC** 와 직교. 핵심 4가지: (1) `Subscribe` RPC 만으로는 다운로드 자동 트리거되지 않음 — DB 의 `enabledWorkshopItemIds` 갱신 후 `startWorkshopDownload` 명시 호출 필수. (2) `:84-219` 의 `getSubscribedItems`/`fetchSubscribedFilesViaRPC` 가 PublishedFile RPC 호출의 정확한 5단계 템플릿 (handler 획득 → builder → 30s timeout await → EResult check → 변환) 제공 — `QueryFiles`/`GetDetails`/`Subscribe`/`Unsubscribe` 모두 동일 패턴 복제. (3) gbe_fork 경로는 `startWorkshopDownload` → `downloadItems` 가 자동 처리 — 신규 코드 경로 직접 다룰 필요 X. (4) 절대 손대지 말 영역: `downloadItems`, `configureModSymlinks` (960줄), `patchSupportedWorkshopFileTypes`, `startWorkshopDownload` finally cleanup, `workshopPausedApps`/`setAppDownloadInfo`/`notifyDownloadStarted` 트리오. 진입점은 기존 `AppOptionMenuType.ManageWorkshop` (구독자 토글 UI) 와 분리해 게임 디테일 헤더 폴더 아이콘으로 별도 추가 (`LibraryAppScreen.kt:866` 직전).
