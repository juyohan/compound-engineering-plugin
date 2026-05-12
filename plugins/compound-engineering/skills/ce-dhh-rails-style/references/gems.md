# 젬 (Gems) - DHH Rails 스타일

<what_they_use>
## 37signals가 사용하는 것

**핵심 Rails 스택:**
- turbo-rails, stimulus-rails, importmap-rails
- propshaft (에셋 파이프라인)

**데이터베이스 기반 서비스 (Solid 제품군):**
- solid_queue - 백그라운드 작업
- solid_cache - 캐싱
- solid_cable - 웹소켓/Action Cable

**인증 및 보안:**
- bcrypt (패스워드 해싱이 필요한 경우)

**자체 제작 젬:**
- geared_pagination (커서 기반 페이지네이션)
- lexxy (리치 텍스트 에디터)
- mittens (메일러 유틸리티)

**유틸리티:**
- rqrcode (QR 코드 생성)
- redcarpet + rouge (마크다운 렌더링)
- web-push (푸시 알림)

**배포 및 운영:**
- kamal (Docker 배포)
- thruster (HTTP/2 프록시)
- mission_control-jobs (작업 모니터링)
- autotuner (GC 튜닝)
</what_they_use>

<what_they_avoid>
## 그들이 의도적으로 피하는 것

**인증 (Authentication):**
```
devise → 커스텀 ~150라인 인증
```
이유: 완전한 제어 가능, 매직 링크를 통한 패스워드 책임 제거, 더 단순함.

**권한 부여 (Authorization):**
```
pundit/cancancan → 모델에서의 단순한 역할 확인
```
이유: 대부분의 앱에는 정책(policy) 객체가 필요하지 않습니다. 모델의 메서드로 충분합니다:
```ruby
class Board < ApplicationRecord
  def editable_by?(user)
    user.admin? || user == creator
  end
end
```

**백그라운드 작업 (Background Jobs):**
```
sidekiq → Solid Queue
```
이유: 데이터베이스 기반은 Redis가 필요 없음을 의미하며, 동일한 트랜잭션 보장을 제공합니다.

**캐싱 (Caching):**
```
redis → Solid Cache
```
이유: 데이터베이스가 이미 존재하므로 인프라가 단순해집니다.

**검색 (Search):**
```
elasticsearch → 커스텀 샤딩 검색
```
이유: 외부 서비스 의존성 없이 필요한 기능을 정확히 구축했습니다.

**뷰 레이어 (View Layer):**
```
view_component → 표준 partial
```
이유: partial만으로도 충분히 작동합니다. ViewComponent는 그들의 유즈케이스에서 명확한 이점 없이 복잡성만 더합니다.

**API:**
```
GraphQL → Turbo를 포함한 REST
```
이유: 양 끝단을 직접 제어할 때는 REST로 충분합니다. GraphQL의 복잡성이 정당화되지 않습니다.

**Factory:**
```
factory_bot → Fixture
```
이유: Fixture가 더 단순하고 빠르며, 데이터 관계에 대해 미리 생각하도록 유도합니다.

**서비스 객체 (Service Objects):**
```
Interactor, Trailblazer → Fat 모델
```
이유: 비즈니스 로직은 모델에 머뭅니다. `CardCloser.call(card)` 대신 `card.close` 같은 메서드를 사용합니다.

**폼 객체 (Form Objects):**
```
Reform, dry-validation → params.expect + 모델 검증
```
이유: Rails 7.1의 `params.expect`는 충분히 깔끔합니다. 모델의 컨텍스트별 검증을 활용합니다.

**데코레이터 (Decorators):**
```
Draper → 뷰 헬퍼 + partial
```
이유: 헬퍼와 partial이 더 단순합니다. 데코레이터의 간접 참조(indirection)가 없습니다.

**CSS:**
```
Tailwind, Sass → 네이티브 CSS
```
이유: 현대적 CSS에는 중첩(nesting), 변수, 레이어가 있습니다. 빌드 단계가 필요 없습니다.

**프론트엔드 (Frontend):**
```
React, Vue, SPAs → Turbo + Stimulus
```
이유: 약간의 JS가 가미된 서버 렌더링 HTML을 선호합니다. SPA의 복잡성이 정당화되지 않습니다.

**테스트 (Testing):**
```
RSpec → Minitest
```
이유: 더 단순하고, 부팅이 빠르며, DSL 마법이 적고, Rails와 함께 제공됩니다.
</what_they_avoid>

<testing_philosophy>
## 테스트 철학

**Minitest** - 더 단순하고 빠름:
```ruby
class CardTest < ActiveSupport::TestCase
  test "closing creates closure" do
    card = cards(:one)
    assert_difference -> { Card::Closure.count } do
      card.close
    end
    assert card.closed?
  end
end
```

**Fixture** - 한 번 로드되고 결정적임:
```yaml
# test/fixtures/cards.yml
open_card:
  title: Open Card
  board: main
  creator: alice

closed_card:
  title: Closed Card
  board: main
  creator: bob
```

ERB를 통한 **동적 타임스탬프**:
```yaml
recent:
  title: Recent
  created_at: <%= 1.hour.ago %>

old:
  title: Old
  created_at: <%= 1.month.ago %>
```

시간 의존적 테스트를 위한 **시간 여행(Time travel)**:
```ruby
test "expires after 15 minutes" do
  magic_link = MagicLink.create!(user: users(:alice))

  travel 16.minutes

  assert magic_link.expired?
end
```

외부 API를 위한 **VCR**:
```ruby
VCR.use_cassette("stripe/charge") do
  charge = Stripe::Charge.create(amount: 1000)
  assert charge.paid
end
```

**기능과 함께 테스트를 배포** - 전이나 후가 아닌, 동일한 커밋에 함께 포함합니다.
</testing_philosophy>

<decision_framework>
## 의사결정 프레임워크

젬을 추가하기 전 다음을 자문하십시오:

1. **순수 Rails(vanilla Rails)로 가능한가?**
   - ActiveRecord는 Sequel이 할 수 있는 대부분을 할 수 있습니다.
   - ActionMailer는 이메일을 잘 처리합니다.
   - ActiveJob은 대부분의 작업 요구사항에 부합합니다.

2. **복잡성을 감수할 가치가 있는가?**
   - 10,000라인의 젬 vs 150라인의 커스텀 코드.
   - 자신의 코드를 더 잘 이해하게 될 것입니다.
   - 업그레이드 시 골칫거리가 적습니다.

3. **인프라를 추가하는가?**
   - Redis? 데이터베이스 기반 대안을 고려하십시오.
   - 외부 서비스? 자체 구축을 고려하십시오.
   - 단순한 인프라 = 더 적은 실패 모드.

4. **신뢰할 만한 사람의 것인가?**
   - 37signals 젬: 대규모 환경에서 검증되었습니다.
   - 잘 관리되고 집중된 젬: 대개 괜찮습니다.
   - 모든 것을 다 하는 젬: 아마 과할 것입니다.

**그들의 철학:**
> "젬을 찾기 전에 솔루션을 먼저 구축하라."

젬 반대론자가 아니라, 이해 중심론자입니다. 있을지도 모를 문제가 아니라, 실제로 당면한 문제를 진정으로 해결할 때 젬을 사용하십시오.
</decision_framework>

<gem_patterns>
## 젬 사용 패턴

**페이지네이션 (Pagination):**
```ruby
# geared_pagination - 커서 기반
class CardsController < ApplicationController
  def index
    @cards = @board.cards.geared(page: params[:page])
  end
end
```

**마크다운 (Markdown):**
```ruby
# redcarpet + rouge
class MarkdownRenderer
  def self.render(text)
    Redcarpet::Markdown.new(
      Redcarpet::Render::HTML.new(filter_html: true),
      autolink: true,
      fenced_code_blocks: true
    ).render(text)
  end
end
```

**백그라운드 작업 (Background jobs):**
```ruby
# solid_queue - Redis 불필요
class ApplicationJob < ActiveJob::Base
  queue_as :default
  # 데이터베이스 기반으로 즉시 작동합니다
end
```

**캐싱 (Caching):**
```ruby
# solid_cache - Redis 불필요
# config/environments/production.rb
config.cache_store = :solid_cache_store
```
</gem_patterns>
