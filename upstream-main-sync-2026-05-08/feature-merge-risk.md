# feature/nanobot-fork-runtime merge 시 이슈 포인트

## 비교 기준

- merge 이전 remote feature: `origin/feature/nanobot-fork-runtime` at `5dcb41e4`
- merge 대상 main: `451d7408`
- merge 완료 commit: `a10fbde3`
- 반영 규모: `177 files changed, 14133 insertions(+), 1943 deletions(-)`

## 결론 먼저

이번 merge 는 단순 rebase 수준이 아니라, upstream 의 WebUI lifecycle / auth / image generation / ask_user / session-history 변경과 local fork 의 model-target / continuity / owner-aware summary / action result UX 가 정면으로 겹친 merge 였다.

즉, 앞으로 같은 계열 merge 를 다시 할 때도 가장 먼저 `runtime + websocket + App/ThreadShell/Composer + locale + tests` 를 한 묶음으로 봐야 한다.

## 실제 충돌이 집중된 영역

### 1. Python runtime / session 영역

충돌 집중 파일:

- `nanobot/command/builtin.py`
- `nanobot/config/schema.py`
- `nanobot/session/manager.py`
- `nanobot/api/server.py`
- `nanobot/agent/loop.py`
- `nanobot/agent/runner.py`
- `nanobot/agent/tools/filesystem.py`
- `nanobot/agent/tools/shell.py`
- `nanobot/channels/websocket.py`
- `nanobot/channels/telegram.py`

핵심 이유:

- upstream 는 ask_user, image generation, stream / turn lifecycle, SSE/bootstrap, exec safety 를 강화했다.
- local fork 는 continuity metadata, action_result, owner-aware summary, model target, smart-router 연결성을 유지하고 있었다.
- `session/manager.py` 는 memory/history 경로와 local memory store wiring 이 겹쳐 순환 import 위험이 컸다.

실제 발생한 문제:

- `nanobot.agent.memory` 와 `nanobot.session.manager` 사이 circular import
- exec security test 의 에러 문구 길이 변경으로 assertion mismatch

실제 조치:

- `MemoryStore` import 를 `SessionManager.__init__` 내부 lazy import 로 이동
- guard error assertion 을 exact match 가 아니라 substring 기반으로 완화

### 2. WebUI 핵심 surface

충돌 집중 파일:

- `webui/src/App.tsx`
- `webui/src/components/thread/ThreadComposer.tsx`
- `webui/src/components/thread/ThreadShell.tsx`
- `webui/src/components/thread/ThreadHeader.tsx`
- `webui/src/components/settings/SettingsView.tsx`
- `webui/src/components/MessageBubble.tsx`
- `webui/src/hooks/useNanobotStream.ts`
- `webui/src/hooks/useSessions.ts`
- `webui/src/lib/api.ts`

충돌 이유:

- upstream 가 추가한 것:
  - saved-secret bootstrap / stricter auth refresh
  - ask_user choice rendering
  - slash command UX
  - image generation mode
  - turn_end / session_updated lifecycle
  - blank-home dashboard / quick action 흐름
- local fork 가 유지하려 한 것:
  - active model target / model label / smart-router 표현
  - continuity / owner-aware summary / memory correction / action result 카드
  - Telegram bridge / linked external session summary

실제 merge 시 주의해야 했던 포인트:

- `ThreadComposer` 의 `onSend` 시그니처가 3번째 options 인자를 가지는 구조와 테스트가 어긋나기 쉬움
- `ThreadShell` blank home 상태가 기존 hero composer 전제에서 dashboard landing 으로 바뀜
- `useNanobotStream` 은 `stream_end` 와 `turn_end` 를 분리해서 처리하므로, 완료 상태 test 나 custom UI 가 여기 맞춰져야 함
- header / settings button 의 accessible name 이 `Settings` 에서 `Open settings` 류로 바뀌어 test 회귀가 자주 남

### 3. i18n locale JSON

충돌 파일:

- `webui/src/i18n/locales/en/common.json`
- `webui/src/i18n/locales/es/common.json`
- `webui/src/i18n/locales/fr/common.json`
- `webui/src/i18n/locales/id/common.json`
- `webui/src/i18n/locales/ja/common.json`
- `webui/src/i18n/locales/ko/common.json`
- `webui/src/i18n/locales/vi/common.json`
- `webui/src/i18n/locales/zh-CN/common.json`
- `webui/src/i18n/locales/zh-TW/common.json`

이슈 포인트:

- upstream 가 stop / copy reply / navigation / mobile sheet / quick actions / image quick actions key 를 추가함
- local fork 는 model menu, approval, continuity 관련 key 를 이미 들고 있었음
- JSON conflict 를 대충 해결하면 build 는 지나가도 실제 UI 에서 key miss 가 늦게 드러날 수 있음

실제 조치:

- 기존 번역 style 을 유지하면서 feature-side key 와 upstream key 를 둘 다 보존
- ICU 문법 같은 새로운 포맷은 도입하지 않음

### 4. 테스트 레이어

충돌 또는 후속 수정이 컸던 파일:

- `tests/agent/test_loop_save_turn.py`
- `tests/agent/test_session_manager_history.py`
- `tests/channels/test_websocket_channel.py`
- `tests/cli/test_commands.py`
- `tests/tools/test_exec_security.py`
- `webui/src/tests/app-layout.test.tsx`
- `webui/src/tests/thread-composer.test.tsx`
- `webui/src/tests/thread-shell.test.tsx`
- `webui/src/tests/useNanobotStream.test.tsx`

이슈 포인트:

- source merge marker 제거만으로는 끝나지 않고, test contract 자체가 바뀌어 expectation drift 가 남았다.
- blank-home, settings aria-label, quick action flow, bridged ask_user, turn_end 완료 상태 같은 부분이 대표적이다.
- `useNanobotStream.test.tsx` 는 merge 과정에서 구조가 깨지기 쉬운 파일이어서 syntax-level 손상이 실제로 남았다.

## 이번 merge 에서 실제로 확인된 주요 리스크

### 리스크 1. session/history 계층은 다음 merge 때 다시 충돌할 가능성이 높다

근거:

- upstream 가 replay/file-cap/history trimming/memory write 경로를 계속 건드리고 있다.
- local fork 도 continuity / owner profile / memory correction 을 session metadata 와 강하게 묶고 있다.

예상 재발 포인트:

- `nanobot/session/manager.py`
- `nanobot/agent/memory.py`
- `tests/agent/test_session_manager_history.py`

### 리스크 2. WebUI 는 `ThreadShell` 중심으로 반복 충돌이 날 가능성이 매우 높다

근거:

- blank-home, status rail, action result, model target, linked session, quick actions, composer wiring 이 모두 `ThreadShell` 에 모여 있다.
- upstream 와 local fork 가 같은 surface 를 다른 목적에서 동시에 수정하고 있다.

예상 재발 포인트:

- `webui/src/components/thread/ThreadShell.tsx`
- `webui/src/components/thread/ThreadComposer.tsx`
- `webui/src/hooks/useNanobotStream.ts`
- `webui/src/App.tsx`

### 리스크 3. test 가 green 이어도 live runtime 확인이 별도로 필요하다

이번 merge 에서 확인한 것:

- focused Python pytest: 통과
- WebUI `npm run build`: 통과
- focused WebUI vitest: 통과

아직 별도로 봐야 하는 것:

- launchd live runtime 과 실제 gateway 반영
- WebUI bootstrap 실서비스 경로
- image generation provider 실제 설정값
- LAN bootstrap / `token_issue_secret` 운영 설정

## merge 후 유지한 로컬 동작

이번 merge 에서 유지하려고 한 local fork 핵심은 아래다.

- model target 선택 및 smart-router 관련 표시
- continuity / owner-aware summary
- action_result / approval / memory correction UI
- linked Telegram session bridge 흐름
- automation / task summary 와 WebUI 상태 블록

동시에 보존한 upstream 핵심은 아래다.

- ask_user choices
- `/history`
- image generation mode
- saved-secret bootstrap / reauth refresh
- `stream_end` / `turn_end` / `session_updated` lifecycle
- blank-home dashboard / quick actions

## 다음 merge 때 권장 순서

1. 먼저 `main` sync 범위를 commit subject 기준으로 기능군 분류한다.
2. `session/manager.py`, `agent/memory.py`, `agent/loop.py` 를 먼저 resolve 한다.
3. `channels/websocket.py`, `api/server.py` 로 streaming/bootstrap 경로를 맞춘다.
4. WebUI 는 `App.tsx` -> `ThreadShell.tsx` -> `ThreadComposer.tsx` -> `useNanobotStream.ts` 순으로 resolve 한다.
5. locale JSON 을 source resolve 직후 같이 맞춘다.
6. 테스트는 Python focused -> WebUI build -> WebUI focused vitest 순으로 바로 검증한다.

## 이번 merge 기준 최소 검증 세트

실제로 유효했던 검증 순서:

1. focused Python pytest
2. `webui/npm run build`
3. focused WebUI vitest

이 순서를 유지하면 conflict 제거 후 실제 회귀를 가장 빨리 드러낼 수 있다.

## 2026-05-09 운영 후속 확인 결과

merge 이후 실제 launchd live runtime 에서 추가로 확인한 내용은 아래와 같다.

### 1. WebSocket/WebUI bootstrap 는 운영 config 보정이 필요했다

실제 현상:

- gateway / API health 는 정상이었지만 WebUI bootstrap 포트 `127.0.0.1:8765` 가 열리지 않았다.
- gateway stderr 로그에는 아래 경고가 남았다.
  - `host is 0.0.0.0 (all interfaces) but neither token nor token_issue_secret is set`

원인:

- upstream runtime 은 `websocket.host = 0.0.0.0` 인 경우 `token` 또는 `token_issue_secret` 중 하나가 반드시 있어야 websocket channel 을 enable 한다.
- 기존 live config 는 `channels.websocket.host = 0.0.0.0` 만 설정되어 있었고 인증 설정이 비어 있었다.

실제 조치:

- `~/.nanobot/nanobot.env` 에 `NANOBOT_WEBUI_BOOTSTRAP_SECRET` 추가
- `~/.nanobot/config.json` 의 `channels.websocket.token_issue_secret` 를 `${NANOBOT_WEBUI_BOOTSTRAP_SECRET}` 로 연결
- launchd service 재기동

결과:

- gateway 로그에 `WebSocket channel enabled` 확인
- `http://127.0.0.1:18790/health` 정상
- `http://127.0.0.1:8900/health` 정상
- `http://127.0.0.1:8765/webui/bootstrap` 가 secret header 와 함께 200 응답
- 브라우저에서 `http://127.0.0.1:8765/` 접속 시 secret 입력 화면이 먼저 나타났고, secret 저장 후 기존 session/sidebar 가 보이는 WebUI shell 로 정상 진입 확인

### 2. source 회귀와 live 운영 검증은 둘 다 통과했다

추가로 2026-05-09 시점에 아래를 다시 확인했다.

- broader Python regression slice: `381 passed`
- WebUI static root `http://127.0.0.1:8765/` GET 200
- WebUI bootstrap JSON 에서 `active_target = smart-router`, model target 목록 응답 확인

### 3. 다음에도 같은 증상이 보이면 먼저 볼 것

- health 는 green 인데 WebUI 가 안 열리면 먼저 websocket channel enablement 를 확인한다.
- 특히 `0.0.0.0` 바인딩 환경에서는 `token_issue_secret` 누락 여부를 먼저 본다.
- bundle 문제를 의심하기 전에 `/tmp/nanobot-gateway.err.log` 와 `~/.nanobot/config.json` 의 websocket auth 필드를 확인하는 편이 더 빠르다.
