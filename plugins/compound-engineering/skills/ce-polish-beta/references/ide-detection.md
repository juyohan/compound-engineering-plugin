# 브라우저 핸드오프를 위한 IDE 감지

Polish는 사용자가 문맥 전환 없이 테스트할 수 있도록 실행 중인 개발 서버 URL을 IDE의 임베디드 브라우저로 전달하려고 시도합니다. 감지는 최선형(best-effort) 방식으로 이루어지며, 실패할 경우 인터랙티브 요약에 URL을 출력하는 방식으로 폴백합니다.

## 감지 순서

다음 순서대로 환경 변수를 조사하고 첫 번째로 일치하는 항목에서 멈춥니다. 앞쪽 항목일수록 더 구체적이며, 뒤쪽 항목은 일반적인 폴백입니다.

| 순서 | 신호 | IDE | 핸드오프 방식 |
|-------|--------|-----|----------------|
| 1 | `CLAUDE_CODE` 환경 변수 설정됨 (값 무관) | Claude Code 데스크톱 | 클릭 가능한 힌트로 `claude-code://browser?url=http://localhost:<port>`를 출력합니다. Claude Code 데스크톱 앱은 `claude-code://` URL을 가로챕니다. |
| 2 | `CURSOR_TRACE_ID` 환경 변수 설정됨 | Cursor | 사용자의 버전에서 Cursor의 URL 스킴이 안정적인 경우 `cursor://anysphere.cursor-retrieval/open?url=...`을 내보냅니다. 그렇지 않은 경우 Cursor의 단순 브라우저(simple-browser) 뷰에서 열라는 메모와 함께 URL을 출력합니다. |
| 3 | `TERM_PROGRAM=vscode`이며 Cursor/Claude Code 신호 없음 | 일반 VS Code | `Open in VS Code: Ctrl+Shift+P → "Simple Browser: Show" → paste URL` 힌트와 함께 URL을 출력합니다. |
| 4 | 위 사항에 해당 없음 | 터미널 / 알 수 없는 IDE | URL을 출력합니다. 핸드오프를 시도하지 않습니다. |

## 왜 더 정교한 방식 대신 환경 변수 조사를 사용하는가?

- 환경 변수는 플랫폼 간 호환성이 높습니다 (macOS, Linux, Windows/WSL).
- "실패 시 개방(fail open)" 방식입니다. 조사가 아무것도 반환하지 않더라도 polish는 여전히 작동합니다.
- IDE API나 소켓 연결이 필요하지 않습니다.
- 추측 없이 "이 쉘이 알려진 IDE 내부에서 실행 중인가"를 인코딩합니다.

## Codex 및 기타 플랫폼

Codex (Claude Agent SDK, Gemini CLI 등)는 아직 임베디드 브라우저 핸드오프를 노출하지 않습니다. 이러한 플랫폼의 경우 polish는 터미널 분기로 폴백하여 URL을 출력합니다. 컨벤션이 생기면 위의 감지 테이블에 새로운 행을 추가하십시오.

## 감지 실패는 치명적이지 않음

환경 변수 조사가 실패하거나 모호한 결과를 반환하더라도, polish는 URL을 그대로 출력하고 계속 진행합니다. 이 시점에는 개발 서버가 이미 실행 중이므로, 사용자는 언제든지 URL을 복사하여 아무 브라우저에나 붙여넣을 수 있습니다. IDE 핸드오프는 편의 기능일 뿐 필수 관문은 아닙니다.

## 조사 패턴 (참고)

이 스킬은 쉘 스크립트를 통하지 않고 인라인으로 이러한 조사를 수행합니다 (상태 없음, 파싱 없음, 일회성 읽기). 전형적인 사용법:

```
if [ -n "${CLAUDE_CODE:-}" ]; then
  IDE="claude-code"
elif [ -n "${CURSOR_TRACE_ID:-}" ]; then
  IDE="cursor"
elif [ "${TERM_PROGRAM:-}" = "vscode" ]; then
  IDE="vscode"
else
  IDE="none"
fi
```

서로 다른 변수 사이에 `||`를 사용하여 조사를 연결하지 마십시오. 누락된 환경 변수는 "에러"가 아니라 "신호 없음"으로 해석되어야 합니다. `set -u` 환경에서 `${VAR:-}` 형태의 기본값 빈 문자열 패턴은 필수입니다.
