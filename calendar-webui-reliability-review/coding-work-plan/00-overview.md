# Coding Work Plan Overview

## 목적

이 디렉토리는 calendar/WebUI reliability review 를 실제 코딩 작업으로 옮기기 위한 세부 실행 계획이다.

상위 문서는 문제 진단과 방향을 정리한다. 이 디렉토리는 아래 질문에 답한다.

1. 어떤 파일을 수정할 것인가
2. 어떤 순서로 구현할 것인가
3. 각 단계에서 어떤 테스트를 추가할 것인가
4. live WebUI 에서 무엇을 확인할 것인가
5. 어느 시점에 다음 단계로 넘어갈 수 있는가

## 작업 원칙

1. backend state 를 먼저 고정한다.
2. WebUI 는 message history 가 아니라 metadata state 를 우선 신뢰한다.
3. calendar 자연어 라우팅은 delete/reschedule 보다 먼저 추가하되, 위험한 작업은 approval 로 막는다.
4. delete/reschedule 은 Google Calendar 실행 기능이므로 별도 승인 flow 로 둔다.
5. dashboard 는 기능 신뢰성 확보 후 actionable overview 로 정리한다.
6. 각 단계는 focused test 와 Playwright live check 를 통과해야 다음 단계로 넘어간다.

## 문서 구성

1. [01-backend-pending-interaction.md](./01-backend-pending-interaction.md)
   - calendar pending interaction metadata 와 cleanup helper 구현 계획

2. [02-webui-prompt-source.md](./02-webui-prompt-source.md)
   - WebUI prompt derivation 을 metadata 우선으로 변경하는 계획

3. [03-calendar-flow-cleanup.md](./03-calendar-flow-cleanup.md)
   - create/check/conflict/approval/cancel/deny 흐름의 stale state 제거 계획

4. [04-natural-language-routing.md](./04-natural-language-routing.md)
   - 자연어 calendar 요청을 automation 으로 연결하는 rule-based bridge 계획

5. [05-delete-reschedule-automation.md](./05-delete-reschedule-automation.md)
   - delete/update/reschedule webhook 과 approval flow 구현 계획

6. [06-dashboard-reliability.md](./06-dashboard-reliability.md)
   - dashboard navigation, actionable queue, accessibility regression 계획

7. [07-test-and-live-validation.md](./07-test-and-live-validation.md)
   - focused tests, build, launchd restart, Playwright 시나리오 실행 계획

## 권장 구현 순서

1. Backend pending interaction model
2. WebUI metadata-driven prompt rendering
3. Calendar flow cleanup
4. Natural language routing
5. Delete/reschedule automation
6. Dashboard reliability
7. Full live validation

## Definition of Done

전체 작업 완료 기준은 아래와 같다.

1. calendar conflict prompt 는 refresh 후에도 사라지지 않는다.
2. force create, reschedule, cancel, approve, deny 후 stale metadata 가 남지 않는다.
3. 자연어 `오늘 일정 보여줘` 가 일반 LLM 답변이 아니라 calendar automation 으로 연결된다.
4. 기존 event 삭제는 candidate selection 과 approval 을 거친다.
5. dashboard 는 Playwright role selector 로 안정적으로 열린다.
6. Telegram fallback 과 WebUI linked session prompt 가 같은 pending interaction 을 바라본다.
7. focused Python/WebUI tests 와 live Playwright check 가 통과한다.