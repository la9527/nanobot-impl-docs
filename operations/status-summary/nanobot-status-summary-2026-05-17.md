# Nanobot 작업 상태 정리 - 2026-05-17

## 문서 목적

이 문서는 LFM2 llama.cpp cache 경로를 `LLM_Test` 에서 분리하고, non-git `infra` 변경과 live 검증 결과를 다시 추적할 수 있게 남기는 status summary 이다.

## 현재 상태 요약

- LFM2 llama.cpp 기본 cache/model 경로는 이제 `Nanobot/infra/.local-model-cache/llama-cpp-storage` 를 사용한다.
- live launchd service `com.nanobot.local-model-lfm2` 는 새 경로 기준으로 정상 기동 중이다.
- legacy `LLM_Test` 디렉터리는 삭제했고, 현재 Nanobot 운영 경로는 이에 의존하지 않는다.
- `LLMTestTool_v2` 가 최신 운영 저장소이고, 기존 `LLMTestTool` 은 legacy v1 문서/구조 보존용으로 분리했다.

## 이번에 반영한 주요 작업

- non-git `infra/scripts/local-models/start-lfm2-llama-cpp.sh`
  - LFM2 GGUF 기본 model file 과 storage root 를 `Nanobot/infra/.local-model-cache/llama-cpp-storage` 기준으로 전환했다.
- non-git `infra/scripts/local-models/start-local-model-services.sh`
  - `start/restart` 시 `~/.nanobot-launchd-scripts/local-models` 아래의 copied wrapper 와 `common.sh` 를 다시 복사하도록 바꿨다.
- non-git `infra/scripts/local-models/common.sh`
  - legacy launch agent cleanup 시 `com.zeroclaw.llama-cpp.plist` 같은 stale plist 파일도 실제로 삭제하도록 보강했다.
- `source/tests/test_local_model_scripts.py`
  - 새 cache root, wrapper refresh, legacy plist cleanup 을 고정하는 회귀 테스트를 추가했다.
- legacy `LLMTestTool`
  - README 와 migration notes 에서 `LLMTestTool_v2` 를 최신 운영 저장소로 명시하고, v1 저장소는 legacy 로 정리했다.

## 원인 메모

- 초기 cache 경로 전환 뒤 `./local-models.sh restart lfm2` 검증에서 listener 가 올라오지 않았다.
- 직접 원인은 launchd 가 repo 스크립트가 아니라 `~/.nanobot-launchd-scripts/local-models` 아래 copied wrapper 를 계속 실행한 점이었다.
- 따라서 repo 쪽 wrapper 만 수정하면 live restart 는 반영되지 않았고, `start-local-model-services.sh` 가 copied wrapper 를 항상 refresh 하도록 바꾸는 것이 근본 수정이었다.

## 검증 결과

- source focused tests
  - `PYTHONPATH=$PWD ./.venv/bin/pytest tests/test_local_model_scripts.py -q`
  - 결과: `6 passed`
- live local model 검증
  - `./local-models.sh restart lfm2`
  - `./local-models.sh status lfm2`
  - 결과: `listener: up`, `endpoint: ok`
- live endpoint 검증
  - `curl -fsS http://127.0.0.1:1242/v1/models`
  - 결과: `LiquidAI/LFM2-24B-A2B-GGUF:Q4_0` 모델 응답 확인
- gateway health 검증
  - `curl -fsS http://127.0.0.1:18790/health`
  - 결과: `{"status": "ok"}`
- 삭제 검증
  - `/Volumes/ExtData/AI_Project/LLM_Test` 디렉터리 제거 확인
  - 현재 실행 중인 `llama-server` 프로세스가 새 cache 경로를 사용함을 확인

## Git 및 정리 상태

- Nanobot source commit
  - `ef4401c7` `test: guard local-model cache migration`
  - push 완료: `origin/feature/nanobot-fork-runtime`
- legacy `LLMTestTool` commit
  - `13d8b5e` `docs: mark v1 repo as legacy`
  - remote `main` 은 이미 `LLMTestTool_v2` 기준이라 legacy commit 은 `origin/legacy-v1-docs` branch 로 push 했다.

## 남아 있는 참고성 언급

- `source/tests/test_local_model_scripts.py` 의 negative assertion 은 old path 미재도입 회귀 검증용으로 유지한다.
- planning 문서의 `LLM_Test` 언급은 과거 의존성 제거 맥락 설명이며 운영 의존성은 아니다.

## MyOpenClawRepo photo MCP 분리 및 live cutover 검증

### 문서 목적

이 메모는 Nanobot 가 더 이상 `/Volumes/ExtData/MyOpenClawRepo/mcp-servers/*` 아래의 `photo-source`, `photo-ranker`, `apple-terminal-helper` copy 에 의존하지 않고, 새 `/Volumes/ExtData/my-mcp-servers/mcp-my-photos/*` 루트에서 MCP 를 실행한다는 근거를 같은 날짜 summary 에 추가로 남기기 위한 기록이다.

### 현재 상태 요약

- `my-mcp-servers` 는 새 git repo 로 초기화했고 `origin` 은 `https://github.com/la9527/my-mcp-servers.git` 이다.
- `mcp-my-photos` 초기 import commit `3456701` 은 `origin/main` 으로 push 완료했다.
- Nanobot live gateway 는 현재 `photo-source`, `photo-ranker` 를 `/Volumes/ExtData/my-mcp-servers/mcp-my-photos/*` 경로에서 child process 로 실행한다.
- `MyOpenClawRepo` 안의 legacy `mcp-servers/apple-terminal-helper`, `mcp-servers/photo-source`, `mcp-servers/photo-ranker` 는 commit `fd1c19f` (`Remove legacy photo MCP bundle`) 로 제거했고, 해당 commit 도 원격 `main` 에 push 완료했다.

### 이번에 반영한 주요 작업

- non-git `infra/scripts/run-photo-source-mcp.sh`
  - 새 상수 `MY_PHOTOS_MCP_ROOT=/Volumes/ExtData/my-mcp-servers/mcp-my-photos` 기준으로 `photo-source` 를 실행하도록 전환했다.
- non-git `infra/scripts/run-photo-ranker-mcp.sh`
  - 같은 새 루트 기준으로 `photo-ranker` 를 실행하도록 전환했다.
- non-git `infra/scripts/install-nanobot-photo-mcps.sh`
  - dependency install 대상 디렉터리를 `mcp-my-photos/photo-source`, `mcp-my-photos/photo-ranker` 로 변경했다.
- `source/tests/test_local_model_scripts.py`
  - old `MyOpenClawRepo` photo MCP path 가 다시 들어오지 않도록 negative assertion 을 추가했다.
- `docs/planning/execution-backlog/13-nanobot-my-mcp-photos-cutover-validation-phase1.md`
  - push, 회귀 테스트, 재설치, 재기동, health, 실프로세스 경로, old path 검색 순서의 실행 계획을 기록했다.

### 검증 결과

- Nanobot source focused test
  - `PYTHONPATH=$PWD ./.venv/bin/pytest tests/test_local_model_scripts.py -q`
  - 결과: `8 passed`
- 새 MCP 루트 dependency 재설치
  - `cd /Volumes/ExtData/Nanobot/infra/scripts && ./install-nanobot-photo-mcps.sh`
  - 결과: `photo-source`, `photo-ranker` 모두 Python 3.12 기준으로 설치 완료
- live 서비스 재기동 및 health
  - `stop-nanobot-services.sh`
  - `start-nanobot-services.sh`
  - `curl -fsS http://127.0.0.1:18790/health`
  - 결과: `{"status": "ok"}`
- live process path 확인
  - `ps -axo pid,ppid,command | rg '/Volumes/ExtData/my-mcp-servers/mcp-my-photos/(photo-source|photo-ranker)|nanobot gateway --config'`
  - 결과: gateway parent 아래 `uv run --python 3.12 --directory /Volumes/ExtData/my-mcp-servers/mcp-my-photos/photo-source server.py`, `photo-ranker server.py` child process 확인
- copied MCP server test
  - `cd /Volumes/ExtData/my-mcp-servers/mcp-my-photos/photo-source && uv run --python 3.12 --with pytest python -m pytest tests/test_server.py -q`
  - 결과: `12 passed`
  - `cd /Volumes/ExtData/my-mcp-servers/mcp-my-photos/photo-ranker && uv run --python 3.12 --with pytest --with pytest-asyncio python -m pytest tests/test_server.py -q`
  - 결과: `23 passed`
- old path 잔존 검색
  - `rg -n '/Volumes/ExtData/MyOpenClawRepo/mcp-servers/photo-source|/Volumes/ExtData/MyOpenClawRepo/mcp-servers/photo-ranker' /Volumes/ExtData/Nanobot`
  - 결과: 실제 실행 경로 참조는 없고 `source/tests/test_local_model_scripts.py` 의 negative assertion 만 남아 있음

### 운영 포인트

- `apple-terminal-helper/apple_terminal_helper/` 는 `pyproject.toml` 의 `packages = ["apple_terminal_helper"]` 에 맞는 일반적인 Python package layout 이므로 이번 slice 에서 flatten 대상이 아니다.
- source tree 테스트 과정에서 `uv run` 이 `.venv` 를 다시 만들 수 있으므로, 최종 live 확인 전에는 `install-nanobot-photo-mcps.sh` 를 한 번 더 실행해 운영용 `.venv` 를 Python 3.12 기준으로 다시 맞추는 편이 안전하다.
- source repo 와 docs repo, non-git `infra` 변경은 서로 분리돼 있다. 이번 workstream 에서는 `my-mcp-servers` 와 `MyOpenClawRepo` 는 push 완료, `Nanobot/docs` 는 planning/status summary 기록을 별도 repo 변경으로 유지한다.

### 다음 작업 후보

- `docs/planning/execution-backlog/13-nanobot-my-mcp-photos-cutover-validation-phase1.md` 기준으로 남은 운영 리스크를 close 할지, 아니면 status summary 수준에서 종료할지 결정
- `MyOpenClawRepo` 안에 photo MCP 관련 문서/메모 참조가 추가로 남아 있는지 좁게 검색해 후속 정리
- 필요하면 `Nanobot/docs/guide` 계열에 `my-mcp-servers` 운영 루트와 재설치 명령을 더 직접적으로 남기기