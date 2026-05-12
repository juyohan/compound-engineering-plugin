# Codex 위임 워크플로우 (Codex Delegation Workflow)

`delegation_active`가 `true`인 경우, 코드 구현은 직접 수행되지 않고 Codex CLI (`codex exec`)로 위임됩니다. 오케스트레이션 역할을 하는 Claude Code 에이전트는 계획 수립, 리뷰, git 작업 및 전체적인 관리를 담당합니다.

## 위임 결정 (Delegation Decision)

`work_delegate_decision`이 `ask`인 경우, 권장 사항을 제시하고 사용자가 선택할 때까지 대기한 후 진행합니다.

**Codex 위임을 권장하는 경우:**

> "Codex delegation active. [N] implementation units -- delegating in one batch."
> 1. Delegate to Codex *(권장)*
> 2. Execute with Claude Code instead

**Codex 위임을 권장하되 여러 배치로 나누는 경우:**

> "Codex delegation active. [N] implementation units -- delegating in [X] batches."
> 1. Delegate to Codex *(권장)*
> 2. Execute with Claude Code instead

**Claude Code 실행을 권장하는 경우 (모든 단위 작업이 사소한 경우):**

> "Codex delegation active, but these are small changes where the cost of delegating outweighs having Claude Code do them."
> 1. Execute with Claude Code *(권장)*
> 2. Delegate to Codex anyway

사용자가 위임 옵션을 선택하면 '위임 전 체크' 단계로 진행합니다. 사용자가 Claude Code 옵션을 선택하면 `delegation_active`를 `false`로 설정하고 상위 스킬의 표준 실행 모드로 돌아갑니다.

`work_delegate_decision`이 `auto`(기본값)인 경우, 실행 계획을 한 줄로 알리고 대기 없이 진행합니다: "Codex delegation active. Delegating [N] units in [X] batch(es)." 만약 모든 단위 작업이 사소하다면 `delegation_active`를 `false`로 설정하고 진행합니다: "Codex delegation active. All units are trivial -- executing with Claude Code."

## 위임 전 체크 (Pre-Delegation Checks)

**첫 번째 배치를 실행하기 전 한 번만** 이 체크를 수행합니다. 체크에 실패하면 남은 계획 실행 동안 표준 모드로 폴백합니다. 이후 배치에서 다시 실행하지 마십시오.

**0. 플랫폼 게이트 (Platform Gate)**

Codex 위임은 오케스트레이팅 에이전트가 Claude Code에서 실행 중일 때만 지원됩니다. 현재 세션이 Codex, Gemini CLI, OpenCode 또는 다른 플랫폼인 경우 `delegation_active`를 `false`로 설정하고 표준 모드로 진행합니다.

**1. 환경 보호 (Environment Guard)**

현재 에이전트가 이미 Codex 샌드박스 내부에서 실행 중인지 확인합니다:

```bash
if [ -n "$CODEX_SANDBOX" ] || [ -n "$CODEX_SESSION_ID" ]; then
  echo "inside_sandbox=true"
else
  echo "inside_sandbox=false"
fi
```

`inside_sandbox`가 `true`인 경우 위임은 재귀적으로 발생하거나 실패하게 됩니다.

- `delegation_source`가 `argument`인 경우: "Already inside Codex sandbox -- using standard mode."를 출력하고 `delegation_active`를 `false`로 설정합니다.
- `delegation_source`가 `config` 또는 `default`인 경우: 조용히 `delegation_active`를 `false`로 설정합니다.

**2. 가용성 확인 (Availability Check)**

**Codex CLI 경로 (사전 확인):**
!`command -v codex 2>/dev/null || true`

위 라인이 절대 경로(예: `/opt/homebrew/bin/codex`)를 보여주면 Codex CLI를 사용할 수 있는 것이므로 다음 체크로 진행합니다.
그 외의 경우 — 비어 있거나, `!` 사전 확인을 처리하지 못하는 하네스에 의해 `command -v codex 2>/dev/null` 문자열이 그대로 남은 경우 등 — 쉘/Bash 도구를 통해 런타임에 `command -v codex`를 실행하여 확인하십시오. 절대 경로가 출력되면 사용 가능한 것이므로 진행합니다. 실패하거나 아무것도 출력되지 않으면 "Codex CLI not found (install via `npm install -g @openai/codex` or `brew install codex`) -- using standard mode."를 출력하고 `delegation_active`를 `false`로 설정합니다.

**3. 동의 플로우 (Consent Flow)**

`consent_granted`가 `true`가 아닌 경우 (설정 `work_delegate_consent`로부터 확인):

플랫폼의 차단형 질문 도구(Claude Code의 `AskUserQuestion`, Codex의 `request_user_input`, Gemini의 `ask_user`, Pi의 `ask_user`)를 사용하여 일회성 동의 경고를 제시하십시오. 동의 경고는 다음을 설명합니다:
- 위임은 구현 단위 작업을 구조화된 프롬프트 형태로 `codex exec`에 보냅니다.
- **yolo 모드** (`--dangerously-bypass-approvals-and-sandbox`): 네트워크를 포함한 전체 시스템 접근 권한을 가집니다. 테스트 실행이나 의존성 설치가 필요한 검증 단계에 필수적입니다. **권장됨.**
- **full-auto 모드** (`-s workspace-write`): 작업 공간 쓰기가 가능한 샌드박스로, 기본적으로 네트워크 접근이 차단됩니다. 네트워크는 `~/.codex/config.toml`의 `[sandbox_workspace_write]` 섹션에서 `network_access = true`로 설정하여 다시 활성화할 수 있습니다.

샌드박스 모드 선택지를 제시하십시오: (1) yolo (권장), (2) full-auto.

동의 시:
- 리포지토리 루트 확인: `git rev-parse --show-toplevel`. `<repo-root>/.compound-engineering/config.local.yaml`에 `work_delegate_consent: true` 및 `work_delegate_sandbox: <선택한 모드>`를 작성합니다.
- 작성 방법: (1) 파일이나 디렉토리가 없으면 `<repo-root>/.compound-engineering/`을 생성하고 YAML 파일을 작성합니다. (2) 파일이 이미 있으면 기존 키를 보존하며 새 키를 병합합니다.
- 확인된 상태의 `consent_granted`와 `sandbox_mode`를 업데이트합니다.

거절 시:
- 이 프로젝트에서 위임을 완전히 비활성화할지 묻습니다.
- '예' 선택 시: 위와 동일한 방식으로 `<repo-root>/.compound-engineering/config.local.yaml`에 `work_delegate: false`를 작성합니다. `delegation_active`를 `false`로 설정하고 표준 모드로 진행합니다.
- '아니오' 선택 시: 이번 호출에 대해서만 `delegation_active`를 `false`로 설정하고 표준 모드로 진행합니다.

**헤드리스 동의:** 헤드리스 또는 비대화형 컨텍스트에서 실행 중인 경우, 설정 파일의 `work_delegate_consent`가 이미 `true`인 경우에만 위임이 진행됩니다. 동의 기록이 없으면 조용히 `delegation_active`를 `false`로 설정합니다.

## 배치 처리 (Batching)

모든 단위 작업을 하나의 배치로 위임합니다. 계획이 5개 단위를 초과하는 경우, 계획의 페이즈 경계에서 나누거나 약 5개 단위씩 묶으십시오. 파일을 공유하는 단위 작업들은 절대로 나누지 마십시오. 모든 단위 작업이 사소한 경우 위임을 완전히 건너뜁니다.

## 배치별 리소스 수준 (Per-Batch Effort)

각 배치는 복잡도에 비례하는 리소스 수준(effort level)을 선택하며, 호출 전 설정된 하한선(floor)과 대조하여 결정합니다.

**리소스 수준 — 술어가 아닌 가이드라인**

배치에 가장 적합한 수준을 선택하십시오. 이들은 판단을 돕는 신호일 뿐 절대적인 기준은 아닙니다.

- **default (플래그 없음)** — 동작 변경이 없는 사소한 작업: 한 줄의 설정 수정, 이름 변경, 오타나 주석 수정, 순수 문서 업데이트. 사용자의 `~/.codex/config.toml` 기본값(표준 Codex 설치 시 `medium`)을 따릅니다.
- **`medium`** — 위험 지역을 벗어난 작고 잘 정의된 동작 변경. 소수의 파일, 단일 관심사, 새로운 아키텍처 없음.
- **`high`** — 위험 지역(인증/세션 로직, 결제, 데이터베이스 마이그레이션, 외부 API 계약, 재시도/폴백이 포함된 에러 처리)을 건드리는 작업, 또는 하나의 실수가 연쇄적인 영향을 미칠 수 있는 넓은 영역에 걸친 작업.
- **`xhigh`** — 아키텍처 관련 작업: 횡단적 리팩토링, 동일 배치 내의 여러 위험 지역, 광범위하게 전파되는 변경 사항, 또는 잘못된 판단이 프로젝트를 실질적으로 악화시킬 수 있는 모든 경우.

확실하지 않을 때는 한 단계 높은 수준을 선택하십시오 — 위험한 작업에 리소스를 적게 할당하는 것이 일반적인 작업에 과하게 할당하는 것보다 비용이 더 많이 듭니다. 선택한 수준과 그 이유(예: "`high` — DB/마이그레이션 수정")를 짧게 기록하여 감사가 가능하게 하십시오.

명시적으로 처리할 가치가 있는 예외 상황:
- **테스트 전용 배치:** 파일 경로가 아닌 테스트가 *검증하는 대상*에 따라 분류하십시오. 인증 플로우, 결제 로직 또는 마이그레이션에 대한 테스트는 해당 구현 작업과 동일한 수준을 받습니다.
- **복합 복잡도 배치:** 배치는 하나의 수준을 선택합니다. 오타 수정과 결제 로직 재작성이 섞여 있다면 높은 수준을 선택하십시오. 격차가 너무 크다면 평균을 내기보다 배치 단계에서 나누는 것을 선호하십시오.
- **삭제 전용 배치:** 남은 내용의 양이 아닌 삭제되는 대상의 리스크에 따라 분류하십시오. 인증 모듈을 삭제하는 것은 변경 사항이 0라인이라 하더라도 `high`입니다.
- **문서 또는 주석 전용 배치:** `default`.

**하한선 및 결정 — 엄격한 규칙**

리소스 수준의 순서는 다음과 같습니다: `minimal < low < medium < high < xhigh`.

`effective_effort`를 계산합니다:

- `delegate_effort`가 설정되지 않은 경우: `effective_effort = picked_level`.
- `delegate_effort`가 설정된 경우: `picked_level`에서 `default`를 `medium`으로 치환한 후, `effective_effort = max(picked_level, delegate_effort)`를 취합니다.

`effective_effort`에 따라 다음과 같이 명령어를 구성합니다:

- `medium`, `high`, 또는 `xhigh` 인 경우 → `-c 'model_reasoning_effort="<value>"'`를 추가합니다.
- `default` 인 경우 → 플래그를 생략합니다 (`~/.codex/config.toml`에 위임). `delegate_effort`가 설정되지 않고 선택된 수준이 `default`일 때만 가능합니다.

`codex exec`에 `"default"`라는 문자열을 그대로 전달하지 마십시오.

`effective_effort`를 배치별 파생 상태 값으로 저장하고 (세션 수준의 `delegate_effort`와 함께), 실행 루프 전체에서 `delegate_effort` 대신 사용하십시오.

## 프롬프트 템플릿 (Prompt Template)

위임 실행 시작 시, `mktemp -d`를 통해 실행당 OS 임시 스크래치 디렉토리를 생성하고 모든 후속 사용을 위해 해당 **절대 경로**를 캡처하십시오. 이 호출을 위한 모든 스크래치 파일은 해당 디렉토리 하위에 위치합니다. `.context/`를 사용하지 마십시오 — 이 스크래치 파일들은 위임 실행이 종료되면 정리되는 일회용 아티팩트이며(아래 정리 섹션 참조), 리포지토리 스크래치 공간 컨벤션을 따릅니다. 쉘 도구가 아닌 도구(Write, Read)에 확인되지 않은 쉘 변수 문자열을 전달하지 말고, `mktemp -d`가 반환한 절대 경로를 사용하십시오.

```bash
SCRATCH_DIR="$(mktemp -d -t ce-work-codex-XXXXXX)"
echo "$SCRATCH_DIR"
```

이 워크플로우 전반에서 위 절대 경로를 `<scratch-dir>`로 지칭합니다.

각 배치 실행 전, `<scratch-dir>/prompt-batch-<batch-num>.md`에 프롬프트 파일을 작성하십시오.

배치의 단위 작업들을 사용하여 다음 XML 태그 섹션으로 프롬프트를 구성하십시오:

```xml
<task>
[단일 단위 작업 배치의 경우: 해당 작업의 Goal.
다중 단위 작업 배치의 경우: 각 작업의 Goal을 나열하며, 구체적인 작업 내용, 리포지토리 컨텍스트 및 기대되는 최종 상태를 명시합니다.]
</task>

<files>
[배치 내 모든 작업의 파일 목록을 통합하여 나열 — 생성, 수정 또는 읽을 파일들.]
</files>

<patterns>
[모든 작업의 "Patterns to follow" 필드에 있는 파일 경로들. 패턴이 없는 경우:
"No explicit patterns referenced -- follow existing conventions in the
modified files."]
</patterns>

<approach>
[단일 단위 작업 배치의 경우: 해당 작업의 Approach.
다중 단위 작업 배치의 경우: 각 작업의 Approach를 나열하며, 의존성 및 권장 순서를 명시합니다.]
</approach>

<constraints>
- git commit, git push 또는 PR 생성을 수행하지 마십시오 -- 오케스트레이팅 에이전트가 모든 git 작업을 처리합니다.
- 모든 수정 사항은 리포지토리 루트 내부의 파일로 제한하십시오.
- 명시된 작업 범위에 집중하십시오 -- 관련 없는 리팩토링, 이름 변경 또는 정리를 피하십시오.
- 중단하기 전 작업을 완전히 해결하십시오 -- 첫 번째로 그럴싸한 답이 나왔다고 멈추지 마십시오.
- 실행 도중 리포지토리 루트 외부의 파일을 수정해야 함을 발견하면, 리포지토리 내부에서 할 수 있는 것들을 완료하고 할 수 없었던 내용을 결과 스키마의 issues 필드에 보고하십시오.
</constraints>

<testing>
테스트를 작성하기 전, 계획의 테스트 시나리오가 각 작업에 해당하는 모든 카테고리를 커버하는지 확인하십시오. 부족한 부분을 보완하여 테스트를 작성하십시오:
- Happy path: 각 작업 목표의 핵심 입력/출력 쌍
- Edge cases: 경계값, 빈 값/nil 입력, 타입 불일치
- Error/failure paths: 잘못된 입력, 권한 거부, 하류 시스템 실패
- Integration: 모의 객체(mock)만으로는 증명할 수 없는 레이어 간 시나리오

구체적인 입력과 기대 결과를 명시하는 테스트를 작성하십시오. 콜백, 미들웨어 또는 이벤트 핸들러가 포함된 코드를 수정하는 경우, 상호작용 체인이 엔드투엔드로 작동하는지 확인하십시오.
</testing>

<verify>
구현 후, 모든 테스트 파일을 개별적이 아닌 하나의 명령어로 한꺼번에 실행하십시오. 파일 간의 오염(예: 테스트 파일 간에 유출되는 모킹된 글로벌 변수)은 동일한 프로세스에서 실행할 때만 표면화됩니다. 테스트가 실패하면 수정하고 통과할 때까지 다시 실행하십시오. 검증이 통과되지 않는 한 상태를 "completed"로 보고하지 마십시오. 이것은 당신의 책임입니다 -- 오케스트레이터가 검증을 별도로 다시 실행하지 않습니다.

[프로젝트의 테스트 및 린트 명령어. 모든 작업의 검증 명령어를 통합하여 단일 호출로 사용하십시오.]
</verify>

<output_contract>
--output-schema 메커니즘을 통해 결과를 보고하십시오. 모든 필드를 채우십시오:
- status: 모든 변경 사항이 적용되고 검증이 통과된 경우에만 "completed", 미완성인 경우 "partial", 유의미한 진전이 없는 경우 "failed"
- files_modified: 변경한 파일 경로 배열
- issues: 발견된 문제, 공백 또는 범위를 벗어난 작업을 설명하는 문자열 배열
- summary: 수행한 작업에 대한 한 문단 설명
- verification_summary: 검증을 위해 실행한 내용(명령어 및 결과).
  예: "Ran `bun test` -- 14 tests passed, 0 failed."
  검증이 불가능했다면 이유를 설명하십시오.
</output_contract>
```

## 결과 스키마 (Result Schema)

위임 실행 시작 시 한 번, `<scratch-dir>/result-schema.json`에 결과 스키마를 작성하십시오 (확인된 절대 경로 사용):

```json
{
  "type": "object",
  "properties": {
    "status": { "enum": ["completed", "partial", "failed"] },
    "files_modified": { "type": "array", "items": { "type": "string" } },
    "issues": { "type": "array", "items": { "type": "string" } },
    "summary": { "type": "string" },
    "verification_summary": { "type": "string" }
  },
  "required": ["status", "files_modified", "issues", "summary", "verification_summary"],
  "additionalProperties": false
}
```

각 배치의 결과는 `-o` 플래그를 통해 `<scratch-dir>/result-batch-<batch-num>.json`에 작성됩니다. 계획 실패 시 디버깅을 위해 파일들은 그대로 유지됩니다.

성공적인 종료 코드 이후 결과 JSON이 없거나 잘못된 경우 작업 실패로 분류합니다.

## 실행 루프 (Execution Loop)

첫 번째 배치 전 `consecutive_failures` 카운터를 0으로 초기화합니다.

**깨끗한 기준선 확인 (Clean-baseline preflight):** 첫 번째 배치 전, 추적되는 파일에 커밋되지 않은 변경 사항이 없는지 확인합니다:

```bash
git diff --quiet HEAD
```

이것은 의도적으로 추적되지 않는 파일을 무시합니다. 추적되는 파일에 대한 수정 사항만이 롤백을 안전하지 않게 만들기 때문입니다. 하지만 배치의 계획된 파일 목록에 있는 경로에 추적되지 않는 파일이 존재하는 경우, 롤백(`git clean -fd -- <paths>`) 시 해당 파일이 삭제됩니다. 이러한 중복이 감지되면 사용자에게 경고하고 진행 전 커밋하거나 스태시(stash)할 것을 권장하십시오.

추적되는 파일이 지저분한 경우, 작업을 멈추고 옵션을 제시하십시오: (1) 현재 변경 사항 커밋, (2) 명시적으로 스태시 (`git stash push -m "pre-delegation"`), (3) 표준 모드로 계속 (`delegation_active`를 `false`로 설정). 사용자의 변경 사항을 자동으로 스태시하지 마십시오.

**위임 호출 (Delegation invocation):** 각 배치에 대해 다음을 **별도의 Bash 도구 호출**로 실행하십시오 (하나로 묶지 마십시오):

**단계 A — 실행 (백그라운드, 별도 Bash 호출):**

프롬프트 파일을 작성한 후, 도구 매개변수에 `run_in_background: true`가 설정된 단일 Bash 도구 호출을 수행합니다. 이 호출은 즉시 반환되며 타임아웃 제한이 없습니다.

아래의 모든 `<scratch-dir>` 자리에 설정 단계에서 확인한 실제 절대 경로를 대입하십시오. 각 Bash 도구 호출은 새로운 쉘에서 시작되므로, 설정 스니펫의 `$SCRATCH_DIR` 변수는 유지되지 않습니다 — 확인되지 않은 `$SCRATCH_DIR`은 빈 값으로 확장되어 결과 감지를 망가뜨립니다.

```bash
# 스킬 상태에서 확인된 sandbox_mode 값 (yolo 또는 full-auto) 대입
SANDBOX_MODE="<sandbox_mode>"

# 샌드박스 플래그 결정
if [ "$SANDBOX_MODE" = "full-auto" ]; then
  SANDBOX_FLAG="-s workspace-write"
else
  SANDBOX_FLAG="--dangerously-bypass-approvals-and-sandbox"
fi

codex exec \
  $SANDBOX_FLAG \
  --output-schema "<scratch-dir>/result-schema.json" \
  -o "<scratch-dir>/result-batch-<batch-num>.json" \
  - < "<scratch-dir>/prompt-batch-<batch-num>.md"
```

**조건부 플래그** — 해당 스킬 상태 값이 설정된 경우에만 각 라인을 포함하십시오:

- `delegate_model`이 설정된 경우, `$SANDBOX_FLAG` 앞 줄에 `  -m "<delegate_model>" \`을 삽입합니다.
- `effective_effort`가 `medium`, `high` 또는 `xhigh`인 경우 (리소스 수준 결정 단계에서 확인됨), `$SANDBOX_FLAG` 앞 줄에 `  -c 'model_reasoning_effort="<effective_effort>"' \`를 삽입합니다. `effective_effort`가 `default`인 경우(하한선이 없고 선택 수준이 `default`일 때만 가능) 해당 라인을 생략하십시오 — `"default"`라는 문자열을 그대로 전달하지 마십시오.

두 값 모두 설정되지 않은 경우 해당 라인을 완전히 생략하십시오 — Codex는 사용자의 `~/.codex/config.toml` (그리고 최종적으로 CLI 자체의 기본값)로부터 기본값을 결정합니다. 설정되지 않은 값에 대해 플레이스홀더 문자열을 대입하지 마십시오.

중요: `run_in_background: true`는 쉘의 `&` 접미사가 아닌 **Bash 도구 매개변수**로 설정되어야 합니다. 도구 매개변수만이 타임아웃 제한을 제거합니다. 포그라운드 Bash 호출 내부의 쉘 `&`는 여전히 기본 2분의 타임아웃 제한에 걸립니다.

`-c` 플래그가 포함될 경우 인용이 중요합니다: 전체 key=value에 작은따옴표를 사용하고, 내부의 TOML 문자열 값에 큰따옴표를 사용하십시오. 예: `-c 'model_reasoning_effort="high"'`.

이 호출 템플릿에 명시된 조건부 삽입 외에 CLI 플래그를 임의로 추가하거나 수정하지 마십시오.

**단계 B — 폴링 (포그라운드, 별도 Bash 호출):**

실행 호출이 반환된 후, 결과 파일을 확인하는 **새로운 별도의** 포그라운드 Bash 도구 호출을 수행하십시오. 이는 에이전트의 턴을 활성 상태로 유지하여 사용자가 작업 트리에 간섭하지 못하게 합니다.

`<scratch-dir>` 자리에 설정 단계에서 확인한 실제 절대 경로를 대입하십시오. 단계 A의 쉘 변수는 별도의 Bash 도구 호출 간에 유지되지 않습니다.

```bash
RESULT_FILE="<scratch-dir>/result-batch-<batch-num>.json"
for i in $(seq 1 6); do
  test -s "$RESULT_FILE" && echo "DONE" && exit 0
  sleep 10
done
echo "Waiting for Codex..."
```

출력이 "Waiting for Codex..."인 경우, 동일한 폴링 명령을 별도의 Bash 호출로 다시 실행하십시오. 출력이 "DONE"이 될 때까지 반복한 후 결과 파일을 읽고 분류 단계로 진행하십시오.

**폴링 종료 조건:** 다음 조건 중 하나가 충족되면 폴링을 중단합니다:

- **결과 파일 나타남** (출력이 "DONE") -- 정상적으로 결과 분류로 진행합니다.
- **백그라운드 프로세스가 0이 아닌 코드로 종료됨** -- CLI 실패(표 1번)로 분류합니다. 롤백하고 표준 모드로 폴백합니다.
- **백그라운드 프로세스가 0으로 종료되었으나 결과 파일이 없음** -- 작업 실패(표 2번: 종료 코드 0, 결과 JSON 누락)로 분류합니다. 롤백하고 `consecutive_failures`를 증가시킵니다.
- **폴링 5회 경과** (약 5분) 동안 결과 파일이 나타나지 않고 백그라운드 프로세스 알림이 없음 -- 프로세스가 중단된(hung) 것으로 간주합니다. CLI 실패(표 1번)로 분류합니다. 롤백하고 표준 모드로 폴백합니다.

**결과 분류:** Codex는 내부적으로 검증을 실행하고 보고 전 실패를 수정할 책임이 있습니다 — 오케스트레이터가 검증을 별도로 다시 실행하지 않습니다.

| # | 신호 | 분류 | 조치 |
|---|--------|---------------|--------|
| 1 | 종료 코드 != 0 | CLI 실패 | HEAD로 롤백. 남은 모든 작업에 대해 표준 모드로 폴백. |
| 2 | 종료 코드 0, 결과 JSON 누락 또는 손상됨 | 작업 실패 | HEAD로 롤백. `consecutive_failures` 증가. |
| 3 | 종료 코드 0, `status: "failed"` | 작업 실패 | HEAD로 롤백. `consecutive_failures` 증가. |
| 4 | 종료 코드 0, `status: "partial"` | 부분 성공 | diff를 유지. 남은 작업을 로컬에서 완료하고 검증 후 커밋. `consecutive_failures` 증가. |
| 5 | 종료 코드 0, `status: "completed"` | 성공 | 변경 사항 커밋. `consecutive_failures`를 0으로 리셋. |

**결과 전달 — 사용자에게 노출:** 결과 JSON을 읽고 커밋 또는 롤백을 수행하기 전에, 발생한 상황을 볼 수 있도록 요약을 표시하십시오. 형식:

> **Codex batch <batch-num> — <분류>**
> <결과 JSON의 요약 내용>
>
> **Files:** <files_modified의 쉼표로 구분된 목록>
> **Verification:** <결과 JSON의 verification_summary 내용>
> **Issues:** <이슈 목록, 없으면 "None">

실패 또는 부분 성공 시 분류 사유(예: "status: failed", "result JSON missing")를 포함하여 오케스트레이터가 왜 롤백하거나 로컬에서 작업을 마무리하는지 사용자가 이해할 수 있게 하십시오.

짧게 유지하십시오 — 목적은 투명성이지 방대한 텍스트 노출이 아닙니다. 배치당 하나의 짧은 블록이면 충분합니다.

**롤백 절차:**

```bash
git checkout -- .
git clean -fd -- <배치의 통합 파일 목록에 있는 경로들>
```

경로 인수 없이 `git clean -fd`만 사용하지 마십시오.

**성공 시 커밋:**

```bash
git add $(git diff --name-only HEAD; git ls-files --others --exclude-standard)
git commit -m "feat(<scope>): <배치 요약>"
```

**배치 간 작업** (여러 배치로 나뉜 계획): 완료된 내용, 테스트 결과, 다음 작업을 보고합니다. 사용자가 개입하지 않는 한 즉시 계속 진행합니다 — 체크포인트는 사용자가 *조정할 수 있도록* 존재하는 것이지, 반드시 조정해야 하는 것은 아닙니다.

**차단기 (Circuit breaker):** 3회 연속 실패 시 `delegation_active`를 `false`로 설정하고 다음을 출력합니다: "Codex delegation disabled after 3 consecutive failures -- completing remaining units in standard mode."

**스크래치 정리:** 명시적인 정리가 필요하지 않습니다 — OS 임시 디렉토리 관리에 따라 정리됩니다 (macOS `$TMPDIR` 주기적 정리, Linux/WSL `/tmp` 재부팅 또는 주기적 정리). 실행 후 `<scratch-dir>`을 남겨두면 문제가 발생했을 때 디버깅을 위한 중간 아티팩트를 보존할 수 있습니다.

## 혼합 모델 기여 표시 (Mixed-Model Attribution)

일부 단위 작업은 Codex가 수행하고 나머지는 로컬에서 수행된 경우:
- 모든 작업이 위임을 통해 수행된 경우: Codex 모델의 기여로 표시
- 모든 작업이 표준 모드에서 수행된 경우: 현재 에이전트 모델의 기여로 표시
- 혼합된 경우: PR 본문에 어떤 작업이 위임되었는지 명시하고 두 모델 모두에 기여를 표시
