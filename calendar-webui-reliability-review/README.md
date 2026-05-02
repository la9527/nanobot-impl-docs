# Calendar and WebUI Reliability Review - 2026-05-02

## 목적

이 디렉토리는 Nanobot WebUI 의 dashboard, thread, action prompt, Google Calendar 자동화 흐름을 다시 점검하고, 기능을 안정적으로 재적용하기 위한 단계별 작업 문서를 모아둔다.

이번 정리는 단순 UI polish 가 아니라 아래 문제를 함께 다룬다.

1. Google Calendar 요청이 자연어 대화에서 실제 캘린더 automation 으로 연결되지 않는 문제
2. conflict review, approval, input prompt 가 나타났다가 사라지는 문제
3. session metadata 와 message history 가 서로 다른 상태를 보여주는 문제
4. dashboard, status rail, inline result, composer 가 같은 상태를 중복 또는 불완전하게 보여주는 문제
5. Playwright 로 실제 화면을 확인하면서 기능별 회귀 검증 절차를 고정해야 하는 문제

## 문서 구성

1. [01-current-audit.md](./01-current-audit.md)
   - Playwright 와 API 응답으로 확인한 현재 문제 상태
   - thread, dashboard, prompt, calendar flow 의 실제 관찰 결과

2. [02-feature-verification-matrix.md](./02-feature-verification-matrix.md)
   - 기능별 기대 동작, 현재 상태, 필요한 검증 방법
   - calendar, prompt, dashboard, linked session, Telegram fallback 을 분리

3. [03-calendar-state-machine.md](./03-calendar-state-machine.md)
   - calendar create/check/delete/reschedule 을 단일 상태 모델로 재정리
   - pending prompt 의 canonical source 를 message tail 이 아니라 metadata 로 이동하는 방안

4. [04-playwright-regression-plan.md](./04-playwright-regression-plan.md)
   - live WebUI 에서 반복 확인해야 할 Playwright 시나리오
   - dashboard, thread, prompt, conflict, approval, refresh 검증 절차

5. [05-implementation-phases.md](./05-implementation-phases.md)
   - 실제 재적용 작업 순서
   - backend state cleanup, WebUI prompt source, natural language routing, delete/reschedule 기능, 검증 순서

6. [coding-work-plan/](./coding-work-plan/00-overview.md)
   - 실제 코딩 작업을 파일, 모듈, 테스트, live 검증 단위로 세분화한 실행 계획
   - backend pending interaction, WebUI prompt source, natural language routing, delete/reschedule, dashboard reliability 를 단계별로 분리

## 현재 결론

가장 먼저 고쳐야 할 부분은 UI 카드 높이가 아니라 `pending interaction` 의 source of truth 이다.

현재 WebUI 는 버튼이 있는 마지막 assistant message 를 찾아 prompt 를 보여준다. 그러나 calendar conflict, approval, proactive summary, action result 가 섞이면 버튼 메시지가 history 뒤쪽으로 밀리거나 사라지고, metadata 에는 여전히 pending 상태가 남는다. 그 결과 사용자는 상태 카드는 보지만 선택 박스는 보지 못한다.

따라서 다음 구현은 아래 순서로 진행한다.

1. calendar pending interaction metadata 를 명시적으로 만든다.
2. WebUI prompt 는 message history 보다 metadata 를 우선한다.
3. calendar conflict, input, approval, cancel, deny, approve 의 metadata cleanup 을 하나의 helper 로 통합한다.
4. 자연어 calendar 요청을 slash command 또는 calendar intent 로 연결한다.
5. delete/reschedule 기능은 별도 approval flow 로 추가한다.
6. Playwright 로 dashboard 와 thread 를 모두 통과시킨 뒤 live 반영한다.