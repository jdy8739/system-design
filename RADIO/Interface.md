## libs

```js
const TOAST_QUEUE = [];

const pushToToastQueue = (toastElement, { duration }) => {
    if (toastElement.type === Toast) {
        TOAST_QUEUE.push(toastElement);

        setTimeout(() => {
            const idx = TOAST_QUEUE.indexOf(toastElement);
            if (idx !== -1) TOAST_QUEUE.splice(idx, 1);
        }, duration);
    }
}

const openToast = ({ duration = 3000, ...props }) => pushToToastQueue(<Toast {...props} />, { duration })
```

토스트 기능은 완전 다른 도메인과 분리하여 라이브러리 형식의 feature로 모듈화 및 다른 도메인과 격리시킨다.

유저는 토스트를 사용하는 소비처의 도메인 로직에서 명시적으로 `<Toast message={"session expired!"} />` 처럼 정적으로 토스트를 달 수도 있고,
axios의 interceptor나, tanstack-query의 cache의 생명주기 훅에 Toast를 호출하는 함수를 주입해서 동적으로 토스트를 띄울 수 있다.

```js
import toast from "@toast";

axios.beforeRequestError = (error) => openToast({ message: error.message, duration: 2000, exposeCloseButton: true, position: "top" });
```
