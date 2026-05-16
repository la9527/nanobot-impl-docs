# Nanobot 작업 상태 정리 - 2026-05-15

## 문서 목적

이 문서는 2026-05-15 기준으로 Nanobot의 기존 `infra/scripts/llama/` 단일 경로를 `infra/scripts/local-models/` 공용 계층으로 재정리하고, `mlx-community/Qwen3.6-35B-A3B-4bit` 를 Nanobot `infra` 아래 cache 경로에 실제 설치해 launchd + raw benchmark 까지 검증한 결과를 운영 메모 형태로 남기기 위한 문서다.

## 현재 상태 요약

현재 확인된 상태는 다음과 같다.

- local model control entrypoint 는 `infra/scripts/local-models/local-models.sh` 기준으로 정리했다.
- 기존 `infra/scripts/llama/` 경로는 제거했고, local model 제어는 `local-models` 경로만 source of truth 로 둔다.
- launchd label 은 현재 `com.nanobot.local-model-lfm2`, `com.nanobot.local-model-qwen36` 기준으로 올라와 있다.
- LFM2 `llama.cpp` endpoint 는 `http://127.0.0.1:1242/v1` 에서 `ok` 로 확인했다.
- Qwen3.6 MLX endpoint 는 `http://127.0.0.1:1246/v1` 에서 `ok` 로 확인했다.
- Qwen3.6 model cache 는 `infra/.local-model-cache/huggingface/hub/models--mlx-community--Qwen3.6-35B-A3B-4bit` 아래에 실제 생성됐다.
- Nanobot infra cache/result 경로는 root `.gitignore` 로 non-tracked 처리했다.
- focused pytest 는 `source/tests/test_local_model_scripts.py` 1개 기준으로 통과했다.
- Qwen3.6 raw smoke 와 1회 benchmark 까지 성공했고, benchmark 결과는 `infra/.local-model-results/` 아래에 저장됐다.
- 이후 `start all`, `install all` 은 금지했고, 두 모델은 순차 benchmark 로만 검증했다.

## 이번에 반영한 주요 작업

### 1. local-models 공용 control layer 를 추가했다

- `infra/scripts/local-models/local-models.sh` 를 새 통합 entrypoint 로 추가했다.
- 공통 helper `common.sh` 에 model metadata, launchd plist 경로, cache root, MLX runtime bootstrap 함수를 모았다.
- `install`, `start`, `status`, `stop`, `uninstall`, `log`, `smoke`, `benchmark` 명령을 같은 UX로 제공하도록 정리했다.

### 2. LFM2 와 Qwen3.6 을 같은 control surface 아래에 묶었다

- LFM2 는 `start-lfm2-llama-cpp.sh` 로 분리해 기존 `llama.cpp` 흐름을 유지했다.
- Qwen3.6 은 `start-qwen36-mlx.sh` 를 추가해 `mlx_lm.server` 기반 OpenAI-compatible endpoint 로 띄우도록 했다.
- launchd plist 는 `com.nanobot.local-model-lfm2.plist`, `com.nanobot.local-model-qwen36.plist` 로 분리했다.

### 3. MLX runtime bootstrap 과 cache 경로를 Nanobot infra 기준으로 고정했다

- `common.sh` 에서 `LOCAL_MODEL_CACHE_ROOT` 기본값을 `/Volumes/ExtData/Nanobot/infra/.local-model-cache` 로 잡았다.
- Qwen3.6 start script 는 필요 시 `uv venv` + `uv pip install mlx-lm` 으로 local MLX runtime 을 bootstrap 하도록 했다.
- 실제 설치 후 `infra/.local-model-cache` 사용량은 약 `19G` 로 확인됐다.

### 4. LLMTestTool 스타일의 focused test 경로를 Nanobot 쪽에 추가했다

- `run-local-model-smoke.sh` 로 raw `/v1/chat/completions` 단일 요청 smoke 를 추가했다.
- `run-local-model-benchmark.sh` 로 count 기반 raw benchmark 결과를 `summary.json` + `results.jsonl` 로 저장하도록 했다.
- Qwen3.6 기준 1회 benchmark 결과는 `successCount=1`, `avgLatencyMs=269` 로 확인됐다.

### 5. legacy llama 경로는 제거하고 local-models 로 단일화했다

- `infra/scripts/llama/` 경로는 더 이상 source of truth 가 아니라고 판단해 제거했다.
- LFM2 와 Qwen3.6 모두 `infra/scripts/local-models/` 아래 스크립트만 유지한다.
- legacy plist `infra/launchd/com.nanobot.llama-lfm2-server.plist` 도 제거했다.

### 6. 두 모델 동시 기동 기본값은 막았다

- 메모리 보호를 위해 `local-models.sh start all` 과 `local-models.sh install all` 을 명시적으로 금지했다.
- `all` 대상은 `status`, `log`, `stop`, `uninstall` 같이 동시에 켜지지 않는 동작에만 남겼다.
- 보조적으로 `local-models.sh list` 명령을 추가해 현재 지원 target 을 빠르게 확인할 수 있게 했다.

### 7. 순차 benchmark 로 두 모델을 다시 검증했다

- 메모리 보호 정책에 맞춰 `stop all` 후 Qwen3.6 과 LFM2 를 하나씩만 기동해 benchmark 했다.
- Qwen3.6 결과는 `successCount=3`, `failureCount=0`, `avgLatencyMs=932.67` 였고 결과 파일은 `infra/.local-model-results/qwen36/2026-05-15_032043/summary.json` 이다.
- LFM2 결과는 `successCount=3`, `failureCount=0`, `avgLatencyMs=204.67` 였고 결과 파일은 `infra/.local-model-results/lfm2/2026-05-15_032104/summary.json` 이다.
- benchmark 후에는 두 모델을 다시 모두 내려서 `status all` 기준 `not-loaded / down` 상태를 확인했다.

## 이번에 확인된 운영 포인트

### 1. launchd local wrapper 에서 repo root 를 env 로 넘겨야 한다

- copied local wrapper 는 `~/.nanobot-launchd-scripts/local-models` 아래에 놓이므로, 단순 상대경로 계산만으로는 repo root 를 다시 찾을 수 없다.
- `NANOBOT_INFRA_ROOT_DIR` env 를 plist rewrite 단계에서 주입하고, `common.sh` 가 이를 우선 사용하도록 바꿔 launchd 경로를 안정화했다.

### 2. Qwen3.6 start script 는 copied `common.sh` 가 같이 있어야 한다

- `start-qwen36-mlx.sh` 는 runtime metadata 와 MLX bootstrap helper 를 `common.sh` 에 의존한다.
- 따라서 install 단계에서 model wrapper 뿐 아니라 `common.sh` 도 함께 local script dir 로 복사해야 실제 launchd 실행이 정상 동작한다.

### 3. cache 와 benchmark 결과는 tracked path 와 분리하는 편이 안전하다

- Qwen3.6 모델 cache 는 다운로드 직후 수십 GB까지 빠르게 커질 수 있다.
- 이번에는 `infra/.local-model-cache/`, `infra/.local-model-results/` 를 `.gitignore` 처리해 repo diff churn 을 막았다.

## 검증 결과

focused test:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/test_local_model_scripts.py -q
# 1 passed
```

정책/CLI focused 검증:

```bash
cd /Volumes/ExtData/Nanobot
./infra/scripts/local-models/local-models.sh list
! ./infra/scripts/local-models/local-models.sh start all
! ./infra/scripts/local-models/local-models.sh install all
```

결과:

- `list` 는 `lfm2`, `qwen36` 를 정상 출력
- `start all` 은 `Error: start all is not allowed`
- `install all` 은 `Error: install all is not allowed`

shell syntax / wrapper validation:

```bash
cd /Volumes/ExtData/Nanobot
zsh -n infra/scripts/local-models/*.sh
./infra/scripts/local-models/local-models.sh list
./infra/scripts/local-models/local-models.sh status all
```

live install / endpoint / benchmark validation:

```bash
cd /Volumes/ExtData/Nanobot
./infra/scripts/local-models/local-models.sh install qwen36
./infra/scripts/local-models/local-models.sh status qwen36
./infra/scripts/local-models/local-models.sh smoke qwen36
./infra/scripts/local-models/local-models.sh benchmark qwen36 1

./infra/scripts/local-models/local-models.sh install lfm2
./infra/scripts/local-models/local-models.sh status all

launchctl list | rg 'com\.nanobot\.local-model-(lfm2|qwen36)'
du -sh infra/.local-model-cache infra/.local-model-results
```

순차 benchmark 검증:

```bash
cd /Volumes/ExtData/Nanobot
./infra/scripts/local-models/local-models.sh stop all
./infra/scripts/local-models/local-models.sh start qwen36
./infra/scripts/local-models/local-models.sh benchmark qwen36 3
./infra/scripts/local-models/local-models.sh stop qwen36
./infra/scripts/local-models/local-models.sh start lfm2
./infra/scripts/local-models/local-models.sh benchmark lfm2 3
./infra/scripts/local-models/local-models.sh stop lfm2
./infra/scripts/local-models/local-models.sh status all
```

실제 확인 결과:

- `com.nanobot.local-model-qwen36` loaded
- `com.nanobot.local-model-lfm2` loaded
- `Qwen3.6 MLX` endpoint `ok`
- `LFM2 llama.cpp` endpoint `ok`
- raw smoke response: `OK`
- benchmark summary: `successCount=1`, `avgLatencyMs=269`
- cache usage: `19G`
- sequential benchmark summary:
	- `qwen36`: `3/3 success`, `avgLatencyMs=932.67`
	- `lfm2`: `3/3 success`, `avgLatencyMs=204.67`
- final state after benchmark: 두 모델 모두 `launchd: not-loaded`, `listener: down`, `endpoint: unavailable`

## 현재 남아 있는 제약과 리스크

- `source` repo 코드 자체의 provider/default model 설정은 아직 Qwen3.6 기준으로 전환하지 않았다.
- 현재 benchmark 는 raw OpenAI-compatible endpoint 기준이고, Nanobot agent/full workflow benchmark 까지 붙인 것은 아니다.
- `docs/operations/status-summary/` 와 `docs/guide/` 는 이번 라운드에 맞췄지만, 과거 status-summary 문서에는 이전 label `com.nanobot.llama-lfm2-server` 참조가 남아 있다.
- 과거 문서 중 일부에는 `infra/scripts/llama/` 와 `com.nanobot.llama-lfm2-server` 참조가 남아 있을 수 있다.

## 현재 기준 권장 운영 명령

```bash
cd /Volumes/ExtData/Nanobot

./infra/scripts/local-models/local-models.sh status all
./infra/scripts/local-models/local-models.sh list
./infra/scripts/local-models/local-models.sh log qwen36
./infra/scripts/local-models/local-models.sh smoke qwen36
./infra/scripts/local-models/local-models.sh benchmark qwen36 3
```

## 다음 작업 후보

1. `~/.nanobot/config*.json` 에서 Qwen3.6 을 별도 provider/model target 으로 노출할지 결정
2. `local-models.sh benchmark` 를 Nanobot agent 경로나 OpenClaw/LLMTestTool comparative benchmark 와 연결할지 결정
3. 과거 status-summary / archive 문서의 `com.nanobot.llama-lfm2-server` 참조를 언제 정리할지 기준 확정

## 결론

이번 라운드에서 `llama` 고유 경로를 `local-models` 공용 운영 계층으로 바꾸는 1차 구현은 완료됐다. Qwen3.6 MLX 모델은 실제로 `infra` 아래 cache 로 설치됐고, launchd 기동, raw smoke, focused benchmark 까지 현재 live 기준으로 확인됐다.

<!-- llmtesttool-status:start -->
## LLMTestTool_v2 비교 벤치마크 요약
- 비교 run: `lfm2__vs__qwen36/bundle-standard-r100-2026-05-15_183119`
- 실행 시각: `2026-05-15_183119`
- 요약 모델: `gpt-5.4-2026-03-05` (external:openai)
- 가장 빠른 시작 시간: `lfm2`
- 가장 빠른 벤치마크 시간: `lfm2`
- 가장 빠른 전체 시간: `lfm2`
- 상세 비교 Markdown: `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/compare/lfm2__vs__qwen36/bundle-standard-r100-2026-05-15_183119/comparison.md`
- 최종 요약 Markdown: `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/reports/report-summary-lfm2_vs_qwen36-2026-05-15_183119.md`

### 모델 간 핵심 차이

| 지표 | lfm2 | qwen36 | 차이 | 우세 모델 |
| --- | ---: | ---: | ---: | --- | 
| 직접 호출 시작 시간 | 18185 | 22248 | -4063 | lfm2 |
| 직접 호출 벤치마크 시간 | 215 | 2337 | -2122 | lfm2 |
| 직접 호출 전체 시간 | 18755 | 24881 | -6126 | lfm2 |
| quick.ttftMs | 197 | 2312 | -2115 | lfm2 |
| quick.promptTokens | 14 | 17 | -3 | lfm2 |
| quick.completionTokens | 2 | 1 | 1 | qwen36 |
| quick.totalTokens | 16 | 18 | -2 | lfm2 |
| quick.completionTokensPerSec | 9.3 | 0.43 | 8.87 | lfm2 |
| API 경유 전체 시간 | 15159 | 23591 | -8432 | lfm2 |
| assistant.ttftMs | 2151 | 2450 | -299 | lfm2 |
| assistant.promptTokens | 5823 | 5712 | 111 | qwen36 |
| assistant.completionTokens | 5 | 5 | 0 | 동점 |
| assistant.totalTokens | 5828 | 5717 | 111 | qwen36 |
| assistant.completionTokensPerSec | 2.04 | 1.97 | 0.07 | lfm2 |
| 부하 테스트 실패율 | 0 | 0 | 0 | 동점 |
| 부하 테스트 최악 지연 | 23759 | 23366 | 393 | qwen36 |
<!-- llmtesttool-status:end -->
