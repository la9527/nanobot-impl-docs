# Nanobot 작업 상태 정리 - 2026-05-01

## 문서 목적

이 문서는 2026-05-01 기준으로 최근 Nanobot 개인비서 backlog 구현, proactive heartbeat 라우팅/quiet-hours 보강, mail/calendar pilot 명령 경로 추가, owner-aware WebUI 정리, Direction A 문서군 재정리 상태를 운영 메모 형태로 남기기 위한 문서다.

이번 시점에서 중요한 변화는 네 가지다.

- `mail` / `calendar` automation 결과를 session metadata 와 WebUI status surface 에 연결하는 최소 backbone 이 source tree 기준으로 정리됐다.
- proactive heartbeat 가 `webui_first=true` 여도 항상 WebUI 로만 가지 않고, 낮 시간대에는 최근 external channel 을 우선하고 quiet hours 에는 WebUI hold/suppression 으로 돌아가는 정책이 live 기준으로 검증됐다.
- WebUI 가 owner-aware summary, task summary, action result preview, memory correction draft injection 을 표시하도록 보강됐지만, 상단 정보 밀도가 높아 thread 화면이 과밀해지는 문제가 확인돼 별도 UX 재설계 검토 문서를 `docs/webui-ux-redesign/` 아래에 추가했다.
- `docs/todo.md` 와 `docs/assistant-direction-a/**` 는 active backlog 와 archive/reference 경계를 다시 정리했지만, 이 문서군은 현재 git root 밖에 있으므로 이번 push 범위에는 포함되지 않는다.

## 현재 상태 요약

- git 기준 실제 저장소 루트는 `/Volumes/ExtData/Nanobot/source` 이고, 현재 브랜치는 `feature/nanobot-fork-runtime` 이다.
- `docs/status-summary` 와 `docs/todo.md`, `docs/assistant-direction-a/**` 는 `source` 저장소 바깥 운영 문서 영역이다.
- live launchd 상태 확인 결과 `com.nanobot.gateway`, `com.nanobot.api`, `com.nanobot.llama-lfm2-server` 가 모두 올라와 있다.
- `curl -fsS http://127.0.0.1:18790/health` 는 `{"status": "ok"}` 를 반환했다.
- `curl -fsS http://127.0.0.1:8765/webui/bootstrap` 는 정상 응답을 반환했고, 현재 bootstrap payload 의 `model_name`, `active_target` 는 `smart-router` 로 보인다.
- source 기준 focused pytest 재검증은 아래 10개 파일 묶음으로 다시 실행했고 `96 passed` 로 녹색이다.
  - `tests/agent/test_heartbeat_proactive.py`
  - `tests/agent/test_heartbeat_service.py`
  - `tests/test_mail_automation.py`
  - `tests/test_calendar_automation.py`
  - `tests/command/test_builtin_mail.py`
  - `tests/command/test_builtin_calendar.py`
  - `tests/session/test_memory_corrections.py`
  - `tests/channels/test_websocket_http_routes.py`
  - `tests/test_automation_results.py`
  - `tests/agent/test_session_manager_history.py`

## 이번에 반영한 주요 작업

### 1. mail/calendar pilot backbone 확장

- `nanobot/automation_results.py` 에 mail draft/send, calendar list/conflict/create 결과 shape 를 추가했다.
- `nanobot/automation/mail.py`, `nanobot/automation/calendar.py` 를 새로 두고 n8n webhook 기반 phase-1 client + session runner 를 정리했다.
- `/mail`, `/calendar` builtin command 와 approval 흐름을 `nanobot/command/builtin.py` 에 연결했다.
- focused pytest 기준으로 mail/calendar automation client 와 command 레이어가 녹색이다.

### 2. session metadata backbone 및 memory correction 경로 보강

- `nanobot/session/continuity.py` 에 `task_summary`, `owner_profile`, `memory_boundary`, `memory_correction`, `action_result` 최소 backbone 을 추가했다.
- `nanobot/session/manager.py` 에 `set_action_result`, `set_approval_summary`, `set_proactive_summary` 계열 helper 를 추가했다.
- `nanobot/session/memory_corrections.py` 와 `nanobot/channels/websocket.py` 를 통해 WebUI/bridged session 입력에서 `기억해`, `잊어`, `이건 기본 선호가 아님`, `이 프로젝트는 끝났어` 형태를 가로채 USER.md / memory/MEMORY.md 에 반영하는 1차 경로를 추가했다.
- 관련 route/session 테스트도 녹색이다.

### 3. proactive heartbeat 와 quiet-hours 라우팅 정리

- `nanobot/heartbeat/proactive.py` 를 추가해 proactive context digest, target 결정, quiet-hours suppression 정책을 분리했다.
- `nanobot/cli/commands.py` 에서 heartbeat 실행/notify 시 proactive context 와 `proactive_summary` metadata 를 기록하도록 연결했다.
- 현재 정책은 다음과 같이 정리됐다.
  - quiet hours 밖에서는 최근 external channel 이 있으면 그 채널을 우선한다.
  - websocket 만 있거나 quiet hours 로 external push 가 막히면 WebUI 쪽에 hold/suppression 상태를 남긴다.
- source test 와 이전 live 검증 기준으로 daytime Telegram 우선, quiet-hours WebUI fallback 이 확인된 상태다.

### 4. WebUI owner-aware summary / action preview 정리

- `webui/src/components/thread/ThreadShell.tsx`, `ThreadComposer.tsx`, `threadStatus.ts`, `ChatList.tsx`, `webui/src/lib/sessionMetadata.ts`, `webui/src/lib/types.ts` 를 보강해 owner-aware summary, current task block, memory correction quick action, mail/calendar action result preview, proactive hold summary 를 표시하도록 정리했다.
- `ThreadComposer` 는 memory correction quick action 클릭 시 draft template 를 주입하는 흐름을 갖게 됐다.
- chat list 와 thread-shell 테스트도 focused run 에 포함해 녹색을 다시 확인했다.

### 5. Direction A backlog 문서 재정리

- `/Volumes/ExtData/Nanobot/docs/todo.md` 에서 우선순위 2 `Telegram WebUI websocket push mirror 검증 및 마무리` 는 이미 동작 확인이 끝난 항목으로 닫았다.
- `docs/assistant-direction-a/00~08`, `workplans/05-1~05-5` 문서 상단에 archive/reference 상태 메모를 넣어 active source of truth 가 section 5 + `docs/execution-backlog/*.md` 임을 명시했다.
- 이 변경은 운영 문서 정합성에는 반영됐지만, 현재 git root 가 `source` 이므로 이번 commit/push 범위에는 포함되지 않는다.

### 6. WebUI UX 재설계 검토 문서 추가

- live WebUI 스냅샷과 실제 화면 확인 기준으로 owner-aware summary / current task / action result / linked session / status block 이 동시에 누적되며 채팅 시작점이 지나치게 아래로 밀리는 문제가 확인됐다.
- 이에 따라 `docs/webui-ux-redesign/README.md`, `docs/webui-ux-redesign/thread-surface-redesign-2026-05-01.md` 를 추가해 대안 비교, dashboard 검토, 추천안, 단계별 작업안을 문서화했다.
- 현재 추천 방향은 `thread 는 chat-first 로 축소`, `owner-wide aggregate 는 dashboard/home 로 이동`, `세부 메타는 drawer`, `action result 는 message 인접 inline card` 의 hybrid 안이다.

## 이번에 확인된 운영 포인트

### 1. `source` 저장소와 바깥 운영 문서 영역을 분리해서 관리해야 한다

- `/Volumes/ExtData/Nanobot/source` 만 git 저장소다.
- `/Volumes/ExtData/Nanobot/docs/status-summary`, `/Volumes/ExtData/Nanobot/docs/todo.md`, `/Volumes/ExtData/Nanobot/docs/assistant-direction-a/**` 는 같은 프로젝트 영역이지만 git commit/push 대상은 아니다.
- 따라서 코드 변경 commit 과 운영 문서 갱신이 동시에 필요할 때는 둘을 같은 작업 묶음으로 설명하되, push 범위에는 차이가 있다는 점을 명시하는 편이 맞다.

### 2. source-tree 검증과 live runtime 상태를 계속 분리해서 적는 편이 안전하다

- 이번 묶음은 source 기준 focused pytest 96개가 녹색이다.
- live runtime 은 gateway/api/llama launchd 상태와 health/bootstrap 응답만 오늘 기준으로 재확인했다.
- proactive Telegram dispatch, quiet-hours WebUI fallback, approval visibility, morning briefing preview 같은 deeper live 흐름은 이번 문서 작성 직전 재실행하지 않았고, 같은 세션 earlier validation 결과를 유지하는 상태다.

### 3. `webui/webui-dev.pid` 는 커밋 대상이 아니다

- 현재 source 워크트리에는 `webui/webui-dev.pid` 가 생겨 있지만 개발 아티팩트 성격이라 commit 에 포함하지 않는 것이 맞다.
- 이후 dev server 사용 시에도 같은 류의 pid/log 파일은 계속 제외하는 편이 안전하다.

## 현재 남아 있는 제약과 리스크

- `/model smart-router` 실제 채널 inbound end-to-end 검증은 여전히 top-priority active item 으로 남아 있다.
- `docs/todo.md` 와 `assistant-direction-a` 재정리 내용은 source repo push 와 별개라, 다른 환경에서 git pull 만으로는 따라오지 않는다.
- mail/calendar automation 은 n8n webhook contract 와 approval/session metadata backbone 까지 정리했지만, live 운영에서 모든 경로를 다시 end-to-end 돌린 것은 아니다.
- memory correction 은 현재 phrase-driven 1차 경로다. 더 복잡한 자연어 보정 요청은 아직 deterministic parser 밖이다.
- source 워크트리에 `docs/chat-commands.md` 도 변경돼 있으므로 help text/HEARTBEAT template 반영 범위를 문서와 실제 사용자 안내에서 함께 볼 필요가 있다.

## 현재 기준 권장 운영 명령

```bash
launchctl list | grep nanobot
curl -fsS http://127.0.0.1:18790/health
curl -fsS http://127.0.0.1:8765/webui/bootstrap

cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest \
  tests/agent/test_heartbeat_proactive.py \
  tests/agent/test_heartbeat_service.py \
  tests/test_mail_automation.py \
  tests/test_calendar_automation.py \
  tests/command/test_builtin_mail.py \
  tests/command/test_builtin_calendar.py \
  tests/session/test_memory_corrections.py \
  tests/channels/test_websocket_http_routes.py \
  tests/test_automation_results.py \
  tests/agent/test_session_manager_history.py -q
```

## 다음 작업 후보

1. `/model smart-router` 를 Telegram 또는 실제 외부 채널 세션에서 다시 end-to-end 검증하고 `docs/todo.md` 1번을 닫을지 판단
2. mail/calendar pilot 경로를 live n8n credential 상태 기준으로 다시 한 번 end-to-end 검증
3. proactive summary 와 owner-aware summary 가 실제 daily operation 에 충분한지 WebUI 실사용 관점에서 정리
4. git root 밖 운영 문서(`docs/todo.md`, `assistant-direction-a/**`)를 어떻게 버전 관리할지 별도 운영 원칙 정리

## 결론

2026-05-01 기준으로 Nanobot 개인비서 backlog 중 핵심 runtime 보강은 source tree 에서 상당 부분 연결됐다. mail/calendar pilot command, session metadata backbone, proactive heartbeat routing, owner-aware WebUI surface 는 focused test 기준으로 다시 녹색이며, live launchd 상태와 health/bootstrap 도 현재 정상이다.

반면 문서 구조 재정리 영역은 `source` 저장소 밖에 있어 git push 와 같은 단위로 관리되지 않는다. 따라서 이번 시점의 운영 해석은 `코드/테스트는 source repo commit+push 대상`, `status-summary와 todo/direction-a 문서는 같은 프로젝트의 운영 메모이지만 별도 파일 영역` 으로 분리해서 보는 것이 맞다.