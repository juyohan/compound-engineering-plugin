<overview>
에이전트가 사용자가 할 수 있는 모든 것을 할 수 있도록 보장하기 위한 구조화된 규율입니다. 모든 UI 액션(action)은 그에 상응하는 에이전트 도구(tool)를 가져야 합니다. 이는 일회성 체크가 아니라, 개발 워크플로우에 통합된 지속적인 관행입니다.

**핵심 원칙:** UI 기능을 추가할 때 동일한 PR에서 그에 상응하는 도구를 추가하십시오.
</overview>

<why_parity>
## 액션 패리티(Action Parity)가 중요한 이유

**실패 사례:**
```
사용자: "내 독서 피드에 예카테리나 대제에 관한 글을 하나 써줘"
에이전트: "어떤 시스템을 말씀하시는 건가요? 독서 피드가 무엇인지 잘 모르겠습니다."
```

사용자는 UI를 통해 피드에 글을 게시할 수 있었습니다. 하지만 에이전트에게는 `publish_to_feed` 도구가 없었습니다. 해결 방법은 간단했습니다 — 도구를 추가하는 것이죠. 하지만 여기서 얻은 통찰은 심오합니다:

**사용자가 UI를 통해 취할 수 있는 모든 액션은 에이전트가 호출할 수 있는 상응하는 도구를 가져야 합니다.**

이 패리티(동등성)가 없다면:
- 사용자는 에이전트가 할 수 없는 일을 요청하게 됩니다.
- 에이전트는 당연히 이해해야 할 기능에 대해 명확히 해달라는 질문을 던지게 됩니다.
- 사용자는 직접 앱을 사용하는 것에 비해 에이전트가 제한적이라고 느끼게 됩니다.
- 사용자는 에이전트의 능력에 대한 신뢰를 잃게 됩니다.
</why_parity>

<capability_mapping>
## 기능 지도 (Capability Map)

UI 액션과 에이전트 도구의 구조화된 지도를 유지하십시오:

| UI 액션 | UI 위치 | 에이전트 도구 | 시스템 프롬프트 참조 |
|-----------|-------------|------------|-------------------------|
| 라이브러리 보기 | 라이브러리 탭 | `read_library` | "책과 하이라이트 보기" |
| 책 추가 | 라이브러리 → 추가 | `add_book` | "라이브러리에 책 추가" |
| 통찰 게시 | 분석 보기 | `publish_to_feed` | "피드 탭을 위한 통찰 생성" |
| 조사 시작 | 책 상세 정보 | `start_research` | "웹 검색을 통해 책 조사" |
| 프로필 수정 | 설정 | `write_file(profile.md)` | "독서 프로필 업데이트" |
| 스크린샷 찍기 | 카메라 | N/A (사용자 전용 액션) | — |
| 웹 검색 | 채팅 | `web_search` | "인터넷 검색" |

**기능을 추가할 때마다 이 테이블을 업데이트하십시오.**

### 앱을 위한 템플릿

```markdown
# 기능 지도 - [앱 이름]

| UI 액션 | UI 위치 | 에이전트 도구 | 시스템 프롬프트 | 상태 |
|-----------|-------------|------------|---------------|--------|
| | | | | ⚠️ 누락됨 |
| | | | | ✅ 완료 |
| | | | | 🚫 해당 없음 |
```

상태 의미:
- ✅ 완료: 도구가 존재하며 시스템 프롬프트에 문서화됨
- ⚠️ 누락됨: UI 액션은 존재하나 상응하는 에이전트 도구가 없음
- 🚫 해당 없음: 사용자 전용 액션 (예: 생체 인증, 카메라 촬영)
</capability_mapping>

<parity_workflow>
## 액션 패리티 워크플로우

### 새로운 기능을 추가할 때

UI 기능을 추가하는 PR을 머지하기 전에:

```
1. 이것은 어떤 액션인가?
   → "사용자가 독서 피드에 통찰을 게시할 수 있음"

2. 이에 대한 에이전트 도구가 존재하는가?
   → 도구 정의 확인
   → NO인 경우: 도구 생성

3. 시스템 프롬프트에 문서화되어 있는가?
   → 시스템 프롬프트의 기능(capabilities) 섹션 확인
   → NO인 경우: 문서화 추가

4. 컨텍스트가 가용한가?
   → 에이전트가 "피드"가 무엇인지 아는가?
   → 에이전트가 가용한 책들을 볼 수 있는가?
   → NO인 경우: 컨텍스트 주입(context injection)에 추가

5. 기능 지도를 업데이트한다
   → 추적 문서에 행 추가
```

### PR 체크리스트

PR 템플릿에 다음을 추가하십시오:

```markdown
## 에이전트 네이티브 체크리스트

- [ ] 모든 새로운 UI 액션에 상응하는 에이전트 도구가 있음
- [ ] 새로운 기능을 언급하도록 시스템 프롬프트가 업데이트됨
- [ ] 에이전트가 UI가 사용하는 것과 동일한 데이터에 접근할 수 있음
- [ ] 기능 지도가 업데이트됨
- [ ] 자연어 요청으로 테스트 완료됨
```
</parity_workflow>

<parity_audit>
## 패리티 감사 (Parity Audit)

주기적으로 앱의 액션 패리티 격차를 감사하십시오:

### 1단계: 모든 UI 액션 나열

모든 화면을 훑어보며 사용자가 할 수 있는 일을 나열하십시오:

```
라이브러리 화면:
- 책 목록 보기
- 책 검색
- 카테고리별 필터링
- 새 책 추가
- 책 삭제
- 책 상세 정보 열기

책 상세 화면:
- 책 정보 보기
- 조사 시작
- 하이라이트 보기
- 하이라이트 추가
- 책 공유
- 라이브러리에서 제거

피드 화면:
- 통찰 보기
- 새 통찰 생성
- 통찰 수정
- 통찰 삭제
- 통찰 공유

설정:
- 프로필 수정
- 테마 변경
- 데이터 내보내기
- 계정 삭제
```

### 2단계: 도구 커버리지 확인

각 액션에 대해 확인하십시오:

```
✅ 책 목록 보기        → read_library
✅ 책 검색            → read_library (쿼리 파라미터 포함)
⚠️ 카테고리별 필터링  → 누락 (read_library에 필터 파라미터 추가 필요)
⚠️ 새 책 추가         → 누락 (add_book 도구 필요)
✅ 책 삭제            → delete_book
✅ 책 상세 정보 열기   → read_library (단일 도서)

✅ 조사 시작          → start_research
✅ 하이라이트 보기     → read_library (하이라이트 포함)
⚠️ 하이라이트 추가     → 누락 (add_highlight 도구 필요)
⚠️ 책 공유            → 누락 (또는 공유가 UI 전용인 경우 N/A)

✅ 통찰 보기          → read_library (피드 포함)
✅ 새 통찰 생성        → publish_to_feed
⚠️ 통찰 수정          → 누락 (update_feed_item 도구 필요)
⚠️ 통찰 삭제          → 누락 (delete_feed_item 도구 필요)
```

### 3단계: 격차 우선순위 지정

모든 격차가 동일하게 중요한 것은 아닙니다:

**높은 우선순위 (사용자가 이를 요청할 가능성이 높음):**
- 새 책 추가
- 콘텐츠 생성/수정/삭제
- 핵심 워크플로우 액션

**중간 우선순위 (가끔 발생하는 요청):**
- 필터/검색 변형
- 내보내기 기능
- 공유 기능

**낮은 우선순위 (에이전트를 통해 요청되는 경우가 드문 경우):**
- 테마 변경
- 계정 삭제
- UI 개인 취향에 해당하는 설정
</parity_audit>

<tool_design_for_parity>
## 패리티를 위한 도구 설계

### 도구의 세분성을 UI 세분성과 일치시키십시오

UI에 "수정"과 "삭제" 버튼이 따로 있다면, 도구도 따로 만드는 것을 고려하십시오:

```typescript
// UI 세분성과 일치함
tool("update_feed_item", { id, content, headline }, ...);
tool("delete_feed_item", { id }, ...);

// vs. 하나로 합쳐짐 (에이전트가 발견하기 더 어려움)
tool("modify_feed_item", { id, action: "update" | "delete", ... }, ...);
```

### 도구 이름에 사용자 어휘를 사용하십시오

```typescript
// 좋음: 사용자가 말하는 방식과 일치함
tool("publish_to_feed", ...);  // "내 피드에 게시해줘"
tool("add_book", ...);         // "이 책 추가해줘"
tool("start_research", ...);   // "이거 조사 시작해줘"

// 나쁨: 기술적인 전문 용어
tool("create_analysis_record", ...);
tool("insert_library_item", ...);
tool("initiate_web_scrape_workflow", ...);
```

### UI가 보여주는 것을 반환하십시오

UI가 상세 정보와 함께 확인 메시지를 보여준다면, 도구도 그렇게 해야 합니다:

```typescript
// UI 표시: "라이브러리에 '모비 딕'을 추가했습니다"
// 도구도 동일하게 반환해야 함:
tool("add_book", async ({ title, author }) => {
  const book = await library.add({ title, author });
  return {
    text: `라이브러리에 ${book.author}의 "${book.title}"을 추가했습니다 (id: ${book.id})`
  };
});
```
</tool_design_for_parity>

<context_parity>
## 컨텍스트 패리티 (Context Parity)

사용자가 보는 것은 무엇이든 에이전트도 접근할 수 있어야 합니다.

### 문제 상황

```swift
// UI는 최근 분석 결과들을 리스트로 보여줌
ForEach(analysisRecords) { record in
    AnalysisRow(record: record)
}

// 하지만 시스템 프롬프트는 책만 언급하고 분석 결과는 언급하지 않음
let systemPrompt = """
## 가용한 책 목록
\(books.map { $0.title })
// 누락됨: 최근 분석 결과들!
"""
```

사용자는 자신의 독서 저널을 보고 있지만, 에이전트는 보지 못합니다. 이는 단절을 야기합니다.

### 해결 방법

```swift
// 시스템 프롬프트에 UI가 보여주는 내용을 포함함
let systemPrompt = """
## 가용한 책 목록
\(books.map { "- \($0.title)" }.joined(separator: "\n"))

## 최근 독서 저널
\(analysisRecords.prefix(10).map { "- \($0.summary)" }.joined(separator: "\n"))
"""
```

### 컨텍스트 패리티 체크리스트

앱의 각 화면에 대해:
- [ ] 이 화면은 어떤 데이터를 표시하는가?
- [ ] 그 데이터를 에이전트가 사용할 수 있는가?
- [ ] 에이전트가 동일한 상세 수준으로 접근할 수 있는가?
</context_parity>

<continuous_parity>
## 시간이 지나도 패리티 유지하기

### Git Hooks / CI 체크

```bash
#!/bin/bash
# pre-commit hook: 도구 없이 새로운 UI 액션이 추가되었는지 확인

# 새로운 SwiftUI Button/onTapGesture 추가 확인
NEW_ACTIONS=$(git diff --cached --name-only | xargs grep -l "Button\|onTapGesture")

if [ -n "$NEW_ACTIONS" ]; then
    echo "⚠️  새로운 UI 액션이 감지되었습니다. 상응하는 에이전트 도구를 추가하셨나요?"
    echo "파일: $NEW_ACTIONS"
    echo ""
    echo "체크리스트:"
    echo "  [ ] 새로운 액션에 대한 에이전트 도구가 존재함"
    echo "  [ ] 시스템 프롬프트에 새로운 기능이 문서화됨"
    echo "  [ ] 기능 지도가 업데이트됨"
fi
```

### 자동화된 패리티 테스트

```typescript
// parity.test.ts
describe('Action Parity', () => {
  const capabilityMap = loadCapabilityMap();

  for (const [action, toolName] of Object.entries(capabilityMap)) {
    if (toolName === 'N/A') continue;

    test(`${action}은(는) 에이전트 도구 ${toolName}을(를) 가짐`, () => {
      expect(agentTools.map(t => t.name)).toContain(toolName);
    });

    test(`${toolName}이(가) 시스템 프롬프트에 문서화됨`, () => {
      expect(systemPrompt).toContain(toolName);
    });
  }
});
```

### 정기 감사

주기적인 검토 일정을 잡으십시오:

```markdown
## 월간 패리티 감사

1. 이번 달에 머지된 모든 PR 검토
2. 각 PR에서 새로운 UI 액션 확인
3. 도구 커버리지 검증
4. 기능 지도 업데이트
5. 자연어 요청으로 테스트
```
</continuous_parity>

<examples>
## 실제 사례: 피드(Feed) 격차

**이전:** Every Reader에는 통찰(insight)이 나타나는 피드가 있었지만, 에이전트가 거기에 게시할 수 있는 도구가 없었습니다.

```
사용자: "내 독서 피드에 예카테리나 대제에 관한 글을 하나 써줘"
에이전트: "어떤 시스템을 말씀하시는 건가요? 명확히 말씀해 주시겠어요?"
```

**진단:**
- ✅ UI 액션: 사용자는 분석 보기에서 통찰을 게시할 수 있음
- ❌ 에이전트 도구: `publish_to_feed` 도구가 없음
- ❌ 시스템 프롬프트: "피드"나 게시 방법에 대한 언급이 없음
- ❌ 컨텍스트: 에이전트가 "피드"가 무엇인지 모름

**해결:**

```swift
// 1. 도구 추가
tool("publish_to_feed",
    "사용자의 독서 피드에 통찰 게시",
    {
        bookId: z.string().describe("책 ID"),
        content: z.string().describe("통찰 내용"),
        headline: z.string().describe("강렬한 헤드라인")
    },
    async ({ bookId, content, headline }) => {
        await feedService.publish({ bookId, content, headline });
        return { text: `독서 피드에 "${headline}"을(를) 게시했습니다` };
    }
);

// 2. 시스템 프롬프트 업데이트
"""
## 당신의 기능

- **피드에 게시**: `publish_to_feed`를 사용하여 피드 탭에 나타나는 통찰을 생성합니다.
  book_id, 내용, 그리고 강렬한 헤드라인을 포함하십시오.
"""

// 3. 컨텍스트 주입에 추가
"""
사용자가 "피드" 또는 "독서 피드"를 언급하면, 통찰이 나타나는 피드 탭을 의미합니다.
거기에 콘텐츠를 생성하려면 `publish_to_feed`를 사용하십시오.
"""
```

**이후:**
```
사용자: "내 독서 피드에 예카테리나 대제에 관한 글을 하나 써줘"
에이전트: [publish_to_feed를 사용하여 통찰 생성]
       "완료했습니다! 독서 피드에 '계몽 전제 군주'라는 제목으로 게시했습니다."
```
</examples>

<checklist>
## 액션 패리티 체크리스트

UI 변경이 포함된 모든 PR에 대해:
- [ ] 모든 새로운 UI 액션 나열 완료
- [ ] 각 액션에 상응하는 에이전트 도구 존재 확인
- [ ] 새로운 기능을 포함하도록 시스템 프롬프트 업데이트 완료
- [ ] 기능 지도에 추가 완료
- [ ] 자연어 요청으로 테스트 완료

정기 감사 시:
- [ ] 모든 화면 훑어보기 완료
- [ ] 가능한 모든 사용자 액션 나열 완료
- [ ] 각각에 대한 도구 커버리지 확인 완료
- [ ] 사용자 요청 가능성에 따라 격차 우선순위 지정 완료
- [ ] 높은 우선순위 격차에 대한 이슈 생성 완료
</checklist>
