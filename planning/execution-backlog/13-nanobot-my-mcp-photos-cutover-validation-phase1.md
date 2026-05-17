# 13 Nanobot My MCP Photos Cutover Validation Phase 1

## 1. 문서 목적

이 문서는 Nanobot 가 기존 `/Volumes/ExtData/MyOpenClawRepo/mcp-servers/*` 의존성 대신 `/Volumes/ExtData/my-mcp-servers/mcp-my-photos/*` 아래의 `photo-source`, `photo-ranker`, `apple-terminal-helper` 묶음을 실제 운영 경로로 사용하는지 검증하고, 남은 후속 정리 작업을 phase-1 범위로 고정하기 위한 실행 문서다.

이번 단계의 목표는 두 가지다.

- Nanobot live gateway 가 새 MCP 루트에서 `photo-source`, `photo-ranker` 를 안정적으로 기동하고 상태를 유지하는지 확인하기
- 이후 `MyOpenClawRepo` 쪽 legacy photo MCP 디렉터리를 정리해도 되는 판단 기준과 후속 작업 순서를 문서로 고정하기

## 2. 왜 지금 필요한가

현재 `photo-source`, `photo-ranker`, `apple-terminal-helper` 는 이미 `/Volumes/ExtData/my-mcp-servers/mcp-my-photos/` 아래로 복사됐고, Nanobot wrapper 도 새 경로를 보도록 전환됐다.

하지만 이번 cutover 는 단순 파일 복사만으로 끝나는 작업이 아니다. 운영 기준으로는 아래 항목이 함께 확인돼야 한다.

1. launchd 기반 gateway 재기동 후에도 MCP child process 가 새 경로에서 다시 떠야 한다.
2. `apple-terminal-helper` sibling 상대 경로 의존 때문에 세 디렉터리 구조가 깨지지 않아야 한다.
3. `install-nanobot-photo-mcps.sh` 가 새 경로에 `.venv` 를 만들고 dependency 를 정상 해석해야 한다.
4. Nanobot config 와 wrapper 가 새 경로를 source of truth 로 삼아야 한다.
5. `MyOpenClawRepo` 쪽 legacy photo MCP copy 를 바로 지웠을 때 어떤 검증이 더 필요한지 기준이 있어야 한다.

즉, 이번 slice 는 “복사 성공” 이 아니라 “Nanobot 운영 경로 cutover 가 안정화됐는지” 를 확인하는 validation slice 로 보는 것이 맞다.

## 3. phase-1 범위

포함할 것:

- `run-photo-source-mcp.sh`, `run-photo-ranker-mcp.sh`, `install-nanobot-photo-mcps.sh` 의 새 경로 기준 검증
- gateway 재기동 뒤 실제 child process 경로 재확인
- `photo-source`, `photo-ranker` 기본 실행 및 프로세스 생존 확인
- `apple-terminal-helper` 가 sibling path dependency 로 정상 해석되는지 확인
- planning/status-summary 수준의 운영 문서화
- legacy `MyOpenClawRepo/mcp-servers/*` 삭제 전 확인 항목 정리

제외할 것:

- `photo-source`, `photo-ranker` 내부 feature 자체의 대규모 리팩터링
- review UI 재설계
- photos-classify OpenClaw plugin 정리
- `mcp-my-photos` 를 즉시 별도 git remote/repo 로 productize 하는 작업

## 4. 현재 코드 anchor

직접 영향을 받는 주요 위치는 아래다.

- `infra/scripts/run-photo-source-mcp.sh`
- `infra/scripts/run-photo-ranker-mcp.sh`
- `infra/scripts/install-nanobot-photo-mcps.sh`
- `infra/nanobot/config.template.json`
- `~/.nanobot/config.json`
- `infra/nanobot/workspace/memory/MEMORY.md`
- `source/tests/test_local_model_scripts.py`
- `/Volumes/ExtData/my-mcp-servers/mcp-my-photos/photo-source/`
- `/Volumes/ExtData/my-mcp-servers/mcp-my-photos/photo-ranker/`
- `/Volumes/ExtData/my-mcp-servers/mcp-my-photos/apple-terminal-helper/`

현재 확인된 사실:

- live gateway child process 는 이미 새 루트에서 `photo-source` 와 `photo-ranker` 를 띄우고 있다.
- `apple-terminal-helper/pyproject.toml` 의 `packages = ["apple_terminal_helper"]` 와 package directory `apple_terminal_helper/` 는 일반적인 Python package layout 이므로 phase-1 에서 flatten 대상이 아니다.
- `my-mcp-servers` 는 2026-05-17 기준 새 git repo 로 초기화됐고 `origin` 은 `https://github.com/la9527/my-mcp-servers.git` 로 설정됐다.

## 5. 핵심 설계 결정

### 5.1 구조는 유지하고, sibling dependency 를 깨지 않는다

`photo-source`, `photo-ranker`, `apple-terminal-helper` 는 현재 상대 경로 의존을 전제로 한다.

따라서 phase-1 에서는 아래 구조를 그대로 유지한다.

- `mcp-my-photos/photo-source`
- `mcp-my-photos/photo-ranker`
- `mcp-my-photos/apple-terminal-helper`

이 구조를 유지하면 `pyproject.toml`, `uv.lock`, local path dependency 를 대규모로 손대지 않고도 운영 cutover 를 검증할 수 있다.

### 5.2 검증 대상은 “기능”보다 “운영 실행 경로” 다

이번 문서에서 우선 보는 것은 아래다.

- gateway restart 후 새 경로 child process 재생성
- `.venv` 재설치 가능 여부
- wrapper 와 installer 가 더 이상 `MyOpenClawRepo` 를 가리키지 않는지
- gateway health 를 깨지 않는지

`photo-source` 도구 출력 품질이나 `photo-ranker` 분류 성능은 다음 단계에서 별도 slice 로 볼 수 있다.

### 5.3 legacy copy 삭제는 validation 기준을 통과한 뒤 판단한다

현재 `MyOpenClawRepo` 안의 old `mcp-servers/photo-source`, `photo-ranker`, `apple-terminal-helper` 는 Nanobot live runtime 기준 직접 의존성이 끊긴 상태로 본다.

다만 삭제 전에는 아래를 다시 확인한다.

1. live process path 가 모두 새 루트인지
2. wrapper/install script 와 config template 에 old path 가 남지 않았는지
3. `~/.nanobot/config.json` 의 MCP command 가 wrapper 기준으로 유지되는지
4. gateway 재기동 후에도 새 루트 `.venv` 로 동일하게 살아나는지

## 6. 작업 분해

### Step 1. 새 루트 기준 wrapper/install/config 정합성 확인

- `run-photo-source-mcp.sh` / `run-photo-ranker-mcp.sh` 가 새 루트 상수를 사용하고 있는지 확인
- `install-nanobot-photo-mcps.sh` 가 새 루트에 `.venv` 를 설치하는지 확인
- config template 과 runtime memory 문구가 새 루트를 가리키는지 확인

### Step 2. live gateway 재기동 후 child process 경로 검증

- launchd 기준 gateway 재기동
- `photo-source`, `photo-ranker` child process 의 실제 실행 경로 확인
- 부모가 `nanobot gateway --config ~/.nanobot/config.json` 인지 확인

### Step 3. 기본 health 및 MCP bootstrap 확인

- gateway health
- 새 루트 `.venv` 생성 여부
- `uv run --directory <new-root>/photo-source server.py`
- `uv run --directory <new-root>/photo-ranker server.py`

### Step 4. legacy path 제거 readiness 확인

- Nanobot 코드/설정에서 `MyOpenClawRepo/mcp-servers/photo-*` 참조가 남아 있는지 재검색
- negative regression test 로 old path 재도입 방지
- 필요하면 status summary 에 현재 cutover 상태를 남김

## 7. 실제 backlog

### 7.1 wrapper and installer slice

- wrapper 상수화 유지
- installer 의 새 root bootstrap 유지
- local-model style regression test 계속 유지

### 7.2 live verification slice

- `stop-nanobot-services.sh` / `start-nanobot-services.sh`
- `curl -fsS http://127.0.0.1:18790/health`
- `ps aux | rg '/Volumes/ExtData/my-mcp-servers/mcp-my-photos'`

### 7.3 deletion readiness slice

- old path grep
- `MyOpenClawRepo` 쪽 legacy photo MCP 디렉터리 삭제 여부 판단
- `my-mcp-servers` 의 git 관리 전략은 별도 후속 slice 로 분리

## 8. 검증 계획

권장 검증 순서는 아래다.

1. `PYTHONPATH=$PWD ./.venv/bin/pytest tests/test_local_model_scripts.py -q`
2. `cd /Volumes/ExtData/Nanobot/infra/scripts && ./install-nanobot-photo-mcps.sh`
3. `/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh`
4. `/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh`
5. `curl -fsS http://127.0.0.1:18790/health`
6. `ps aux | rg '/Volumes/ExtData/my-mcp-servers/mcp-my-photos/(photo-source|photo-ranker)'`
7. `rg -n '/Volumes/ExtData/MyOpenClawRepo/mcp-servers/photo-source|/Volumes/ExtData/MyOpenClawRepo/mcp-servers/photo-ranker' /Volumes/ExtData/Nanobot`

## 8.1 2026-05-17 실행 체크리스트

오늘 즉시 수행할 실제 실행 순서는 아래로 고정한다.

1. `cd /Volumes/ExtData/my-mcp-servers && git push -u origin main`
2. `cd /Volumes/ExtData/Nanobot/source && PYTHONPATH=$PWD ./.venv/bin/pytest tests/test_local_model_scripts.py -q`
3. `cd /Volumes/ExtData/Nanobot/infra/scripts && ./install-nanobot-photo-mcps.sh`
4. `/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh`
5. `/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh`
6. `curl -fsS http://127.0.0.1:18790/health`
7. `ps aux | rg '/Volumes/ExtData/my-mcp-servers/mcp-my-photos/(photo-source|photo-ranker)'`
8. `rg -n '/Volumes/ExtData/MyOpenClawRepo/mcp-servers/photo-source|/Volumes/ExtData/MyOpenClawRepo/mcp-servers/photo-ranker' /Volumes/ExtData/Nanobot`

이 체크리스트는 `push`, `회귀 테스트`, `재설치`, `재기동`, `health`, `실프로세스 경로`, `old path 잔존 여부` 순으로 검증해 “새 경로 MCP 가 live 에서 문제없이 동작하는가” 를 가장 짧게 판정하기 위한 순서다.

## 9. 완료 기준

1. Nanobot live gateway 가 새 `mcp-my-photos` 루트에서 `photo-source`, `photo-ranker` 를 안정적으로 기동한다.
2. `apple-terminal-helper` sibling dependency 때문에 추가 재배치가 필요하지 않다는 판단이 확인된다.
3. wrapper/install/config/memory 기준 old `MyOpenClawRepo/mcp-servers/photo-*` 직접 의존성이 끊긴다.
4. legacy photo MCP copy 삭제 여부를 결정할 수 있을 만큼 운영 근거가 모인다.