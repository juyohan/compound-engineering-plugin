# 계층 (Tier): Browser Reel

주요 UI 상태에서 3-5개의 브라우저 스크린샷을 캡처하고 애니메이션 GIF로 결합합니다.

**최적 용도:** 웹 앱, localhost 또는 CDP를 통해 접근 가능한 데스크톱 앱.
**출력물:** GIF (ffmpeg two-pass palette를 통해 스티칭된 PNG 스스크린샷)
**레이블:** "Demo"
**필수 도구:** agent-browser, ffmpeg

`agent-browser`가 설치되지 않은 경우 사용자에게 알리십시오: "`agent-browser` is not installed. Run `/ce-setup` to install required dependencies." 그 후 낮은 계층(정적 스크린샷 또는 건너뛰기)으로 폴백하십시오.

## 1단계: 애플리케이션 연결

**웹 앱의 경우** -- 개발 서버에 접근 가능한지 확인하십시오:

- `package.json`의 `scripts`에서 `dev`, `start`, `serve` 명령 확인
- `Procfile`, `Procfile.dev`, 또는 `bin/dev`가 존재하는지 확인
- Rails(`bin/rails server`) 또는 Sinatra를 위한 `Gemfile` 확인
- 일반적인 포트(3000, 5000, 8080)에서 실행 중인 프로세스 확인

서버가 실행 중이지 않다면 감지된 시작 명령을 사용자에게 알리고 시작해 달라고 요청하십시오. 자동으로 시작하지 마십시오 (환경 변수, 데이터베이스 설정 등이 필요할 수 있습니다).

사용자가 서버가 실행 중이어야 함을 확인했음에도 서버에 도달할 수 없는 경우, 정적 스크린샷 계층으로 폴백하십시오.

접근이 가능해지면 베이스 URL(예: `http://localhost:3000`)을 기록하십시오.

**Electron/데스크톱 앱의 경우** -- Chrome DevTools Protocol (CDP)을 통해 연결하십시오:

1. 일반적인 포트를 조사하여 CDP가 활성화된 상태로 앱이 이미 실행 중인지 확인하십시오:
   ```bash
   curl -s http://localhost:9222/json/version
   ```
   JSON 응답이 반환되면 앱이 준비된 것입니다 -- agent-browser를 연결하십시오:
   ```bash
   agent-browser connect 9222
   ```

2. 실행 중이지 않다면 앱을 `--remote-debugging-port`와 함께 실행해야 합니다. `package.json`에서 진입점을 감지하고(scripts의 `main` 필드나 `electron` 확인), 사용자에게 다음 명령으로 실행해 달라고 요청하십시오:
   ```
   your-electron-app --remote-debugging-port=9222
   ```
   9222 포트가 사용 중이면 9223-9230 포트를 시도하십시오.

3. CDP가 준비될 때까지 폴링하십시오 (30초 후 타임아웃):
   ```bash
   curl -s http://localhost:9222/json/version
   ```

4. agent-browser 연결:
   ```bash
   agent-browser connect 9222
   ```

**CDP의 장점:** 스크린샷이 macOS 화면 캡처가 아닌 렌더러의 프레임 버퍼에서 생성되므로, 손쉬운 사용(Accessibility)이나 화면 기록(Screen Recording) 권한이 필요하지 않습니다.

**CDP 연결 실패 시:** 정적 스크린샷 계층으로 폴백하십시오. 사용자에게 알리십시오: "Could not connect to the app via CDP. Falling back to static screenshots."

## 2단계: 스크린샷 캡처

관련 페이지로 이동하여 주요 UI 상태에서 3-5개의 스크린샷을 캡처하십시오:

1. **초기/빈 상태** -- 기능 사용 전
2. **내비게이션** -- 사용자가 기능에 도달하는 방법 (랜딩 페이지가 아닌 경우)
3. **기능 작동 중** -- 기능이 작동하는 모습을 보여주는 핵심 샷
4. **결과 상태** -- 상호작용 후 (데이터 존재, 항목 생성, 성공 메시지)
5. **상세 뷰** (선택 사항) -- 확장된 항목, 설정 패널, 모달

각 스크린샷은 부모 스킬에 의해 생성된 구체적인 `RUN_DIR`에 작성하십시오:

```bash
agent-browser open [URL]
```

```bash
agent-browser wait --load networkidle
```

```bash
agent-browser wait 1000
```

```bash
agent-browser screenshot [RUN_DIR]/frame-01-initial.png
```

**캡처 팁:**
- SPA 요소 클릭보다는 URL 내비게이션(`agent-browser open URL`)을 사용하십시오 (React/Vue/Svelte SPA에서는 클릭이 실패하는 경우가 많습니다).
- 내비게이션 후 `--load networkidle`을 기다린 뒤, 페인트 후 데이터를 가져오는 SPA를 위해 짧은 고정 버퍼를 두십시오. 고정된 `wait 2000`만으로는 부족할 수 있습니다 -- 빈 껍데기만 캡처될 수 있습니다.
- 네트워크 활동이 계속 열려 있는 페이지(websockets, long-polling)의 경우, `agent-browser wait --text "<known content>"`를 사용하여 특정 문자열이 나타날 때까지 기다리거나 `agent-browser wait --fn "<expression>"`으로 커스텀 준비 조건을 사용하십시오.
- 전체 뷰포트를 캡처하십시오 (사이드바, 헤더는 리뷰어에게 컨텍스트를 제공합니다).

**프레임 내 비밀 정보 제외:**
- DevTools, 네트워크 패널, 또는 Application/Storage를 열지 마십시오 -- 이들은 인증 헤더, 쿠키, 세션 스토리지, 토큰을 그대로 노출합니다.
- 원본 자격 증명이 표시되는 페이지(마스킹되지 않은 API 키 설정, OAuth 동의 화면, `.env` 뷰어, 결제 상세 정보 등)는 건너뛰십시오.
- 각 스크린샷 전 URL 바를 확인하십시오 -- 세션 토큰이나 자격 증명 쿼리 파라미터(`?access_token=`, `?api_key=`, `#id_token=`)가 포함되어 있다면 먼저 깨끗한 정식 URL로 이동하십시오.
- 스크린샷에 민감한 계정 식별자가 포함될 경우, 실제 로그인된 계정보다는 데모 계정이나 시드된 픽스처 데이터를 우선적으로 사용하십시오.

## 3단계: GIF로 스티칭

캡처 파이프라인 스크립트를 사용하여 프레임 크기를 정규화하고, two-pass palette로 스티칭하며, 10MB 초과 시 자동 축소하십시오:

```bash
python3 scripts/capture-demo.py stitch [RUN_DIR]/demo.gif [RUN_DIR]/frame-*.png
```

이 스크립트는 크기 정규화(ffprobe + ffmpeg padding), concat demuxer 스티칭, 팔레트 생성, 그리고 GitHub의 10MB 인라인 제한 초과 시 자동 프레임 축소를 처리합니다. 기본값은 프레임당 3초입니다. 조정이 필요한 경우:

```bash
python3 scripts/capture-demo.py stitch --duration 2.0 [RUN_DIR]/demo.gif [RUN_DIR]/frame-*.png
```

**스티칭 실패 시:** 이미 캡처된 개별 PNG를 사용하여 정적 스크린샷 계층으로 폴백하십시오. 캡처된 PNG가 없다면 실패를 보고하십시오.

## 4단계: 비밀 정보 스캔 및 정리

업로드 전, 최종 GIF 화면에 자격 증명 물질이 보이는지 검사하십시오. 만약 발견된다면 GIF를 폐기하고 문제가 되는 페이지나 상태를 프레임 밖으로 돌려 다시 캡처하십시오. 업로드하지 말고, 블러 처리도 하지 마십시오.

깨끗한 GIF가 확인되면 개별 PNG 프레임을 제거하십시오. 업로드용 최종 GIF만 유지합니다.

`references/upload-and-approval.md`로 진행하십시오.
