# 프론트엔드 (Frontend) - DHH Rails 스타일

<turbo_patterns>
## Turbo 패턴

**Turbo Streams**를 이용한 부분 업데이트:
```erb
<%# app/views/cards/closures/create.turbo_stream.erb %>
<%= turbo_stream.replace @card %>
```

**Morphing**을 이용한 복잡한 업데이트:
```ruby
render turbo_stream: turbo_stream.morph(@card)
```

**Global morphing** - 레이아웃에서 활성화:
```ruby
turbo_refreshes_with method: :morph, scroll: :preserve
```

`cached: true`를 이용한 **조각 캐싱(Fragment caching)**:
```erb
<%= render partial: "card", collection: @cards, cached: true %>
```

**ViewComponents 미사용** - 표준 partial만으로 충분합니다.
</turbo_patterns>

<turbo_morphing>
## Turbo Morphing 베스트 프랙티스

클라이언트 상태 복원을 위해 **morph 이벤트 리스닝**:
```javascript
document.addEventListener("turbo:morph-element", (event) => {
  // morph 후 클라이언트 측 상태 복원
})
```

**영구 요소(Permanent elements)** - data 속성으로 morphing 건너뛰기:
```erb
<div data-turbo-permanent id="notification-count">
  <%= @count %>
</div>
```

**프레임 morphing** - refresh 속성 추가:
```erb
<%= turbo_frame_tag :assignment, src: path, refresh: :morph %>
```

**일반적인 문제와 해결책:**

| 문제 | 해결책 |
|---------|----------|
| 타이머가 업데이트되지 않음 | morph 이벤트 리스너에서 초기화/재시작 |
| 폼이 리셋됨 | 폼 섹션을 turbo frame으로 감싸기 |
| 페이지네이션이 깨짐 | `refresh: :morph`가 설정된 turbo frame 사용 |
| 교체 시 깜빡임 | replace 대신 morph로 전환 |
| localStorage 손실 | `turbo:morph-element` 리스닝 후 상태 복원 |
</turbo_morphing>

<turbo_frames>
## Turbo Frames

스피너를 포함한 **지연 로딩(Lazy loading)**:
```erb
<%= turbo_frame_tag "menu",
      src: menu_path,
      loading: :lazy do %>
  <div class="spinner">Loading...</div>
<% end %>
```

edit/view 토글을 이용한 **인라인 편집**:
```erb
<%= turbo_frame_tag dom_id(card, :edit) do %>
  <%= link_to "Edit", edit_card_path(card),
        data: { turbo_frame: dom_id(card, :edit) } %>
<% end %>
```

하드코딩 없이 **부모 프레임 타겟팅**:
```erb
<%= form_with model: @card, data: { turbo_frame: "_parent" } do |f| %>
```

**실시간 구독(Real-time subscriptions):**
```erb
<%= turbo_stream_from @card %>
<%= turbo_stream_from @card, :activity %>
```
</turbo_frames>

<stimulus_controllers>
## Stimulus 컨트롤러

Fizzy 앱에는 52개의 컨트롤러가 있으며, 62%는 재사용 가능하고 38%는 도메인 특화되어 있습니다.

**특징:**
- 컨트롤러당 단일 책임
- values/classes를 통한 설정
- 통신을 위한 이벤트 사용
- #를 사용한 프라이빗 메서드
- 대부분 50라인 미만

**예시:**

```javascript
// copy-to-clipboard (25라인)
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { content: String }

  copy() {
    navigator.clipboard.writeText(this.contentValue)
    this.#showFeedback()
  }

  #showFeedback() {
    this.element.classList.add("copied")
    setTimeout(() => this.element.classList.remove("copied"), 1500)
  }
}
```

```javascript
// auto-click (7라인)
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    this.element.click()
  }
}
```

```javascript
// toggle-class (31라인)
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static classes = ["toggle"]
  static values = { open: { type: Boolean, default: false } }

  toggle() {
    this.openValue = !this.openValue
  }

  openValueChanged() {
    this.element.classList.toggle(this.toggleClass, this.openValue)
  }
}
```

```javascript
// auto-submit (28라인) - 디바운스된 폼 제출
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 300 } }

  connect() {
    this.timeout = null
  }

  submit() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }

  disconnect() {
    clearTimeout(this.timeout)
  }
}
```

```javascript
// dialog (45라인) - 네이티브 HTML 다이얼로그 관리
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  open() {
    this.element.showModal()
  }

  close() {
    this.element.close()
    this.dispatch("closed")
  }

  clickOutside(event) {
    if (event.target === this.element) this.close()
  }
}
```

```javascript
// local-time (40라인) - 상대 시간 표시
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { datetime: String }

  connect() {
    this.#updateTime()
  }

  #updateTime() {
    const date = new Date(this.datetimeValue)
    const now = new Date()
    const diffMinutes = Math.floor((now - date) / 60000)

    if (diffMinutes < 60) {
      this.element.textContent = `${diffMinutes}m ago`
    } else if (diffMinutes < 1440) {
      this.element.textContent = `${Math.floor(diffMinutes / 60)}h ago`
    } else {
      this.element.textContent = `${Math.floor(diffMinutes / 1440)}d ago`
    }
  }
}
```
</stimulus_controllers>

<stimulus_best_practices>
## Stimulus 베스트 프랙티스

getAttribute 대신 **Values API** 사용:
```javascript
// 권장
static values = { delay: { type: Number, default: 300 } }

// 지양
this.element.getAttribute("data-delay")
```

**disconnect에서 정리(Cleanup):**
```javascript
disconnect() {
  clearTimeout(this.timeout)
  this.observer?.disconnect()
  document.removeEventListener("keydown", this.boundHandler)
}
```

**액션 필터** - `:self`는 버블링을 방지합니다:
```erb
<div data-action="click->menu#toggle:self">
```

**헬퍼 추출** - 별도 모듈의 공통 유틸리티:
```javascript
// app/javascript/helpers/timing.js
export function debounce(fn, delay) {
  let timeout
  return (...args) => {
    clearTimeout(timeout)
    timeout = setTimeout(() => fn(...args), delay)
  }
}
```

느슨한 결합(loose coupling)을 위한 **이벤트 디스패칭**:
```javascript
this.dispatch("selected", { detail: { id: this.idValue } })
```
</stimulus_best_practices>

<view_helpers>
## 뷰 헬퍼 (Stimulus 통합)

**다이얼로그 헬퍼:**
```ruby
def dialog_tag(id, &block)
  tag.dialog(
    id: id,
    data: {
      controller: "dialog",
      action: "click->dialog#clickOutside keydown.esc->dialog#close"
    },
    &block
  )
end
```

**자동 제출 폼 헬퍼:**
```ruby
def auto_submit_form_with(model:, delay: 300, **options, &block)
  form_with(
    model: model,
    data: {
      controller: "auto-submit",
      auto_submit_delay_value: delay,
      action: "input->auto-submit#submit"
    },
    **options,
    &block
  )
end
```

**복사 버튼 헬퍼:**
```ruby
def copy_button(content:, label: "Copy")
  tag.button(
    label,
    data: {
      controller: "copy",
      copy_content_value: content,
      action: "click->copy#copy"
    }
  )
end
```
</view_helpers>

<css_architecture>
## CSS 아키텍처

전처리기 없이 현대적 기능을 갖춘 Vanilla CSS를 사용합니다.

캐스케이드 제어를 위한 **CSS @layer**:
```css
@layer reset, base, components, modules, utilities;

@layer reset {
  *, *::before, *::after { box-sizing: border-box; }
}

@layer base {
  body { font-family: var(--font-sans); }
}

@layer components {
  .btn { /* 버튼 스타일 */ }
}

@layer modules {
  .card { /* 카드 모듈 스타일 */ }
}

@layer utilities {
  .hidden { display: none; }
}
```

지각적 균일성을 위한 **OKLCH 컬러 시스템**:
```css
:root {
  --color-primary: oklch(60% 0.15 250);
  --color-success: oklch(65% 0.2 145);
  --color-warning: oklch(75% 0.15 85);
  --color-danger: oklch(55% 0.2 25);
}
```

CSS 변수를 통한 **다크 모드**:
```css
:root {
  --bg: oklch(98% 0 0);
  --text: oklch(20% 0 0);
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg: oklch(15% 0 0);
    --text: oklch(90% 0 0);
  }
}
```

**네이티브 CSS 중첩 (Nesting):**
```css
.card {
  padding: var(--space-4);

  & .title {
    font-weight: bold;
  }

  &:hover {
    background: var(--bg-hover);
  }
}
```

Tailwind의 수백 개 유틸리티 대신 **약 60개의 최소 유틸리티**만 사용합니다.

**사용된 현대적 기능:**
- 진입 애니메이션을 위한 `@starting-style`
- 색상 조작을 위한 `color-mix()`
- 부모 선택을 위한 `:has()`
- 논리적 속성 (Logical properties) (`margin-inline`, `padding-block`)
- 컨테이너 쿼리 (Container queries)
</css_architecture>

<view_patterns>
## 뷰 패턴

**표준 partial** - ViewComponents 미사용:
```erb
<%# app/views/cards/_card.html.erb %>
<article id="<%= dom_id(card) %>" class="card">
  <%= render "cards/header", card: card %>
  <%= render "cards/body", card: card %>
  <%= render "cards/footer", card: card %>
</article>
```

**조각 캐싱(Fragment caching):**
```erb
<% cache card do %>
  <%= render "cards/card", card: card %>
<% end %>
```

**컬렉션 캐싱(Collection caching):**
```erb
<%= render partial: "card", collection: @cards, cached: true %>
```

**단순한 컴포넌트 명명** - 엄격한 BEM 지양:
```css
.card { }
.card .title { }
.card .actions { }
.card.golden { }
.card.closed { }
```
</view_patterns>

<caching_with_personalization>
## 캐시 내 사용자별 콘텐츠

캐시 유지를 위해 개인화 요소를 클라이언트 측 JavaScript로 이동하십시오:

```erb
<%# 캐시 가능한 조각 %>
<% cache card do %>
  <article class="card"
           data-creator-id="<%= card.creator_id %>"
           data-controller="ownership"
           data-ownership-current-user-value="<%= Current.user.id %>">
    <button data-ownership-target="ownerOnly" class="hidden">Delete</button>
  </article>
<% end %>
```

```javascript
// 캐시 히트 후 사용자별 요소 노출
export default class extends Controller {
  static values = { currentUser: Number }
  static targets = ["ownerOnly"]

  connect() {
    const creatorId = parseInt(this.element.dataset.creatorId)
    if (creatorId === this.currentUserValue) {
      this.ownerOnlyTargets.forEach(el => el.classList.remove("hidden"))
    }
  }
}
```

**동적 콘텐츠를 별도 프레임으로 분리:**
```erb
<% cache [card, board] do %>
  <article class="card">
    <%= turbo_frame_tag card, :assignment,
          src: card_assignment_path(card),
          refresh: :morph %>
  </article>
<% end %>
```

할당 드롭다운은 부모 캐시를 무효화하지 않고 독립적으로 업데이트됩니다.
</caching_with_personalization>

<broadcasting>
## Turbo Streams를 이용한 브로드캐스팅

실시간 업데이트를 위한 **모델 콜백**:
```ruby
class Card < ApplicationRecord
  include Broadcastable

  after_create_commit :broadcast_created
  after_update_commit :broadcast_updated
  after_destroy_commit :broadcast_removed

  private
    def broadcast_created
      broadcast_append_to [Current.account, board], :cards
    end

    def broadcast_updated
      broadcast_replace_to [Current.account, board], :cards
    end

    def broadcast_removed
      broadcast_remove_to [Current.account, board], :cards
    end
end
```

`[Current.account, resource]` 패턴을 사용하여 **테넌트별 범위 지정**.
</broadcasting>
