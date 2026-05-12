# 개발 서버 포트 감지 (Dev-server port detection)

포트 확인은 `scripts/resolve-port.sh`를 통해 실행됩니다. 이 문서는 탐색 순서, 프레임워크 기본값 및 `test-browser` 스킬의 인라인 연쇄(cascade) 규칙과의 의도적인 차이점을 설명합니다.

이 연쇄 규칙은 `.claude/launch.json`이 없거나 확인된 구성에 `port` 필드가 없을 때**만** 실행됩니다. `launch.json`에 포트가 지정된 경우, 해당 포트를 그대로 사용하고 이 연쇄 규칙 전체를 건너뜁니다.

## 우선순위 순서

1. **명시적 `--port` 플래그** -- 호출자가 `--port <n>`을 전달한 경우 직접 사용합니다.
2. **프레임워크 설정 파일** -- 숫자 리터럴 포트 값만 일치하는 보수적인 정규식을 사용하여 `next.config.*`, `vite.config.*`, `nuxt.config.*`, `astro.config.*`를 스캔합니다. 변수 참조 (`process.env.PORT`, `getPort()`)는 의도적으로 일치시키지 않습니다.
3. **Rails `config/puma.rb`** -- `port <n>`을 찾습니다 (grep).
4. **`Procfile.dev`** -- web 라인에서 `-p <n>` / `--port <n>` / `-p=<n>` / `--port=<n>`을 스캔합니다.
5. **`docker-compose.yml`** -- `"<n>:<n>"` 포트 매핑 패턴에 대해 라인 앵커 방식의 grep을 수행합니다. 전체 YAML 파싱은 하지 않습니다.
6. **`package.json`** -- `dev`/`start` 스크립트에서 `--port <n>` / `-p <n>` / `--port=<n>` / `-p=<n>`을 스캔합니다.
7. **`.env` 파일** -- 오버라이드 순서대로 확인합니다: `.env.local` -> `.env.development` -> `.env` (먼저 발견된 것이 승리). 따옴표 제거 및 주석 절단을 포함하여 `PORT=<n>`을 파싱합니다.
8. **프레임워크 기본값 조회 테이블** -- 아래 테이블 참조.

## 프레임워크 기본값

| 프레임워크 | 기본 포트 |
|-----------|-------------|
| Rails | 3000 |
| Next.js | 3000 |
| Nuxt | 3000 |
| Remix (classic) | 3000 |
| Vite | 5173 |
| SvelteKit | 5173 |
| Astro | 4321 |
| Procfile | 3000 |
| 알 수 없음 | 3000 |

## 동기화 메모 (Sync-note block)

`resolve-port.sh`와 `test-browser` 스킬의 인라인 연쇄 규칙은 목적이 겹치지만 세 가지 구체적인 방식에서 차이가 납니다. 이러한 차이는 의도적인 것이므로, 근거를 이해하지 못한 채 어느 한 쪽을 다른 쪽에 맞춰 "수정"하지 마십시오.

**(a) `.env` 값의 따옴표 제거.** `resolve-port.sh`는 `PORT=` 값에서 주변의 `"` 및 `'`를 제거합니다 (따라서 `PORT="3001"`은 `3001`로 해석됩니다). `test-browser` 인라인 연쇄 규칙은 따옴표를 제거하지 않습니다. 스크립트 버전이 따옴표 사용이 흔한 실제 `.env` 파일에 대해 더 견고합니다.

**(b) `.env` 값의 주석 제거.** `resolve-port.sh`는 공백 제거 후 `#`에서 자릅니다 (따라서 `PORT=3001 # dev only`는 `3001`로 해석됩니다). `test-browser` 인라인 연쇄 규칙은 주석을 제거하지 않습니다. 동일한 근거입니다: 실제 `.env` 파일에는 인라인 주석이 빈번하게 포함됩니다.

**(c) `AGENTS.md`/`CLAUDE.md` grep 제거.** `resolve-port.sh`는 지침 파일에서 포트 참조를 스캔하지 않습니다. `test-browser` 인라인 연쇄 규칙은 스캔합니다. 지침 파일에는 개발 서버와 무관한 문맥(문서, 예제, 문제 해결)에서 포트가 언급될 수 있는 자연어가 포함되어 있어, 디버깅하기 어려운 오탐(false positive)을 생성합니다. 프레임워크 설정 파일과 `.env`가 더 신뢰할 수 있는 소스입니다.
