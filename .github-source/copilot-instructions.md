# Nanobot Workspace Instructions

이 저장소에서 작업하는 에이전트는 아래 원칙을 기본값으로 따른다.

## 목적

- Nanobot 의 현재 로컬 운영 환경과 source tree 작업 흐름을 서로 어긋나지 않게 유지한다.
- 코드 수정, WebUI 빌드, launchd 재기동, live 검증을 한 흐름으로 다룬다.
- source foreground 검증과 실제 운영 runtime 을 구분해서 다룬다.

## 현재 기준 실행 환경

- 실제 source tree 기준 작업 경로는 `/Volumes/ExtData/Nanobot/source` 이다.
- 현재 사용자 실행 `nanobot` 엔트리포인트는 `~/.local/bin/nanobot` 이고, `uv tool` 기반 editable install 경로를 통해 source tree 를 참조한다.
- live 설정 파일은 `~/.nanobot/config.json`, `~/.nanobot/config.api.json`, `~/.nanobot/nanobot.env` 이다.
- API 키, channel token, WebUI 로그인 password 입력값, WebUI/bootstrap secret 같은 실운영 credential 은 일반 shell init 파일보다 `~/.nanobot/nanobot.env` 를 우선 기준으로 확인한다.
- 특히 WebUI 로그인 화면에서 password 를 입력해야 할 때는 shell init 파일이나 추측값이 아니라 `~/.nanobot/nanobot.env` 의 관련 값(예: `NANOBOT_WEBUI_BOOTSTRAP_SECRET`)을 먼저 확인한다.
- `LOCAL_LLM_MODEL` 같은 핵심 환경변수는 일반 shell init 파일이 아니라 `~/.nanobot/nanobot.env` 에서 공급된다.
- WebUI 소스는 `/Volumes/ExtData/Nanobot/source/webui` 이고, production bundle 출력 경로는 `/Volumes/ExtData/Nanobot/source/nanobot/web/dist` 이다.

## 현재 운영 서비스 기준

- Nanobot gateway 는 launchd label `com.nanobot.gateway` 로 운영한다.
- Nanobot API 는 launchd label `com.nanobot.api` 로 운영한다.
- local model 런타임은 launchd label `com.nanobot.local-model-lfm2`, `com.nanobot.local-model-qwen36` 기준 운영 경로를 우선 확인한다.
- WebUI 브라우저 확인은 `http://127.0.0.1:8765/` 또는 실제 브라우저 새로고침 기준으로 본다.
- WebUI bootstrap 확인 endpoint 는 `http://127.0.0.1:8765/webui/bootstrap` 이고, 최근 운영 기준으로는 `X-Nanobot-Auth` 또는 실제 브라우저 세션이 필요할 수 있다.
- gateway health 확인은 `http://127.0.0.1:18790/health` 기준으로 본다.
- API health 는 현재 운영 설정에 맞는 포트를 확인하되, 최근 운영 기준은 `8900` 계열이다.

## 작업 원칙

- 문서는 한글 우선으로 쓴다. 기술 식별자, 경로, 명령, 환경변수명은 영어 원문을 유지해도 된다.
- 사용자가 구현을 요청하면 설명만 하지 말고 가능한 범위에서 실제 코드, 테스트, 빌드, 재기동, 검증까지 이어서 수행한다.
- 변경은 최소 범위로 유지한다.
- source repo 수정과 live config 변경은 구분해서 다룬다.
- `/Volumes/ExtData/Nanobot/source` 와 `/Volumes/ExtData/Nanobot/docs` 는 서로 다른 git repo 로 다루고, 루트 `/Volumes/ExtData/Nanobot` 는 git 기준 경로로 가정하지 않는다.
- `~/.nanobot/config*.json`, `~/.nanobot/nanobot.env` 는 실운영 파일이므로 사용자가 명시적으로 요청하지 않는 한 덮어쓰거나 초기화하지 않는다.

## WebUI 작업 가이드

- WebUI 수정 후에는 `/Volumes/ExtData/Nanobot/source/webui` 에서 `npm run build` 를 실행해 bundle 을 갱신한다.
- live 반영은 Vite dev server 가 아니라 bundle + gateway 재기동 기준으로 처리한다.
- 사용자가 명시적으로 요청하지 않는 한 `vite dev` 를 운영 반영 수단으로 간주하지 않는다.
- WebUI 테스트는 가능하면 touched slice 기준으로 좁게 실행한다.

권장 명령 예시:

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-composer.test.tsx
npm run build
```

## Backend 작업 가이드

- Python 검증은 source tree 기준으로 수행한다.
- 필요 시 `PYTHONPATH=/Volumes/ExtData/Nanobot/source` 를 명시해 import 경로를 고정한다.
- source venv 기준 검증과 launchd live runtime 검증을 혼동하지 않는다.

권장 명령 예시:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/channels/test_websocket_http_routes.py -q
```

## 재기동 원칙

- live 서비스 재기동은 직접 `nanobot gateway` 를 foreground 로 띄우는 방식보다 launchd 스크립트 경로를 우선한다.
- 이유: `~/.nanobot-launchd-scripts/start-nanobot-gateway.sh` 와 `start-nanobot-api.sh` 는 `~/.nanobot/nanobot.env` 를 source 해서 live 환경변수를 함께 로드한다.
- 수동으로 직접 `nanobot gateway --config ...` 를 실행하면 `LOCAL_LLM_MODEL` 같은 환경변수가 빠져 운영 상태와 다른 결과가 날 수 있다.

권장 재기동 명령:

```bash
/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
```

## 검증 기준

- gateway 재기동 후에는 최소한 `curl -fsS http://127.0.0.1:18790/health` 를 확인한다.
- WebUI 관련 변경 후에는 authenticated `GET /webui/bootstrap` 또는 실제 브라우저 새로고침으로 bundle 반영 여부를 확인한다.
- launchd 상태가 중요하면 `launchctl list | grep nanobot` 로 `com.nanobot.gateway`, `com.nanobot.api` 를 확인한다.
- 필요 시 현재 실행 프로세스와 로그를 함께 확인한다.

권장 명령 예시:

```bash
launchctl list | grep nanobot
ps aux | grep nanobot | grep -v grep
curl -fsS http://127.0.0.1:18790/health
curl -fsS -H 'X-Nanobot-Auth: <bootstrap-secret>' http://127.0.0.1:8765/webui/bootstrap
```

## 주의사항

- source foreground 검증 결과를 곧바로 live 운영 상태로 간주하지 않는다.
- launchd live runtime 과 수동 foreground 프로세스가 동시에 떠서 포트 충돌을 만들지 않게 주의한다.
- `.github` 아래 instruction, skill, 문서는 실제 운영 경로와 모순되지 않게 유지한다.
- smart-router, plugin, gateway, launchd, editable install 동작이 바뀌면 필요 시 `/Volumes/ExtData/Nanobot/docs/operations/status-summary/` 아래 운영 메모도 함께 갱신한다.

## Superpowers 연동

- `.github/skills/` 아래 Superpowers 관련 skill 은 upstream `obra/superpowers` 원문을 vendoring 한 복사본으로 취급한다.
- 사용자가 `superpowers`, `workflow`, `brainstorm`, `design`, `spec`, `plan`, `TDD`, `debug`, `root cause`, `verify` 같은 표현을 쓰거나 구조화된 진행을 원하면 먼저 `.github/skills/using-superpowers/SKILL.md` 를 기준 진입점으로 본다.
- vendored Superpowers skill 내용과 Nanobot 운영 제약이 충돌하면 이 문서의 규칙이 우선한다.

## Impeccable 디자인 skill 연동

- `.github/skills/impeccable/` 은 upstream `pbakaus/impeccable` 의 `.github/skills/impeccable` 을 vendoring 한 복사본으로 취급한다.
- 사용자가 UI/UX 디자인, redesign, critique, audit, polish, layout, typography, color, motion, responsive UI, UX writing 관련 작업을 요청하면 `.github/skills/impeccable/SKILL.md` 를 먼저 확인한다.
- vendored Impeccable skill 내용과 Nanobot 운영 제약이 충돌하면 이 문서의 규칙이 우선한다.

## 우선순위 가이드

- 1순위: 현재 launchd 기반 live 서비스와 충돌하지 않는 변경
- 2순위: source 수정, WebUI bundle, 재기동, live 검증의 일관성 유지
- 3순위: 운영 config 와 문서의 일치 유지
- 4순위: foreground 테스트와 live runtime 결과를 명확히 분리
