# Remix dev-server 레시피 (자동 감지 폴백)

`detect-project-type.sh`가 `remix`를 반환하고 참고할 `.claude/launch.json`이 없을 때 로드됩니다.

## 서명 (Signature)

- `remix.config.js` 또는 `remix.config.ts` 파일이 존재함 (클래식 Remix)
- Vite 기반의 Remix 2.x 이상은 `remix.config.*`가 없으며 Remix 플러그인과 함께 `vite.config.ts`를 사용하므로 `remix`가 아닌 `vite` 유형으로 분류됩니다.

## 시작 명령

표준:

```bash
npm run dev
```

`package.json`의 `dev` 스크립트는 일반적으로 `remix dev`를 래핑합니다. 다음 명령어도 유효합니다 (프로젝트가 사용하는 명령어를 확인하려면 `package.json` 스크립트를 읽으십시오):

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

기본값: `3000`. Remix는 `--port <port>` 플래그를 준수합니다. 클래식 Remix 개발 서버는 `PORT` 환경 변수도 읽습니다. 오버라이드 규칙은 `references/dev-server-detection.md`의 연쇄(cascade) 규칙을 따릅니다.

## 스텁 생성 (Stub generation)

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

결정된 패키지 매니저 (`npm` / `pnpm` / `yarn` / `bun`)와 포트로 대체하십시오.

## 일반적인 주의 사항

- **클래식 vs Vite:** 클래식 Remix는 `remix.config.js`를 사용합니다. 새로운 Remix (v2+)는 Vite를 사용하며, 이는 `remix`가 아닌 `vite` 유형으로 감지됩니다. `remix` 유형은 여전히 `remix.config.*` 파일을 가지고 있는 클래식 Remix 프로젝트를 위한 것입니다.
- **Remix v1 vs v2 개발 서버:** v2의 `remix dev`는 포트에 바인딩되는 Express 기반 개발 서버를 시작합니다. v1의 `remix dev`는 와쳐(watcher) 역할만 수행했습니다(서버 없음). Polish가 포트에 바인딩하고 도달 가능성 조사를 수행하려면 v2 이상이 필요합니다.
- **Vite 기반 Remix는 Vite의 포트를 상속함:** Vite 기반 Remix(`remix.config.*` 없음)의 기본 포트는 3000이 아니라 5173(Vite 기본값)입니다. 이 경우는 본 레시피가 아닌 `vite` 레시피에서 처리됩니다.
