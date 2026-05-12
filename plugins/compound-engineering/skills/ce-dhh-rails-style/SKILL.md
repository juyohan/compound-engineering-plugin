---
name: ce-dhh-rails-style
description: 이 스킬은 DHH 특유의 37signals 스타일로 Ruby 및 Rails 코드를 작성할 때 사용해야 합니다. Ruby 코드 작성, Rails 애플리케이션 개발, 모델 및 컨트롤러 생성 또는 모든 Ruby 파일 작성 시에 적용됩니다. Ruby/Rails 코드 생성, 리팩토링 요청, 코드 리뷰 또는 사용자가 DHH, 37signals, Basecamp, HEY, Campfire 스타일을 언급할 때 트리거됩니다. REST 순수성, 뚱뚱한 모델(fat models), 얇은 컨트롤러(thin controllers), Current 속성, Hotwire 패턴, 그리고 "기교보다 명확함(clarity over cleverness)" 철학을 구현합니다.
allowed-tools:
  - gem
---

<objective>
Ruby 및 Rails 코드에 37signals/DHH Rails 컨벤션을 적용합니다. 이 스킬은 프로덕션 37signals 코드베이스(Fizzy/Campfire) 분석과 DHH의 코드 리뷰 패턴에서 추출된 포괄적인 도메인 전문 지식을 제공합니다.
</objective>

<essential_principles>
## 핵심 철학

"가장 좋은 코드는 작성하지 않은 코드입니다. 두 번째로 좋은 코드는 명백하게 올바른 코드입니다."

**순수 Rails(Vanilla Rails)로도 충분합니다:**
- 서비스 객체(service objects)보다 풍부한 도메인 모델 지향
- 커스텀 액션보다 CRUD 컨트롤러 지향
- 수평적 코드 공유를 위한 Concern 사용
- 불리언(boolean) 컬럼 대신 상태 기록(Records as state) 사용
- 모든 것에 데이터베이스 기반 접근 (Redis 사용 지양)
- Gem을 찾기 전에 직접 해결책 구축

**의도적으로 피하는 것들:**
- devise (대신 약 150줄의 커스텀 인증 사용)
- pundit/cancancan (모델에서 간단한 역할 체크 사용)
- sidekiq (데이터베이스를 사용하는 Solid Queue 지향)
- redis (모든 것에 데이터베이스 사용)
- view_component (부분 템플릿(partial)으로 충분)
- GraphQL (Turbo가 포함된 REST로 충분)
- factory_bot (Fixture가 더 간단함)
- rspec (Rails에 내장된 Minitest 지향)
- Tailwind (레이어가 있는 네이티브 CSS 지향)

**개발 철학:**
- 출시, 검증, 개선(Ship, Validate, Refine) - 학습을 위해 프로덕션에 프로토타입 품질의 코드 투입
- 증상이 아닌 근본 원인 해결
- 읽기 시점의 계산보다 쓰기 시점의 작업 지향
- ActiveRecord 검증보다 데이터베이스 제약 조건 지향
</essential_principles>

<intake>
어떤 작업을 하고 계신가요?

1. **컨트롤러 (Controllers)** - REST 매핑, Concern, Turbo 응답, API 패턴
2. **모델 (Models)** - Concern, 상태 기록, 콜백, 스코프, PORO
3. **뷰 및 프론트엔드 (Views & Frontend)** - Turbo, Stimulus, CSS, 부분 템플릿(partial)
4. **아키텍처 (Architecture)** - 라우팅, 멀티 테넌시, 인증, 작업(Jobs), 캐싱
5. **테스팅 (Testing)** - Minitest, Fixture, 통합 테스트
6. **Gem 및 종속성 (Gems & Dependencies)** - 사용해야 할 것과 피해야 할 것
7. **코드 리뷰 (Code Review)** - DHH 스타일에 따른 코드 리뷰
8. **일반 가이드 (General Guidance)** - 철학 및 컨벤션

**번호를 지정하거나 작업을 설명해 주세요.**
</intake>

<routing>

| 응답 | 읽어야 할 참조 (Reference) |
|----------|-------------------|
| 1, controller, 컨트롤러 | `references/controllers.md` |
| 2, model, 모델 | `references/models.md` |
| 3, view, frontend, turbo, stimulus, css, 뷰, 프론트엔드 | `references/frontend.md` |
| 4, architecture, routing, auth, job, cache, 아키텍처, 라우팅, 인증, 작업, 캐시 | `references/architecture.md` |
| 5, test, testing, minitest, fixture, 테스트, 테스팅 | `references/testing.md` |
| 6, gem, dependency, library, 젬, 종속성, 라이브러리 | `references/gems.md` |
| 7, review, 리뷰 | 모든 참조를 읽은 후 코드 리뷰 수행 |
| 8, general task, 일반 작업 | 문맥에 따라 관련 참조 읽기 |

**관련 참조를 읽은 후, 사용자의 코드에 패턴을 적용하십시오.**
</routing>

<quick_reference>
## 명명 규칙 (Naming Conventions)

**동사 (Verbs):** `card.close`, `card.gild`, `board.publish` (`set_style` 같은 메서드 지양)

**서술어 (Predicates):** `card.closed?`, `card.golden?` (관련 레코드의 존재 여부에서 유도)

**Concern:** 능력을 설명하는 형용사 (`Closeable`, `Publishable`, `Watchable`)

**컨트롤러:** 리소스와 일치하는 명사 (`Cards::ClosuresController`)

**스코프 (Scopes):**
- `chronologically`, `reverse_chronologically`, `alphabetically`, `latest`
- `preloaded` (표준적인 Eager loading 이름)
- `indexed_by`, `sorted_by` (파라미터화됨)
- `active`, `unassigned` (SQL 용어가 아닌 비즈니스 용어)

## REST 매핑

커스텀 액션 대신 새로운 리소스를 생성하십시오:

```
POST /cards/:id/close    → POST /cards/:id/closure
DELETE /cards/:id/close  → DELETE /cards/:id/closure
POST /cards/:id/archive  → POST /cards/:id/archival
```

## Ruby 구문 선호도

```ruby
# 대괄호 안에 공백이 있는 심볼 배열

## 다중 에이전트 협업 (Multi-Agent Collaboration)

사용자의 입력(`$ARGUMENTS`) 내에 `--add <ai-이름>` 형태의 플래그가 포함되어 있는지 확인하십시오. 
현재 지원되는 외부 AI 인터페이스는 `--add gemini` (또는 `--add gem`)입니다.

만약 해당 플래그가 감지되면, 작업을 단독으로 확정하지 말고 다음 절차를 따르십시오:
1. **의도 파악:** 플래그를 제외한 나머지 문자열을 실제 지시사항으로 간주합니다.
2. **초안 작성:** 본인(주 에이전트)의 지식과 코드베이스 컨텍스트를 바탕으로 작업의 초기 뼈대나 접근법을 생각합니다.
3. **MCP 협업 호출:** `gem` 도구를 호출하여 외부 Gemini 에이전트에게 조언이나 검토를 구합니다.
   - 호출 시 전달할 메시지 예시: "나는 현재 이 작업에 대한 초안을 세우고 있어. 내 초안은 [초안 요약]이야. 이 접근 방식의 기술적 타당성을 검토하고 누락된 에지 케이스나 더 나은 패턴을 조언해줄 수 있어?"
4. **결과 통합:** `gem` 도구가 반환한 피드백을 당신의 최종 결과물에 통합(Synthesis)합니다. 
5. **명시적 표시:** 최종 산출물의 상단 또는 설명 부분에 "이 결과물은 Gemini와의 협업을 통해 검토 및 보완되었습니다."라는 문구를 추가하십시오.

이 협업 절차를 염두에 두고 아래의 본래 스킬 워크플로우를 진행하십시오.

before_action :set_message, only: %i[ show edit update destroy ]

# private 메서드 들여쓰기
  private
    def set_message
      @message = Message.find(params[:id])
    end

# 조건문을 위한 Expression-less case
case
when params[:before].present?
  messages.page_before(params[:before])
else
  messages.last_page
end

# 빠른 실패(fail-fast)를 위한 Bang 메서드
@message = Message.create!(params)

# 간단한 조건문을 위한 삼항 연산자
@room.direct? ? @room.users : @message.mentionees
```

## 주요 패턴

**상태를 레코드로 (State as Records):**
```ruby
Card.joins(:closure)         # 닫힌 카드
Card.where.missing(:closure) # 열린 카드
```

**Current 속성 (Current Attributes):**
```ruby
belongs_to :creator, default: -> { Current.user }
```

**모델에서의 권한 부여 (Authorization on Models):**
```ruby
class User < ApplicationRecord
  def can_administer?(message)
    message.creator == self || admin?
  end
end
```
</quick_reference>

<reference_index>
## 도메인 지식

`references/` 디렉토리의 상세 패턴:

| 파일 | 주제 |
|------|--------|
| `references/controllers.md` | REST 매핑, Concern, Turbo 응답, API 패턴, HTTP 캐싱 |
| `references/models.md` | Concern, 상태 기록, 콜백, 스코프, PORO, 권한 부여, 브로드캐스팅 |
| `references/frontend.md` | Turbo Stream, Stimulus 컨트롤러, CSS 레이어, OKLCH 색상, 부분 템플릿 |
| `references/architecture.md` | 라우팅, 인증, 작업, Current 속성, 캐싱, 데이터베이스 패턴 |
| `references/testing.md` | Minitest, Fixture, 단위/통합/시스템 테스트, 테스트 패턴 |
| `references/gems.md` | 사용하거나 피해야 할 것, 결정 프레임워크, Gemfile 예시 |
</reference_index>

<success_criteria>
다음과 같은 경우 코드가 DHH 스타일을 따르는 것으로 간주됩니다:
- 컨트롤러가 리소스의 CRUD 동사에 매핑됨
- 모델이 수평적 동작을 위해 Concern을 사용함
- 상태가 불리언이 아닌 레코드로 추적됨
- 불필요한 서비스 객체나 추상화가 없음
- 외부 서비스보다 데이터베이스 기반 해결책을 선호함
- 테스트가 Fixture와 함께 Minitest를 사용함
- 상호작용을 위해 Turbo/Stimulus 사용 (무거운 JS 프레임워크 지양)
- 현대적 기능(레이어, OKLCH, 네스팅)이 포함된 네이티브 CSS 사용
- 권한 부여 로직이 User 모델에 위치함
- 작업(Jobs)이 모델 메서드를 호출하는 얇은 래퍼(wrapper)임
</success_criteria>

<credits>
[Marc Köhlbrugge](https://x.com/marckohlbrugge)의 [The Unofficial 37signals/DHH Rails Style Guide](https://github.com/marckohlbrugge/unofficial-37signals-coding-style-guide)를 기반으로 하며, Fizzy 코드베이스의 265개 풀 리퀘스트에 대한 심층 분석을 통해 생성되었습니다.

**중요 고지:**
- LLM에 의해 생성된 가이드로, 부정확한 내용이 포함될 수 있습니다.
- Fizzy의 코드 예시는 O'Saasy 라이선스에 따라 라이선스가 부여됩니다.
- 37signals와 제휴하거나 보증을 받은 것이 아닙니다.
</credits>
