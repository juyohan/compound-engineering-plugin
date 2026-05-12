---
name: ce-polish-beta
description: "[BETA] 개발 서버를 시작하고, 브라우저에서 기능을 열어 함께 개선 사항을 반복 적용합니다."
disable-model-invocation: true
argument-hint: "[PR 번호, 브랜치 이름 또는 현재 브랜치의 경우 빈칸]"
allowed-tools:
  - gem
---

# 폴리시 (Polish)

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


개발 서버를 시작하고, 브라우저에서 기능을 열어 반복적으로 개선합니다. 기능을 직접 사용해 보며 어색한 부분을 지적하면 즉시 수정이 이루어집니다.

## Phase 0: 적절한 브랜치로 이동

1. PR 번호 또는 브랜치 이름이 제공되면 해당 브랜치를 체크아웃합니다 (기존 워크트리를 먼저 조사하십시오).
2. 빈칸인 경우 현재 브랜치를 사용합니다.
3. 현재 브랜치가 main/master가 아닌지 확인합니다.

## Phase 1: 개발 서버 시작

### 1.1 `.claude/launch.json` 확인

`bash scripts/read-launch-json.sh`를 실행합니다. 구성을 찾으면 그것을 사용하십시오. 사용자가 이미 프로젝트 시작 방법을 설정해 둔 것입니다.

### 1.2 자동 감지 (launch.json이 없는 경우)

`bash scripts/detect-project-type.sh`를 실행하여 프레임워크를 식별합니다.

유형별로 일치하는 레시피 참조를 찾아 시작 명령어와 포트 기본값을 결정합니다:

| 유형 | 레시피 |
|------|--------|
| `rails` | `references/dev-server-rails.md` |
| `next` | `references/dev-server-next.md` |
| `vite` | `references/dev-server-vite.md` |
| `nuxt` | `references/dev-server-nuxt.md` |
| `astro` | `references/dev-server-astro.md` |
| `remix` | `references/dev-server-remix.md` |
| `sveltekit` | `references/dev-server-sveltekit.md` |
| `procfile` | `references/dev-server-procfile.md` |
| `unknown` | 사용자에게 프로젝트 시작 방법을 질문하십시오 |

패키지 매니저가 필요한 프레임워크 유형의 경우, `bash scripts/resolve-package-manager.sh`를 실행하고 그 결과를 시작 명령어에 대입합니다.

`bash scripts/resolve-port.sh --type <type>`으로 포트를 결정합니다.

### 1.3 서버 시작

개발 서버를 백그라운드에서 시작하고, 출력을 임시 파일에 기록합니다. 최대 30초 동안 `http://localhost:<port>`를 조사합니다. 서버가 뜨지 않으면 로그의 마지막 20줄을 보여주고 사용자에게 조치를 요청합니다.

### 1.4 브라우저에서 열기

`references/ide-detection.md`에서 환경 변수 조사 테이블을 로드합니다. IDE의 메커니즘을 사용하여 브라우저를 엽니다 (Claude Code → `open`, Cursor → Cursor browser, VS Code → Simple Browser).

사용자에게 다음과 같이 안내합니다:
```
개발 서버가 http://localhost:<port> 에서 실행 중입니다.
기능을 살펴보고 개선할 점을 말씀해 주세요.
```

## Phase 2: 반복 (Iterate)

핵심 루프 단계입니다. 사용자가 기능을 살펴보며 개선할 점을 말하면 에이전트가 이를 수정합니다. 사용자가 만족할 때까지 반복합니다.

- 사용자가 수정할 내용을 설명하면 → 변경 사항을 적용하고, 개발 서버가 핫 리로드(hot-reload)됩니다.
- 사용자가 무언가 확인을 요청하면 → `agent-browser`를 사용하여 페이지를 스크린샷하거나 검사합니다.
- 사용자가 완료되었다고 말하면 → 수정 사항을 커밋하고 중단합니다.

체크리스트나 봉투(envelope) 절차 없이 대화로만 진행됩니다.

## 참조 (References)

참조 파일 (필요 시 로드):
- `references/launch-json-schema.md` — launch.json 스키마 + 프레임워크별 스텁
- `references/ide-detection.md` — 호스트 IDE 감지 및 브라우저 핸드오프
- `references/dev-server-detection.md` — 포트 결정 문서
- `references/dev-server-rails.md` — Rails 개발 서버 기본값
- `references/dev-server-next.md` — Next.js 개발 서버 기본값
- `references/dev-server-vite.md` — Vite 개발 서버 기본값
- `references/dev-server-nuxt.md` — Nuxt 개발 서버 기본값
- `references/dev-server-astro.md` — Astro 개발 서버 기본값
- `references/dev-server-remix.md` — Remix 개발 서버 기본값
- `references/dev-server-sveltekit.md` — SvelteKit 개발 서버 기본값
- `references/dev-server-procfile.md` — Procfile 기반 개발 서버 기본값

스크립트 (`bash scripts/<name>`으로 호출):
- `scripts/read-launch-json.sh` — launch.json 리더
- `scripts/detect-project-type.sh` — 프로젝트 유형 분류기
- `scripts/resolve-package-manager.sh` — lockfile 기반 패키지 매니저 결정기
- `scripts/resolve-port.sh` — 포트 결정 연쇄 스크립트
