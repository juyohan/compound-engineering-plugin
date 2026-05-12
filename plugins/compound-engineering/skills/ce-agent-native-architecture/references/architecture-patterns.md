<overview>
에이전트 네이티브 시스템을 구축하기 위한 아키텍처 패턴입니다. 이 패턴들은 패리티(Parity), 세분성(Granularity), 조합성(Composability), 창발적 기능(Emergent Capability), 그리고 지속적 개선이라는 다섯 가지 핵심 원칙에서 파생되었습니다.

기능(Features)은 여러분이 작성하는 함수가 아니라, 에이전트가 루프(loop) 내에서 달성하는 결과입니다. 도구(Tools)는 원자적인 프리미티브(primitives)이며, 에이전트가 판단을 내리고 프롬프트가 결과를 정의합니다.

함께 보기:
- [files-universal-interface.md](./files-universal-interface.md): 파일 구조 및 context.md 패턴
- [agent-execution-patterns.md](./agent-execution-patterns.md): 완료 신호 및 부분적 완료
- [product-implications.md](./product-implications.md): 단계적 공개 및 승인 패턴
</overview>

<pattern name="event-driven-agent">
## 이벤트 기반 에이전트 아키텍처 (Event-Driven Agent Architecture)

에이전트는 이벤트에 응답하는 오래 지속되는 프로세스로 실행됩니다. 이벤트가 곧 프롬프트가 됩니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    에이전트 루프 (Agent Loop)                  │
├─────────────────────────────────────────────────────────────┤
│  이벤트 소스 → 에이전트 (Claude) → 도구 호출 → 응답            │
└─────────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌─────────┐    ┌──────────┐    ┌───────────┐
    │ 콘텐츠  │    │   자체   │    │   데이터  │
    │  도구   │    │   도구   │    │    도구   │
    └─────────┘    └──────────┘    └───────────┘
    (write_file)   (read_source)   (store_item)
                   (restart)       (list_items)
```

**주요 특징:**
- 이벤트(메시지, 웹훅, 타이머)가 에이전트 턴(turn)을 트리거합니다.
- 에이전트는 시스템 프롬프트를 바탕으로 어떻게 응답할지 결정합니다.
- 도구는 비즈니스 로직이 아닌 입출력(IO)을 위한 프리미티브입니다.
- 데이터 도구를 통해 이벤트 간에 상태가 유지됩니다.

**예시: 디스코드 피드백 봇**
```typescript
// 이벤트 소스
client.on("messageCreate", (message) => {
  if (!message.author.bot) {
    runAgent({
      userMessage: `${message.author}로부터의 새 메시지: "${message.content}"`,
      channelId: message.channelId,
    });
  }
});

// 시스템 프롬프트가 동작을 정의함
const systemPrompt = `
누군가 피드백을 공유하면:
1. 따뜻하게 피드백을 확인해주십시오.
2. 필요한 경우 명확히 하기 위한 질문을 하십시오.
3. 피드백 도구를 사용하여 저장하십시오.
4. 피드백 사이트를 업데이트하십시오.

중요도와 분류에 대해서는 당신의 판단을 사용하십시오.
`;
```
</pattern>

<pattern name="two-layer-git">
## 2계층 Git 아키텍처 (Two-Layer Git Architecture)

자가 수정(self-modifying) 에이전트를 위해 코드(공유됨)와 데이터(인스턴스 전용)를 분리합니다.

```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub (공유 저장소)                      │
│  - src/           (에이전트 코드)                            │
│  - site/          (웹 인터페이스)                            │
│  - package.json   (의존성)                                  │
│  - .gitignore     (data/, logs/ 제외)                       │
└─────────────────────────────────────────────────────────────┘
                          │
                     git clone
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  인스턴스 (서버)                             │
│                                                              │
│  GITHUB로부터 (추적됨):                                       │
│  - src/           → 코드 변경 시 push됨                      │
│  - site/          → push되어 배포를 트리거함                  │
│                                                              │
│  로컬 전용 (추적 안 됨):                                      │
│  - data/          → 인스턴스 전용 저장소                     │
│  - logs/          → 런타임 로그                              │
│  - .env           → 비밀 키                                 │
└─────────────────────────────────────────────────────────────┘
```

**이 방식이 작동하는 이유:**
- 코드와 사이트는 버전 관리됩니다 (GitHub).
- 원시 데이터는 로컬에 유지됩니다 (인스턴스 전용).
- 사이트는 데이터로부터 생성되므로 재현 가능합니다.
- git 히스토리를 통해 자동 롤백이 가능합니다.
</pattern>

<pattern name="multi-instance">
## 다중 인스턴스 브랜칭 (Multi-Instance Branching)

각 에이전트 인스턴스는 핵심 코드를 공유하면서 자신만의 브랜치를 가집니다.

```
main                        # 공유 기능, 버그 수정
├── instance/feedback-bot   # Every Reader 피드백 봇
├── instance/support-bot    # 고객 지원 봇
└── instance/research-bot   # 조사 어시스턴트
```

**변경 흐름:**
| 변경 유형 | 작업 위치 | 이후 조치 |
|-------------|---------|------|
| 핵심 기능 | main | 인스턴스 브랜치들로 머지 |
| 버그 수정 | main | 인스턴스 브랜치들로 머지 |
| 인스턴스 구성 | 인스턴스 브랜치 | 완료 |
| 인스턴스 데이터 | 인스턴스 브랜치 | 완료 |

**동기화 도구:**
```typescript
tool("self_deploy", "main으로부터 최신 내용을 가져와 재빌드 및 재시작", ...)
tool("sync_from_instance", "다른 인스턴스로부터 머지", ...)
tool("propose_to_main", "개선 사항 공유를 위해 PR 생성", ...)
```
</pattern>

<pattern name="site-as-output">
## 출력으로서의 사이트 (Site as Agent Output)

에이전트는 특화된 사이트 도구가 아니라 자연스러운 출력으로서 웹사이트를 생성하고 유지 관리합니다.

```
디스코드 메시지
      ↓
에이전트가 처리하여 통찰 추출
      ↓
에이전트가 필요한 사이트 업데이트 결정
      ↓
에이전트가 write_file 프리미티브를 사용하여 파일 작성
      ↓
Git commit + push가 배포 트리거
      ↓
사이트가 자동으로 업데이트됨
```

**핵심 통찰:** 사이트 생성 도구를 만들지 마십시오. 에이전트에게 파일 도구를 주고, 프롬프트를 통해 좋은 사이트를 만드는 방법을 가르치십시오.

```markdown
## 사이트 관리

당신은 공개 피드백 사이트를 유지 관리합니다. 피드백이 들어오면:
1. `write_file`을 사용하여 site/public/content/feedback.json을 업데이트하십시오.
2. 사이트의 React 컴포넌트 개선이 필요하다면 직접 수정하십시오.
3. 변경 사항을 커밋하고 푸시하여 Vercel 배포를 트리거하십시오.

사이트는 다음과 같아야 합니다:
- 깔끔하고 현대적인 대시보드 미학
- 명확한 시각적 계층 구조
- 상태별 정리 (수신함, 진행 중, 완료)

당신이 구조를 결정하십시오. 멋지게 만드세요.
```
</pattern>

<pattern name="approval-gates">
## 승인 게이트 패턴 (Approval Gates Pattern)

위험한 작업에 대해 "제안"과 "적용"을 분리합니다.

```typescript
// 대기 중인 변경 사항을 별도로 저장
const pendingChanges = new Map<string, string>();

tool("write_file", async ({ path, content }) => {
  if (requiresApproval(path)) {
    // 승인을 위해 저장
    pendingChanges.set(path, content);
    const diff = generateDiff(path, content);
    return {
      text: `변경 사항에 승인이 필요합니다.\n\n${diff}\n\n적용하려면 "yes"라고 답해주세요.`
    };
  } else {
    // 즉시 적용
    writeFileSync(path, content);
    return { text: `${path} 파일을 작성했습니다.` };
  }
});

tool("apply_pending", async () => {
  for (const [path, content] of pendingChanges) {
    writeFileSync(path, content);
  }
  pendingChanges.clear();
  return { text: "대기 중인 모든 변경 사항을 적용했습니다" };
});
```

**승인이 필요한 항목:**
- src/*.ts (에이전트 코드)
- package.json (의존성)
- 시스템 프롬프트 변경 사항

**승인이 필요 없는 항목:**
- data/* (인스턴스 데이터)
- site/* (생성된 콘텐츠)
- docs/* (문서)
</pattern>

<pattern name="unified-agent-architecture">
## 통합 에이전트 아키텍처 (Unified Agent Architecture)

하나의 실행 엔진으로 여러 에이전트 유형을 처리합니다. 모든 에이전트는 동일한 오케스트레이터를 사용하지만 구성(configuration)은 다릅니다.

```
┌─────────────────────────────────────────────────────────────┐
│                  AgentOrchestrator                         │
├─────────────────────────────────────────────────────────────┤
│  - 라이프사이클 관리 (시작, 일시정지, 재개, 중지)               │
│  - 체크포인트/복원 (백그라운드 실행용)                         │
│  - 도구 실행                                               │
│  - 채팅 통합                                               │
└─────────────────────────────────────────────────────────────┘
          │                    │                    │
    ┌─────┴─────┐        ┌─────┴─────┐        ┌─────┴─────┐
    │ 조사 에이전트 │        │ 채팅 에이전트 │        │ 프로필 에이전트│
    └───────────┘        └───────────┘        └───────────┘
    - web_search         - read_library       - read_photos
    - write_file         - publish_to_feed    - write_file
    - read_file          - web_search         - analyze_image
```

**구현:**

```swift
// 모든 에이전트는 동일한 오케스트레이터를 사용함
let session = try await AgentOrchestrator.shared.startAgent(
    config: ResearchAgent.create(book: book),  // 구성이 다름
    tools: ResearchAgent.tools,                 // 도구가 다름
    context: ResearchAgent.context(for: book)   // 컨텍스트가 다름
)

// 에이전트 유형별로 자신만의 구성을 정의함
struct ResearchAgent {
    static var tools: [AgentTool] {
        [
            FileTools.readFile(),
            FileTools.writeFile(),
            WebTools.webSearch(),
            WebTools.webFetch(),
        ]
    }

    static func context(for book: Book) -> String {
        """
        당신은 \(book.author)의 "\(book.title)"을 조사하고 있습니다.
        조사 결과는 Documents/Research/\(book.id)/에 저장하십시오.
        """
    }
}

struct ChatAgent {
    static var tools: [AgentTool] {
        [
            FileTools.readFile(),
            FileTools.writeFile(),
            BookTools.readLibrary(),
            BookTools.publishToFeed(),  // 채팅에서 직접 게시 가능
            WebTools.webSearch(),
        ]
    }

    static func context(library: [Book]) -> String {
        """
        당신은 사용자의 독서를 돕습니다.
        가용한 책들: \(library.map { $0.title }.joined(separator: ", "))
        """
    }
}
```

**장점:**
- 모든 에이전트 유형에서 일관된 라이프사이클 관리
- 자동 체크포인트/재개 (모바일에 필수적)
- 도구 프로토콜 공유
- 새로운 에이전트 유형 추가 용이
- 중앙 집중식 오류 처리 및 로깅
</pattern>

<pattern name="agent-to-ui-communication">
## 에이전트-UI 통신 (Agent-to-UI Communication)

에이전트가 액션을 취하면 UI가 즉시 이를 반영해야 합니다. 사용자는 에이전트가 무엇을 했는지 볼 수 있어야 합니다.

**패턴 1: 공유 데이터 저장소 (권장)**

에이전트가 UI가 관찰하는 것과 동일한 서비스를 통해 데이터를 씁니다:

```swift
// 공유 서비스
class BookLibraryService: ObservableObject {
    static let shared = BookLibraryService()
    @Published var books: [Book] = []
    @Published var feedItems: [FeedItem] = []

    func addFeedItem(_ item: FeedItem) {
        feedItems.append(item)
        persist()
    }
}

// 에이전트 도구가 공유 서비스를 통해 데이터를 씀
tool("publish_to_feed", async ({ bookId, content, headline }) => {
    let item = FeedItem(bookId: bookId, content: content, headline: headline)
    BookLibraryService.shared.addFeedItem(item)  // UI가 사용하는 것과 동일한 서비스
    return { text: "피드에 게시했습니다" }
})

// UI가 동일한 서비스를 관찰함
struct FeedView: View {
    @StateObject var library = BookLibraryService.shared

    var body: some View {
        List(library.feedItems) { item in
            FeedItemRow(item: item)
            // 에이전트가 항목을 추가하면 자동으로 업데이트됨
        }
    }
}
```

**패턴 2: 파일 시스템 관찰**

파일 기반 데이터의 경우, 파일 시스템을 감시합니다:

```swift
class ResearchWatcher: ObservableObject {
    @Published var files: [URL] = []
    private var watcher: DirectoryWatcher?

    func watch(bookId: String) {
        let path = documentsURL.appendingPathComponent("Research/\(bookId)")

        watcher = DirectoryWatcher(path: path) { [weak self] in
            self?.reload(from: path)
        }

        reload(from: path)
    }
}

// 에이전트가 파일을 씀
tool("write_file", { path, content }) -> {
    writeFile(documentsURL.appendingPathComponent(path), content)
    // DirectoryWatcher가 자동으로 UI 업데이트를 트리거함
}
```

**패턴 3: 이벤트 버스 (컴포넌트 간 통신)**

여러 독립적인 컴포넌트로 구성된 복잡한 앱의 경우:

```typescript
// 공유 이벤트 버스
const agentEvents = new EventEmitter();

// 에이전트 도구가 이벤트를 방출함
tool("publish_to_feed", async ({ content }) => {
    const item = await feedService.add(content);
    agentEvents.emit('feed:new-item', item);
    return { text: "게시됨" };
});

// UI 컴포넌트가 구독함
function FeedView() {
    const [items, setItems] = useState([]);

    useEffect(() => {
        const handler = (item) => setItems(prev => [...prev, item]);
        agentEvents.on('feed:new-item', handler);
        return () => agentEvents.off('feed:new-item', handler);
    }, []);

    return <FeedList items={items} />;
}
```

**피해야 할 것:**

```swift
// 나쁨: UI가 에이전트의 변경 사항을 관찰하지 않음
// 에이전트가 데이터베이스에 직접 씀
tool("publish_to_feed", { content }) {
    database.insert("feed", content)  // UI는 이를 알지 못함
}

// UI가 시작 시 한 번 로드되고 절대 갱신되지 않음
struct FeedView: View {
    let items = database.query("feed")  // 데이터가 최신이 아님!
}
```
</pattern>

<pattern name="model-tier-selection">
## 모델 티어 선택 (Model Tier Selection)

에이전트마다 서로 다른 지능 수준이 필요합니다. 결과를 달성할 수 있는 가장 저렴한 모델을 사용하십시오.

| 에이전트 유형 | 권장 티어 | 이유 |
|------------|-----------------|-----------|
| 채팅/대화 | Balanced | 빠른 응답, 우수한 추론 |
| 조사 | Balanced | 도구 루프, 아주 복잡한 합성이 아닌 경우 |
| 콘텐츠 생성 | Balanced | 창의적이지만 합성 중심이 아닌 경우 |
| 복잡한 분석 | Powerful | 다중 문서 합성, 미묘한 판단 필요 |
| 프로필/온보딩 | Powerful | 사진 분석, 복잡한 패턴 인식 |
| 단순 질의 | Fast/Haiku | 간단한 조회, 빠른 변환 |

**구현:**

```swift
enum ModelTier {
    case fast      // claude-3-haiku: 빠르고 저렴하며 단순한 작업용
    case balanced  // claude-3-sonnet: 대부분의 작업에 적합한 균형
    case powerful  // claude-3-opus: 복잡한 추론 및 합성용
}

struct AgentConfig {
    let modelTier: ModelTier
    let tools: [AgentTool]
    let systemPrompt: String
}

// 조사 에이전트: balanced 티어
let researchConfig = AgentConfig(
    modelTier: .balanced,
    tools: researchTools,
    systemPrompt: researchPrompt
)

// 프로필 분석: powerful 티어 (복잡한 사진 해석)
let profileConfig = AgentConfig(
    modelTier: .powerful,
    tools: profileTools,
    systemPrompt: profilePrompt
)

// 빠른 조회: fast 티어
let lookupConfig = AgentConfig(
    modelTier: .fast,
    tools: [readLibrary],
    systemPrompt: "사용자의 라이브러리에 대한 빠른 질문에 답합니다."
)
```

**비용 최적화 전략:**
- Balanced 티어로 시작하고, 품질이 부족한 경우에만 업그레이드하십시오.
- 각 턴이 단순한 도구 중심 루프에는 Fast 티어를 사용하십시오.
- Powerful 티어는 합성 작업(여러 소스 비교 등)을 위해 아껴두십시오.
- 비용 제어를 위해 턴당 토큰 제한을 고려하십시오.
</pattern>

<design_questions>
## 설계 시 고려해야 할 질문들

1. **어떤 이벤트가 에이전트 턴을 트리거하는가?** (메시지, 웹훅, 타이머, 사용자 요청)
2. **에이전트에게 어떤 프리미티브가 필요한가?** (읽기, 쓰기, API 호출, 재시작)
3. **에이전트가 어떤 결정을 내려야 하는가?** (형식, 구조, 우선순위, 액션)
4. **어떤 결정이 하드코딩되어야 하는가?** (보안 경계, 승인 요구 사항)
5. **에이전트가 자신의 작업을 어떻게 검증하는가?** (상태 체크, 빌드 검증)
6. **에이전트가 실수로부터 어떻게 복구하는가?** (git 롤백, 승인 게이트)
7. **에이전트가 상태를 변경했을 때 UI가 어떻게 아는가?** (공유 저장소, 파일 감시, 이벤트)
8. **각 에이전트 유형에 어떤 모델 티어가 필요한가?** (fast, balanced, powerful)
9. **에이전트들이 인프라를 어떻게 공유하는가?** (통합 오케스트레이터, 공유 도구)
</design_questions>
