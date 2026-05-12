# Implementation Phases

## 원칙

이번 작업은 UI surface 를 먼저 고치는 순서가 아니다. 먼저 backend state 를 안정화하고, 그 다음 WebUI 가 그 상태를 신뢰하게 만들어야 한다.

## Phase 0. Baseline capture

목표: 현재 문제를 고정된 재현 조건으로 남긴다.

작업:

1. 현재 live session API payload 저장
2. current thread screenshot 저장
3. console 401 반복 여부 기록
4. dashboard role click 실패 여부 기록
5. calendar natural language 요청이 LLM 답변으로 빠지는 예시 기록

완료 기준:

- `01-current-audit.md` 가 최신 상태와 맞음
- 재현 가능한 입력 문장과 session key 가 기록됨

## Phase 1. Calendar pending interaction model

목표: prompt 선택 박스가 message history 에 의존하지 않게 한다.

Backend 작업:

1. `calendar_pending_interaction` metadata schema 추가
2. conflict review, missing input, approval 상태를 이 schema 로 저장
3. legacy metadata 와 동기화 또는 migration helper 작성
4. cancel/deny/approve/reschedule/force-create cleanup helper 추가

WebUI 작업:

1. `sessionMetadata.ts` 에 pending interaction parser 추가
2. `ThreadShell` 의 `pendingAsk` 를 metadata 우선으로 변경
3. message tail buttons 는 fallback 으로만 사용
4. prompt click 은 기존 websocket/session_message path 를 유지

테스트:

- backend calendar command tests
- WebUI ThreadShell metadata prompt tests
- useSessions hydration tests

## Phase 2. Calendar flow cleanup

목표: create/check/approval 흐름의 stale state 를 제거한다.

작업:

1. conflict 발생 시 action_result 와 pending interaction 을 같은 transaction 처럼 저장
2. force create 시 conflict metadata 제거 후 approval metadata 만 유지
3. reschedule 시 old action_result 를 stale 로 남기지 않도록 input pending 으로 전환
4. cancel 시 pending interaction 과 approval summary 를 모두 제거하고 cancelled result 로 교체
5. deny without pending 이 approve 로 fallback 하는 현재 동작 제거

테스트:

- conflict -> cancel
- conflict -> reschedule -> new conflict/no conflict
- conflict -> force create -> approve
- approve/deny after refresh

## Phase 3. Natural language calendar routing

목표: 사용자가 slash command 를 몰라도 calendar 기능을 사용할 수 있게 한다.

작업:

1. rule-based calendar intent bridge 추가
2. list intent: `일정 보여줘`, `캘린더 확인`, `오늘 일정`
3. create intent: `일정 잡아줘`, `캘린더에 추가`
4. delete intent: `이 일정 지워줘`, `삭제해줘`
5. ambiguous input 은 바로 LLM 답변하지 않고 clarification prompt 로 전환

주의:

- 삭제나 생성은 항상 approval 을 거친다.
- 자연어 parsing 이 불완전하면 structured input prompt 로 보낸다.

테스트:

- 자연어 list 가 calendar automation 을 호출
- 자연어 create 가 missing input prompt 로 연결
- 자연어 delete 가 candidate selection 으로 연결

## Phase 4. Delete and reschedule automation

목표: conflict 를 해결할 수 있는 실제 선택지를 제공한다.

Backend 작업:

1. n8n delete webhook config 추가
2. `N8NCalendarAutomationClient.delete_event` 추가
3. `CalendarAutomationSessionRunner.request_delete_approval` 추가
4. `approve_delete`, `deny_delete` 추가
5. 필요 시 update/reschedule webhook 추가 또는 delete+create 정책 문서화

WebUI 작업:

1. conflict detail 에 conflicting event selection 추가
2. delete approval prompt 표시
3. delete 완료 후 action_result 표시

테스트:

- list -> delete approval -> approve -> list 재확인
- conflict -> existing event delete review -> approve delete

## Phase 5. Dashboard reliability and density

목표: dashboard 가 실제 운영 surface 로 안정적으로 동작하게 한다.

작업:

1. Dashboard button 의 accessible name 과 role 안정화
2. pending approval/block row 를 clickable 하게 만들고 details/prompt 로 연결
3. dashboard 에 composer 가 섞이지 않도록 유지
4. visual density simplification plan 과 연결해 briefing stack 으로 축소
5. dashboard Playwright scenario 를 regression 으로 추가

테스트:

- `getByRole('button', { name: /Dashboard/ })` 성공
- pending item 클릭 시 해당 thread/prompt 로 이동
- dashboard screenshot desktop/mobile 비교

## Phase 6. Live validation and restart

목표: source 수정, bundle build, launchd live runtime 을 일치시킨다.

절차:

1. Python focused tests 실행
2. WebUI focused tests 실행
3. `npm run build`
4. launchd services 재기동
5. gateway health 확인
6. bootstrap 확인
7. Playwright scenario A-H 실행
8. 결과를 status summary 또는 review docs 에 반영

## 작업 순서 요약

1. Phase 0 baseline capture
2. Phase 1 pending interaction model
3. Phase 2 calendar state cleanup
4. Phase 3 natural language routing
5. Phase 4 delete/reschedule automation
6. Phase 5 dashboard reliability
7. Phase 6 live validation

이 순서를 지키면 prompt disappearance 와 stale metadata 문제를 먼저 막고, 그 위에 자연어 라우팅과 delete/reschedule 기능을 안전하게 얹을 수 있다.