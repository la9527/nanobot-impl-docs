# 09 Nanobot LLMTestTool_v2 Sequential Benchmark Phase 1

## 1. 문서 목적

이 문서는 기존 `/Volumes/ExtData/AI_Project/LLMTestTool` 에 남아 있는 OpenClaw 결합 경로를 제외하고, Nanobot 현재 운영 경로에 맞는 새 benchmark 전용 작업공간을 `/Volumes/ExtData/AI_Project/LLMTestTool_v2` 로 다시 세우기 위한 phase-1 실행 문서다.

이번 단계의 목표는 다섯 가지다.

- `LLMTestTool` 에 남아 있는 OpenClaw runtime benchmark, 설정 의존성, 문서 서술을 `v2` 에서 제거하기
- `LLMTestTool_v2` 가 Nanobot `infra/scripts/local-models/local-models.sh` 를 lifecycle backend 로 사용하게 만들기
- 메모리 제약에 맞춰 한 번에 한 모델만 올리고, 각 모델당 기본 benchmark 를 1회만 실행하는 sequential test 흐름을 기본값으로 고정하기
- 모델별 `start -> ready -> benchmark -> stop` 전체 시간을 기록해, startup/shutdown latency 와 raw inference latency 를 함께 비교 가능한 결과 포맷을 만들기
- 최종 benchmark 결과를 가장 성능이 좋은 benchmark 대상 모델이 직접 요약해 Markdown 문서로 저장하는 흐름을 추가하기

추가로 이번 문서에서는 아래 두 기능도 phase-1 범위에 포함한다.

- 여러 prompt set 을 같은 모델에 순차 반복 적용하는 explicit stress harness
- Nanobot agent loop 를 실제로 경유하는 end-to-end assistant benchmark lane

이번 문서의 최신 확장 범위는 아래 세 항목도 포함한다.

- direct/assistant/stress 각 요청 단위 raw row 에 token, throughput, time-to-first-token 계측값을 보존하는 detailed metrics lane
- 100회 stress 실행 결과를 케이스별로 left/right 모델에 직접 대응시키는 상세 비교 표와 raw appendix
- 특정 prompt category, language, difficulty 같은 케이스 속성별로 어느 모델이 느리거나 빠른지 설명 가능한 비교 집계

## 2. 왜 지금 필요한가

현재 상황은 아래 다섯 문제가 같이 묶여 있다.

1. 기존 `LLMTestTool` 은 raw benchmark 뿐 아니라 OpenClaw runtime benchmark 와 `~/.openclaw/*` 로컬 설정 의존성을 함께 들고 있어, 현재 운영하지 않는 비교 surface 가 기본 UX 안에 섞여 있다.
2. Nanobot 쪽 로컬 모델 운영은 이미 `infra/scripts/local-models/` 와 launchd label 기준으로 재정리됐는데, benchmark 도구 쪽은 아직 그 control plane 을 source-of-truth 로 삼지 않는다.
3. 현재 머신 메모리 제약상 여러 모델을 동시에 띄우는 benchmark 기본값은 운영에 맞지 않고, 실제 확인하고 싶은 값도 `모델별 단독 기동 시 걸리는 시간` 과 `단발 benchmark 성능` 이다.
4. 현재 계획은 raw one-shot lane 에 치우쳐 있어, 같은 모델에 서로 다른 prompt set 을 길게 반복했을 때의 tail latency 와 안정성 저하를 읽기 어렵다.
5. 실제 Nanobot 운영에서는 agent loop, session, tool policy, response formatting 이 함께 작동하는데, 현재 문서 초안만으로는 raw endpoint 와 assistant end-to-end 경로를 분리 비교하기 어렵다.

따라서 이번 slice 는 `기존 benchmark 도구 정리` 가 아니라, `Nanobot local-model control plane 을 이용하는 단일 모델 순차 benchmark harness` 를 다시 정의하는 작업으로 보는 편이 맞다.

## 3. phase-1 범위

포함할 것:

- `/Volumes/ExtData/AI_Project/LLMTestTool_v2` 초기 구조와 README 초안
- Nanobot local-models control shell 호출 기반 model catalog 정리
- `lfm2`, `qwen36` 2개 모델 기준 catalog 와 lifecycle 연결
- 기본 benchmark 명령에서 모델별 1회 실행만 수행하는 sequential runner 추가
- `lfm2`, `qwen36` 두 benchmark 대상을 하나의 compare run 으로 묶는 flow 추가
- prompt set 파일을 읽어 같은 모델을 여러 번 돌리는 stress runner 추가
- Nanobot API server `/v1/chat/completions` 를 통과하는 assistant benchmark runner 추가
- 모델별 `startupMs`, `readyMs`, `benchmarkMs`, `shutdownMs`, `totalMs` 기록
- 결과 저장 포맷(`summary.json`, `results.jsonl`, optional markdown report) 정의
- `lfm2`, `qwen36` 기준 비교 report 생성 경로 설계
- 최종 benchmark 결과를 LLM이 요약한 Markdown 문서와 선택된 요약 모델 metadata 저장 경로 설계
- 결과 디렉터리에서 모델명/비교대상/프로파일이 먼저 드러나는 path 규칙 적용
- Nanobot docs/planning 과 운영 문서가 새 benchmark 방향을 가리키도록 정리

제외할 것:

- OpenClaw runtime benchmark 재이식
- 다중 모델 동시 기동 benchmark
- multi-user / multi-session 경쟁 부하 테스트
- WebUI websocket 프런트엔드를 포함한 브라우저 E2E 벤치마크
- `gemma4-e4b` 포함 추가 모델군 onboarding

## 4. 핵심 설계 결정

### 4.1 source of truth: Nanobot `local-models` 를 직접 사용

`LLMTestTool_v2` 는 모델 서버를 직접 띄우는 별도 lifecycle 계층을 다시 만들지 않고, 아래 명령을 직접 호출하는 thin harness 로 두는 편이 맞다.

- `./infra/scripts/local-models/local-models.sh list`
- `./infra/scripts/local-models/local-models.sh start <target>`
- `./infra/scripts/local-models/local-models.sh status <target>`
- `./infra/scripts/local-models/local-models.sh benchmark <target> 1`
- `./infra/scripts/local-models/local-models.sh stop <target>`

이렇게 두면 model install/start/stop/status 정책이 Nanobot 와 benchmark 도구 사이에서 어긋나지 않는다. `v2` 는 control plane 을 소유하지 않고, `실행 순서`, `시간 측정`, `결과 집계`, `비교 보고서` 만 담당한다.

### 4.2 benchmark lane 분리

phase-1 에서는 benchmark lane 을 세 가지로 나눈다.

- `raw`: `local-models.sh benchmark <target> <count>` 를 감싼 direct endpoint lane
- `assistant`: Nanobot OpenAI-compatible API server `POST /v1/chat/completions` 를 통해 agent loop 를 통과시키는 lane
- `stress`: 선택한 prompt set 을 같은 모델에 순차 반복 적용하는 explicit long-run lane

`assistant` lane 의 기준 진입점은 `source/nanobot/api/server.py` 의 `handle_chat_completions()` 과 `source/nanobot/cli/commands.py` 의 `nanobot serve` command 로 둔다. 즉 `LLMTestTool_v2` 는 WebUI 나 websocket 대신, Nanobot 가 이미 제공하는 OpenAI-compatible API server 를 end-to-end benchmark anchor 로 사용한다.

### 4.3 기본 실행 정책: 모델별 1회, 순차 실행, 동시 기동 금지

phase-1 기본 정책은 아래로 고정한다.

- default target set 은 `lfm2`, `qwen36`
- default benchmark count 는 각 모델당 `1`
- 실행 순서는 `stop all -> start model -> wait ready -> benchmark 1회 -> stop model -> 다음 모델`
- `all` 은 `동시 start` 의미가 아니라 `미리 정의된 model list 를 순차 순회` 한다는 뜻으로만 사용

단, long-run 검증은 기본 명령이 아니라 explicit mode 로만 연다.

- `run all` 은 여전히 모델당 1회 quick 비교만 수행
- `stress <target>` 또는 `stress all` 만 prompt set 반복 실행을 수행
- `assistant` lane 도 각 모델을 하나씩만 붙이고, 이전 모델이 완전히 내려간 뒤 다음 모델로 넘어간다

즉, `LLMTestTool_v2 run all` 의 의미는 `둘 다 동시에 올리기` 가 아니라 `하나씩 돌고 내리기` 다. 이 semantics 를 README 와 CLI help 에서 명확히 고정해야 한다.

### 4.4 성능 지표: inference 만이 아니라 lifecycle 시간까지 본다

사용자가 확인하려는 값은 단순 raw latency 하나가 아니라, 실제 운영에서 체감되는 `기동/종료 비용 + 1회 응답 성능` 이다. phase-1 에서는 아래 지표를 최소 집계로 본다.

- `startRequestedAt`
- `readyAt`
- `benchmarkStartedAt`
- `benchmarkFinishedAt`
- `stopRequestedAt`
- `stoppedAt`
- `startupMs = readyAt - startRequestedAt`
- `benchmarkMs = benchmarkFinishedAt - benchmarkStartedAt`
- `shutdownMs = stoppedAt - stopRequestedAt`
- `totalMs = stoppedAt - startRequestedAt`

raw benchmark 자체에서 나온 `successCount`, `failureCount`, `avgLatencyMs`, `responsePreview` 도 함께 summary 에 남겨야 한다.

assistant/stress lane 에서는 아래 지표를 추가로 남긴다.

- `lane`: `raw`, `assistant`, `stress`
- `promptSetId`
- `repeatCount`
- `assistantSessionKey` 또는 동등한 API session 식별값
- `stopReason` 또는 surfaced error string
- `toolCallCount`, `usage.prompt_tokens`, `usage.completion_tokens` 처럼 Nanobot API 응답에서 바로 읽을 수 있는 메타데이터

latest metric 확장에서는 아래 값을 lane 공통 raw row schema 로 추가한다.

- `caseId`
- `promptCategory`
- `language`
- `difficulty`
- `promptText`
- `promptChars`
- `responseChars`
- `promptTokens`
- `completionTokens`
- `totalTokens`
- `ttftMs`
- `completionTokensPerSec`
- `startedAt`, `finishedAt`
- `rawResponsePath` 또는 동등한 raw payload 참조 경로

여기서 `ttftMs` 는 단순 전체 latency 가 아니라, streaming mode 기준 first token 이 도착한 시점까지의 시간으로 정의한다. 즉 phase-1 최신 범위에서는 direct endpoint 와 assistant endpoint 모두 `stream: true` 측정 경로를 추가해 진짜 initial utterance latency 를 계측하는 쪽으로 고정한다. streaming fallback 이 불가능한 runtime 이면 `ttftMs` 를 `null` 로 남기고 reason field 에 unsupported 를 기록한다.

### 4.5 결과 저장: model별 raw 결과 + run-level 비교 요약

결과 저장은 두 층으로 나누는 편이 맞다.

- model-level artifact: `LLMTestTool_v2/results/models/<model>/<lane>/<profile>-<timestamp>/summary.json`, `results.jsonl`
- compare-run artifact: `LLMTestTool_v2/results/compare/<left>__vs__<right>/<profile>-<timestamp>/comparison.json`, `comparison.md`
- LLM summary artifact: `LLMTestTool_v2/results/compare/<left>__vs__<right>/<profile>-<timestamp>/reports/final-summary.md`, `summary-metadata.json`

이 구조를 쓰면 모델별 개별 실행 기록과, 한 번의 sequential batch 에서 얻은 비교표를 동시에 유지할 수 있다.

stress/assistant lane 까지 포함하면 run-level report 에는 최소 아래 축이 같이 들어가야 한다.

- model별 quick run 결과
- model별 assistant run 결과
- prompt set 별 stress run 성공률, 평균 지연, 최악 지연
- raw 대비 assistant 오버헤드
- left/right 모델 간 delta 와 winner 요약

latest reporting 확장에서는 위 요약 외에 아래 두 종류를 추가로 생성한다.

- `per-case` 상세 비교 표: 100회 실행 각각에 대해 left/right 모델의 `ttftMs`, `latencyMs`, `promptTokens`, `completionTokens`, `completionTokensPerSec`, `winner` 를 한 줄씩 비교
- `grouped-insight` 비교 표: `promptCategory`, `language`, `difficulty` 단위로 평균/중앙값/p95/worst 를 집계하고 어느 모델이 상대적으로 느린지 또는 빠른지 서술

`profile` 은 최소 아래 정보를 path 에 반영한다.

- quick run: `default`
- stress run: `standard-r10` 같은 `promptSetId + repeatCount`
- compare run: `quick`, `assistant`, `stress-standard-r10` 같은 비교 목적

즉, 현재처럼 날짜만 있는 `results/runs/<timestamp>` 디렉터리는 phase-1 완료 전 제거하고, 경로만 봐도 `무슨 모델을 어떤 형식으로 돌렸는지` 읽히게 만드는 것이 목표다.

### 4.6 최종 Markdown 요약 생성 전략

최종 결과 문서는 사람이 직접 쓰는 것이 아니라, benchmark 결과 JSON 과 comparison artifact 를 입력으로 받아 LLM 이 Markdown 으로 요약하는 흐름을 phase-1 에 포함한다.

기본 원칙은 아래와 같다.

- 요약 대상 입력은 `comparison.json`, 모델별 `summary.json`, stress 결과 집계 JSON 으로 고정
- 산출물은 `LLMTestTool_v2/results/runs/<timestamp>/reports/final-summary.md` 에 저장
- 같은 디렉터리에 `summary-metadata.json` 을 두어 어떤 모델이 요약을 생성했는지, 어떤 점수 기준으로 선택됐는지 기록
- 요약 생성 시에도 한 번에 한 모델만 기동한다

요약 LLM 선택 기준은 deterministic 하게 고정한다.

1. quick run 성공 모델만 후보로 둔다
2. 후보 중 `assistant` lane 이 성공한 모델을 우선한다
3. 그 안에서 `assistant.totalMs` 가 가장 낮은 모델을 선택한다
4. `assistant` lane 성공 모델이 없으면 `raw.totalMs` 가 가장 낮은 모델을 선택한다
5. 동률이면 `stress` 실패율이 더 낮은 모델, 그다음 model key lexical order 로 정한다

즉, 요약 LLM 은 "가장 품질이 좋아 보이는 모델" 이 아니라, 이번 benchmark 에서 실제 운영 비용이 가장 낮고 성공적으로 끝난 모델을 기준으로 자동 선택한다.

요약 요청 자체는 selected model 의 direct endpoint 를 우선 사용한다. 이유는 summary 생성 단계에서 Nanobot agent loop 오버헤드를 다시 섞지 않고, 순수하게 benchmark 결과만 근거로 문서를 만들기 위해서다.

`final-summary.md` 최소 포함 항목:

- 실행 timestamp, 대상 모델, prompt set, repeat count
- fastest startup / fastest benchmark / fastest total time winner
- fastest TTFT winner
- prompt token / completion token / total token 비교
- completion tokens/sec 비교
- raw 대비 assistant overhead 요약
- stress run 에서 가장 안정적이었던 모델과 가장 불안정했던 모델
- `lfm2` 대 `qwen36` 상세 비교 delta
- 100회 per-case 상세 비교 표 또는 appendix 링크
- category/language/difficulty 별 어떤 케이스에서 어떤 모델이 느리거나 빠른지에 대한 deterministic 해설
- 선택된 summary model 과 선택 근거
- 운영 관점 next action 2~4개

본문이 과도하게 길어지는 것을 막기 위해, `comparison.md` 는 summary + grouped insight + 상위 slow/fast outlier table 을 포함하고, 전체 100회 per-case 표는 별도 markdown 또는 CSV/JSONL appendix artifact 로 분리해 링크하는 방식을 기본으로 한다. 단, repeatCount 가 100 이하인 기본 stress run 에서는 full markdown table 을 함께 생성해도 된다.

### 4.7 phase-2 deferred model onboarding

`gemma4-e4b` 는 현재 phase-1 범위에서 제외하고, phase-2 deferred target 으로 둔다.

- phase-1 구현은 `lfm2`, `qwen36` 2개 모델만 기준으로 닫는다.
- Gemma onboarding 은 `local-models` 계층에 Gemma start/status/stop/benchmark 가 준비된 뒤 별도 slice 로 연다.
- phase-2 에서 필요하면 `gemma4-e4b` alias, benchmark lane, summary model 후보 편입을 함께 진행한다.

## 5. 현재 코드 anchor

현재 기준 직접 참고/영향 위치는 아래다.

- `/Volumes/ExtData/AI_Project/LLMTestTool/README.md`
- `/Volumes/ExtData/AI_Project/LLMTestTool/supported-models.md`
- `/Volumes/ExtData/AI_Project/LLMTestTool/llmtesttool.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool/runners/openclaw/*`
- `/Volumes/ExtData/Nanobot/source/nanobot/api/server.py`
- `/Volumes/ExtData/Nanobot/source/nanobot/cli/commands.py`
- `/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh`
- `/Volumes/ExtData/Nanobot/infra/scripts/local-models/run-local-model-benchmark.sh`
- `/Volumes/ExtData/Nanobot/infra/scripts/local-models/status-local-model-services.sh`
- `/Volumes/ExtData/Nanobot/infra/scripts/local-models/start-local-model-services.sh`
- `/Volumes/ExtData/Nanobot/infra/scripts/local-models/stop-local-model-services.sh`
- `/Volumes/ExtData/Nanobot/docs/planning/todo.md`
- `/Volumes/ExtData/Nanobot/docs/planning/execution-backlog/README.md`

새 작업공간 기준 초기 대상은 아래로 본다.

- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/README.md`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/supported-models.md`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/llmtesttool.mjs` 또는 동등한 단일 CLI 엔트리
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/runners/nanobot/*`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/models/*/*/`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/compare/*/reports/`

## 6. 작업 분해

### Step 1. `LLMTestTool_v2` 구조와 naming 고정

- `v2` 저장소의 목적을 `Nanobot sequential benchmark harness` 로 고정
- OpenClaw 관련 용어와 경로를 초기 README/CLI scope 에서 제거
- 모델 catalog 를 `lfm2`, `qwen36` 2개 target 중심으로 정리

### Step 2. Nanobot lifecycle bridge 추가

- `local-models.sh` 를 호출하는 thin wrapper 또는 helper 추가
- `list/status/start/stop/benchmark` 호출 결과를 구조화 데이터로 정규화
- 실패 시 어떤 단계에서 막혔는지 run-level reason code 를 남기도록 설계

### Step 3. raw + assistant lane 추가

- raw lane 은 `local-models.sh benchmark` 결과를 직접 수집
- assistant lane 은 `nanobot serve` 가 노출하는 `/v1/chat/completions` 를 호출
- 두 lane 이 공통 summary schema 로 합쳐지도록 정리

### Step 4. sequential benchmark runner 추가

- 기본 run 은 모델당 1회 benchmark 만 실행
- 이전 모델이 완전히 내려간 뒤 다음 모델을 시작
- startup/shutdown 시간을 별도 측정

### Step 5. stress harness 추가

- prompt set catalog 를 정의하고 repeat 실행을 지원
- long-run 에서 모델은 여전히 하나씩만 기동
- stress 결과를 quick run 과 분리 저장

### Step 6. comparison report 추가

- 한 번의 batch run 이후 모델별 결과를 표 형태로 요약
- `lfm2` 와 `qwen36` 를 하나의 compare run 으로 묶는 상세 delta report 추가
- markdown report 와 JSON report 를 함께 남길지 결정
- run 결과를 Nanobot status-summary 에 쉽게 인용할 수 있게 출력 포맷 고정

latest scope 에서는 아래 세 하위 작업을 같이 포함한다.

- prompt set 각 항목을 `caseId/category/language/difficulty/text` 구조로 확장
- per-case raw row normalization 과 compare payload 내 `cases[]` 비교 배열 추가
- TTFT/token/throughput 기준 상세 markdown table 과 grouped insight section 추가

### Step 7. final LLM summary 추가

- benchmark 결과를 입력으로 final Markdown summary 를 생성
- best-performing model selection rule 을 코드와 문서에서 같은 기준으로 고정
- `reports/final-summary.md` 와 `summary-metadata.json` 저장

### Step 8. 문서와 운영 경로 동기화

- `LLMTestTool_v2/README.md`, `supported-models.md` 작성
- Nanobot planning/todo 에 새 benchmark slice 연결
- 구현 후 status-summary 로 이어질 기록 포맷 초안 명시
- 새 결과 디렉터리 규칙을 README 와 예시 경로에 반영

## 7. 실제 코드 작업 backlog

### 7.1 workspace bootstrap slice

- `LLMTestTool_v2` 루트 CLI 엔트리 정의
- `results/`, `runners/nanobot/`, `scripts/_shared/` 필요 여부 판단
- `.gitignore` 와 실행 launcher 기본값 정리

### 7.2 catalog slice

- `lfm2`, `qwen36` model metadata 정의
- Nanobot target name, display label, expected port, runtime kind 정리
- 추후 모델 추가를 위한 data-driven catalog 최소 뼈대 확보

### 7.3 lifecycle measurement slice

- `start` 요청 시각, endpoint ready 시각, `stop` 완료 시각 측정
- `status` polling timeout 과 실패 출력 형식 정의
- benchmark 결과를 lifecycle 측정값과 merge 하는 summary builder 추가

### 7.4 assistant lane slice

- Nanobot API server bootstrap helper 추가
- `POST /v1/chat/completions` payload/response 정규화
- agent loop 경유 latency 와 usage snapshot 수집

### 7.5 sequential execution slice

- default `run all` semantics 를 순차 실행으로 구현
- default count=`1` 고정, 필요 시 override 가능하되 help 에서 기본값 강조
- 이전 모델 cleanup 실패 시 다음 모델로 넘어갈지 즉시 중단할지 정책 결정

### 7.6 paired compare slice

- `lfm2` 와 `qwen36` 두 대상을 하나의 run 으로 묶는 compare command 추가
- left/right 공통 metric, delta, winner 계산 추가
- compare 결과를 별도 상세 markdown/json artifact 로 저장

### 7.7 stress harness slice

- prompt set 파일 포맷 정의
- repeat 횟수와 cool-down 간격 옵션 정의
- 실패율과 worst-case latency 집계 추가

### 7.8 reporting slice

- `comparison.json` 생성
- `comparison.md` 또는 CLI 표 출력 생성
- 가장 빠른 startup, 가장 빠른 benchmark, 가장 짧은 total time 을 한 번에 보이도록 요약
- left/right delta 와 winner summary 추가

### 7.9 final summary slice

- summary model selection helper 추가
- benchmark artifact 를 prompt input 으로 정리하는 summary payload builder 추가
- `reports/final-summary.md` 와 `summary-metadata.json` writer 추가

### 7.10 docs slice

- OpenClaw 미사용 원칙 명시
- `Nanobot infra/scripts/local-models` 연계 방식 설명
- `nanobot serve` 기반 assistant benchmark lane 설명
- final summary Markdown 저장 위치와 summary model 선택 규칙 설명
- `results/models/...` 와 `results/compare/...` path 규칙 설명
- 결과 파일을 어떻게 읽는지 예시 추가

## 8. 검증 계획

권장 검증 순서는 아래다.

1. `cd /Volumes/ExtData/Nanobot && ./infra/scripts/local-models/local-models.sh list`
2. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool_v2 help`
3. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool_v2 models`
4. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool_v2 run lfm2`
5. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool_v2 run qwen36`
6. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool_v2 compare lfm2 qwen36`
7. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool_v2 assistant all`
8. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool_v2 stress qwen36 --prompt-set standard --repeat 10`
9. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool_v2 run all`
10. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool_v2 summarize <compare-run-id>`
11. 결과물 `results/models/...`, `results/compare/.../comparison.json`, `comparison.md`, `reports/final-summary.md` 확인

focused 운영 검증은 아래를 포함한다.

- 각 모델 실행 전후 `./infra/scripts/local-models/local-models.sh status all`
- run 종료 후 모든 대상 모델이 `not-loaded / down` 상태인지 확인
- benchmark 결과의 `avgLatencyMs` 와 lifecycle 측정값이 summary 에 함께 기록됐는지 확인
- assistant lane 결과에 `usage` 와 stop reason 이 함께 기록됐는지 확인
- stress lane 결과에 prompt set 별 success/failure 와 worst-case latency 가 분리 기록됐는지 확인
- compare run 결과에 left/right delta 와 winner 가 함께 기록됐는지 확인
- `summary-metadata.json` 에 선택된 summary model, selection reason, source run timestamp 가 기록됐는지 확인
- `reports/final-summary.md` 가 LLM 생성 요약 형식으로 저장됐는지 확인
- `LLMTestTool_v2` 내부에 OpenClaw 경로, `~/.openclaw/*`, `runners/openclaw` 참조가 없는지 확인

## 9. 완료 기준

1. `/Volumes/ExtData/AI_Project/LLMTestTool_v2` 는 OpenClaw 비교 경로 없이 Nanobot benchmark 전용 구조로 동작한다.
2. 기본 실행은 어떤 경우에도 한 번에 한 모델만 기동한다.
3. `lfm2`, `qwen36` 각각에 대해 기본 1회 benchmark 와 startup/shutdown 시간 비교가 가능하다.
4. `lfm2` 와 `qwen36` 를 하나의 compare run 으로 묶어 상세 delta report 를 생성할 수 있다.
5. raw lane 과 assistant lane 이 모두 동작하고, stress lane 이 prompt set 반복 결과를 남긴다.
6. run 종료 후 가장 성능이 좋은 benchmark 대상 모델이 최종 결과를 요약해 `reports/final-summary.md` 를 생성한다.
7. 결과 디렉터리는 모델명/비교대상/프로파일이 먼저 드러나는 path 규칙을 따른다.
8. run 종료 후 모델이 자동으로 내려가고, 비교 결과가 JSON/Markdown 또는 동등한 보고 형식으로 남는다.
9. Nanobot planning 문서가 새 benchmark 방향과 일치한다.

## 10. 다음 단계 연결

- 구현 시작 전 `planning/todo.md` 에 active item 으로 승격
- 구현 후 첫 실제 비교 결과는 `operations/status-summary/` 날짜형 메모에 기록
- phase-2 에서 필요하면 `gemma4-e4b` onboarding, websocket lane, multi-session 경쟁 부하를 별도 slice 로 연다