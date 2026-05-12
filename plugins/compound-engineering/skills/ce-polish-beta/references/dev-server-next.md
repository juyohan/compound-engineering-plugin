# Next.js dev-server 레시피 (자동 감지 폴백)

`detect-project-type.sh`가 `next`를 반환하고 참고할 `.claude/launch.json`이 없을 때 로드됩니다.

## 서명 (Signature)

- `next.config.js`, `next.config.mjs`, `next.config.ts`, 또는 `next.config.cjs` 파일이 존재함
- `package.json`에 `next` 종속성이 포함되어 있음

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

기본값: `3000`. Next.js는 `-p <port>` / `--port <port>` 플래그와 `PORT` 환경 변수를 준수합니다. 오버라이드 규칙은 `references/dev-server-detection.md`의 연쇄(cascade) 규칙을 따릅니다.

## Turbopack

Next.js 14 이상은 `--turbo`를 지원하며(15 이상에서는 기본값), `package.json`의 `dev` 스크립트에 `--turbo`가 포함되어 있다면 이를 유지하십시오. Turbopack은 리로드 동작을 변경하지만 포트나 URL 컨벤션은 변경하지 않습니다.

## 스텁 생성 (Stub generation)

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

결정된 패키지 매니저 (`npm` / `pnpm` / `yarn` / `bun`)와 포트로 대체하십시오.

## 일반적인 주의 사항

- **App Router vs Pages Router:** 개발 서버 동작은 동일하며 polish는 이를 신경 쓰지 않습니다. 체크리스트 생성(Unit 5) 시에는 `app/`과 `pages/`에 있는 페이지가 서로 다른 영역이므로 구분해야 합니다.
- **모노레포 루트:** pnpm/Turborepo 모노레포에서 루트의 `npm run dev`는 일반적으로 여러 패키지로 전파됩니다. 사용자는 `.claude/launch.json`에서 `cwd`를 특정 Next 앱으로 설정해야 합니다 (`cwd: "apps/web"`).
- **환경 변수 로딩:** `.env.local`은 Next.js에 의해 자동으로 로드됩니다. Polish가 이를 별도로 export할 필요는 없습니다.
