# 테스트 (Testing) - DHH Rails 스타일

## 핵심 철학

"Fixture를 활용한 Minitest - 단순하고 빠르며 결정적입니다." 관습보다 실용주의를 우선하는 접근 방식입니다.

## RSpec 대신 Minitest를 사용하는 이유

- **더 단순함**: DSL 마법이 적고 순수 루비 어설션(assertions)을 사용합니다.
- **Rails와 함께 제공**: 추가 의존성이 필요하지 않습니다.
- **빠른 부팅**: 오버헤드가 적습니다.
- **순수 루비**: 배울 정교한 문법이 따로 없습니다.

## 테스트 데이터로서의 Fixture

Factory 대신 Fixture가 미리 로드된 데이터를 제공합니다:
- 한 번 로드되어 여러 테스트에서 재사용됩니다.
- 런타임 객체 생성 오버헤드가 없습니다.
- 관계의 가시성이 명시적입니다.
- 디버깅을 위한 결정적인(deterministic) ID를 제공합니다.

### Fixture 구조
```yaml
# test/fixtures/users.yml
david:
  identity: david
  account: basecamp
  role: admin

jason:
  identity: jason
  account: basecamp
  role: member

# test/fixtures/rooms.yml
watercooler:
  name: Water Cooler
  creator: david
  direct: false

# test/fixtures/messages.yml
greeting:
  body: Hello everyone!
  room: watercooler
  creator: david
```

### 테스트에서 Fixture 사용
```ruby
test "sending a message" do
  user = users(:david)
  room = rooms(:watercooler)

  # fixture 데이터를 이용한 테스트
end
```

### 동적 Fixture 값
ERB를 사용하여 시간 민감형 데이터를 생성할 수 있습니다:
```yaml
recent_card:
  title: Recent Card
  created_at: <%= 1.hour.ago %>

old_card:
  title: Old Card
  created_at: <%= 1.month.ago %>
```

## 테스트 구성

### 유닛 테스트 (Unit Tests)
setup 블록과 표준 어설션을 사용하여 비즈니스 로직을 검증합니다:

```ruby
class CardTest < ActiveSupport::TestCase
  setup do
    @card = cards(:one)
    @user = users(:david)
  end

  test "closing a card creates a closure" do
    assert_difference -> { Card::Closure.count } do
      @card.close(creator: @user)
    end

    assert @card.closed?
    assert_equal @user, @card.closure.creator
  end

  test "reopening a card destroys the closure" do
    @card.close(creator: @user)

    assert_difference -> { Card::Closure.count }, -1 do
      @card.reopen
    end

    refute @card.closed?
  end
end
```

### 통합 테스트 (Integration Tests)
전체 요청/응답 사이클을 테스트합니다:

```ruby
class CardsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:david)
    sign_in @user
  end

  test "closing a card" do
    card = cards(:one)

    post card_closure_path(card)

    assert_response :success
    assert card.reload.closed?
  end

  test "unauthorized user cannot close card" do
    sign_in users(:guest)
    card = cards(:one)

    post card_closure_path(card)

    assert_response :forbidden
    refute card.reload.closed?
  end
end
```

### 시스템 테스트 (System Tests)
Capybara를 사용한 브라우저 기반 테스트:

```ruby
class MessagesTest < ApplicationSystemTestCase
  test "sending a message" do
    sign_in users(:david)
    visit room_path(rooms(:watercooler))

    fill_in "Message", with: "Hello, world!"
    click_button "Send"

    assert_text "Hello, world!"
  end

  test "editing own message" do
    sign_in users(:david)
    visit room_path(rooms(:watercooler))

    within "#message_#{messages(:greeting).id}" do
      click_on "Edit"
    end

    fill_in "Message", with: "Updated message"
    click_button "Save"

    assert_text "Updated message"
  end

  test "drag and drop card to new column" do
    sign_in users(:david)
    visit board_path(boards(:main))

    card = find("#card_#{cards(:one).id}")
    target = find("#column_#{columns(:done).id}")

    card.drag_to target

    assert_selector "#column_#{columns(:done).id} #card_#{cards(:one).id}"
  end
end
```

## 고급 패턴

### 시간 테스트 (Time Testing)
결정적인 시간 의존적 어설션을 위해 `travel_to`를 사용하십시오:

```ruby
test "card expires after 30 days" do
  card = cards(:one)

  travel_to 31.days.from_now do
    assert card.expired?
  end
end
```

### VCR을 이용한 외부 API 테스트
HTTP 상호작용을 기록하고 재현합니다:

```ruby
test "fetches user data from API" do
  VCR.use_cassette("user_api") do
    user_data = ExternalApi.fetch_user(123)

    assert_equal "John", user_data[:name]
  end
end
```

### 백그라운드 작업 테스트
작업 예약 및 이메일 발송 여부를 확인합니다:

```ruby
test "closing card enqueues notification job" do
  card = cards(:one)

  assert_enqueued_with(job: NotifyWatchersJob, args: [card]) do
    card.close
  end
end

test "welcome email is sent on signup" do
  assert_emails 1 do
    Identity.create!(email: "new@example.com")
  end
end
```

### Turbo Streams 테스트
```ruby
test "message creation broadcasts to room" do
  room = rooms(:watercooler)

  assert_turbo_stream_broadcasts [room, :messages] do
    room.messages.create!(body: "Test", creator: users(:david))
  end
end
```

## 테스트 원칙

### 1. 관찰 가능한 동작(Observable Behavior) 테스트
코드가 어떻게 하는지가 아니라 무엇을 하는지에 집중하십시오:

```ruby
# ❌ 구현을 테스트함
test "calls notify method on each watcher" do
  card.expects(:notify).times(3)
  card.close
end

# ✅ 동작을 테스트함
test "watchers receive notifications when card closes" do
  assert_difference -> { Notification.count }, 3 do
    card.close
  end
end
```

### 2. 모든 것을 모킹(Mock)하지 마십시오

```ruby
# ❌ 과도하게 모킹된 테스트
test "sending message" do
  room = mock("room")
  user = mock("user")
  message = mock("message")

  room.expects(:messages).returns(stub(create!: message))
  message.expects(:broadcast_create)

  MessagesController.new.create
end

# ✅ 실제 객체를 테스트
test "sending message" do
  sign_in users(:david)
  post room_messages_url(rooms(:watercooler)),
    params: { message: { body: "Hello" } }

  assert_response :success
  assert Message.exists?(body: "Hello")
end
```

### 3. 기능과 함께 테스트를 배포
전이나 후가 아닌, 동일한 커밋에 함께 포함합니다. 엄격한 TDD(전)도, 지연된 테스트(후)도 아닙니다.

### 4. 보안 수정 시 항상 회귀 테스트 포함
모든 보안 수정에는 해당 취약점을 잡아낼 수 있는 테스트가 포함되어야 합니다.

### 5. 통합 테스트로 전체 워크플로 검증
개별 조각들만 테스트하지 말고, 그것들이 함께 잘 작동하는지 테스트하십시오.

## 파일 구성

```
test/
├── controllers/         # 컨트롤러 통합 테스트
├── fixtures/           # 모든 모델용 YAML fixture
├── helpers/            # 헬퍼 메서드 테스트
├── integration/        # API 통합 테스트
├── jobs/               # 백그라운드 작업 테스트
├── mailers/            # 메일러 테스트
├── models/             # 모델 유닛 테스트
├── system/             # 브라우저 기반 시스템 테스트
└── test_helper.rb      # 테스트 설정
```

## Test Helper 설정

```ruby
# test/test_helper.rb
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
require "rails/test_help"

class ActiveSupport::TestCase
  fixtures :all

  parallelize(workers: :number_of_processors)
end

class ActionDispatch::IntegrationTest
  include SignInHelper
end

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :headless_chrome
end
```

## 로그인 헬퍼 (Sign In Helper)

```ruby
# test/support/sign_in_helper.rb
module SignInHelper
  def sign_in(user)
    session = user.identity.sessions.create!
    cookies.signed[:session_id] = session.id
  end
end
```
