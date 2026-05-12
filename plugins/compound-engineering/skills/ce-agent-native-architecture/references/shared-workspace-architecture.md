<overview>
에이전트와 사용자는 별도의 샌드박스가 아닌 동일한 데이터 공간에서 작업해야 합니다. 에이전트가 파일을 쓰면 사용자가 볼 수 있어야 하고, 사용자가 무언가를 편집하면 에이전트가 그 변경 사항을 읽을 수 있어야 합니다. 이는 투명성을 높이고 협업을 가능하게 하며, 별도의 동기화 계층이 필요 없게 만듭니다.

**핵심 원칙:** 에이전트는 격리된 정원이 아니라 사용자 데이터와 동일한 파일 시스템에서 작동합니다.
</overview>

<why_shared_workspace>
## 왜 공유 작업 공간인가?

### 샌드박스 안티 패턴 (The Sandbox Anti-Pattern)

많은 에이전트 구현체들이 에이전트를 격리시킵니다:

```
┌─────────────────┐     ┌─────────────────┐
│   사용자 공간    │     │   에이전트 공간  │
├─────────────────┤     ├─────────────────┤
│ Documents/      │     │ agent_output/   │
│ user_files/     │  ←→ │ temp_files/     │
│ settings.json   │sync │ cache/          │
└─────────────────┘     └─────────────────┘
```

문제점:
- 공간 사이에 데이터를 이동시키기 위한 동기화 계층이 필요함
- 사용자가 에이전트의 작업을 쉽게 검사할 수 없음
- 에이전트가 사용자의 기여를 바탕으로 작업을 이어갈 수 없음
- 상태 데이터의 중복 발생
- 두 공간 사이의 일관성을 유지하는 데 따른 복잡성

### 공유 작업 공간 패턴 (The Shared Workspace Pattern)

```
┌─────────────────────────────────────────┐
│              공유 작업 공간               │
├─────────────────────────────────────────┤
│ Documents/                              │
│ ├── Research/                           │
│ │   └── {bookId}/        ← 에이전트가 씀 │
│ │       ├── full_text.txt               │
│ │       ├── introduction.md  ← 사용자가 편집 가능 │
│ │       └── sources/                    │
│ ├── Chats/               ← 양쪽 모두 읽기/쓰기 │
│ └── profile.md           ← 에이전트 생성, 사용자 개선 │
└─────────────────────────────────────────┘
         ↑                    ↑
       사용자                에이전트
       (UI)                (Tools)
```

장점:
- 사용자가 에이전트의 작업을 검사하고, 편집하고, 확장할 수 있음
- 에이전트가 사용자의 기여를 바탕으로 작업을 이어갈 수 있음
- 별도의 동기화 계층이 필요 없음
- 완벽한 투명성 제공
- 단일 진실 공급원 (Single source of truth)
</why_shared_workspace>

<directory_structure>
## 공유 작업 공간 설계

### 도메인별 구조화

누가 만들었느냐가 아니라, 데이터가 무엇을 나타내느냐에 따라 조직화하십시오:

```
Documents/
├── Research/
│   └── {bookId}/
│       ├── full_text.txt        # 에이전트가 다운로드함
│       ├── introduction.md      # 에이전트가 생성, 사용자가 편집 가능
│       ├── notes.md             # 사용자가 추가, 에이전트가 읽을 수 있음
│       └── sources/
│           └── {source}.md      # 에이전트가 수집함
├── Chats/
│   └── {conversationId}.json    # 양쪽 모두 읽기/쓰기
├── Exports/
│   └── {date}/                  # 에이전트가 사용자를 위해 생성함
└── profile.md                   # 에이전트가 사진으로부터 생성함
```

### 행위자별로 구조화하지 마십시오 (안티 패턴)

```
# 나쁨 - 누가 만들었느냐에 따라 분리함
Documents/
├── user_created/
│   └── notes.md
├── agent_created/
│   └── research.md
└── system/
    └── config.json
```

이는 인위적인 경계를 만들고 협업을 어렵게 만듭니다.

### 메타데이터를 위한 컨벤션 사용

누가 무언가를 만들거나 수정했는지 추적해야 하는 경우:

```markdown
<!-- introduction.md -->
---
created_by: agent
created_at: 2024-01-15
last_modified_by: user
last_modified_at: 2024-01-16
---

# 모비딕 서문

이 개인화된 서문은 당신의 독서 도우미에 의해 생성되었으며,
2024년 1월 16일에 당신에 의해 개선되었습니다.
```
</directory_structure>

<file_tools>
## 공유 작업 공간을 위한 파일 도구

앱이 사용하는 것과 동일한 파일 프리미티브를 에이전트에게 제공하십시오:

```swift
// iOS/Swift 구현 예시
struct FileTools {
    static func readFile() -> AgentTool {
        tool(
            name: "read_file",
            description: "사용자의 도큐먼트에서 파일 읽기",
            parameters: ["path": .string("Documents/ 기준 상대 경로")],
            execute: { params in
                let path = params["path"] as! String
                let documentsURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
                let fileURL = documentsURL.appendingPathComponent(path)
                let content = try String(contentsOf: fileURL)
                return ToolResult(text: content)
            }
        )
    }

    static func writeFile() -> AgentTool {
        tool(
            name: "write_file",
            description: "사용자의 도큐먼트에 파일 쓰기",
            parameters: [
                "path": .string("Documents/ 기준 상대 경로"),
                "content": .string("파일 내용")
            ],
            execute: { params in
                let path = params["path"] as! String
                let content = params["content"] as! String
                let documentsURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
                let fileURL = documentsURL.appendingPathComponent(path)

                // 필요한 경우 상위 디렉토리 생성
                try FileManager.default.createDirectory(
                    at: fileURL.deletingLastPathComponent(),
                    withIntermediateDirectories: true
                )

                try content.write(to: fileURL, atomically: true, encoding: .utf8)
                return ToolResult(text: "\(path) 쓰기 완료")
            }
        )
    }

    static func listFiles() -> AgentTool {
        tool(
            name: "list_files",
            description: "디렉토리 내 파일 나열",
            parameters: ["path": .string("Documents/ 기준 상대 경로")],
            execute: { params in
                let path = params["path"] as! String
                let documentsURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
                let dirURL = documentsURL.appendingPathComponent(path)
                let contents = try FileManager.default.contentsOfDirectory(atPath: dirURL.path)
                return ToolResult(text: contents.joined(separator: "\n"))
            }
        )
    }

    static func searchText() -> AgentTool {
        tool(
            name: "search_text",
            description: "파일 전체에서 텍스트 검색",
            parameters: [
                "query": .string("검색할 텍스트"),
                "path": .string("검색할 디렉토리 경로").optional()
            ],
            execute: { params in
                // 도큐먼트 전체에서 텍스트 검색 구현
                // 일치하는 파일과 스니펫 반환
            }
        )
    }
}
```

### TypeScript/Node.js 구현 예시

```typescript
const fileTools = [
  tool(
    "read_file",
    "작업 공간에서 파일 읽기",
    { path: z.string().describe("파일 경로") },
    async ({ path }) => {
      const content = await fs.readFile(path, 'utf-8');
      return { text: content };
    }
  ),

  tool(
    "write_file",
    "작업 공간에 파일 쓰기",
    {
      path: z.string().describe("파일 경로"),
      content: z.string().describe("파일 내용")
    },
    async ({ path, content }) => {
      await fs.mkdir(dirname(path), { recursive: true });
      await fs.writeFile(path, content, 'utf-8');
      return { text: `${path} 쓰기 완료` };
    }
  ),

  tool(
    "list_files",
    "디렉토리 내 파일 나열",
    { path: z.string().describe("디렉토리 경로") },
    async ({ path }) => {
      const files = await fs.readdir(path);
      return { text: files.join('\n') };
    }
  ),

  tool(
    "append_file",
    "파일에 내용 추가",
    {
      path: z.string().describe("파일 경로"),
      content: z.string().describe("추가할 내용")
    },
    async ({ path, content }) => {
      await fs.appendFile(path, content, 'utf-8');
      return { text: `${path}에 내용 추가 완료` };
    }
  ),
];
```
</file_tools>

<ui_integration>
## 공유 작업 공간과 UI 통합

UI는 에이전트가 파일을 쓰는 동일한 위치를 감시(observe)해야 합니다:

### 패턴 1: 파일 기반 반응성 (iOS)

```swift
class ResearchViewModel: ObservableObject {
    @Published var researchFiles: [ResearchFile] = []

    private var watcher: DirectoryWatcher?

    func startWatching(bookId: String) {
        let researchPath = documentsURL
            .appendingPathComponent("Research")
            .appendingPathComponent(bookId)

        watcher = DirectoryWatcher(url: researchPath) { [weak self] in
            // 에이전트가 새 파일을 쓸 때 리로드
            self?.loadResearchFiles(from: researchPath)
        }

        loadResearchFiles(from: researchPath)
    }
}

// 파일이 변경되면 SwiftUI가 자동으로 업데이트됨
struct ResearchView: View {
    @StateObject var viewModel = ResearchViewModel()

    var body: some View {
        List(viewModel.researchFiles) { file in
            ResearchFileRow(file: file)
        }
    }
}
```

### 패턴 2: 공유 데이터 스토어

파일 감시(file-watching)가 실용적이지 않은 경우, 공유 데이터 스토어를 사용하십시오:

```swift
// UI와 에이전트 도구가 공통으로 사용하는 공유 서비스
class BookLibraryService: ObservableObject {
    static let shared = BookLibraryService()

    @Published var books: [Book] = []
    @Published var analysisRecords: [AnalysisRecord] = []

    func addAnalysisRecord(_ record: AnalysisRecord) {
        analysisRecords.append(record)
        // 공유 저장소에 저장
        saveToStorage()
    }
}

// 에이전트 도구는 동일한 서비스를 통해 기록함
tool("publish_to_feed", async ({ bookId, content, headline }) => {
    let record = AnalysisRecord(bookId: bookId, content: content, headline: headline)
    BookLibraryService.shared.addAnalysisRecord(record)
    return { text: "피드에 게시 완료" }
})

// UI는 동일한 서비스를 감시함
struct FeedView: View {
    @StateObject var library = BookLibraryService.shared

    var body: some View {
        List(library.analysisRecords) { record in
            FeedItemRow(record: record)
        }
    }
}
```

### 패턴 3: 하이브리드 (파일 + 인덱스)

콘텐츠에는 파일을 사용하고, 인덱싱에는 데이터베이스를 사용하십시오:

```
Documents/
├── Research/
│   └── book_123/
│       └── introduction.md   # 실제 콘텐츠 (파일)

데이터베이스:
├── research_index
│   └── { bookId: "book_123", path: "Research/book_123/introduction.md", ... }
```

```swift
// 에이전트가 파일 작성
await writeFile("Research/\(bookId)/introduction.md", content)

// 그리고 인덱스 업데이트
await database.insert("research_index", {
    bookId: bookId,
    path: "Research/\(bookId)/introduction.md",
    title: extractTitle(content),
    createdAt: Date()
})

// UI는 인덱스를 쿼리한 후 파일을 읽음
let items = database.query("research_index", where: bookId == "book_123")
for item in items {
    let content = readFile(item.path)
    // 화면에 표시...
}
```
</ui_integration>

<collaboration_patterns>
## 에이전트-사용자 협업 패턴

### 패턴: 에이전트가 초안을 잡고, 사용자가 개선함

```
1. 에이전트가 introduction.md 생성
2. 사용자가 파일 앱 또는 인앱 에디터에서 파일을 엶
3. 사용자가 내용을 수정하여 개선함
4. 에이전트가 read_file을 통해 변경 사항을 확인
5. 향후 에이전트 작업은 사용자의 개선 사항을 바탕으로 이어짐
```

에이전트의 시스템 프롬프트는 이를 인지해야 합니다:

```markdown
## 사용자 콘텐츠 작업 가이드

당신이 생성한 콘텐츠(서문, 조사 노트 등)를 사용자가 나중에 편집할 수 있습니다.
파일을 수정하기 전에 항상 기존 파일을 읽으십시오. 사용자가 적용한 개선 사항을
보존해야 하기 때문입니다.

파일이 이미 존재하고 사용자에 의해 수정된 경우(메타데이터 확인 또는 마지막 버전과 비교),
덮어쓰기 전에 사용자에게 물어보십시오.
```

### 패턴: 사용자가 씨앗을 뿌리고, 에이전트가 확장함

```
1. 사용자가 초기 생각을 담은 notes.md 생성
2. 사용자가 "이 내용에 대해 더 조사해줘"라고 요청
3. 에이전트가 맥락을 이해하기 위해 notes.md를 읽음
4. 에이전트가 notes.md에 내용을 추가하거나 관련 파일들을 생성함
5. 사용자가 에이전트의 추가 내용을 바탕으로 작업을 계속함
```

### 패턴: 추가 전용(Append-Only) 협업

채팅 로그나 활동 스트림의 경우:

```markdown
<!-- activity.md - 둘 다 내용을 추가만 하며, 덮어쓰지 않음 -->

## 2024-01-15

**사용자:** "모비딕" 읽기 시작함

**에이전트:** 전체 텍스트를 다운로드하고 조사 폴더를 생성함

**사용자:** 고래 상징주의에 대한 하이라이트 추가함

**에이전트:** 멜빌의 저작물에서 고래 상징주의에 관한 3개의 학술 자료를 찾음
```
</collaboration_patterns>

<security_considerations>
## 공유 작업 공간에서의 보안

### 작업 공간 범위 제한 (Scoping)

에이전트에게 전체 파일 시스템에 대한 접근 권한을 주지 마십시오:

```swift
// 좋음: 앱의 도큐먼트 폴더로 범위 제한
let documentsURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]

tool("read_file", { path }) {
    // 경로가 도큐먼트 기준이며, 상위로 탈출 불가
    let fileURL = documentsURL.appendingPathComponent(path)
    guard fileURL.path.hasPrefix(documentsURL.path) else {
        throw ToolError("잘못된 경로")
    }
    return try String(contentsOf: fileURL)
}

// 나쁨: 절대 경로 사용 시 탈출 가능
tool("read_file", { path }) {
    return try String(contentsOf: URL(fileURLWithPath: path))  // /etc/passwd를 읽을 수도 있음!
}
```

### 민감한 파일 보호

```swift
let protectedPaths = [".env", "credentials.json", "secrets/"]

tool("read_file", { path }) {
    if protectedPaths.any({ path.contains($0) }) {
        throw ToolError("보호된 파일에 접근할 수 없습니다")
    }
    // ...
}
```

### 에이전트 행동 감사(Audit)

에이전트가 무엇을 읽고 쓰는지 로깅하십시오:

```swift
func logFileAccess(action: String, path: String, agentId: String) {
    logger.info("[\(agentId)] \(action): \(path)")
}

tool("write_file", { path, content }) {
    logFileAccess(action: "WRITE", path: path, agentId: context.agentId)
    // ...
}
```
</security_considerations>

<examples>
## 실제 사례: Every Reader

Every Reader 앱은 조사를 위해 공유 작업 공간을 사용합니다:

```
Documents/
├── Research/
│   └── book_moby_dick/
│       ├── full_text.txt           # 에이전트가 구텐베르크에서 다운로드
│       ├── introduction.md         # 에이전트가 생성한 개인화 서문
│       ├── sources/
│       │   ├── whale_symbolism.md  # 에이전트가 조사함
│       │   └── melville_bio.md     # 에이전트가 조사함
│       └── user_notes.md           # 사용자가 직접 추가한 노트
├── Chats/
│   └── 2024-01-15.json             # 채팅 이력
└── profile.md                       # 에이전트가 사진으로부터 생성함
```

**작동 방식:**

1. 사용자가 "모비딕"을 라이브러리에 추가함
2. 사용자가 조사 에이전트를 시작함
3. 에이전트가 `Research/book_moby_dick/full_text.txt`에 전체 텍스트를 다운로드함
4. 에이전트가 조사한 내용을 `sources/`에 작성함
5. 에이전트가 사용자의 독서 프로필을 바탕으로 `introduction.md`를 생성함
6. 사용자는 앱 내에서 또는 파일 앱을 통해 모든 파일을 볼 수 있음
7. 사용자가 내용을 정교화하기 위해 `introduction.md`를 편집함
8. 채팅 에이전트는 질문에 답할 때 이 모든 맥락을 읽을 수 있음
</examples>

<icloud_sync>
## 멀티 디바이스 동기화를 위한 iCloud 파일 저장소 (iOS)

에이전트 네이티브 iOS 앱의 경우, 공유 작업 공간으로 iCloud Drive의 Documents 폴더를 사용하십시오. 이를 통해 동기화 계층을 구축하거나 서버를 운영하지 않고도 **무료로 자동 기기 간 동기화**가 가능해집니다.

### 왜 iCloud Documents인가?

| 방식 | 비용 | 복잡도 | 오프라인 | 기기 간 동기화 |
|----------|------|------------|---------|--------------|
| 커스텀 백엔드 + 동기화 | $$$ | 높음 | 수동 구현 | 가능 |
| CloudKit 데이터베이스 | 무료 티어 제한 | 중간 | 수동 구현 | 가능 |
| **iCloud Documents** | 무료 (사용자 용량) | 낮음 | 자동 | 자동 |

iCloud Documents의 특징:
- 사용자의 기존 iCloud 저장 공간 사용 (무료 5GB, 대부분의 사용자는 더 많음)
- 모든 사용자 기기에서 자동 동기화
- 오프라인에서 작동하며, 온라인 시 동기화됨
- 투명성을 위해 '파일(Files.app)' 앱에서 파일 확인 가능
- 서버 비용 없음, 유지보수할 동기화 코드 없음

### 구현 패턴

```swift
// iCloud Documents 컨테이너 URL 가져오기
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
            self.rootURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask).first!
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

### iCloud의 디렉토리 구조

```
iCloud Drive/
└── YourApp/                          # 앱 컨테이너
    └── Documents/                    # 파일 앱에서 보임
        ├── Journal/
        │   ├── user/
        │   │   └── 2025-01-15.md     # 기기 간 동기화됨
        │   └── agent/
        │       └── 2025-01-15.md     # 에이전트의 관찰 결과도 동기화됨
        ├── Experiments/
        │   └── magnesium-sleep/
        │       ├── config.json
        │       └── log.json
        └── Research/
            └── {topic}/
                └── sources.md
```

### 동기화 충돌 처리

iCloud가 충돌을 자동으로 처리하지만, 이를 고려하여 설계해야 합니다:

```swift
// 읽기 시 확인
func readJournalEntry(at url: URL) throws -> JournalEntry {
    // iCloud가 아직 다운로드되지 않은 콘텐츠에 대해 .icloud 플레이스홀더를 만들 수 있음
    if url.pathExtension == "icloud" {
        // 다운로드 트리거
        try FileManager.default.startDownloadingUbiquitousItem(at: url)
        throw FileNotYetAvailableError()
    }

    let data = try Data(contentsOf: url)
    return try JSONDecoder().decode(JournalEntry.self, from: data)
}

// 쓰기 시에는 조정된 파일 접근(coordinated file access) 사용
func writeJournalEntry(_ entry: JournalEntry, to url: URL) throws {
    let coordinator = NSFileCoordinator()
    var error: NSError?

    coordinator.coordinate(writingItemAt: url, options: .forReplacing, error: &error) { newURL in
        let data = try? JSONEncoder().encode(entry)
        try? data?.write(to: newURL)
    }

    if let error = error {
        throw error
    }
}
```

### 이것이 가능하게 하는 것들

1. **사용자가 iPhone에서 실험 시작** → 에이전트가 `Experiments/sleep-tracking/config.json` 생성
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
</icloud_sync>

<checklist>
## 공유 작업 공간 체크리스트

아키텍처:
- [ ] 에이전트와 사용자 데이터를 위한 단일 공유 디렉토리
- [ ] 행위자가 아닌 도메인별로 조직화
- [ ] 파일 도구의 범위가 작업 공간 내로 제한됨 (탈출 방지)
- [ ] 민감한 파일을 위한 보호 경로 설정

도구:
- [ ] `read_file` - 작업 공간 내 모든 파일 읽기
- [ ] `write_file` - 작업 공간 내 모든 파일 쓰기
- [ ] `list_files` - 디렉토리 구조 탐색
- [ ] `search_text` - 파일 전체에서 콘텐츠 찾기 (선택 사항)

UI 통합:
- [ ] UI가 에이전트가 쓰는 동일한 파일을 감시함
- [ ] 변경 사항이 즉시 반영됨 (파일 감시 또는 공유 스토어)
- [ ] 사용자가 에이전트 생성 파일을 편집할 수 있음
- [ ] 에이전트가 덮어쓰기 전 사용자의 수정을 확인

협업:
- [ ] 시스템 프롬프트가 사용자의 파일 편집 가능성을 인지함
- [ ] 에이전트가 덮어쓰기 전 사용자의 수정을 확인
- [ ] 메타데이터로 생성자/수정자 추적 (선택 사항)

멀티 디바이스 (iOS):
- [ ] 공유 작업 공간에 iCloud Documents 사용 (무료 동기화)
- [ ] iCloud 사용 불가 시 로컬 Documents로 폴백 처리
- [ ] `.icloud` 플레이스홀더 파일 처리 (다운로드 트리거)
- [ ] 충돌 없는 쓰기를 위해 NSFileCoordinator 사용
</checklist>
