# 07. Test and Live Validation

## 목표

각 코딩 단계가 실제 Nanobot live runtime 에 반영되고, WebUI/Telegram/Calendar 기능이 end-to-end 로 동작하는지 검증한다.

## Local focused tests

### Backend calendar tests

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/command/test_builtin_calendar.py tests/test_calendar_automation.py -q
```

### Agent mirror / ask-user tests

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_ask_user.py -q
```

### Telegram fallback tests

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/channels/test_telegram_channel.py -q
```

### WebUI prompt/dashboard tests

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-shell.test.tsx src/tests/useSessions.test.tsx src/tests/app-layout.test.tsx
```

## Build

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm run build
```

확인할 것:

1. `tsc` 통과
2. Vite build 통과
3. 새 `assets/index-*.js` 생성

## Live restart

Nanobot instruction 기준으로 launchd path 를 사용한다.

```bash
/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
```

Health check:

```bash
curl -fsS http://127.0.0.1:18790/health
curl -i http://127.0.0.1:8765/webui/bootstrap
```

## Browser validation

### Asset check

확인:

1. browser loaded script hash
2. dist `index.html` script hash
3. 둘이 일치해야 함

### Console check

기록:

1. 반복 401 여부
2. WebSocket reconnect 여부
3. failed resource URL

## Playwright scenario execution

상위 문서 [04-playwright-regression-plan.md](../04-playwright-regression-plan.md) 의 Scenario A-H 를 실행한다.

최소 release gate:

1. Scenario A: Dashboard navigation
2. Scenario B: Natural language calendar list
3. Scenario C: Conflict prompt persistence
4. Scenario D: Reschedule flow
5. Scenario E: Force create approval
6. Scenario F: Cancel and deny cleanup

Delete 기능 구현 후 추가 gate:

1. Scenario G: Delete event flow
2. Scenario H: Telegram fallback

## Result logging

각 실행 결과는 아래 형식으로 `docs/archive/calendar-webui-reliability-review/validation-runs/` 아래에 저장한다.

파일명 예시:

```text
2026-05-02-phase-1-pending-interaction.md
```

형식:

```markdown
# Validation Run - 2026-05-02 - Phase 1

## Build

- Command:
- Result:
- Asset hash:

## Backend Tests

- Command:
- Result:

## WebUI Tests

- Command:
- Result:

## Playwright

| Scenario | Result | Notes |
| --- | --- | --- |
| A | pass/fail | ... |

## API Metadata Snapshot

```json
{}
```

## Follow-up

1. ...
```

## 완료 기준

1. test command 결과가 문서에 남는다.
2. live browser 결과가 문서에 남는다.
3. 실패 scenario 는 다음 phase 로 넘어가기 전에 backlog 로 남긴다.
4. source foreground 결과와 launchd live runtime 결과를 구분해서 적는다.