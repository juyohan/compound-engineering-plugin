---
name: ce-gemini-imagegen
description: 이 스킬은 Gemini API (Nano Banana Pro)를 사용하여 이미지를 생성하고 편집할 때 사용해야 합니다. 텍스트 프롬프트로부터 이미지 생성, 기존 이미지 편집, 스타일 전송 적용, 텍스트가 포함된 로고 생성, 스티커 제작, 제품 목업(mockup) 생성 또는 모든 이미지 생성/조작 작업에 적용됩니다. 텍스트-투-이미지, 이미지 편집, 멀티 턴(multi-turn) 개선, 그리고 여러 참조 이미지로부터의 구성을 지원합니다.
allowed-tools:
  - gem
---

# Gemini 이미지 생성 (Nano Banana Pro)

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


Google의 Gemini API를 사용하여 이미지를 생성하고 편집합니다. 환경 변수 `GEMINI_API_KEY`가 설정되어 있어야 합니다.

## 기본 모델

| 모델 | 해상도 | 최적 용도 |
|-------|------------|----------|
| `gemini-3-pro-image-preview` | 1K-4K | 모든 이미지 생성 (기본값) |

**참고:** 항상 이 Pro 모델을 사용하십시오. 명시적으로 요청된 경우에만 다른 모델을 사용하십시오.

## 빠른 참조

### 기본 설정
- **모델:** `gemini-3-pro-image-preview`
- **해상도:** 1K (기본값, 옵션: 1K, 2K, 4K)
- **가로세로비 (Aspect Ratio):** 1:1 (기본값)

### 사용 가능한 가로세로비
`1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`

### 사용 가능한 해상도
`1K` (기본값), `2K`, `4K`

## 핵심 API 패턴

```python
import os
from google import genai
from google.genai import types

client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])

# 기본 생성 (1K, 1:1 - 기본값)
response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=["여기에 프롬프트 입력"],
    config=types.GenerateContentConfig(
        response_modalities=['TEXT', 'IMAGE'],
    ),
)

for part in response.parts:
    if part.text:
        print(part.text)
    elif part.inline_data:
        image = part.as_image()
        image.save("output.jpg")
```

## 커스텀 해상도 및 가로세로비

```python
from google.genai import types

response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=[prompt],
    config=types.GenerateContentConfig(
        response_modalities=['TEXT', 'IMAGE'],
        image_config=types.ImageConfig(
            aspect_ratio="16:9",  # 와이드 포맷
            image_size="2K"       # 더 높은 해상도
        ),
    )
)
```

### 해상도 예시

```python
# 1K (기본값) - 빠름, 미리보기용으로 적합
image_config=types.ImageConfig(image_size="1K")

# 2K - 품질과 속도의 균형
image_config=types.ImageConfig(image_size="2K")

# 4K - 최고 품질, 느림
image_config=types.ImageConfig(image_size="4K")
```

### 가로세로비 예시

```python
# 정사각형 (기본값)
image_config=types.ImageConfig(aspect_ratio="1:1")

# 가로로 긴 와이드
image_config=types.ImageConfig(aspect_ratio="16:9")

# 파노라마 울트라 와이드
image_config=types.ImageConfig(aspect_ratio="21:9")

# 세로로 긴 포트레이트
image_config=types.ImageConfig(aspect_ratio="9:16")

# 사진 표준
image_config=types.ImageConfig(aspect_ratio="4:3")
```

## 이미지 편집

기존 이미지와 함께 텍스트 프롬프트를 전달합니다:

```python
from PIL import Image

img = Image.open("input.jpg")
response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=["이 장면에 일몰을 추가해줘", img],
    config=types.GenerateContentConfig(
        response_modalities=['TEXT', 'IMAGE'],
    ),
)
```

## 멀티 턴 (Multi-Turn) 개선

반복적인 편집을 위해 채팅을 사용합니다:

```python
from google.genai import types

chat = client.chats.create(
    model="gemini-3-pro-image-preview",
    config=types.GenerateContentConfig(response_modalities=['TEXT', 'IMAGE'])
)

response = chat.send_message("'Acme Corp'을 위한 로고를 만들어줘")
# 첫 번째 이미지 저장...

response = chat.send_message("텍스트를 더 굵게 하고 파란색 그라데이션을 추가해줘")
# 개선된 이미지 저장...
```

## 프롬프트 작성 모범 사례

### 사실적인 사진 장면
카메라 상세 정보 포함: 렌즈 유형, 조명, 각도, 분위기.
> "A photorealistic close-up portrait, 85mm lens, soft golden hour light, shallow depth of field"

### 스타일화된 아트
스타일을 명시적으로 지정:
> "A kawaii-style sticker of a happy red panda, bold outlines, cel-shading, white background"

### 이미지 내 텍스트
폰트 스타일과 위치를 구체적으로 지정:
> "Create a logo with text 'Daily Grind' in clean sans-serif, black and white, coffee bean motif"

### 제품 목업 (Mockups)
조명 설정과 표면 설명:
> "Studio-lit product photo on polished concrete, three-point softbox setup, 45-degree angle"

## 고급 기능

### Google 검색 그라운딩 (Grounding)
실시간 데이터를 기반으로 이미지 생성:

```python
response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=["오늘 도쿄의 날씨를 인포그래픽으로 시각화해줘"],
    config=types.GenerateContentConfig(
        response_modalities=['TEXT', 'IMAGE'],
        tools=[{"google_search": {}}]
    )
)
```

### 여러 참조 이미지 (최대 14개)
여러 소스의 요소를 결합:

```python
response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=[
        "이 사람들의 그룹 사진을 사무실 배경으로 만들어줘",
        Image.open("person1.jpg"),
        Image.open("person2.jpg"),
        Image.open("person3.jpg"),
    ],
    config=types.GenerateContentConfig(
        response_modalities=['TEXT', 'IMAGE'],
    ),
)
```

## 중요: 파일 포맷 및 미디어 타입

**중요:** Gemini API는 기본적으로 JPEG 포맷으로 이미지를 반환합니다. 저장할 때는 미디어 타입 불일치를 피하기 위해 항상 `.jpg` 확장자를 사용하십시오.

```python
# 올바름 - .jpg 확장자 사용 (Gemini는 JPEG 반환)
image.save("output.jpg")

# 틀림 - "Image does not match media type" 오류 발생 가능
image.save("output.png")  # PNG 확장자를 가진 JPEG 파일을 생성함!
```

### PNG로 변환 (필요한 경우)

특히 PNG 포맷이 필요한 경우:

```python
from PIL import Image

# Gemini로 생성
for part in response.parts:
    if part.inline_data:
        img = part.as_image()
        # 명시적 포맷 지정으로 PNG로 저장하여 변환
        img.save("output.png", format="PNG")
```

### 이미지 포맷 확인

`file` 명령으로 실제 포맷과 확장자를 확인하십시오:

```bash
file image.png
# 출력에 "JPEG image data"라고 나오면 .jpg로 이름을 바꾸십시오!
```

## 참고 사항

- 모든 생성된 이미지에는 SynthID 워터마크가 포함됩니다.
- Gemini는 **기본적으로 JPEG 포맷**을 반환하므로 항상 `.jpg` 확장자를 사용하십시오.
- 이미지 전용 모드(`responseModalities: ["IMAGE"]`)는 Google 검색 그라운딩과 함께 작동하지 않습니다.
- 편집 시에는 변경 사항을 대화하듯 설명하십시오 — 모델이 의미론적 마스킹(semantic masking)을 이해합니다.
- 속도를 위해 기본적으로 1K 해상도를 사용하고, 품질이 중요한 경우에만 2K/4K를 사용하십시오.
