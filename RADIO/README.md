# 🍞 Toast Design

토스트 UI 컴포넌트의 설계 연습

## RADIO

**[RADIO](https://github.com/wooiljeong/RADIO)** 는 UI 컴포넌트 설계를 위한 경량 문서 포맷입니다.

- **R**equirements — 기능/비기능 요구사항
- **A**rchitecture — 디렉토리 구조, 유즈케이스, 설계 원칙
- **D**ata model — 타입 인터페이스, 필드 정의
- **I**nterface — 공개 API, 사용 패턴
- **O**ptimization — 렌더링 최적화 전략

## 구조

```
RADIO/
├── Requirements.md   # 무엇을 만들지
├── Architecture.md   # 어떻게 구성할지
├── DataModel.md      # 어떤 데이터를 다룰지
├── Interface.md      # 어떻게 사용할지
└── Optimization.md   # 어떻게 최적화할지
```

## 📌 핵심 결정 요약

| 항목 | 결정 |
|------|------|
| 동시 노출 | 최대 3개, 대기 큐 20개 |
| 유지 시간 | `success`·`info` 3초 / `warning` 5초 / `error`·`loading` 자동 닫힘 없음 |
| 렌더링 | Portal 로 `document.body` 하위 (stacking context 격리) |
| 배치 | flex column + gap (오프셋 계산으로 인한 forced reflow 회피) |
| 애니메이션 | `transform` + `opacity` 만, `will-change` 는 구간 한정 |
| 중복 | `dedupeKey` 로 억제하고 `repeatCount` 갱신 |
| 접근성 | live region 사전 마운트, `polite` / `assertive` 구분, hover·focus 시 타이머 정지 |
| 호출 책임 | mutation 은 ✅ / 전역 interceptor 는 전역 문제만 / `QueryCache.onError` 는 ❌ |

## 🔄 개정 이력

### 2차 — 피드백 반영

| 영역 | 변경 |
|------|------|
| Requirements | "무엇을 결정할 것인가"(질문) → **결정된 정책과 근거**(표)로 전환 |
| Architecture | 토스트로 처리할 것 / 부족한 것 분류 추가, Portal 렌더링 명시, 레이어별 React 의존성 정리 |
| Data Model | `meaning` → `type` 리네임, `CreateToastInput` / `Toast` 분리, `description` · `createdAt` · `dedupeKey` · `action` · `repeatCount` · `status` 추가, 종류별 기본값 표 |
| Interface | `ToastAPI` 정의 (`open` 이 `id` 반환 + `update`), 호출 책임 정책 표, 큐 · 우선순위 · 중복 억제 로직, `QueryCache.onError` 무차별 호출 제거 |
| Optimization | stacking context 설명, `absolute` 오프셋 계산 → flex 배치, `will-change` 한시 적용, live region 사전 마운트, `prefers-reduced-motion`, 타이머 pause, Observability 이벤트 |
