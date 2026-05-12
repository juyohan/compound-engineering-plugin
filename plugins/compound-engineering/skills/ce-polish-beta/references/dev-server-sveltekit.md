# SvelteKit dev-server 레시피 (자동 감지 폴백)

`detect-project-type.sh`가 `sveltekit`을 반환하고 참고할 `.claude/launch.json`이 없을 때 로드됩니다.

## 서명 (Signature)

- `svelte.config.js`, `svelte.config.mjs`, 또는 `svelte.config.ts` 파일이 존재함
- `package.json`에 `@sveltejs/kit` 종속성이 포함되어 있음

## 시작 명령

표준:

```bash
npm run dev
```

`package.json`의 `dev` 스크립트는 일반적으로 SvelteKit을 통해 `vite dev`를 래핑합니다. 다음 명령어도 유효합니다 (프로젝트가 사용하는 명령어를 확인하려면 `package.json` 스크립트를 읽으십시오):

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

기본값: `5173` (Vite로부터 상속). SvelteKit은 `--port <port>` 플래그와 `vite.config.ts`의 Vite `server.port` 설정을 준수합니다. 오버라이드 규칙은 `references/dev-server-detection.md`의 연쇄(cascade) 규칙을 따릅니다.

## 스텁 생성 (Stub generation)

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

결정된 패키지 매니저 (`npm` / `pnpm` / `yarn` / `bun`)와 포트로 대체하십시오.

## 일반적인 주의 사항

- **내부적으로 Vite 사용:** SvelteKit은 내부적으로 Vite를 사용합니다. 동일한 기본 포트(5173)와 HMR 동작을 가집니다. `sveltekit` 유형이 별도로 존재하는 이유는 `svelte.config.js`가 일반적인 `vite.config.ts`보다 더 정확한 신호이기 때문이며, 이를 통해 polish가 SvelteKit 전용 스텁 이름과 라벨을 생성할 수 있습니다.
- **개발 시 어댑터(Adapter)는 무관함:** `adapter-auto`, `adapter-node`, `adapter-static` 등 모든 어댑터는 동일한 개발 서버를 생성합니다. 어댑터는 운영 빌드 출력물에만 영향을 미칩니다.
- **`svelte.config.js`가 주요 서명임:** SvelteKit 프로젝트에는 `vite.config.ts`가 존재하더라도 항상 `svelte.config.js`가 존재합니다. 이 파일이 SvelteKit 프로젝트와 일반 Vite 프로젝트를 구분하는 기준이 됩니다.
