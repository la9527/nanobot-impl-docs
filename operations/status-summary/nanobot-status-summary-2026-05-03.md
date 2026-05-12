# Nanobot 작업 상태 정리 - 2026-05-03

## 문서 목적

이 문서는 2026-05-03 기준으로 Telegram 에 반복 전달되던 calendar create cancelled proactive briefing 의 원인, 근본 수정, 자연어 calendar create/update/delete/approve/cancel 확장, heartbeat push 억제 보강, Telegram/WebUI chat context clear 기능, 검증 상태를 정리한 운영 메모다.

## 현재 상태 요약

현재 확인된 상태는 다음과 같다.

- Nanobot gateway 는 launchd label `com.nanobot.gateway` 로 실행 중이다.
- Nanobot API 는 launchd label `com.nanobot.api` 로 실행 중이다.
- source tree `/Volumes/ExtData/Nanobot/source` 변경이 editable install 기반 live runtime 에 반영됐다.
- live gateway health 는 `http://127.0.0.1:18790/health` 에서 `{"status":"ok"}` 로 확인했다.
- live API health 는 `http://127.0.0.1:8900/health` 에서 `{"status":"ok"}` 로 확인했다.

## 이번에 확인한 원인

Telegram 으로 계속 전달되던 `캘린더 생성이 취소된 상태` 메시지는 활성 calendar approval 이 살아 있어서 발생한 것이 아니었다.

실제 원인은 다음 흐름이었다.

- `telegram:8688817632` session metadata 에 `action_result.status=rejected`, `error.code=approval_rejected`, `title=Calendar create cancelled` 가 남아 있었다.
- 기존 `task_summary` 정규화가 `rejected` action result 를 `blocked` 로 분류했다.
- heartbeat proactive context builder 가 이 blocked task 를 morning briefing prompt 에 계속 포함했다.
- heartbeat 가 30분마다 같은 briefing 을 생성하고 Telegram 으로 `_channel_delivery=true` 메시지를 전달했다.

## 이번에 반영한 주요 작업

### Rejected approval 정규화 수정

- `source/nanobot/session/continuity.py` 에서 `action_result.status=rejected` 를 더 이상 `blocked` task 로 승격하지 않도록 수정했다.
- rejected approval 은 사용자가 닫은 terminal outcome 으로 보고 `task_summary.status=completed` 로 정규화한다.
- `failed`, `blocked` 는 기존처럼 follow-up 이 필요한 blocked task 로 유지한다.

### Proactive digest 이중 방어

- `source/nanobot/heartbeat/proactive.py` 에서 `status=rejected` 이고 `error.code=approval_rejected` 인 action result 는 proactive context 에서 제외하도록 했다.
- 기존 session file 에 stale `task_summary.status=blocked` 가 남아 있어도, action result 를 함께 보면 closed user rejection 이므로 blocked digest 에 넣지 않는다.
- `API session follow-up`, `Reopen the interrupted session and continue the task` 같은 일반 중단 세션은 외부 push briefing 재료에서 제외한다.
- proactive context 가 비어 있으면 heartbeat 실행 자체를 건너뛰어, 보낼 정보가 없을 때 Telegram push 를 만들지 않는다.
- 같은 category 와 같은 summary 를 이미 보낸 경우 `duplicate` 으로 억제해 동일 briefing 을 반복 전송하지 않는다.

### 자연어 calendar create 1차 지원

- `source/nanobot/command/builtin.py` 에 자연어 calendar create interceptor 를 추가했다.
- `2026-05-05 오후 3시에 치과 일정 1시간 잡아줘` 같은 입력을 `CalendarCreateRequest` 로 정규화한다.
- 기존 `/calendar create` 와 동일하게 conflict check, conflict review, create approval 흐름을 재사용한다.
- 종료 시간이 없으면 기존 waiting-input 경로로 end time 을 다시 질문한다.

### 자연어 calendar update/delete 1차 지원

- `source/nanobot/automation/calendar.py` 에 `CalendarUpdateRequest`, `CalendarDeleteRequest`, update/delete client method, update/delete approval runner 를 추가했다.
- `source/nanobot/automation_results.py` 에 `CalendarUpdateEventResult`, `CalendarDeleteEventResult` 를 추가했다.
- `source/nanobot/command/builtin.py` 에 자연어 update/delete interceptor 를 추가했다.
- update/delete 는 제목, 날짜, 기존 시작 시간으로 기존 event 후보를 먼저 조회하고 후보가 하나로 확정될 때만 approval pending 으로 넘어간다.
- 실제 수정/삭제 실행은 `/calendar approve` 이후에만 수행하며, `/calendar cancel` 또는 `/calendar deny` 로 취소할 수 있다.
- n8n workflow 가 HTTP 200 으로 `calendar-update-error`, `calendar-delete-error`, `*-not-found` payload 를 반환해도 completed 로 오인하지 않고 failed/blocked action result 로 정규화한다.
- `5월 2일 오후 3시` 처럼 연도를 생략한 한국어 날짜/시간을 자연어 create/update/delete 에서 처리한다.
- create 는 연도 생략 날짜가 이미 지난 날짜이면 다음 해로 roll-forward 하는 기존 예약 성격을 유지한다.
- update/delete 는 과거 일정도 수정/삭제 대상이 될 수 있으므로 연도 생략 날짜를 현재 연도로 해석한다.

### 자연어 calendar approve/cancel 지원

- pending calendar create/update/delete approval 이 있을 때 `승인해줘`, `좋아 진행해줘`, `확정해줘` 같은 자연어를 `/calendar approve` 와 같은 효과로 처리한다.
- pending calendar approval 또는 입력/충돌 검토가 있을 때 `취소해줘`, `반려해줘`, `거절해줘` 같은 자연어를 `/calendar cancel` 과 같은 효과로 처리한다.
- conflict review 상태에서 `승인해줘` 는 기존 `그래도 생성 승인 요청` 버튼과 같은 의미로 create approval 단계로 넘긴다.
- pending 상태가 없으면 일반 대화의 `승인`, `취소` 표현은 calendar 명령으로 가로채지 않는다.

### Chat context clear 명령 추가

- `source/nanobot/command/builtin.py` 에 `/context`, `/context clear`, `/clear` 명령을 추가했다.
- `/context` 는 현재 채팅 세션에 저장된 message count 와 clear 방법을 보여준다.
- `/context clear` 와 `/clear` 는 현재 채팅 세션의 `session.messages` 를 비우고, pending approval, stale action result, proactive summary, runtime checkpoint, auto-compact `_last_summary` 같은 transient context marker 를 제거한다.
- 모델 target override, response footer 설정, owner profile, 장기 memory 는 유지한다.
- clear 명령 자체와 clear 결과 응답은 다시 session history 에 저장하지 않도록 command persistence 경로에 internal `skip` 처리를 추가했다.
- Telegram bot command menu 에 `clear`, `context` 를 노출했다.

## 검증 결과

source-tree 기준 focused test:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_heartbeat_proactive.py tests/agent/test_session_manager_history.py -q
# 30 passed in 0.25s
```

source-tree 기준 확장 focused test:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_heartbeat_proactive.py tests/agent/test_session_manager_history.py tests/channels/test_websocket_http_routes.py -q
# 55 passed in 7.04s
```

heartbeat push 억제 focused test:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_heartbeat_proactive.py tests/agent/test_heartbeat_service.py tests/heartbeat/test_heartbeat_deliverability.py -q
# 34 passed in 0.21s
```

context clear / command persistence / Telegram command menu focused test:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_command_persistence.py tests/channels/test_telegram_channel.py tests/command/test_command_router.py tests/command/test_builtin_model.py tests/command/test_builtin_usage.py -q
# 87 passed in 1.99s
```

context clear 포함 확장 regression:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_command_persistence.py tests/agent/test_loop_save_turn.py tests/agent/test_stop_preserves_context.py tests/agent/test_task_cancel.py tests/channels/test_telegram_channel.py tests/command/test_command_router.py tests/command/test_builtin_model.py tests/command/test_builtin_usage.py tests/command/test_builtin_calendar.py tests/test_calendar_automation.py tests/agent/test_heartbeat_proactive.py tests/agent/test_heartbeat_service.py tests/heartbeat/test_heartbeat_deliverability.py -q
# 194 passed in 2.41s
```

자연어 calendar approve/cancel 및 연도 생략 날짜 포함 확장 regression:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_command_persistence.py tests/agent/test_loop_save_turn.py tests/agent/test_stop_preserves_context.py tests/agent/test_task_cancel.py tests/channels/test_telegram_channel.py tests/command/test_command_router.py tests/command/test_builtin_model.py tests/command/test_builtin_usage.py tests/command/test_builtin_calendar.py tests/test_calendar_automation.py tests/agent/test_heartbeat_proactive.py tests/agent/test_heartbeat_service.py tests/heartbeat/test_heartbeat_deliverability.py -q
# 199 passed in 2.42s
```

자연어 calendar create focused test:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/command/test_builtin_calendar.py tests/test_calendar_automation.py tests/test_automation_results.py -q
# 39 passed in 0.26s
```

자연어 calendar update/delete live dry-run:

- live n8n summary webhook 으로 `Nanobot calendar live validation` event 후보를 조회했다.
- `2026-05-03 오후 3시에 Nanobot calendar live validation 일정을 오후 4시로 변경해줘` 는 `Calendar update approval required`, `action_status=waiting_approval`, `pending_kind=update_approval` 까지 확인했다.
- `2026-05-03 오후 3시에 Nanobot calendar live validation 일정 삭제해줘` 는 `Calendar delete approval required`, `action_status=waiting_approval`, `pending_kind=delete_approval` 까지 확인했다.
- 두 dry-run 모두 `/calendar approve` 를 실행하지 않았으므로 실제 calendar update/delete 는 발생하지 않았다.

live session 재해석 확인:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/python - <<'PY'
from pathlib import Path
from nanobot.session.manager import SessionManager
from nanobot.heartbeat.proactive import build_proactive_context
sm = SessionManager(Path.home() / '.nanobot' / 'workspace')
print(build_proactive_context(sm.list_sessions(), max_digest_items=3) or '<empty>')
PY
```

확인 결과 `Calendar create cancelled` 는 proactive context 에서 빠지고, 남은 항목은 API session follow-up blocked task 뿐이었다.

heartbeat push 억제 보강 후 live session 기준 `build_proactive_context(...)` 는 `<empty>` 로 확인했다. 즉 현재 상태에서는 별도 전달할 정보가 없으므로 heartbeat 가 Telegram push 를 생성하지 않는다.

live runtime 검증:

- `/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh && /Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh` 로 launchd gateway/API 재기동 완료
- `launchctl list | rg 'nanobot'` 에서 `com.nanobot.gateway`, `com.nanobot.api`, `com.nanobot.llama-lfm2-server` 확인
- `curl -fsS http://127.0.0.1:18790/health` 성공
- `curl -fsS http://127.0.0.1:8900/health` 성공
- live `/api/sessions` 기준 `telegram:8688817632` 의 `task_summary.status` 가 `completed` 로 재해석되는 것을 확인
- 자연어 calendar create 변경 후에도 launchd gateway/API 재기동과 health 확인을 다시 완료

## 현재 남아 있는 제약과 리스크

- 이번 수정은 stale rejected approval 이 proactive blocked digest 로 반복 전달되는 문제를 막는다.
- `HEARTBEAT.md` 의 morning briefing task 자체는 계속 활성 상태이므로, 실제 blocked task 가 남아 있으면 Telegram briefing 은 계속 발생할 수 있다.
- 일반 API session follow-up blocked task 는 외부 push 대상에서 제외된다.
- 기존 Telegram session history 에 이미 전달된 과거 `_channel_delivery=true` 메시지는 삭제하지 않았다.
- 새 `/clear` 명령은 Nanobot 이 저장한 session context 를 비우지만, Telegram 앱 자체의 채팅방 메시지를 Telegram 서버에서 삭제하지는 않는다.
- 이미 Dream memory 로 들어간 장기 memory 는 `/clear` 대상이 아니며, 필요한 경우 별도 forget/memory 보정 흐름으로 다뤄야 한다.
- 자연어 calendar create/update/delete 는 deterministic parser 기반 1차 지원이므로, 복잡한 반복 일정, 참석자, 장소 추론, 제목 변경은 아직 포함하지 않는다.
- 자연어 update 는 기존 시작 시간과 새 시작 시간이 모두 있어야 안정적으로 처리된다.
- 자연어 delete 는 후보가 하나로 확정될 때만 approval pending 으로 넘어간다.
- 연도 생략 날짜는 현재 운영 timezone 기준 현재 연도를 사용한다. create 는 미래 예약을 우선해 지난 날짜를 다음 해로 넘기고, update/delete 는 현재 연도 그대로 사용한다.

## 권장 운영 명령

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_heartbeat_proactive.py tests/agent/test_session_manager_history.py tests/channels/test_websocket_http_routes.py -q
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_heartbeat_proactive.py tests/agent/test_heartbeat_service.py tests/heartbeat/test_heartbeat_deliverability.py -q
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_command_persistence.py tests/channels/test_telegram_channel.py tests/command/test_command_router.py tests/command/test_builtin_model.py tests/command/test_builtin_usage.py -q
PYTHONPATH=$PWD ./.venv/bin/pytest tests/command/test_builtin_calendar.py tests/test_calendar_automation.py tests/test_automation_results.py -q
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_command_persistence.py tests/agent/test_loop_save_turn.py tests/agent/test_stop_preserves_context.py tests/agent/test_task_cancel.py tests/channels/test_telegram_channel.py tests/command/test_command_router.py tests/command/test_builtin_model.py tests/command/test_builtin_usage.py tests/command/test_builtin_calendar.py tests/test_calendar_automation.py tests/agent/test_heartbeat_proactive.py tests/agent/test_heartbeat_service.py tests/heartbeat/test_heartbeat_deliverability.py -q

/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
curl -fsS http://127.0.0.1:18790/health
curl -fsS http://127.0.0.1:8900/health
```

## 다음 작업 후보

- API session follow-up blocked task 의 UI 표시 문구를 WebUI dashboard 에서 더 명확히 다듬을지 검토한다.
- proactive duplicate suppression 에 시간 기반 재알림 정책이 필요한지 운영 경험을 보고 조정한다.
- `/clear` 를 실제 Telegram 앱에서 호출해 bot command menu 갱신과 WebUI mirrored session refresh 동작을 한 번 더 확인한다.
- calendar update/delete 의 n8n workflow 가 `event_id` 를 직접 우선 사용하도록 보강할지 검토한다.
- 후보가 여러 개일 때 번호 선택형 follow-up interaction 을 추가할지 검토한다.

## 결론

반복 Telegram 알림의 직접 원인은 stale calendar rejection 이 blocked task 로 재분류된 것이었다. 현재 source 코드와 live runtime 모두에서 해당 session 은 `completed` 로 재해석되고, proactive context 에서 calendar cancellation 항목은 제외되는 것을 확인했다. 자연어 calendar create/update/delete 는 현재 approval-first 방식으로 동작하며, update/delete 는 live dry-run 에서 실제 변경 없이 approval pending 까지 검증했다. 추가로 자연어 approve/cancel 과 연도 생략 날짜를 지원하고, `/context clear` 와 `/clear` 를 통해 Telegram/WebUI 채팅별 저장 context 를 사용자가 직접 비울 수 있는 경로를 마련했다.
