---
name: ce-report-bug
description: compound-engineering 플러그인의 버그를 보고합니다.
argument-hint: "[선택 사항: 버그에 대한 간략한 설명]"
disable-model-invocation: true
allowed-tools:
  - gem
---

# Compound Engineering 플러그인 버그 보고

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


compound-engineering 플러그인을 사용하는 동안 발생한 버그를 보고합니다. 이 스킬은 구조화된 정보를 수집하여 메인테이너를 위한 GitHub 이슈를 생성합니다.

## Step 1: 버그 정보 수집

플랫폼의 블로킹 질문 도구를 사용하여 사용자에게 다음 질문들을 하십시오: Claude Code의 `AskUserQuestion` (스키마가 로드되지 않은 경우 `select:AskUserQuestion`으로 `ToolSearch`를 먼저 호출), Codex의 `request_user_input`, Gemini의 `ask_user`, Pi의 `ask_user` (`pi-ask-user` 확장 기능 필요). 스키마 로드가 필요해서가 아니라, 하네스에 블로킹 도구가 없거나 호출 오류(예: Codex 편집 모드)가 발생하는 경우에만 채팅의 번호 매기기 옵션으로 대체하십시오. 질문을 무시하고 넘어가서는 안 됩니다:

**질문 1: 버그 카테고리**
- 어떤 유형의 문제를 겪고 계십니까?
- 옵션: 에이전트 작동 안 함, 명령어 작동 안 함, 스킬 작동 안 함, MCP 서버 문제, 설치 문제, 기타

**질문 2: 특정 컴포넌트**
- 어떤 컴포넌트가 영향을 받았습니까?
- 에이전트, 명령어, 스킬 또는 MCP 서버의 이름을 물어보십시오.

**질문 3: 발생한 현상 (실제 동작)**
- 질문: "해당 컴포넌트를 사용했을 때 어떤 일이 발생했나요?"
- 실제 동작에 대한 명확한 설명을 얻으십시오.

**질문 4: 예상 동작 (기대했던 동작)**
- 질문: "대신 어떤 결과가 발생할 것으로 기대하셨나요?"
- 예상되는 동작에 대한 명확한 설명을 얻으십시오.

**질문 5: 재현 단계**
- 질문: "버그가 발생하기 전에 어떤 단계를 수행하셨나요?"
- 재현 단계를 얻으십시오.

**질문 6: 오류 메시지**
- 질문: "오류 메시지를 보셨나요? 있다면 공유해 주세요."
- 모든 오류 출력을 캡처하십시오.

## Step 2: 환경 정보 수집

환경 세부 정보를 자동으로 수집합니다. 코딩 에이전트 플랫폼을 감지하고 사용 가능한 정보를 수집하십시오:

**OS 정보 (모든 플랫폼):**
```bash
uname -a
```

**플러그인 버전:** 플러그인 매니페스트 또는 설치된 플러그인 메타데이터를 읽습니다. 일반적인 위치:
- Claude Code: `~/.claude/plugins/installed_plugins.json`
- Codex: `.codex/plugins/` 또는 프로젝트 구성
- 기타 플랫폼: 플랫폼의 플러그인 레지스트리를 확인하십시오.

**에이전트 CLI 버전:** 플랫폼의 버전 명령어를 실행합니다:
- Claude Code: `claude --version`
- Codex: `codex --version`
- 기타 플랫폼: 적절한 CLI 버전 플래그를 사용하십시오.

이 중 실패하는 항목이 있으면 "unknown"으로 표시하고 계속 진행하십시오. 보고서 작성을 중단하지 마십시오.

## Step 3: 버그 보고서 형식 지정

다음과 같이 잘 구조화된 버그 보고서를 생성합니다:

```markdown
## Bug Description

**Component:** [Type] - [Name]
**Summary:** [인자 또는 수집된 정보에서 가져온 간략한 설명]

## Environment

- **Plugin Version:** [플러그인 매니페스트/레지스트리에서 수집]
- **Agent Platform:** [예: Claude Code, Codex, Copilot, Pi, Kilo]
- **Agent Version:** [CLI 버전 명령어에서 수집]
- **OS:** [uname에서 수집]

## What Happened

[실제 동작 설명]

## Expected Behavior

[예상 동작 설명]

## Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Step 3]

## Error Messages

[모든 오류 출력]

## Additional Context

[기타 관련 정보]

---
*Reported via `/ce-report-bug` skill*
```

## Step 4: GitHub 이슈 생성

GitHub CLI를 사용하여 이슈를 생성합니다:

```bash
gh issue create \
  --repo EveryInc/compound-engineering-plugin \
  --title "[compound-engineering] Bug: [Brief description]" \
  --body "[Step 3에서 생성된 보고서]" \
  --label "bug,compound-engineering"
```

**참고:** 레이블이 존재하지 않는 경우 레이블 없이 생성합니다:
```bash
gh issue create \
  --repo EveryInc/compound-engineering-plugin \
  --title "[compound-engineering] Bug: [Brief description]" \
  --body "[Formatted bug report]"
```

## Step 5: 제출 확인

이슈가 생성된 후:
1. 사용자에게 이슈 URL을 표시합니다.
2. 버그 보고에 대해 감사를 표합니다.
3. 메인테이너(Kieran Klaassen)에게 알림이 전송됨을 알립니다.

## 출력 형식

```
버그 보고서가 성공적으로 제출되었습니다!

이슈: https://github.com/EveryInc/compound-engineering-plugin/issues/[NUMBER]
제목: [compound-engineering] Bug: [description]

compound-engineering 플러그인 개선에 도움을 주셔서 감사합니다!
메인테이너가 보고서를 검토하고 가능한 한 빨리 답변을 드릴 것입니다.
```

## 오류 처리

- `gh` CLI가 설치되지 않았거나 인증되지 않은 경우: 사용자에게 먼저 설치/인증하도록 안내합니다.
- 이슈 생성이 실패한 경우: 사용자가 수동으로 이슈를 생성할 수 있도록 형식화된 보고서를 표시합니다.
- 필수 정보가 누락된 경우: 해당 필드에 대해 다시 질문하십시오.

## 개인정보 보호 공지

이 스킬은 다음을 수집하지 않습니다:
- 개인 정보
- API 키 또는 자격 증명
- 프로젝트의 비공개 코드
- 기본 OS 정보 이외의 파일 경로

보고서에는 버그에 대한 기술 정보만 포함됩니다.
