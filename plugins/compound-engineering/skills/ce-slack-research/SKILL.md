---
name: ce-slack-research
description: "해석된 조직 맥락(현재 작업에 영향을 주는 결정, 제약 조건 및 논의 흐름)을 Slack에서 검색합니다. 단순한 메시지 목록이 아닌, 교차 분석 및 조사 가치 평가가 포함된 조사 요약본을 생성합니다. 계획 수립, 브레인스토밍 또는 조직 지식이 중요한 모든 작업 중에 Slack에서 맥락을 검색할 때 사용하세요. 트리거 구문: 'search slack for', 'what did we discuss about', 'slack context for', 'organizational context about', 'what does the team think about', 'any slack discussions on'. 요약 없이 개별 메시지 결과를 반환하는 slack:find-discussions와 다릅니다."
allowed-tools:
  - gem
---

# /ce-slack-research

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


Slack에서 조직의 맥락을 검색하고 해석된 조사 요약본을 받습니다.

## 사용법

```
/ce-slack-research [주제 또는 질문]
/ce-slack-research
```

## 예시

```
/ce-slack-research free trial
/ce-slack-research What did we say about free trial recently?
/ce-slack-research free trial in #proj-reverse-trial
/ce-slack-research onboarding flow after:2026-03-01
```

입력값은 키워드, 자연어 질문이 될 수 있으며, 채널 힌트(`in:#channel`) 및 날짜 필터(`after:YYYY-MM-DD`)와 같은 Slack 검색 수식어를 포함할 수 있습니다. 에이전트는 입력 형식이 무엇이든 주제를 추출하고 검색어를 구성합니다.

## 실행

인수가 제공되지 않으면 조사할 주제를 묻습니다. 플랫폼의 차단형 질문 도구를 사용하세요: Claude Code의 `AskUserQuestion` (스키마가 로드되지 않은 경우 먼저 `select:AskUserQuestion`으로 `ToolSearch` 호출), Codex의 `request_user_input`, Gemini의 `ask_user`, Pi의 `ask_user` (`pi-ask-user` 확장 기능 필요). 하네스에 차단 도구가 존재하지 않거나 호출 오류가 발생하는 경우(예: Codex 편집 모드)에만 일반 텍스트로 질문하세요. 스키마 로드가 필요하다는 이유로 텍스트를 사용해서는 안 됩니다. 절대로 질문을 건너뛰지 마세요.

사용자의 주제를 작업 프롬프트로 하여 `ce-slack-researcher`를 파견합니다. 사용자의 구성된 권한 설정이 적용되도록 `mode` 파라미터는 생략합니다.

이후 과정(Slack MCP 탐색, 검색 실행, 스레드 읽기 및 종합)은 에이전트가 처리합니다. 에이전트는 다음을 포함하는 요약본을 반환합니다:

- 사용자가 올바른 Slack 인스턴스를 검색했는지 확인할 수 있는 **워크스페이스 식별자**
- 근거가 포함된 **조사 가치 평가** (높음 / 보통 / 낮음 / 없음)
- **주제별로 정리된 결과** (출처 채널 및 날짜 포함)
- 결과 전반에 걸친 패턴을 보여주는 **교차 분석**

에이전트가 Slack을 사용할 수 없다고 보고하면(MCP가 연결되지 않았거나 인증이 만료됨) 해당 메시지를 사용자에게 전달합니다. 다른 조사 방법을 시도하지 마세요.
