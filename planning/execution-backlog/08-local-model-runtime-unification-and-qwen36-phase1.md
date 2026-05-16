# 08 Local Model Runtime Unification And Qwen3.6 Phase 1

## 1. 문서 목적

이 문서는 Nanobot의 현재 `infra/scripts/llama/` 기반 로컬 모델 운영 경로를 `llama 전용 관리`가 아닌 `로컬 모델 공용 관리` 계층으로 재정리하고, 그 첫 phase-1 대상으로 `mlx-community/Qwen3.6-35B-A3B-4bit` 를 운영 가능한 형태로 붙이기 위한 실행 문서다.

이번 단계의 목표는 네 가지다.

- `llama` 라는 이름에 묶여 있는 현재 운영 스크립트와 launchd label 체계를 `local-models` 계층으로 정리하기
- 기존 LFM2 `llama.cpp` 런타임과 새 Qwen3.6 `mlx_lm.server` 런타임을 같은 UX로 설치, 시작, 중지, 상태확인, 제거할 수 있게 만들기
- Qwen3.6 모델 다운로드/캐시 위치와 launchd 기반 실행 경로를 Nanobot 운영 기준에 맞게 고정하기
- `/Volumes/ExtData/AI_Project/LLMTestTool` 의 lifecycle/benchmark 자산을 참고해 Nanobot에서도 focused LLM test 가 가능한 최소 검증 경로를 만들기

## 2. 왜 지금 필요한가

현재 Nanobot 로컬 모델 운영 흐름은 아래 세 제약이 있다.

1. `infra/scripts/llama/` 는 이름과 구현 모두 사실상 `LFM2 + llama.cpp` 단일 경로로 고정돼 있다.
2. 사용자가 붙이려는 새 모델은 `mlx-community/Qwen3.6-35B-A3B-4bit` 이고, 이는 현재 경로와 다른 `MLX` 런타임 계열이다.
3. 모델 lifecycle 을 통합적으로 관리하는 상위 naming 이 없어, 이후 Gemma/Qwen 계열을 추가할수록 `llama` 디렉터리 안에 runtime 이 섞이는 구조가 된다.

따라서 이번 slice 는 단순히 Qwen 스크립트 몇 개를 추가하는 작업이 아니라, `로컬 모델 운영 계층 naming`, `runtime 추상화`, `launchd label`, `검증 방식` 을 함께 다시 고정하는 작업으로 보는 것이 맞다.

## 3. phase-1 범위

포함할 것:

- `infra/scripts/llama/` 를 대체할 새 `infra/scripts/local-models/` 관리 계층 도입
- 기존 LFM2 lifecycle 을 새 계층으로 이관하고 legacy wrapper 또는 alias 유지 여부 정리
- `mlx-community/Qwen3.6-35B-A3B-4bit` 용 install/start/status/stop/uninstall 경로 추가
- Qwen 모델 다운로드 및 Hugging Face cache/storage root 기본값 정리
- launchd plist label 과 wrapper script naming 을 새 계층 기준으로 정리
- focused raw endpoint smoke test 와 benchmark entrypoint 초안 추가
- planning/docs/guide 수준의 운영 문서 동기화

제외할 것:

- Nanobot의 기본 provider/model 을 즉시 Qwen3.6 으로 전환하는 작업
- 여러 MLX 모델을 한 번에 모두 productized 하는 작업
- OpenClaw 전체 benchmark CLI 를 Nanobot 안으로 그대로 복제하는 작업
- GPU/메모리 auto-tuning 고도화
- 다중 모델 동시 상주 운영 정책

## 4. 핵심 설계 결정

### 4.1 naming: `llama` 대신 `local-models`

phase-1 기준 권장 naming 은 아래다.

- 디렉터리: `infra/scripts/local-models/`
- 통합 엔트리포인트: `infra/scripts/local-models/local-models.sh`
- 모델별 runtime start script: `start-lfm2-llama-cpp.sh`, `start-qwen36-mlx.sh`
- launchd label: `com.nanobot.local-model-lfm2`, `com.nanobot.local-model-qwen36`
- 메모리 보호 정책: `start all`, `install all` 금지

이 naming 을 쓰는 이유는 아래와 같다.

- runtime 구현체 이름(`llama`, `mlx`)보다 운영 대상(`local models`)을 상위 개념으로 두는 편이 확장에 안전하다.
- 이후 Gemma, 다른 Qwen, LM Studio 계열을 붙여도 동일한 명령 UX를 유지할 수 있다.
- 현재 `com.nanobot.llama-lfm2-server` 처럼 런타임 세부 구현이 label 에 박혀 있는 구조보다 운영 의도가 명확하다.

### 4.2 runtime 추상화: model별 전용 start script + 공통 control shell

phase-1 에서는 완전한 manifest 시스템보다 아래 구조를 우선한다.

- 공통 control shell 은 `list/install/start/status/stop/uninstall/log/benchmark` UX를 제공한다.
- 실제 런타임 차이는 모델별 start script 가 숨긴다.
- 상태 확인은 `port`, `launchd label`, `pid`, `/v1/models` 응답으로 공통화한다.
- uninstall 은 launchd plist, local wrapper, pid/log 정리까지 포함하되 모델 cache 자체 삭제는 명시적 옵션으로 분리한다.
- `all` target 은 `status/log/stop/uninstall` 에만 허용하고, 기동 계열에는 허용하지 않는다.

즉, `control plane 은 공통`, `runtime launch 는 per-model` 로 나누는 방식이다. 이는 현재 LLMTestTool 의 `scripts/<model>/... + _shared helper` 패턴과 가장 가깝고, 현재 Nanobot 스크립트 규모에도 맞다.

### 4.3 Qwen3.6 모델 storage 기본값

Qwen3.6 MLX 모델의 download/cache 기본 경로는 Nanobot 쪽에서 명시적으로 고정한다.

권장 기본값:

- Hugging Face root: `/Volumes/ExtData/Nanobot/infra/.local-model-cache/huggingface`
- MLX cache 보조 root: `/Volumes/ExtData/Nanobot/infra/.local-model-cache/mlx`

이 경로를 쓰는 이유는 아래와 같다.

- 사용자가 요구한 `Nanobot/infra 아래` 운영 자산 요구를 충족한다.
- 기존 `LLM_Test` 캐시 경로에 의존하지 않고 Nanobot 운영 자산을 분리할 수 있다.
- git tracked repo 안에 대형 모델 파일이 직접 섞이지 않도록 `.local-model-cache` 를 별도 취급할 수 있다.

phase-1 에서는 이 경로를 `.gitignore` 또는 운영 문서에서 명시적으로 non-tracked cache 로 다룬다.

### 4.4 LLM test 방향: LLMTestTool 참조 + Nanobot focused runner

이번 단계에서 가져올 핵심은 아래 두 가지다.

- `scripts/_shared/mlx-lm-common.sh` 의 MLX lifecycle 패턴
- raw OpenAI-compatible endpoint 기준 smoke/benchmark 습관

다만 Nanobot에는 아래 수준만 우선 반영한다.

- 모델별 `status` 에 endpoint 응답 확인 포함
- `curl` 또는 간단한 JSON payload 기반 smoke script 추가
- 필요 시 `LLMTestTool` 러너를 호출하는 thin wrapper 또는 사용 가이드 문서화

즉, phase-1 목표는 `LLMTestTool 전체 이식` 이 아니라 `Nanobot 운영 경로에서 재현 가능한 focused test 확보` 다.

## 5. 현재 코드 anchor

현재 기준 직접 영향을 받는 주요 위치는 아래다.

- `infra/scripts/llama/llama.sh`
- `infra/scripts/llama/install-launchd-services.sh`
- `infra/scripts/llama/start-llama-services.sh`
- `infra/scripts/llama/status-llama-services.sh`
- `infra/scripts/llama/stop-llama-services.sh`
- `infra/scripts/llama/uninstall-llama-services.sh`
- `infra/scripts/llama/start-llama-cpp-lfm2-server.sh`
- `infra/launchd/com.nanobot.llama-lfm2-server.plist`
- `infra/scripts/setup-nanobot.sh`
- `infra/scripts/validate-nanobot-setup.sh`
- `docs/planning/todo.md`
- `docs/planning/execution-backlog/README.md`
- `docs/guide/*` 또는 `docs/operations/status-summary/*` 의 관련 운영 문서

참고 자산은 아래다.

- `/Volumes/ExtData/AI_Project/LLMTestTool/scripts/_shared/mlx-lm-common.sh`
- `/Volumes/ExtData/AI_Project/LLMTestTool/scripts/gemma4-e4b-mlx/*`
- `/Volumes/ExtData/AI_Project/LLMTestTool/README.md`
- `/Volumes/ExtData/AI_Project/LLMTestTool/supported-models.md`

## 6. 작업 분해

### Step 1. local-models naming skeleton 도입

- `infra/scripts/local-models/` 디렉터리 생성
- 통합 control script 초안 작성
- 기존 `infra/scripts/llama/` 제거 여부를 같은 slice 에서 판단

### Step 2. LFM2 경로를 새 naming 으로 이관

- 기존 LFM2 `llama.cpp` start script 를 `local-models` 계층으로 이동
- launchd plist/label 을 `com.nanobot.local-model-lfm2` 형태로 재정리
- 기존 label 은 migration 기간 동안 clean uninstall 대상에 포함

### Step 3. Qwen3.6 MLX runtime 추가

- `mlx_lm.server` 기반 start/status/stop/install/uninstall 흐름 추가
- port, concurrency, cache path 기본값 정리
- Hugging Face 모델 install 또는 lazy download 동작을 운영 기준으로 문서화

### Step 4. focused test 경로 추가

- raw `/v1/models` health check
- raw `/v1/chat/completions` smoke test
- 필요 시 간단 benchmark wrapper 추가
- `LLMTestTool` 러너 호출 방식 또는 결과 저장 위치 기준 문서화

### Step 5. setup/validation/docs 동기화

- `setup-nanobot.sh` 에 필요한 dependency 안내 또는 bootstrap 보강
- `validate-nanobot-setup.sh` 에 local model 상태 확인 항목 추가 여부 판단
- guide/status-summary/planning 문서 동기화

## 7. 실제 코드 작업 backlog

### 7.1 control surface slice

- 새 `local-models.sh` UX 정의
- 모델 목록, 기본 target, help wording 정리
- install/start/status/stop/uninstall/log 공통 인터페이스 정의

### 7.2 LFM2 migration slice

- `llama.cpp` 실행 스크립트 이관
- launchd plist 파일 rename 및 install/uninstall 로직 갱신
- legacy `com.nanobot.llama-lfm2-server`, `com.aiassistant.llama-lfm2-server`, `com.zeroclaw.llama-cpp` cleanup 범위 포함
- migration 완료 시 `infra/scripts/llama/` 와 legacy plist 제거

### 7.3 Qwen3.6 MLX slice

- `mlx_lm.server` binary resolution
- model/cache 경로, port, log, pid 관리
- launchd plist 및 local wrapper 설치

### 7.4 test slice

- endpoint smoke script
- optional benchmark script 또는 LLMTestTool bridge script
- 결과 저장/로그 확인 경로 문서화

### 7.5 docs and operations slice

- planning/todo active item 반영
- execution backlog index 갱신
- guide 또는 operations memo 에 새 local model 운영 기준 반영

## 8. 검증 계획

권장 검증 순서는 아래다.

1. `infra/scripts/local-models/local-models.sh install lfm2`
2. `infra/scripts/local-models/local-models.sh status lfm2`
3. `infra/scripts/local-models/local-models.sh install qwen36`
4. `infra/scripts/local-models/local-models.sh status qwen36`
5. raw `curl http://127.0.0.1:<port>/v1/models`
6. raw `POST /v1/chat/completions` smoke
7. 필요 시 `LLMTestTool` 연계 benchmark 또는 동등 benchmark wrapper 실행

launchd 기준 검증은 아래를 포함한다.

- `launchctl list | grep nanobot`
- model별 port listener 확인
- log tail 확인
- uninstall 이후 plist/wrapper 정리 여부 확인

## 9. 완료 기준

1. `llama` 라는 이름 없이도 Nanobot에서 로컬 모델 lifecycle 을 관리할 수 있다.
2. 기존 LFM2 와 새 Qwen3.6 을 같은 control UX로 설치, 시작, 중지, 상태확인, 제거할 수 있다.
3. Qwen3.6 모델 cache/storage 위치가 Nanobot 운영 경로 기준으로 고정된다.
4. raw OpenAI-compatible endpoint smoke test 가 Nanobot 경로에서 재현 가능하다.
5. 필요한 planning/documentation 이 현재 운영 기준과 어긋나지 않게 갱신된다.

## 10. 다음 단계 연결

- 구현 시작 전 `planning/todo.md` 에 active item 으로 승격
- 구현 후 live 결과는 `operations/status-summary/` 날짜형 메모에 기록
- 필요 시 phase-2 에서 Gemma 등 추가 MLX 모델을 같은 계층으로 편입