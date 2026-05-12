# 03. Calendar Flow Cleanup

## 목표

calendar create/check/conflict/approval/cancel/deny 경로에서 stale metadata 와 오래된 action result 가 남지 않게 한다.

## 수정 대상 파일

1. `source/nanobot/command/builtin.py`
2. `source/nanobot/automation/calendar.py`
3. `source/tests/command/test_builtin_calendar.py`
4. `source/tests/test_calendar_automation.py`

## 현재 문제

현재 calendar 상태는 여러 key 에 흩어져 있다.

- `calendar_create_input`
- `calendar_conflict_review`
- `calendar_create_approval`
- `approval_summary`
- `action_result`
- `task_summary`

각 경로가 일부만 수정하기 때문에 아래 문제가 발생한다.

1. 취소했는데 blocked action result 가 남는다.
2. conflict review 후 force create 를 했는데 conflict metadata 가 남는다.
3. reschedule 로 넘어갔는데 이전 conflict card 가 계속 주요 상태처럼 보인다.
4. deny without pending 이 approve 로 fallback 한다.

## 구현 작업

### 1. Cancel result 명시화

conflict review cancel 과 input cancel 은 단순 text message 가 아니라 action result 를 cancelled/rejected 로 교체한다.

예상 result:

```python
CalendarCreateEventResult(
    status="rejected",
    title="Calendar create cancelled",
    summary="The calendar create request was cancelled before approval.",
    ...
)
```

### 2. Force create 전환

`그래도 생성 승인 요청` 선택 시:

1. conflict pending interaction 제거
2. conflict legacy key 제거
3. create approval pending interaction 저장
4. action_result 를 waiting_approval 로 교체
5. approval_summary 저장

### 3. Reschedule 전환

`새 시간 다시 입력` 선택 시:

1. conflict pending interaction 제거
2. collect_input pending interaction 저장
3. `expected_field = start_at`
4. action_result 는 stale conflict 가 아니라 `Calendar create needs a new time` 계열로 교체하거나, 최소한 conflict result 를 main prompt 보다 낮은 priority 로 둔다.

### 4. Approve cleanup

approve 성공 시:

1. pending interaction 제거
2. `calendar_create_approval` 제거
3. `approval_summary` 제거
4. action_result completed 로 교체
5. task_summary completed 로 갱신 또는 제거 정책 결정

### 5. Deny cleanup

deny 시:

1. no pending 이면 approve_create 를 호출하지 않는다.
2. no pending result 는 blocked 또는 rejected 로 저장하되 pending 은 만들지 않는다.
3. pending 이 있으면 all pending metadata 제거 후 rejected result 저장

## 테스트 계획

1. conflict cancel clears all pending metadata
2. input cancel clears pending interaction
3. force create leaves only create approval metadata
4. reschedule leaves only collect_input metadata
5. approve clears pending metadata and approval_summary
6. deny clears pending metadata and does not call approve path

## 완료 기준

1. API metadata 에 서로 다른 pending 상태가 동시에 남지 않는다.
2. dashboard pending count 와 thread prompt 가 같은 상태를 가리킨다.
3. Playwright Scenario C-F 통과 준비가 된다.