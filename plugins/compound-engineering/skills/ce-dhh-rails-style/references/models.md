# 모델 (Models) - DHH Rails 스타일

<model_concerns>
## 수평적 동작을 위한 Concern

모델은 concern을 집중적으로 사용합니다. 전형적인 Card 모델에는 14개 이상의 concern이 포함될 수 있습니다:

```ruby
class Card < ApplicationRecord
  include Assignable
  include Attachments
  include Broadcastable
  include Closeable
  include Colored
  include Eventable
  include Golden
  include Mentions
  include Multistep
  include Pinnable
  include Postponable
  include Readable
  include Searchable
  include Taggable
  include Watchable
end
```

각 concern은 연관 관계(associations), scope, 메서드를 포함하며 독립적으로 구성됩니다.

**명명:** 능력을 설명하는 형용사 (`Closeable`, `Publishable`, `Watchable`)
</model_concerns>

<state_records>
## 불리언이 아닌 레코드로 상태 관리

불리언 컬럼 대신 별도의 레코드를 생성하십시오:

```ruby
# 지양:
closed: boolean
is_golden: boolean
postponed: boolean

# 레코드 생성:
class Card::Closure < ApplicationRecord
  belongs_to :card
  belongs_to :creator, class_name: "User"
end

class Card::Goldness < ApplicationRecord
  belongs_to :card
  belongs_to :creator, class_name: "User"
end

class Card::NotNow < ApplicationRecord
  belongs_to :card
  belongs_to :creator, class_name: "User"
end
```

**장점:**
- 자동 타임스탬프 (언제 발생했는지)
- 변경을 수행한 사람 추적 가능
- 조인과 `where.missing`을 통한 손쉬운 필터링
- 언제/누가 수행했는지 보여주는 풍부한 UI 가능

**모델 내부:**
```ruby
module Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
  end

  def closed?
    closure.present?
  end

  def close(creator: Current.user)
    create_closure!(creator: creator)
  end

  def reopen
    closure&.destroy
  end
end
```

**쿼리:**
```ruby
Card.joins(:closure)         # 닫힌 카드
Card.where.missing(:closure) # 열린 카드
```
</state_records>

<callbacks>
## 콜백 - 절제해서 사용

Fizzy 앱의 30개 파일 중 콜백 발생 횟수는 단 38회에 불과합니다. 지침:

**사용 권장:**
- 비동기 작업을 위한 `after_commit`
- 유도된 데이터(derived data)를 위한 `before_save`
- 부수 효과를 위한 `after_create_commit`

**지양:**
- 복잡한 콜백 체인
- 콜백 내의 비즈니스 로직
- 동기적인 외부 호출

```ruby
class Card < ApplicationRecord
  after_create_commit :notify_watchers_later
  before_save :update_search_index, if: :title_changed?

  private
    def notify_watchers_later
      NotifyWatchersJob.perform_later(self)
    end
end
```
</callbacks>

<scopes>
## Scope 명명

표준적인 scope 이름들:

```ruby
class Card < ApplicationRecord
  scope :chronologically, -> { order(created_at: :asc) }
  scope :reverse_chronologically, -> { order(created_at: :desc) }
  scope :alphabetically, -> { order(title: :asc) }
  scope :latest, -> { reverse_chronologically.limit(10) }

  # 표준적인 지연 로딩
  scope :preloaded, -> { includes(:creator, :assignees, :tags) }

  # 매개변수화된 scope
  scope :indexed_by, ->(column) { order(column => :asc) }
  scope :sorted_by, ->(column, direction = :asc) { order(column => direction) }
end
```
</scopes>

<poros>
## 순수 루비 객체 (POROs)

부모 모델 아래에 네임스페이스로 구분된 PORO들:

```ruby
# app/models/event/description.rb
class Event::Description
  def initialize(event)
    @event = event
  end

  def to_s
    # 이벤트 설명을 위한 프레젠테이션 로직
  end
end

# app/models/card/eventable/system_commenter.rb
class Card::Eventable::SystemCommenter
  def initialize(card)
    @card = card
  end

  def comment(message)
    # 비즈니스 로직
  end
end

# app/models/user/filtering.rb
class User::Filtering
  # 뷰 컨텍스트 번들링
end
```

**서비스 객체용으로 사용하지 않습니다.** 비즈니스 로직은 모델에 머뭅니다.
</poros>

<verbs_predicates>
## 메서드 명명

**동사(Verbs)** - 상태를 변경하는 액션:
```ruby
card.close
card.reopen
card.gild      # 골든으로 만들기
card.ungild
board.publish
board.archive
```

**술어(Predicates)** - 상태에서 유도된 쿼리:
```ruby
card.closed?    # closure.present?
card.golden?    # goldness.present?
board.published?
```

범용 setter를 **피하십시오**:
```ruby
# 나쁨
card.set_closed(true)
card.update_golden_status(false)

# 좋음
card.close
card.ungild
```
</verbs_predicates>

<validation_philosophy>
## 검증 철학 (Validation Philosophy)

모델에는 최소한의 검증만 둡니다. 폼이나 오퍼레이션 객체에서 컨텍스트별 검증을 사용하십시오:

```ruby
# 모델 - 최소한의 검증
class User < ApplicationRecord
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
end

# 폼 객체 - 컨텍스트별 검증
class Signup
  include ActiveModel::Model

  attr_accessor :email, :name, :terms_accepted

  validates :email, :name, presence: true
  validates :terms_accepted, acceptance: true

  def save
    return false unless valid?
    User.create!(email: email, name: name)
  end
end
```

데이터 무결성을 위해 모델 검증보다는 **데이터베이스 제약 조건**을 선호하십시오:
```ruby
# migration
add_index :users, :email, unique: true
add_foreign_key :cards, :boards
```
</validation_philosophy>

<error_handling>
## "Let It Crash" 철학

실패 시 예외를 발생시키는 bang(!) 메서드를 사용하십시오:

```ruby
# 권장 - 실패 시 예외 발생
@card = Card.create!(card_params)
@card.update!(title: new_title)
@comment.destroy!

# 지양 - 소리 없는 실패
@card = Card.create(card_params)  # 실패 시 false 반환
if @card.save
  # ...
end
```

에러가 자연스럽게 전파되도록 두십시오. Rails는 `ActiveRecord::RecordInvalid`를 422 응답으로 처리합니다.
</error_handling>

<default_values>
## Lambda를 이용한 기본값

Current를 사용하는 연관 관계에는 lambda 기본값을 사용하십시오:

```ruby
class Card < ApplicationRecord
  belongs_to :creator, class_name: "User", default: -> { Current.user }
  belongs_to :account, default: -> { Current.account }
end

class Comment < ApplicationRecord
  belongs_to :commenter, class_name: "User", default: -> { Current.user }
end
```

Lambda는 생성 시점에 동적으로 해결됨을 보장합니다.
</default_values>

<rails_71_patterns>
## Rails 7.1+ 모델 패턴

**Normalizes** - 검증 전 데이터 정규화:
```ruby
class User < ApplicationRecord
  normalizes :email, with: ->(email) { email.strip.downcase }
  normalizes :phone, with: ->(phone) { phone.gsub(/\D/, "") }
end
```

**Delegated Types** - 다형성 연관 관계(polymorphic associations) 대체:
```ruby
class Message < ApplicationRecord
  delegated_type :messageable, types: %w[Comment Reply Announcement]
end

# 이제 다음을 얻을 수 있습니다:
message.comment?        # Comment인 경우 true
message.comment         # Comment 반환
Message.comments        # Comment 메시지용 scope
```

**Store Accessor** - 구조화된 JSON 저장:
```ruby
class User < ApplicationRecord
  store :settings, accessors: [:theme, :notifications_enabled], coder: JSON
end

user.theme = "dark"
user.notifications_enabled = true
```
</rails_71_patterns>

<concern_guidelines>
## Concern 지침

- concern당 **50-150라인** (대부분 ~100라인)
- **응집력** - 관련된 기능만 포함
- **능력 중심의 명명** - `CardHelpers`가 아닌 `Closeable`, `Watchable`
- **독립성** - 연관 관계, scope, 메서드를 함께 구성
- **단순 조직화가 아님** - 진정한 재사용이 필요할 때 생성

캐시 무효화를 위한 **Touch 체인**:
```ruby
class Comment < ApplicationRecord
  belongs_to :card, touch: true
end

class Card < ApplicationRecord
  belongs_to :board, touch: true
end
```

댓글이 업데이트되면 카드의 `updated_at`이 변경되고, 이것이 보드까지 전파됩니다.

관련된 업데이트를 위한 **트랜잭션 래핑**:
```ruby
class Card < ApplicationRecord
  def close(creator: Current.user)
    transaction do
      create_closure!(creator: creator)
      record_event(:closed)
      notify_watchers_later
    end
  end
end
```
</concern_guidelines>
