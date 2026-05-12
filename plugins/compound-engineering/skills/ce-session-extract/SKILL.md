---
name: ce-session-extract
description: "지정된 경로의 단일 세션 파일에서 대화 골격 또는 오류 신호를 추출합니다. 세션 조사 에이전트가 심층 분석할 세션을 선택한 후 호출하며, 사용자의 직접 쿼리용이 아닙니다."
user-invocable: false
context: fork
allowed-tools:
  - gem
---

# 세션 추출 (Session extract)

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


에이전트용 프리미티브(primitive)입니다. Claude Code, Codex 또는 Cursor 세션 파일 하나에서 필터링된 내용(대화 골격 또는 오류 신호)을 추출합니다.

이 스킬은 에이전트가 수 메가바이트에 달하는 세션 파일을 컨텍스트로 직접 읽어 들이지 않도록 하기 위해 존재합니다. `scripts/` 아래의 스크립트들이 JSONL 형식에 대한 지식을 보유하며, 서사적으로 읽기 쉬운 요약본(digest)을 생성합니다.

## 인자 (Arguments)

공백으로 구분된 위치 인자:

1. `<file>` — 세션 JSONL 파일의 절대 경로 (일반적으로 `ce-session-inventory`가 반환하는 `file` 값).
2. `<mode>` — `skeleton` 또는 `errors`.
3. `<limit>` *(선택 사항)* — 출력을 N 라인으로 제한하기 위한 `head:N` 또는 `tail:N` (예: `head:200`). 전체 추출 결과를 반환하려면 생략하세요.

## 실행 (Execution)

**Skeleton 모드** — 사용자 메시지, 어시스턴트 텍스트 및 축약된 도구 호출 요약의 서사적 구성:

```bash
cat <file> | python3 scripts/extract-skeleton.py
```

**Errors 모드** — 오류 신호만 추출:

```bash
cat <file> | python3 scripts/extract-errors.py
```

`<limit>`가 `head:N`인 경우 `head -n N`으로 파이프를 연결합니다. `tail:N`인 경우 `tail -n N`으로 연결합니다. 제한(limit)은 항상 파이썬 스크립트 실행 *이후*에 적용해야 합니다. `_meta` 라인이 마지막에 출력되므로 `head` 제한 시 유실될 수 있으나, 호출자가 명시적으로 `head` 제한을 요청한 경우 이는 허용됩니다.

raw stdout을 그대로 반환하세요. 내용을 바꾸어 말하거나 주석을 달거나 요약하지 마세요. 여러 세션에 걸친 요약은 호출자가 수행합니다.

## 각 모드 반환 내용 (What each mode returns)

### Skeleton

블록당 하나의 논리적 이벤트가 포함된 서사적 출력이며, `---`로 구분됩니다:

- 사용자 메시지 (텍스트 전용, 도구 결과 없음, 프레임워크 래퍼 태그 제거됨)
- 어시스턴트 텍스트 (사고/추론 블록 제외 — 이는 내부용이거나 암호화되어 있음)
- 도구 호출 요약: 3회 이상 연속된 동일한 도구 호출은 축약됨 (예: `[tools] 5x Read (file1, file2, +3 more) -> all ok`)

`_meta` 라인으로 종료됨: `{"_meta": true, "lines": N, "parse_errors": N, "user": N, "assistant": N, "tool": N}`.

### Errors

오류당 한 줄씩 출력되며, `---`로 구분됩니다:

- Claude Code: `is_error: true`인 도구 결과
- Codex: 종료 코드가 0이 아니거나 stderr가 비어 있지 않은 `exec_command_end` 이벤트
- Cursor: 항상 비어 있음 — Cursor 에이전트 트랜스크립트는 도구 결과를 기록하지 않음

`_meta` 라인으로 종료됨: `{"_meta": true, "lines": N, "parse_errors": N, "errors_found": N}`.

## 오류 처리 (Error handling)

파일을 읽을 수 없는 경우 호출자에게 오류가 그대로 전달되도록 하세요. `_meta`에서 `parse_errors > 0`을 보고하는 경우 출력을 그대로 반환하세요. 부분적인 추출 결과도 여전히 유용하며, 검색 범위를 넓히거나 더 심층적으로 분석할지 여부는 호출자가 결정합니다.
