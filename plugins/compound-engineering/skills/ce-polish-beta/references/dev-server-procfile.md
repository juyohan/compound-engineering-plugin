# Procfile / Overmind dev-server 레시피 (자동 감지 폴백)

`detect-project-type.sh`가 `procfile`을 반환하고 참고할 `.claude/launch.json`이 없을 때 로드됩니다. `bin/dev`가 있는 Rails 앱은 단순 Procfile 경로보다 우선순위가 높습니다 (`dev-server-rails.md` 참조).

## 서명 (Signature)

- 레포지토리 루트에 `Procfile` 또는 `Procfile.dev`가 존재함
- `bin/dev`가 존재하지 않음 (존재하는 경우 Rails 레시피 사용)

## 시작 명령

`overmind`를 사용할 수 있는 경우 이를 선호합니다. 소켓 파일을 처리하고 프로세스별 핫 리스타트를 지원하며, 멀티 프로세스 개발을 위한 커뮤니티 표준입니다:

```bash
overmind start -f Procfile.dev
```

`overmind`가 설치되어 있지 않은 경우 `foreman`으로 폴백합니다:

```bash
foreman start -f Procfile.dev
```

둘 다 없는 경우 추측하기보다는 사용자에게 시작 명령을 요청하십시오.

## 포트 (Port)

기본값: `3000`. Procfile 기반 프로젝트는 `Procfile.dev`에 프로세스를 나열하므로, 권위 있는 포트 정보는 `web:` 라인에서 가져옵니다:

```
web: bundle exec puma -p 3000 -C config/puma.rb
worker: bundle exec sidekiq
```

`web:` 라인에서 `-p <n>` 또는 `--port <n>`을 파싱하십시오. 둘 다 없는 경우 `references/dev-server-detection.md`의 연쇄(cascade) 규칙을 따릅니다.

## 스텁 생성 (Stub generation)

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

사용자의 머신에서 `overmind`를 사용할 수 없는 경우 `foreman`으로 대체하십시오. 스텁은 표준 레시피가 아니라 사용자가 실제로 실행할 명령을 나타냅니다.

## 일반적인 주의 사항

- **소켓 파일:** `overmind`는 기본적으로 `.overmind.sock`에 소켓을 씁니다. Polish의 포트 기준 종료(kill-by-port) 로직은 포트를 회수하지만 소켓을 정리하지는 않습니다. Overmind가 이미 실행 중인데 polish가 이를 재시작하면, 오래된 소켓이 제거될 때까지 새 프로세스가 "connection refused" 에러로 실패할 수 있습니다. 필요한 경우 `OVERMIND_SOCKET` 환경 변수를 사용하여 소켓을 실행 시마다 다른 경로로 리다이렉트할 수 있습니다.
- **Procfile vs Procfile.dev:** 운영용(production)과 개발용(development) Procfile은 자주 다릅니다. Polish에서는 항상 `Procfile.dev`를 선호하십시오.
- **다중 웹 프로세스:** 일부 Procfile은 웹 트래픽을 여러 프로세스(API + 프론트엔드)로 분산합니다. Polish는 하나의 URL만 열 수 있습니다. 다중 웹 설정을 사용하는 사용자는 `.claude/launch.json`을 직접 작성하여 어떤 프로세스가 polish를 위한 "개발 서버"인지 명시해야 합니다.
