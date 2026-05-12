---
name: ce-test-browser
description: "현재 PR 또는 브랜치의 영향을 받는 페이지에서 브라우저 테스트를 실행합니다."
argument-hint: "[PR 번호, 브랜치 이름, 'current' 또는 --port PORT]"
allowed-tools:
  - gem
---

# 브라우저 테스트 스킬

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


`agent-browser` CLI를 사용하여 PR 또는 브랜치 변경 사항의 영향을 받는 페이지에서 엔드 투 엔드(end-to-end) 브라우저 테스트를 실행합니다.

## 브라우저 자동화에는 `agent-browser`만 사용하세요

이 워크플로는 `agent-browser` CLI를 독점적으로 사용합니다. 다른 브라우저 자동화 시스템, 브라우저 MCP 통합 또는 내장 브라우저 제어 도구를 사용하지 마세요. 플랫폼에서 브라우저를 제어하는 여러 방법을 제공하는 경우 항상 `agent-browser`를 선택하세요.

`agent-browser`를 다음 용도로 사용하세요: 페이지 열기, 요소 클릭, 양식 채우기, 스크린샷 캡처 및 렌더링된 콘텐츠 스크래핑.

플랫폼별 힌트:
- Claude Code에서 Chrome MCP 도구(`mcp__claude-in-chrome__*`)를 사용하지 마세요.
- Codex에서 관련 없는 다른 브라우징 도구로 대체하지 마세요.

## 전제 조건

- 로컬 개발 서버 실행 중 (예: `bin/dev`, `rails server`, `npm run dev`)
- `agent-browser` CLI 설치됨 (아래 설정 참조)
- 테스트할 변경 사항이 있는 Git 저장소

## 설정

`agent-browser`가 설치되어 있는지 확인합니다:

```bash
command -v agent-browser >/dev/null 2>&1 && echo "Installed" || echo "NOT INSTALLED"
```

설치되어 있지 않은 경우 사용자에게 알립니다: "`agent-browser`가 설치되어 있지 않습니다. 필요한 종속성을 설치하려면 `/ce-setup`을 실행하세요." 그런 다음 중단합니다 — 이 스킬은 agent-browser 없이는 작동할 수 없습니다.

## 워크플로

### 1. 설치 확인

시작하기 전에 `agent-browser`를 사용할 수 있는지 확인합니다:

```bash
command -v agent-browser >/dev/null 2>&1 && echo "Ready" || echo "NOT INSTALLED"
```

설치되어 있지 않은 경우 사용자에게 알립니다: "`agent-browser`가 설치되어 있지 않습니다. 필요한 종속성을 설치하려면 `/ce-setup`을 실행하세요." 그런 다음 중단합니다.

### 2. 브라우저 모드 질문

**파이프라인 모드 (`mode:pipeline`):** 이 단계를 완전히 건너뜁니다. 기본적으로 headless(헤드리스) 모드를 사용하며, 질문이나 차단 없이 3단계로 바로 진행합니다.

**수동 모드:** 플랫폼의 차단형 질문 도구를 사용하여 사용자에게 headed(헤디드) 또는 headless(헤드리스) 모드 실행 여부를 묻습니다: Claude Code의 `AskUserQuestion` (스키마가 로드되지 않은 경우 먼저 `select:AskUserQuestion`으로 `ToolSearch` 호출), Codex의 `request_user_input`, Gemini의 `ask_user`, Pi의 `ask_user` (`pi-ask-user` 확장 기능 필요). 하네스에 차단 도구가 존재하지 않거나 호출 오류가 발생하는 경우(예: Codex 편집 모드)에만 채팅에서 옵션을 제시하세요. 스키마 로드가 필요하다는 이유로 채팅을 사용해서는 안 됩니다. 절대로 질문을 건너뛰지 마세요.

```
브라우저 테스트가 실행되는 것을 직접 확인하시겠습니까?

1. Headed (보기) - 가시적인 브라우저 창을 열어 테스트 실행 과정을 확인합니다.
2. Headless (빠름) - 배경에서 실행됩니다. 가시적이지 않지만 더 빠릅니다.
```

선택 사항을 저장하고 사용자가 1번 옵션을 선택한 경우 `--headed` 플래그를 사용합니다.

### 3. 테스트 범위 결정

**PR 번호가 제공된 경우:**
```bash
gh pr view [number] --json files -q '.files[].path'
```

**'current' 또는 비어 있는 경우:**
```bash
git diff --name-only main...HEAD
```

**브랜치 이름이 제공된 경우:**
```bash
git diff --name-only main...[branch]
```

### 4. 파일을 경로(Route)에 매핑

변경된 파일을 테스트 가능한 경로에 매핑합니다:

| 파일 패턴 | 경로(Route) |
|-------------|----------|
| `app/views/users/*` | `/users`, `/users/:id`, `/users/new` |
| `app/controllers/settings_controller.rb` | `/settings` |
| `app/javascript/controllers/*_controller.js` | 해당 Stimulus 컨트롤러를 사용하는 페이지 |
| `app/components/*_component.rb` | 해당 컴포넌트를 렌더링하는 페이지 |
| `app/views/layouts/*` | 모든 페이지 (최소한 홈페이지 테스트) |
| `app/assets/stylesheets/*` | 주요 페이지의 시각적 회귀(visual regression) |
| `app/helpers/*_helper.rb` | 해당 헬퍼를 사용하는 페이지 |
| `src/app/*` (Next.js) | 대응하는 경로 |
| `src/components/*` | 해당 컴포넌트를 사용하는 페이지 |

매핑을 기반으로 테스트할 URL 목록을 작성합니다.

### 5. 빈 포트 감지 및 점유

**파이프라인 모드 전용 (`mode:pipeline`):** LFG 또는 다른 자동화된 파이프라인에서 호출된 경우, 실제로 비어 있는 포트를 찾아야 합니다. 여러 에이전트가 동일한 머신에서 병렬로 실행될 수 있으므로 3000번 포트가 사용 가능하다고 가정하지 마세요.

**수동 모드 (`mode:pipeline` 아님):** 선호하는 포트를 그대로 사용합니다. 사용자가 자신의 서버를 제어하므로 대안을 검색하지 마세요.

다음 우선순위에 따라 선호하는 포트를 결정합니다:

1. **명시적 인수** — 사용자가 `--port 5000`을 전달한 경우 이를 직접 사용합니다 (빈 포트 스캔 건너뜀).
2. **프로젝트 지침** — `AGENTS.md`, `CLAUDE.md` 또는 기타 지침 파일에서 포트 참조를 확인합니다.
3. **package.json** — dev/start 스크립트에서 `--port` 플래그를 확인합니다.
4. **환경 파일** — `.env`, `.env.local`, `.env.development`에서 `PORT=`를 확인합니다.
5. **Default** — `3000`으로 대체합니다.

**파이프라인 모드**에서는 선호하는 포트가 비어 있는지 확인하고, 그렇지 않으면 더 높은 번호의 포트를 스캔합니다. **수동 모드**에서는 선호하는 포트를 직접 사용합니다.

```bash
# 1단계: 선호하는 포트 결정
PORT="${EXPLICIT_PORT:-}"
if [ -z "$PORT" ]; then
  PORT=$(grep -Eio '(port\s*[:=]\s*|localhost:)([0-9]{4,5})' AGENTS.md 2>/dev/null | grep -Eo '[0-9]{4,5}' | head -1)
fi
if [ -z "$PORT" ]; then
  PORT=$(grep -Eio '(port\s*[:=]\s*|localhost:)([0-9]{4,5})' CLAUDE.md 2>/dev/null | grep -Eo '[0-9]{4,5}' | head -1)
fi
if [ -z "$PORT" ]; then
  PORT=$(grep -Eo '\-\-port[= ]+[0-9]{4,5}' package.json 2>/dev/null | grep -Eo '[0-9]{4,5}' | head -1)
fi
if [ -z "$PORT" ]; then
  PORT=$(grep -h '^PORT=' .env .env.local .env.development 2>/dev/null | tail -1 | cut -d= -f2)
fi
PORT="${PORT:-3000}"

# 2단계 (파이프라인 모드 전용): 빈 포트 스캔
if [ "${PIPELINE_MODE}" = "1" ]; then
  find_free_port() {
    local p=$1
    while lsof -i ":$p" -sTCP:LISTEN -t >/dev/null 2>&1; do
      p=$((p + 1))
    done
    echo $p
  }
  PORT=$(find_free_port "$PORT")
fi
echo "개발 서버 포트 사용: $PORT"
```

`mode:pipeline` 인수가 있는 경우 쉘에서 `PIPELINE_MODE=1`을 설정합니다.

### 6. 실행 중이 아니면 개발 서버 시작 후 확인

**파이프라인 모드 전용:** `$PORT`에서 수신 대기 중인 서버가 없으면 백그라운드에서 자동으로 서버를 시작합니다. 수동 모드에서는 사용자에게 알리고 중단합니다.

```bash
if lsof -i ":${PORT}" -sTCP:LISTEN -t >/dev/null 2>&1; then
  echo "포트 ${PORT}에서 서버가 이미 실행 중입니다."
else
  if [ "${PIPELINE_MODE}" = "1" ]; then
    # 파이프라인에서 자동 시작 — 이 프로젝트에 적합한 명령 선택
    echo "포트 ${PORT}에서 개발 서버를 시작하는 중..."
    if [ -f "bin/dev" ]; then
      PORT=${PORT} bin/dev > /tmp/dev-server-${PORT}.log 2>&1 &
    elif [ -f "bin/rails" ]; then
      bin/rails server -p ${PORT} > /tmp/dev-server-${PORT}.log 2>&1 &
    elif [ -f "package.json" ]; then
      PORT=${PORT} npm run dev > /tmp/dev-server-${PORT}.log 2>&1 &
    fi
    # 서버가 준비될 때까지 최대 30초 대기
    for i in $(seq 1 30); do
      lsof -i ":${PORT}" -sTCP:LISTEN -t >/dev/null 2>&1 && break
      sleep 1
    done
    if ! lsof -i ":${PORT}" -sTCP:LISTEN -t >/dev/null 2>&1; then
      echo "30초 내에 서버가 시작되지 않았습니다. 마지막 출력:"
      tail -20 /tmp/dev-server-${PORT}.log 2>/dev/null
      exit 1
    fi
  else
    # 수동 모드 — 사용자에게 서버 시작 요청
    echo "포트 ${PORT}에서 서버가 실행 중이지 않습니다."
    echo ""
    echo "개발 서버를 시작해 주세요:"
    echo "  Rails: bin/dev  또는  rails server -p ${PORT}"
    echo "  Node/Next.js: npm run dev"
    echo "  사용자 지정 포트: --port <포트번호>와 함께 이 스킬을 다시 실행하세요."
    exit 0
  fi
fi

agent-browser open http://localhost:${PORT}
agent-browser snapshot -i
```

### 7. 영향 받는 각 페이지 테스트

영향을 받는 각 경로에 대해:

**이동 및 스냅샷 캡처:**
```bash
agent-browser open "http://localhost:${PORT}/[route]"
agent-browser snapshot -i
```

**Headed 모드의 경우:**
```bash
agent-browser --headed open "http://localhost:${PORT}/[route]"
agent-browser --headed snapshot -i
```

**주요 요소 확인:**
- `agent-browser snapshot -i`를 사용하여 ref가 포함된 대화형 요소를 가져옵니다.
- 페이지 제목/헤더 존재 여부
- 기본 콘텐츠 렌더링 확인
- 오류 메시지 표시 여부
- 양식에 예상되는 필드 확인

**중요 상호작용 테스트:**
```bash
agent-browser click @e1
agent-browser snapshot -i
```

**스크린샷 촬영:**
```bash
agent-browser screenshot page-name.png
agent-browser screenshot --full page-name-full.png
```

### 8. 수동 확인 (필요한 경우)

외부 상호작용이 필요한 흐름을 테스트할 때는 수동 입력을 위해 일시 중지합니다:

| 흐름 유형 | 요청 내용 |
|-----------|-------------|
| OAuth | "[공급업체]로 로그인하여 작동하는지 확인해 주세요" |
| 이메일 | "테스트 이메일 수신함을 확인하고 수신 여부를 알려주세요" |
| 결제 | "샌드박스 모드에서 테스트 구매를 완료해 주세요" |
| SMS | "SMS 코드를 받았는지 확인해 주세요" |
| 외부 API | "[서비스] 통합이 작동하는지 확인해 주세요" |

사용자에게 묻습니다 (플랫폼의 질문 도구 사용 또는 번호가 매겨진 옵션 제시 후 대기):

```
수동 확인 필요

이 테스트는 [흐름 유형]을 포함합니다. 다음을 수행해 주세요:
1. [수행할 작업]
2. [확인할 내용]

정상적으로 작동했습니까?
1. 예 - 테스트 계속 진행
2. 아니요 - 문제 설명
```

### 9. 실패 처리

테스트 실패 시:

1. **실패 기록:**
   - 오류 상태 스크린샷: `agent-browser screenshot error.png`
   - 정확한 재현 단계 기록

2. **진행 방식 질문:**

   ```
   테스트 실패: [경로]

   문제: [설명]
   콘솔 오류: [있는 경우]

   어떻게 진행할까요?
   1. 지금 수정 - 실패한 테스트를 디버깅하고 수정합니다.
   2. 건너뛰기 - 다른 페이지 테스트를 계속 진행합니다.
   ```

3. **"지금 수정" 선택 시:** 조사하고, 수정을 제안하고, 적용한 뒤 실패한 테스트를 다시 실행합니다.
4. **"건너뛰기" 선택 시:** 건너뜀으로 기록하고 계속 진행합니다.

### 10. 테스트 요약

모든 테스트가 완료되면 요약을 제시합니다:

```markdown
## 브라우저 테스트 결과

**테스트 범위:** PR #[번호] / [브랜치 이름]
**서버:** http://localhost:${PORT}

### 테스트된 페이지: [개수]

| 경로 | 상태 | 메모 |
|-------|--------|-------|
| `/users` | 통과 | |
| `/settings` | 통과 | |
| `/dashboard` | 실패 | 콘솔 오류: [메시지] |
| `/checkout` | 건너뜀 | 결제 자격 증명 필요 |

### 콘솔 오류: [개수]
- [발견된 오류 목록]

### 수동 확인: [개수]
- OAuth 흐름: 확인됨
- 이메일 발송: 확인됨

### 실패: [개수]
- `/dashboard` - [문제 설명]

### 결과: [통과 / 실패 / 부분 통과]
```

## 빠른 사용 예시

```bash
# 현재 브랜치 변경 사항 테스트 (포트 자동 감지)
/ce-test-browser

# 특정 PR 테스트
/ce-test-browser 847

# 특정 브랜치 테스트
/ce-test-browser feature/new-dashboard

# 특정 포트에서 테스트
/ce-test-browser --port 5000
```

## agent-browser CLI 참조

모든 명령어를 확인하려면 `agent-browser --help`를 실행하세요.

주요 명령어:

```bash
# 이동
agent-browser open <url>           # URL로 이동
agent-browser back                 # 뒤로 가기
agent-browser close                # 브라우저 닫기

# 스냅샷 (요소 ref 가져오기)
agent-browser snapshot -i          # ref(@e1, @e2 등)가 포함된 대화형 요소
agent-browser snapshot -i --json   # JSON 출력

# 상호작용 (스냅샷의 ref 사용)
agent-browser click @e1            # 요소 클릭
agent-browser fill @e1 "text"      # 입력란 채우기
agent-browser type @e1 "text"      # 내용을 지우지 않고 입력
agent-browser press Enter          # 키 누르기

# 스크린샷
agent-browser screenshot out.png       # 뷰포트 스크린샷
agent-browser screenshot --full out.png # 전체 페이지 스크린샷

# Headed 모드 (가시적 브라우저)
agent-browser --headed open <url>      # 가시적 브라우저로 열기
agent-browser --headed click @e1       # 가시적 브라우저에서 클릭

# 대기
agent-browser wait @e1             # 요소 대기
agent-browser wait 2000            # 밀리초 단위 대기
```
