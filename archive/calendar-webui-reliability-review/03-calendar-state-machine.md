# Calendar State Machine

## 목표

Calendar 기능은 message history 에 흩어진 버튼과 여러 metadata key 에 의존하지 않고, 단일 pending interaction 상태를 기준으로 동작해야 한다.

## Canonical metadata

새 metadata key 는 아래처럼 둔다.

```json
{
  "calendar_pending_interaction": {
    "id": "calendar-interaction-...",
    "kind": "collect_input | conflict_review | create_approval | delete_approval | reschedule_approval",
    "status": "pending",
    "question": "...",
    "buttons": [["...", "..."], ["취소"]],
    "request": {},
    "conflicts": [],
    "created_at": "...",
    "updated_at": "..."
  }
}
```

WebUI 는 이 metadata 를 우선 사용해 prompt 를 렌더링한다. persisted assistant message 의 `buttons` 는 backward compatibility fallback 으로만 사용한다.

## 상태 정의

### idle

진행 중인 calendar interaction 이 없는 상태다.

허용 action:

- list
- create request
- check conflict
- delete request

### collecting_input

create 또는 delete/reschedule 에 필요한 필드가 부족한 상태다.

필수 데이터:

- expected_field
- partial request
- cancel button

종료 조건:

- 모든 필수 필드 수집 -> checking_conflict 또는 delete candidate selection
- cancel -> cancelled result 로 종료

### checking_conflict

create request 의 시간 충돌을 확인하는 상태다. UI 에는 긴 prompt 를 띄우기보다 running indicator 만 노출한다.

종료 조건:

- no conflict -> create_approval
- conflict -> conflict_review
- unavailable -> failed 또는 blocked

### conflict_review

요청 시간이 기존 일정과 겹친 상태다.

기본 버튼:

- `그래도 생성 승인 요청`
- `새 시간 다시 입력`
- `기존 일정 삭제 검토`
- `취소`

주의:

- `기존 일정 삭제 검토` 는 delete 기능이 구현된 뒤에만 노출한다.
- 버튼이 message history 에 없어도 metadata 로 WebUI prompt 가 유지되어야 한다.

### create_approval

이벤트 생성 직전 사용자 승인을 기다리는 상태다.

버튼:

- `승인`
- `취소`

현재 slash command 의 `/calendar approve`, `/calendar deny` 는 이 상태의 text fallback 으로 유지한다.

### delete_approval

기존 Google Calendar 이벤트 삭제 전 사용자 승인을 기다리는 상태다.

필수 데이터:

- event_id
- title
- start_at
- end_at
- calendar_id 또는 provider reference

버튼:

- `삭제 승인`
- `취소`

### completed / cancelled / failed

interaction 이 종료된 상태다.

종료 시 반드시 정리할 metadata:

- `calendar_pending_interaction`
- legacy `calendar_create_input`
- legacy `calendar_conflict_review`
- stale `approval_summary`
- stale `calendar_create_approval`

단, 최신 `action_result` 는 completed/cancelled/failed 상태로 교체한다.

## Cleanup helper

backend 에 아래 helper 계층을 둔다.

1. `set_calendar_pending_interaction(session, interaction)`
2. `clear_calendar_pending_interaction(session)`
3. `finish_calendar_interaction(session, result)`
4. `clear_calendar_legacy_pending_state(session)`

모든 cancel, approve, deny, force-create, reschedule 경로는 이 helper 를 거쳐야 한다.

## WebUI derivation

WebUI 의 prompt 우선순위는 아래와 같다.

1. `session.metadata.calendar_pending_interaction`
2. `session.metadata.approval_summary` 중 calendar approval
3. 최신 assistant message 의 `buttons`
4. 없음

이렇게 하면 refresh, polling, proactive message, assistant follow-up 이 있어도 prompt 가 사라지지 않는다.