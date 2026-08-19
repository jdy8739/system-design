# 🔌 Interface

> 토스트 라이브러리의 공개 API 및 사용 패턴

---

## 📤 공개 API

```ts
type ToastAPI = {
    open: (input: CreateToastInput) => string;
    update: (id: string, patch: Partial<CreateToastInput>) => void;
    close: (id: string, reason?: ToastDismissReason) => void;
    clear: () => void;

    success: (message: string, options?: ToastOptions) => string;
    error:   (message: string, options?: ToastOptions) => string;
    warning: (message: string, options?: ToastOptions) => string;
    info:    (message: string, options?: ToastOptions) => string;
    loading: (message: string, options?: ToastOptions) => string;
};

type ToastOptions = Omit<CreateToastInput, "type" | "message">;
```

`open` 이 **`id` 를 반환**하는 것이 핵심이다.
반환된 `id` 로 `update` 를 호출할 수 있어야 `loading → success | error` 전환이 표현된다.

---

## 📦 코어 로직

`toast.open` · `toast.success` 같은 공개 메서드는 아래 `openToast` 를 감싼 얇은 파사드다.
노출 제한은 `top-right` · `bottom-right` 를 합산한 **전역 기준**으로 적용한다 — 위치별로 따로 세면 화면에는 결국 6개가 뜬다.

```js
const MAX_VISIBLE = 3;              // 위치 무관 전역 상한
const MAX_QUEUE = 20;

let toasts = [];                    // visible + queued
const listeners = new Set();

const notify = () => listeners.forEach((fn) => fn(toasts));
const countBy = (status) => toasts.filter((t) => t.status === status).length;

// error/warning 은 대기열에서 앞으로 당긴다
const priorityOf = (type) => (type === "error" || type === "warning" ? 1 : 0);

const flush = () => {
    const slots = MAX_VISIBLE - countBy("visible");
    if (slots <= 0) return;

    toasts
        .filter((t) => t.status === "queued")
        .sort((a, b) => priorityOf(b.type) - priorityOf(a.type) || a.createdAt - b.createdAt)
        .slice(0, slots)
        .forEach((t) => {
            t.status = "visible";
            t.shownAt = performance.now();
            startTimer(t);
        });
};

const openToast = (input) => {
    // 1. 중복 억제 — 같은 dedupeKey 가 이미 있으면 갱신만 한다
    const existing = input.dedupeKey && toasts.find((t) => t.dedupeKey === input.dedupeKey);
    if (existing) {
        existing.repeatCount += 1;
        existing.message = input.message;
        restartTimer(existing);
        notify();
        return existing.id;
    }

    // 2. 큐 오버플로 — 우선순위가 낮고 가장 오래된 대기 토스트부터 버린다
    if (countBy("queued") >= MAX_QUEUE) {
        const [victim] = toasts
            .filter((t) => t.status === "queued")
            .sort((a, b) => priorityOf(a.type) - priorityOf(b.type) || a.createdAt - b.createdAt);
        removeToast(victim.id, "programmatic");
    }

    const toast = withDefaults(input);   // id, duration, position, dismissible, createdAt 확정

    // 3. 자리가 있으면 즉시 노출, 없으면 대기
    toast.status = countBy("visible") < MAX_VISIBLE ? "visible" : "queued";
    if (toast.status === "visible") {
        toast.shownAt = performance.now();
        startTimer(toast);
    }

    toasts.push(toast);
    notify();
    return toast.id;
};

const closeToast = (id, reason = "programmatic") => {
    removeToast(id, reason);
    flush();                            // 빈 자리에 대기 토스트를 채운다
    notify();
};
```

타이머는 `duration` 이 `Infinity` 인 `error` · `loading` 에는 걸지 않으며, hover · focus 중에는 정지한다.
타이머의 잔여 시간 같은 런타임 상태는 `Toast` 엔티티가 아니라 별도 레지스트리가 보관한다 — 모델은 렌더링에 필요한 값만 갖는다.
노출 · 닫힘 · 중복 억제 시점의 계측은 [Optimization](./Optimization.md#-observability) 에서 정의한다.

> 실제 렌더링은 store 를 구독하는 `ToastContainer` 가 Portal 로 수행한다.

---

## 🧭 호출 책임 정책

토스트를 **어디서** 띄우는지가 실제로 UX 를 결정한다.

| 발생 지점 | 토스트 | 정책 |
|-----------|:------:|------|
| mutation `onSuccess` / `onError` | ✅ | 사용자가 유발한 액션의 결과. 구체적인 메시지를 띄운다 |
| 전역 axios 인터셉터 | ⚠️ | 세션 만료 · 네트워크 끊김 · 공통 서버 장애처럼 **전역적으로 의미 있는 문제만** |
| `QueryCache.onError` | ❌ | 모든 query 실패에 띄우지 않는다. 백그라운드 refetch 실패까지 알림이 된다 |
| `QueryCache.onSuccess` | ❌ | 사용자가 요청하지도 않은 "데이터 갱신 완료" 는 노이즈다 |
| 페이지 초기 로딩 실패 | ❌ | 화면 자체의 error state 로 처리한다 |
| 폼 유효성 오류 | ❌ | 해당 필드 인라인 메시지로 처리한다 |

> 판단 기준은 **"사용자가 방금 이걸 시켰는가"** 이다.
> 사용자가 시키지 않은 일의 결과를 토스트로 띄우면, 사용자는 알림이 왜 떴는지 이해하지 못한다.

---

## 🧩 사용 패턴

### 액션 결과 — 가장 기본

```js
import { toast } from "@toast";

const handleSave = async () => {
    try {
        await saveProfile();
        toast.success("프로필이 저장되었습니다");
    } catch {
        toast.error("프로필 저장에 실패했습니다", { action: { label: "다시 시도", onClick: handleSave } });
    }
};
```

### loading → success | error 전환

```js
const toastId = toast.loading("파일을 업로드하는 중입니다.");

try {
    await uploadFile();

    toast.update(toastId, {
        type: "success",
        message: "파일 업로드가 완료되었습니다.",
        duration: 3000,
    });
} catch {
    toast.update(toastId, {
        type: "error",
        message: "파일 업로드에 실패했습니다.",
        dismissible: true,
    });
}
```

토스트를 닫았다 새로 여는 것이 아니라 **같은 토스트를 갱신**하기 때문에, 위치가 튀거나 큐 순서가 바뀌지 않는다.

### 전역 인터셉터 — 전역적 문제만

```js
import { toast } from "@toast";

axios.interceptors.response.use(
    (response) => response,
    (error) => {
        const status = error.response?.status;

        if (status === 401) {
            toast.error("세션이 만료되었습니다. 다시 로그인해 주세요.", {
                dedupeKey: "session-expired",
                position: "top-right",
                action: { label: "로그인", onClick: goToLogin },
            });
        }

        if (!error.response) {
            toast.error("네트워크 연결을 확인해 주세요.", {
                dedupeKey: "network-error",   // 반복 발생해도 하나로 유지
                position: "top-right",
            });
        }

        return Promise.reject(error);        // 개별 처리 기회를 남긴다
    }
);
```

`dedupeKey` 가 없으면 요청 20개가 동시에 실패했을 때 토스트 20개가 뜬다.

> 축출 대상을 우선순위와 무관하게 항상 하나 고르는 이유는 `open` 이 **언제나 `id` 를 반환**해야 하기 때문이다.
> 반환값이 `null` 이 될 수 있으면 호출부는 매번 `update` 전에 null 검사를 해야 한다.

### TanStack Query — 전역 훅에서 무차별 호출하지 않는다

```js
const queryClient = new QueryClient({
    queryCache: new QueryCache({
        // ❌ onError 로 모든 query 실패를 띄우지 않는다.
        //    백그라운드 refetch 실패는 사용자가 유발한 것이 아니다.
        //    페이지 로딩 실패는 화면의 error state 가 담당한다.
    }),
    mutationCache: new MutationCache({
        onError: (error, _vars, _ctx, mutation) => {
            // 개별 mutation 이 스스로 처리했다면 전역에서는 침묵한다
            if (mutation.options.onError) return;
            toast.error(toMessage(error));
        },
        // onSuccess 는 두지 않는다. 성공 메시지는 액션마다 문구가 달라야 한다
    }),
});
```

전역에는 **아무도 처리하지 않은 실패의 최후 방어선**만 남기고, 성공 메시지는 각 액션이 직접 띄운다.
