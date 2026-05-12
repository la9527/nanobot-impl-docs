# 04. Natural Language Calendar Routing

## 목표

사용자가 slash command 를 몰라도 calendar 기능을 사용할 수 있게 한다. 자연어 calendar 요청은 일반 LLM 답변으로 빠지기 전에 command/automation path 로 연결되어야 한다.

## 수정 대상 파일

1. `source/nanobot/agent/loop.py`
2. `source/nanobot/command/builtin.py`
3. 새 파일 후보: `source/nanobot/command/calendar_intent.py`
4. `source/tests/command/test_builtin_calendar.py`
5. 새 테스트 후보: `source/tests/command/test_calendar_intent.py`

## 접근 방식

1차는 rule-based intent bridge 로 시작한다.

이유:

- calendar 기능은 안전성이 중요하다.
- LLM classification 을 먼저 넣으면 misroute 위험이 있다.
- 지금 필요한 것은 “명확한 calendar 요청이 일반 답변으로 빠지지 않게 하는 것”이다.

## Intent 분류

### list intent

예시:

- `오늘 일정 보여줘`
- `구글 캘린더 일정 리스트 보여줘`
- `캘린더 확인해줘`
- `내 일정 알려줘`

동작:

- `/calendar list` 와 같은 경로로 연결

### create intent

예시:

- `내일 3시에 치과 일정 잡아줘`
- `캘린더에 회의 추가해줘`
- `5월 4일 2시에 Nanobot 검증 일정 넣어줘`

동작:

- structured parse 가 가능하면 create request
- 부족하면 collect_input pending interaction

### delete intent

예시:

- `이 일정 지워줘`
- `Nanobot webui calendar validation 삭제해줘`
- `5월 4일 14시 일정 삭제`

동작:

- delete 기능 구현 전에는 candidate selection 준비 상태 또는 안내 prompt
- delete 기능 구현 후에는 list/search -> delete approval 로 연결

### reschedule intent

예시:

- `이 일정 3시로 옮겨줘`
- `겹치니까 다른 시간으로 잡아줘`

동작:

- update 기능 구현 전에는 collect_input + create/check path
- update 기능 구현 후에는 reschedule approval path

## 구현 위치

권장 구조:

```python
def try_calendar_intent(raw: str) -> CalendarIntent | None:
    ...

async def dispatch_calendar_intent(ctx: CommandContext, intent: CalendarIntent) -> OutboundMessage | None:
    ...
```

이 로직은 `builtin.py` 에 바로 넣기보다 `calendar_intent.py` 로 분리하는 것이 좋다. 다만 처음에는 command package 내부로 작게 둔다.

## 안전 규칙

1. delete/create/update 는 즉시 실행하지 않는다.
2. ambiguous delete 는 candidate selection 으로 보낸다.
3. 시간이 불명확하면 collect_input 으로 보낸다.
4. intent confidence 가 낮으면 일반 LLM 답변이 아니라 clarification prompt 를 띄운다.

## 테스트 계획

1. `오늘 일정 보여줘` -> calendar list runner 호출
2. `구글 캘린더 일정 리스트를 먼저 보여줘` -> calendar list runner 호출
3. `내일 3시에 치과 일정 잡아줘` -> create/check path 또는 collect_input
4. `이거 지워줘 ...` -> delete candidate path 또는 not-yet-implemented prompt
5. unrelated text 는 calendar intent 로 오인하지 않음

## 완료 기준

1. 명확한 calendar 자연어 요청이 일반 LLM 답변으로 빠지지 않는다.
2. delete 가 구현되기 전에도 “툴이 없다” 답변 대신 안전한 next step prompt 를 제공한다.
3. command tests 와 live Scenario B 가 통과한다.