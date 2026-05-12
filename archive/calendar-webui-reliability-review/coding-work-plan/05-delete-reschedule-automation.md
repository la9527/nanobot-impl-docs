# 05. Delete and Reschedule Automation

## 목표

calendar conflict 가 났을 때 사용자가 기존 일정을 제거하거나 시간을 변경할 수 있는 실제 실행 경로를 추가한다.

## 수정 대상 파일

1. `source/nanobot/automation/calendar.py`
2. `source/nanobot/automation_results.py`
3. `source/nanobot/command/builtin.py`
4. `source/tests/test_calendar_automation.py`
5. `source/tests/command/test_builtin_calendar.py`
6. n8n workflow 관련 문서 또는 설정 파일

## n8n webhook 필요 항목

### Delete webhook

환경변수 후보:

```text
N8N_CALENDAR_DELETE_WEBHOOK_PATH=webhook/assistant-calendar-delete
```

요청 payload:

```json
{
  "event_id": "...",
  "calendar_id": "primary",
  "title": "...",
  "start_at": "...",
  "end_at": "..."
}
```

응답 payload:

```json
{
  "ok": true,
  "event_id": "...",
  "reply": "Deleted calendar event ..."
}
```

### Update/reschedule webhook

환경변수 후보:

```text
N8N_CALENDAR_UPDATE_WEBHOOK_PATH=webhook/assistant-calendar-update
```

요청 payload:

```json
{
  "event_id": "...",
  "calendar_id": "primary",
  "title": "...",
  "start_at": "2026-05-04T15:00:00+09:00",
  "end_at": "2026-05-04T15:30:00+09:00",
  "location": "...",
  "description": "..."
}
```

## Backend model 추가

`automation_results.py` 에 delete/update result type 을 추가한다.

후보:

- `CalendarDeleteEventResult`
- `CalendarDeleteEventDetails`
- `CalendarUpdateEventResult`
- `CalendarUpdateEventDetails`

또는 초기에는 `CalendarCreateEventResult` 를 재사용하지 않는다. delete 와 create 는 의미가 다르므로 별도 result 가 명확하다.

## Calendar client 추가

`N8NCalendarAutomationClient` 에 추가:

```python
async def delete_event(self, request: CalendarDeleteRequest) -> CalendarDeleteEventResult:
    ...

async def update_event(self, request: CalendarUpdateRequest) -> CalendarUpdateEventResult:
    ...
```

## Session runner 추가

`CalendarAutomationSessionRunner` 에 추가:

```python
async def request_delete_approval(self, session_key: str, request: CalendarDeleteRequest) -> CalendarDeleteEventResult:
    ...

async def approve_delete(self, session_key: str) -> CalendarDeleteEventResult:
    ...

async def deny_delete(self, session_key: str) -> CalendarDeleteEventResult:
    ...
```

reschedule 도 같은 패턴을 따른다.

## Conflict flow 연결

delete 기능 구현 후 conflict review 버튼을 확장한다.

```text
그래도 생성 승인 요청
새 시간 다시 입력
기존 일정 삭제 검토
취소
```

`기존 일정 삭제 검토` 선택 시:

1. conflict list 에서 삭제 후보가 1개면 delete approval 로 바로 이동
2. 후보가 여러 개면 event selection prompt 표시
3. event_id 없는 conflict 는 delete option 을 비활성화하거나 안내한다.

## 안전 규칙

1. 삭제는 반드시 approval 을 거친다.
2. event_id 없는 일정은 삭제 실행하지 않는다.
3. title/time 만으로 바로 삭제하지 않는다.
4. delete 완료 후 list 재확인 또는 action result 로 삭제 대상 표시한다.

## 테스트 계획

1. delete webhook success normalizes result
2. delete webhook 401 maps authentication_needed
3. request_delete_approval stores pending interaction
4. approve_delete clears pending metadata
5. conflict -> delete review -> approve delete
6. event_id missing -> delete option not offered or blocked result

## 완료 기준

1. conflict 해결 선택지에 실제 delete path 가 생긴다.
2. 삭제 승인 전에는 Google Calendar 변경이 일어나지 않는다.
3. 삭제 후 stale conflict/action_result 가 남지 않는다.
4. Playwright Scenario G 가 통과한다.