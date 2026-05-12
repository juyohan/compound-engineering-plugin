<overview>
에이전트 시스템 프롬프트에 동적인 런타임 컨텍스트(runtime context)를 주입하는 방법입니다. 에이전트는 앱에 무엇이 존재하는지 알아야 무엇을 가지고 작업할 수 있는지 알 수 있습니다. 정적인 프롬프트만으로는 부족합니다 — 에이전트는 사용자가 보는 것과 동일한 컨텍스트를 볼 수 있어야 합니다.

**핵심 원칙:** 사용자의 컨텍스트가 곧 에이전트의 컨텍스트입니다.
</overview>

<why_context_matters>
## 왜 동적 컨텍스트 주입이 필요한가요?

정적인 시스템 프롬프트는 에이전트에게 무엇을 할 수 있는지(CAN do) 알려줍니다. 동적 컨텍스트는 사용자의 실제 데이터를 가지고 지금 당장(RIGHT NOW) 무엇을 할 수 있는지 알려줍니다.

**실패 사례:**
```
사용자: "내 독서 피드에 예카테리나 대제에 관한 글을 하나 써줘"
에이전트: "어떤 시스템을 말씀하시는 건가요? 독서 피드가 무엇인지 잘 모르겠습니다."
```

에이전트가 실패한 이유는 다음을 몰랐기 때문입니다:
- 사용자의 라이브러리에 어떤 책들이 있는지
- "독서 피드"가 무엇인지
- 거기에 게시하기 위해 어떤 도구를 가지고 있는지

**해결 방법:** 앱 상태에 대한 런타임 컨텍스트를 시스템 프롬프트에 주입하십시오.
</why_context_matters>

<pattern name="context-injection">
## 컨텍스트 주입 패턴 (The Context Injection Pattern)

현재 앱 상태를 포함하여 시스템 프롬프트를 동적으로 구성하십시오:

```swift
func buildSystemPrompt() -> String {
    // 현재 상태 수집
    let availableBooks = libraryService.books
    let recentActivity = analysisService.recentRecords(limit: 10)
    let userProfile = profileService.currentProfile

    return """
    # 당신의 정체성

    당신은 \(userProfile.name)의 라이브러리를 위한 독서 어시스턴트입니다.

    ## 사용자 라이브러리의 가용한 책 목록

    \(availableBooks.map { "- \"\($0.title)\" (저자: \($0.author), id: \($0.id))" }.joined(separator: "\n"))

    ## 최근 독서 활동

    \(recentActivity.map { "- \"\($0.bookTitle)\" 분석함: \($0.excerptPreview)" }.joined(separator: "\n"))

    ## 당신의 기능

    - **publish_to_feed**: 피드 탭에 나타나는 통찰 생성
    - **read_library**: 책, 하이라이트 및 분석 결과 보기
    - **web_search**: 조사를 위해 인터넷 검색
    - **write_file**: Documents/Research/{bookId}/에 조사 결과 저장

    사용자가 "피드" 또는 "독서 피드"를 언급하면, 통찰이 나타나는 피드 탭을 의미합니다.
    거기에 콘텐츠를 생성하려면 `publish_to_feed`를 사용하십시오.
    """
}
```
</pattern>

<what_to_inject>
## 주입해야 할 컨텍스트 목록

### 1. 가용 자원 (Available Resources)
에이전트가 접근할 수 있는 어떤 데이터/파일이 존재하는가?

```swift
## 사용자 라이브러리의 가용 자원

책:
- 헤르만 멜빌의 "모비 딕" (id: book_123)
- 조지 오웰의 "1984" (id: book_456)

조사 폴더:
- Documents/Research/book_123/ (파일 3개)
- Documents/Research/book_456/ (파일 1개)
```

### 2. 현재 상태 (Current State)
사용자가 최근에 무엇을 했는가? 현재 문맥은 무엇인가?

```swift
## 최근 활동

- 2시간 전: "1984"에서 감시에 관한 문구 하이라이트함
- 어제: "모비 딕" 고래 상징주의 조사 완료함
- 이번 주: 라이브러리에 새 책 3권 추가함
```

### 3. 기능 매핑 (Capabilities Mapping)
어떤 도구가 어떤 UI 기능과 매칭되는가? 사용자의 언어를 사용하십시오.

```swift
## 당신이 할 수 있는 일

| 사용자의 말 | 사용해야 할 도구 | 결과 |
|-----------|----------------|--------|
| "내 피드" / "독서 피드" | `publish_to_feed` | 피드 탭에 통찰 생성 |
| "내 라이브러리" / "내 책들" | `read_library` | 도서 컬렉션 표시 |
| "이거 조사해줘" | `web_search` + `write_file` | 조사 폴더에 저장 |
| "내 프로필" | `read_file("profile.md")` | 독서 프로필 표시 |
```

### 4. 도메인 어휘 (Domain Vocabulary)
사용자가 사용할 수 있는 앱 특유의 용어들을 설명하십시오.

```swift
## 용어 설명

- **피드 (Feed)**: 독서 통찰과 분석 결과가 표시되는 피드 탭
- **조사 폴더 (Research folder)**: 조사 결과가 저장되는 Documents/Research/{bookId}/ 경로
- **독서 프로필 (Reading profile)**: 사용자의 독서 취향이 기술된 마크다운 파일
- **하이라이트 (Highlight)**: 사용자가 책에서 표시한 문구
```
</what_to_inject>

<implementation_patterns>
## 구현 패턴

### 패턴 1: 서비스 기반 주입 (Swift/iOS)

```swift
class AgentContextBuilder {
    let libraryService: BookLibraryService
    let profileService: ReadingProfileService
    let activityService: ActivityService

    func buildContext() -> String {
        let books = libraryService.books
        let profile = profileService.currentProfile
        let activity = activityService.recent(limit: 10)

        return """
        ## 라이브러리 (책 \(books.count)권)
        \(formatBooks(books))

        ## 프로필
        \(profile.summary)

        ## 최근 활동
        \(formatActivity(activity))
        """
    }

    private func formatBooks(_ books: [Book]) -> String {
        books.map { "- \"\($0.title)\" (id: \($0.id))" }.joined(separator: "\n")
    }
}

// 에이전트 초기화 시 사용
let context = AgentContextBuilder(
    libraryService: .shared,
    profileService: .shared,
    activityService: .shared
).buildContext()

let systemPrompt = basePrompt + "\n\n" + context
```

### 패턴 2: 훅(Hook) 기반 주입 (TypeScript)

```typescript
interface ContextProvider {
  getContext(): Promise<string>;
}

class LibraryContextProvider implements ContextProvider {
  async getContext(): Promise<string> {
    const books = await db.books.list();
    const recent = await db.activity.recent(10);

    return `
## 라이브러리
${books.map(b => `- "${b.title}" (${b.id})`).join('\n')}

## 최근 활동
${recent.map(r => `- ${r.description}`).join('\n')}
    `.trim();
  }
}

// 여러 프로바이더 조합
async function buildSystemPrompt(providers: ContextProvider[]): Promise<string> {
  const contexts = await Promise.all(providers.map(p => p.getContext()));
  return [BASE_PROMPT, ...contexts].join('\n\n');
}
```

### 패턴 3: 템플릿 기반 주입

```markdown
# 시스템 프롬프트 템플릿 (system-prompt.template.md)

당신은 독서 어시스턴트입니다.

## 가용한 책 목록

{{#each books}}
- "{{title}}" (저자: {{author}}, id: {{id}})
{{/each}}

## 기능

{{#each capabilities}}
- **{{name}}**: {{description}}
{{/each}}

## 최근 활동

{{#each recentActivity}}
- {{timestamp}}: {{description}}
{{/each}}
```

```typescript
// 런타임에 렌더링
const prompt = Handlebars.compile(template)({
  books: await libraryService.getBooks(),
  capabilities: getCapabilities(),
  recentActivity: await activityService.getRecent(10),
});
```
</implementation_patterns>

<context_freshness>
## 컨텍스트 최신성 (Context Freshness)

컨텍스트는 에이전트 초기화 시점에 주입되어야 하며, 긴 세션의 경우 선택적으로 갱신되어야 합니다.

**초기화 시:**
```swift
// 에이전트 시작 시 항상 최신 컨텍스트 주입
func startChatAgent() async -> AgentSession {
    let context = await buildCurrentContext()  // 최신 컨텍스트
    return await AgentOrchestrator.shared.startAgent(
        config: ChatAgent.config,
        systemPrompt: basePrompt + context
    )
}
```

**긴 세션 도중 (선택 사항):**
```swift
// 오래 실행되는 에이전트를 위해 갱신 도구 제공
tool("refresh_context", "현재 앱 상태 가져오기") { _ in
    let books = libraryService.books
    let recent = activityService.recent(10)
    return """
    현재 라이브러리: 책 \(books.count)권
    최근 활동: \(recent.map { $0.summary }.joined(separator: ", "))
    """
}
```

**피해야 할 것:**
```swift
// 나쁨: 앱 실행 시점의 오래된 컨텍스트 사용
let cachedContext = appLaunchContext  // 최신이 아님!
// 책이 추가되었거나 활동 내역이 변경되었을 수 있음
```
</context_freshness>

<examples>
## 실제 사례: Every Reader

Every Reader 앱은 채팅 에이전트를 위해 컨텍스트를 주입합니다:

```swift
func getChatAgentSystemPrompt() -> String {
    // 현재 라이브러리 상태 가져오기
    let books = BookLibraryService.shared.books
    let analyses = BookLibraryService.shared.analysisRecords.prefix(10)
    let profile = ReadingProfileService.shared.getProfileForSystemPrompt()

    let bookList = books.map { book in
        "- \"\(book.title)\" (저자: \(book.author), id: \(book.id))"
    }.joined(separator: "\n")

    let recentList = analyses.map { record in
        let title = books.first { $0.id == record.bookId }?.title ?? "알 수 없음"
        return "- \"\(title)\"에서: \"\(record.excerptPreview)\""
    }.joined(separator: "\n")

    return """
    # 독서 어시스턴트

    당신은 사용자의 독서와 도서 조사를 돕습니다.

    ## 사용자 라이브러리의 가용한 책 목록

    \(bookList.isEmpty ? "아직 책이 없습니다." : bookList)

    ## 최근 독서 저널 (최신 분석 결과)

    \(recentList.isEmpty ? "아직 분석 결과가 없습니다." : recentList)

    ## 독서 프로필

    \(profile)

    ## 당신의 기능

    - **피드에 게시**: `publish_to_feed`를 사용하여 피드 탭에 나타나는 통찰 생성
    - **라이브러리 접근**: `read_library`를 사용하여 책과 하이라이트 보기
    - **조사**: 웹을 검색하고 Documents/Research/{bookId}/에 저장
    - **프로필**: 사용자의 독서 프로필 읽기/업데이트

    사용자가 "피드에 글을 써달라"거나 "독서 피드에 추가해달라"고 요청하면,
    관련된 book_id와 함께 `publish_to_feed` 도구를 사용하십시오.
    """
}
```

**결과:** 사용자가 "내 독서 피드에 예카테리나 대제에 관한 글을 하나 써줘"라고 말하면, 에이전트는:
1. "독서 피드"를 확인 → `publish_to_feed`를 사용해야 함을 인지
2. 가용한 책 목록을 확인 → 관련된 책 ID를 찾음
3. 피드 탭에 적절한 콘텐츠를 생성함
</examples>

<checklist>
## 컨텍스트 주입 체크리스트

에이전트를 실행하기 전에:
- [ ] 시스템 프롬프트에 현재 자원(책, 파일, 데이터)이 포함되어 있는가?
- [ ] 최근 활동이 에이전트에게 보이는가?
- [ ] 기능들이 사용자의 어휘와 매핑되어 있는가?
- [ ] 도메인 특화 용어들이 설명되어 있는가?
- [ ] 컨텍스트가 최신인가? (에이전트 시작 시점에 수집되었는가?)

새로운 기능을 추가할 때:
- [ ] 새로운 자원이 컨텍스트 주입에 포함되었는가?
- [ ] 새로운 기능이 시스템 프롬프트에 문서화되었는가?
- [ ] 해당 기능에 대한 사용자 어휘가 매핑되었인가?
</checklist>
