# ⚡ Optimization

> 토스트 애니메이션의 렌더링 최적화 전략과 운영 관측

---

## 🚪 Portal 과 stacking context

토스트 컨테이너는 컴포넌트 트리 안이 아니라 **Portal 로 `document.body` 하위**에 렌더링한다.

`z-index` 만으로는 부족하다. 부모에 `overflow: hidden`, `transform`, `filter`, `opacity` 중 하나라도 걸리면 새로운 stacking context 가 생성되고, 자식은 그 context 를 벗어날 수 없다.
`z-index: 999999` 를 줘도 부모 바깥의 요소 위로 올라가지 못한다.

```jsx
createPortal(<ToastContainer />, document.body)
```

값 조정이 아니라 구조로 해결하는 문제다.

---

## 📐 배치

컨테이너는 `position: fixed` 로 viewport 에 고정하고, 토스트는 **flex column** 으로 쌓는다.

```css
.toast-container {
    position: fixed;
    right: 16px;
    display: flex;
    gap: 8px;
    pointer-events: none;       /* 컨테이너 자체는 클릭을 막지 않는다 */
}

/* 새 토스트가 항상 모서리 쪽에 오도록 방향을 맞춘다 */
.toast-container[data-position="bottom-right"] { bottom: 16px; flex-direction: column; }
.toast-container[data-position="top-right"]    { top: 16px;    flex-direction: column-reverse; }

.toast-item { pointer-events: auto; }
```

**오프셋을 직접 계산하지 않는 이유**
토스트마다 높이가 다르면 다음 토스트의 위치를 구하려고 `getBoundingClientRect()` 를 호출하게 되고, 이는 강제 동기 레이아웃(forced reflow)을 유발한다. reflow 를 피하려던 목적과 충돌한다.
flex 에게 세로 배치를 맡기면 높이 측정이 필요 없다.

> 다만 flex 배치도 진입 순간에는 형제 토스트가 밀리는 레이아웃 변화가 생긴다.
> 이 변화까지 부드럽게 만들려면 높이 측정과 FLIP 이 필요하므로, **1차 구현에서는 밀림을 허용**하고 진입 · 퇴장 애니메이션만 처리한다.

애니메이션은 진입 · 퇴장에만 적용한다.

```css
/* 등장한 방향으로 되돌아가며 사라지도록 위치별 오프셋을 변수로 둔다 */
.toast-container[data-position="bottom-right"] { --enter-from: 8px; }
.toast-container[data-position="top-right"]    { --enter-from: -8px; }

.toast-item {
    transform: translateY(var(--enter-from));
    opacity: 0;
    transition: transform 150ms ease-out, opacity 150ms ease-out;
}

.toast-item[data-state="visible"] { transform: translateY(0); opacity: 1; }
.toast-item[data-state="leaving"] {
    transform: translateY(var(--enter-from));
    opacity: 0;
    transition-duration: 120ms;
}
```

---

## 🎬 렌더링 파이프라인

| 단계 | 전략 | 효과 |
|------|------|------|
| **Layout** | `height` · `margin` · `top` 대신 `transform` 만 사용 | 애니메이션 중 Layout 재계산 회피 |
| **Paint** | 색 · 그림자는 정적으로 두고 애니메이션은 `opacity` 로만 | Paint 반복 제거 |
| **Composite** | `transform` + `opacity` 조합 → GPU 합성 레이어에서 처리 | Layout · Paint 를 건너뛴다 |

### `will-change` 는 한시적으로만

```js
element.style.willChange = "transform, opacity";   // 애니메이션 시작 직전
element.addEventListener("transitionend", () => {
    element.style.willChange = "auto";             // 끝나면 즉시 해제
}, { once: true });
```

`will-change` 는 합성 레이어를 미리 만들라는 힌트이고, 레이어마다 메모리를 소비한다.
상시로 걸어두면 최적화가 아니라 낭비이므로 등장 · 퇴장 구간에만 부여한다.

---

## ♿ 접근성 최적화

### live region 은 미리 마운트한다

```jsx
// 앱 마운트 시점부터 빈 채로 DOM 에 존재해야 한다
<div role="status" aria-live="polite"    aria-atomic="true" />
<div role="alert"  aria-live="assertive" aria-atomic="true" />
```

live region 엘리먼트와 그 안의 텍스트를 **동시에 삽입하면 일부 스크린 리더가 변경을 감지하지 못한다.**
컨테이너를 먼저 존재시키고 텍스트만 나중에 삽입해야 안정적으로 읽힌다.

| `type` | `role` | `aria-live` |
|--------|--------|-------------|
| `success` · `info` · `loading` | `status` | `polite` |
| `error` · `warning` | `alert` | `assertive` |

`assertive` 는 사용자의 현재 작업을 끊는 강한 알림이다.
따라서 **중복 제거 정책과 함께 설계해야 한다** — 같은 에러가 10번 발화하면 스크린 리더 사용자는 아무것도 할 수 없다.

### 모션 최소화

```css
@media (prefers-reduced-motion: reduce) {
    .toast-item { transition: opacity 100ms linear; transform: none; }
}
```

---

## ⏱️ 타이머 제어

자동 닫힘 타이머는 **hover · focus 중 일시정지**하고, 벗어나면 남은 시간부터 재개한다.

WCAG 2.2.1(Timing Adjustable) 요구사항이면서, 실용적인 이유가 더 크다.
`action` 버튼이 달린 토스트가 3초 뒤에 사라지면 **되돌리기 버튼을 누르는 것 자체가 불가능**하다.

```js
const timers = new Map();   // 잔여 시간 등 런타임 상태는 Toast 엔티티 밖에 둔다

const startTimer = (toast) => {
    if (!Number.isFinite(toast.duration)) return;   // error, loading 은 타이머 없음
    timers.set(toast.id, {
        remaining: toast.duration,
        startedAt: performance.now(),
        timerId: setTimeout(() => closeToast(toast.id, "timeout"), toast.duration),
    });
};

const pauseTimer = (id) => {
    const t = timers.get(id);
    if (!t) return;
    clearTimeout(t.timerId);
    t.remaining -= performance.now() - t.startedAt;
};

// 중복 발생으로 기존 토스트를 갱신했을 때 유지 시간을 다시 채운다
const restartTimer = (toast) => {
    clearTimeout(timers.get(toast.id)?.timerId);
    startTimer(toast);
};
```

컴포넌트 언마운트 시 남은 타이머를 정리해 누수를 막는다.

---

## 📊 Observability

토스트는 잠깐 떴다 사라지므로, 문제가 생겨도 **로그가 없으면 아무도 모른다.**

| 이벤트 | 페이로드 | 무엇을 보는가 |
|--------|----------|----------------|
| `toast_shown` | `type`, `source` | 어느 기능에서 얼마나 뜨는가 |
| `toast_dismissed` | `reason`, `visibleDuration` | 사용자가 직접 닫는 비율이 높은가 |
| `toast_duplicate_suppressed` | `dedupeKey`, `repeatCount` | 특정 에러가 폭주하고 있는가 |

```js
track("toast_dismissed", {
    toastId,
    reason,                                          // timeout | manual | route-change | programmatic
    visibleDuration: performance.now() - toast.shownAt,
});
```

읽는 방법은 다음과 같다.

- `manual` 닫힘 비율이 높다 → 유지 시간이 너무 길다
- `duplicate_suppressed` 가 급증한다 → 그 기능의 에러 처리 자체를 봐야 한다
