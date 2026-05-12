# Astro dev-server 레시피 (자동 감지 폴백)

`detect-project-type.sh`가 `astro`를 반환하고 참고할 `.claude/launch.json`이 없을 때 로드됩니다.

## 서명 (Signature)

- `astro.config.js`, `astro.config.mjs`, 또는 `astro.config.ts` 파일이 존재함
- `package.json`에 `astro` 종속성이 포함되어 있음

## 시작 명령

표준:

```bash
npm run dev
```

`package.json`의 `dev` 스크립트는 일반적으로 `astro dev`를 래핑합니다. 다음 명령어도 유효합니다 (프로젝트가 사용하는 명령어를 확인하려면 `package.json` 스크립트를 읽으십시오):

```bash
pnpm dev
yarn dev
bun run dev
```

락파일(lockfile)이 가리키는 패키지 매니저를 우선적으로 사용합니다:
- `pnpm-lock.yaml` -> `pnpm dev`
- `yarn.lock` -> `yarn dev`
- `bun.lock` / `bun.lockb` -> `bun run dev`
- `package-lock.json` 또는 없음 -> `npm run dev`

## 포트 (Port)

기본값: `4321`. Astro는 `--port <port>` 플래그와 `astro.config.*`의 `server.port` 필드를 준수합니다. 오버라이드 규칙은 `references/dev-server-detection.md`의 연쇄(cascade) 규칙을 따릅니다.

## 스텁 생성 (Stub generation)

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

결정된 패키지 매니저 (`npm` / `pnpm` / `yarn` / `bun`)와 포트로 대체하십시오.

## 일반적인 주의 사항

- **SSR vs SSG:** `astro dev`는 두 출력 모드 모두에서 동일하게 실행됩니다. 차이는 빌드 시점에만 중요합니다. Polish는 이를 구분할 필요가 없습니다.
- **Astro 설정이 Vite 설정보다 우선함:** Astro는 내부적으로 Vite를 사용하지만 자체 설정 파일을 제공합니다. `astro.config.*`와 `vite.config.*`가 모두 존재할 경우 `astro` 유형이 `vite`보다 우선합니다. 이는 드문 경우이며, Astro 프로젝트는 보통 별도의 Vite 설정 파일을 가지지 않습니다.
- **개발 툴바 (Astro 4+):** Astro 4 이상에는 브라우저에 UI를 오버레이하는 개발 툴바가 포함되어 있습니다. 이는 포트 바인딩이나 URL 라우팅에 영향을 주지 않으므로 polish는 이를 무시해도 됩니다.
