---
name: ce-session-inventory
description: "Claude Code, Codex, Cursor 전체에서 리포지토리의 세션 파일을 검색하고 세션 메타데이터(타임스탬프, 브랜치, cwd, 크기, 플랫폼)를 추출합니다. 세션 조사 에이전트가 호출하며, 사용자의 직접 쿼리용이 아닙니다."
user-invocable: false
context: fork
allowed-tools:
  - gem
---

# 세션 인벤토리 (Session inventory)

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


에이전트용 프리미티브(primitive)입니다. Claude Code, Codex, Cursor 전체에서 세션 파일을 검색하고 세션 메타데이터를 JSONL 형식으로 출력합니다.

이 스킬은 세션 이력을 조사하는 에이전트가 디스크 상의 세션 저장소 구조나 각 플랫폼의 JSONL 형식을 알 필요가 없도록 하기 위해 존재합니다. `scripts/` 아래의 스크립트들이 해당 지식을 보유합니다.

## 인자 (Arguments)

공백으로 구분된 위치 인자:

1. `<repo>` — 리포지토리 폴더 이름 (예: `my-project`). Claude Code 및 Cursor의 디렉토리 매칭과 Codex 세션의 CWD 필터로 사용됩니다.
2. `<days>` — 검색 기간 (일 단위, 예: `7`). 이보다 오래된 세션 파일은 건너뜁니다.
3. `<platform>` *(선택 사항)* — `claude`, `codex`, `cursor` 중 하나. 세 가지 모두 검색하려면 생략하세요.
4. `--keyword K1[,K2,...]` *(선택 사항)* — 전체 파일 내용이 쉼표로 구분된 키워드 중 최소 하나와 일치하는 세션만 필터링합니다 (대소문자 구분 없는 부분 문자열 일치). 각 세션 라인에는 `match_count` 및 `keyword_matches` ({K: N, ...}) 필드가 추가되며, `_meta` 라인에는 `files_matched` 필드가 추가됩니다. 여러 세션을 주제별 관련성에 따라 순위를 매길 때 개별 파일마다 `grep -l`을 호출하는 대신 이 옵션을 사용하세요.

## 실행 (Execution)

스킬의 `scripts/` 디렉토리에서 검색 및 메타데이터 추출 파이프라인을 실행합니다:

```bash
bash scripts/discover-sessions.sh <repo> <days> [--platform <platform>] \
  | tr '\n' '\0' \
  | xargs -0 python3 scripts/extract-metadata.py --cwd-filter <repo>
```

키워드로 필터링하려면 `extract-metadata.py` 호출 시 `--keyword K1[,K2,...]`를 추가합니다. 키워드 스캔은 헤더 메타데이터뿐만 아니라 파일 전체를 읽으므로 메타데이터 전용 실행보다 비용이 더 많이 듭니다. 기본적으로 사용하기보다는 많은 세션 중에서 특정 주제별로 후보를 선별해야 할 때 사용하세요.

raw stdout을 그대로 반환하세요. 세션당 하나의 JSON 객체와 마지막 `_meta` 라인이 출력됩니다. 호출자가 JSONL을 직접 파싱하므로 내용을 바꾸어 말하거나 형식을 변경하거나 요약하지 마세요.

검색 결과 파일이 없더라도 파이프라인은 깨끗한 `_meta` 라인(`files_processed: 0`)을 출력합니다. 이를 그대로 반환하세요.

## 출력 형식 (Output format)

각 세션 라인은 JSON 객체입니다. 플랫폼 공통 필드:

- `platform` — `claude`, `codex`, 또는 `cursor`
- `file` — 세션 JSONL 파일의 절대 경로
- `size` — 바이트 단위 파일 크기
- `ts` — 세션 시작 타임스탬프 (ISO 8601)
- `session` — 세션 식별자

플랫폼별 필드:

- Claude Code는 `branch` (git 브랜치) 및 `last_ts` (마지막 메시지 타임스탬프)를 추가합니다.
- Codex는 `cwd` (작업 디렉토리), `source`, `cli_version`, `model`, `last_ts`를 추가합니다.
- Cursor는 파일 내 타임스탬프나 메타데이터가 없습니다. `ts`는 파일의 mtime(수정 시간)에서, `session`은 포함된 디렉토리 이름에서 유도됩니다.

마지막 `_meta` 라인에는 `files_processed`, `parse_errors` 필드가 포함되며, 선택적으로 `filtered_by_cwd` (CWD 필터에 의해 제외된 Codex 세션 수) 및 `files_matched` (`--keyword` 설정 시 키워드 필터에 의해 유지된 세션 수) 필드가 포함됩니다.

`--keyword`가 설정된 경우, 각 세션 라인에는 다음 필드가 추가됩니다:

- `match_count` — 모든 키워드의 총 발생 횟수
- `keyword_matches` — 키워드별 발생 횟수, 예: `{"middleware": 4, "auth": 12}`

`match_count: 0`인 세션은 출력에서 제외됩니다.

## 오류 처리 (Error handling)

검색 스크립트에서 오류(예: 홈 디렉토리를 읽을 수 없음, 권한 오류 등)가 발생하면 호출자에게 그대로 전달되도록 하세요. git 로그, 파일 목록 또는 다른 소스로 대체하지 마세요. 이 스킬의 규약은 세션 메타데이터 제공일 뿐이며 그 이상은 아닙니다.

`_meta`에서 `parse_errors > 0`을 보고하는 경우 JSONL을 그대로 반환하세요. 부분적인 데이터를 처리할지 여부는 호출자가 결정합니다.
