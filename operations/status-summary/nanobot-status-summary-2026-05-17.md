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