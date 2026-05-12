# Vite dev-server 레시피 (자동 감지 폴백)

`detect-project-type.sh`가 `vite`를 반환하고 참고할 `.claude/launch.json`이 없을 때 로드됩니다.

## 서명 (Signature)

- `vite.config.js`, `vite.config.ts`, `vite.config.mjs`, 또는 `vite.config.cjs` 파일이 존재함

## 시작 명령

표준:

```bash
npm run dev
```

`package.json`의 `dev` 스크립트는 일반적으로 `vite`를 직접 래핑합니다. 락파일이 가리키는 패키지 매니저를 우선적으로 사용하십시오 (락파일과 명령어 매핑은 Next.js 레시피를 참조하십시오).

## 포트 (Port)

기본값: `5173`. Vite는 `--port <n>` 플래그와 `VITE_PORT` 환경 변수를 준수합니다. `references/dev-server-detection.md`의 연쇄 규칙은 `package.json` 스크립트의 `--port`와 `.env*`의 `PORT`를 감지합니다.

Vite의 `--strictPort` 플래그는 요청된 포트가 이미 사용 중일 때 다음 가용 포트로 자동 증가하는 대신 개발 서버 실행을 실패하게 만듭니다. Polish의 포트 기준 종료(kill-by-port) 단계가 시작 전 포트를 회수하므로 `strictPort`는 실제 운용 시 문제가 되지 않습니다. 하지만 포트 회수를 비활성화하고 여러 Vite 인스턴스를 실행하는 사용자는 `vite.config.ts`에 `strictPort: true`가 설정되어 있지 않는 한 포트 자동 증가 현상을 보게 될 것입니다.

## 호스트 바인딩 (Host binding)

Vite는 기본적으로 `127.0.0.1`에 바인딩됩니다. Devcontainer나 WSL 내부에서 실행되는 polish의 경우, 사용자는 `runtimeArgs`에 `--host 0.0.0.0`을 추가해야 할 수도 있습니다. 체크리스트 생성 시 관련이 있다면 이를 기록할 수 있습니다.

## 스텁 생성 (Stub generation)

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

## 일반적인 주의 사항

- **HMR 웹소켓 포트:** Vite의 HMR은 기본적으로 개발 서버 포트를 상속받는 별도의 웹소켓을 사용합니다. 프로젝트가 `vite.config.ts`에서 `server.hmr.port`를 고정한 경우에도 개발 서버 포트에 대한 polish의 도달 가능성 조사는 여전히 작동하지만, 임베디드 브라우저에서 HMR에 도달하기 위해 추가 설정이 필요할 수 있습니다.
- **Vite 기반 프레임워크:** SvelteKit, SolidStart, Qwik City, Astro 모두 Vite를 사용하지만 자체적인 개발 스크립트를 추가합니다. `vite` 서명은 이들을 감지하며, `npm run dev`는 이들 모두에 적합한 명령입니다. 프레임워크마다 서로 다른 기본 포트가 적용될 수 있으므로(SvelteKit: 5173, Astro: 4321, Qwik: 5173), `package.json` 또는 `.env`에서 실제 포트를 확인하기 위해 연쇄 규칙에 의존하십시오.
