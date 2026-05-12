# 컨트롤러 (Controllers) - DHH Rails 스타일

<rest_mapping>
## 모든 것은 CRUD로 매핑됩니다

커스텀 액션은 새로운 리소스가 됩니다. 기존 리소스에 동사를 추가하는 대신, 명사 형태의 리소스를 생성하십시오:

```ruby
# 지양:
POST /cards/:id/close
DELETE /cards/:id/close
POST /cards/:id/archive

# 권장:
POST /cards/:id/closure      # closure 생성
DELETE /cards/:id/closure    # closure 삭제
POST /cards/:id/archival     # archival 생성
```

**37signals의 실제 사례:**
```ruby
resources :cards do
  resource :closure       # 닫기/다시 열기
  resource :goldness      # 중요 표시
  resource :not_now       # 연기하기
  resources :assignments  # 담당자 관리
end
```

각 리소스는 표준 CRUD 액션을 갖는 전용 컨트롤러를 가집니다.
</rest_mapping>

<controller_concerns>
## 공통 동작을 위한 Concern

컨트롤러는 concern을 광범위하게 사용합니다. 일반적인 패턴:

**CardScoped** - @card, @board를 로드하고 render_card_replacement 제공
```ruby
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card
  end

  private
    def set_card
      @card = Card.find(params[:card_id])
      @board = @card.board
    end

    def render_card_replacement
      render turbo_stream: turbo_stream.replace(@card)
    end
end
```

**BoardScoped** - @board 로드
**CurrentRequest** - 요청 데이터로 Current 채우기
**CurrentTimezone** - 사용자 타임존으로 요청 감싸기
**FilterScoped** - 복잡한 필터링 처리
**TurboFlash** - Turbo Stream을 통한 플래시 메시지
**ViewTransitions** - 페이지 새로고침 시 비활성화
**BlockSearchEngineIndexing** - X-Robots-Tag 헤더 설정
**RequestForgeryProtection** - Sec-Fetch-Site CSRF (현대적 브라우저)
</controller_concerns>

<authorization_patterns>
## 권한 부여 패턴 (Authorization Patterns)

컨트롤러는 before_action을 통해 권한을 확인하고, 모델은 권한의 의미를 정의합니다:

```ruby
# 컨트롤러 concern
module Authorization
  extend ActiveSupport::Concern

  private
    def ensure_can_administer
      head :forbidden unless Current.user.admin?
    end

    def ensure_is_staff_member
      head :forbidden unless Current.user.staff?
    end
end

# 사용법
class BoardsController < ApplicationController
  before_action :ensure_can_administer, only: [:destroy]
end
```

**모델 레벨 권한 부여:**
```ruby
class Board < ApplicationRecord
  def editable_by?(user)
    user.admin? || user == creator
  end

  def publishable_by?(user)
    editable_by?(user) && !published?
  end
end
```

권한 부여는 단순하고 가독성 있게 유지하며, 도메인 로직과 함께 위치시키십시오.
</authorization_patterns>

<security_concerns>
## 보안 Concern

**Sec-Fetch-Site CSRF 보호:**
현대적 브라우저는 Sec-Fetch-Site 헤더를 보냅니다. 이를 심층 방어(defense in depth)에 활용하십시오:

```ruby
module RequestForgeryProtection
  extend ActiveSupport::Concern

  included do
    before_action :verify_request_origin
  end

  private
    def verify_request_origin
      return if request.get? || request.head?
      return if %w[same-origin same-site].include?(
        request.headers["Sec-Fetch-Site"]&.downcase
      )
      # 구형 브라우저를 위해 토큰 검증으로 폴백
      verify_authenticity_token
    end
end
```

**Rate Limiting (Rails 8+):**
```ruby
class MagicLinksController < ApplicationController
  rate_limit to: 10, within: 15.minutes, only: :create
end
```

인증 엔드포인트, 이메일 전송, 외부 API 호출, 리소스 생성 등에 적용하십시오.
</security_concerns>

<request_context>
## 요청 컨텍스트 Concern

**CurrentRequest** - HTTP 메타데이터로 Current 채우기:
```ruby
module CurrentRequest
  extend ActiveSupport::Concern

  included do
    before_action :set_current_request
  end

  private
    def set_current_request
      Current.request_id = request.request_id
      Current.user_agent = request.user_agent
      Current.ip_address = request.remote_ip
      Current.referrer = request.referrer
    end
end
```

**CurrentTimezone** - 사용자 타임존으로 요청 감싸기:
```ruby
module CurrentTimezone
  extend ActiveSupport::Concern

  included do
    around_action :set_timezone
    helper_method :timezone_from_cookie
  end

  private
    def set_timezone
      Time.use_zone(timezone_from_cookie) { yield }
    end

    def timezone_from_cookie
      cookies[:timezone] || "UTC"
    end
end
```

**SetPlatform** - 모바일/데스크톱 감지:
```ruby
module SetPlatform
  extend ActiveSupport::Concern

  included do
    helper_method :platform
  end

  def platform
    @platform ||= request.user_agent&.match?(/Mobile|Android/) ? :mobile : :desktop
  end
end
```
</request_context>

<turbo_responses>
## Turbo Stream 응답

부분 업데이트를 위해 Turbo Stream을 사용하십시오:

```ruby
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    @card.close
    render_card_replacement
  end

  def destroy
    @card.reopen
    render_card_replacement
  end
end
```

복잡한 업데이트에는 morphing을 사용하십시오:
```ruby
render turbo_stream: turbo_stream.morph(@card)
```
</turbo_responses>

<api_patterns>
## API 디자인

동일한 컨트롤러, 다른 포맷. 응답 컨벤션:

```ruby
def create
  @card = Card.create!(card_params)

  respond_to do |format|
    format.html { redirect_to @card }
    format.json { head :created, location: @card }
  end
end

def update
  @card.update!(card_params)

  respond_to do |format|
    format.html { redirect_to @card }
    format.json { head :no_content }
  end
end

def destroy
  @card.destroy

  respond_to do |format|
    format.html { redirect_to cards_path }
    format.json { head :no_content }
  end
end
```

**상태 코드:**
- Create: 201 Created + Location 헤더
- Update: 204 No Content
- Delete: 204 No Content
- Bearer 토큰 인증
</api_patterns>

<http_caching>
## HTTP 캐싱

ETag과 조건부 GET을 광범위하게 사용하십시오:

```ruby
class CardsController < ApplicationController
  def show
    @card = Card.find(params[:id])
    fresh_when etag: [@card, Current.user.timezone]
  end

  def index
    @cards = @board.cards.preloaded
    fresh_when etag: [@cards, @board.updated_at]
  end
end
```

핵심 통찰: 시간은 사용자 타임존에 따라 서버에서 렌더링되므로, 다른 타임존에 잘못된 시간을 제공하는 것을 방지하기 위해 타임존이 ETag에 영향을 미쳐야 합니다.

**ApplicationController 글로벌 ETag:**
```ruby
class ApplicationController < ActionController::Base
  etag { "v1" }  # 모든 캐시를 무효화하려면 버전을 올리십시오
end
```

캐시 무효화를 위해 연관 관계(associations)에 `touch: true`를 사용하십시오.
</http_caching>
