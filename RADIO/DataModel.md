# 🗂️ Data Model

> Toast 컴포넌트의 타입 정의

---

## `Toast` interface

```ts
interface Toast {
    id?: string;
    message: string;
    duration?: number;
    position?: "top-right" | "bottom-right";
    meaning?: "success" | "error" | "warning" | "info" | "loading";
    exposeCloseButton?: boolean;
};
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | `string` _(optional)_ | 토스트 식별자. 미지정 시 자동 생성 |
| `message` | `string` | 토스트의 메시지 |
| `duration` | `number` _(optional)_ | 토스트의 유효시간 |
| `position` | `"top-right"` \| `"bottom-right"` _(optional)_ | 토스트의 위치 |
| `meaning` | `"success"` \| `"error"` \| `"warning"` \| `"info"` \| `"loading"` _(optional)_ | 토스트의 종류 |
| `exposeCloseButton` | `boolean` _(optional)_ | 사용자 닫기 버튼 노출 여부 |
