<overview>
프롬프트 네이티브(prompt-native) 원칙에 따른 MCP 도구 설계 방법입니다. 도구는 결정을 포함하는 워크플로우(workflow)가 아니라, 능력을 부여하는 프리미티브(primitives)여야 합니다.

**핵심 원칙:** 사용자가 할 수 있는 모든 것은 에이전트도 할 수 있어야 합니다. 에이전트를 인위적으로 제한하지 마십시오 — 파워 유저가 가질 수 있는 것과 동일한 프리미티브를 에이전트에게 제공하십시오.
</overview>

<principle name="primitives-not-workflows">
## 도구는 워크플로우가 아닌 프리미티브입니다

**틀린 접근 방식:** 비즈니스 로직을 포함하는 도구
```typescript
tool("process_feedback", {
  feedback: z.string(),
  category: z.enum(["bug", "feature", "question"]),
  priority: z.enum(["low", "medium", "high"]),
}, async ({ feedback, category, priority }) => {
  // 도구가 처리 방식을 결정함
  const processed = categorize(feedback);
  const stored = await saveToDatabase(processed);
  const notification = await notify(priority);
  return { processed, stored, notification };
});
```

**옳은 접근 방식:** 모든 워크플로우를 가능하게 하는 프리미티브
```typescript
tool("store_item", {
  key: z.string(),
  value: z.any(),
}, async ({ key, value }) => {
  await db.set(key, value);
  return { text: `${key} 저장 완료` };
});

tool("send_message", {
  channel: z.string(),
  content: z.string(),
}, async ({ channel, content }) => {
  await messenger.send(channel, content);
  return { text: "전송 완료" };
});
```

에이전트는 시스템 프롬프트를 바탕으로 분류, 우선순위, 알림 시점 등을 직접 결정합니다.
</principle>

<principle name="descriptive-names">
## 도구는 서술적이고 프리미티브한 이름을 가져야 합니다

이름은 사용 사례가 아닌 기능을 설명해야 합니다:

| 틀린 이름 | 옳은 이름 |
|-------|-------|
| `process_user_feedback` | `store_item` |
| `create_feedback_summary` | `write_file` |
| `send_notification` | `send_message` |
| `deploy_to_production` | `git_push` |

프롬프트는 에이전트에게 프리미티브를 *언제* 사용할지 알려줍니다. 도구는 단지 *기능*을 제공할 뿐입니다.
</principle>

<principle name="simple-inputs">
## 입력은 단순해야 합니다

도구는 데이터를 받습니다. 결정을 받지 않습니다.

**틀림:** 도구가 결정을 입력으로 받음
```typescript
tool("format_content", {
  content: z.string(),
  format: z.enum(["markdown", "html", "json"]),
  style: z.enum(["formal", "casual", "technical"]),
}, ...)
```

**맞음:** 도구는 데이터를 받고, 에이전트가 형식을 결정함
```typescript
tool("write_file", {
  path: z.string(),
  content: z.string(),
}, ...)
// 에이전트가 HTML 콘텐츠로 index.html을 쓸지, JSON으로 data.json을 쓸지 결정함
```
</principle>

<principle name="rich-outputs">
## 출력은 풍부해야 합니다

에이전트가 검증하고 반복 작업을 할 수 있도록 충분한 정보를 반환하십시오.

**틀림:** 최소한의 출력
```typescript
async ({ key }) => {
  await db.delete(key);
  return { text: "삭제됨" };
}
```

**맞음:** 풍부한 출력
```typescript
async ({ key }) => {
  const existed = await db.has(key);
  if (!existed) {
    return { text: `${key} 키가 존재하지 않습니다` };
  }
  await db.delete(key);
  return { text: `${key} 삭제 완료. ${await db.count()}개의 항목이 남았습니다.` };
}
```
</principle>

<design_template>
## 도구 설계 템플릿

```typescript
import { createSdkMcpServer, tool } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

export const serverName = createSdkMcpServer({
  name: "server-name",
  version: "1.0.0",
  tools: [
    // 읽기(READ) 작업
    tool(
      "read_item",
      "키를 사용하여 항목 읽기",
      { key: z.string().describe("항목 키") },
      async ({ key }) => {
        const item = await storage.get(key);
        return {
          content: [{
            type: "text",
            text: item ? JSON.stringify(item, null, 2) : `찾을 수 없음: ${key}`,
          }],
          isError: !item,
        };
      }
    ),

    tool(
      "list_items",
      "모든 항목 나열 (필터링 선택 가능)",
      {
        prefix: z.string().optional().describe("키 접두사로 필터링"),
        limit: z.number().default(100).describe("최대 항목 수"),
      },
      async ({ prefix, limit }) => {
        const items = await storage.list({ prefix, limit });
        return {
          content: [{
            type: "text",
            text: `${items.length}개의 항목 발견:\n${items.map(i => i.key).join("\n")}`,
          }],
        };
      }
    ),

    // 쓰기(WRITE) 작업
    tool(
      "store_item",
      "항목 저장",
      {
        key: z.string().describe("항목 키"),
        value: z.any().describe("항목 데이터"),
      },
      async ({ key, value }) => {
        await storage.set(key, value);
        return {
          content: [{ type: "text", text: `${key} 저장 완료` }],
        };
      }
    ),

    tool(
      "delete_item",
      "항목 삭제",
      { key: z.string().describe("항목 키") },
      async ({ key }) => {
        const existed = await storage.delete(key);
        return {
          content: [{
            type: "text",
            text: existed ? `${key} 삭제 완료` : `${key} 항목이 존재하지 않음`,
          }],
        };
      }
    ),

    // 외부(EXTERNAL) 작업
    tool(
      "call_api",
      "HTTP 요청 수행",
      {
        url: z.string().url(),
        method: z.enum(["GET", "POST", "PUT", "DELETE"]).default("GET"),
        body: z.any().optional(),
      },
      async ({ url, method, body }) => {
        const response = await fetch(url, { method, body: JSON.stringify(body) });
        const text = await response.text();
        return {
          content: [{
            type: "text",
            text: `${response.status} ${response.statusText}\n\n${text}`,
          }],
          isError: !response.ok,
        };
      }
    ),
  ],
});
```
</design_template>

<example name="feedback-server">
## 예시: 피드백 저장 서버

이 서버는 피드백 저장을 위한 프리미티브를 제공합니다. 피드백을 어떻게 분류하거나 정리할지는 결정하지 않습니다 — 그것은 프롬프트를 통한 에이전트의 몫입니다.

```typescript
export const feedbackMcpServer = createSdkMcpServer({
  name: "feedback",
  version: "1.0.0",
  tools: [
    tool(
      "store_feedback",
      "피드백 항목 저장",
      {
        item: z.object({
          id: z.string(),
          author: z.string(),
          content: z.string(),
          importance: z.number().min(1).max(5),
          timestamp: z.string(),
          status: z.string().optional(),
          urls: z.array(z.string()).optional(),
          metadata: z.any().optional(),
        }).describe("피드백 항목"),
      },
      async ({ item }) => {
        await db.feedback.insert(item);
        return {
          content: [{
            type: "text",
            text: `${item.author}로부터의 피드백 ${item.id} 저장 완료`,
          }],
        };
      }
    ),

    tool(
      "list_feedback",
      "피드백 항목 나열",
      {
        limit: z.number().default(50),
        status: z.string().optional(),
      },
      async ({ limit, status }) => {
        const items = await db.feedback.list({ limit, status });
        return {
          content: [{
            type: "text",
            text: JSON.stringify(items, null, 2),
          }],
        };
      }
    ),

    tool(
      "update_feedback",
      "피드백 항목 업데이트",
      {
        id: z.string(),
        updates: z.object({
          status: z.string().optional(),
          importance: z.number().optional(),
          metadata: z.any().optional(),
        }),
      },
      async ({ id, updates }) => {
        await db.feedback.update(id, updates);
        return {
          content: [{ type: "text", text: `피드백 ${id} 업데이트 완료` }],
        };
      }
    ),
  ],
});
```

그 후 시스템 프롬프트는 에이전트에게 이러한 프리미티브를 사용하는 *방법*을 알려줍니다:

```markdown
## 피드백 처리 지침

누군가 피드백을 공유하면:
1. 작성자, 내용, 그리고 포함된 URL을 추출하십시오.
2. 실행 가능성에 따라 중요도를 1-5점으로 평가하십시오.
3. `feedback.store_feedback`을 사용하여 저장하십시오.
4. 중요도가 높다면(4-5점), 채널에 알리십시오.

중요도 평가는 당신의 판단을 사용하십시오.
```
</example>

<principle name="dynamic-capability-discovery">
## 동적 기능 발견 vs 정적 도구 매핑

**이 패턴은 에이전트가 외부 API에 대해 사용자와 동일한 완전한 접근 권한을 가지길 원하는 에이전트 네이티브 앱을 위한 것입니다.** 이는 "사용자가 할 수 있는 것은 무엇이든 에이전트도 할 수 있다"는 핵심 원칙을 따릅니다.

제한된 기능을 가진 에이전트를 만드는 중이라면 정적 도구 매핑이 의도적일 수 있습니다. 하지만 HealthKit, HomeKit, GraphQL 등과 같은 API를 통합하는 에이전트 네이티브 앱의 경우:

**정적 도구 매핑 (에이전트 네이티브에서의 안티 패턴):**
각 API 기능에 대해 개별 도구를 만듭니다. 항상 최신 상태가 아니며, 에이전트를 여러분이 예상한 범위 내로 제한합니다.

```typescript
// ❌ 정적 방식: 모든 API 타입에 대해 하드코딩된 도구가 필요함
tool("read_steps", async ({ startDate, endDate }) => {
  return healthKit.query(HKQuantityType.stepCount, startDate, endDate);
});

tool("read_heart_rate", async ({ startDate, endDate }) => {
  return healthKit.query(HKQuantityType.heartRate, startDate, endDate);
});

// HealthKit에 혈당 추적 기능이 추가되면... 코드 수정이 필요합니다.
```

**동적 기능 발견 (권장 방식):**
가용한 기능을 발견하는 메타 도구와 무엇이든 접근할 수 있는 범용 도구를 만듭니다.

```typescript
// ✅ 동적 방식: 에이전트가 기능을 스스로 발견하고 사용함

// 발견 도구 - 런타임에 가용한 기능을 반환함
tool("list_available_capabilities", async () => {
  const quantityTypes = await healthKit.availableQuantityTypes();
  const categoryTypes = await healthKit.availableCategoryTypes();

  return {
    text: `가용한 건강 지표:\n` +
          `수량 타입: ${quantityTypes.join(", ")}\n` +
          `카테고리 타입: ${categoryTypes.join(", ")}\n` +
          `\n이 중 어떤 타입이든 read_health_data와 함께 사용하십시오.`
  };
});

// 범용 접근 도구 - 타입은 문자열이며 API가 이를 검증함
tool("read_health_data", {
  dataType: z.string(),  // z.enum이 아님 - HealthKit이 검증하게 함
  startDate: z.string(),
  endDate: z.string(),
  aggregation: z.enum(["sum", "average", "samples"]).optional()
}, async ({ dataType, startDate, endDate, aggregation }) => {
  // HealthKit이 타입을 검증하고, 유효하지 않으면 도움이 되는 오류를 반환함
  const result = await healthKit.query(dataType, startDate, endDate, aggregation);
  return { text: JSON.stringify(result, null, 2) };
});
```

**각 접근 방식의 사용 시점:**

| 동적 방식 (에이전트 네이티브) | 정적 방식 (제한된 에이전트) |
|------------------------|---------------------------|
| 에이전트가 사용자와 동일하게 모든 것에 접근해야 함 | 의도적으로 에이전트의 범위를 제한함 |
| 엔드포인트가 많은 외부 API (HealthKit, GraphQL 등) | 고정된 작업이 있는 내부 도메인 |
| API가 코드와 독립적으로 진화함 | 밀접하게 결합된 도메인 로직 |
| 완전한 액션 패리티를 원함 | 엄격한 가드레일을 원함 |

**에이전트 네이티브의 기본값은 '동적'입니다.** 에이전트의 기능을 의도적으로 제한할 때만 '정적' 방식을 사용하십시오.

**완전한 동적 패턴 예시:**

```swift
// 1. 발견 도구: 내가 무엇에 접근할 수 있는가?
tool("list_health_types", "가용한 건강 데이터 타입 가져오기") { _ in
    let store = HKHealthStore()

    let quantityTypes = HKQuantityTypeIdentifier.allCases.map { $0.rawValue }
    let categoryTypes = HKCategoryTypeIdentifier.allCases.map { $0.rawValue }

    return ToolResult(text: """
        가용한 HealthKit 타입:

        ## 수량 타입 (숫자 값)
        \(quantityTypes.joined(separator: ", "))

        ## 카테고리 타입 (범주 데이터)
        \(categoryTypes.joined(separator: ", "))

        위의 타입들로 read_health_data 또는 write_health_data를 사용하십시오.
        목록에 없는 새로운 타입은 list_health_types를 통해 확인하십시오.
        """)
}

// 2. 범용 읽기: 이름으로 모든 타입에 접근
tool("read_health_data", "모든 건강 지표 읽기", {
    dataType: z.string().describe("list_health_types에서 얻은 타입 이름"),
    startDate: z.string(),
    endDate: z.string()
}) { request in
    // HealthKit이 타입 이름을 검증하게 함
    guard let type = HKQuantityTypeIdentifier(rawValue: request.dataType)
                     ?? HKCategoryTypeIdentifier(rawValue: request.dataType) else {
        return ToolResult(
            text: "알 수 없는 타입: \(request.dataType). list_health_types로 확인하십시오.",
            isError: true
        )
    }

    let samples = try await healthStore.querySamples(type: type, start: startDate, end: endDate)
    return ToolResult(text: samples.formatted())
}

// 3. 컨텍스트 주입: 시스템 프롬프트에서 에이전트에게 무엇이 가용한지 알려줌
func buildSystemPrompt() -> String {
    let availableTypes = healthService.getAuthorizedTypes()

    return """
    ## 가용한 건강 데이터

    당신은 다음 건강 지표들에 접근할 수 있습니다:
    \(availableTypes.map { "- \($0)" }.joined(separator: "\n"))

    위의 타입들로 `read_health_data`를 사용하십시오. 목록에 없는 새로운 타입은
    `list_health_types`를 사용하여 가용한 지표를 발견하십시오.
    """
}
```

**장점:**
- 에이전트가 코드를 배포한 후에 추가된 API 기능도 사용할 수 있음
- enum 정의가 아닌 실제 API가 검증자가 됨
- 도구 표면적이 작아짐 (N개의 도구 대신 2-3개의 도구)
- 에이전트가 질문을 통해 자연스럽게 기능을 발견함
- 자기 성찰(introspection) 기능이 있는 모든 API와 작동함 (HealthKit, GraphQL, OpenAPI 등)
</principle>

<principle name="crud-completeness">
## CRUD 완결성 (CRUD Completeness)

에이전트가 생성할 수 있는 모든 데이터 타입은 읽기, 업데이트, 삭제도 가능해야 합니다. 불완전한 CRUD는 액션 패리티를 깨뜨립니다.

**안티 패턴: 생성 전용 도구**
```typescript
// ❌ 생성은 가능하지만 수정이나 삭제는 불가능
tool("create_experiment", { hypothesis, variable, metric })
tool("write_journal_entry", { content, author, tags })
// 사용자: "그 실험 삭제해줘" → 에이전트: "그건 할 수 없습니다"
```

**올바른 방식: 각 엔티티에 대한 완전한 CRUD**
```typescript
// ✅ 완전한 CRUD
tool("create_experiment", { hypothesis, variable, metric })
tool("read_experiment", { id })
tool("update_experiment", { id, updates: { hypothesis?, status?, endDate? } })
tool("delete_experiment", { id })

tool("create_journal_entry", { content, author, tags })
tool("read_journal", { query?, dateRange?, author? })
tool("update_journal_entry", { id, content, tags? })
tool("delete_journal_entry", { id })
```

**CRUD 감사(Audit):**
앱의 각 엔티티 타입에 대해 다음을 확인하십시오:
- [ ] 생성(Create): 에이전트가 새 인스턴스를 만들 수 있는가?
- [ ] 읽기(Read): 에이전트가 인스턴스를 조회/검색/목록화할 수 있는가?
- [ ] 업데이트(Update): 에이전트가 기존 인스턴스를 수정할 수 있는가?
- [ ] 삭제(Delete): 에이전트가 인스턴스를 제거할 수 있는가?

어떤 작업이라도 누락되면 결국 사용자가 이를 요청할 것이고 에이전트는 실패하게 됩니다.
</principle>

<checklist>
## MCP 도구 설계 체크리스트

**기본 원칙:**
- [ ] 도구 이름이 사용 사례가 아닌 기능을 설명함
- [ ] 입력은 결정이 아닌 데이터임
- [ ] 출력이 풍부함 (에이전트가 검증할 수 있을 정도)
- [ ] CRUD 작업이 개별 도구로 분리됨 (거대 도구 하나가 아님)
- [ ] 도구 구현 내에 비즈니스 로직이 없음
- [ ] `isError`를 통해 오류 상태가 명확히 전달됨
- [ ] 설명(description)이 언제 사용할지가 아니라 무엇을 하는지 설명함

**동적 기능 발견 (에이전트 네이티브 앱용):**
- [ ] 에이전트가 완전한 접근 권한을 가져야 하는 외부 API에 대해 동적 발견 사용
- [ ] 각 API 표면에 대해 `list_*` 또는 `discover_*` 도구 포함
- [ ] API가 검증하는 경우 enum이 아닌 문자열 입력 사용
- [ ] 가용한 기능을 런타임에 시스템 프롬프트에 주입
- [ ] 에이전트 범위를 의도적으로 제한할 때만 정적 도구 매핑 사용

**CRUD 완결성:**
- [ ] 모든 엔티티가 생성, 읽기, 업데이트, 삭제 작업을 가짐
- [ ] 모든 UI 액션에 상응하는 에이전트 도구가 있음
- [ ] 테스트: "에이전트가 방금 한 일을 되돌릴 수 있는가?"
</checklist>
