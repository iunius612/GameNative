# PublishedFile RPC 호출 패턴 — DD2 fork 신규 코드 템플릿

**파일 핀**: D:/git/GameNative-DD2 @ feature/workshop-browser (off v0.9.0 sha 24508c2a)
**기준 패턴 출처**: `app/src/main/java/app/gamenative/workshop/WorkshopManager.kt:84-219` (`getSubscribedItems` + `fetchSubscribedFilesViaRPC`)

## 5단계 템플릿 (모든 RPC 공통)

```kotlin
// === 1. 핸들러/서비스 획득 (호출 시점마다 1회) ===
val unifiedMessages = steamClient.getHandler<SteamUnifiedMessages>()
if (unifiedMessages == null) {
    Timber.tag(TAG).e("SteamUnifiedMessages handler not available")
    return /* 적절한 fallback */
}
val publishedFile = unifiedMessages.createService<PublishedFile>()

// === 2. Request builder (protobuf) ===
val request = CPublishedFile_<RPC>_Request.newBuilder().apply {
    /* 필드 채움 */
}.build()

// === 3. RPC 호출 + timeout + 에러 처리 ===
val response = withTimeoutOrNull(<TIMEOUT_MS>) {
    publishedFile.<rpc>(request).toFuture().await()
}
if (response == null) { /* timeout */ return /* fallback */ }
if (response.result != EResult.OK) {
    Timber.tag(TAG).w("RPC <name> failed: ${response.result}")
    return /* fallback */
}

// === 4. 응답 변환 ===
val body = response.body.build()
/* body 의 protobuf 필드를 도메인 타입으로 매핑 */

// === 5. 예외 처리 (CancellationException 재throw 필수) ===
} catch (e: CancellationException) { throw e }
catch (e: Exception) { Timber.tag(TAG).e(e, "..."); return /* fallback */ }
```

## RPC 별 templated 호출

### QueryFiles (검색·브라우즈)

```kotlin
import `in`.dragonbra.javasteam.protobufs.steamclient.SteammessagesPublishedfileSteamclient.CPublishedFile_QueryFiles_Request

val request = CPublishedFile_QueryFiles_Request.newBuilder().apply {
    this.appid = appId
    this.queryType = 9  // k_PublishedFileQueryType_RankedByTextSearch (또는 0=RankedByVote, 1=RankedByPublicationDate, 3=RankedByTrend, 12=RankedByLastUpdatedDate)
    this.page = page                  // 1-based
    this.numperpage = 20
    this.searchText = userQuery       // 비어있으면 전체 브라우즈
    this.returnDetails = true
    this.returnPreviews = true        // 썸네일 URL
    this.returnVoteData = true        // voteUp/voteDown
    this.returnShortDescription = true
    // this.returnTags = true          // protobuf 정의에 있으면
    // this.requiredTagsList — 태그 필터 시
}.build()
val response = withTimeoutOrNull(30_000L) {
    publishedFile.queryFiles(request).toFuture().await()
}
```

**queryType 상수표** (`k_PublishedFileQueryType_*`):
| 값 | 의미 |
|---|---|
| 0 | RankedByVote (인기) |
| 1 | RankedByPublicationDate (신규) |
| 3 | RankedByTrend (트렌드) |
| 9 | RankedByTextSearch (검색어) |
| 12 | RankedByLastUpdatedDate (최근 업데이트) |
| 21 | RankedByVotesUp (좋아요 많은 순) |

DD2 브라우저 정렬 옵션은 위 셋(0/1/12) 노출 권장.

### GetDetails (상세 정보)

```kotlin
import `in`.dragonbra.javasteam.protobufs.steamclient.SteammessagesPublishedfileSteamclient.CPublishedFile_GetDetails_Request

val request = CPublishedFile_GetDetails_Request.newBuilder().apply {
    addAllPublishedfileids(listOf(publishedFileId))    // bulk 가능
    this.includetags = true
    this.includeadditionalpreviews = true               // 추가 스크린샷
    this.includechildren = true                          // 컬렉션 자식
    this.includevotes = true
    this.includeforsaledata = true
    this.includemetadata = true                          // 작성자 등
}.build()
val response = withTimeoutOrNull(30_000L) {
    publishedFile.getDetails(request).toFuture().await()
}
```

**작성자 이름 별도 RPC**: `creator` 필드는 SteamID64. 사용자 이름 필요시 `ISteamUser.GetPlayerSummaries` 또는 `Player.GetPlayerInfo` 별도 호출. 우선순위 낮음 — 초기 구현은 SteamID 만 표시 후 v0.2 이상에서 이름 fetch.

### Subscribe / Unsubscribe

```kotlin
import `in`.dragonbra.javasteam.protobufs.steamclient.SteammessagesPublishedfileSteamclient.CPublishedFile_Subscribe_Request
// (Unsubscribe 는 CPublishedFile_Unsubscribe_Request)

val request = CPublishedFile_Subscribe_Request.newBuilder().apply {
    this.publishedfileid = publishedFileId
    this.listType = 1                                    // k_PublishedFileListType_MySubscriptions
    this.appid = appId
    // notifyClient = true (protobuf 에 있으면 — 기존 클라 sync 권장)
}.build()
val response = withTimeoutOrNull(15_000L) {           // 검색보다 짧게
    publishedFile.subscribe(request).toFuture().await()
    // Unsubscribe: publishedFile.unsubscribe(request).toFuture().await()
}
return response?.result ?: EResult.Timeout
```

### CanSubscribe (선택적 사전 검증)

DRM·지역 제한 모드에 사전 차단:

```kotlin
val request = CPublishedFile_CanSubscribe_Request.newBuilder().apply {
    this.publishedfileid = publishedFileId
}.build()
val response = withTimeoutOrNull(10_000L) {
    publishedFile.canSubscribe(request).toFuture().await()
}
// response.body.build().canSubscribe : Boolean
```

### AreFilesInSubscriptionList (UI 배지)

브라우저 그리드에서 "이미 구독 중" 표시:

```kotlin
val request = CPublishedFile_AreFilesInSubscriptionList_Request.newBuilder().apply {
    this.appid = appId
    addAllPublishedfileids(idsOnPage)  // 현재 페이지 ID들
    this.listType = 1
}.build()
// response.body.build().inListList : List<Pair<Long, Boolean>>
```

## 복제 시 주의사항 (모든 RPC 공통)

1. **handler null 체크 필수** — login 직후 잠깐 null 가능 (`WorkshopManager.kt:93-97`).
2. **`CancellationException` 항상 재throw** (`:213-214`) — Compose `LaunchedEffect` 취소 시 leak 방지.
3. **타임아웃 권장값**: 검색 30s, 상세 30s, Subscribe/Unsubscribe 15s, CanSubscribe 10s.
4. **`steamClient` 인스턴스**: `SteamService.instance?.steamClient ?: return` 패턴 (`startWorkshopDownload:3029, WorkshopManagerDialog.kt:102`).
5. **Subscribe 응답 결과 사용자 토스트로** — `EResult` != OK 시 `SnackbarManager.show(...)` (`PluviaMain.kt:1950, 1987` 패턴).
6. **Subscribe 후 sync 트리거** — `workshop_manager_existing_vs_new.md` §4 의 3단계 (RPC → DB → `startWorkshopDownload`) 절차 준수.
7. **페이지네이션** — `QueryFiles` 응답의 `total` 필드로 끝 검출. `WorkshopManager.kt:152-219` 의 `MAX_PAGES = 50` 패턴 참조.

## 신규 파일 권장 구조 (Phase 16 진입 시)

```
app/src/main/java/app/gamenative/workshop/
  WorkshopBrowser.kt              -- 신규: queryWorkshopItems, getWorkshopItemDetails, subscribe, unsubscribe
  WorkshopBrowserItem.kt          -- 신규: WorkshopItemDetail 데이터 클래스
  WorkshopManager.kt              -- 기존: 손대지 말 것 (위 §3 참조)
  WorkshopItem.kt                 -- 기존: 그대로 재사용
```

```
app/src/main/java/app/gamenative/ui/screen/workshop/
  WorkshopBrowserScreen.kt         -- 신규: 검색바 + 그리드 + 페이지네이션
  WorkshopDetailScreen.kt          -- 신규: 상세 + Subscribe 버튼
  components/
    WorkshopItemCard.kt            -- 신규: 그리드 셀
    WorkshopFilterBar.kt           -- 신규: 정렬·태그 드롭다운
```

```
app/src/main/java/app/gamenative/ui/model/
  WorkshopBrowserViewModel.kt      -- 신규: paging state + RPC 호출 + Subscribe trigger
```

## 빈 칸 (구현 단계에서 확정)

- `notifyClient` 필드가 `CPublishedFile_Subscribe_Request` protobuf 에 있는지 (JavaSteam 1.8.0.1 기준) — 빌드 시 컴파일 에러로 검증 가능
- `returnTags` / `requiredTagsList` 필드 정확한 이름 (proto 정의 따라 다름)
- `body.build()` 호출 패턴이 모든 응답에 동일한지 (response.body 가 builder 인지 직접 메시지인지) — `getUserFiles` 케이스에서 확인됨, 다른 RPC도 동일 추정
