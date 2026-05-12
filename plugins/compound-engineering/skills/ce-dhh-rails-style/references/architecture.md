# 아키텍처 (Architecture) - DHH Rails 스타일

<routing>
## 라우팅 (Routing)

모든 것은 CRUD로 매핑됩니다. 관련된 액션은 중첩된 리소스(nested resources)를 사용하십시오:

```ruby
Rails.application.routes.draw do
  resources :boards do
    resources :cards do
      resource :closure
      resource :goldness
      resource :not_now
      resources :assignments
      resources :comments
    end
  end
end
```

**동사에서 명사로의 변환:**
| 액션 | 리소스 |
|--------|----------|
| 카드를 닫기 | `card.closure` |
| 보드를 주시(watch)하기 | `board.watching` |
| 골든(golden)으로 표시하기 | `card.goldness` |
| 카드를 아카이브하기 | `card.archival` |

**얕은 중첩(Shallow nesting)** - 깊은 URL을 피하십시오:
```ruby
resources :boards do
  resources :cards, shallow: true  # /boards/:id/cards 이지만, 개별 카드는 /cards/:id
end
```

**단수 리소스(Singular resources)** - 부모 당 하나만 존재하는 경우:
```ruby
resource :closure   # resources가 아님
resource :goldness
```

**URL 생성을 위한 resolve:**
```ruby
# config/routes.rb
resolve("Comment") { |comment| [comment.card, anchor: dom_id(comment)] }

# 이제 url_for(@comment)가 올바르게 작동합니다.
```
</routing>

<multi_tenancy>
## 멀티 테넌시 (경로 기반)

**미들웨어가 URL 접두사에서 테넌트를 추출합니다:**

```ruby
# lib/tenant_extractor.rb
class TenantExtractor
  def initialize(app)
    @app = app
  end

  def call(env)
    path = env["PATH_INFO"]
    if match = path.match(%r{^/(\d+)(/.*)?$})
      env["SCRIPT_NAME"] = "/#{match[1]}"
      env["PATH_INFO"] = match[2] || "/"
    end
    @app.call(env)
  end
end
```

**테넌트 경로별 쿠키 범위(scoping):**
```ruby
# 테넌트 경로로 범위가 지정된 쿠키
cookies.signed[:session_id] = {
  value: session.id,
  path: "/#{Current.account.id}"
}
```

**백그라운드 작업 컨텍스트** - 테넌트 직렬화:
```ruby
class ApplicationJob < ActiveJob::Base
  around_perform do |job, block|
    Current.set(account: job.arguments.first.account) { block.call }
  end
end
```

**반복 작업(Recurring jobs)**은 모든 테넌트를 순회해야 합니다:
```ruby
class DailyDigestJob < ApplicationJob
  def perform
    Account.find_each do |account|
      Current.set(account: account) do
        send_digest_for(account)
      end
    end
  end
end
```

**컨트롤러 보안** - 항상 테넌트를 통해 범위를 제한하십시오:
```ruby
# 권장 - 사용자가 접근 가능한 레코드를 통해 범위 제한
@card = Current.user.accessible_cards.find(params[:id])

# 지양 - 직접 조회
@card = Card.find(params[:id])
```
</multi_tenancy>

<authentication>
## 인증 (Authentication)

커스텀 패스워드리스 매직 링크 인증 (총 ~150라인):

```ruby
# app/models/session.rb
class Session < ApplicationRecord
  belongs_to :user

  before_create { self.token = SecureRandom.urlsafe_base64(32) }
end

# app/models/magic_link.rb
class MagicLink < ApplicationRecord
  belongs_to :user

  before_create do
    self.code = SecureRandom.random_number(100_000..999_999).to_s
    self.expires_at = 15.minutes.from_now
  end

  def expired?
    expires_at < Time.current
  end
end
```

**Devise를 사용하지 않는 이유:**
- 거대한 의존성 대신 ~150라인의 코드로 구현 가능
- 패스워드 저장에 따른 책임(liability) 없음
- 사용자에게 더 단순한 UX 제공
- 흐름을 완전히 제어 가능

**API용 Bearer 토큰:**
```ruby
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :authenticate
  end

  private
    def authenticate
      if bearer_token = request.headers["Authorization"]&.split(" ")&.last
        Current.session = Session.find_by(token: bearer_token)
      else
        Current.session = Session.find_by(id: cookies.signed[:session_id])
      end

      redirect_to login_path unless Current.session
    end
end
```
</authentication>

<background_jobs>
## 백그라운드 작업 (Background Jobs)

Job은 모델 메서드를 호출하는 얇은 래퍼(wrapper)입니다:

```ruby
class NotifyWatchersJob < ApplicationJob
  def perform(card)
    card.notify_watchers
  end
end
```

**명명 규칙:**
- 비동기 실행 시 `_later` 접미사: `card.notify_watchers_later`
- 즉시 실행 시 `_now` 접미사: `card.notify_watchers_now`

```ruby
module Watchable
  def notify_watchers_later
    NotifyWatchersJob.perform_later(self)
  end

  def notify_watchers_now
    NotifyWatchersJob.perform_now(self)
  end

  def notify_watchers
    watchers.each do |watcher|
      WatcherMailer.notification(watcher, self).deliver_later
    end
  end
end
```

**Solid Queue를 이용한 데이터베이스 기반 작업:**
- Redis 불필요
- 데이터와 동일한 트랜잭션 보장
- 단순한 인프라 구성

**트랜잭션 안전성:**
```ruby
# config/application.rb
config.active_job.enqueue_after_transaction_commit = true
```

**유형별 에러 처리:**
```ruby
class DeliveryJob < ApplicationJob
  # 일시적 에러 - backoff와 함께 재시도
  retry_on Net::OpenTimeout, Net::ReadTimeout,
           Resolv::ResolvError,
           wait: :polynomially_longer

  # 영구적 에러 - 로그 기록 후 폐기
  discard_on Net::SMTPSyntaxError do |job, error|
    Sentry.capture_exception(error, level: :info)
  end
end
```

**Continuable을 이용한 배치 처리:**
```ruby
class ProcessCardsJob < ApplicationJob
  include ActiveJob::Continuable

  def perform
    Card.in_batches.each_record do |card|
      checkpoint!  # 중단될 경우 여기서부터 재개
      process(card)
    end
  end
end
```
</background_jobs>

<database_patterns>
## 데이터베이스 패턴

**기본 키로 UUID 사용** (시간순 정렬 가능한 UUIDv7):
```ruby
# migration
create_table :cards, id: :uuid do |t|
  t.references :board, type: :uuid, foreign_key: true
end
```

장점: ID 추측 불가, 분산 환경 친화적, 클라이언트 측 생성 가능.

**상태를 레코드로 관리** (불리언이 아님):
```ruby
# closed: boolean 대신
class Card::Closure < ApplicationRecord
  belongs_to :card
  belongs_to :creator, class_name: "User"
end

# 쿼리는 조인이 됨
Card.joins(:closure)          # 닫힌 카드
Card.where.missing(:closure)  # 열린 카드
```

**영구 삭제(Hard deletes)** - 소프트 삭제 지양:
```ruby
# 그냥 삭제
card.destroy!

# 이력을 위해 이벤트를 기록
card.record_event(:deleted, by: Current.user)
```

쿼리를 단순화하고 감사를 위해 이벤트 로그를 사용합니다.

**성능을 위한 카운터 캐시(Counter caches):**
```ruby
class Comment < ApplicationRecord
  belongs_to :card, counter_cache: true
end

# 쿼리 없이 card.comments_count 사용 가능
```

**모든 테이블에 계정 범위(Account scoping) 적용:**
```ruby
class Card < ApplicationRecord
  belongs_to :account
  default_scope { where(account: Current.account) }
end
```
</database_patterns>

<current_attributes>
## Current Attributes

요청 범위(request-scoped) 상태를 위해 `Current`를 사용하십시오:

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :account, :request_id

  delegate :user, to: :session, allow_nil: true

  def account=(account)
    super
    Time.zone = account&.time_zone || "UTC"
  end
end
```

컨트롤러에서 설정:
```ruby
class ApplicationController < ActionController::Base
  before_action :set_current_request

  private
    def set_current_request
      Current.session = authenticated_session
      Current.account = Account.find(params[:account_id])
      Current.request_id = request.request_id
    end
end
```

앱 전반에서 사용:
```ruby
class Card < ApplicationRecord
  belongs_to :creator, default: -> { Current.user }
end
```
</current_attributes>

<caching>
## 캐싱 (Caching)

**ETag을 이용한 HTTP 캐싱:**
```ruby
fresh_when etag: [@card, Current.user.timezone]
```

**조각 캐싱(Fragment caching):**
```erb
<% cache card do %>
  <%= render card %>
<% end %>
```

**러시아 인형 캐싱 (Russian doll caching):**
```erb
<% cache @board do %>
  <% @board.cards.each do |card| %>
    <% cache card do %>
      <%= render card %>
    <% end %>
  <% end %>
<% end %>
```

**`touch: true`를 통한 캐시 무효화:**
```ruby
class Card < ApplicationRecord
  belongs_to :board, touch: true
end
```

**Solid Cache - 데이터베이스 기반 캐시:**
- Redis 불필요
- 애플리케이션 데이터와 일관성 유지
- 단순한 인프라 구성
</caching>

<configuration>
## 설정 (Configuration)

**기본값을 포함한 ENV.fetch:**
```ruby
# config/application.rb
config.active_job.queue_adapter = ENV.fetch("QUEUE_ADAPTER", "solid_queue").to_sym
config.cache_store = ENV.fetch("CACHE_STORE", "solid_cache").to_sym
```

**다중 데이터베이스:**
```yaml
# config/database.yml
production:
  primary:
    <<: *default
  cable:
    <<: *default
    migrations_paths: db/cable_migrate
  queue:
    <<: *default
    migrations_paths: db/queue_migrate
  cache:
    <<: *default
    migrations_paths: db/cache_migrate
```

**ENV를 통한 SQLite와 MySQL 간 전환:**
```ruby
adapter = ENV.fetch("DATABASE_ADAPTER", "sqlite3")
```

**ENV를 통해 확장 가능한 CSP:**
```ruby
config.content_security_policy do |policy|
  policy.default_src :self
  policy.script_src :self, *ENV.fetch("CSP_SCRIPT_SRC", "").split(",")
end
```
</configuration>

<testing>
## 테스트 (Testing)

RSpec 대신 **Minitest**:
```ruby
class CardTest < ActiveSupport::TestCase
  test "closing a card creates a closure" do
    card = cards(:one)

    card.close

    assert card.closed?
    assert_not_nil card.closure
  end
end
```

Factory 대신 **Fixture**:
```yaml
# test/fixtures/cards.yml
one:
  title: First Card
  board: main
  creator: alice

two:
  title: Second Card
  board: main
  creator: bob
```

컨트롤러용 **통합 테스트**:
```ruby
class CardsControllerTest < ActionDispatch::IntegrationTest
  test "closing a card" do
    card = cards(:one)
    sign_in users(:alice)

    post card_closure_path(card)

    assert_response :success
    assert card.reload.closed?
  end
end
```

**기능과 함께 테스트를 배포** - TDD가 우선은 아니더라도 동일한 커밋에 함께 포함되어야 합니다.

**보안 수정 시 항상 회귀 테스트(regression tests) 포함.**
</testing>

<events>
## 이벤트 트래킹 (Event Tracking)

이벤트는 단일 진실 공급원(single source of truth)입니다:

```ruby
class Event < ApplicationRecord
  belongs_to :creator, class_name: "User"
  belongs_to :eventable, polymorphic: true

  serialize :particulars, coder: JSON
end
```

**Eventable concern:**
```ruby
module Eventable
  extend ActiveSupport::Concern

  included do
    has_many :events, as: :eventable, dependent: :destroy
  end

  def record_event(action, particulars = {})
    events.create!(
      creator: Current.user,
      action: action,
      particulars: particulars
    )
  end
end
```

**이벤트 기반 웹훅** - 이벤트가 정식 소스가 됩니다.
</events>

<email_patterns>
## 이메일 패턴

**멀티 테넌트 URL 헬퍼:**
```ruby
class ApplicationMailer < ActionMailer::Base
  def default_url_options
    options = super
    if Current.account
      options[:script_name] = "/#{Current.account.id}"
    end
    options
  end
end
```

**타임존 인식 배달:**
```ruby
class NotificationMailer < ApplicationMailer
  def daily_digest(user)
    Time.use_zone(user.timezone) do
      @user = user
      @digest = user.digest_for_today
      mail(to: user.email, subject: "Daily Digest")
    end
  end
end
```

**배치 배달:**
```ruby
emails = users.map { |user| NotificationMailer.digest(user) }
ActiveJob.perform_all_later(emails.map(&:deliver_later))
```

**원클릭 구독 취소 (RFC 8058):**
```ruby
class ApplicationMailer < ActionMailer::Base
  after_action :set_unsubscribe_headers

  private
    def set_unsubscribe_headers
      headers["List-Unsubscribe-Post"] = "List-Unsubscribe=One-Click"
      headers["List-Unsubscribe"] = "<#{unsubscribe_url}>"
    end
end
```
</email_patterns>

<security_patterns>
## 보안 패턴

**XSS 방지** - 헬퍼에서 이스케이프:
```ruby
def formatted_content(text)
  # 먼저 이스케이프한 뒤, safe로 표시
  simple_format(h(text)).html_safe
end
```

**SSRF 보호:**
```ruby
# DNS를 한 번 해결하고 IP를 고정
def fetch_safely(url)
  uri = URI.parse(url)
  ip = Resolv.getaddress(uri.host)

  # 사설 네트워크 차단
  raise "Private IP" if private_ip?(ip)

  # 고정된 IP로 요청 수행
  Net::HTTP.start(uri.host, uri.port, ipaddr: ip) { |http| ... }
end

def private_ip?(ip)
  ip.start_with?("127.", "10.", "192.168.") ||
    ip.match?(/^172\.(1[6-9]|2[0-9]|3[0-1])\./)
end
```

**콘텐츠 보안 정책 (CSP):**
```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.script_src :self
    policy.style_src :self, :unsafe_inline
    policy.base_uri :none
    policy.form_action :self
    policy.frame_ancestors :self
  end
end
```

**ActionText 새니타이제이션(Sanitization):**
```ruby
# config/initializers/action_text.rb
Rails.application.config.after_initialize do
  ActionText::ContentHelper.allowed_tags = %w[
    strong em a ul ol li p br h1 h2 h3 h4 blockquote
  ]
end
```
</security_patterns>

<active_storage>
## Active Storage 패턴

**변형(Variant) 전처리:**
```ruby
class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb, resize_to_limit: [100, 100], preprocessed: true
    attachable.variant :medium, resize_to_limit: [300, 300], preprocessed: true
  end
end
```

**직접 업로드 만료** - 느린 연결을 위해 확장:
```ruby
# config/initializers/active_storage.rb
Rails.application.config.active_storage.service_urls_expire_in = 48.hours
```

**아바타 최적화** - blob으로 리다이렉트:
```ruby
def show
  expires_in 1.year, public: true
  redirect_to @user.avatar.variant(:thumb).processed.url, allow_other_host: true
end
```

**마이그레이션을 위한 미러 서비스:**
```yaml
# config/storage.yml
production:
  service: Mirror
  primary: amazon
  mirrors: [google]
```
</active_storage>
