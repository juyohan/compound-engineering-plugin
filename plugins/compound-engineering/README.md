# 컴파운딩 엔지니어링 플러그인 (Compounding Engineering Plugin)

사용할수록 더 똑똑해지는 AI 기반 개발 도구입니다. 매번의 엔지니어링 작업 단위를 이전보다 더 쉽게 만들어 줍니다.

## 시작하기

설치 후, 프로젝트에서 `/ce-setup`을 실행하세요. 이 명령은 대화형 흐름을 통해 사용자 환경을 진단하고, 누락된 도구를 설치하며, 프로젝트 설정을 초기화합니다.

## 구성 요소

| 구성 요소 | 개수 |
|-----------|-------|
| 에이전트 (Agents) | 50+ |
| 스킬 (Skills) | 38+ |

## 스킬 (Skills)

엔지니어링 작업을 위한 주요 진입점으로, 슬래시 명령어(/)로 호출됩니다. 많은 스킬에 대한 상세한 사용자 가이드는 [`docs/skills/`](../../docs/skills/)에 보관되어 있습니다. 아래 나열된 각 스킬 이름은 해당 문서 페이지(목적, 새로운 메커니즘, 사용 사례, 체인 위치 등)로 연결됩니다. 전용 문서가 없는 스킬들도 아래에 나열되어 있으며, 소스 트리의 `SKILL.md` 파일이 공식 지침입니다.

### 핵심 워크플로우 (Core Workflow)

`ce-strategy`가 상위 단계에서 루프를 고정하고, `ce-product-pulse`가 사용자 결과에 대한 리딩으로 루프를 닫습니다.

| 스킬 | 설명 |
|-------|-------------|
| [`/ce-strategy`](../../docs/skills/ce-strategy.md) | `STRATEGY.md` 파일을 생성하거나 유지 관리합니다. 제품의 목표 문제, 접근 방식, 페르소나, 핵심 지표 및 트랙을 정의합니다. 업데이트를 위해 재실행 가능하며, `/ce-ideate`, `/ce-brainstorm`, `/ce-plan` 실행 시 참고 자료로 사용됩니다. |
| [`/ce-ideate`](../../docs/skills/ce-ideate.md) | 선택 사항인 거시적 아이데이션: 근거 있는 아이디어를 생성하고 비판적으로 평가한 뒤, 가장 강력한 아이디어를 브레인스토밍 단계로 전달합니다. |
| [`/ce-brainstorm`](../../docs/skills/ce-brainstorm.md) | 계획을 세우기 전, 대화형 Q&A를 통해 기능이나 문제를 깊이 있게 생각하고 적절한 크기의 요구사항 문서를 작성합니다. |
| [`/ce-plan`](../../docs/skills/ce-plan.md) | 소프트웨어 기능, 리서치 워크플로우, 이벤트, 학습 계획 등 모든 다단계 작업을 위한 구조화된 계획을 수립하며, 자동으로 신뢰도를 체크합니다. |
| [`/ce-code-review`](../../docs/skills/ce-code-review.md) | 계층화된 페르소나 에이전트, 신뢰도 게이팅, 중복 제거 파이프라인을 갖춘 구조화된 코드 리뷰를 수행합니다. |
| [`/ce-work`](../../docs/skills/ce-work.md) | 작업 항목을 체계적으로 실행합니다. |
| [`/ce-debug`](../../docs/skills/ce-debug.md) | 체계적으로 근본 원인을 찾고 버그를 수정합니다. 인과 관계를 추적하고, 테스트 가능한 가설을 세우며, 테스트 우선(test-first) 방식으로 수정을 구현합니다. |
| [`/ce-compound`](../../docs/skills/ce-compound.md) | 해결된 문제들을 문서화하여 팀의 지식을 축적(Compounding)합니다. |
| [`/ce-compound-refresh`](../../docs/skills/ce-compound-refresh.md) | 오래되거나 변질된 지식들을 최신화하고 유지, 업데이트, 교체 또는 아카이브할지 결정합니다. |
| [`/ce-optimize`](../../docs/skills/ce-optimize.md) | 병렬 실험, 측정 게이트, LLM 기반 품질 평가를 통해 반복적인 최적화 루프를 실행합니다. |
| [`/ce-product-pulse`](../../docs/skills/ce-product-pulse.md) | 사용량, 성능, 오류 및 후속 조치에 대한 단일 페이지의 시간대별 보고서를 생성합니다. 보고서를 `docs/pulse-reports/`에 저장하여 사용자가 경험한 내용을 타임라인으로 탐색할 수 있게 합니다. |

### 리서치 및 컨텍스트 (Research & Context)

| 스킬 | 설명 |
|-------|-------------|
| [`/ce-sessions`](../../docs/skills/ce-sessions.md) | Claude Code, Codex, Cursor를 아우르는 세션 이력에 대해 질문합니다. |
| [`/ce-slack-research`](../../docs/skills/ce-slack-research.md) | Slack을 검색하여 조직의 컨텍스트(결정 사항, 제약 조건, 논의 과정 등)를 해석합니다. |
| [`ce-riffrec-feedback-analysis`](../../docs/skills/ce-riffrec-feedback-analysis.md) | [Riffrec](https://github.com/kieranklaassen/riffrec) 녹화물, 비디오, 오디오 또는 메모를 구조화된 피드백으로 변환합니다. 설정, 빠른 버그 보고, `ce-brainstorm`으로 연결되는 광범위한 분석 사이를 연결합니다. |

### Git 워크플로우 (Git Workflow)

| 스킬 | 설명 |
|-------|-------------|
| [`ce-clean-gone-branches`](../../docs/skills/ce-clean-gone-branches.md) | 원격 브랜치가 삭제된 로컬 브랜치들을 정리합니다. |
| [`ce-commit`](../../docs/skills/ce-commit.md) | 가치를 전달하는 메시지와 함께 git 커밋을 생성합니다. |
| [`ce-commit-push-pr`](../../docs/skills/ce-commit-push-pr.md) | 커밋, 푸시 후 적응형 설명을 포함한 PR을 생성합니다. 기존 PR 설명을 업데이트하거나, 커밋 없이 설명만 생성할 수도 있습니다. |
| [`ce-worktree`](../../docs/skills/ce-worktree.md) | 병렬 개발을 위해 Git worktree를 관리합니다. |

### 워크플로우 유틸리티 (Workflow Utilities)

| 스킬 | 설명 |
|-------|-------------|
| [`/ce-demo-reel`](../../docs/skills/ce-demo-reel.md) | PR을 위한 시각적 데모 릴(GIF 데모, 터미널 녹화, 스크린샷)을 캡처하며, 프로젝트 유형에 맞는 등급을 선택합니다. |
| [`/ce-report-bug`](../../docs/skills/ce-report-bug.md) | compound-engineering 플러그인의 버그를 보고합니다. |
| [`/ce-resolve-pr-feedback`](../../docs/skills/ce-resolve-pr-feedback.md) | PR 리뷰 피드백을 병렬로 해결합니다. |
| [`/ce-test-browser`](../../docs/skills/ce-test-browser.md) | PR의 영향을 받는 페이지들에 대해 브라우저 테스트를 실행합니다. |
| [`/ce-test-xcode`](../../docs/skills/ce-test-xcode.md) | XcodeBuildMCP를 사용하여 시뮬레이터에서 iOS 앱을 빌드하고 테스트합니다. |
| [`/ce-setup`](../../docs/skills/ce-setup.md) | 환경을 진단하고, 누락된 도구를 설치하며, 프로젝트 설정을 초기화합니다. |
| [`/ce-update`](../../docs/skills/ce-update.md) | 플러그인 버전을 확인하고 오래된 캐시를 수정합니다 (Claude Code 전용). |
| [`/ce-release-notes`](../../docs/skills/ce-release-notes.md) | 최근 플러그인 릴리스를 요약하거나, 버전 인용과 함께 과거 릴리스에 대한 질문에 답변합니다. |

### 개발 프레임워크 (Development Frameworks)

| 스킬 | 설명 |
|-------|-------------|
| `ce-agent-native-architecture` | 프롬프트 네이티브 아키텍처를 사용하여 AI 에이전트를 구축합니다. |
| `ce-dhh-rails-style` | DHH의 37signals 스타일로 Ruby/Rails 코드를 작성합니다. |
| [`ce-frontend-design`](../../docs/skills/ce-frontend-design.md) | 프로덕션급 프론트엔드 인터페이스를 제작합니다. |

### 리뷰 및 품질 (Review & Quality)

| 스킬 | 설명 |
|-------|-------------|
| [`ce-doc-review`](../../docs/skills/ce-doc-review.md) | 병렬 페르소나 에이전트를 사용하여 역할별 피드백을 통해 문서를 리뷰합니다. |
| [`/ce-simplify-code`](../../docs/skills/ce-simplify-code.md) | 재사용성, 품질 및 효율성을 위해 최근 코드 변경 사항을 단순화합니다. 병렬 리뷰어가 문제를 찾고, 수정 사항을 적용하며, 테스트를 통해 동작을 검증합니다. |

### 콘텐츠 및 협업 (Content & Collaboration)

| 스킬 | 설명 |
|-------|-------------|
| [`ce-proof`](../../docs/skills/ce-proof.md) | Proof 협업 에디터를 통해 문서를 작성, 편집 및 공유합니다. |

### 자동화 및 도구 (Automation & Tools)

| 스킬 | 설명 |
|-------|-------------|
| `ce-gemini-imagegen` | Google Gemini API를 사용하여 이미지를 생성하고 편집합니다. |

### 베타 / 실험 기능 (Beta / Experimental)

| 스킬 | 설명 |
|-------|-------------|
| [`ce-polish-beta`](../../docs/skills/ce-polish-beta.md) | `/ce-code-review` 이후의 인간 개입(HITL) 다듬기 단계입니다. 리뷰 및 CI를 확인하고, 데브 서버를 실행하며, 테스트 체크리스트를 생성하고, 수정을 위해 다듬기용 서브 에이전트를 파견합니다. 규모가 큰 작업의 경우 stacked-PR 시드를 생성합니다. |
| `/lfg` | 완전 자율 엔지니어링 워크플로우입니다. |

## 에이전트 (Agents)

에이전트는 스킬에 의해 호출되는 특화된 서브 에이전트입니다. 일반적으로 사용자가 직접 호출하지 않습니다.

### 리뷰 (Review)

| 에이전트 | 설명 |
|-------|-------------|
| `ce-agent-native-reviewer` | 기능이 에이전트 네이티브(동작 및 컨텍스트 동등성)인지 확인합니다. |
| `ce-api-contract-reviewer` | 파괴적인 API 컨트랙트 변경 사항을 감지합니다. |
| `ce-architecture-strategist` | 아키텍처 결정 사항 및 준수 여부를 분석합니다. |
| `ce-code-simplicity-reviewer` | 단순성 및 미니멀리즘을 위한 최종 점검을 수행합니다. |
| `ce-correctness-reviewer` | 로직 오류, 에지 케이스, 상태 버그를 검토합니다. |
| `ce-data-integrity-guardian` | 데이터베이스 마이그레이션 및 데이터 무결성을 점검합니다. |
| `ce-data-migration-expert` | ID 매핑이 프로덕션과 일치하는지 확인하고 값이 바뀌지 않았는지 점검합니다. |
| `ce-data-migrations-reviewer` | 신뢰도 보정과 함께 마이그레이션 안전성을 검토합니다. |
| `ce-deployment-verification-agent` | 위험한 데이터 변경에 대한 Go/No-Go 배포 체크리스트를 생성합니다. |
| `ce-dhh-rails-reviewer` | DHH의 관점에서 Rails 코드를 리뷰합니다. |
| `ce-julik-frontend-races-reviewer` | JavaScript/Stimulus 코드의 레이스 컨디션을 리뷰합니다. |
| `ce-kieran-rails-reviewer` | 엄격한 컨벤션을 바탕으로 Rails 코드를 리뷰합니다. |
| `ce-kieran-python-reviewer` | 엄격한 컨벤션을 바탕으로 Python 코드를 리뷰합니다. |
| `ce-kieran-typescript-reviewer` | 엄격한 컨벤션을 바탕으로 TypeScript 코드를 리뷰합니다. |
| `ce-maintainability-reviewer` | 결합도, 복잡성, 명명 규칙, 데드 코드를 검토합니다. |
| `ce-pattern-recognition-specialist` | 코드의 패턴 및 안티 패턴을 분석합니다. |
| `ce-performance-oracle` | 성능 분석 및 최적화를 수행합니다. |
| `ce-performance-reviewer` | 신뢰도 보정과 함께 런타임 성능을 검토합니다. |
| `ce-reliability-reviewer` | 프로덕션 안정성 및 실패 모드를 검토합니다. |
| `ce-schema-drift-detector` | PR에서 무관한 schema.rb 변경 사항을 감지합니다. |
| `ce-security-reviewer` | 신뢰도 보정과 함께 취약점을 검토합니다. |
| `ce-security-sentinel` | 보안 감사 및 취약성 평가를 수행합니다. |
| `ce-swift-ios-reviewer` | Swift 및 iOS 코드 리뷰(SwiftUI 상태, 리테인 사이클, 동시성, Core Data 스레딩, 접근성 등)를 수행합니다. |
| `ce-testing-reviewer` | 테스트 커버리지 공백 및 약한 어설션을 검토합니다. |
| `ce-project-standards-reviewer` | CLAUDE.md 및 AGENTS.md 준수 여부를 검토합니다. |
| `ce-adversarial-reviewer` | 컴포넌트 경계를 아우르는 구현을 깨뜨리기 위한 실패 시나리오를 구성합니다. |

### 문서 리뷰 (Document Review)

| 에이전트 | 설명 |
|-------|-------------|
| `ce-coherence-reviewer` | 문서의 내부 일관성, 모순, 용어 혼선 등을 리뷰합니다. |
| `ce-design-lens-reviewer` | 누락된 디자인 결정, 상호작용 상태, AI 슬롭(slop) 위험 등을 리뷰합니다. |
| `ce-feasibility-reviewer` | 제안된 기술적 접근 방식이 실제 환경에서 유효할지 평가합니다. |
| `ce-product-lens-reviewer` | 문제 정의에 도전하고, 범위 결정을 평가하며, 목표 불일치를 노출합니다. |
| `ce-scope-guardian-reviewer` | 정당화되지 않은 복잡성, 범위 확장(scope creep), 섣부른 추상화에 도전합니다. |
| `ce-security-lens-reviewer` | 계획 단계의 보안 공백(인증, 데이터, API)을 평가합니다. |
| `ce-adversarial-document-reviewer` | 전제 조건에 도전하고, 명시되지 않은 가정을 드러내며, 결정 사항을 스트레스 테스트합니다. |

### 리서치 (Research)

| 에이전트 | 설명 |
|-------|-------------|
| `ce-best-practices-researcher` | 외부 모범 사례 및 사례를 수집합니다. |
| `ce-framework-docs-researcher` | 프레임워크 문서 및 모범 사례를 조사합니다. |
| `ce-git-history-analyzer` | git 히스토리 및 코드 진화 과정을 분석합니다. |
| `ce-issue-intelligence-analyst` | GitHub 이슈를 분석하여 반복되는 테마와 고충 패턴을 파악합니다. |
| `ce-learnings-researcher` | 내부 지식 저장소에서 관련 과거 솔루션을 검색합니다. |
| `ce-repo-research-analyst` | 저장소 구조 및 컨벤션을 조사합니다. |
| `ce-session-historian` | 관련 조사 컨텍스트를 위해 이전 Claude Code, Codex, Cursor 세션을 검색합니다. |
| `ce-slack-researcher` | 현재 작업과 관련된 조직의 컨텍스트를 Slack에서 검색합니다. |
| `ce-web-researcher` | 반복적인 웹 리서치를 수행하고 구조화된 외부 근거(선행 사례, 인접 솔루션, 시장 신호 등)를 반환합니다. |

### 디자인 (Design)

| 에이전트 | 설명 |
|-------|-------------|
| `ce-design-implementation-reviewer` | UI 구현이 Figma 디자인과 일치하는지 확인합니다. |
| `ce-design-iterator` | 체계적인 디자인 반복을 통해 UI를 개선합니다. |
| `ce-figma-design-sync` | 웹 구현 사항을 Figma 디자인과 동기화합니다. |

### 워크플로우 (Workflow)

| 에이전트 | 설명 |
|-------|-------------|
| `ce-pr-comment-resolver` | PR 코멘트를 해결하고 수정 사항을 구현합니다. |
| `ce-spec-flow-analyzer` | 사용자 흐름을 분석하고 명세서의 공백을 식별합니다. |

### 문서화 (Docs)

| 에이전트 | 설명 |
|-------|-------------|
| `ce-ankane-readme-writer` | Ruby gem을 위한 Ankane 스타일 템플릿의 README를 작성합니다. |

## 설치

Claude Code, Codex, Cursor, Copilot 등 다양한 플랫폼에서의 설치 방법은 저장소 루트의 [설치 섹션](../../README.md#install)을 참조하세요.

설치 후 `/ce-setup`을 실행하여 환경을 점검하고 권장 도구들을 설치하세요.

## 버전 히스토리

공식 릴리스 히스토리는 저장소 루트의 [CHANGELOG.md](../../CHANGELOG.md)를 참조하세요.

## 라이선스

MIT
