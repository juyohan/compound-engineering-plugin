<overview>
기존 에이전트 코드를 프롬프트 네이티브(prompt-native) 원칙에 따라 리팩토링하는 방법입니다. 목표는 행동 로직을 코드에서 프롬프트로 옮기고, 복잡한 도구를 단순한 프리미티브(primitives)로 간소화하는 것입니다.
</overview>

<diagnosis>
## 비-프롬프트 네이티브 코드 진단

에이전트가 프롬프트 네이티브가 아니라는 신호들:

**워크플로우를 인코딩한 도구들:**
```typescript
// 위험 신호: 도구가 비즈니스 로직을 포함함
tool("process_feedback", async ({ message }) => {
  const category = categorize(message);        // 코드 내의 로직
  const priority = calculatePriority(message); // 코드 내의 로직
  await store(message, category, priority);    // 코드 내의 오케스트레이션
  if (priority > 3) await notify();            // 코드 내의 결정
});
```

**에이전트가 스스로 판단하는 대신 함수를 호출만 함:**
```typescript
// 위험 신호: 에이전트가 단순한 함수 호출기에 불과함
"들어오는 메시지를 처리하기 위해 process_feedback을 사용해"
// vs.
"피드백이 들어오면 중요도를 결정하고, 저장하고, 중요도가 높으면 알림을 보내"
```

**에이전트 능력에 대한 인위적인 제한:**
```typescript
// 위험 신호: 사용자가 할 수 있는 일을 도구가 막음
tool("read_file", async ({ path }) => {
  if (!ALLOWED_PATHS.includes(path)) {
    throw new Error("이 파일을 읽을 권한이 없습니다");
  }
  return readFile(path);
});
```

**'무엇(WHAT)'이 아닌 '어떻게(HOW)'를 지정하는 프롬프트:**
```markdown
// 위험 신호: 에이전트를 마이크로매니징함
요약을 생성할 때:
1. 정확히 3개의 불렛 포인트를 사용해
2. 각 포인트는 20단어 미만이어야 해
3. 서브 포인트는 em-dash로 형식을 맞춰
4. 각 포인트의 첫 단어는 굵게 표시해
```
</diagnosis>

<refactoring_workflow>
## 단계별 리팩토링

**1단계: 워크플로우 도구 식별**

모든 도구를 나열하고 다음 특성을 가진 도구들을 표시하십시오:
- 비즈니스 로직(분류, 계산, 결정)을 가짐
- 여러 작업을 오케스트레이션함
- 에이전트를 대신해 결정을 내림
- 콘텐츠에 기반한 조건부 로직(if/else)을 포함함

**2단계: 프리미티브 추출**

각 워크플로우 도구에서 기반이 되는 프리미티브를 식별하십시오:

| 워크플로우 도구 | 숨겨진 프리미티브 |
|---------------|-------------------|
| `process_feedback` | `store_item`, `send_message` |
| `generate_report` | `read_file`, `write_file` |
| `deploy_and_notify` | `git_push`, `send_message` |

**3단계: 행동 로직을 프롬프트로 이동**

워크플로우 도구에 있던 로직을 자연어로 표현하십시오:

```typescript
// 리팩토링 전 (코드):
async function processFeedback(message) {
  const priority = message.includes("crash") ? 5 :
                   message.includes("bug") ? 4 : 3;
  await store(message, priority);
  if (priority >= 4) await notify();
}
```

```markdown
// 리팩토링 후 (프롬프트):
## 피드백 처리

누군가 피드백을 공유하면:
1. 중요도를 1-5점으로 평가하십시오:
   - 5: 충돌(crash), 데이터 손실, 보안 문제
   - 4: 명확한 재현 단계가 포함된 버그 보고
   - 3: 일반적인 제안, 사소한 문제
2. store_item을 사용하여 저장하십시오.
3. 중요도가 4점 이상이면 팀에 알리십시오.

당신의 판단을 사용하십시오. 키워드보다 맥락이 중요합니다.
```

**4단계: 도구를 프리미티브로 단순화**

```typescript
// 전: 1개의 워크플로우 도구
tool("process_feedback", { message, category, priority }, ...복잡한 로직...)

// 후: 2개의 프리미티브 도구
tool("store_item", { key: z.string(), value: z.any() }, ...단순 저장...)
tool("send_message", { channel: z.string(), content: z.string() }, ...단순 전송...)
```

**5단계: 인위적인 제한 제거**

```typescript
// 전: 제한된 능력
tool("read_file", async ({ path }) => {
  if (!isAllowed(path)) throw new Error("Forbidden");
  return readFile(path);
});

// 후: 완전한 능력
tool("read_file", async ({ path }) => {
  return readFile(path);  // 에이전트는 무엇이든 읽을 수 있음
});
// 읽기(READ)에 인위적인 제한을 두는 대신, 쓰기(WRITE)에 승인 게이트를 사용하십시오.
```

**6단계: 절차가 아닌 결과로 테스트**

"올바른 함수를 호출하는가?"가 아니라 "원하는 결과를 달성하는가?"를 테스트하십시오.

```typescript
// 전: 절차 테스트
expect(mockProcessFeedback).toHaveBeenCalledWith(...)

// 후: 결과 테스트
// 피드백 전송 → 합리적인 중요도로 저장되었는지 확인
// 높은 중요도 피드백 전송 → 알림이 전송되었는지 확인
```
</refactoring_workflow>

<before_after>
## 전/후 예시

**예시 1: 피드백 처리**

리팩토링 전:
```typescript
tool("handle_feedback", async ({ message, author }) => {
  const category = detectCategory(message);
  const priority = calculatePriority(message, category);
  const feedbackId = await db.feedback.insert({
    id: generateId(),
    author,
    message,
    category,
    priority,
    timestamp: new Date().toISOString(),
  });

  if (priority >= 4) {
    await discord.send(ALERT_CHANNEL, `${author}로부터 높은 중요도의 피드백 도착`);
  }

  return { feedbackId, category, priority };
});
```

리팩토링 후:
```typescript
// 단순 저장 프리미티브
tool("store_feedback", async ({ item }) => {
  await db.feedback.insert(item);
  return { text: `피드백 ${item.id} 저장 완료` };
});

// 단순 메시지 전송 프리미티브
tool("send_message", async ({ channel, content }) => {
  await discord.send(channel, content);
  return { text: "전송 완료" };
});
```

시스템 프롬프트:
```markdown
## 피드백 처리

누군가 피드백을 공유하면:
1. 유니크한 ID를 생성하십시오.
2. 영향도와 긴급성에 기반하여 중요도를 1-5점으로 평가하십시오.
3. store_feedback을 사용하여 전체 항목을 저장하십시오.
4. 중요도가 4점 이상이면 팀 채널에 알림을 보내십시오.

중요도 가이드라인:
- 5: 치명적 (충돌, 데이터 손실, 보안)
- 4: 높음 (상세한 버그 보고, 차단 이슈)
- 3: 중간 (제안, 사소한 버그)
- 2: 낮음 (UI 개선, 엣지 케이스)
- 1: 최소 (주제에서 벗어남, 중복)
```

**예시 2: 보고서 생성**

리팩토링 전:
```typescript
tool("generate_weekly_report", async ({ startDate, endDate, format }) => {
  const data = await fetchMetrics(startDate, endDate);
  const summary = summarizeMetrics(data);
  const charts = generateCharts(data);

  if (format === "html") {
    return renderHtmlReport(summary, charts);
  } else if (format === "markdown") {
    return renderMarkdownReport(summary, charts);
  } else {
    return renderPdfReport(summary, charts);
  }
});
```

리팩토링 후:
```typescript
tool("query_metrics", async ({ start, end }) => {
  const data = await db.metrics.query({ start, end });
  return { text: JSON.stringify(data, null, 2) };
});

tool("write_file", async ({ path, content }) => {
  writeFileSync(path, content);
  return { text: `${path} 쓰기 완료` };
});
```

시스템 프롬프트:
```markdown
## 보고서 생성

보고서 생성 요청을 받으면:
1. query_metrics를 사용하여 관련 지표를 조회하십시오.
2. 데이터를 분석하고 주요 트렌드를 파악하십시오.
3. 명확하고 형식이 잘 갖춰진 보고서를 작성하십시오.
4. write_file을 사용하여 적절한 형식으로 저장하십시오.

형식과 구조에 대해서는 당신의 판단을 사용하십시오. 유용하게 만드십시오.
```
</before_after>

<common_challenges>
## 일반적인 리팩토링 과제

**"하지만 에이전트가 실수를 할 수도 있어요!"**

네, 그렇기에 반복(iterate)할 수 있습니다. 프롬프트를 수정하여 가이드를 추가하십시오:
```markdown
// 수정 전
중요도를 1-5점으로 평가하십시오.

// 수정 후 (에이전트가 너무 높게 평가할 경우)
중요도를 1-5점으로 평가하십시오. 보수적으로 평가하십시오. 대부분의 피드백은 2-3점입니다.
정말로 작업을 차단하거나 치명적인 문제에만 4-5점을 사용하십시오.
```

**"워크플로우가 너무 복잡해요!"**

복잡한 워크플로우도 프롬프트로 표현할 수 있습니다. 에이전트는 똑똑합니다.
```markdown
비디오 피드백을 처리할 때:
1. Loom, YouTube 또는 직접 링크인지 확인하십시오.
2. YouTube의 경우 URL을 비디오 분석 도구에 직접 전달하십시오.
3. 그 외의 경우 먼저 다운로드한 후 분석하십시오.
4. 타임스탬프가 찍힌 이슈들을 추출하십시오.
5. 이슈의 밀도와 심각도에 따라 점수를 매기십시오.
```

**"결정론적인(deterministic) 동작이 필요해요!"**

일부 작업은 코드에 유지하는 것이 좋습니다. 그것도 괜찮습니다. 프롬프트 네이티브가 '전부 아니면 전무'여야 하는 것은 아닙니다.

코드에 유지할 것:
- 보안 검증
- 속도 제한 (Rate limiting)
- 감사 로깅 (Audit logging)
- 정확한 형식 요구 사항

프롬프트로 옮길 것:
- 분류 결정
- 우선순위 판단
- 콘텐츠 생성
- 워크플로우 오케스트레이션

**"테스트는 어떻게 하나요?"**

절차가 아닌 결과를 테스트하십시오:
- "이 입력이 주어졌을 때, 에이전트가 올바른 결과를 달성하는가?"
- "저장된 피드백의 중요도 점수가 합리적인가?"
- "정말로 높은 중요도의 항목에 대해 알림이 전송되는가?"
</common_challenges>

<checklist>
## 리팩토링 체크리스트

진단:
- [ ] 비즈니스 로직을 가진 모든 도구 목록화
- [ ] 에이전트 능력에 대한 인위적인 제한 식별
- [ ] '어떻게(HOW)'를 마이크로매니징하는 프롬프트 발견

리팩토링:
- [ ] 워크플로우 도구에서 프리미티브 추출
- [ ] 비즈니스 로직을 시스템 프롬프트로 이동
- [ ] 인위적인 제한 제거
- [ ] 도구 입력을 결정이 아닌 데이터로 단순화

검증:
- [ ] 에이전트가 프리미티브를 사용하여 동일한 결과를 달성함
- [ ] 프롬프트 편집만으로 행동을 변경할 수 있음
- [ ] 새로운 도구 추가 없이 새로운 기능을 추가할 수 있음
</checklist>
