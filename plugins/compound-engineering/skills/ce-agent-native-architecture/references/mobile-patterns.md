<overview>
모바일은 에이전트 네이티브 앱을 위한 일급 시민(first-class platform)입니다. 모바일은 고유한 제약 조건과 기회를 동시에 가지고 있습니다. 이 가이드는 왜 모바일이 중요한지, iOS 저장소 아키텍처, 체크포인트/재개(checkpoint/resume) 패턴, 그리고 비용을 고려한 설계에 대해 다룹니다.
</overview>

<why_mobile>
## 왜 모바일인가?

모바일 기기는 에이전트 네이티브 앱에 다음과 같은 독특한 이점을 제공합니다:

### 파일 시스템
에이전트는 다른 곳에서 작동하는 것과 동일한 프리미티브를 사용하여 자연스럽게 파일을 다룰 수 있습니다. 파일 시스템은 유니버설 인터페이스입니다.

### 풍부한 컨텍스트 (Rich Context)
접근 가능한 '가둬진 정원(walled garden)'입니다. 건강 데이터, 위치, 사진, 캘린더 등 데스크톱이나 웹에는 존재하지 않는 컨텍스트가 있습니다. 이를 통해 깊이 있게 개인화된 에이전트 경험이 가능합니다.

### 로컬 앱
모든 사용자가 앱의 개별 사본을 소유합니다. 이는 아직 완전히 실현되지 않은 기회들을 열어줍니다: 스스로 수정하고, 스스로 분기하며, 사용자별로 진화하는 앱입니다. 현재 앱 스토어 정책이 일부를 제한하고 있지만, 기반은 이미 마련되어 있습니다.

### 기기 간 동기화
iCloud와 함께 파일 시스템을 사용하면 모든 기기가 동일한 파일 시스템을 공유합니다. 한 기기에서의 에이전트 작업이 별도의 서버 구축 없이도 모든 기기에 나타납니다.

### 도전 과제

**에이전트는 오래 실행되지만, 모바일 앱은 그렇지 않습니다.**

에이전트는 작업을 완료하는 데 30초, 5분, 또는 한 시간이 필요할 수 있습니다. 하지만 iOS는 비활성 상태가 된 지 몇 초 만에 앱을 백그라운드로 전환하고, 메모리 확보를 위해 앱을 완전히 종료할 수도 있습니다. 사용자는 작업 중간에 앱을 바꾸거나, 전화를 받거나, 폰을 잠글 수 있습니다.

따라서 모바일 에이전트 앱에는 다음이 필요합니다:
- **체크포인팅 (Checkpointing)** — 작업 손실을 방지하기 위해 상태 저장
- **재개 (Resuming)** — 중단된 지점부터 다시 시작
- **백그라운드 실행** — iOS가 부여한 제한된 시간을 현명하게 사용
- **온디바이스 vs 클라우드 결정** — 로컬에서 실행할 것과 서버가 필요한 것의 구분
</why_mobile>

<ios_storage>
## iOS 저장소 아키텍처

> **검증 필요:** 이 방식은 잘 작동하는 하나의 접근 방식이며, 더 나은 해결책이 존재할 수 있습니다.

에이전트 네이티브 iOS 앱의 경우, 공유 작업 공간으로 iCloud Drive의 Documents 폴더를 사용하십시오. 이를 통해 동기화 계층을 구축하거나 서버를 운영하지 않고도 **무료로 자동 기기 간 동기화**가 가능해집니다.

### 왜 iCloud Documents인가?

| 방식 | 비용 | 복잡도 | 오프라인 | 기기 간 동기화 |
|----------|------|------------|---------|--------------|
| 커스텀 백엔드 + 동기화 | $$$ | 높음 | 수동 구현 | 가능 |
| CloudKit 데이터베이스 | 무료 티어 제한 | 중간 | 수동 구현 | 가능 |
| **iCloud Documents** | 무료 (사용자 용량) | 낮음 | 자동 | 자동 |

iCloud Documents의 장점:
- 사용자의 기존 iCloud 저장 공간 사용 (무료 5GB, 대부분의 사용자는 더 많음)
- 모든 사용자 기기에서 자동 동기화
- 오프라인에서 작동하며, 온라인 시 동기화됨
- 투명성을 위해 '파일(Files.app)' 앱에서 파일 확인 가능
- 서버 비용 없음, 유지보수할 동기화 코드 없음

### 구현: 로컬 폴백을 포함한 iCloud 우선 방식

```swift
// iCloud Documents 컨테이너 가져오기
func iCloudDocumentsURL() -> URL? {
    FileManager.default.url(forUbiquityContainerIdentifier: nil)?
        .appendingPathComponent("Documents")
}

// 공유 작업 공간은 iCloud에 위치함
class SharedWorkspace {
    let rootURL: URL

    init() {
        // iCloud를 우선 사용하고, 불가능할 경우 로컬로 폴백
        if let iCloudURL = iCloudDocumentsURL() {
            self.rootURL = iCloudURL
        } else {
            // 로컬 Documents로 폴백 (iCloud 로그인 안 된 경우)
            self.rootURL = FileManager.default.urls(
                for: .documentDirectory,
                in: .userDomainMask
            ).first!
        }
    }

    // 모든 파일 작업은 이 루트를 통해 수행됨
    func researchPath(for bookId: String) -> URL {
        rootURL.appendingPathComponent("Research/\(bookId)")
    }

    func journalPath() -> URL {
        rootURL.appendingPathComponent("Journal")
    }
}
```

### iCloud 파일 상태 처리

iCloud 파일이 로컬로 다운로드되지 않았을 수 있습니다. 이를 처리하십시오:

```swift
func readFile(at url: URL) throws -> String {
    // iCloud가 .icloud 플레이스홀더 파일을 생성했을 수 있음
    if url.pathExtension == "icloud" {
        // 다운로드 트리거
        try FileManager.default.startDownloadingUbiquitousItem(at: url)
        throw FileNotYetAvailableError()
    }

    return try String(contentsOf: url, encoding: .utf8)
}

// 쓰기 작업 시에는 조정된 파일 접근(coordinated file access)을 사용하십시오
func writeFile(_ content: String, to url: URL) throws {
    let coordinator = NSFileCoordinator()
    var error: NSError?

    coordinator.coordinate(
        writingItemAt: url,
        options: .forReplacing,
        error: &error
    ) { newURL in
        try? content.write(to: newURL, atomically: true, encoding: .utf8)
    }

    if let error = error { throw error }
}
```

### iCloud가 가능하게 하는 것들

1. **사용자가 iPhone에서 실험 시작** → 에이전트가 설정 파일 생성
2. **사용자가 iPad에서 앱 열기** → 별도의 동기화 코드 없이 동일한 실험 확인 가능
3. **에이전트가 iPhone에서 관찰 기록** → iPad로 자동 동기화
4. **사용자가 iPad에서 저널 수정** → iPhone에서 즉시 반영됨

### 필요한 권한(Entitlements)

앱의 entitlements에 다음을 추가하십시오:

```xml
<key>com.apple.developer.icloud-container-identifiers</key>
<array>
    <string>iCloud.com.yourcompany.yourapp</string>
</array>
<key>com.apple.developer.icloud-services</key>
<array>
    <string>CloudDocuments</string>
</array>
<key>com.apple.developer.ubiquity-container-identifiers</key>
<array>
    <string>iCloud.com.yourcompany.yourapp</string>
</array>
```

### iCloud Documents를 사용하지 말아야 할 때

- **민감한 데이터** - Keychain이나 암호화된 로컬 저장소 사용
- **고빈도 쓰기** - iCloud 동기화에는 지연 시간이 있으므로 로컬 저장 후 주기적 동기화 사용
- **대용량 미디어 파일** - CloudKit Assets 또는 온디맨드 리소스 고려
- **사용자 간 공유** - iCloud Documents는 단일 사용자용임. 공유에는 CloudKit 사용
</ios_storage>

<background_execution>
## 백그라운드 실행 및 재개

> **검증 필요:** 이 패턴들은 작동하지만 더 나은 해결책이 존재할 수 있습니다.

모바일 앱은 언제든 중단되거나 종료될 수 있습니다. 에이전트는 이를 우아하게 처리해야 합니다.

### 도전 과제

```
사용자가 조사 에이전트 시작
     ↓
에이전트가 웹 검색 시작
     ↓
사용자가 다른 앱으로 전환
     ↓
iOS가 앱을 중단(suspend)시킴
     ↓
에이전트가 실행 도중인데... 무슨 일이 벌어질까요?
```

### 체크포인트/재개 패턴

백그라운드로 전환되기 전에 에이전트 상태를 저장하고, 포그라운드로 돌아왔을 때 복원하십시오:

```swift
class AgentOrchestrator: ObservableObject {
    @Published var activeSessions: [AgentSession] = []

    // 앱이 백그라운드로 가기 직전 호출됨
    func handleAppWillBackground() {
        for session in activeSessions {
            saveCheckpoint(session)
            session.transition(to: .backgrounded)
        }
    }

    // 앱이 포그라운드로 돌아왔을 때 호출됨
    func handleAppDidForeground() {
        for session in activeSessions where session.state == .backgrounded {
            if let checkpoint = loadCheckpoint(session.id) {
                resumeFromCheckpoint(session, checkpoint)
            }
        }
    }

    private func saveCheckpoint(_ session: AgentSession) {
        let checkpoint = AgentCheckpoint(
            sessionId: session.id,
            conversationHistory: session.messages,
            pendingToolCalls: session.pendingToolCalls,
            partialResults: session.partialResults,
            timestamp: Date()
        )
        storage.save(checkpoint, for: session.id)
    }

    private func resumeFromCheckpoint(_ session: AgentSession, _ checkpoint: AgentCheckpoint) {
        session.messages = checkpoint.conversationHistory
        session.pendingToolCalls = checkpoint.pendingToolCalls

        // 대기 중인 도구 호출이 있었다면 실행 재개
        if !checkpoint.pendingToolCalls.isEmpty {
            session.transition(to: .running)
            Task { await executeNextTool(session) }
        }
    }
}
```

### 에이전트 라이프사이클을 위한 상태 머신

```swift
enum AgentState {
    case idle           // 실행 중 아님
    case running        // 활발히 실행 중
    case waitingForUser // 일시 정지, 사용자 입력 대기 중
    case backgrounded   // 앱이 백그라운드로 이동, 상태 저장됨
    case completed      // 성공적으로 완료됨
    case failed(Error)  // 오류와 함께 종료됨
}

class AgentSession: ObservableObject {
    @Published var state: AgentState = .idle

    func transition(to newState: AgentState) {
        let validTransitions: [AgentState: Set<AgentState>] = [
            .idle: [.running],
            .running: [.waitingForUser, .backgrounded, .completed, .failed],
            .waitingForUser: [.running, .backgrounded],
            .backgrounded: [.running, .completed],
        ]

        guard validTransitions[state]?.contains(newState) == true else {
            logger.warning("잘못된 전이: \(state) → \(newState)")
            return
        }

        state = newState
    }
}
```

### 백그라운드 작업 연장 (iOS)

중요한 작업 도중 백그라운드로 전환될 때 추가 시간을 요청하십시오:

```swift
class AgentOrchestrator {
    private var backgroundTask: UIBackgroundTaskIdentifier = .invalid

    func handleAppWillBackground() {
        // 상태 저장을 위해 추가 시간 요청
        backgroundTask = UIApplication.shared.beginBackgroundTask { [weak self] in
            self?.endBackgroundTask()
        }

        // 모든 체크포인트 저장
        Task {
            for session in activeSessions {
                await saveCheckpoint(session)
            }
            endBackgroundTask()
        }
    }

    private func endBackgroundTask() {
        if backgroundTask != .invalid {
            UIApplication.shared.endBackgroundTask(backgroundTask)
            backgroundTask = .invalid
        }
    }
}
```

### 사용자 커뮤니케이션

사용자에게 현재 상황을 알려주십시오:

```swift
struct AgentStatusView: View {
    @ObservedObject var session: AgentSession

    var body: some View {
        switch session.state {
        case .backgrounded:
            Label("일시 정지됨 (앱이 백그라운드에 있음)", systemImage: "pause.circle")
                .foregroundColor(.orange)
        case .running:
            Label("작업 중...", systemImage: "ellipsis.circle")
                .foregroundColor(.blue)
        case .waitingForUser:
            Label("사용자 입력을 기다리는 중", systemImage: "person.circle")
                .foregroundColor(.green)
        // ...
        }
    }
}
```
</background_execution>

<permissions>
## 권한 처리

모바일 에이전트는 시스템 자원에 접근해야 할 수 있습니다. 권한 요청을 우아하게 처리하십시오.

### 일반적인 권한들

| 자원 | iOS 권한 | 사용 사례 |
|----------|---------------|----------|
| 사진 라이브러리 | PHPhotoLibrary | 사진을 통한 프로필 생성 |
| 파일 | Document picker | 사용자 문서 읽기 |
| 카메라 | AVCaptureDevice | 책 표지 스캔 |
| 위치 | CLLocationManager | 위치 기반 추천 |
| 네트워크 | (자동) | 웹 검색, API 호출 |

### 권한 인식 도구

실행 전 권한을 확인하십시오:

```swift
struct PhotoTools {
    static func readPhotos() -> AgentTool {
        tool(
            name: "read_photos",
            description: "사용자의 사진 라이브러리에서 사진 읽기",
            parameters: [
                "limit": .number("최대 사진 수"),
                "dateRange": .string("날짜 범위 필터").optional()
            ],
            execute: { params, context in
                // 먼저 권한 확인
                let status = await PHPhotoLibrary.requestAuthorization(for: .readWrite)

                switch status {
                case .authorized, .limited:
                    // 사진 읽기 진행
                    let photos = await fetchPhotos(params)
                    return ToolResult(text: "\(photos.count)개의 사진을 찾았습니다", images: photos)

                case .denied, .restricted:
                    return ToolResult(
                        text: "사진 접근 권한이 필요합니다. 설정 → 개인정보 보호 → 사진에서 권한을 허용해 주세요.",
                        isError: true
                    )

                case .notDetermined:
                    return ToolResult(
                        text: "사진 권한이 필요합니다. 다시 시도해 주세요.",
                        isError: true
                    )

                @unknown default:
                    return ToolResult(text: "알 수 없는 권한 상태", isError: true)
                }
            }
        )
    }
}
```

### 단계적 기능 저하 (Graceful Degradation)

권한이 거부되었을 때 대안을 제시하십시오:

```swift
func readPhotos() async -> ToolResult {
    let status = PHPhotoLibrary.authorizationStatus(for: .readWrite)

    switch status {
    case .denied, .restricted:
        // 대안 제안
        return ToolResult(
            text: """
            사진에 접근할 수 없습니다. 다음 방법 중 하나를 선택해 주세요:
            1. 설정 → 개인정보 보호 → 사진에서 접근 권한 허용
            2. 특정 사진을 채팅창에 직접 공유

            대신 다른 작업을 도와드릴까요?
            """,
            isError: false  // 치명적 오류가 아닌 기능적 제한임
        )
    // ...
    }
}
```

### 권한 요청 시점

필요할 때까지 권한을 요청하지 마십시오:

```swift
// 나쁨: 시작 시 모든 권한 요청
func applicationDidFinishLaunching() {
    requestPhotoAccess()
    requestCameraAccess()
    requestLocationAccess()
    // 사용자는 쏟아지는 권한 요청 팝업에 당황하게 됨
}

// 좋음: 기능을 사용할 때 요청
tool("analyze_book_cover", async ({ image }) => {
    // 사용자가 표지를 스캔하려고 할 때만 카메라 권한 요청
    let status = await AVCaptureDevice.requestAccess(for: .video)
    if status {
        return await scanCover(image)
    } else {
        return ToolResult(text: "책 스캔을 위해 카메라 권한이 필요합니다")
    }
})
```
</permissions>

<cost_awareness>
## 비용 고려 설계

모바일 사용자는 셀룰러 데이터를 사용 중이거나 API 비용을 걱정할 수 있습니다. 에이전트가 효율적으로 작동하도록 설계하십시오.

### 모델 티어 선택

결과를 달성할 수 있는 가장 저렴한 모델을 사용하십시오:

```swift
enum ModelTier {
    case fast      // claude-3-haiku: 약 $0.25/1M 토큰
    case balanced  // claude-3-sonnet: 약 $3/1M 토큰
    case powerful  // claude-3-opus: 약 $15/1M 토큰

    var modelId: String {
        switch self {
        case .fast: return "claude-3-haiku-20240307"
        case .balanced: return "claude-3-sonnet-20240229"
        case .powerful: return "claude-3-opus-20240229"
        }
    }
}

// 작업 복잡도에 모델 매칭
let agentConfigs: [AgentType: ModelTier] = [
    .quickLookup: .fast,        // "내 라이브러리에 뭐 있어?"
    .chatAssistant: .balanced,  // 일반적인 대화
    .researchAgent: .balanced,  // 웹 검색 + 합성
    .profileGenerator: .powerful, // 복잡한 사진 분석
    .introductionWriter: .balanced,
]
```

### 토큰 예산 (Token Budgets)

에이전트 세션당 토큰 사용량을 제한하십시오:

```swift
struct AgentConfig {
    let modelTier: ModelTier
    let maxInputTokens: Int
    let maxOutputTokens: Int
    let maxTurns: Int

    static let research = AgentConfig(
        modelTier: .balanced,
        maxInputTokens: 50_000,
        maxOutputTokens: 4_000,
        maxTurns: 20
    )

    static let quickChat = AgentConfig(
        modelTier: .fast,
        maxInputTokens: 10_000,
        maxOutputTokens: 1_000,
        maxTurns: 5
    )
}

class AgentSession {
    var totalTokensUsed: Int = 0

    func checkBudget() -> Bool {
        if totalTokensUsed > config.maxInputTokens {
            transition(to: .failed(AgentError.budgetExceeded))
            return false
        }
        return true
    }
}
```

### 네트워크 상태 인식 실행

무거운 작업은 WiFi 환경으로 미루십시오:

```swift
class NetworkMonitor: ObservableObject {
    @Published var isOnWiFi: Bool = false
    @Published var isExpensive: Bool = false  // 셀룰러 또는 핫스팟

    private let monitor = NWPathMonitor()

    func startMonitoring() {
        monitor.pathUpdateHandler = { [weak self] path in
            DispatchQueue.main.async {
                self?.isOnWiFi = path.usesInterfaceType(.wifi)
                self?.isExpensive = path.isExpensive
            }
        }
        monitor.start(queue: .global())
    }
}

class AgentOrchestrator {
    @ObservedObject var network = NetworkMonitor()

    func startResearchAgent(for book: Book) async {
        if network.isExpensive {
            // 사용자에게 경고하거나 작업을 미룸
            let proceed = await showAlert(
                "조사에 데이터 사용됨",
                message: "이 작업은 약 1-2 MB의 셀룰러 데이터를 사용합니다. 계속하시겠습니까?"
            )
            if !proceed { return }
        }

        // 조사 진행
        await runAgent(ResearchAgent.create(book: book))
    }
}
```

### API 호출 배치 처리 (Batching)

여러 개의 작은 요청을 하나로 합치십시오:

```swift
// 나쁨: 많은 수의 작은 API 호출
for book in books {
    await agent.chat("\(book.title) 요약해줘")
}

// 좋음: 하나의 요청으로 배치 처리
let bookList = books.map { $0.title }.joined(separator: ", ")
await agent.chat("다음 책들을 각각 짧게 요약해줘: \(bookList)")
```

### 캐싱 (Caching)

비싼 작업은 캐싱하십시오:

```swift
class ResearchCache {
    private var cache: [String: CachedResearch] = [:]

    func getCachedResearch(for bookId: String) -> CachedResearch? {
        guard let cached = cache[bookId] else { return nil }

        // 24시간 후 만료
        if Date().timeIntervalSince(cached.timestamp) > 86400 {
            cache.removeValue(forKey: bookId)
            return nil
        }

        return cached
    }

    func cacheResearch(_ research: Research, for bookId: String) {
        cache[bookId] = CachedResearch(
            research: research,
            timestamp: Date()
        )
    }
}

// 조사 도구 내에서
tool("web_search", async ({ query, bookId }) => {
    // 먼저 캐시 확인
    if let cached = cache.getCachedResearch(for: bookId) {
        return ToolResult(text: cached.research.summary, cached: true)
    }

    // 캐시 없으면 검색 수행
    let results = await webSearch(query)
    cache.cacheResearch(results, for: bookId)
    return ToolResult(text: results.summary)
})
```

### 비용 가시성

사용자에게 얼마를 쓰고 있는지 보여주십시오:

```swift
struct AgentCostView: View {
    @ObservedObject var session: AgentSession

    var body: some View {
        VStack(alignment: .leading) {
            Text("세션 통계")
                .font(.headline)

            HStack {
                Label("\(session.turnCount) 턴", systemImage: "arrow.2.squarepath")
                Spacer()
                Label(formatTokens(session.totalTokensUsed), systemImage: "text.word.spacing")
            }

            if let estimatedCost = session.estimatedCost {
                Text("예상 비용: \(estimatedCost, format: .currency(code: "USD"))")
                    .font(.caption)
                    .foregroundColor(.secondary)
            }
        }
    }
}
```
</cost_awareness>

<offline_handling>
## 오프라인에서의 기능 저하 처리

오프라인 시나리오를 우아하게 처리하십시오:

```swift
class ConnectivityAwareAgent {
    @ObservedObject var network = NetworkMonitor()

    func executeToolCall(_ toolCall: ToolCall) async -> ToolResult {
        // 네트워크가 필요한 도구인지 확인
        let requiresNetwork = ["web_search", "web_fetch", "call_api"]
            .contains(toolCall.name)

        if requiresNetwork && !network.isConnected {
            return ToolResult(
                text: """
                지금은 인터넷에 연결할 수 없습니다. 오프라인에서 가능한 작업은 다음과 같습니다:
                - 라이브러리 및 기존 조사 결과 읽기
                - 캐시된 데이터로 질문에 답변하기
                - 나중에 사용할 노트 및 초안 작성

                오프라인에서 가능한 작업을 시도해 볼까요?
                """,
                isError: false
            )
        }

        return await executeOnline(toolCall)
    }
}
```

### 오프라인 우선 도구 (Offline-First Tools)

일부 도구는 완전히 오프라인에서 작동해야 합니다:

```swift
let offlineTools: Set<String> = [
    "read_file",
    "write_file",
    "list_files",
    "read_library",  // 로컬 데이터베이스
    "search_local",  // 로컬 검색
]

let onlineTools: Set<String> = [
    "web_search",
    "web_fetch",
    "publish_to_cloud",
]

let hybridTools: Set<String> = [
    "publish_to_feed",  // 오프라인에서 작동하고 나중에 동기화
]
```

### 액션 큐 (Queued Actions)

연결이 필요한 액션은 큐에 담아두십시오:

```swift
class OfflineQueue: ObservableObject {
    @Published var pendingActions: [QueuedAction] = []

    func queue(_ action: QueuedAction) {
        pendingActions.append(action)
        persist()
    }

    func processWhenOnline() {
        network.$isConnected
            .filter { $0 }
            .sink { [weak self] _ in
                self?.processPendingActions()
            }
    }

    private func processPendingActions() {
        for action in pendingActions {
            Task {
                try await execute(action)
                remove(action)
            }
        }
    }
}
```
</offline_handling>

<battery_awareness>
## 배터리 인식 실행

기기 배터리 상태를 존중하십시오:

```swift
class BatteryMonitor: ObservableObject {
    @Published var batteryLevel: Float = 1.0
    @Published var isCharging: Bool = false
    @Published var isLowPowerMode: Bool = false

    var shouldDeferHeavyWork: Bool {
        return batteryLevel < 0.2 && !isCharging
    }

    func startMonitoring() {
        UIDevice.current.isBatteryMonitoringEnabled = true

        NotificationCenter.default.addObserver(
            forName: UIDevice.batteryLevelDidChangeNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.batteryLevel = UIDevice.current.batteryLevel
        }

        NotificationCenter.default.addObserver(
            forName: NSNotification.Name.NSProcessInfoPowerStateDidChange,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.isLowPowerMode = ProcessInfo.processInfo.isLowPowerModeEnabled
        }
    }
}

class AgentOrchestrator {
    @ObservedObject var battery = BatteryMonitor()

    func startAgent(_ config: AgentConfig) async {
        if battery.shouldDeferHeavyWork && config.isHeavy {
            let proceed = await showAlert(
                "배터리 부족",
                message: "이 작업은 배터리를 많이 소모합니다. 계속하시겠습니까, 아니면 충전 중으로 미루시겠습니까?"
            )
            if !proceed { return }
        }

        // 배터리 상태에 따라 모델 티어 조정
        let adjustedConfig = battery.isLowPowerMode
            ? config.withModelTier(.fast)
            : config

        await runAgent(adjustedConfig)
    }
}
```
</battery_awareness>

<on_device_vs_cloud>
## 온디바이스 vs 클라우드

모바일 에이전트 네이티브 앱에서 각 컴포넌트가 어디서 실행되는지 이해하기:

| 컴포넌트 | 온디바이스 (On-Device) | 클라우드 (Cloud) |
|-----------|-----------|-------|
| 오케스트레이션 | ✅ | |
| 도구 실행 | ✅ (파일 작업, 사진 접근, HealthKit 등) | |
| LLM 호출 | | ✅ (Anthropic API 등) |
| 체크포인트 | ✅ (로컬 파일) | iCloud를 통해 선택 가능 |
| 오래 실행되는 에이전트 | iOS에 의해 제한됨 | 서버를 통해 가능 |

### 함의 (Implications)

**추론을 위해 네트워크 필수:**
- LLM 호출을 위해 앱은 네트워크 연결이 필요합니다.
- 네트워크가 없을 때 기능이 우아하게 저하되도록 도구를 설계하십시오.
- 일반적인 질의에 대해서는 오프라인 캐싱을 고려하십시오.

**데이터는 로컬에 유지:**
- 파일 작업은 기기 내에서 발생합니다.
- 명시적으로 동기화하지 않는 한 민감한 데이터는 기기를 떠나지 않습니다.
- 프라이버시는 기본적으로 보존됩니다.

**오래 실행되는 에이전트:**
수 시간 동안 실행되어야 하는 에이전트의 경우, 기기는 뷰어 및 입력 수단으로 사용하고 서버 사이드 오케스트레이터를 두는 것을 고려하십시오.
</on_device_vs_cloud>

<checklist>
## 모바일 에이전트 네이티브 체크리스트

**iOS 저장소:**
- [ ] iCloud Documents를 기본 저장소로 사용 (또는 의식적인 대안 선택)
- [ ] iCloud 사용 불가 시 로컬 Documents로 폴백 처리
- [ ] `.icloud` 플레이스홀더 파일 처리 (다운로드 트리거)
- [ ] 충돌 없는 쓰기를 위해 NSFileCoordinator 사용

**백그라운드 실행:**
- [ ] 모든 에이전트 세션에 체크포인트/재개 구현
- [ ] 에이전트 라이프사이클 상태 머신 구현 (idle, running, backgrounded 등)
- [ ] 중요 저장 작업을 위한 백그라운드 작업 연장 구현 (30초 윈도우)
- [ ] 백그라운드 상태인 에이전트에 대한 사용자 가시성 확보

**권장:**
- [ ] 권한은 시작 시가 아닌 필요할 때 요청
- [ ] 권한 거부 시 단계적 기능 저하 처리
- [ ] 설정 앱 딥링크를 포함한 명확한 오류 메시지 제공
- [ ] 권한 미보유 시 대안 경로 제공

**비용 인식:**
- [ ] 작업 복잡도에 맞는 모델 티어 매칭
- [ ] 세션당 토큰 예산 설정
- [ ] 네트워크 상태 인식 (무거운 작업은 WiFi로 미룸)
- [ ] 비싼 작업을 위한 캐싱 구현
- [ ] 사용자에게 비용 가시성 제공

**오프라인 처리:**
- [ ] 오프라인 가능 도구 식별
- [ ] 온라인 전용 기능에 대한 단계적 기능 저하 처리
- [ ] 온라인 복귀 시 동기화를 위한 액션 큐 구현
- [ ] 오프라인 상태에 대한 명확한 사용자 안내

**배터리 인식:**
- [ ] 무거운 작업을 위한 배터리 모니터링
- [ ] 저전력 모드 감지
- [ ] 배터리 상태에 따른 작업 연기 또는 티어 다운그레이드
</checklist>
