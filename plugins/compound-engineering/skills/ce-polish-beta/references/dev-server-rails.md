# Rails dev-server 레시피 (자동 감지 폴백)

`detect-project-type.sh`가 `rails`를 반환하고 참고할 `.claude/launch.json`이 없을 때 로드됩니다.

## 서명 (Signature)

- `bin/dev` 파일이 존재하고 실행 가능함
- `Gemfile` 파일이 존재함

## 시작 명령

```bash
bin/dev
```

`bin/dev`는 "모든 것을 시작"(웹 + 에셋 와쳐 + 선택적 워커)하기 위한 Rails 7 이상의 컨벤션입니다. 이는 내부적으로 `foreman start -f Procfile.dev`를 호출하는 한 줄짜리 스크립트이므로, `bin/dev`가 없거나 실행 불가능한 경우 `Procfile.dev`가 *실제* 명령을 확인할 수 있는 표준적인 장소입니다.

## 포트 (Port)

기본값: `3000`. 오버라이드 규칙은 `references/dev-server-detection.md`의 연쇄(cascade) 규칙을 따릅니다:
1. `Procfile.dev`의 `web:` 라인에 `-p <n>`이 포함될 수 있음
2. `config/puma.rb`가 기본값이 아닌 포트에 바인딩될 수 있음
3. `.env` / `.env.development`의 `PORT=<n>`
4. `AGENTS.md` / `CLAUDE.md` 프로젝트 지침

## `.claude/launch.json`을 위한 스텁 생성

사용자가 "이 내용을 `.claude/launch.json`으로 저장할까요?"라는 질문에 수락하면, `launch-json-schema.md`에 정의된 Rails 스텁을 생성합니다:

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

연쇄 규칙을 통해 3000이 아닌 포트가 결정된 경우, 쓰기 전에 스텁의 `port` 필드를 해당 값으로 교체하십시오.

## 일반적인 주의 사항

- **Bundler 경로:** 일부 환경에서는 `bundle exec bin/dev`가 필요할 수 있습니다. `bin/dev`가 로드 경로(load-path) 에러로 실패하면 `bundle exec bin/dev`로 폴백하십시오.
- **Foreman vs Overmind:** `Procfile`과 `Procfile.dev`가 둘 다 존재하는 경우가 많습니다. Rails의 `bin/dev`는 `Procfile.dev`를 사용합니다. 프로젝트에서 명시적으로 `overmind`를 사용하는 경우, `overmind start -f Procfile.dev`를 선호하십시오 (`dev-server-procfile.md` 참조).
- **SSL 개발 서버:** `--ssl` 옵션이 있는 `rails s`는 URL 스킴을 변경합니다. Polish의 도달 가능성(reachability) 조사에는 `http://`를 사용합니다. SSL 개발 서버를 사용하는 사용자는 `.claude/launch.json`에 `port`를 명시적으로 설정하고 체크리스트에 스킴을 기록해야 합니다.
