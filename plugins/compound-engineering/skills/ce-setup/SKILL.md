---
name: ce-setup
description: "compound-engineering 환경을 진단하고 구성합니다. CLI 종속성, 플러그인 버전 및 저장소 로컬 설정을 확인합니다. 누락된 도구에 대한 안내형 설치를 제공합니다. 도구 누락 문제를 해결하거나, 설정을 확인하거나, 온보딩 전에 사용하세요."
disable-model-invocation: true
allowed-tools:
  - gem
---

# Compound Engineering 설정

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


## 상호작용 방법

플랫폼의 차단형 질문 도구를 사용하여 사용자에게 아래의 각 질문을 하세요: Claude Code의 `AskUserQuestion` (스키마가 로드되지 않은 경우 먼저 `select:AskUserQuestion`으로 `ToolSearch` 호출), Codex의 `request_user_input`, Gemini의 `ask_user`, Pi의 `ask_user` (`pi-ask-user` 확장 기능 필요). 하네스에 차단 도구가 존재하지 않거나 호출 오류가 발생하는 경우(예: Codex 편집 모드)에만 채팅에서 번호가 매겨진 목록으로 각 질문을 제시하세요. 스키마 로드가 필요하다는 이유로 채팅을 사용해서는 안 됩니다. 절대로 자동으로 구성하거나 질문을 건너뛰지 마세요. multiSelect 질문의 경우 쉼표로 구분된 숫자(예: `1, 3`)를 허용합니다.

compound-engineering을 위한 대화형 설정 — 환경 상태를 진단하고, 더 이상 사용되지 않는 저장소 로컬 CE 구성을 정리하며, 필요한 도구 구성을 돕습니다. 리뷰 에이전트 선택은 `ce-code-review`에서 자동으로 처리되며, 프로젝트별 리뷰 가이드는 `CLAUDE.md` 또는 `AGENTS.md`에 포함됩니다.

## 1단계: 진단

### 1단계: 플러그인 버전 확인

플러그인 메타데이터 또는 매니페스트를 읽어 설치된 compound-engineering 플러그인 버전을 감지합니다. 이는 플랫폼에 따라 다르므로 사용 가능한 메커니즘(예: 플러그인 루트 또는 캐시 디렉토리에서 `plugin.json` 읽기)을 사용하세요. 버전을 확인할 수 없는 경우 이 단계를 건너뜁니다.

버전이 발견되면 `--version`을 통해 확인 스크립트에 전달합니다. 그렇지 않으면 플래그를 생략합니다.

### 2단계: 상태 확인 스크립트 실행

스크립트를 실행하기 전에 "Compound Engineering -- 환경을 확인하는 중..." 메시지를 표시합니다.

번들로 제공되는 확인 스크립트를 실행합니다. 수동으로 종속성을 확인하지 마세요. 스크립트가 모든 CLI 도구, 에이전트 스킬, 저장소 로컬 CE 파일 확인 및 `.gitignore` 안내를 한 번에 처리합니다.

```bash
bash scripts/check-health --version VERSION
```

1단계에서 버전을 확인할 수 없는 경우 버전 없이 실행합니다:

```bash
bash scripts/check-health
```

스크립트 참조: `scripts/check-health`

사용자에게 스크립트 출력을 표시합니다.

### 3단계: 결과 평가

**플러그인 루트 (사전 확인):** !`echo "${CLAUDE_PLUGIN_ROOT}"`

위의 행이 절대 경로(`/`로 시작하고 `${`를 포함하지 않음)로 확인되면 Claude Code 세션이며 `/ce-update`를 사용할 수 있습니다. 그 외의 경우(비어 있거나, `${CLAUDE_PLUGIN_ROOT}` 토큰 문자열이거나, `!` 사전 확인을 처리하지 않는 비 Claude 하네스에 의해 남겨진 `echo "${CLAUDE_PLUGIN_ROOT}"`와 같은 미확인 명령 문자열)는 Claude Code가 아니므로 출력에서 `/ce-update` 참조를 생략합니다.

진단 보고서 작성 후 다음 사항을 확인합니다:

- 누락된 CLI 도구가 있는지 (Tools 섹션에 노란색으로 표시됨)
- 누락된 에이전트 스킬이 있는지 (Skills 섹션에 노란색으로 표시됨)
- `compound-engineering.local.md`가 존재하며 정리가 필요한지
- `.compound-engineering/config.local.yaml`이 존재하지 않거나 `.gitignore`에 안전하게 포함되지 않았는지
- `.compound-engineering/config.local.example.yaml`이 누락되었거나 최신 버전이 아닌지

모든 것이 설치되어 있고, 저장소 로컬 정리가 필요하지 않으며, `.compound-engineering/config.local.yaml`이 이미 존재하고 `.gitignore`에 포함되어 있다면 도구 및 스킬 목록과 완료 메시지를 표시합니다. 스크립트 출력에서 도구 및 스킬 이름을 파싱하고 각 항목을 녹색 원과 함께 나열합니다. 스크립트 출력에 Skills 섹션이 없으면 Skills 줄을 생략합니다:

```
 ✅ Compound Engineering 설정 완료

    도구:  🟢 agent-browser  🟢 gh  🟢 jq  🟢 vhs  🟢 silicon  🟢 ffmpeg  🟢 ast-grep
    스킬:  🟢 ast-grep
    설정:  ✅

    언제든지 /ce-setup을 실행하여 다시 확인할 수 있습니다.
```

이것이 Claude Code 세션인 경우(위의 **플러그인 루트**가 비어 있지 않은 경로로 확인됨) 메시지에 다음을 추가합니다: "최신 플러그인 버전을 가져오려면 /ce-update를 실행하세요."

여기서 멈춥니다.

그렇지 않으면 2단계로 진행하여 문제를 해결합니다. 저장소 로컬 정리(4단계)를 먼저 처리한 다음, 설정 부트스트랩(5단계), 누락된 종속성(6단계) 순으로 처리합니다.

## 2단계: 수정

### 4단계: 저장소 로컬 CE 문제 해결

저장소 루트를 확인합니다 (`git rev-parse --show-toplevel`). 저장소 루트에 `compound-engineering.local.md`가 존재하는 경우, 리뷰 에이전트 선택이 자동화되었고 CE가 이제 남은 머신 로컬 상태에 대해 `.compound-engineering/config.local.yaml`을 사용하므로 해당 파일이 더 이상 사용되지 않음을 설명합니다. 지금 삭제할지 묻습니다. 삭제할 때는 저장소 루트 경로를 사용합니다.

### 5단계: 프로젝트 설정 부트스트랩

저장소 루트를 확인합니다 (`git rev-parse --show-toplevel`). 아래의 모든 경로는 현재 작업 디렉토리가 아닌 저장소 루트에 대한 상대 경로입니다.

**예시 파일 (항상 새로 고침):** 필요한 경우 디렉토리를 생성하고 `references/config-template.yaml`을 `<repo-root>/.compound-engineering/config.local.example.yaml`로 복사합니다. 이 파일은 저장소에 커밋되며, 팀원들이 사용 가능한 설정을 볼 수 있도록 항상 최신 템플릿으로 덮어씁니다.

**로컬 설정 (한 번 생성):** `.compound-engineering/config.local.yaml`이 존재하지 않는 경우 생성을 요청합니다:

```
이 프로젝트를 위한 로컬 설정 파일을 구성하시겠습니까?
여기에 Compound Engineering 기본 설정(사용할 도구 및 워크플로 동작 방식 등)을 저장합니다.
모든 항목은 주석 처리된 상태로 시작되며, 필요한 항목만 활성화하면 됩니다.

1. 예, 생성합니다 (권장)
2. 아니요
```

사용자가 승인하면 `references/config-template.yaml`을 `<repo-root>/.compound-engineering/config.local.yaml`로 복사합니다. `.compound-engineering/config.local.yaml`이 아직 `.gitignore`에 포함되어 있지 않다면 항목 추가를 제안합니다:

```text
.compound-engineering/*.local.yaml
```

로컬 설정이 이미 존재하는 경우 `.gitignore`에 안전하게 포함되어 있는지 확인합니다. 그렇지 않은 경우 위와 같이 `.gitignore` 항목 추가를 제안합니다.

### 6단계: 설치 제안

모든 항목이 미리 선택된 multiSelect 질문을 사용하여 누락된 도구 및 스킬을 제시합니다. 스크립트의 진단 출력에 있는 설치 명령과 URL을 사용합니다. 사용자가 각 항목이 어떤 런타임을 대상으로 하는지 알 수 있도록 `Tools:` 및 `Skills:`로 항목을 그룹화합니다. 항목이 모두 설치된 그룹은 생략합니다.

```
다음 항목이 누락되었습니다. 설치할 항목을 선택하세요:
(모든 항목이 미리 선택되어 있습니다)

도구:
  [x] agent-browser - 테스트 및 스크린샷을 위한 브라우저 자동화
  [x] gh - 이슈 및 PR을 위한 GitHub CLI
  [x] jq - JSON 프로세서
  [x] vhs (charmbracelet/vhs) - CLI 출력으로 GIF 생성
  [x] silicon (Aloxaf/silicon) - 코드 스크린샷 생성
  [x] ffmpeg - 기능 데모를 위한 비디오 처리
  [x] ast-grep - AST 패턴을 사용한 구조적 코드 검색

스킬:
  [x] ast-grep - ast-grep을 사용한 구조적 코드 검색을 위한 에이전트 스킬
```

실제로 누락된 항목만 표시합니다. 설치된 항목은 생략합니다.

### 7단계: 선택한 종속성 설치

선택한 각 종속성에 대해 순서대로 다음을 수행합니다:

1. **설치 명령 표시** (진단 출력 결과 사용) 및 승인 요청:

   ```
   agent-browser를 설치하시겠습니까?
   명령어: CI=true npm install -g agent-browser --no-audit --no-fund --loglevel=error && agent-browser install && npx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser -g -y

   1. 이 명령어 실행
   2. 건너뛰기 - 수동으로 설치하겠습니다
   ```

2. **승인된 경우:** 쉘 실행 도구를 사용하여 설치 명령을 실행합니다. 명령이 완료되면 설치를 확인합니다:
   - CLI 도구의 경우 종속성 확인 명령(예: `command -v agent-browser`)을 실행합니다.
   - 에이전트 스킬의 경우, `npx`가 사용 가능하면 `npx --yes skills list --global --json | jq -r '.[].name' | grep -qx <skill-name>`을 선호합니다. 그렇지 않으면 `~/.claude/skills/<skill-name>`, `~/.agents/skills/<skill-name>` 또는 `~/.codex/skills/<skill-name>`이 존재하는지(파일, 디렉토리 또는 심볼릭 링크) 확인합니다.

3. **확인에 성공하면:** 성공을 보고합니다.

4. **확인에 실패하거나 설치 오류가 발생하면:** 폴백으로 프로젝트 URL을 표시하고 다음 종속성으로 넘어갑니다.

### 8단계: 요약

간략한 요약을 표시합니다:

```
 ✅ Compound Engineering 설정 완료

    설치됨: agent-browser, gh, jq
    건너뜀: rtk

    언제든지 /ce-setup을 실행하여 다시 확인할 수 있습니다.
```

이것이 Claude Code 세션인 경우(3단계의 플랫폼 감지에 따라) 다음을 추가합니다: "최신 플러그인 버전을 가져오려면 /ce-update를 실행하세요."
