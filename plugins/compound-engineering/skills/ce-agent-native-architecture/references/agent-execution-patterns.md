<overview>
강력한 에이전트 루프(loop)를 구축하기 위한 에이전트 실행 패턴입니다. 에이전트가 완료 신호를 보내는 방법, 재개를 위해 부분적 진행 상황을 추적하는 방법, 적절한 모델 티어(tier)를 선택하는 방법, 그리고 컨텍스트 제한을 처리하는 방법을 다룹니다.
</overview>

<completion_signals>
## 완료 신호 (Completion Signals)

에이전트에게는 "작업이 끝났다"고 명시적으로 말할 수 있는 방법이 필요합니다.

### 안티 패턴: 휴리스틱 감지 (Heuristic Detection)

휴리스틱을 통해 완료를 감지하는 것은 취약합니다:

- 도구 호출 없이 반복되는 이터레이션(iteration)
- 예상되는 출력 파일의 존재 확인
- "진전 없음" 상태 추적
- 시간 기반 타임아웃

이러한 방식은 에지 케이스(edge case)에서 깨지며 예측 불가능한 동작을 만듭니다.

### 패턴: 명시적 완료 도구

다음과 같은 기능을 수행하는 `complete_task` 도구를 제공하십시오:
- 달성된 작업의 요약을 인자로 받음
- 루프를 중지하는 신호를 반환함
- 모든 에이전트 유형에서 동일하게 작동함

```typescript
tool("complete_task", {
  summary: z.string().describe("달성된 작업의 요약"),
  status: z.enum(["success", "partial", "blocked"]).optional(),
}, async ({ summary, status = "success" }) => {
  return {
    text: summary,
    shouldContinue: false,  // 핵심: 루프를 중단해야 함을 신호함
  };
});
```

### ToolResult 패턴

성공 여부와 계속 진행 여부를 분리하도록 도구 결과를 구조화하십시오:

```swift
struct ToolResult {
    let success: Bool           // 도구가 성공했는가?
    let output: String          // 무슨 일이 일어났는가?
    let shouldContinue: Bool    // 에이전트 루프가 계속되어야 하는가?
}

// 세 가지 흔한 사례:
extension ToolResult {
    static func success(_ output: String) -> ToolResult {
        // 도구 성공, 계속 진행
        ToolResult(success: true, output: output, shouldContinue: true)
    }

    static func error(_ message: String) -> ToolResult {
        // 도구 실패했으나 복구 가능, 에이전트가 다른 시도를 할 수 있음
        ToolResult(success: false, output: message, shouldContinue: true)
    }

    static func complete(_ summary: String) -> ToolResult {
        // 작업 완료, 루프 중단
        ToolResult(success: true, output: summary, shouldContinue: false)
    }
}
```

### 핵심 통찰

**이것은 성공/실패와는 다릅니다:**

- 도구가 **성공**하면서 **중단**(작업 완료) 신호를 보낼 수 있습니다.
- 도구가 **실패**하면서 **계속**(복구 가능한 오류, 다른 시도 필요) 신호를 보낼 수 있습니다.

```typescript
// 예시:
read_file("/missing.txt")
// → { success: false, output: "File not found", shouldContinue: true }
// 에이전트는 다른 파일을 시도하거나 명확히 해달라고 요청할 수 있음

complete_task("다운로드한 모든 파일을 폴더별로 정리했습니다")
// → { success: true, output: "...", shouldContinue: false }
// 에이전트 작업 완료

write_file("/output.md", content)
// → { success: true, output: "Wrote file", shouldContinue: true }
// 에이전트는 목표를 향해 계속 작업함
```

### 시스템 프롬프트 지침

에이전트에게 언제 완료해야 하는지 알려주십시오:

```markdown
## 작업 완료하기

사용자의 요청을 달성했을 때:
1. 작업을 검증하십시오 (생성한 파일 다시 읽기, 결과 확인 등)
2. 수행한 작업의 요약과 함께 `complete_task`를 호출하십시오
3. 목표가 달성된 후에는 작업을 계속하지 마십시오

진행이 불가능하여 막힌 경우:
- 상태를 "blocked"로 설정하여 `complete_task`를 호출하고 이유를 설명하십시오
- 동일한 시도를 반복하며 무한 루프에 빠지지 마십시오
```
</completion_signals>

<partial_completion>
## 부분적 완료 (Partial Completion)

여러 단계로 구성된 작업의 경우, 재개(resume) 기능을 위해 작업 수준에서 진행 상황을 추적하십시오.

### 작업 상태 추적

```swift
enum TaskStatus {
    case pending      // 아직 시작되지 않음
    case inProgress   // 현재 작업 중
    case completed    // 성공적으로 완료됨
    case failed       // 완료 실패 (이유 포함)
    case skipped      // 의도적으로 수행하지 않음
}

struct AgentTask {
    let id: String
    let description: String
    var status: TaskStatus
    var notes: String?  // 실패 이유, 수행된 내용 등
}

struct AgentSession {
    var tasks: [AgentTask]

    var isComplete: Bool {
        tasks.allSatisfy { $0.status == .completed || $0.status == .skipped }
    }

    var progress: (completed: Int, total: Int) {
        let done = tasks.filter { $0.status == .completed }.count
        return (done, tasks.count)
    }
}
```

### UI 진행 상태 표시

사용자에게 무슨 일이 일어나고 있는지 보여주십시오:

```
진행률: 5개 작업 중 3개 완료 (60%)
✅ [1] 소스 자료 찾기
✅ [2] 전체 텍스트 다운로드
✅ [3] 핵심 문구 추출
❌ [4] 요약 생성 - 오류: 컨텍스트 제한 초과
⏳ [5] 개요 작성 - 대기 중
```

### 부분적 완료 시나리오

**에이전트가 완료 전 최대 이터레이션 횟수에 도달한 경우:**
- 일부 작업은 완료되었고 일부는 대기 중입니다.
- 현재 상태와 함께 체크포인트(checkpoint)가 저장됩니다.
- 재개 시 처음부터가 아니라 중단된 지점부터 계속됩니다.

**에이전트가 하나의 작업에서 실패한 경우:**
- 해당 작업은 `.failed`로 표시되고 노트에 오류가 기록됩니다.
- 다른 작업들은 계속될 수 있습니다 (에이전트가 결정).
- 오케스트레이터(orchestrator)가 전체 세션을 자동으로 중단하지 않습니다.

**작업 도중 네트워크 오류 발생:**
- 현재 이터레이션에서 예외가 발생합니다.
- 세션은 `.failed`로 표시됩니다.
- 체크포인트는 해당 지점까지의 메시지들을 보존합니다.
- 체크포인트를 통해 재개가 가능합니다.

### 체크포인트 구조

```swift
struct AgentCheckpoint: Codable {
    let sessionId: String
    let agentType: String
    let messages: [Message]          // 전체 대화 기록
    let iterationCount: Int
    let tasks: [AgentTask]           // 작업 상태
    let customState: [String: Any]   // 에이전트 전용 상태
    let timestamp: Date

    var isValid: Bool {
        // 체크포인트 만료 (기본 1시간)
        Date().timeIntervalSince(timestamp) < 3600
    }
}
```

### 재개(Resume) 흐름

1. 앱 실행 시 유효한 체크포인트가 있는지 스캔합니다.
2. 사용자에게 표시: "완료되지 않은 세션이 있습니다. 재개하시겠습니까?"
3. 재개 시:
   - 메시지들을 대화에 복원합니다.
   - 작업 상태들을 복원합니다.
   - 에이전트 루프를 중단된 지점부터 다시 시작합니다.
4. 거절 시:
   - 체크포인트를 삭제합니다.
   - 사용자가 다시 시도하면 새로 시작합니다.
</partial_completion>

<model_tier_selection>
## 모델 티어 선택 (Model Tier Selection)

에이전트마다 서로 다른 지능 수준이 필요합니다. 결과를 달성할 수 있는 가장 저렴한 모델을 사용하십시오.

### 티어 가이드라인

| 에이전트 유형 | 권장 티어 | 이유 |
|------------|-----------------|-----------|
| 채팅/대화 | Balanced (Sonnet) | 빠른 응답, 우수한 추론 |
| 조사 | Balanced (Sonnet) | 도구 루프, 아주 복잡한 합성이 아닌 경우 |
| 콘텐츠 생성 | Balanced (Sonnet) | 창의적이지만 합성 중심이 아닌 경우 |
| 복잡한 분석 | Powerful (Opus) | 다중 문서 합성, 미묘한 판단 필요 |
| 프로필 생성 | Powerful (Opus) | 사진 분석, 복잡한 패턴 인식 |
| 빠른 질의 | Fast (Haiku) | 간단한 조회, 빠른 변환 |
| 단순 분류 | Fast (Haiku) | 대량의 작업, 단순한 결정 |

### 구현 예시

```swift
enum ModelTier {
    case fast      // claude-3-haiku: 빠르고 저렴하며 단순한 작업용
    case balanced  // claude-sonnet: 대부분의 작업에 적합한 균형
    case powerful  // claude-opus: 복잡한 추론 및 합성용

    var modelId: String {
        switch self {
        case .fast: return "claude-3-haiku-20240307"
        case .balanced: return "claude-sonnet-4-20250514"
        case .powerful: return "claude-opus-4-20250514"
        }
    }
}

struct AgentConfig {
    let name: String
    let modelTier: ModelTier
    let tools: [AgentTool]
    let systemPrompt: String
    let maxIterations: Int
}

// 예시
let researchConfig = AgentConfig(
    name: "research",
    modelTier: .balanced,
    tools: researchTools,
    systemPrompt: researchPrompt,
    maxIterations: 20
)

let quickLookupConfig = AgentConfig(
    name: "lookup",
    modelTier: .fast,
    tools: [readLibrary],
    systemPrompt: "사용자의 라이브러리에 대한 빠른 질문에 답합니다.",
    maxIterations: 3
)
```

### 비용 최적화 전략

1. **Balanced로 시작하고 품질이 부족한 경우에만 업그레이드하십시오.**
2. **각 턴이 단순한 도구 중심 루프에는 Fast 티어를 사용하십시오.**
3. **Powerful 티어는 합성 작업(여러 소스 비교 등)을 위해 아껴두십시오.**
4. **비용 제어를 위해 턴당 토큰 제한을 고려하십시오.**
5. **반복적인 호출을 피하기 위해 비싼 작업은 캐싱하십시오.**
</model_tier_selection>

<context_limits>
## 컨텍스트 제한 (Context Limits)

에이전트 세션은 무기한 연장될 수 있지만, 컨텍스트 윈도우(context window)는 그렇지 않습니다. 처음부터 제한된 컨텍스트를 고려하여 설계하십시오.

### 문제 상황

```
턴 1: 사용자가 질문함 → 500 토큰
턴 2: 에이전트가 파일 읽음 → 10,000 토큰
턴 3: 에이전트가 다른 파일 읽음 → 10,000 토큰
턴 4: 에이전트가 조사함 → 20,000 토큰
...
턴 10: 컨텍스트 윈도우 초과
```

### 설계 원칙

**1. 도구는 반복적인 정밀화(iterative refinement)를 지원해야 합니다.**

모 아니면 도 식이 아니라, 요약 → 상세 → 전체 순으로 설계하십시오:

```typescript
// 좋음: 반복적인 정밀화를 지원함
tool("read_file", {
  path: z.string(),
  preview: z.boolean().default(true),  // 기본적으로 처음 1000자만 반환
  full: z.boolean().default(false),    // 명시적으로 전체 내용 요청
}, ...);

tool("search_files", {
  query: z.string(),
  summaryOnly: z.boolean().default(true),  // 전체 파일이 아닌 매칭 결과만 반환
}, ...);
```

**2. 통합 도구를 제공하십시오.**

에이전트가 세션 중간에 학습 내용을 통합할 수 있는 방법을 제공하십시오:

```typescript
tool("summarize_and_continue", {
  keyPoints: z.array(z.string()),
  nextSteps: z.array(z.string()),
}, async ({ keyPoints, nextSteps }) => {
  // 요약을 저장하고 이전 메시지들을 잘라낼(truncate) 가능성 있음
  await saveSessionSummary({ keyPoints, nextSteps });
  return { text: "요약이 저장되었습니다. 다음에 집중하여 계속합니다: " + nextSteps.join(", ") };
});
```

**3. 잘라내기(truncation)를 고려하여 설계하십시오.**

오케스트레이터가 초기 메시지들을 잘라낼 수 있다고 가정하십시오. 중요한 컨텍스트는 다음과 같아야 합니다:
- 시스템 프롬프트 내에 존재 (항상 유지됨)
- 파일 내에 존재 (다시 읽을 수 있음)
- context.md에 요약되어 존재

### 구현 전략

```swift
class AgentOrchestrator {
    let maxContextTokens = 100_000
    let targetContextTokens = 80_000  // 여유 공간 확보

    func shouldTruncate() -> Bool {
        estimateTokens(messages) > targetContextTokens
    }

    func truncateIfNeeded() {
        if shouldTruncate() {
            // 시스템 프롬프트 + 최근 메시지들은 유지
            // 오래된 메시지들은 요약하거나 버림
            messages = [systemMessage] + summarizeOldMessages() + recentMessages
        }
    }
}
```

### 시스템 프롬프트 지침

```markdown
## 컨텍스트 관리

긴 작업의 경우, 주기적으로 학습한 내용을 통합하십시오:
1. 많은 정보를 수집했다면 핵심 포인트를 요약하십시오.
2. 중요한 발견 사항은 파일에 저장하십시오 (컨텍스트를 넘어서도 유지됨).
3. 대화가 길어지면 `summarize_and_continue`를 사용하십시오.

모든 것을 메모리에 담아두려 하지 말고, 파일에 기록하십시오.
```
</context_limits>

<orchestrator_pattern>
## 통합 에이전트 오케스트레이터 (Unified Agent Orchestrator)

하나의 실행 엔진으로 여러 에이전트 유형을 처리합니다. 모든 에이전트는 서로 다른 구성을 가진 동일한 오케스트레이터를 사용합니다.

```swift
class AgentOrchestrator {
    static let shared = AgentOrchestrator()

    func run(config: AgentConfig, userMessage: String) async -> AgentResult {
        var messages: [Message] = [
            .system(config.systemPrompt),
            .user(userMessage)
        ]

        var iteration = 0

        while iteration < config.maxIterations {
            // 에이전트 응답 받기
            let response = await claude.message(
                model: config.modelTier.modelId,
                messages: messages,
                tools: config.tools
            )

            messages.append(.assistant(response))

            // 도구 호출 처리
            for toolCall in response.toolCalls {
                let result = await executeToolCall(toolCall, config: config)
                messages.append(.toolResult(result))

                // 완료 신호 확인
                if !result.shouldContinue {
                    return AgentResult(
                        status: .completed,
                        output: result.output,
                        iterations: iteration + 1
                    )
                }
            }

            // 도구 호출 없음 = 에이전트가 응답 중이며 완료되었을 가능성 있음
            if response.toolCalls.isEmpty {
                // 완료되었거나 사용자의 응답을 기다리는 중일 수 있음
                break
            }

            iteration += 1
        }

        return AgentResult(
            status: iteration >= config.maxIterations ? .maxIterations : .responded,
            output: messages.last?.content ?? "",
            iterations: iteration
        )
    }
}
```

### 장점

- 모든 에이전트 유형에서 일관된 라이프사이클 관리
- 자동 체크포인트/재개 (모바일에 필수적)
- 도구 프로토콜 공유
- 새로운 에이전트 유형 추가 용이
- 중앙 집중식 오류 처리 및 로깅
</orchestrator_pattern>

<checklist>
## 에이전트 실행 체크리스트

### 완료 신호 (Completion Signals)
- [ ] `complete_task` 도구 제공 (명시적 완료)
- [ ] 휴리스틱 완료 감지 사용 안 함
- [ ] 도구 결과에 `shouldContinue` 플래그 포함
- [ ] 시스템 프롬프트에서 완료 시점 가이드 제공

### 부분적 완료 (Partial Completion)
- [ ] 상태(대기, 진행 중, 완료, 실패)와 함께 작업 추적
- [ ] 재개를 위한 체크포인트 저장
- [ ] 사용자에게 진행 상태 표시
- [ ] 재개 시 중단된 지점부터 계속 진행

### 모델 티어 (Model Tiers)
- [ ] 작업 복잡도에 따라 티어 선택
- [ ] 비용 최적화 고려
- [ ] 단순 작업에는 Fast 티어 사용
- [ ] 합성을 위해 Powerful 티어 아껴둠

### 컨텍스트 제한 (Context Limits)
- [ ] 도구가 반복적 정밀화 지원 (미리보기 vs 전체)
- [ ] 통합(consolidation) 메커니즘 제공
- [ ] 중요한 컨텍스트는 파일로 영구 저장
- [ ] 잘라내기(truncation) 전략 정의
</checklist>
