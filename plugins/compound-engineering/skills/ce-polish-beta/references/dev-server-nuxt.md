# Nuxt dev-server 레시피 (자동 감지 폴백)

`detect-project-type.sh`가 `nuxt`를 반환하고 참고할 `.claude/launch.json`이 없을 때 로드됩니다.

## 서명 (Signature)

- `nuxt.config.js`, `nuxt.config.mjs`, 또는 `nuxt.config.ts` 파일이 존재함
- `package.json`에 `nuxt` 종속성이 포함되어 있음

## 시작 명령

표준:

```bash
npm run dev
```

다음 명령어도 유효합니다 (프로젝트가 사용하는 명령어를 확인하려면 `package.json` 스크립트를 읽으십시오):

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

기본값: `3000`. Nuxt는 `--port <port>` 플래그와 `PORT` 환경 변수를 준수합니다. 오버라이드 규칙은 `references/dev-server-detection.md`의 연쇄(cascade) 규칙을 따릅니다.

## 스텁 생성 (Stub generation)

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

결정된 패키지 매니저 (`npm` / `pnpm` / `yarn` / `bun`)와 포트로 대체하십시오.

## 일반적인 주의 사항

- **Nitro 서버 엔진:** Nitro(Nuxt의 서버 엔진)는 Nuxt 뒤에서 자체 개발 서버를 추가합니다. Polish는 Nuxt 포트에만 관심을 가집니다. Nitro 내부 포트를 별도로 탐색하지 마십시오.
- **포트 자동 증가:** 3000 포트가 이미 점유된 경우, Nuxt는 (에러를 내는 Next.js와 달리) 포트를 자동으로 증가시킵니다. Polish의 포트 기준 종료(kill-by-port) 단계는 시작 전 포트를 회수하여 이를 처리하므로, 실제 운용 시 자동 증가 동작이 문제를 일으키지 않습니다.
- **Nuxt 3 vs Nuxt 2:** Nuxt 3는 `nuxt.config.ts`를 사용하고 Nuxt 2는 `nuxt.config.js`를 사용합니다. 두 버전 모두 서명 확인을 통해 감지됩니다. 개발 서버 명령 및 포트 기본값은 두 버전 모두 동일합니다.
