# Feature Verification Matrix

## 목적

이 문서는 Nanobot WebUI 와 Google Calendar 자동화 기능을 하나씩 다시 검증하기 위한 matrix 이다. 구현 전후에 이 표를 기준으로 기능 누락과 regression 을 확인한다.

## Calendar automation

| 기능 | 기대 동작 | 현재 관찰 | 필요한 조치 | 검증 |
| --- | --- | --- | --- | --- |
| calendar list | 오늘 또는 지정 기간 일정을 structured events 로 반환 | slash command 경로는 존재. 자연어 요청은 일반 LLM 답변으로 빠짐 | 자연어 intent bridge 추가 | `/calendar list`, 자연어 `오늘 일정 보여줘` 모두 확인 |
| conflict check | 지정 시간과 겹치는 이벤트를 찾고 result metadata 저장 | conflict result 는 저장됨 | result 와 pending interaction 동기화 | conflict 발생 시 prompt 와 result 동시 표시 |
| create missing input | title/start/end 누락 시 입력 prompt 표시 | backend prompt 존재 | refresh 후에도 prompt 유지 필요 | 누락 입력 -> reload -> prompt 유지 |
| conflict review | 겹침 발생 시 force create/reschedule/cancel 선택 | 버튼 메시지는 생성되나 사라질 수 있음 | metadata 기반 pending interaction 필요 | prompt 선택 박스 persistence 확인 |
| force create | conflict 를 감수하고 approval 단계로 전환 | backend 경로 존재 | conflict metadata cleanup 검증 필요 | 선택 후 approval prompt 로 전환 |
| reschedule | 새 start/end 입력으로 재검사 | backend 입력 prompt 존재 | 이전 conflict/action_result 정리 필요 | 새 시간 입력 후 conflict 재검사 |
| approve create | pending create 를 실제 Google Calendar 에 생성 | backend 경로 존재 | stale approval cleanup 필요 | approve 후 pending metadata 제거 |
| deny create | pending create 를 취소 | backend 경로 존재 | no pending 시 approve 로 fallback 하는 현재 동작 수정 필요 | deny 후 cancelled result 와 metadata 제거 |
| delete event | 기존 이벤트 삭제 | 현재 없음 | n8n delete webhook, backend runner, approval flow 추가 | list -> select -> confirm -> delete |
| reschedule event | 기존 이벤트 시간 변경 | 현재 없음 | update webhook 또는 delete+create 정책 결정 | conflict 해결 flow 에서 선택 가능 |

## WebUI prompt and thread

| 기능 | 기대 동작 | 현재 관찰 | 필요한 조치 | 검증 |
| --- | --- | --- | --- | --- |
| prompt display | pending interaction 이 있으면 항상 prompt 노출 | message tail 에 buttons 없으면 사라짐 | metadata 우선 prompt derivation | reload/poll/proactive 후 유지 |
| prompt click | 버튼 클릭이 올바른 session 으로 전달 | linked session 에서 별도 send path 필요 | existing session_message path 유지 및 테스트 확대 | WebUI/TG linked 모두 클릭 검증 |
| direct text reply | 사용자가 직접 입력해도 pending interaction 처리 | 일부 흐름에서 prompt 가 잠깐 나타났다 사라짐 | backend interceptor 와 UI 상태 동기화 | direct text reply 후 다음 prompt 확인 |
| inline result | 최근 action result 를 compact 표시 | compact card 는 있음 | pending prompt 와 중복/경쟁하지 않게 우선순위 조정 | result + prompt 동시 표시 확인 |
| status rail | owner-wide pending/blocked 요약 | pending count 는 보이나 action 불가 | rail item 클릭 시 details/prompt 위치 연결 | pending row click 검증 |
| composer disabled | 처리 중이 아닐 때 입력 가능 | disabled 순간 관찰됨 | remoteReplyPending/token error 원인 확인 | idle 상태 입력/전송 가능 확인 |

## Dashboard

| 기능 | 기대 동작 | 현재 관찰 | 필요한 조치 | 검증 |
| --- | --- | --- | --- | --- |
| dashboard navigation | sidebar `Dashboard` 로 항상 진입 | Playwright role click timeout | accessible name/click target 정리 | `getByRole` click 성공 |
| overview priority | pending approvals, blocked tasks 를 actionable 하게 표시 | summary 는 보이나 해결 surface 연결 약함 | pending interaction 또는 details sheet 로 연결 | approval item 클릭 시 prompt/details 표시 |
| no composer on dashboard | dashboard 는 overview 전용 | 이전 redesign 기준은 제거 방향 | regression 유지 | dashboard 화면에 composer 없음 |
| compact density | briefing stack 형태 | 일부 card board 느낌 남음 | density pass 문서와 병행 | desktop/mobile screenshot 비교 |

## Telegram and linked sessions

| 기능 | 기대 동작 | 현재 관찰 | 필요한 조치 | 검증 |
| --- | --- | --- | --- | --- |
| button fallback | native keyboard off 일 때 `[Label]` 텍스트 fallback | normalization 추가됨 | pending interaction 과 동일 semantics 유지 | `[취소]`, `[새 시간 다시 입력]` 처리 |
| WebUI mirror | Telegram prompt buttons 가 WebUI 에도 보임 | mirror buttons 보존 수정됨 | history polling 과 metadata source 통합 | refresh 후 buttons 유지 |
| session_message | linked session reply 는 원 채널 session 으로 전달 | 별도 path 수정됨 | pending interaction id 와 연결 | prompt click 후 Telegram session 처리 확인 |

## Runtime/API stability

| 기능 | 기대 동작 | 현재 관찰 | 필요한 조치 | 검증 |
| --- | --- | --- | --- | --- |
| bootstrap token | API polling 이 token 만료를 복구 | console 401 반복 관찰 | reauth/polling failure path 점검 | 10분 이상 idle 후 refresh |
| session key consistency | 화면 title 과 active session key 일치 | 표시 title 과 API 선택 key 불일치 의심 | active session selection/debug info 확인 | selected session key API 대조 |
| live bundle | dist asset 과 browser loaded asset 일치 | 최신 asset 로드 확인됨 | build 후 hash 확인 절차 유지 | script hash 비교 |