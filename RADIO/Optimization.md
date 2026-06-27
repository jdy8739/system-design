# ⚡ Optimization

> 토스트 애니메이션의 렌더링 최적화 전략

---

## 📐 위치 계산

처음 위치계산은 document의 자식으로 window의 full width, height를 갖는 toast container를 relative로 하여,
토스트의 속성을 absolute, 초기값 `transform: translateY(-100%)` (position이 top-right일 경우) 로 처리.
이렇게 하면 토스트 높이와 무관하게 항상 올바른 오프셋을 얻는다.

---

## 🎬 렌더링 최적화

| 단계 | 전략 | 효과 |
|------|------|------|
| **Reflow** | `will-change: transform, opacity` + `transform: translate` 로 reflow 최소화, repaint 단계까지만 사용 | Layout 재계산 방지 |
| **Composite** | setTimeout 이후 `transform: translate`과 `opacity` 함께 transition → composition 단계까지 재계산 | GPU 가속, Paint 생략 |
| **Stacking** | 그리고 z-index로 겹치는 토스트 관리 | 토스트 간 레이어 정렬 |

> 따라서 총 reflow 작업이 없도록 CPU 연산 최소화 가능.
