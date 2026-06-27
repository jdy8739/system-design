```ts
interface Toast {
    message: string;
    duration?: number;
    position?: "right-top" | "right-bottom";
    meaning?: "success" | "error" | "warning" | "info" | "loading";
};
```

- 토스트의 메시지
- 토스트의 유효시간
- 토스트의 위치
- 토스트의 종류
