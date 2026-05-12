# 에이전트 지침 (Agent Instructions)

이 저장소는 주로 `compound-engineering` 코딩 에이전트 플러그인과 이를 배포하는 데 사용되는 Claude Code 마켓플레이스/카탈로그 메타데이터를 보관합니다.

또한 다음을 포함합니다:
- Claude Code 플러그인을 다른 에이전트 플랫폼 형식으로 변환하는 Bun/TypeScript CLI
- `plugins/` 아래의 추가 플러그인 (예: `coding-tutor`)
- CLI, 마켓플레이스 및 플러그인을 위한 공통 릴리스 및 메타데이터 인프라

`AGENTS.md`는 저장소의 공식 지침 파일입니다. 루트의 `CLAUDE.md`는 여전히 이 파일을 찾는 도구 및 변환과의 호환성을 위해서만 존재합니다.

## 빠른 시작 (Quick Start)

```bash
bun install
bun test                  # 전체 테스트 스위트 실행
bun run release:validate  # 플러그인/마켓플레이스 일관성 확인
```

## 작업 협약 (Working Agreement)

- **브랜칭 (Branching):** 사소하지 않은 모든 변경 사항에 대해 기능(feature) 브랜치를 생성하세요. 이미 해당 작업에 적합한 브랜치에 있다면 그대로 사용하고, 명시적인 요청이 없는 한 추가 브랜치나 워크트리를 만들지 마세요.
- **머지 정책 (Merge policy):** `main` 브랜치에 대한 모든 변경은 풀 리퀘스트(PR)를 통해 이루어집니다. 직접 푸시 및 직접 머지는 허용되지 않습니다. `main` 브랜치의 보호 기능은 `test` 상태 확인 통과를 요구함으로써 이를 강제합니다. 직접 경로는 `release:validate`, 테스트 스위트 및 PR 제목 검증을 우회하며, 과거의 직접 머지는 여러 PR을 통한 복구가 필요한 버전 드리프트를 유발했습니다.
- **안전 (Safety):** 사용자 데이터를 삭제하거나 덮어쓰지 마세요. 파괴적인 명령어는 피하세요.
- **테스트 (Testing):** 파싱, 변환 또는 출력에 영향을 주는 변경 후에는 `bun test`를 실행하세요.
- **릴리스 버전 관리 (Release versioning):** 릴리스는 일반 기능 PR이 아닌 릴리스 자동화에 의해 준비됩니다. 저장소에는 여러 릴리스 구성 요소(`cli`, `compound-engineering`, `coding-tutor`, `marketplace`)가 있습니다. GitHub 릴리스 PR 및 GitHub Releases가 새 릴리스의 공식 릴리스 노트 표면이며, 루트의 `CHANGELOG.md`는 해당 이력에 대한 포인터일 뿐입니다. 릴리스 자동화가 변경 의도를 분류할 수 있도록 `feat:`, `fix:`와 같은 관례적인 제목을 사용하되, 일반 PR에서 릴리스 소유 버전을 직접 올리거나 릴리스 노트를 직접 작성하지 마세요.
- **연결된 버전 (cli + compound-engineering):** `linked-versions` release-please 플러그인은 `cli`와 `compound-engineering`의 버전을 동일하게 유지합니다. 이는 의도된 것이며, CLI와 함께 제공되는 플러그인 간의 버전 추적을 단순화합니다. 결과적으로 플러그인 변경 사항만 있는 릴리스도 CLI 버전을 올리게 됩니다.
- **출력 경로 (Output Paths):** 각 에이전트 플랫폼의 표준(Idiomatic) 경로를 준수하며, 기존 사용자 설정을 보호하기 위해 **딥 머지(Deep-merge)**를 우선합니다.
  - **Claude Code:** 루트의 `CLAUDE.md` 및 `.claude/` 폴더를 사용합니다.
  - **Gemini CLI:** `~/.gemini/` 또는 `.gemini/` 아래의 `skills/`, `agents/`, `commands/` 폴더를 사용합니다.
  - **Codex:** `.codex/` 폴더를 사용합니다.
  - **OpenCode:** `opencode.json` 및 `.opencode/` 폴더를 사용하며, 명령어는 `commands/<name>.md`로 개별 저장됩니다.
  - **공통 원칙:** 설치 시 기존의 `json`, `toml`, `yaml` 설정 파일을 통째로 덮어쓰지 않고, 플러그인에 필요한 키만 안전하게 합칩니다. 사용자가 직접 설정한 값(모델, 테마 등)이 항상 우선권을 가집니다.
- **임시 공간 (Scratch Space):** 기본적으로 OS 임시 디렉토리를 사용하세요. 아래 규칙에 의해 명시적으로 정당화되는 경우에만 `.context/`를 사용하세요.
  - **기본: OS 임시 디렉토리** — 저장소 존재 여부나 다른 스킬의 파일 읽기 여부와 관계없이 실행당 일회용 및 호출 간 재사용 가능한 대부분의 임시 파일을 처리합니다.
    - **실행당 일회용**: `mktemp -d -t <prefix>-XXXXXX` (OS가 정리를 담당). 한 번 소비되고 버려지는 파일(스크린샷, GIF, 중간 빌드 출력, 녹화 등)에 사용하세요.
    - **호출 간 재사용 가능**: 고정된 경로 `/tmp/compound-engineering/<skill-name>/<run-id>/`를 사용하여 동일한 스킬의 나중 호출이 이전 실행 ID를 찾을 수 있도록 하세요. macOS의 `$TMPDIR`은 사용자가 접근하기 어려우므로 `/tmp`를 직접 사용하세요.
  - **예외: `.context/`** — 아티팩트가 현재 작업 디렉토리(CWD) 저장소에 진정으로 바인딩되어 있고 다음 중 하나 이상을 충족하는 경우에만 사용하세요.
    - (a) **사용자 큐레이션**: 사용자가 스킬 외부에서 아티팩트를 수동으로 검토하거나 조작할 것으로 예상되는 경우.
    - (b) **저장소+브랜치 불가분성**: 아티팩트의 의미가 이 특정 저장소나 브랜치와 분리될 수 없는 경우.
    - (c) **경로가 핵심 UX인 경우**: 아티팩트 경로를 사용자에게 보여주는 것이 스킬 출력의 핵심이며, 해당 경로를 OS 임시 위치보다 저장소 상대 경로로 전달하는 것이 더 쉬운 경우.
  - **영구 출력물** (계획, 명세, 학습 내용, 문서 등)은 임시 계층이 아닌 `docs/` 또는 다른 저장소 추적 위치에 속합니다.
- **문자 인코딩:**
  - **식별자** (파일명, 에이전트명, 명령어명): ASCII만 사용하세요. 변환기 및 정규식 패턴이 이에 의존합니다.
  - **마크다운 표:** 박스 그리기 문자가 아닌 파이프 구분 기호(`| col | col |`)를 사용하세요.
  - **산문 및 스킬 콘텐츠:** 유니코드(이모지 등) 사용이 가능합니다. 코드 블록 및 터미널 예시에서는 유니코드 화살표보다 ASCII 화살표(`->`, `<-`)를 선호하세요.

## 디렉토리 레이아웃 (Directory Layout)

```
src/              CLI 진입점, 파서, 변환기, 타겟 라이터
plugins/          플러그인 워크스페이스 (compound-engineering, coding-tutor)
.claude-plugin/   Claude 마켓플레이스 카탈로그 메타데이터
tests/            변환기, 라이터 및 CLI 테스트 + 피스처(fixtures)
docs/             요구사항, 계획, 솔루션 및 타겟 명세
```

## 저장소 표면 (Repo Surfaces)

이 저장소의 변경 사항은 다음 중 하나 이상의 표면에 영향을 줄 수 있습니다.

- `plugins/compound-engineering/` 아래의 `compound-engineering`
- `.claude-plugin/` 아래의 Claude 마켓플레이스 카탈로그 메타데이터
- `src/` 및 `package.json`의 변환기/설치 CLI
- `plugins/coding-tutor/`와 같은 보조 플러그인

영향을 받는 파일을 확인하지 않고 변경 사항이 "CLI 전용" 또는 "플러그인 전용"이라고 가정하지 마세요.

## 플러그인 유지 관리 (Plugin Maintenance)

`plugins/compound-engineering/` 콘텐츠를 변경할 때:

- 플러그인 동작, 인벤토리 또는 사용법이 변경되면 `plugins/compound-engineering/README.md`와 같은 주요 문서를 업데이트하세요.
- 플러그인 또는 마켓플레이스 매니페스트에서 릴리스 소유 버전을 직접 올리지 마세요.
- `CHANGELOG.md`에 릴리스 항목을 직접 추가하지 마세요.
- 에이전트, 명령어, 스킬 등이 변경된 경우 `bun run release:validate`를 실행하세요.
- 스킬, 에이전트 또는 명령어를 제거할 때, 업그레이드 시 이전 아티팩트가 정리되도록 `src/utils/legacy-cleanup.ts` 및 `src/data/plugin-legacy-artifacts.ts`의 정리 레지스트리에 이름을 추가하세요.

## 에이전트 및 스킬 변경 검증 (Validating Agent and Skill Changes)

플러그인 에이전트나 스킬(`plugins/*/agents/` 또는 `plugins/*/skills/` 아래의 파일들)에 대한 동작 변경은 Claude Code가 플러그인을 로드하는 방식 때문에 기계적인 코드 변경과는 다른 검증 경로가 필요합니다.

- **변경 사항 테스트를 위해 `skill-creator` 스킬을 사용하세요.** 이 스킬은 범용 서브에이전트를 생성하고 디스패치 시점에 에이전트 또는 스킬 콘텐츠를 프롬프트에 주입하므로, 각 실행 시 디스크에서 현재 소스를 읽습니다. `/skill-creator`를 호출하고 해당 에발(eval) 워크플로우를 사용하세요.
- **플러그인 에이전트 및 스킬 정의는 모두 세션 시작 시 캐시됩니다.** Claude Code 세션이 열리면 세션 시작 시 로드된 메모리 내 복사본이 실행됩니다. 세션 시작 후 해당 레이어에 대한 파일 편집은 동일한 세션 내에서 전파되지 않습니다.
- **캐시를 강제로 다시 로드하기 위해 `~/.claude/plugins/cache/` 등을 편집하지 마세요.** 이는 사용자 머신 상태이지 저장소 관리 대상이 아닙니다. `skill-creator` 패턴이 올바른 접근 방식입니다.

## 코딩 컨벤션 (Coding Conventions)

- 플랫폼 간 변환 시 암시적인 마법보다 명시적인 매핑을 선호하세요.
- 타겟별 동작은 개별 라이터 모듈에 유지하고 조건문을 여러 파일에 흩뿌리지 마세요.
- 설치된 타겟에 대해 안정적인 출력 경로와 머지 의미론을 유지하세요.

## 커밋 컨벤션 (Commit Conventions)

- **접두사는 파일 형식이 아닌 의도에 따릅니다.** `feat:`, `fix:`, `docs:`, `refactor:` 등의 관례적인 접두사를 사용하되, 파일 확장자가 아닌 변경 사항이 하는 일에 따라 분류하세요. `plugins/*/skills/`, `plugins/*/agents/` 아래의 파일은 마크다운이나 JSON일지라도 제품 코드입니다. `docs:`는 문서화가 유일한 목적인 파일(`README.md`, `docs/`, `CHANGELOG.md`)을 위해 예약해 두세요.
- **컴포넌트 스코프(Scope)를 포함하세요.** 변경 로그에 표시될 좁고 유용한 라벨을 선택하세요: 스킬/에이전트 이름(`document-review`, `learnings-researcher`), 플러그인 또는 CLI 영역(`coding-tutor`, `cli`) 등. `compound-engineering`은 너무 광범위하므로 사용하지 마세요.

## 새로운 타겟 제공자 추가 (Adding a New Target Provider)

타겟 형식이 안정적이고 문서화되어 있으며 도구/권한/후크에 대한 명확한 매핑이 있는 경우에만 제공자를 추가하세요. `src/targets/`에서 새 핸들러를 정의하고, `src/types/` 및 `src/converters/`에서 매핑을 구현하며, CLI가 `--to <provider>`를 지원하도록 하고, 필요한 테스트와 문서를 추가하세요.

## 스킬 내 에이전트 참조 (Agent References in Skills)

스킬 `SKILL.md` 파일 내에서 에이전트를 참조할 때(예: `Agent` 또는 `Task` 도구 사용 시) `ce-<agent-name>` 형식을 사용하세요. `ce-` 접두사는 에이전트를 compound-engineering 구성 요소로 식별하며 플러그인 간 고유성을 보장합니다.

## 스킬 내 파일 참조 (File References in Skills)

각 스킬 디렉토리는 독립적인 단위입니다. `SKILL.md` 파일은 스킬 루트로부터의 상대 경로를 사용하여 자신의 디렉토리 트리(`references/`, `assets/`, `scripts/` 등) 내에 있는 파일만 참조해야 합니다. 상대 경로 트래버스(`../`)나 절대 경로를 사용하여 스킬 디렉토리 외부의 파일을 참조하지 마세요.

## 스킬 내 플랫폼별 변수 (Platform-Specific Variables in Skills)

이 플러그인은 여러 플랫폼을 위해 작성되고 변환됩니다. `${CLAUDE_PLUGIN_ROOT}`와 같은 플랫폼별 환경 변수나 문자열 치환을 사용할 때, 해당 변수를 사용할 수 없을 때를 대비한 우아한 폴백(fallback)을 포함해야 합니다. 가능한 경우 스킬 디렉토리로부터의 상대 경로를 사용하는 것이 좋습니다.

## 저장소 문서 관례 (Repository Docs Convention)

문서의 양이 많아짐에 따라 시인성을 높이기 위해 **년도/월별 폴더 구조**를 사용하며, 파일명에는 일자(`DD`)를 포함하여 정렬 순서를 유지합니다. (형식: `docs/<카테고리>/YYYY/MM/DD-<제목>.md`)

- **요구사항(Requirements)**: `docs/brainstorms/` 하위의 년도/월 폴더에 위치합니다.
- **계획(Plans)**: `docs/plans/` 하위의 년도/월 폴더에 위치합니다.
- **아이데이션(Ideation)**: `docs/ideation/` 하위의 년도/월 폴더에 위치합니다.
- **솔루션(Solutions)**: `docs/solutions/`에 위치하며, 날짜별이 아닌 카테고리별로 관리합니다.
- **명세(Specs)**: `docs/specs/`에 위치하며, 타겟 플랫폼 형식 명세를 담습니다.
- **보고서(Reports)**: `docs/pulse-reports/` 하위의 년도/월 폴더에 위치합니다.

### 솔루션 카테고리 (`docs/solutions/`)
- **`developer-experience/`**: 이 저장소 기여와 관련된 문제 (로컬 개발 설정, CI 등).
- **`integrations/`**: 플러그인 출력이 타겟 플랫폼이나 OS에서 올바르게 작동하지 않는 문제.
- **`workflow/`**, **`skill-design/`**: 플러그인 스킬 및 에이전트 디자인 패턴, 워크플로우 개선.
