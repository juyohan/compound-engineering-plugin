# Compound Engineering (복리 엔지니어링)

[![Build Status](https://github.com/EveryInc/compound-engineering-plugin/actions/workflows/ci.yml/badge.svg)](https://github.com/EveryInc/compound-engineering-plugin/actions/workflows/ci.yml)
[![npm](https://img.shields.io/npm/v/@every-env/compound-plugin)](https://www.npmjs.com/package/@every-env/compound-plugin)

엔지니어링의 매 순간이 이전보다 더 수월해지도록 돕는 AI 스킬과 에이전트 모음입니다.

## 철학 (Philosophy)

**"모든 엔지니어링 작업은 다음 작업을 더 어렵게 만드는 것이 아니라, 더 쉽게 만들어야 합니다."**

기존의 개발 방식은 시간이 흐를수록 기술 부채를 쌓아갑니다. 새로운 기능이 추가될 때마다 복잡도는 높아지고, 버그를 고칠 때마다 파편화된 지식들이 남겨져 나중에 누군가 이를 다시 파악하는 데 시간을 허비하게 됩니다. 결국 코드베이스는 방대해지고, 전체 맥락(Context)을 유지하기는 힘들어지며, 다음 변경 작업은 점점 더 느려집니다.

**복리 엔지니어링(Compound Engineering)**은 이 흐름을 완전히 뒤바꿉니다. 전체 작업의 80%를 기획과 리뷰에 투자하고, 나머지 20%를 실행에 집중하는 방식입니다.

- `/ce-brainstorm`과 `/ce-plan`으로 코드를 작성하기 전 완벽하게 설계를 마칩니다.
- `/ce-code-review`와 `/ce-doc-review`를 통해 문제를 조기에 발견하고 기술적 판단력을 높입니다.
- 습득한 지식은 `/ce-compound`로 구조화하여 팀의 자산으로 만듭니다.
- 높은 품질을 유지하여 미래의 변경 작업이 언제나 가볍게 느껴지도록 합니다.

이것은 단순히 형식을 갖추는 것이 아니라, **지속적인 레버리지(Leverage)**를 만드는 과정입니다. 날카로운 브레인스토밍은 계획을 정교하게 만들고, 정교한 계획은 실행 단위를 최소화합니다. 수준 높은 리뷰는 단편적인 버그가 아닌 반복되는 패턴을 잡아내며, 잘 정리된 '복리 노트'는 다음 에이전트나 동료가 같은 시행착오를 겪지 않게 해줍니다.

**더 알아보기**

- [전체 컴포넌트 레퍼런스](plugins/compound-engineering/README.md) - 모든 에이전트 및 스킬 목록
- [복리 엔지니어링: Every 팀이 에이전트로 코딩하는 방법 (영문)](https://every.to/chain-of-thought/compound-engineering-how-every-codes-with-agents)
- [복리 엔지니어링 탄생 비화 (영문)](https://every.to/source-code/my-ai-had-already-fixed-the-code-before-i-saw-it)

## 워크플로우 (Workflow)

모든 작업의 최상단에는 `/ce-strategy`가 있습니다. 제품의 핵심 문제, 접근 방식, 페르소나, 주요 지표를 `STRATEGY.md`라는 문서로 관리합니다. 아이디어 제안, 브레인스토밍, 계획 수립 단계에서 이 문서를 기반으로 하므로, 모든 기능이 제품의 전략적 방향성과 일치하게 됩니다.

핵심 루프는 다음과 같습니다: **요구사항 브레인스토밍 -> 구현 계획 수립 -> 계획대로 실행 -> 결과 리뷰 -> 학습 내용 기록(Compound) -> 더 강화된 맥락을 바탕으로 반복.**

본격적인 브레인스토밍 전, 아이디어를 자유롭게 제안하고 검증하고 싶다면 `/ce-ideate`를 사용하세요.

| 스킬 | 용도 |
|-------|---------|
| `/ce-strategy` | `STRATEGY.md` 관리. 제품의 핵심 전략과 목표를 정의하며 다른 모든 스킬의 기반이 됨 |
| `/ce-ideate` | (선택 사항) 아이디어 구상: 아이디어를 생성하고 비판적으로 평가하여 최적의 안을 선별 |
| `/ce-brainstorm` | 대화형 Q&A를 통해 요구사항을 구체화하고 기획 문서 작성 |
| `/ce-plan` | 구상한 기능을 상세한 구현 계획으로 전환 |
| `/ce-work` | 워크트리와 작업 추적을 통해 계획을 체계적으로 실행 |
| `/ce-debug` | 체계적인 원인 분석과 테스트 중심의 버그 수정 |
| `/ce-code-review` | 다중 에이전트를 활용한 고차원 코드 리뷰 |
| `/ce-compound` | 미래 작업을 돕기 위해 습득한 지식과 패턴을 문서화 |
| `/ce-product-pulse` | 제품 성능 및 사용자 경험 리포트 생성 (`docs/pulse-reports/` 저장) |

`/ce-product-pulse`는 분석 도구로, 실제 사용자 경험과 성능 지표를 리포트로 만듭니다. 이 리포트들은 타임라인으로 쌓여 다음 전략 수립과 기획에 강력한 데이터 근거가 됩니다.

사이클이 반복될수록 모든 과정이 '복리'로 쌓여갑니다.

## 퀵 예시 (Quick Example)

보통은 아이디어를 기획 문서로 만들고, 계획을 세운 뒤 `/ce-work`에 실행을 맡기는 것으로 시작합니다.

```text
/ce-brainstorm "백그라운드 작업 재시도 로직 안정성 강화"
/ce-plan docs/brainstorms/background-job-retry-safety-requirements.md
/ce-work
/ce-code-review
/ce-compound
```

특정 버그를 조사할 때:

```text
/ce-debug "결제 웹훅에서 발생하는 인보이스 중복 생성 문제 조사"
/ce-code-review
/ce-compound
```

## 시작하기 (Getting Started)

설치 후, 프로젝트에서 `/ce-setup`을 실행하세요. 환경을 점검하고 누락된 도구를 설치하며 프로젝트 설정을 도와줍니다.

현재 `compound-engineering` 플러그인은 37개의 스킬과 51개의 에이전트를 포함하고 있습니다. 전체 목록은 [컴포넌트 참조](plugins/compound-engineering/README.md)를 확인하세요.

---

## 설치 (Install)

### Claude Code

```text
/plugin marketplace add juyohan/compound-engineering-plugin
/plugin install compound-engineering
```

### Cursor

커서 에이전트 채팅의 플러그인 마켓플레이스에서 `compound-engineering`을 추가하거나 검색하여 설치하세요.

### Codex

마켓플레이스 등록, 에이전트 설치, TUI를 통한 플러그인 설치의 3단계가 필요합니다.

1. **Codex에 마켓플레이스 등록:**

   ```bash
   codex plugin marketplace add juyohan/compound-engineering-plugin
   ```

2. **Compound Engineering 에이전트 설치** (Codex의 플러그인 사양은 아직 커스텀 에이전트를 등록하지 않음):

   ```bash
   bunx @every-env/compound-plugin install compound-engineering --to codex
   ```

3. **Codex TUI를 통한 플러그인 설치:** `codex`를 실행하고 `/plugins`를 실행하여 **Compound Engineering** 마켓플레이스를 찾은 다음, **compound-engineering** 플러그인을 선택하고 **Install**을 선택하세요. 설치가 완료되면 Codex를 재시작하세요.

### GitHub Copilot

**VS Code Copilot Agent Plugins**의 경우:

1. VS Code 커맨드 팔레트에서 `Chat: Install Plugin from Source` 실행
2. 저장소로 `EveryInc/compound-engineering-plugin` 입력
3. VS Code가 이 저장소의 플러그인을 표시할 때 `compound-engineering` 선택

**Copilot CLI**의 경우:

Copilot CLI 내부에서:

```text
/plugin marketplace add EveryInc/compound-engineering-plugin
/plugin install compound-engineering@compound-engineering-plugin
```

### 기타 플랫폼 (OpenCode, Pi, Gemini, Kiro 등)

이 저장소는 Compound Engineering 플러그인을 다른 에이전트 플랫폼 형식으로 변환하는 Bun/TypeScript 설치 도구를 포함하고 있습니다.

```bash
bunx @every-env/compound-plugin install compound-engineering --to opencode
bunx @every-env/compound-plugin install compound-engineering --to pi
bunx @every-env/compound-plugin install compound-engineering --to gemini
bunx @every-env/compound-plugin install compound-engineering --to kiro
```

---

## 로컬 개발 (Local Development)

```bash
bun install
bun test
bun run release:validate
```

## 라이선스 (License)

[MIT](LICENSE)
