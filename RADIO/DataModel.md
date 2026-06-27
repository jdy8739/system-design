# 🗂️ Data Model

> Toast 컴포넌트의 타입 정의

---

## `Toast` interface

```ts
interface Toast {
    message: string;
    duration?: number;
    position?: "right-top" | "right-bottom";
    meaning?: "success" | "error" | "warning" | "info" | "loading";
};
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `message` | `string` | 토스트의 메시지 |
| `duration` | `number` _(optional)_ | 토스트의 유효시간 |
| `position` | `"right-top"` \| `"right-bottom"` _(optional)_ | 토스트의 위치 |
| `meaning` | `"success"` \| `"error"` \| `"warning"` \| `"info"` \| `"loading"` _(optional)_ | 토스트의 종류 |
