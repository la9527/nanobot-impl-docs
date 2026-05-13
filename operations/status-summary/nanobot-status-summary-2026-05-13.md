# Nanobot 작업 상태 정리 - 2026-05-13

## 문서 목적

이 문서는 2026-05-13 기준으로 Telegram linked WebUI thread 에서 확인된 중복 assistant 표시와 `/status` pending 이슈, heartbeat periodic task 가 WebUI idle 상태에서 내부 지시문을 노출하던 경로, 그리고 후속 WebUI duplicate reply root-cause 및 live 다회 검증 결과를 운영 메모 형태로 남기기 위한 문서다.

## 현재 상태 요약

현재 확인된 상태는 다음과 같다.

- source repo `/Volumes/ExtData/Nanobot/source` 는 현재 `feature/nanobot-fork-runtime` 브랜치 기준이다.
- source repo 에는 이번 slice 의 runtime/test/doc 수정이 반영돼 있고, focused test 와 build 는 통과했다.
- Nanobot gateway 는 launchd label `com.nanobot.gateway` 로 실행 중이다.
- Nanobot API 는 launchd label `com.nanobot.api` 로 실행 중이다.
- local llama runtime 은 launchd label `com.nanobot.llama-lfm2-server` 로 실행 중이다.
- live gateway health 는 `http://127.0.0.1:18790/health` 에서 `{"status": "ok"}` 로 확인했다.
- live WebUI bootstrap 은 `X-Nanobot-Auth` 헤더 포함 호출 기준으로 `200 OK` 와 token 발급이 확인됐다.
- Telegram linked WebUI thread 에서 같은 assistant 답변이 연속 중복 노출되던 문제는 현재 browser 기준으로 재현되지 않았다.
- 일반 WebUI websocket thread 와 Telegram linked thread 모두에서 assistant reply duplicate 후속 이슈를 다시 확인했고, 2026-05-13 늦은 밤 live browser 기준 12회 turn 검증에서는 재현되지 않았다.
- 같은 thread 에서 `/status` 실행 시 Telegram 에만 결과가 가고 WebUI 는 계속 pending 으로 남던 문제는 현재 live browser 기준으로 재현되지 않는다.
- WebUI idle 상태에서 heartbeat morning briefing 내부 지시문이 assistant 메시지처럼 노출되던 문제는 현재 source 수정과 launchd 재기동 후 재현 경로가 차단된 상태다.
- linked Telegram WebUI thread 에서 `/help`, `/status`, `/usage` 같은 inline slash command 본문이 markdown 대신 plain-text 규칙으로 표시되어 줄바꿈과 원문 기호가 보존되도록 수정했다.
- `/Volumes/ExtData/Nanobot/docs` 는 source 와 별도 git repo 이므로, 아래 운영 메모 갱신은 source repo commit 과 분리된 docs repo commit 으로 관리된다.

## 이번에 반영한 주요 작업

### 1. Telegram linked WebUI 중복 assistant 표시 경로를 두 군데로 나눠 수정했다

- 첫 번째 원인은 live websocket `message` frame 이 이미 history 로 들어온 assistant 답변을 다시 append 하던 경로였다.
- 이 부분은 `webui/src/hooks/useNanobotStream.ts` 에서 최근 assistant bubble 과 content/buttons/media 를 비교해 짧은 시간 창 안의 중복 append 를 막도록 보강했다.
- 두 번째 원인은 session history 자체에 같은 assistant row 가 연속 저장되어 브라우저가 hydration 시 그대로 보여 주던 경로였다.
- 이 부분은 `webui/src/hooks/useSessions.ts` 에서 연속 duplicate assistant history row 를 collapse 하도록 보강했다.
- 관련 회귀는 `webui/src/tests/thread-shell.test.tsx`, `webui/src/tests/useSessions.test.tsx` 에 추가했다.

### 2. linked Telegram slash command 의 fast path 가 session history 를 건너뛰던 문제를 수정했다

- `/status` 는 일반 assistant turn 과 달리 `AgentLoop.run()` 의 priority command inline path 로 바로 처리된다.
- 기존 `_dispatch_command_inline()` 은 결과를 Telegram outbound 로만 publish 하고 session history 에 user/assistant turn 을 남기지 않았다.
- 그 결과 Telegram 에서는 즉시 답변이 보였지만, WebUI linked session poll 은 새 assistant history 를 찾지 못해 pending 을 끝내지 못했다.
- 이 부분은 `nanobot/agent/loop.py` 에서 inline command result 도 `_process_message()` 와 같은 규칙으로 session history 에 저장하도록 공통 persistence helper 로 정리했다.
- 관련 회귀는 `tests/agent/test_command_persistence.py` 에 `/status` inline path persistence test 를 추가해 검증했다.

### 3. WebUI linked-session waiting placeholder 가 답변 후에도 남는 경로를 정리했다

- backend 수정 후 `/status` 결과 본문은 WebUI 에 붙었지만, `연결된 외부 세션의 응답을 기다리고 있습니다.` placeholder 가 남는 현상이 추가로 확인됐다.
- 이 문제는 `ThreadShell.tsx` 에서 linked session optimistic assistant placeholder 정리와 `waitingExternal` synthetic reasoning cache 가 분리되지 않아 발생했다.
- `webui/src/components/thread/ThreadShell.tsx` 에서 live assistant reply 가 오면 pending poll 을 중단하고 optimistic placeholder 를 제거하도록 보강했다.
- 동시에 `remoteReplyPending` 중의 `waitingExternal` 상태는 reasoning cache 에 저장하지 않도록 바꿔 stale 상태 문구가 남지 않게 했다.
- 관련 회귀는 `webui/src/tests/thread-shell.test.tsx` 에 linked Telegram live reply 도착 시 waiting placeholder 가 사라지는 테스트를 추가했다.

### 4. build, launchd 재기동, live browser 확인까지 한 흐름으로 검증했다

- WebUI 수정 후 `npm run build` 로 production bundle 을 갱신했다.
- 이후 `/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh` 와 `start-nanobot-services.sh` 로 live 서비스를 재기동했다.
- `curl -fsS http://127.0.0.1:18790/health` 와 authenticated `GET /webui/bootstrap` 으로 gateway/WebUI 기본 상태를 확인했다.
- 마지막으로 live browser 에서 실제 Telegram linked thread 를 열고 `/status` 를 다시 실행해 결과 bubble 과 pending 문구 소거를 직접 확인했다.

### 5. heartbeat periodic task 가 WebUI 에 내부 지시문을 노출하던 경로를 차단했다

- idle 상태의 WebUI 에서 `[Your response will be delivered directly to the user's messaging app ...]` 같은 heartbeat preamble 이 assistant 메시지처럼 보이는 현상이 추가로 확인됐다.
- git history 확인 결과 이 preamble 자체는 `fix(heartbeat): prevent internal reasoning leaks and finalization fallback in delivery` 계열 수정에서 heartbeat 최종 응답 품질을 높이기 위해 도입된 안전장치였다.
- 반면 heartbeat 실행을 실제 활성 channel context 로 태우는 구조는 더 이른 `fix(heartbeat): route heartbeat runs to enabled chat context` 계열 수정에서 들어왔고, 이후 `webui_first` proactive routing 과 결합되면서 WebUI target 에서도 내부 실행 입력이 흘러나올 수 있는 조건이 만들어졌다.
- 이번 수정에서는 `nanobot/cli/commands.py` 의 `on_heartbeat_execute()` 가 target channel 과 상관없이 항상 내부 `cli/direct` 채널에서 `agent.process_direct(...)` 를 실행하도록 바꿨다.
- 최종 전달은 기존처럼 `on_heartbeat_notify()` 에서 target channel 정책을 따라가므로, delivery behavior 는 유지하면서 WebUI 누출 경로만 차단했다.
- 관련 회귀는 `tests/cli/test_commands.py` 에 heartbeat target 이 `websocket` 이어도 execution channel 은 `cli/direct` 로 고정되는 focused test 를 추가해 검증했다.

### 6. WebUI 가 slash command 의 plain-text render hint 를 실제로 반영하도록 맞췄다

- backend builtin command 는 이미 `/help`, `/status`, `/usage` 결과에 `render_as = "text"` metadata 를 붙이고 있었지만, WebUI 는 history hydrate 와 live websocket event 모두에서 이 힌트를 버리고 있었다.
- 그 결과 `/help` 는 markdown heading/list 로 재해석되고, `/status` 는 plain text 기준 줄바꿈과 고정폭 느낌이 의도대로 보이지 않았다.
- 이번 수정에서는 websocket outbound frame 에 `render_as` 를 실어 보내고, `webui/src/hooks/useSessions.ts` 와 `webui/src/hooks/useNanobotStream.ts` 에서 이를 `UIMessage.renderAs` 로 보존하도록 연결했다.
- `webui/src/components/MessageBubble.tsx` 에서는 `renderAs === "text"` 인 assistant message 를 markdown 대신 `whitespace-pre-wrap` plain-text block 으로 렌더해 slash command 응답 본문이 그대로 보이게 바꿨다.
- 관련 회귀는 `webui/src/tests/message-bubble.test.tsx`, `webui/src/tests/useSessions.test.tsx`, `webui/src/tests/useNanobotStream.test.tsx`, `tests/channels/test_websocket_channel.py` 로 추가했다.

### 7. todo P1 follow-up 을 추가로 재검증해 close 범위를 정리했다

- live browser 에서 보였던 Telegram linked `/model list` 이중 표시는 raw persisted history 중복이 아니라 stale client state 로 판단했다.
- authenticated `GET /api/sessions/telegram:8688817632/messages` raw payload 를 직접 확인한 결과, `/model list` user turn 은 3개였고 각 turn 에 대응하는 assistant row 도 3개뿐이었다.
- 해당 assistant row 3개는 모두 `metadata.render_as = "text"` 와 `_webui_bridge = true` 를 갖고 있었고, 한 turn 당 같은 assistant row 가 두 번 저장된 흔적은 없었다.
- 같은 thread 를 reload 한 뒤에는 `Model Targets` markdown heading duplicate 는 사라지고 `whitespace-pre-wrap` plain-text block 만 남았으므로, 이번 slice 의 backend persistence 회귀로 보지는 않는다.
- linked Telegram history sampling 과 live event/polled history merge 규칙은 이 메모에 운영 근거로 남겼고, 별도 새 문서를 여는 것보다는 현재 status-summary 기준으로 유지하기로 했다.
- linked-session slice 는 이번 라운드에서 `webui/src/tests/thread-shell-linked-session.test.tsx` 로 분리했고, Telegram/external session focused gate 로 유지하기로 했다. 남은 `thread-shell.test.tsx` 는 owner-aware summary 와 action-result 중심 broader shell gate 로 좁혀졌다.

### 8. action result / detail panel follow-up 은 focused gate 로 다시 확인했다

- `webui/src/tests/thread-shell-action-result.test.tsx` 로 pinned action result 가 scroll 영역 위에 유지되는지와 dismiss clear 가 동작하는지를 다시 확인했다.
- `webui/src/tests/thread-status.test.ts` 와 targeted `thread-shell.test.tsx` 케이스로 action result card 가 이미 있을 때 failed status block 이 중복되지 않는지 다시 확인했다.
- `webui/src/tests/thread-inline-action-result.test.tsx` 로 `한 줄 compact summary + 세부 정보 disclosure` 패턴이 메일/calendar inline result 에서 유지되는지 다시 확인했다.
- targeted `thread-shell.test.tsx` 케이스로 linked external session detail sheet, owner-aware summary, blocked/recent completion hint, duplicate-suppressed proactive state 표현을 다시 검증했다.
- `webui/src/tests/session-metadata.test.ts` 로 metadata localization 경로도 함께 확인해 `대화` / `작업` / `메모리` 축의 detail surface 가 현재 계약과 어긋나지 않는 상태로 봤다.

### 9. WebUI command surface / dashboard split phase-1 은 runtime payload와 focused gate 기준으로 close 했다

- `App.tsx` 의 `emptyThreadView` 와 `Sidebar.tsx` action 연결을 다시 확인한 결과, `대시보드` 와 `새 채팅` 은 이미 explicit state 로 분리되어 있었다.
- `ThreadShell.tsx` 는 `dashboard` 에서만 `AssistantDashboard` feed surface 를 노출하고, `new-chat` 에서만 hero composer 아래 quick action strip 을 보이도록 이미 분기돼 있었다.
- live `/api/commands` payload 를 직접 조회한 결과 현재 명령 집합은 `/new`, `/status`, `/model`, `/usage`, `/mail`, `/calendar`, `/history`, `/dream`, `/dream-log`, `/dream-restore`, `/help` 를 포함했고, WebUI palette source-of-truth 는 이 endpoint 를 그대로 사용하고 있었다.
- 남아 있던 drift 는 test fixture 가 `/dream-restore` 를 빠뜨리고 있던 coverage gap 뿐이어서, `webui/src/tests/api.test.ts` 와 `webui/src/tests/thread-composer.test.tsx` 를 runtime payload 기준으로 보강했다.
- 이후 `npm test -- src/tests/api.test.ts src/tests/thread-composer.test.tsx src/tests/app-layout.test.tsx` 와 `npm run build` 를 다시 실행해 phase-1 범위를 focused gate 로 재검증했다.

### 10. WebUI duplicate reply 후속 이슈는 stream 단계와 persisted history 단계로 나눠 root cause 를 정리했다

- 일반 WebUI websocket chat 에서 같은 assistant 답변이 두 번 보이는 후속 제보가 다시 들어왔고, 이번에는 live stream 단계와 persisted history hydrate 단계를 분리해 확인했다.
- 첫 번째 원인은 `webui/src/hooks/useNanobotStream.ts` 에서 `delta -> stream_end -> final message -> turn_end` 순서일 때 streamed placeholder ID 를 `stream_end` 에서 너무 일찍 잃어 final assistant message 가 기존 bubble 을 교체하지 못하던 경로였다.
- 이 부분은 마지막 streamed message ID 를 turn 종료 전까지 유지하도록 수정했고, `webui/src/tests/useNanobotStream.test.tsx` 에 동일 event sequence 회귀를 추가했다.
- 두 번째 원인은 `webui/src/hooks/useSessions.ts` 에서 persisted history assistant row 두 개가 같은 content 와 같은 `visible_reasoning` 을 가진 채 저장되면, hydration 이 `trace + assistant + trace + assistant` 로 풀리면서 중복 답변이 그대로 남던 경로였다.
- 이 부분은 UI message hydrate 전에 raw assistant history row 단계에서 duplicate collapse 를 먼저 수행하도록 바꿨고, 동점일 때는 뒤의 whitespace 변형이 아니라 먼저 저장된 row 를 유지하도록 tie-break 를 정리했다.
- 관련 회귀는 `webui/src/tests/useSessions.test.tsx` 에 `visible_reasoning` 이 붙은 duplicate persisted assistant reply collapse 케이스를 추가해 검증했다.
- 결과적으로 이번 duplicate 이슈는 단순 frontend append 방지 하나가 아니라 `live stream dedupe + persisted history dedupe` 두 단계가 같이 있어야 막히는 구조로 정리됐다.

## 이번에 확인된 운영 포인트

### 1. linked Telegram 이슈는 frontend 만이 아니라 backend inline command path 도 같이 봐야 한다

- 같은 증상처럼 보여도 원인이 history hydration, websocket mirror, inline slash dispatch 로 갈라질 수 있다.
- 특히 `/status` 같은 priority slash command 는 일반 assistant turn 과 다른 제어 경로를 타므로 별도 확인이 필요했다.

### 2. WebUI linked session poll 은 결국 session history 를 기준으로 끝난다

- Telegram 채널에 답이 도착했다는 사실만으로는 WebUI pending 이 풀리지 않는다.
- linked thread 는 새 assistant history row 확인을 기준으로 종료되므로, backend persistence 가 빠지면 pending 이 길게 남는다.

### 3. `waitingExternal` 는 reasoning 처럼 cache 하면 안 된다

- reasoning cache 는 새로고침 후에도 남겨야 하는 안전한 한 줄 상태를 위한 장치다.
- 반면 linked external waiting 문구는 일시 상태이므로 assistant reply 이후 즉시 사라져야 한다.
- 이번 수정으로 `waitingExternal` 는 cache 대상에서 제외했다.

### 4. launchd 재기동 직후 health/bootstrap 은 잠깐 실패할 수 있다

- 실제로 재기동 직후 첫 `curl` 은 connection refused 가 발생했다.
- 하지만 launchd 상태와 gateway log 를 확인한 뒤 재시도하면 정상 복구됐다.
- 운영 검증 시에는 restart 직후 1회 실패만으로 장애로 단정하지 않는 편이 맞다.

### 5. heartbeat 는 delivery target 과 execution channel 을 분리해서 봐야 한다

- 현재 구조에서 heartbeat 결과가 어느 채널에 최종 전달되는지는 proactive target policy 가 결정한다.
- 하지만 실제 LLM 실행은 내부 heartbeat session 과 `cli/direct` 채널에서 수행해야 WebUI, Telegram, 기타 external channel 로 중간 출력이 새지 않는다.
- follow-through 를 위해 channel session 에 assistant delivery 를 주입하는 경로는 `notify` 단계에서 유지하면 충분하고, execution 단계까지 target channel 과 결합할 필요는 없었다.

## 검증 결과

source-tree 기준 focused 검증:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_command_persistence.py -q
# 3 passed

PYTHONPATH=$PWD ./.venv/bin/pytest tests/cli/test_commands.py -q -k 'heartbeat_executes_explicit_tasks_without_proactive_context or heartbeat_executes_on_internal_channel_even_with_websocket_target'
# 2 passed

cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-shell-linked-session.test.tsx src/tests/thread-shell.test.tsx
# 47 passed

npm test -- src/tests/api.test.ts src/tests/thread-composer.test.tsx src/tests/app-layout.test.tsx
# 30 passed

npm run build
# 통과

npm test -- src/tests/thread-shell-action-result.test.tsx src/tests/thread-status.test.ts src/tests/session-metadata.test.ts
# 6 passed

npm test -- src/tests/thread-shell.test.tsx -t "renders the latest mail action result from session metadata in the pinned card|renders approval pending badge for a mail send action result|does not duplicate a failed status block when an action result card is already shown"
# 3 passed, 40 skipped

npm test -- src/tests/thread-shell.test.tsx -t "keeps linked session details available after loading completed external history|renders linked continuity metadata in the external session summary|renders an owner-aware summary block from linked sessions and pending approvals|renders blocked and recent completion hints in the owner-aware summary|renders duplicate-suppressed proactive state in the owner-aware summary"
# 5 passed, remaining skipped

npm test -- src/tests/thread-inline-action-result.test.tsx
# 3 passed

npm test -- src/tests/message-bubble.test.tsx src/tests/useSessions.test.tsx src/tests/useNanobotStream.test.tsx
# 39 passed

npm run build
# 통과

cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/channels/test_websocket_channel.py -q -k response_model_metadata
# 2 passed
```

live runtime 기준 검증:

```bash
/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh

launchctl list | grep nanobot
curl -fsS http://127.0.0.1:18790/health
curl -fsS -H 'X-Nanobot-Auth: <bootstrap-secret>' http://127.0.0.1:8765/webui/bootstrap
```

live browser 기준 확인:

- authenticated WebUI browser 에서 실제 Telegram linked thread `채팅 868881` 을 열었다.
- `/status` 실행 후 WebUI 에 assistant status bubble 이 즉시 붙는 것을 확인했다.
- 같은 turn 에서 `연결된 외부 세션의 응답을 기다리고 있습니다.` 문구가 최종 DOM 에 남지 않는 것을 확인했다.
- 중복 assistant bubble 은 현재 동일 thread 에서 재현되지 않았다.
- `/model list` duplicate surface 는 raw history API 와 reload 후 DOM 상태를 같이 확인한 결과 persisted backend duplicate 가 아니라 stale client state 로 판정했다.
- heartbeat prompt 누출 이슈는 이번 메모 작성 시점에 live browser 에서 재현을 다시 만들지 않았고, focused test + launchd live health 재확인 기준으로 차단 상태만 확인했다.
- duplicate reply 후속 검증은 authenticated WebUI browser 에서 일반 WebUI websocket session 6회, Telegram linked session 6회로 총 12회 turn 을 실제 전송해 확인했다.
- 일반 WebUI websocket session 에서는 짧은 응답 4회와 긴 응답 2회를 보냈고, 각 turn 마다 `Copy reply` 증가량이 정확히 `+1` 이며 reload 후 hydrate 상태에서도 추가 bubble 이 생기지 않는 것을 확인했다.
- Telegram linked session 에서는 짧은 응답 3회와 긴 응답 3회를 보냈고, 각 turn 마다 linked-session pending 종료 후 `Copy reply` 증가량이 정확히 `+1` 이며 settle 구간 이후 추가 증가가 없는 것을 확인했다.
- authenticated `GET /api/sessions/websocket:6eaca585-85f3-4c61-aac2-d558a5612876/messages` 조회 결과 최근 14행은 정확히 `user 7 + assistant 7` 이었고, assistant row 는 모두 단일 persisted row 였다.
- authenticated `GET /api/sessions/telegram:8688817632/messages` 조회 결과 최근 12행은 정확히 `user 6 + assistant 6` 이었고, linked Telegram assistant row 중 같은 turn 이 두 번 저장된 흔적은 없었다.

## 현재 남아 있는 제약과 리스크

- 기존 session history 에 이미 누적된 비연속 duplicate row 까지 정리하는 migration 은 아직 없다. 현재 보강은 연속 duplicate collapse 와 live append 억제에 집중돼 있다.
- 현재 live 12회 검증에서는 duplicate reply 를 재현하지 못했지만, 향후 다시 보이면 동일 세션 key 기준 raw history 와 browser hydrate 상태를 같이 캡처해 stream 단계인지 persisted history 단계인지 먼저 분리해서 봐야 한다.
- `thread-shell` 전체 파일은 현재도 다양한 linked-session 시나리오를 함께 다루므로, 이후 회귀가 생기면 narrow test 를 먼저 추가하는 편이 안전하다.
- heartbeat periodic task 의 최종 target routing 은 유지했지만, WebUI idle 상태에서 같은 누출이 다시 보이면 websocket outbound 경로 또는 mirrored delivery 경로를 추가로 점검해야 한다.
- `/Volumes/ExtData/Nanobot/docs` 는 source 와 별도 git repo 이므로 status-summary 와 guide 문서 변경은 source repo commit/push 와 분리된 docs repo history 로 추적된다.

## 현재 기준 권장 운영 명령

```bash
git -C /Volumes/ExtData/Nanobot/source status --short --branch

cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_command_persistence.py -q
PYTHONPATH=$PWD ./.venv/bin/pytest tests/cli/test_commands.py -q -k 'heartbeat_executes_explicit_tasks_without_proactive_context or heartbeat_executes_on_internal_channel_even_with_websocket_target'

cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-shell.test.tsx
npm run build

/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
launchctl list | grep nanobot
curl -fsS http://127.0.0.1:18790/health
curl -fsS -H 'X-Nanobot-Auth: <bootstrap-secret>' http://127.0.0.1:8765/webui/bootstrap
```

## 다음 작업 후보

- stale client state 가 다시 보이면 Telegram linked thread reload/cache invalidation 타이밍을 추가 샘플링한다.

## 결론

2026-05-13 작업의 핵심은 Telegram linked WebUI 이슈를 단순 UI glitch 로 보지 않고, history hydration, live websocket mirror, inline slash command persistence, waiting placeholder cache, 그리고 duplicate reply 의 `stream 단계 / persisted history 단계` 를 각각 분리해 원인별로 정리한 것이다. 현재는 focused test, production build, launchd live runtime, authenticated bootstrap, 일반 WebUI 6회 + Telegram linked 6회 실제 browser turn 검증, raw history API 대조까지 마친 상태이며, 운영 관점에서 보면 duplicate reply 와 linked `/status` 흐름은 다시 안정화된 상태로 돌아왔다.