---
name: ce-update
description: |
  compound-engineering 플러그인이 최신 상태인지 확인하고, 최신이 아닌 경우 업데이트 명령을 추천합니다. 
  사용자가 "update compound engineering", "check compound engineering version", "ce update", 
  "is compound engineering up to date", "update ce plugin"이라고 말하거나, 
  오래된 플러그인 버전으로 인해 발생할 수 있는 문제를 보고할 때 사용합니다. 
  이 스킬은 Claude Code에서만 작동하며, 플러그인 하네스 캐시 레이아웃에 의존합니다.
disable-model-invocation: true
ce_platforms: [claude]
allowed-tools: Bash(bash *upstream-version.sh), Bash(bash *currently-loaded-version.sh), Bash(bash *marketplace-name.sh)
  - gem
---

# 플러그인 버전 확인

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


설치된 compound-engineering 플러그인 버전이 main 브랜치의 업스트림 `plugin.json`과 일치하는지 확인하고, 일치하지 않으면 업데이트 명령을 추천합니다. Claude Code 전용입니다.

업스트림 버전은 최신 GitHub 릴리스 태그가 아닌 `main` 브랜치의 `plugins/compound-engineering/.claude-plugin/plugin.json`에서 가져옵니다. 이는 마켓플레이스가 `main`의 HEAD에서 플러그인 콘텐츠를 설치하기 때문입니다. `main`이 마지막 태그보다 앞서 있는 경우(릴리스 사이의 일반적인 상태) 릴리스 태그와 비교하면 잘못된 긍정(false-positive)이 발생합니다.

## 1단계: 버전 조사

Bash 도구를 통해 다음 세 가지 스크립트를 병렬로 실행합니다. 각각 한 줄의 출력을 생성하며, 아래의 결정 로직을 위해 해당 값을 캡처합니다. `claude --plugin-dir` 로컬 개발 세션과 표준 마켓플레이스 캐시 설치 모두에서 경로가 올바르게 해석되도록 `${CLAUDE_SKILL_DIR}`을 사용합니다.

```bash
bash "${CLAUDE_SKILL_DIR}/scripts/upstream-version.sh"
bash "${CLAUDE_SKILL_DIR}/scripts/currently-loaded-version.sh"
bash "${CLAUDE_SKILL_DIR}/scripts/marketplace-name.sh"
```

`scripts/upstream-version.sh`는 `gh api`를 통해 `main` 브랜치의 `plugin.json`을 읽습니다. 버전 문자열을 출력하거나, `gh`를 사용할 수 없거나 속도 제한이 걸린 경우 `__CE_UPDATE_VERSION_FAILED__` 센티널을 출력합니다.

`scripts/currently-loaded-version.sh` 및 `scripts/marketplace-name.sh`는 `${CLAUDE_SKILL_DIR}`을 마켓플레이스 캐시 레이아웃 `~/.claude/plugins/cache/<marketplace>/compound-engineering/<version>/skills/ce-update`에 대해 파싱합니다. 버전 세그먼트 / 마켓플레이스 세그먼트를 출력하거나, 경로가 일치하지 않는 경우(로컬 개발 시 일반적임) `__CE_UPDATE_NOT_MARKETPLACE__` 센티널을 출력합니다.

## 2단계: 결정 로직 적용

### 실패 케이스 처리

`scripts/upstream-version.sh`가 `__CE_UPDATE_VERSION_FAILED__`를 출력한 경우: 사용자에게 업스트림 버전을 가져올 수 없음을 알리고(`gh`를 사용할 수 없거나 속도 제한이 걸렸을 수 있음) 중단합니다.

`scripts/currently-loaded-version.sh`가 `__CE_UPDATE_NOT_MARKETPLACE__`를 출력한 경우: 스킬이 표준 마켓플레이스 캐시 외부에서 로드된 것입니다. 이 경우 두 가지 상황이 동일하게 처리됩니다: `claude --plugin-dir` 로컬 개발 세션이거나, Claude Code가 아닌 플랫폼인 경우(이 스킬은 플러그인 하네스 캐시 레이아웃에 의존하므로 Claude Code 전용입니다). 사용자에게 다음과 같이 말합니다:

> "스킬이 `~/.claude/plugins/cache/` 외부의 마켓플레이스 캐시에서 로드되었습니다. 이는 로컬 개발을 위해 `claude --plugin-dir`을 사용할 때 정상적인 현상입니다. 이 세션에서는 아무런 조치도 취하지 않습니다. 설치된 마켓플레이스 플러그인(있는 경우)에는 영향을 주지 않습니다. 해당 캐시를 확인하려면 일반 Claude Code 세션(`--plugin-dir` 없이)에서 `/ce-update`를 실행하세요."

그 후 중단합니다.

### 버전 비교

**최신 상태** — `currently_loaded == upstream`:

> "compound-engineering **v{version}**이 설치되어 있으며 최신 상태입니다."

**이전 버전** — `currently_loaded != upstream`:

> "compound-engineering이 **v{currently_loaded}** 버전이지만 **v{upstream}** 버전을 사용할 수 있습니다.
>
> 업데이트 명령:
> ```
> claude plugin update compound-engineering@{marketplace_name}
> ```
> 그런 다음 Claude Code를 다시 시작하여 적용하세요."

`claude plugin update` 명령은 Claude Code와 함께 제공되며 설치된 플러그인을 최신 버전으로 업데이트합니다. 이는 이전의 수동 캐시 삭제/마켓플레이스 새로고침 우회 방법을 대체합니다. 마켓플레이스 이름은 하드코딩되지 않고 스킬 경로에서 파생됩니다. 이 플러그인은 여러 마켓플레이스 이름으로 배포되기 때문입니다(예: README에 따른 공용 설치용 `compound-engineering-plugin` 또는 내부/팀 마켓플레이스용 기타 이름).
