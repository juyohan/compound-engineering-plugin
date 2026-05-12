<overview>
순수 프리미티브(primitives)인 bash, 파일 조작, 기본 저장소에서 시작하십시오. 이는 아키텍처가 작동함을 증명하고 에이전트에게 실제로 무엇이 필요한지 드러내 줍니다. 패턴이 나타나면 의도적으로 도메인 특화 도구를 추가하십시오. 이 문서는 언제 어떻게 프리미티브에서 도메인 도구로 진화할지, 그리고 언제 최적화된 코드로 넘어갈지에 대해 다룹니다.
</overview>

<start_with_primitives>
## 순수 프리미티브에서 시작하기

모든 에이전트 네이티브 시스템은 가능한 한 가장 원자적인 도구들로 시작하십시오:

- `read_file` / `write_file` / `list_files`
- `bash` (그 외 모든 작업용)
- 기본 저장소 (`store_item` / `get_item`)
- HTTP 요청 (`fetch_url`)

**여기서 시작해야 하는 이유:**

1. **아키텍처 증명** - 프리미티브만으로 작동한다면 여러분의 프롬프트가 제 역할을 하고 있다는 증거입니다.
2. **실제 필요성 발견** - 어떤 도메인 개념이 중요한지 발견하게 될 것입니다.
3. **최대의 유연성** - 에이전트가 여러분이 예상한 것뿐만 아니라 무엇이든 할 수 있습니다.
4. **좋은 프롬프트 강제** - 도구 로직에 의존할 수 없으므로 프롬프트를 잘 작성하게 됩니다.

### 예시: 시작 단계의 프리미티브

```typescript
// 처음에는 이 정도면 충분합니다
const tools = [
  tool("read_file", { path: z.string() }, ...),
  tool("write_file", { path: z.string(), content: z.string() }, ...),
  tool("list_files", { path: z.string() }, ...),
  tool("bash", { command: z.string() }, ...),
];

// 프롬프트에서 도메인 로직을 처리합니다
const prompt = `
피드백을 처리할 때:
1. data/feedback.json에서 기존 피드백을 읽으십시오.
2. 당신의 중요도 평가(1-5)와 함께 새 피드백을 추가하십시오.
3. 업데이트된 파일을 다시 쓰십시오.
4. 중요도가 4 이상인 경우, data/alerts/ 폴더에 알림 파일을 생성하십시오.
`;
```
</start_with_primitives>

<when_to_add_domain_tools>
## 도메인 도구 추가 시점

패턴이 나타남에 따라 도메인 특화 도구를 추가하고 싶어질 것입니다. 이는 바람직한 현상이지만, 의도적으로 수행하십시오.

### 어휘 고정 (Vocabulary Anchoring)

**도메인 도구 추가 사유:** 에이전트가 도메인 개념을 이해해야 할 때.

`create_note` 도구는 "노트 디렉토리에 이 형식으로 파일을 작성해"라고 시키는 것보다 시스템에서 "노트"가 무엇인지 에이전트에게 훨씬 더 잘 가르쳐줍니다.

```typescript
// 도메인 도구 없는 경우 - 에이전트가 구조를 추론해야 함
await agent.chat("미팅에 대한 노트를 만들어줘");
// 에이전트: notes/에 쓸까요? documents/에 쓸까요? 형식은 어떻게 하죠?

// 도메인 도구 있는 경우 - 어휘가 고정됨
tool("create_note", {
  title: z.string(),
  content: z.string(),
  tags: z.array(z.string()).optional(),
}, async ({ title, content, tags }) => {
  // 도구가 구조를 강제하고, 에이전트는 "노트"라는 개념을 이해함
});
```

### 가드레일 (Guardrails)

**도메인 도구 추가 사유:** 에이전트의 판단에만 맡기지 않고 검증이나 제약이 필요한 작업인 경우.

```typescript
// publish_to_feed 도구는 형식 요구 사항이나 콘텐츠 정책을 강제할 수 있음
tool("publish_to_feed", {
  bookId: z.string(),
  content: z.string(),
  headline: z.string().max(100),  // 헤드라인 길이 제한 강제
}, async ({ bookId, content, headline }) => {
  // 콘텐츠가 지침을 준수하는지 검증
  if (containsProhibitedContent(content)) {
    return { text: "콘텐츠가 지침을 준수하지 않습니다", isError: true };
  }
  // 올바른 구조 강제
  await feedService.publish({ bookId, content, headline, publishedAt: new Date() });
});
```

### 효율성 (Efficiency)

**도메인 도구 추가 사유:** 일반적인 작업이 너무 많은 프리미티브 호출을 필요로 할 때.

```typescript
// 프리미티브 방식: 여러 번의 호출 발생
await agent.chat("책 상세 정보 가져와줘");
// 에이전트: library.json 읽기, 파싱, 책 찾기, full_text.txt 읽기, introduction.md 읽기...

// 도메인 도구: 일반적인 작업에 대해 한 번의 호출로 처리
tool("get_book_with_content", { bookId: z.string() }, async ({ bookId }) => {
  const book = await library.getBook(bookId);
  const fullText = await readFile(`Research/${bookId}/full_text.txt`);
  const intro = await readFile(`Research/${bookId}/introduction.md`);
  return { text: JSON.stringify({ book, fullText, intro }) };
});
```
</when_to_add_domain_tools>

<the_rule>
## 도메인 도구의 원칙

**도메인 도구는 사용자 관점에서 하나의 개념적 액션을 나타내야 합니다.**

기계적인 검증은 포함할 수 있지만, **무엇을 할지 또는 할지 말지에 대한 판단은 프롬프트의 영역입니다.**

### 틀린 사례: 판단을 도구에 포함함

```typescript
// 틀림 - analyze_and_publish 도구가 판단 로직을 포함하고 있음
tool("analyze_and_publish", async ({ input }) => {
  const analysis = analyzeContent(input);      // 도구가 어떻게 분석할지 결정
  const shouldPublish = analysis.score > 0.7;  // 도구가 게시 여부를 결정
  if (shouldPublish) {
    await publish(analysis.summary);            // 도구가 무엇을 게시할지 결정
  }
});
```

### 옳은 사례: 하나의 액션, 에이전트가 결정

```typescript
// 옳음 - 도구를 분리하고 에이전트가 결정하게 함
tool("analyze_content", { content: z.string() }, ...);  // 분석 결과 반환
tool("publish", { content: z.string() }, ...);          // 에이전트가 제공한 내용을 게시

// 프롬프트: "콘텐츠를 분석하십시오. 품질이 높다면 요약을 게시하십시오."
// 에이전트가 "품질이 높다"는 것이 무엇인지, 어떤 요약을 쓸지 결정합니다.
```

### 테스트 방법

질문해 보십시오: "여기서 누가 결정을 내리고 있는가?"

- 답변이 "도구 코드"라면 → 판단을 코드에 박아넣은 것이므로 리팩토링하십시오.
- 답변이 "프롬프트를 바탕으로 한 에이전트"라면 → 바람직합니다.
</the_rule>

<keep_primitives_available>
## 프리미티브를 가용 상태로 유지하십시오

**도메인 도구는 지름길이지, 게이트(gate)가 아닙니다.**

보안이나 데이터 무결성을 위해 접근을 제한해야 할 특별한 이유가 없다면, 에이전트는 에지 케이스(edge case)를 위해 여전히 하위 프리미티브를 사용할 수 있어야 합니다.

```typescript
// 일반적인 경우를 위한 도메인 도구
tool("create_note", { title, content }, ...);

// 하지만 에지 케이스를 위해 프리미티브도 여전히 가용함
tool("read_file", { path }, ...);
tool("write_file", { path, content }, ...);

// 에이전트는 평소에는 create_note를 쓰지만, 특이한 상황에서는:
// "커스텀 메타데이터와 함께 비표준 위치에 노트를 생성해줘"
// → 에이전트가 직접 write_file을 사용함
```

### 접근 제한(Gate) 시점

도메인 도구만을 유일한 방법으로 제한하는 것은 다음의 경우에 적절합니다:

- **보안:** 사용자 인증, 결제 처리
- **데이터 무결성:** 불변성을 유지해야 하는 작업
- **감사 요구 사항:** 특정 방식으로 로그가 남아야 하는 액션

**기본값은 개방입니다.** 접근을 제한할 때는 명확한 이유를 가지고 의식적으로 결정하십시오.
</keep_primitives_available>

<graduating_to_code>
## 코드로의 전환 (Graduating to Code)

일부 작업은 성능이나 안정성을 위해 에이전트 오케스트레이션 방식에서 최적화된 코드로 옮겨가야 할 수도 있습니다.

### 진행 과정

```
1단계: 에이전트가 루프 내에서 프리미티브를 사용함
       → 유연함, 개념 증명 가능
       → 느리고 잠재적으로 비용이 많이 듦

2단계: 일반적인 작업에 대해 도메인 도구 추가
       → 더 빠름, 여전히 에이전트가 오케스트레이션함
       → 에이전트가 여전히 사용 시점/여부를 결정함

3단계: 핫 패스(hot paths)에 대해 최적화된 코드로 구현
       → 빠름, 결정론적(deterministic)임
       → 에이전트가 트리거할 수 있지만 실행은 코드가 담당함
```

### 진행 예시

**1단계: 순수 프리미티브**
```markdown
프롬프트: "사용자가 요약을 요청하면 /notes의 모든 노트를 읽고 분석하여
        /summaries/{date}.md에 요약을 작성하십시오."

에이전트: read_file을 20번 호출하고 내용을 추론하여 요약 작성
소요 시간: 30초, 50k 토큰
```

**2단계: 도메인 도구**
```typescript
tool("get_all_notes", {}, async () => {
  const notes = await readAllNotesFromDirectory();
  return { text: JSON.stringify(notes) };
});

// 에이전트가 여전히 어떻게 요약할지 결정하지만, 정보 가져오기가 빨라짐
// 소요 시간: 10초, 30k 토큰
```

**3단계: 최적화된 코드**
```typescript
tool("generate_weekly_summary", {}, async () => {
  // 핫 패스를 위해 전체 작업을 코드로 처리
  const notes = await getNotes({ since: oneWeekAgo });
  const summary = await generateSummary(notes);  // 더 저렴한 모델 사용 가능
  await writeSummary(summary);
  return { text: "요약이 생성되었습니다" };
});

// 에이전트는 이를 트리거하기만 함
// 소요 시간: 2초, 5k 토큰
```

### 주의 사항

**작업이 코드로 전환되더라도 에이전트는 다음을 할 수 있어야 합니다:**

1. 최적화된 작업 자체를 트리거하기
2. 최적화된 경로가 처리하지 못하는 에지 케이스에 대해 프리미티브로 폴백(fallback)하기

전환은 효율성을 위한 것입니다. **패리티(Parity)는 여전히 유지됩니다.** 최적화했다고 해서 에이전트의 능력을 잃어서는 안 됩니다.
</graduating_to_code>

<decision_framework>
## 의사결정 프레임워크

### 도메인 도구를 추가해야 할까요?

| 질문 | 답변이 '예'라면 |
|----------|--------|
| 에이전트가 이 개념의 의미를 헷갈려 하나요? | 어휘 고정을 위해 추가하십시오 |
| 에이전트가 결정해서는 안 될 검증이 필요한 작업인가요? | 가드레일과 함께 추가하십시오 |
| 자주 발생하는 다단계 작업인가요? | 효율성을 위해 추가하십시오 |
| 동작을 변경할 때 코드 수정이 필요한가요? | 대신 프롬프트로 유지하십시오 |

### 코드로 전환해야 할까요?

| 질문 | 답변이 '예'라면 |
|----------|--------|
| 이 작업이 매우 빈번하게 호출되나요? | 전환을 고려하십시오 |
| 지연 시간(latency)이 매우 중요한가요? | 전환을 고려하십시오 |
| 토큰 비용이 문제가 되나요? | 전환을 고려하십시오 |
| 결정론적인 동작이 필요한가요? | 코드로 전환하십시오 |
| 작업에 복잡한 상태 관리가 필요한가요? | 코드로 전환하십시오 |

### 접근 제한(Gate)해야 할까요?

| 질문 | 답변이 '예'라면 |
|----------|--------|
| 보안 요구 사항이 있나요? | 적절하게 제한하십시오 |
| 데이터 무결성을 반드시 유지해야 하는 작업인가요? | 적절하게 제한하십시오 |
| 감사/컴플라이언스 요구 사항이 있나요? | 적절하게 제한하십시오 |
| 특별한 위험은 없지만 단지 "더 안전해" 보이나요? | 프리미티브를 가용 상태로 두십시오 |
</decision_framework>

<examples>
## 예시

### 피드백 처리의 진화

**1단계: 프리미티브 전용**
```typescript
도구: [read_file, write_file, bash]
프롬프트: "피드백을 data/feedback.json에 저장하십시오. 중요하다면 알리십시오."
// 에이전트가 JSON 구조, 중요도 기준, 알림 방법을 스스로 파악함
```

**2단계: 어휘를 위한 도메인 도구**
```typescript
도구: [
  store_feedback,          // 올바른 구조로 "피드백" 개념을 고정함
  send_notification,       // 올바른 채널로 "알림" 개념을 고정함
  read_file,               // 에지 케이스를 위해 여전히 가용함
  write_file,
]
프롬프트: "store_feedback을 사용하여 저장하십시오. 중요도가 4 이상이면 알리십시오."
// 에이전트가 여전히 중요도를 결정하지만, 어휘가 고정됨
```

**3단계: 전환된 핫 패스**
```typescript
도구: [
  process_feedback_batch,  // 대량 처리에 최적화됨
  store_feedback,          // 개별 항목용
  send_notification,
  read_file,
  write_file,
]
// 대량 처리는 코드이지만, 에이전트는 특수 상황에서 여전히 store_feedback을 쓸 수 있음
```

### 도메인 도구를 추가하지 말아야 할 때

**단지 "깔끔하게" 보이기 위해 도구를 추가하지 마십시오:**
```typescript
// 불필요함 - 에이전트가 프리미티브를 조합할 수 있음
tool("organize_files_by_date", ...)  // 그냥 move_file과 판단력을 사용하면 됨

// 불필요함 - 결정을 엉뚱한 곳에 둠
tool("decide_file_importance", ...)  // 이는 프롬프트의 영역임
```

**동작이 바뀔 가능성이 있다면 도구를 추가하지 마십시오:**
```typescript
// 나쁨 - 코드에 고정됨
tool("generate_standard_report", ...)  // 보고서 형식이 진화한다면?

// 나음 - 프롬프트로 유지
prompt: "X, Y, Z를 포함하는 보고서를 생성하십시오. 가독성을 고려하여 포맷하십시오."
// 프롬프트를 수정하여 형식을 조정할 수 있음
```
</examples>

<checklist>
## 체크리스트: 프리미티브에서 도메인 도구로

### 시작 단계
- [ ] 순수 프리미티브(read, write, list, bash)로 시작함
- [ ] 도구 로직이 아닌 프롬프트에 동작을 기술함
- [ ] 실제 사용 사례에서 패턴이 나타나도록 함

### 도메인 도구 추가
- [ ] 명확한 이유가 있음: 어휘 고정, 가드레일 또는 효율성
- [ ] 도구가 하나의 개념적 액션을 나타냄
- [ ] 판단 로직은 도구 코드가 아닌 프롬프트에 있음
- [ ] 도메인 도구와 함께 프리미티브도 가용 상태로 유지함

### 코드로의 전환
- [ ] 핫 패스(빈번함, 지연 시간 민감 또는 비쌈)를 식별함
- [ ] 최적화된 버전이 에이전트의 능력을 제한하지 않음
- [ ] 에지 케이스를 위한 프리미티브 폴백이 여전히 작동함

### 접근 제한 결정
- [ ] 각 제한에 대해 구체적인 이유가 있음 (보안, 무결성, 감사)
- [ ] 기본값은 오픈 액세스임
- [ ] 제한은 기본값이 아닌 의식적인 결정임
</checklist>
