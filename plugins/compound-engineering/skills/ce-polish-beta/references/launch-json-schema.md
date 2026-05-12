# `.claude/launch.json` 스키마

Polish는 개발 서버 시작 명령을 확인하기 위해 레포지토리 루트의 `.claude/launch.json`을 읽습니다. 이 스키마는 VS Code의 `launch.json` 형식의 하위 집합입니다. Claude Code, Cursor, VS Code가 모두 이를 이해하며 사용자들이 이미 에디터 통합을 위해 가지고 있는 경우가 많기 때문에 이 형식을 선택했습니다.

## 최상위 구조

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "<인간이 읽을 수 있는 라벨>",
      "runtimeExecutable": "<바이너리>",
      "runtimeArgs": ["<인자>", "<인자>"],
      "port": <숫자>,
      "cwd": "<선택 사항, 레포지토리 상대 경로>",
      "env": { "<키>": "<값>" }
    }
  ]
}
```

## Polish가 사용하는 필드

| 필드 | 필수 여부 | 목적 |
|-------|----------|---------|
| `name` | 예 (구성 설정이 여러 개인 경우) | 배열에 항목이 두 개 이상일 때 명확히 구분하기 위해 사용됩니다. Polish는 사용자에게 `name`을 기준으로 선택하도록 요청합니다. |
| `runtimeExecutable` | 예 | Polish가 실행할 바이너리 (예: `bin/dev`, `npm`, `overmind`, `bun`). |
| `runtimeArgs` | 아니요 | `runtimeExecutable`에 전달할 인자 배열입니다. 기본값: 빈 배열. |
| `port` | 예 | 개발 서버가 리스닝할 포트입니다. Polish는 도달 가능성 확인을 위해 `http://localhost:<port>`를 조사하고 IDE 브라우저 핸드오프에 이 포트를 사용합니다. |
| `cwd` | 아니요 | 개발 서버를 위한 레포지토리 상대 작업 디렉토리입니다. 기본값: 레포지토리 루트. 모노레포(`apps/web`, `packages/frontend`)에서 유용합니다. |
| `env` | 아니요 | 개발 서버 프로세스를 위한 추가 환경 변수입니다. 기본값: polish의 환경을 상속합니다. |

## 스텁 템플릿 (첫 실행 시 사용자가 수락할 때 작성됨)

Polish가 프로젝트 유형을 자동 감지하고 사용자가 "이 내용을 `.claude/launch.json`으로 저장할까요?"를 확인하면, 감지된 유형에서 파생된 최소한의 스텁을 작성합니다. 이 템플릿들은 의도적으로 일반적인 기본값들을 하드코딩하며, 사용자는 나중에 이를 편집할 수 있습니다.

### Rails 스텁

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Rails dev",
      "runtimeExecutable": "bin/dev",
      "runtimeArgs": [],
      "port": 3000
    }
  ]
}
```

### Next.js 스텁

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next dev",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 3000
    }
  ]
}
```

### Vite 스텁

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Vite dev",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 5173
    }
  ]
}
```

### Procfile / Overmind 스텁

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Overmind dev",
      "runtimeExecutable": "overmind",
      "runtimeArgs": ["start", "-f", "Procfile.dev"],
      "port": 3000
    }
  ]
}
```

### Nuxt 스텁

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Nuxt dev",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 3000
    }
  ]
}
```

### Astro 스텁

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Astro dev",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 4321
    }
  ]
}
```

### Remix 스텁

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Remix dev",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 3000
    }
  ]
}
```

### SvelteKit 스텁

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "SvelteKit dev",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 5173
    }
  ]
}
```

## 왜 VS Code 스키마의 하위 집합인가?

Polish는 `type`, `request`, `console`, `stopOnEntry` 등 기타 VS Code 필드를 사용하지 않습니다. 이들을 포함해도 무방하며 polish는 무시하겠지만, 스텁 작성기(stub writer)는 이를 추가하지 않습니다. Polish가 관심을 갖는 필드는 *알려진 포트에서 장기 실행되는 개발 서버를 시작하는 방법*을 설명하는 필드들이며, 이는 VS Code가 디버그 스테핑을 위해 사용하는 범위보다 좁은 영역입니다.

## IDE 간 참고 사항

`.claude/launch.json`은 아직 Claude Code, Cursor, VS Code 및 Codex 간에 완전히 통합된 표준은 아닙니다. Polish가 `.claude/launch.json`을 앞세우는 이유는 다음과 같습니다:
- Claude Code, Cursor, VS Code 모두 실행 구성으로 읽을 수 있습니다.
- 레포지토리 루트의 깨끗한 신뢰 경계에 위치합니다 (자동 감지가 아닌 사용자가 작성한 파일).
- `.vscode/launch.json`을 선호하는 사용자는 두 파일을 수동으로 심볼릭 링크하거나 미러링할 수 있습니다.

만약 IDE 간 표준(예: `.workspace/launch.json`)이 등장한다면, 스킬의 나머지 부분을 건드리지 않고 스텁 작성기와 리더의 경로만 교체할 수 있습니다.
