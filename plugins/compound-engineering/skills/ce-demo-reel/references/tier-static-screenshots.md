# 계층 (Tier): 정적 스크린샷 (Static Screenshots)

개별 PNG 스크린샷을 캡처합니다. 애니메이션이나 스티칭은 없습니다.

**최적 용도:** 다른 도구를 사용할 수 없을 때의 폴백, 라이브러리 데모, 또는 애니메이션이 가치를 더하지 않는 기능.
**출력물:** PNG 파일
**레이블:** "Screenshots"
**필수 도구:** 가변적 (웹은 agent-browser, CLI는 silicon, 또는 네이티브 스크린샷)

**비밀 정보 규칙은 여기서도 적용됩니다.** 브라우저 캡처의 경우 DevTools를 열지 말고, 토큰이 포함된 URL을 캡처하지 마십시오. 마스킹되지 않은 자격 증명이 표시되는 페이지도 피하십시오. CLI 캡처의 경우, 환경 변수 덤프, `--api-key` 플래그 값, 에러 트레이스의 인증 헤더 등이 이미 제거된 깨끗한 출력을 렌더링하십시오. 업로드 전 각 PNG를 스캔하십시오. 자격 증명 같은 것이 보인다면 폐기하고 다시 캡처하십시오.

## 프로젝트 유형별 캡처

### 웹 앱 또는 데스크톱 앱 (agent-browser 사용 가능 시)

`agent-browser`가 설치되지 않은 경우 사용자에게 알리십시오: "`agent-browser` is not installed. Run `/ce-setup` to install required dependencies." 그 후 아래의 CLI 또는 폴백 섹션으로 건너뛰십시오.

```bash
agent-browser open [URL]
```

```bash
agent-browser wait 2000
```

```bash
agent-browser screenshot [RUN_DIR]/screenshot-01.png
```

1-3개의 스크린샷을 캡처하십시오: 이전 상태, 기능 작동 중, 결과 상태.

### CLI 도구 (silicon 사용 가능 시)

명령을 실행하고 출력을 텍스트 파일로 캡처한 뒤 silicon으로 렌더링하십시오:

```bash
silicon [RUN_DIR]/output.txt -o [RUN_DIR]/screenshot-01.png --theme Dracula -l bash --pad-horiz 20 --pad-vert 20
```

### CLI 도구 (silicon 미사용 시)

명령을 실행하고 원본 터미널 출력을 캡처하십시오. 이미지 대신 PR 설명에 출력 내용을 코드 블록으로 포함하십시오. 레이블은 절대 "Screenshot"이 아닌 "Terminal output"으로 지정하십시오.

### 라이브러리

새로운 API를 사용하는 예제 코드를 실행하십시오. 위와 동일하게 출력을 캡처하십시오 (사용 가능 시 silicon, 미사용 시 코드 블록).

## 업로드

각 PNG는 개별적으로 업로드됩니다. 각 파일에 대해 `references/upload-and-approval.md`로 진행하거나, 모두 업로드한 뒤 승인을 위해 한 번에 제시하십시오.

여러 개의 스크린샷을 사용할 경우, 마크다운 임베드는 여러 줄의 이미지 코드를 사용합니다:

```markdown
## Screenshots

![Before](url-1)
![After](url-2)
```
