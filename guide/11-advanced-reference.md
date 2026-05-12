# 11. 심화 참고 문서 안내

이 문서는 `docs/guide/` 바깥에 있는 설계 문서, 운영 메모, 회귀 검토 문서를 어떤 상황에서 보면 좋은지 빠르게 연결하기 위한 안내서다.

처음 쓰는 사용자는 먼저 `00` 부터 `10` 까지의 사용자 설명서를 보고, 아래 문서는 필요할 때만 추가로 읽으면 된다.

## 언제 이 문서를 볼까

- WebUI 에 왜 이런 구조가 됐는지 배경이 궁금할 때
- reasoning, status rail, action result 같은 최신 UI 동작의 설계 의도를 보고 싶을 때
- calendar prompt, approval, dashboard reliability 를 더 깊게 추적하고 싶을 때
- 최근 며칠간 실제로 어떤 작업이 반영됐는지 운영 메모를 보고 싶을 때
- upstream sync 나 merge 리스크를 이해하고 싶을 때

## 추천 문서 묶음

### 1. Observable reasoning 설계

- [archive/observable-reasoning-plan/README.md](../archive/observable-reasoning-plan/README.md)

이 문서 묶음은 아래 주제를 다룬다.

- `reasoning_visibility = off | status_only | summary | debug_trace`
- WebUI 에서 reasoning 을 어느 수준까지 보여줄지
- `telegram_v2` 와 Mini App 방향
- runtime, session, channel, WebUI 가 reasoning contract 를 어떻게 공유할지

최근 WebUI 에서 `생각`, `생각 중` 줄이 어떻게 보이는지 이해하려면 여기부터 보는 편이 좋다.

### 2. Calendar / WebUI reliability 재검토

- [archive/calendar-webui-reliability-review/README.md](../archive/calendar-webui-reliability-review/README.md)

이 문서 묶음은 아래 문제를 더 깊게 다룬다.

- prompt 가 사라지는 문제
- dashboard / thread / prompt 간 상태 불일치
- conflict review, approval, cancel, deny cleanup
- Playwright 로 실제 live WebUI 를 검증하는 절차

calendar 자동화가 UI 에서 왜 이런 식으로 보이는지, 왜 metadata cleanup 이 중요한지 보려면 이 문서가 가장 직접적이다.

### 3. WebUI UX 재설계 배경

- [archive/webui-ux-redesign/README.md](../archive/webui-ux-redesign/README.md)

이 문서 묶음은 아래 같은 판단을 설명한다.

- 왜 대화가 여전히 중심이어야 하는지
- 왜 cross-thread aggregate 를 대시보드로 옮기는지
- 왜 action result 를 compact row + detail disclosure 로 줄였는지
- 왜 세부 정보를 필요할 때만 펼치는 방향을 택했는지

현재 user-guide 의 설명이 어떤 UX 판단 위에 서 있는지 알고 싶을 때 적합하다.

### 4. 실행 백로그와 phase 계획

- [planning/execution-backlog/README.md](../planning/execution-backlog/README.md)

이 문서 묶음은 아래를 본다.

- 어떤 기능을 먼저 구현할지
- owner-aware control surface 같은 phase-1 workplan
- product layer 우선순위

현재 사용자 기능이 어떤 큰 흐름 위에서 추가됐는지 보는 데 유용하다.

### 5. 날짜형 운영 메모

- [operations/status-summary/nanobot-status-summary-2026-05-09.md](../operations/status-summary/nanobot-status-summary-2026-05-09.md)
- [operations/status-summary/nanobot-status-summary-2026-05-10.md](../operations/status-summary/nanobot-status-summary-2026-05-10.md)

이 문서들은 특정 날짜에 실제로 무엇을 수정했고, 무엇을 live 로 검증했고, 어떤 리스크가 남았는지를 기록한다.

예를 들어 아래를 확인할 수 있다.

- upstream/main sync 후 live bootstrap 보정
- observable reasoning 과 reasoning visibility 작업
- action result compact row 와 세부 정보 토글
- WebUI i18n / 용어 기준 정리

### 6. upstream sync / merge 배경

- [archive/upstream-main-sync-2026-05-08/main-sync-summary.md](../archive/upstream-main-sync-2026-05-08/main-sync-summary.md)

이 문서는 upstream/main 유입 범위와 local fork 충돌 지점을 요약한다.

아래 상황에서 특히 유용하다.

- 왜 최근 WebUI surface 가 많이 바뀌었는지 확인할 때
- merge hotspot 이 어디인지 보고 싶을 때
- bootstrap/auth, ask_user, image generation 같은 upstream 유입 기능을 한 번에 훑고 싶을 때

## 빠른 읽기 순서

### 현재 WebUI 동작 배경이 궁금할 때

1. [01. 화면 구성과 WebUI 사용법](01-webui-overview.md)
2. [06. 상태, 작업, 메모리 관리](06-status-task-memory.md)
3. [10. WebUI 용어 기준](10-webui-terminology.md)
4. [archive/webui-ux-redesign/README.md](../archive/webui-ux-redesign/README.md)

### reasoning / thinking row 가 궁금할 때

1. [06. 상태, 작업, 메모리 관리](06-status-task-memory.md)
2. [archive/observable-reasoning-plan/README.md](../archive/observable-reasoning-plan/README.md)
3. [operations/status-summary/nanobot-status-summary-2026-05-10.md](../operations/status-summary/nanobot-status-summary-2026-05-10.md)

### calendar prompt / approval / dashboard reliability 가 궁금할 때

1. [04. 일정 자동화와 승인 흐름](04-calendar-automation.md)
2. [09. 운영 확인과 문제 해결](09-troubleshooting.md)
3. [archive/calendar-webui-reliability-review/README.md](../archive/calendar-webui-reliability-review/README.md)

## 참고 메모

- `guide` 는 현재 사용자와 운영자 관점의 설명 문서다.
- 나머지 디렉터리 문서는 설계 배경, 운영 메모, 회귀 검증용 기록을 포함한다.
- dated 문서에는 예전 WebUI 표현이 남아 있을 수 있으므로, 현재 화면 용어는 [10. WebUI 용어 기준](10-webui-terminology.md) 을 우선 기준으로 본다.