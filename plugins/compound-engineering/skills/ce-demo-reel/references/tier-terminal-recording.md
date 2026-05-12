# 계층 (Tier): Terminal Recording

VHS (charmbracelet/vhs)를 사용하여 터미널 세션을 기록하고 GIF 데모를 생성합니다.

**최적 용도:** CLI 도구, 스크립트, 상호작용이나 움직임(타이핑, 스트리밍 출력, 점진적 렌더링)이 있는 커맨드라인 기능.
**출력물:** GIF (VHS에서 직접 생성)
**레이블:** "Demo"
**필수 도구:** vhs

## 1단계: 녹화 계획

.tape 파일을 생성하기 전 다음에 대해 결정하십시오:

- **실행할 명령어(들)** -- 테스트 명령이 아닌 실제 제품 명령어입니다. "npm test를 실행했습니다"는 테스트 증거이지 데모가 아닙니다.
- **예상 출력** -- 명령이 성공했을 때 터미널에 표시되어야 할 내용입니다.
- **터미널 크기** -- 가장 긴 출력 라인을 수용할 수 있을 만큼 넓고, 스크롤을 피할 수 있을 만큼 높아야 합니다.
- **타이밍** -- 총 5-10초를 목표로 합니다. 출력이 렌더링될 수 있도록 각 명령 뒤에 충분한 sleep을 둡니다.
- **비밀 정보 노출 지점** -- 자격 증명이 노출될 수 있는 모든 단계: env exports, `source .env`, `printenv`/`env`/`set`, `--api-key`/`--token` 플래그를 사용하는 CLI, verbose/debug 플래그, 출력이나 에러 트레이스에 토큰을 에코하는 명령, 환경 변수가 삽입된 `$VAR` 세그먼트가 있는 쉘 프롬프트 등. 실제 자격 증명은 `.tape` 상단의 `Hide` 블록 내에 설정하고, 블록 끝에 `clear`를 실행하여 버퍼를 비운 뒤 `Show` 하십시오. 개인 도트파일, 캐시된 CLI 토큰, 환경 변수가 삽입된 프롬프트가 유출되지 않도록 `Hide` 내에서 깨끗한 `HOME`(`export HOME=$(mktemp -d)`)을 사용하십시오.

## 2단계: .tape 파일 생성

`[RUN_DIR]/demo.tape`에 VHS 테이프 파일을 작성하십시오:

```tape
Output [RUN_DIR]/demo.gif

Set FontSize 16
Set Width 800
Set Height 500
Set Theme "Catppuccin Mocha"
Set TypingSpeed 40ms

# 숨겨진 전처리(Hide): 깨끗한 HOME 설정, 실제 비밀 정보 설정, 유출될 수 있는 모든 설정.
# 이 명령들은 실제로 실행되지만 GIF에는 절대 나타나지 않습니다.
# `clear`는 Show가 깨끗한 화면에서 시작되도록 버퍼를 비웁니다.
Hide
Type "export HOME=$(mktemp -d)"
Enter
Type "export API_KEY='real-secret-value'"
Enter
Type "cd /path/to/project"
Enter
Type "clear"
Enter
Show

# 보여지는 데모(Show): 명령어들은 위에서 설정된 환경 변수를 사용하지만, 절대 다시 export하거나
# 에코하거나 출력하지 않습니다. 인증 메커니즘이 아닌 기능이 작동하는 모습을 보여주십시오.
Type "your-cli-command --flag value"
Enter
Sleep 3s

# 시청자가 출력을 읽을 수 있게 대기
Sleep 2s
```

**이러한 형태를 사용하는 이유:** 보여지는 명령어의 성공 자체가 자격 증명이 설정되었음을 증명하므로 인증 단계를 보여줄 필요가 없습니다. 가짜 값을 가진 `export SECRET=...`을 노출하지 마십시오: 이는 변수 이름을 유출하고 데모의 신뢰성을 떨어뜨립니다.

**주요 .tape 지시어:**
- `Output [path]` -- GIF를 작성할 위치 (첫 번째 라인이어야 함)
- `Set FontSize [14-18]` -- 가독성을 위해 크게 설정
- `Set Width/Height [pixels]` -- 내용에 맞게 조정
- `Set Theme [name]` -- "Catppuccin Mocha"나 "Dracula"가 가독성 좋은 기본값입니다
- `Set TypingSpeed [ms]` -- 30-50ms가 자연스럽습니다
- `Hide`/`Show` -- 지루한 설정 단계(cd, source, npm install 등) 건너뛰기
- `Type [text]` -- 문자를 타이핑 (실행하지 않음)
- `Enter` -- 엔터 키 입력 (타이핑된 명령 실행)
- `Sleep [duration]` -- 출력이 렌더링될 때까지 대기

**피해야 할 것:**
- 비결정적인 출력 (랜덤 ID, 실행 시마다 변하는 타임스탬프)
- 대화형 입력이 필요한 명령 (프롬프트, 비밀번호 입력)
- 화면을 벗어나는 매우 긴 출력

## 3단계: VHS 실행

캡처 파이프라인 스크립트를 사용하여 테이프 파일을 실행하고 출력을 검증하십시오:

```bash
python3 scripts/capture-demo.py terminal-recording --output [RUN_DIR]/demo.gif --tape [RUN_DIR]/demo.tape
```

스크립트는 VHS를 실행하고 출력이 존재하는지 확인하며 파일 크기를 보고합니다. GIF가 10MB를 초과하면 터미널 크기 축소(`Set Width/Height`), 녹화 시간 단축(sleep 줄이기), 또는 폰트 크기 축소를 통해 조정하고 다시 실행하십시오.

## 4단계: 품질 확인

생성된 GIF를 읽어 다음을 확인하십시오:

1. 명령어가 명확하고 가독성 있는지
2. 출력이 완전히 렌더링되었는지 (잘리지 않았는지)
3. 시연하려는 기능이 명확하게 보여지는지

**비밀 정보 스캔 (필수 관문):** GIF에서 자격 증명 물질을 스캔하십시오. 만약 발견된다면 해당 단계를 `Hide`/`Show`로 감싸거나 교체하여 폐기하고 다시 녹화하십시오. 업로드하지 말고, 블러 처리도 하지 마십시오.

**드리프트 확인:** `401 Unauthorized`, `Invalid API key`, `0 credits remaining`, 예상 데이터 대신 빈 출력 등 실패한 명령어는 대개 `Show` 이후에 가짜 값을 가진 `export SECRET=...`이 실제 환경 변수를 덮어씌웠음을 의미합니다. 비밀 정보가 `Hide` 내에서만 설정되고 다시 export되지 않도록 `.tape`를 수정하고 다시 녹화하십시오.

품질이 낮으면 .tape 파일을 수정하고 다시 녹화하십시오.

**VHS 실패 시** (크래시, 빈 GIF 생성, 또는 시연 중인 명령 실패): screenshot reel 계층으로 폴백하십시오. 동일한 명령어와 예상 출력을 텍스트 프레임으로 작성하고 silicon + ffmpeg으로 스티칭하십시오. silicon도 사용할 수 없다면 정적 스크린샷으로 폴백하십시오.

`references/upload-and-approval.md`로 진행하십시오.
