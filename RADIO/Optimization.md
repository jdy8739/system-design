## 위치 계산

처음 위치계산은 document의 자식으로 window의 full width, height를 갖는 toast container를 relative로 하여,
토스트의 속성을 absolute, top: -30px (position이 right-top일 경우) 로 초기값 처리.

## 렌더링 최적화

- transition translate 속성 사용해서 reflow 최소화, repaint 단계까지만 사용
- setTimeout 이후 translate과 함께 opacity 사용하여 composition 단계까지 재계산
- 그리고 z-index로 겹치는 토스트 관리

따라서 총 reflow 작업이 없도록 CPU 연산 최소화 가능.
