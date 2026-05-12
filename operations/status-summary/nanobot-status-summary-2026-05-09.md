# Nanobot 작업 상태 정리 - 2026-05-09

## 문서 목적

이 문서는 2026-05-03 이후 2026-05-09까지 진행한 upstream/main 동기화, `feature/nanobot-fork-runtime` merge, merge 후 회귀 검증, live WebUI bootstrap 운영 보정, 현재 남은 리스크를 정리한 운영 메모다.

## 현재 상태 요약

현재 확인된 상태는 다음과 같다.

- source repo `/Volumes/ExtData/Nanobot/source` 는 현재 `feature/nanobot-fork-runtime` 브랜치 `1027801b` 기준이다.
- source repo working tree 는 clean 상태다.
- Nanobot gateway 는 launchd label `com.nanobot.gateway` 로 실행 중이다.
- Nanobot API 는 launchd label `com.nanobot.api` 로 실행 중이다.
- local llama runtime 은 launchd label `com.nanobot.llama-lfm2-server` 로 실행 중이다.
- live gateway health 는 `http://127.0.0.1:18790/health` 에서 `{"status": "ok"}` 로 확인했다.
- live API health 는 `http://127.0.0.1:8900/health` 에서 `{"status": "ok"}` 로 확인했다.
- live WebUI bootstrap 은 `http://127.0.0.1:8765/webui/bootstrap` 에서 인증 후 200 응답과 model target 목록을 반환하는 것을 확인했다.
- 브라우저에서 `http://127.0.0.1:8765/` 진입 시 bootstrap secret 입력 후 기존 session/sidebar 가 보이는 WebUI shell 로 정상 진입했다.

## 이번에 반영한 주요 작업

### 1. upstream/main 을 local main 에 동기화했다

- 비교 기준은 이전 local `main` `e3bca929` 에서 현재 `upstream/main` 과 맞춘 `451d7408` 까지다.
- 범위는 `211 files changed, 22895 insertions(+), 2344 deletions(-)` 수준의 대형 sync 였다.
- 주요 유입 내용은 image generation tool/provider, ask_user, WebUI turn lifecycle 강화, localized slash commands, session/history trimming 보강, channel thread 정합성 개선, WebUI bootstrap/auth 강화였다.

### 2. `feature/nanobot-fork-runtime` 에 main merge 를 완료했다

- merge 완료 commit 은 `a10fbde3` 이다.
- 이번 merge 는 upstream 의 WebUI lifecycle / auth / image generation / ask_user / history 변경과 local fork 의 smart-router 표시 / continuity / owner-aware summary / action result UI 가 강하게 겹치는 형태였다.
- 실제 충돌은 Python runtime, `nanobot/session/manager.py`, `nanobot/channels/websocket.py`, `nanobot/api/server.py`, `webui/src/App.tsx`, `ThreadShell.tsx`, `ThreadComposer.tsx`, `useNanobotStream.ts`, locale JSON, 테스트 레이어에 집중됐다.

### 3. merge 후 source 회귀를 정리하고 재검증했다

- merge 이후 `nanobot.agent.memory` 와 `nanobot.session.manager` 사이 circular import 를 정리하기 위해 `MemoryStore` import 를 lazy import 로 옮겼다.
- merge 과정에서 drift 가 남은 테스트를 현재 runtime 시그니처에 맞게 보정했다.
- 특히 `tests/agent/test_runner.py` 는 `AgentRunner._execute_tools()` 의 인자와 `_run_agent_loop()` 반환 shape 변경을 따라가도록 수정했고, 이 수정은 `1027801b test: align runner signature regressions` 로 반영됐다.
- source-tree 기준 broader Python regression slice 는 `381 passed` 로 확인했다.
- WebUI production build 와 focused WebUI Vitest 도 통과했다.

### 4. live WebUI bootstrap 경로를 운영 설정까지 포함해 복구했다

- merge 후 처음에는 gateway / API health 는 정상이었지만 `127.0.0.1:8765` WebUI bootstrap 이 뜨지 않았다.
- 원인은 upstream websocket auth 강화로, `channels.websocket.host = 0.0.0.0` 인 환경에서 `token` 또는 `token_issue_secret` 이 없으면 websocket channel 이 비활성화되는 점이었다.
- live config 에 `NANOBOT_WEBUI_BOOTSTRAP_SECRET` 를 추가하고 `~/.nanobot/config.json` 의 `channels.websocket.token_issue_secret` 를 연결했다.
- launchd service 재기동 후 `WebSocket channel enabled`, health 복구, bootstrap 200, 브라우저 WebUI 진입까지 확인했다.

### 5. merge 범위와 리스크를 운영 문서로 남겼다

- `/Volumes/ExtData/Nanobot/docs/archive/upstream-main-sync-2026-05-08/main-sync-summary.md`
- `/Volumes/ExtData/Nanobot/docs/archive/upstream-main-sync-2026-05-08/feature-merge-risk.md`
- 위 문서들에는 upstream/main 유입 범위, feature merge 충돌 축, live bootstrap 운영 보정, 재발 시 먼저 볼 포인트를 정리해 두었다.

## 이번에 확인된 운영 포인트

### 1. source-tree 검증과 live runtime 검증은 분리해서 봐야 한다

- source repo test 와 WebUI build 가 green 이어도 launchd live runtime 은 별도 설정 문제로 실패할 수 있다.
- 이번 사례에서는 source merge 자체보다 `token_issue_secret` 누락이 live WebUI bootstrap 을 막는 핵심 원인이었다.

### 2. health 만으로는 WebUI 정상 여부를 판단하기 어렵다

- `18790` gateway health 와 `8900` API health 가 모두 정상이어도 `8765` WebUI bootstrap 은 깨질 수 있다.
- 앞으로는 최소한 `http://127.0.0.1:8765/webui/bootstrap` 와 실제 브라우저 진입까지 확인하는 편이 안전하다.

### 3. 다음 merge 에서 반복 충돌할 surface 가 분명해졌다

- Python 쪽은 `nanobot/session/manager.py`, `nanobot/agent/memory.py`, `nanobot/agent/loop.py`, `nanobot/channels/websocket.py` 를 먼저 보는 편이 낫다.
- WebUI 쪽은 `App.tsx`, `ThreadShell.tsx`, `ThreadComposer.tsx`, `useNanobotStream.ts`, locale JSON 을 한 묶음으로 보는 편이 낫다.
- 테스트는 source merge marker 제거보다 contract drift 제거가 더 오래 걸릴 수 있다.

### 4. docs 는 운영 메모이지만 현재는 repo 밖에 있다

- `/Volumes/ExtData/Nanobot/docs` 는 현재 git repo 가 아니므로 이번 merge 정리 문서와 status-summary 는 운영 메모로만 유지되고 있다.
- source repo 커밋과 운영 메모 커밋은 현재 분리되어 있다.

## 검증 결과

이번 기간에 확인한 검증은 아래와 같다.

source-tree 기준 merge 후 focused / broader 검증:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_runner.py -q
# merge 후 signature drift 보정 뒤 통과

PYTHONPATH=$PWD ./.venv/bin/pytest <broader regression slice> -q
# 381 passed

cd /Volumes/ExtData/Nanobot/source/webui
npm run build
# 통과

npm test -- <focused WebUI slices>
# 통과
```

live runtime 기준 확인:

```bash
launchctl list | rg 'com\.nanobot\.(gateway|api|llama-lfm2-server)'
curl -fsS http://127.0.0.1:18790/health
curl -fsS http://127.0.0.1:8900/health
curl -fsS -H 'X-Nanobot-Auth: <bootstrap-secret>' http://127.0.0.1:8765/webui/bootstrap
```

확인 결과:

- `com.nanobot.gateway`, `com.nanobot.api`, `com.nanobot.llama-lfm2-server` 모두 list 에 존재
- gateway health 정상
- API health 정상
- bootstrap 응답에서 `active_target = smart-router` 와 model target 목록 확인
- 브라우저에서 인증 후 기존 WebUI shell 진입 확인

## 현재 남아 있는 제약과 리스크

- 이번 merge 는 일단 안정화됐지만, upstream 가 session/history 와 WebUI lifecycle 을 계속 건드리고 있어서 다음 sync 에서 같은 파일들이 다시 충돌할 가능성이 높다.
- WebUI bootstrap 은 현재 secret 기반 인증으로 안정화됐지만, secret 값이 빠지거나 launchd env 로드가 어긋나면 다시 같은 증상이 날 수 있다.
- image generation provider 와 ask_user 는 upstream 기능이 들어왔지만, local 운영 시나리오 기준 end-to-end 사용성은 이번 메모 범위에서 다시 깊게 검증하지 않았다.
- `/Volumes/ExtData/Nanobot/docs` 는 source repo 밖이지만, 현재는 별도 git repo 로 추적되고 있다. 따라서 source commit 과 원자적으로 묶이지는 않더라도 docs commit 으로는 별도 관리할 수 있다.

## 현재 기준 권장 운영 명령

```bash
cd /Volumes/ExtData/Nanobot/source
git status --short --branch

PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_runner.py -q

cd /Volumes/ExtData/Nanobot/source/webui
npm run build

launchctl list | rg 'com\.nanobot\.(gateway|api|llama-lfm2-server)'
curl -fsS http://127.0.0.1:18790/health
curl -fsS http://127.0.0.1:8900/health
curl -fsS -H 'X-Nanobot-Auth: <bootstrap-secret>' http://127.0.0.1:8765/webui/bootstrap
```

## 다음 작업 후보

- WebUI bootstrap secret 을 운영 문서가 아니라 더 체계적인 secret 관리 경로로 옮길지 검토한다.
- 다음 upstream sync 전에 `session/history`, `websocket/bootstrap`, `ThreadShell/Composer`, locale JSON 을 merge hotspot 으로 따로 체크리스트화한다.
- image generation mode 와 ask_user choice 흐름을 local 운영 시나리오에서 한 번 더 end-to-end 로 점검한다.
- docs 를 계속 repo 밖 운영 메모로 둘지, 추적 가능한 경로로 옮길지 정책을 정한다.

## 결론

2026-05-03 이후 작업의 핵심은 기능 추가보다도, upstream 대형 sync 를 local fork 에 무리 없이 흡수하고 live runtime 까지 다시 안정화하는 일이었다. 현재 `feature/nanobot-fork-runtime` 는 source-tree 검증, WebUI build, live gateway/API health, WebUI bootstrap, 브라우저 진입까지 모두 확인된 상태이며, 가장 큰 운영 리스크는 다음 sync 때 반복될 merge hotspot 과 `token_issue_secret` 계열 live config 누락이다.