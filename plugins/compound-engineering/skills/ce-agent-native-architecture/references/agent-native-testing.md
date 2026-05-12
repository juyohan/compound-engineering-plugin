<overview>
에이전트 네이티브 앱을 테스트하는 것은 전통적인 유닛 테스트(unit testing)와는 다른 접근 방식이 필요합니다. 특정 함수가 호출되었는지가 아니라, 에이전트가 의도한 결과(outcome)를 달성했는지를 테스트해야 합니다. 이 가이드는 여러분의 앱이 진정으로 에이전트 네이티브인지 검증하기 위한 구체적인 테스트 패턴을 제공합니다.
</overview>

<testing_philosophy>
## 테스트 철학

### 절차가 아닌 결과를 테스트하십시오

**전통적인 방식 (절차 중심):**
```typescript
// 특정 함수가 특정 인자와 함께 호출되었는지 테스트
expect(mockProcessFeedback).toHaveBeenCalledWith({
  message: "좋은 앱이네요!",
  category: "praise",
  priority: 2
});
```

**에이전트 네이티브 방식 (결과 중심):**
```typescript
// 결과가 달성되었는지 테스트
const result = await agent.process("좋은 앱이네요!");
const storedFeedback = await db.feedback.getLatest();

expect(storedFeedback.content).toContain("좋은 앱");
expect(storedFeedback.importance).toBeGreaterThanOrEqual(1);
expect(storedFeedback.importance).toBeLessThanOrEqual(5);
// 에이전트가 어떻게 분류했는지는 중요하지 않습니다 — 결과가 합리적이면 됩니다.
```

### 변동성을 인정하십시오

에이전트는 매번 다른 방식으로 문제를 해결할 수 있습니다. 테스트는 다음을 준수해야 합니다:
- 경로가 아닌 최종 상태를 검증합니다.
- 정확한 값이 아닌 합리적인 범위를 허용합니다.
- 정확한 형식이 아닌 필수 요소의 존재 여부를 확인합니다.
</testing_philosophy>

<can_agent_do_it_test>
## "에이전트가 할 수 있는가?" 테스트

각 UI 기능에 대해 테스트 프롬프트를 작성하고 에이전트가 이를 완수할 수 있는지 확인하십시오.

### 템플릿

```typescript
describe('에이전트 기능 테스트', () => {
  test('에이전트가 라이브러리에 책을 추가할 수 있음', async () => {
    const result = await agent.chat("헤르만 멜빌의 '모비 딕'을 내 라이브러리에 추가해줘");

    // 결과 검증
    const library = await libraryService.getBooks();
    const mobyDick = library.find(b => b.title.includes("모비 딕"));

    expect(mobyDick).toBeDefined();
    expect(mobyDick.author).toContain("멜빌");
  });

  test('에이전트가 피드에 게시할 수 있음', async () => {
    // 준비: 책이 존재함을 보장
    await libraryService.addBook({ id: "book_123", title: "1984" });

    const result = await agent.chat("내 피드에 감시 테마에 관한 글을 써줘");

    // 결과 검증
    const feed = await feedService.getItems();
    const newItem = feed.find(item => item.bookId === "book_123");

    expect(newItem).toBeDefined();
    expect(newItem.content.toLowerCase()).toMatch(/감시|통제|빅 브라더/);
  });

  test('에이전트가 검색하고 조사를 저장할 수 있음', async () => {
    await libraryService.addBook({ id: "book_456", title: "모비 딕" });

    const result = await agent.chat("모비 딕의 고래 상징주의에 대해 조사해줘");

    // 파일이 생성되었는지 검증
    const files = await fileService.listFiles("Research/book_456/");
    expect(files.length).toBeGreaterThan(0);

    // 내용이 관련 있는지 검증
    const content = await fileService.readFile(files[0]);
    expect(content.toLowerCase()).toMatch(/고래|상징|멜빌/);
  });
});
```

### "특정 위치에 쓰기" 테스트

핵심 리트머스 시험(litmus test): 에이전트가 앱의 특정 위치에 콘텐츠를 생성할 수 있는가?

```typescript
describe('위치 인식 테스트', () => {
  const locations = [
    { userPhrase: "내 독서 피드", expectedTool: "publish_to_feed" },
    { userPhrase: "내 라이브러리", expectedTool: "add_book" },
    { userPhrase: "내 조사 폴더", expectedTool: "write_file" },
    { userPhrase: "내 프로필", expectedTool: "write_file" },
  ];

  for (const { userPhrase, expectedTool } of locations) {
    test(`에이전트가 "${userPhrase}"에 쓰는 방법을 알고 있음`, async () => {
      const prompt = `${userPhrase}에 테스트 노트를 작성해줘`;
      const result = await agent.chat(prompt);

      // 에이전트가 올바른 도구를 사용했는지(또는 결과를 달성했는지) 확인
      expect(result.toolCalls).toContainEqual(
        expect.objectContaining({ name: expectedTool })
      );

      // 또는 결과를 직접 검증
      // expect(await locationHasNewContent(userPhrase)).toBe(true);
    });
  }
});
```
</can_agent_do_it_test>

<surprise_test>
## "깜짝 테스트" (Surprise Test)

잘 설계된 에이전트 네이티브 앱은 에이전트가 창의적인 접근 방식을 찾도록 해줍니다. 개방형 요청을 통해 이를 테스트하십시오.

### 테스트 내용

```typescript
describe('에이전트 창의성 테스트', () => {
  test('에이전트가 개방형 요청을 처리할 수 있음', async () => {
    // 준비: 사용자가 몇 권의 책을 가지고 있음
    await libraryService.addBook({ id: "1", title: "1984", author: "오웰" });
    await libraryService.addBook({ id: "2", title: "멋진 신세계", author: "헉슬리" });
    await libraryService.addBook({ id: "3", title: "화씨 451", author: "브래드버리" });

    // 개방형 요청
    const result = await agent.chat("다음 달 독서 계획 세우는 걸 도와줘");

    // 에이전트는 무언가 유용한 일을 해야 합니다.
    // 정확히 무엇을 할지는 지정하지 않습니다 — 그것이 포인트입니다.
    expect(result.toolCalls.length).toBeGreaterThan(0);

    // 라이브러리와 상호작용했어야 합니다.
    const libraryTools = ["read_library", "write_file", "publish_to_feed"];
    const usedLibraryTool = result.toolCalls.some(
      call => libraryTools.includes(call.name)
    );
    expect(usedLibraryTool).toBe(true);
  });

  test('에이전트가 창의적인 해결책을 찾음', async () => {
    // 작업을 달성하는 방법을 지정하지 않습니다.
    const result = await agent.chat(
      "내 SF 소설들에 나타난 디스토피아 테마를 이해하고 싶어"
    );

    // 에이전트는 다음과 같은 일을 할 수 있습니다:
    // - 모든 책을 읽고 비교 문서를 작성함
    // - 디스토피아 문학을 조사하여 사용자의 책과 연결함
    // - 마크다운 파일에 마인드맵을 생성함
    // - 피드에 일련의 통찰을 게시함

    // 우리는 에이전트가 실질적인 작업을 수행했는지만 검증합니다.
    expect(result.response.length).toBeGreaterThan(100);
    expect(result.toolCalls.length).toBeGreaterThan(0);
  });
});
```

### 실패 사례의 모습

```typescript
// 실패: 에이전트가 할 수 없다고만 답함
const result = await agent.chat("독서 토론 모음 준비를 도와줘");

// 나쁜 결과 예시:
expect(result.response).not.toContain("할 수 없습니다");
expect(result.response).not.toContain("도구가 없습니다");
expect(result.response).not.toContain("명확히 말씀해 주시겠어요?");

// 에이전트가 당연히 이해해야 할 내용에 대해 명확히 해달라고 요청한다면,
// 컨텍스트 주입이나 기능 격차가 존재하는 것입니다.
```
</surprise_test>

<parity_testing>
## 자동화된 패리티 테스트 (Automated Parity Testing)

모든 UI 액션이 에이전트 도구와 상응하는지 확인하십시오.

### 기능 지도(Capability Map) 테스트

```typescript
// capability-map.ts
export const capabilityMap = {
  // UI 액션: 에이전트 도구
  "라이브러리 보기": "read_library",
  "책 추가": "add_book",
  "책 삭제": "delete_book",
  "통찰 게시": "publish_to_feed",
  "조사 시작": "start_research",
  "하이라이트 보기": "read_library",  // 동일한 도구, 다른 쿼리
  "프로필 수정": "write_file",
  "웹 검색": "web_search",
  "데이터 내보내기": "N/A",  // UI 전용 액션
};

// parity.test.ts
import { capabilityMap } from './capability-map';
import { getAgentTools } from './agent-config';
import { getSystemPrompt } from './system-prompt';

describe('액션 패리티', () => {
  const agentTools = getAgentTools();
  const systemPrompt = getSystemPrompt();

  for (const [uiAction, toolName] of Object.entries(capabilityMap)) {
    if (toolName === 'N/A') continue;

    test(`"${uiAction}"에 상응하는 에이전트 도구 ${toolName}이(가) 존재함`, () => {
      const toolNames = agentTools.map(t => t.name);
      expect(toolNames).toContain(toolName);
    });

    test(`${toolName}이(가) 시스템 프롬프트에 문서화됨`, () => {
      expect(systemPrompt).toContain(toolName);
    });
  }
});
```

### 컨텍스트 패리티(Context Parity) 테스트

```typescript
describe('컨텍스트 패리티', () => {
  test('에이전트가 UI가 보여주는 모든 데이터를 볼 수 있음', async () => {
    // 준비: 데이터 생성
    await libraryService.addBook({ id: "1", title: "테스트 도서" });
    await feedService.addItem({ id: "f1", content: "테스트 통찰" });

    // 컨텍스트가 포함된 시스템 프롬프트 빌드
    const systemPrompt = await buildSystemPrompt();

    // 데이터가 포함되었는지 검증
    expect(systemPrompt).toContain("테스트 도서");
    expect(systemPrompt).toContain("테스트 통찰");
  });

  test('최근 활동이 에이전트에게 보임', async () => {
    // 액션 수행
    await activityService.log({ action: "highlighted", bookId: "1" });
    await activityService.log({ action: "researched", bookId: "2" });

    const systemPrompt = await buildSystemPrompt();

    // 활동 내용이 포함되었는지 검증
    expect(systemPrompt).toMatch(/highlighted|researched/);
  });
});
```
</parity_testing>

<integration_testing>
## 통합 테스트 (Integration Testing)

사용자 요청부터 결과까지의 전체 흐름을 테스트하십시오.

### End-to-End 흐름 테스트

```typescript
describe('End-to-End 흐름', () => {
  test('조사 흐름: 요청 → 웹 검색 → 파일 생성', async () => {
    // 준비
    const bookId = "book_123";
    await libraryService.addBook({ id: bookId, title: "모비 딕" });

    // 사용자 요청
    await agent.chat("모비 딕의 포경 역사적 배경에 대해 조사해줘");

    // 검증: 웹 검색이 수행됨
    const searchCalls = mockWebSearch.mock.calls;
    expect(searchCalls.length).toBeGreaterThan(0);
    expect(searchCalls.some(call =>
      call[0].query.toLowerCase().includes("포경")
    )).toBe(true);

    // 검증: 파일이 생성됨
    const researchFiles = await fileService.listFiles(`Research/${bookId}/`);
    expect(researchFiles.length).toBeGreaterThan(0);

    // 검증: 내용이 관련 있음
    const content = await fileService.readFile(researchFiles[0]);
    expect(content.toLowerCase()).toMatch(/고래|포경|낸터킷|멜빌/);
  });

  test('게시 흐름: 요청 → 도구 호출 → 피드 업데이트 → UI 반영', async () => {
    // 준비
    await libraryService.addBook({ id: "book_1", title: "1984" });

    // 초기 상태
    const feedBefore = await feedService.getItems();

    // 사용자 요청
    await agent.chat("내 독서 피드에 빅 브라더에 관한 글을 써줘");

    // 피드가 업데이트되었는지 검증
    const feedAfter = await feedService.getItems();
    expect(feedAfter.length).toBe(feedBefore.length + 1);

    // 내용 검증
    const newItem = feedAfter.find(item =>
      !feedBefore.some(old => old.id === item.id)
    );
    expect(newItem).toBeDefined();
    expect(newItem.content.toLowerCase()).toMatch(/빅 브라더|감시|지켜보고 있다/);
  });
});
```

### 실패 복구 테스트

```typescript
describe('실패 복구', () => {
  test('에이전트가 존재하지 않는 책에 대해 적절히 처리함', async () => {
    const result = await agent.chat("'존재하지 않는 책'에 대해 알려줘");

    // 에이전트가 크래시되지 않아야 함
    expect(result.error).toBeUndefined();

    // 에이전트가 문제를 인지해야 함
    expect(result.response.toLowerCase()).toMatch(
      /찾을 수 없음|보이지 않음|라이브러리/
    );
  });

  test('에이전트가 API 실패로부터 복구함', async () => {
    // API 실패 모킹
    mockWebSearch.mockRejectedValueOnce(new Error("네트워크 오류"));

    const result = await agent.chat("이 주제에 대해 조사해줘");

    // 적절히 처리해야 함
    expect(result.error).toBeUndefined();
    expect(result.response).not.toContain("unhandled exception");

    // 문제를 사용자에게 알려야 함
    expect(result.response.toLowerCase()).toMatch(
      /조사할 수 없음|할 수 없음|다시 시도/
    );
  });
});
```
</integration_testing>

<snapshot_testing>
## 시스템 프롬프트 스냅샷 테스트

시간에 따른 시스템 프롬프트 및 컨텍스트 주입의 변화를 추적하십시오.

```typescript
describe('시스템 프롬프트 안정성', () => {
  test('시스템 프롬프트 구조가 스냅샷과 일치함', async () => {
    const systemPrompt = await buildSystemPrompt();

    // 동적 데이터를 제거하고 구조 추출
    const structure = systemPrompt
      .replace(/id: \w+/g, 'id: [ID]')
      .replace(/"[^"]+"/g, '"[TITLE]"')
      .replace(/\d{4}-\d{2}-\d{2}/g, '[DATE]');

    expect(structure).toMatchSnapshot();
  });

  test('모든 기능 섹션이 존재함', async () => {
    const systemPrompt = await buildSystemPrompt();

    const requiredSections = [
      "당신의 기능",
      "가용한 책 목록",
      "최근 활동",
    ];

    for (const section of requiredSections) {
      expect(systemPrompt).toContain(section);
    }
  });
});
```
</snapshot_testing>

<manual_testing>
## 수동 테스트 체크리스트

어떤 것들은 개발 중에 수동으로 테스트하는 것이 가장 좋습니다:

### 자연어 변형 테스트 (Natural Language Variation Test)

동일한 요청에 대해 다양한 표현을 시도해 보십시오:

```
"이거 내 피드에 추가해줘"
"내 독서 피드에 글 하나 써줘"
"이에 대한 통찰을 게시해줘"
"피드에 올려줘"
"내 피드에 이게 있었으면 좋겠어"
```

컨텍스트 주입이 올바르다면 모두 작동해야 합니다.

### 에지 케이스 프롬프트

```
"넌 뭘 할 수 있니?"
→ 에이전트가 자신의 기능을 설명해야 함

"내 책들 좀 도와줘"
→ 에이전트가 "책"이 무엇인지 묻지 않고 라이브러리와 상호작용해야 함

"글 하나 써줘"
→ 어디에 쓸지(피드, 파일 등) 명확하지 않다면 에이전트가 물어봐야 함

"다 지워줘"
→ 파괴적인 액션 전에는 에이전트가 확인을 요청해야 함
```

### 혼란 테스트 (Confusion Test)

존재해야 하지만 제대로 연결되지 않았을 수 있는 것들에 대해 물어보십시오:

```
"내 조사 폴더에 뭐가 들어있니?"
→ "조사 폴더가 뭐죠?"라고 묻지 않고 파일 목록을 보여줘야 함

"최근에 내가 읽은 것들 보여줘"
→ "무슨 뜻이죠?"라고 묻지 않고 활동 내역을 보여줘야 함

"중단했던 부분부터 계속할래"
→ 가용한 경우 최근 활동을 참조해야 함
```
</manual_testing>

<ci_integration>
## CI/CD 통합

CI 파이프라인에 에이전트 네이티브 테스트를 추가하십시오:

```yaml
# .github/workflows/test.yml
name: 에이전트 네이티브 테스트

on: [push, pull_request]

jobs:
  agent-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: 준비
        run: npm install

      - name: 패리티 테스트 실행
        run: npm run test:parity

      - name: 기능 테스트 실행
        run: npm run test:capabilities
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: 시스템 프롬프트 완결성 확인
        run: npm run test:system-prompt

      - name: 기능 지도 검증
        run: |
          # 기능 지도가 최신인지 확인
          npm run generate:capability-map
          git diff --exit-code capability-map.ts
```

### 비용 고려 테스트

에이전트 테스트는 API 토큰 비용이 발생합니다. 관리 전략:

```typescript
// 기본적인 테스트에는 더 작은 모델 사용
const testConfig = {
  model: process.env.CI ? "claude-3-haiku" : "claude-3-opus",
  maxTokens: 500,  // 출력 길이 제한
};

// 결정론적 테스트를 위해 응답 캐싱
const cachedAgent = new CachedAgent({
  cacheDir: ".test-cache",
  ttl: 24 * 60 * 60 * 1000,  // 24시간
});

// 비싼 테스트는 main 브랜치에서만 실행
if (process.env.GITHUB_REF === 'refs/heads/main') {
  describe('전체 통합 테스트', () => { ... });
}
```
</ci_integration>

<test_utilities>
## 테스트 유틸리티 (Test Utilities)

### 에이전트 테스트 하네스 (Agent Test Harness)

```typescript
class AgentTestHarness {
  private agent: Agent;
  private mockServices: MockServices;

  async setup() {
    this.mockServices = createMockServices();
    this.agent = await createAgent({
      services: this.mockServices,
      model: "claude-3-haiku",  // 테스트용으로 더 저렴한 모델
    });
  }

  async chat(message: string): Promise<AgentResponse> {
    return this.agent.chat(message);
  }

  async expectToolCall(toolName: string) {
    const lastResponse = this.agent.getLastResponse();
    expect(lastResponse.toolCalls.map(t => t.name)).toContain(toolName);
  }

  async expectOutcome(check: () => Promise<boolean>) {
    const result = await check();
    expect(result).toBe(true);
  }

  getState() {
    return {
      library: this.mockServices.library.getBooks(),
      feed: this.mockServices.feed.getItems(),
      files: this.mockServices.files.listAll(),
    };
  }
}

// 사용 예시
test('전체 흐름', async () => {
  const harness = new AgentTestHarness();
  await harness.setup();

  await harness.chat("라이브러리에 '모비 딕' 추가해줘");
  await harness.expectToolCall("add_book");
  await harness.expectOutcome(async () => {
    const state = harness.getState();
    return state.library.some(b => b.title.includes("모비"));
  });
});
```
</test_utilities>

<checklist>
## 테스트 체크리스트

자동화된 테스트:
- [ ] 모든 UI 액션에 대한 "에이전트가 할 수 있는가?" 테스트
- [ ] 위치 인식 테스트 ("내 피드에 써줘")
- [ ] 패리티 테스트 (도구 존재 여부, 프롬프트 내 문서화 여부)
- [ ] 컨텍스트 패리티 테스트 (에이전트가 UI가 보여주는 것을 보는지)
- [ ] End-to-End 흐름 테스트
- [ ] 실패 복구 테스트

수동 테스트:
- [ ] 자연어 변형 (다양한 표현이 작동하는지)
- [ ] 에지 케이스 프롬프트 (개방형 요청)
- [ ] 혼란 테스트 (에이전트가 앱 어휘를 아는지)
- [ ] 깜짝 테스트 (에이전트가 창의적일 수 있는지)

CI 통합:
- [ ] 모든 PR에서 패리티 테스트 실행
- [ ] API 키를 사용한 기능 테스트 실행
- [ ] 시스템 프롬프트 완결성 체크
- [ ] 기능 지도 괴리(drift) 감지
</checklist>
