# 01. Backend Pending Interaction

## 목표

calendar prompt 선택 박스가 message history 에 의존하지 않도록 backend 에 canonical pending interaction metadata 를 추가한다.

## 수정 대상 파일

1. `source/nanobot/command/builtin.py`
2. `source/nanobot/automation/calendar.py`
3. `source/nanobot/automation_results.py`
4. `source/tests/command/test_builtin_calendar.py`
5. 필요 시 `source/tests/test_automation_results.py`

## 새 metadata key

```python
CALENDAR_PENDING_INTERACTION_METADATA_KEY = "calendar_pending_interaction"
```

예상 payload:

```json
{
  "id": "calendar-interaction-abc123",
  "kind": "collect_input",
  "status": "pending",
  "question": "Calendar create needs a start time...",
  "buttons": [["취소"]],
  "request": {
    "title": "치과",
    "start_at": null,
    "end_at": null,
    "location": null,
    "description": "정기 검진"
  },
  "expected_field": "start_at",
  "conflicts": [],
  "created_at": "2026-05-02T...",
  "updated_at": "2026-05-02T..."
}
```

## 구현 작업

### 1. Helper 추가

`builtin.py` 에 아래 helper 를 추가한다.

```python
def _set_calendar_pending_interaction(session, payload: dict[str, Any], ctx: CommandContext | None = None) -> None:
    ...

def _get_calendar_pending_interaction(session) -> dict[str, Any] | None:
    ...

def _clear_calendar_pending_interaction(session, ctx: CommandContext | None = None) -> None:
    ...

def _clear_calendar_legacy_pending_state(session) -> None:
    ...

def _finish_calendar_interaction(ctx: CommandContext, session, result, *, add_message: bool = True) -> None:
    ...
```

### 2. Legacy key 정리

아래 key 는 새 helper 에서 함께 다룬다.

- `calendar_create_input`
- `calendar_conflict_review`
- `calendar_create_approval`
- `approval_summary`

주의: 처음부터 legacy key 를 삭제하면 기존 code path 와 test 가 크게 흔들릴 수 있다. 1차 구현에서는 새 key 를 canonical 로 쓰되 legacy key 도 호환 저장한다.

### 3. Input prompt 저장 변경

현재 `_calendar_prompt_response` 는 `calendar_create_input` 만 저장한다.

변경 후:

1. `calendar_create_input` 유지
2. `calendar_pending_interaction.kind = collect_input` 저장
3. question/buttons/request/expected_field 포함

### 4. Conflict review 저장 변경

현재 `_calendar_conflict_review_response` 는 `calendar_conflict_review` 와 action result 를 저장한다.

변경 후:

1. `calendar_conflict_review` 유지
2. `calendar_pending_interaction.kind = conflict_review` 저장
3. conflicts 배열에 `conflicting_events` 요약 저장
4. buttons 는 conflict review choices 로 저장

### 5. Approval 저장 변경

`CalendarAutomationSessionRunner.request_create_approval` 에서 `calendar_create_approval` 과 `approval_summary` 를 저장한다.

변경 후:

1. `calendar_create_approval` 유지
2. `approval_summary` 유지
3. `calendar_pending_interaction.kind = create_approval` 저장
4. buttons 는 `[["승인", "취소"]]` 또는 기존 command fallback 과 맞는 label 로 저장

## 테스트 추가

### Backend tests

`test_builtin_calendar.py` 에 추가한다.

1. missing input prompt stores `calendar_pending_interaction`
2. conflict review stores `calendar_pending_interaction.kind = conflict_review`
3. force create changes pending interaction to `create_approval`
4. reschedule changes pending interaction to `collect_input`
5. cancel clears pending interaction and legacy pending keys

### Assertion 예시

```python
pending = ctx.session.metadata["calendar_pending_interaction"]
assert pending["kind"] == "conflict_review"
assert pending["buttons"] == [["그래도 생성 승인 요청", "새 시간 다시 입력"], ["취소"]]
```

## 완료 기준

1. 모든 calendar prompt 생성 경로가 `calendar_pending_interaction` 을 저장한다.
2. legacy metadata 와 새 metadata 가 서로 모순되지 않는다.
3. cancel/deny/approve 이후 pending interaction 이 제거된다.
4. focused calendar command tests 가 통과한다.