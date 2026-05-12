---
name: ce-product-pulse
description: "사용자가 경험한 것과 제품이 어떻게 작동했는지(사용량, 품질, 에러, 조사할 가치가 있는 신호)에 대해 시간 범위별 펄스(pulse) 보고서를 생성합니다. 사용자가 'run a pulse', 'show me the pulse', 'how are we doing', 'weekly recap', 'launch-day check'라고 말하거나 '24h', '7d'와 같은 시간 범위를 전달할 때 사용합니다. .compound-engineering/config.local.yaml을 통해 설정하며 보고서를 docs/pulse-reports/에 저장합니다."
argument-hint: "[lookback window, 예: '24h', '7d', '1h'; 기본값 24h]"
allowed-tools:
  - gem

  - Read
  - Write
  - Glob
  - Grep
  - Bash
  - AskUserQuestion
---

# 제품 펄스 (Product Pulse)

`ce-product-pulse`는 주어진 시간 범위 동안 제품의 데이터 소스를 쿼리하고 사용량, 성능, 에러 및 후속 조치를 포함하는 압축된 단일 페이지 보고서를 생성합니다. 보고서는 `docs/pulse-reports/`에 저장되며 주요 포인트는 채팅에 표시됩니다.

이 스킬은 제품, 데이터베이스 또는 외부 시스템을 수정하지 않습니다. 유일한 쓰기 작업은 `.compound-engineering/config.local.yaml`(통합 CE 로컬 설정, gitignore됨, 머신 로컬)에 추가되는 펄스 설정과 보고서 파일(`docs/pulse-reports/...`)입니다. MCP 및 기타 데이터 소스 도구는 읽기 전용으로 호출됩니다. 도구가 쓰기 모드를 제공하더라도 사용하지 마십시오.

## 상호작용 방식 (Interaction Method)

플랫폼의 차단 질문 도구를 기본으로 사용합니다: Claude Code의 `AskUserQuestion` (스키마가 로드되지 않은 경우 `ToolSearch`를 `select:AskUserQuestion`으로 먼저 호출), Codex의 `request_user_input`, Gemini의 `ask_user`, Pi의 `ask_user` (`pi-ask-user` 확장 프로그램 필요). 도구에 차단 도구가 없거나 호출 에러가 발생하는 경우(예: Codex 편집 모드)에만 채팅에 번호가 매겨진 옵션으로 폴백합니다. 스키마 로드가 필요하다는 이유로 폴백하지 마십시오. 질문을 조용히 건너뛰지 마십시오.

한 번에 하나의 질문만 던집니다. 다중 선택은 첫 실행 설정 시에만 사용하십시오.

## 룩백 윈도우 (Lookback Window)

<lookback> #$ARGUMENTS </lookback>

## 다중 에이전트 협업 (Multi-Agent Collaboration)

사용자의 입력(`$ARGUMENTS`) 내에 `--add <ai-이름>` 형태의 플래그가 포함되어 있는지 확인하십시오. 
현재 지원되는 외부 AI 인터페이스는 `--add gemini` (또는 `--add gem`)입니다.

만약 해당 플래그가 감지되면, 작업을 단독으로 확정하지 말고 다음 절차를 따르십시오:
1. **의도 파악:** 플래그를 제외한 나머지 문자열을 실제 지시사항으로 간주합니다.
2. **초안 작성:** 본인(주 에이전트)의 지식과 코드베이스 컨텍스트를 바탕으로 작업의 초기 뼈대나 접근법을 생각합니다.
3. **MCP 협업 호출:** `gem` 도구를 호출하여 외부 Gemini 에이전트에게 조언이나 검토를 구합니다.
   - 호출 시 전달할 메시지 예시: "나는 현재 이 작업에 대한 초안을 세우고 있어. 내 초안은 [초안 요약]이야. 이 접근 방식의 기술적 타당성을 검토하고 누락된 에지 케이스나 더 나은 패턴을 조언해줄 수 있어?"
4. **결과 통합:** `gem` 도구가 반환한 피드백을 당신의 최종 결과물에 통합(Synthesis)합니다. 
5. **명시적 표시:** 최종 산출물의 상단 또는 설명 부분에 "이 결과물은 Gemini와의 협업을 통해 검토 및 보완되었습니다."라는 문구를 추가하십시오.

이 협업 절차를 염두에 두고 아래의 본래 스킬 워크플로우를 진행하십시오.


인자를 시간 범위로 해석합니다. 일반적인 형식:

- `24h`, `48h`, `72h` - 지난 시간 단위
- `7d`, `30d` - 지난 일 단위
- `1h` - 짧은 기간 (출시 당일 유용)

인자가 비어 있으면 설정의 `pulse_lookback_default`(Phase 0에서 확인됨)를 기본값으로 사용합니다. 이 역시 설정되어 있지 않으면 `24h`를 강제 기본값으로 사용합니다. 인자를 파싱할 수 없으면 사용자에게 설명을 요청하십시오.

윈도우의 상한선에 **15분 후행 버퍼**를 적용합니다. 많은 분석 및 트레이싱 도구에 수집 지연이 있으므로 '현재' 시점까지 바로 쿼리하면 가장 최근 이벤트가 누락될 수 있습니다. `24h` 윈도우의 경우 `[현재 - 24h - 15m, 현재 - 15m]` 범위를 쿼리합니다.

## 핵심 원칙

1. **창업자의 관점에서 읽으십시오.** 하드코딩된 임계값을 두지 마십시오. 기본적으로 어떤 것을 "나쁨" 또는 "좋음"으로 라벨링하지 마십시오. 숫자만 제시하고 독자가 판단하게 하십시오.
2. **단일 페이지.** 터미널 출력 기준 30~40줄을 목표로 합니다. 보고서가 길어지면 내용을 줄이십시오.
3. **저장된 보고서에 개인정보(PII) 포함 금지.** 디스크에 작성되는 보고서에 사용자 이메일, 계정 ID 또는 메시지 본문을 포함하지 마십시오.
4. **안전한 곳은 병렬로, 중요한 곳은 순차적으로.** 분석 및 트레이싱 쿼리는 병렬로 실행합니다. 데이터베이스 쿼리는 부하를 피하기 위해 순차적으로 실행합니다.
5. **저장된 보고서를 통한 기억.** 매 실행 시 `docs/pulse-reports/`에 기록하여 과거의 펄스를 타임라인으로 탐색할 수 있게 합니다.
6. **읽기 전용 데이터베이스 액세스만 허용.** 데이터베이스를 데이터 소스로 사용하는 경우 연결은 반드시 읽기 전용이어야 합니다. 인터뷰 과정에서 읽기-쓰기 권한을 수락하지 마십시오. 데이터베이스 액세스는 선택 사항입니다. 많은 제품이 분석 및 트레이싱만으로 펄스를 완료합니다.
7. **가능한 경우 전략 기반 시작.** `STRATEGY.md`가 존재하면 질문을 던지기 전에 이를 읽고 제품 이름과 주요 지표를 초기값으로 가져옵니다. 데이터 소스 설정의 목표는 해당 지표를 실제로 측정하는 데 필요한 연결을 구성하는 것입니다.

## 실행 흐름

### Phase 0: 설정 상태에 따른 라우팅

**Config (사전 확인):**
!`(top=$(git rev-parse --show-toplevel 2>/dev/null); [ -n "$top" ] && cat "$top/.compound-engineering/config.local.yaml" 2>/dev/null) || echo '__NO_CONFIG__'`

위 블록에 YAML 키-값 쌍이 포함되어 있으면 아래 "Config keys"에 나열된 `pulse_*` 키의 값을 추출합니다.
`__NO_CONFIG__`가 표시되면 파일이 존재하지 않는 것이므로 첫 실행으로 간주합니다.
확인되지 않은 명령 문자열이 표시되면 기본 파일 읽기 도구(예: Claude Code의 Read, Codex의 read_file)를 사용하여 레포지토리 루트에서 `.compound-engineering/config.local.yaml`을 읽습니다. 파일이 없으면 첫 실행으로 처리합니다.

**Config keys:**
- `pulse_product_name` -- 문자열, 보고서 제목에 사용됨. 라우팅에 필수: 설정되지 않은 경우 스킬이 구성되지 않은 것으로 간주함.
- `pulse_lookback_default` -- `1h`, `24h`, `7d`, `30d` 중 하나 (기본값: `24h`)
- `pulse_primary_event` -- 문자열, 참여(engagement) 이벤트 이름
- `pulse_value_event` -- 문자열, 가치 실현(value-realization) 이벤트 이름
- `pulse_completion_events` -- 0~3개의 이벤트 이름을 쉼표로 구분한 문자열
- `pulse_quality_scoring` -- `true` 또는 기본값 `false` (AI 제품 전용)
- `pulse_quality_dimension` -- `pulse_quality_scoring`이 true일 때 1~5점으로 점수가 매겨지는 문자열. 그 외에는 무시됨.
- `pulse_analytics_source` -- 분석 제공자 식별 문자열 (예: `posthog`, `mixpanel`, `custom`)
- `pulse_tracing_source` -- 트레이싱 제공자 식별 문자열 (예: `sentry`, `datadog`, `custom`)
- `pulse_payments_source` -- 결제 제공자 식별 문자열 (예: `stripe`, `custom`). 사용하지 않으면 생략.
- `pulse_db_enabled` -- `true` 또는 기본값 `false`. `true`인 경우 읽기 전용 DB 액세스가 펄스의 일부가 됨.
- `pulse_metric_sources` -- 전략 문서의 지표별 소스 오버라이드를 제공하는 `metric=source` 쌍의 쉼표 구분 문자열 (예: `retention_d7=posthog,nps=delighted`). 나열되지 않은 지표는 `pulse_analytics_source`로 폴백하며, 암시적 라우팅이 보이도록 `(default source)` 마커와 함께 렌더링됩니다.
- `pulse_pending_metrics` -- 계측 대기 중인 전략 문서 지표 이름의 쉼표 구분 문자열. 계측 전까지 각 펄스 보고서에 `no data`로 렌더링됨.
- `pulse_excluded_metrics` -- 펄스 보고서에서 의도적으로 제외된 전략 문서 지표 이름의 쉼표 구분 문자열. 지표는 `STRATEGY.md`에 유지되지만 보고서에는 표시되지 않음.

**라우팅:**

- **`pulse_product_name`이 미설정됨 (또는 설정 파일 없음)** -> 첫 실행. Phase 1(인터뷰) 진행 후 Phase 2로 이동.
- **`pulse_product_name`이 설정됨** -> Phase 2로 점프.

인자가 `setup`, `reconfigure`, 또는 `edit config`인 경우 설정 상태와 관계없이 Phase 1로 이동합니다.

### Phase 1: 첫 실행 인터뷰

#### 1.0 전략 문서에서 초기값 추출 (있는 경우)

질문을 던지기 전에 기본 파일 읽기 도구를 사용하여 `STRATEGY.md`를 읽습니다. 파일이 존재하면 다음을 추출합니다.

- YAML frontmatter의 `name` 키에서 제품 이름을 가져옵니다. frontmatter가 없으면 H1 제목에서 ` Strategy` 접미사를 제거하고 가져옵니다 (예: `# Spiral Strategy` -> `Spiral`).
- `## Key metrics` 섹션에서 한 줄에 하나씩 나열된 주요 지표 목록을 가져옵니다.

추출된 내용을 보여주며 인터뷰를 시작합니다. 전략 문서를 찾았음을 알리고, 이벤트/데이터 설정으로 이어질 제품 이름과 주요 지표 목록을 보여준 후 사용자가 수정할 사항이 있는지 묻습니다.

`STRATEGY.md`가 존재하지 않으면 채팅에 이를 명시합니다: 전략 문서가 없으므로 처음부터 설정을 진행하며, 나중에 `ce-strategy`를 먼저 실행하면 펄스에 초기값을 제공할 수 있다고 안내합니다.

#### 1.1 인터뷰

`references/interview.md`를 읽습니다. 이 파일의 로드는 필수입니다. 거절 규칙, 안티 패턴 사례 및 지표-소스 매핑 로직이 포함되어 있습니다.

인터뷰를 다음 순서로 진행합니다.

1. 제품 이름 (추출된 값 확인 또는 수정)
2. 주요 참여 이벤트
3. 가치 실현 이벤트
4. 완료 또는 전환 (0~3개)
5. 품질 스코어링 (AI 제품인 경우 선택 사항)
6. 데이터 소스 - 합의된 각 지표와 이벤트에 대한 연결 구성. MCP 사용을 유도합니다. 읽기-쓰기 데이터베이스 액세스는 거부합니다. DB는 완전히 선택 사항입니다.
7. 시스템 성능 - 상위 에러 및 대기 시간에 대해 권장되는 간단한 설정. 사용자는 보통 여기서 강한 의견이 없으므로 기본값을 제시하십시오.

인터뷰가 완료되면 설정을 파일에 저장합니다.

#### 1.2 설정 저장

인터뷰 결과를 `.compound-engineering/config.local.yaml`에 추가합니다. 파일이 없으면 생성합니다. 이 파일은 git에 의해 무시되어야 합니다.

```bash
cat >> .compound-engineering/config.local.yaml <<EOF
pulse_product_name: "Spiral"
pulse_primary_event: "session_start"
pulse_value_event: "content_generated"
pulse_completion_events: "checkout_success,feedback_positive"
pulse_quality_scoring: true
pulse_quality_dimension: "helpfulness"
pulse_analytics_source: "posthog"
pulse_tracing_source: "sentry"
pulse_metric_sources: "nps=delighted"
pulse_excluded_metrics: "user_satisfaction"
EOF
```

### Phase 2: 보고서 생성

#### 2.1 데이터 소스 쿼리

Phase 0 또는 Phase 1에서 로드된 설정을 바탕으로 데이터 소스에 대해 필요한 모든 쿼리를 실행합니다.

윈도우 `[now - <lookback> - 15m, now - 15m]`를 사용합니다.

MCP 도구 및 기타 데이터 소스 커넥터를 호출합니다.

#### 2.2 보고서 작성

`docs/pulse-reports/YYYY-MM-DD-HHMMSS-pulse.md` 위치에 보고서를 작성합니다.

보고서 템플릿:

```markdown
# [Product Name] Pulse: [YYYY-MM-DD] ([Window])

## Engagement
- [Primary Event]: [Count]
- [Value Event]: [Count]
- Conversion Rate: [Value Event / Primary Event %]

## Key Metrics (from STRATEGY.md)
- [Metric 1]: [Value] ([Source])
- [Metric 2]: [Value] ([Source])
...

## Success & Completions
- [Completion Event 1]: [Count]
- [Completion Event 2]: [Count]

## Quality (AI only)
- [Quality Dimension]: [Avg Score] (1-5)

## System Performance
- Top Errors: [List 1-3]
- P95 Latency: [Value]

## Observations & Signals
- [Observation 1]
- [Observation 2]
```

#### 2.3 요약 표시

작성된 보고서의 요약 버전을 채팅에 표시합니다. 보고서의 전체 경로를 포함하여 사용자가 원본을 찾을 수 있게 합니다.

보고서 작성이 완료되면 스킬을 종료합니다.
