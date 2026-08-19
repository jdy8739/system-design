# 🗂️ Data Model

> Toast 컴포넌트의 타입 정의

---

## 🔤 기본 타입

```ts
type ToastType = "success" | "error" | "warning" | "info" | "loading";

type ToastPosition = "top-right" | "bottom-right";

type ToastStatus = "visible" | "queued";

type ToastDismissReason = "timeout" | "manual" | "route-change" | "programmatic";

interface ToastAction {
    label: string;
    onClick: () => void;
}
```

> 종류 필드의 이름은 `meaning` 대신 **`type`** 을 쓴다.
> 디자인 시스템과 UI 라이브러리에서 통용되는 명칭은 `type` · `variant` · `intent` · `severity` 이고, 호출부에서 가장 덜 놀라는 이름을 고른다.

---

## 📥 `CreateToastInput` — 호출자가 넘기는 값

```ts
interface CreateToastInput {
    type?: ToastType;
    message: string;
    description?: string;
    duration?: number;
    position?: ToastPosition;
    dismissible?: boolean;
    dedupeKey?: string;
    action?: ToastAction;
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `type` | `ToastType` _(optional)_ | 토스트의 종류. 기본값 `info` |
| `message` | `string` | 한 줄 요약. 제목 역할 |
| `description` | `string` _(optional)_ | 보조 설명. `message` 로 부족할 때만 사용 |
| `duration` | `number` _(optional)_ | 유지 시간(ms). 미지정 시 `type` 별 기본값 |
| `position` | `ToastPosition` _(optional)_ | 노출 위치. 기본값 `bottom-right` |
| `dismissible` | `boolean` _(optional)_ | 닫기 버튼 노출 여부 |
| `dedupeKey` | `string` _(optional)_ | 중복 판정 키. 같은 키가 이미 있으면 갱신 |
| `action` | `ToastAction` _(optional)_ | 되돌리기 · 재시도 같은 행동 버튼 |

`message` / `description` 분리 예시

```
message:     "업로드 실패"
description: "파일 크기가 너무 큽니다."
```

---

## 📦 `Toast` — store 가 보관하는 엔티티

```ts
interface Toast {
    id: string;
    type: ToastType;
    message: string;
    description?: string;
    duration: number;
    position: ToastPosition;
    dismissible: boolean;
    dedupeKey?: string;
    action?: ToastAction;
    createdAt: number;
    shownAt?: number;
    status: ToastStatus;
    repeatCount: number;
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | `string` | 식별자. 생성 시 반드시 확정된다 |
| `duration` | `number` | 기본값이 채워진 확정 값 |
| `position` | `ToastPosition` | 기본값이 채워진 확정 값 (`bottom-right`) |
| `dismissible` | `boolean` | 기본값이 채워진 확정 값 |
| `createdAt` | `number` | 생성 시각. 큐 정렬 · 관측 지표에 사용 |
| `shownAt` | `number` _(optional)_ | 실제 노출 시각. 노출 시간 측정에 사용 |
| `status` | `ToastStatus` | 화면에 떠 있는지 큐에서 대기 중인지 |
| `repeatCount` | `number` | 중복 발생 횟수. 기본 `1` |

**입력 타입과 엔티티를 분리하는 이유**
호출자에게는 `id` 를 강제하지 않아야 하지만, store 안에서는 `id` 없는 토스트가 존재할 수 없다.
`id?: string` 하나로 두 상태를 겸하면 내부 코드 전체에 불필요한 옵셔널 체크가 퍼진다.

---

## ⚙️ 종류별 기본값

| `type` | `duration` | `dismissible` | `role` / `aria-live` | 자동 닫힘 |
|--------|-----------|:-------------:|----------------------|:---------:|
| `success` | `3000` | `false` | `status` / `polite` | ✅ |
| `info` | `3000` | `false` | `status` / `polite` | ✅ |
| `warning` | `5000` | `true` | `alert` / `assertive` | ✅ |
| `error` | `Infinity` | `true` | `alert` / `assertive` | ❌ |
| `loading` | `Infinity` | `true` | `status` / `polite` | ❌ (결과로 전환) |

---

## 🔁 중복 처리

같은 `dedupeKey` 를 가진 토스트가 이미 있으면 새로 만들지 않고 기존 토스트를 갱신한다.

```ts
// 네트워크 오류가 10번 발생한 경우
{ id: "t_1", type: "error", message: "네트워크 오류가 반복되고 있습니다", repeatCount: 10 }
```

토스트 10개가 쌓이는 대신 하나가 갱신된다.
이 정책은 접근성과도 직결된다 — `assertive` live region 이 10번 연속 발화하면 스크린 리더 사용자에게는 사용 불가 수준의 방해가 된다.
