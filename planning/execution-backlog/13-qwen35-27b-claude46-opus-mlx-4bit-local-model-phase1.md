# 13 Qwen3.5 27B Base MLX 4bit Local Model Phase1 Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `mlx-community/Qwen3.5-27B-4bit` 를 Nanobot 현재 `infra/scripts/local-models` 운영 환경에 phase1 후보로 편입할 수 있도록, 현재 `qwen36` baseline 검증, 새 MLX target 추가 범위, smoke 기준, default 전환 보류 조건을 고정한다.

**Architecture:** 선택한 모델은 `Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled` 계열을 MLX 4-bit 로 양자화한 Apple Silicon 적합 artifact 다. 따라서 기존 Nanobot `local-models` 의 `mlx_lm.server` lane 안에서 새 target 으로 통합하는 경로를 따른다. 이번 단계에서는 기존 `qwen36` 을 즉시 교체하지 않고, 먼저 병행 가능한 새 local-model target 으로 추가한 뒤 start/stop/status/smoke 와 OpenAI-compatible chat-completions 동작을 검증한다.

**Tech Stack:** zsh launchd wrappers, `mlx_lm.server`, Hugging Face MLX safetensors, Nanobot Python runtime, local launchd services, pytest, Vitest.

---

## 1. 최종 결정과 현재 확인된 사실

- 현재 Nanobot local-model 운영 환경은 macOS Mac mini 기준이다.
- 현재 `infra/scripts/local-models` 런타임은 아래 둘만 직접 운영한다.
  - `qwen36`: `mlx_lm.server`
  - `lfm2`: `llama.cpp`
- live 상태 기준 현재 실행 중인 local model 은 `qwen36` 이다.
- 이전 검토 대상이던 `mconcat/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled-NVFP4` 는 현재 macOS local-model lane 에 직접 통합하지 않기로 이미 결론냈다.
- 최종 선택 모델은 `mlx-community/Qwen3.5-27B-4bit` 이다.
- Hugging Face 모델 카드 기준 현재 확인된 사실은 아래다.
  - 포맷: MLX 4-bit
  - 크기: 약 14 GB, Hub 표시상 MLX 15.1 GB
  - 권장 환경: Apple Silicon Mac, 최소 24 GB unified memory, 권장 32 GB+
  - 계보: `Qwen/Qwen3.5-27B` base + `Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled` finetune 계열
  - 추론 엔진: `mlx_lm` 기준 quick-start 와 `apply_chat_template(..., enable_thinking=True)` 예시 제공
  - reasoning 특성: `<think>` block 을 생성할 수 있으며 필요 시 후처리 strip 전략이 필요하다
- 따라서 이 후보는 현재 Nanobot `mlx_lm.server` lane 에 편입 가능한 방향으로 본다.

## 2. 결론 요약

현재 환경 기준 결론은 아래다.

```text
Proceed with a phase1 integration plan for the selected MLX 4-bit candidate, without replacing qwen36 as default yet.
```

이유는 아래와 같다.

1. 선택 모델은 현재 local-models 가 이미 운영 중인 `mlx_lm.server` 경로에 맞는 MLX artifact 다.
2. Apple Silicon 기준 메모리와 디스크 크기가 현행 Mac mini 운영 lane 안에서 현실적이다.
3. 원본 계보가 사용자가 원한 `Qwen3.5-27B Claude 4.6 Opus reasoning distilled` 계열과 맞는다.
4. 현재는 새 모델 품질과 안정성을 검증하는 단계이므로, 기본값 교체보다 병행 target 추가가 더 안전하다.

따라서 이번 계획은 **NVFP4 no-go 정리 문서가 아니라, 선택한 MLX 후보를 현재 `local-models` lane 에 안전하게 붙이기 위한 phase1 plan** 으로 본다.

## 3. 이번 계획의 목표 범위

이번 계획은 아래를 끝내는 데 목적이 있다.

1. 현재 `qwen36` baseline stop/start/status/smoke 가 정상임을 먼저 확인
2. 선택한 MLX 모델을 위한 새 local-model target 설계를 고정
3. phase1 구현 범위를 `start/stop/status/smoke` 와 endpoint 호환성 검증까지로 제한
4. 기본 local model 전환은 별도 판단 포인트로 남겨 둠

## 4. 비범위

이번 계획에서는 아래를 하지 않는다.

- 첫 단계에서 `qwen36` 을 즉시 제거하거나 덮어쓰기
- baseline 검증 없이 `local-models.sh use` 로 새 모델을 default local model 로 즉시 전환
- reasoning `<think>` 처리나 max token 요구를 검증하지 않은 채 WebUI/Telegram 기본 응답 모델로 바로 노출
- NVFP4 전용 원격 `vLLM` lane 구현을 이번 계획 범위에 다시 포함하기

## 5. 선택 모델 기준 구현 방향

### 경로 A: 기존 `mlx_lm.server` lane 에 새 target 추가

- 새 shell wrapper 를 `qwen36` 패턴으로 추가한다.
- 새 모델 ID 는 phase1 에서 명시적으로 분리한다. 예: `qwen35-base-mlx-4bit`.
- 포트, launchd label, PID 파일, log 파일, health URL 을 기존 `qwen36` 과 충돌하지 않게 분리한다.
- `local-models.sh` 의 start/stop/status/smoke/use 목록에 새 target 을 추가한다.

### 경로 B: 기본값 전환은 검증 후 별도 결정

- `qwen36` 과 비교해 start latency, first response, smoke 안정성, reasoning 응답 품질을 먼저 본다.
- baseline 대비 이점이 확인될 때만 `~/.nanobot/local-llm.env` 의 기본값 교체를 검토한다.

## 6. 작업 분해

### Task 1: 선택 모델 facts 와 phase1 decision 고정

**Files:**
- Modify: `docs/planning/execution-backlog/13-qwen35-27b-claude46-opus-mlx-4bit-local-model-phase1.md`
- Reference: `infra/scripts/local-models/start-qwen36-mlx.sh`
- Reference: `infra/scripts/local-models/local-models.sh`

- [ ] **Step 1: 모델 카드 근거 고정**

Record the exact facts:

```text
artifact: MLX 4-bit
lineage: Qwen/Qwen3.5-27B -> Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled
size: about 14 GB (Hub MLX size about 15.1 GB)
host fit: Apple Silicon, 24 GB minimum, 32 GB recommended
runtime path: mlx_lm / mlx_lm.server compatible lane
```

- [ ] **Step 2: phase1 decision 문구 고정**

Decision text:

```text
Add the selected MLX model as a new local-models target first. Do not replace qwen36 as the default until baseline and smoke validation pass.
```

### Task 2: 현재 local-model baseline 검증

**Files:**
- Reference: `infra/scripts/local-models/local-models.sh`
- Reference: `infra/scripts/local-models/status-local-model-services.sh`

- [ ] **Step 1: baseline 상태 저장**

Run:

```bash
cd /Volumes/ExtData/Nanobot
infra/scripts/local-models/local-models.sh status all
launchctl list | grep 'com.nanobot.local-model'
ps aux | grep 'mlx_lm.server\|llama-server' | grep -v grep
```

Expected now:

```text
qwen36 loaded/up/ok
lfm2 down
```

- [ ] **Step 2: qwen36 stop 검증**

Run:

```bash
cd /Volumes/ExtData/Nanobot
infra/scripts/local-models/local-models.sh stop qwen36
infra/scripts/local-models/local-models.sh status qwen36
```

Expected:

```text
listener: down
endpoint: unavailable
```

- [ ] **Step 3: qwen36 start + smoke 검증**

Run:

```bash
cd /Volumes/ExtData/Nanobot
infra/scripts/local-models/local-models.sh start qwen36
infra/scripts/local-models/local-models.sh status qwen36
infra/scripts/local-models/local-models.sh smoke qwen36
```

Expected:

```text
listener: up
endpoint: ok
smoke succeeds
```

### Task 3: 새 MLX target 설계 고정

**Files:**
- Modify: `infra/scripts/local-models/local-models.sh`
- Add: `infra/scripts/local-models/start-qwen35-base-mlx-4bit.sh`
- Add or Modify: related status/smoke helper wiring as needed
- Reference: `infra/scripts/local-models/start-qwen36-mlx.sh`

- [ ] **Step 1: 새 target naming 고정**

Record a stable target id, for example:

```text
qwen35-base-mlx-4bit
```

- [ ] **Step 2: model id 와 runtime args 고정**

Record:

```text
model repo: mlx-community/Qwen3.5-27B-4bit
runtime: mlx_lm.server
chat-template behavior: reasoning-capable, <think> may appear
```

- [ ] **Step 3: conflict-free runtime envelope 고정**

Define:

```text
dedicated port
dedicated launchd label
dedicated pid/log path
no collision with qwen36
```

### Task 4: phase1 구현과 검증 범위 고정

**Files:**
- Modify: `infra/scripts/local-models/local-models.sh`
- Modify: `infra/scripts/local-models/status-local-model-services.sh`
- Optional Modify: `infra/scripts/local-models/use-local-model.sh`
- Optional Modify: `source/nanobot/local_llm_control.py`
- Optional Modify: `source/nanobot/model_targets.py`

- [ ] **Step 1: local-models verbs 연결**

Ensure the new target participates in:

```text
status
start
stop
restart
smoke
use
```

- [ ] **Step 2: smoke 기준 고정**

Record that success means:

```text
OpenAI-compatible chat-completions response returns
basic response-check succeeds
server health/listener checks are green
```

- [ ] **Step 3: default 전환 보류 조건 기록**

Do not switch the default local model until all of the below hold:

```text
baseline qwen36 stop/start/smoke passed
new MLX target stop/start/smoke passed
response shape is compatible with current Nanobot consumers
reasoning output does not break current WebUI/Telegram UX
```

### Task 5: 사용자 의사결정 포인트 재정리

**Files:**
- Modify: this plan file

- [ ] **Step 1: 이번 phase1 종료 조건 기록**

Finish this phase when one of the following is true:

1. 새 MLX target 이 정상 구동되고 default 전환 후보가 된다.
2. 새 MLX target 은 구동되지만 UX 또는 품질 이슈 때문에 optional target 으로만 남긴다.
3. 새 MLX target 이 baseline smoke 를 통과하지 못해 보류한다.

- [ ] **Step 2: 최종 선택지 3개로 압축**

Final choices after phase1:

1. `qwen36` 유지, 새 모델은 optional target 으로 제공
2. 새 MLX 모델을 local default 로 승격
3. 새 MLX 모델 보류 후 다른 MLX/GGUF 후보로 이동

## 7. 현재 시점 권고

현재 가장 현실적인 권고는 아래다.

1. 먼저 현재 `qwen36` baseline stop/start/smoke 를 실제로 검증한다.
2. 그 다음 `mlx-community/Qwen3.5-27B-4bit` 를 새 `mlx_lm.server` target 으로 추가한다.
3. phase1 에서는 default 전환보다 새 target 의 안정적 기동과 smoke 성공을 우선한다.
4. `<think>` block 노출 여부와 긴 reasoning 응답이 현재 WebUI/Telegram UX 에 문제를 만들지 따로 확인한다.

## 8. 바로 다음 실행 후보

가장 작은 다음 실행은 아래다.

```bash
cd /Volumes/ExtData/Nanobot
infra/scripts/local-models/local-models.sh stop qwen36
infra/scripts/local-models/local-models.sh status qwen36
infra/scripts/local-models/local-models.sh start qwen36
infra/scripts/local-models/local-models.sh smoke qwen36
```

그 다음 새 후보 구현 단계는 아래 순서다.

```bash
cd /Volumes/ExtData/Nanobot
# start-qwen35-base-mlx-4bit.sh 초안 추가
# local-models.sh target 목록 연결
# status/smoke/use wiring 추가
```

이 검증은 새 모델을 추가하기 전에 현재 local-model 운영 경로가 정상인지 먼저 확인해 준다.
