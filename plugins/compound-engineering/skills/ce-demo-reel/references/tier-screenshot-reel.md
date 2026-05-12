# 계층 (Tier): Screenshot Reel

텍스트에서 스타일이 적용된 터미널 프레임을 렌더링하고 애니메이션 GIF로 결합합니다. 각 프레임은 CLI 데모의 한 단계(명령어 + 출력)를 보여줍니다.

**최적 용도:** 불연속적인 단계로 보여주는 CLI 도구 (명령어 -> 출력 -> 다음 명령어 -> 출력). VHS가 따옴표나 특수 문자를 제대로 처리하지 못할 때도 유용합니다.
**출력물:** GIF (silicon PNG를 ffmpeg으로 스티칭)
**레이블:** "Demo"
**필수 도구:** silicon, ffmpeg

## 1단계: 데모 내용 작성

프레임 사이에 `---` 구분자를 둔 텍스트 파일을 생성하십시오. 각 프레임은 한 단계의 터미널 상태를 보여줍니다:

`[RUN_DIR]/demo-steps.txt`에 작성:

```
$ your-cli-command --flag value
Output line 1
Output line 2
Success: feature works correctly
---
$ your-cli-command --another-flag
Different output showing another aspect
Result: 42 items processed
---
$ your-cli-command --verify
All checks passed
```

**팁:**
- 사용자가 무엇을 타이핑하는지 보여주기 위해 `$` 프롬프트를 포함하십시오.
- 가독성을 위해 각 프레임을 약 80자 이내의 너비로 유지하십시오.
- 3-5개 프레임이 이상적입니다 -- 스토리를 전달하기에 충분하면서도 GIF 크기가 너무 커지지 않습니다.
- silicon의 기본 폰트가 렌더링할 수 없는 유니코드 문자(체크 표시, 화려한 화살표 등)는 제거하십시오.

**데모 텍스트에 절대 비밀 정보를 작성하지 마십시오:**
- 실제 실행 결과에서 복사했더라도 실제 자격 증명, API 키, 토큰, 세션 ID를 프레임에 붙여넣지 마십시오.
- `sk-xxxxxxxxx`와 같이 가짜처럼 보이는 자격 증명으로 대체하지도 마십시오 -- 이는 오해의 소지가 있는 아티팩트를 생성합니다. 대신 값이 나타나지 않도록 환경 변수 *이름*을 사용하도록 명령어를 재작성하거나(예: `your-cli --api-key "$API_KEY"`), 비밀 정보가 필요 없는 다른 시나리오를 시연하십시오.
- 샘플 출력 라인에 토큰, 인증 헤더가 포함된 에러 트레이스, 또는 기타 자격 증명이 포함될 경우, 해당 라인을 편집하거나 다른 시나리오를 선택하십시오 -- 이를 렌더링하지 마십시오.

## 2단계: 프레임 파일로 분할

데모 내용을 `---` 라인을 기준으로 별도의 텍스트 파일로 분할하십시오 (프레임당 하나):

- `[RUN_DIR]/frame-001.txt`
- `[RUN_DIR]/frame-002.txt`
- `[RUN_DIR]/frame-003.txt`
- 기타 등등.

## 3단계: 렌더링 및 스티칭

캡처 파이프라인 스크립트를 사용하여 각 텍스트 프레임을 silicon으로 렌더링하고 한 번의 호출로 애니메이션 GIF로 스티칭하십시오:

```bash
python3 scripts/capture-demo.py screenshot-reel --output [RUN_DIR]/demo.gif --duration 2.5 --text [RUN_DIR]/frame-001.txt [RUN_DIR]/frame-002.txt [RUN_DIR]/frame-003.txt
```

이 스크립트는 silicon 렌더링, 크기 정규화, two-pass palette 생성, 그리고 GIF가 제한을 초과할 경우의 자동 프레임 축소를 처리합니다. 기본 지속 시간은 프레임당 2.5초입니다 (터미널 프레임은 브라우저보다 읽기 빠르므로 더 짧게 설정됨).

**스크립트 실패 시** (silicon 렌더링 에러, 스티칭 에러, 빈 출력): 정적 스크린샷 계층으로 폴백하십시오. PR 설명에 원본 터미널 출력을 코드 블록으로 포함하십시오. 레이블은 "Screenshot"이 아닌 "Terminal output"으로 지정하십시오.

## 4단계: 정리

개별 PNG와 텍스트 파일을 제거하십시오. 업로드용 최종 GIF만 유지합니다.

`references/upload-and-approval.md`로 진행하십시오.
