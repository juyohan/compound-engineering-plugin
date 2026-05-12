# 업로드 및 승인 (Upload and Approval)

사용자가 검토할 수 있도록 임시 프리뷰를 업로드한 뒤, 사용자 선택에 따라 영구 호스팅으로 승격하거나 로컬에 저장합니다.

## 1단계: 프리뷰 업로드 (임시)

임시 1시간 프리뷰를 위해 증거 파일(GIF 또는 PNG)을 litterbox에 업로드하십시오:

```bash
python3 scripts/capture-demo.py preview [ARTIFACT_PATH]
```

출력의 마지막 라인이 프리뷰 URL입니다 (예: `https://litter.catbox.moe/abc123.gif`). 이 URL은 1시간 후에 만료되므로 정리가 필요하지 않습니다.

여러 파일(정적 스크린샷 계층)의 경우 각 파일을 별도로 업로드하십시오.

**업로드 실패 시** 재시도 후에도 실패한다면, 사용자가 검토할 수 있도록 플랫폼 파일 열기 도구(macOS는 `open`, Linux는 `xdg-open`)를 사용하여 로컬 파일을 여십시오. 목적지 선택 질문에 URL 대신 로컬 경로를 포함하십시오.

## 2단계: 목적지 선택

사용자에게 프리뷰 URL을 제시하고 증거를 어떻게 처리할지 물으십시오. 플랫폼의 차단형 질문 도구를 사용하십시오: Claude Code의 `AskUserQuestion` (스키마가 로드되지 않았다면 먼저 `select:AskUserQuestion`으로 `ToolSearch` 호출), Codex의 `request_user_input`, Gemini의 `ask_user`, Pi의 `ask_user` (pi-ask-user 확장 필요). 하네스에 차단 도구가 없거나 호출 에러가 발생하는 경우(예: Codex 편집 모드)에만 채팅으로 옵션을 제시하십시오 -- 스키마 로드가 필요하다는 이유로 건너뛰지 마십시오. 절대 질문을 조용히 생략하지 마십시오.

**질문:** "Evidence preview (1h link): [PREVIEW_URL]. Where should the evidence go?"

**옵션:**
1. **Upload to catbox (public URL)** -- PR 임베딩을 위해 영구 호스팅으로 승격
2. **Save locally** -- 안정적인 OS 임시 경로(/tmp/compound-engineering/ce-demo-reel/)에 저장
3. **Recapture** -- 변경 사항에 대한 지침 제공
4. **Proceed without evidence** -- 증거를 null로 설정하고 진행

질문 도구를 사용할 수 없는 경우(헤드리스/백그라운드 모드), 번호가 매겨진 옵션을 제시하고 사용자의 응답을 기다린 후 진행하십시오.

### "Upload to catbox (public URL)" 선택 시

3단계: 영구 호스팅으로 승격으로 진행하십시오.

### "Save locally" 선택 시

3단계 b: 로컬 저장으로 진행하십시오.

### "Recapture" 선택 시

계층 실행 단계로 돌아가십시오. 사용자의 지침에 따라 다음 캡처 시도에서 무엇을 변경할지 결정합니다. 다시 캡처한 후 새 프리뷰를 업로드하고 목적지 선택을 반복하십시오.

### "Proceed without evidence" 선택 시

증거를 null로 설정하고 진행하십시오. 프리뷰 링크는 자동으로 만료됩니다.

## 3단계: 영구 호스팅으로 승격

사용자가 "Upload to catbox"를 선택하면 영구 catbox 호스팅으로 업로드하십시오. 이 명령은 프리뷰 URL(권장) 또는 로컬 파일 경로(폴백)를 수락합니다:

```bash
python3 scripts/capture-demo.py upload [PREVIEW_URL or ARTIFACT_PATH]
```

1단계에서 프리뷰 URL이 생성되었다면 이를 전달하십시오 -- catbox는 재업로드 없이 litterbox에서 직접 복사합니다. 1단계에서 로컬 검토로 폴백했다면(프리뷰 URL 없음) 로컬 아티팩트 경로를 전달하십시오.

출력의 마지막 라인이 영구 URL입니다 (예: `https://files.catbox.moe/abc123.gif`). 출력물에 프리뷰 URL이 아닌 이 영구 URL을 사용하십시오.

여러 파일의 경우 각각 별도로 승격하십시오.

## 3단계 b: 로컬 저장

사용자가 "Save locally"를 선택하면 파이프라인 스크립트를 사용하여 아티팩트를 기본 OS 임시 경로에 저장하십시오:

```bash
python3 scripts/capture-demo.py save-local --file [ARTIFACT_PATH] --branch [BRANCH_NAME]
```

`[BRANCH_NAME]`은 `git branch --show-current` 또는 SKILL.md의 0단계에서 발견된 PR 컨텍스트를 통해 결정하십시오.

출력의 마지막 라인이 저장된 파일의 절대 경로입니다. 출력물에 이 경로를 사용하십시오.

여러 파일(정적 스크린샷 계층)의 경우 각각 별도로 저장하십시오.

**저장 실패 시** (권한 거부, 디스크 꽉 참 등) 에러를 보고하고 다시 시도하거나 catbox 업로드(3단계)로 폴백할지 물으십시오.

## 4단계: 결과 반환

SKILL.md 출력 섹션에 정의된 구조화된 결과(`Tier`, `Description`, 그리고 `URL` 또는 `Path`)를 반환하십시오. 호출자는 이 증거를 PR 설명 형식에 맞게 구성합니다. ce-demo-reel은 마크다운을 직접 생성하지 않습니다.

## 5단계: 정리

`[RUN_DIR]` 작업 디렉토리와 모든 임시 파일을 제거하십시오. 아무것도 보존하지 마십시오 -- 증거는 영구 URL에 있거나 로컬 저장 경로로 복사되었습니다.
