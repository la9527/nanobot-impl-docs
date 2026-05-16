# 10 Nanobot LLMTestTool_v2 Answer Evaluation Phase 1

## 0. 2026-05-16 재정렬 메모

이 문서는 최초에는 `results/models/...`, `results/compare/...` 중심 artifact 를 전제로 쓴 설계안이었지만, 2026-05-16 기준으로 `11-nanobot-llmtesttool-v2-usability-results-reporting-phase1.md` 의 core usability slice 가 대부분 반영되면서 answer evaluation 도 `run 중심 구조` 를 기준으로 다시 읽는 편이 맞다.

현재 이 문서는 아래 전제를 기준으로 본다.

- answer evaluation 은 `11` 문서 다음의 실제 active candidate 다.
- 평가 입력은 가능하면 `results/runs/<run-id>/` 아래 artifact 를 우선 사용한다.
- 특히 phase-1 answer quality 평가는 `stress` lane row artifact 를 source-of-truth 로 삼는 편이 가장 현실적이다.
- legacy `results/models/...`, `results/compare/...` 는 초기 migration/fallback 입력으로만 유지한다.

즉, 이 문서는 이제 `run-manifest.json + stress-<prompt-set>-results.jsonl + latest/report entrypoint` 가 이미 있는 현재 구조 위에 평가 계층을 붙이는 설계 문서다.

## 0.1 2026-05-16 구현 상태 갱신

2026-05-16 기준으로 이 문서의 phase-1 범위는 아직 전부 끝난 것은 아니지만, deterministic answer evaluation 을 시작점으로 하는 core slice 는 이미 들어갔다.

현재 반영된 항목:

- `lib/eval-layout.mjs`, `lib/eval-artifacts.mjs`, `lib/eval-rules.mjs`, `lib/eval-reporting.mjs` 추가
- `lib/eval-catalog.mjs`, `lib/eval-judge.mjs` 추가
- `evaluate <run-id> --model <key> --prompt-set <id>` 구현
- `evaluate-compare <run-id> --prompt-set <id>` 구현
- `results/runs/<run-id>/evals/...` artifact 저장 구현
- `results/reports/report-eval-...`, `results/reports/latest-eval-...` pointer 저장 구현
- `reports latest-eval` entrypoint 조회 구현
- `--judge-provider openai`, `--judge-model <id>` option 기반 judge-aware evaluation 구현
- explicit pairwise judge prompt 기반 `winner`, `confidence`, `reason` 재산출 구현
- bundled prompt set 전체(`standard`, `bilingual-mixed`, `logic-intensive`, `programming-intensive`) prompt 에 `referenceAnswer`, `judgeRubric` 반영
- evaluation metadata 에 `requestCount`, `pricing`, `estimatedCostUsd`, `promptSetCoverage` provenance 반영
- `evaluation.md`, `pairwise-evaluation.md` 에 `Judge Provenance` 섹션 반영
- `Judge Provenance` 아래에 `Judge Interpretation` 해석 문구 반영
- `Outcome Interpretation` 섹션에서 pass/fail stability, judge-vs-deterministic gap, pairwise split 해석 문구 반영
- `openai/gpt-5.4-2026-03-05` judge pricing catalog fallback 반영, OpenAI API 가격 페이지 기준 `GPT-5.4` 입력 `$2.50 / 1M`, 출력 `$15.00 / 1M`, env override 우선 유지
- `standard`, `bilingual-mixed`, `logic-intensive`, `programming-intensive` prompt set 에 `expectedLanguage`, `mustInclude`, `formatRules`, `pairwiseFocus` 메타데이터 반영

현재 기준에서 꼭 해야 하는 남은 핵심 항목은 없다.

아래 두 항목은 필요해질 때만 검토하면 되는 후순위 고도화 옵션으로 둔다.

- 기본 `openai/gpt-5.4-2026-03-05` 외 다른 judge provider/model 조합까지 pricing catalog 를 확장하는 작업
- `Judge Interpretation` / `Outcome Interpretation` 규칙을 category, language, grouped failure pattern 같은 운영 기준까지 넓히는 작업

즉 현재 상태는 `phase-1 core 구현 완료` 로 보는 편이 맞고, 남은 것은 필수 미구현이 아니라 선택적 고도화다.

## 1. 문서 목적

이 문서는 `/Volumes/ExtData/AI_Project/LLMTestTool_v2` 가 이미 보존하고 있는 benchmark 결과 artifact 를 재사용해, `속도 비교` 뿐 아니라 `답변 품질 평가` 까지 함께 다루는 phase-1 구현 설계안을 정리하기 위한 실행 문서다.

이번 단계의 목표는 세 가지다.

- 기존 `results/runs/<run-id>/models/<model>/stress-<prompt-set>-results.jsonl` 에 남아 있는 prompt/response/raw metric 을 입력으로 삼아 오프라인 answer evaluation layer 를 추가하기
- `evaluate`, `evaluate-compare` 같은 명시적 CLI surface 를 통해 run 기준 단독 평가와 left/right pairwise 평가를 분리 실행할 수 있게 만들기
- prompt set JSON 에 평가용 metadata 를 추가해, 언어/형식/필수 내용/정답 reference/rubric 을 prompt 단위로 선언 가능한 구조로 정리하기

이번 문서는 앞서 검토한 세 구현 축을 그대로 phase-1 범위로 고정한다.

1. 평가 지표와 결과 schema 설계
2. `evaluate` / `evaluate-compare` 명령 추가 설계
3. prompt set metadata 확장 설계

## 2. 왜 지금 필요한가

현재 `LLMTestTool_v2` 는 startup, TTFT, token, throughput, worst latency, grouped insight 같은 성능 지표는 잘 남기고 있지만, 실제 답변이 `얼마나 잘 답했는지` 는 사람이 raw row 를 열어 직접 읽어야 한다.

현재 구조의 제약은 아래 다섯 가지다.

1. `stress` lane 결과에는 `promptText`, `content`, `language`, `difficulty`, `promptCategory`, `caseId` 가 모두 남지만, 그 답변이 prompt 요구를 잘 따랐는지 기계적으로 판단하는 layer 가 없다.
2. 현재 compare/report 는 latency winner 중심이라, 운영자가 `속도는 빠르지만 답변 품질이 떨어지는 모델` 을 구분하기 어렵다.
3. `bilingual-mixed`, `programming-intensive` 같은 prompt set 은 언어 일치, 코드 형식, 설명 completeness 같이 task-specific 평가가 필요한데, 지금 prompt JSON 에는 이를 선언할 metadata field 가 없다.
4. summary 단계는 OpenAI judge 를 이미 쓰고 있지만, summary 생성과 answer evaluation 이 분리돼 있지 않아 `평가 점수`, `평가 이유`, `pairwise 품질 승자` 를 구조화 artifact 로 남기지 못한다.
5. 현재 benchmark repeat 결과를 regression 기준선으로 쓰려면 성능 지표뿐 아니라 품질 점수도 같은 `run artifact` 체계 안에 들어와야 한다.

외부 문서 기준으로도 방향은 일치한다.

- OpenAI Evals 는 `dataset + testing criteria` 기반 오프라인 eval 을 권장한다.
- LangSmith 는 offline evaluation 에서 dataset, evaluator, pairwise comparison, regression 비교를 기본 흐름으로 둔다.
- DeepEval 과 Ragas 는 referenceless metric 과 task-specific rubric metric 조합, 그리고 metric 수를 3~5개 수준으로 제한하는 방식을 권장한다.

따라서 phase-1 에서는 별도 대형 eval platform 을 바로 도입하기보다, 현재 Node 기반 harness 위에 `경량 오프라인 answer evaluation layer` 를 붙이는 것이 가장 현실적이다.

## 3. phase-1 범위

포함할 것:

- 모델 단독 실행 결과를 평가하는 `evaluate <run-id> --model <key> --prompt-set <id>` command
- compare 품질 결과를 평가하는 `evaluate-compare <run-id> --prompt-set <id>` command
- deterministic rule-based evaluator 추가
- OpenAI judge 기반 rubric evaluator 추가
- pairwise left/right 답변 비교 evaluator 추가
- 평가 결과 JSON/Markdown artifact 저장 규칙 정의
- prompt set JSON 에 evaluation metadata field 추가
- README 와 planning 문서에 새 평가 흐름 반영

제외할 것:

- LangSmith, DeepEval, Ragas 전체 runtime 도입
- live online evaluation 또는 production traffic sampling
- human review UI 또는 annotation 도구 구축
- multi-turn conversation judge
- tool-calling agent correctness 평가
- answer evaluation 결과를 즉시 smart-router routing policy 에 반영하는 자동 튜닝

## 4. 핵심 설계 결정

### 4.1 answer evaluation 은 benchmark generation 이후의 오프라인 layer 로 둔다

phase-1 에서 평가는 모델 응답 생성 시점에 inline 으로 끼워 넣지 않고, 이미 저장된 artifact 를 읽어 후처리하는 오프라인 layer 로 둔다.

이렇게 두는 이유는 아래와 같다.

- benchmark lane 의 실행 시간과 평가 시간/비용을 분리할 수 있다.
- 같은 raw artifact 에 대해 judge model, rubric, threshold 를 바꿔 재평가할 수 있다.
- `repeat 100` 결과를 regression 기준선으로 다시 읽어 일관된 score 를 남길 수 있다.
- local benchmark 실패와 eval judge 실패를 run-level reason 으로 분리 저장할 수 있다.

즉 phase-1 answer evaluation 의 source-of-truth 는 새로운 runtime trace 가 아니라 기존 artifact 다.

우선 입력은 아래처럼 잡는 편이 맞다.

- run-level input: `results/runs/<run-id>/run-manifest.json`
- model-level input: `results/runs/<run-id>/models/<model>/stress-<prompt-set>-results.jsonl`
- pairwise input: 같은 `run-id` 아래 left/right model stress row 의 `caseId` alignment

legacy fallback 입력은 아래처럼만 유지한다.

- fallback model input: `results/models/<model>/<lane>/<profile>-<timestamp>/results.jsonl`
- fallback compare input: `results/compare/<left>__vs__<right>/<profile>-<timestamp>/comparison.json`

### 4.2 평가 방식은 세 층으로 분리한다

phase-1 에서는 아래 세 evaluator 를 함께 둔다.

1. deterministic evaluator
2. rubric-based judge evaluator
3. pairwise comparison evaluator

#### deterministic evaluator

LLM judge 없이 계산 가능한 rule-based 평가다.

예시:

- expected language 일치 여부
- code block required 여부
- bullet/checklist required 여부
- `mustInclude` keyword 포함 여부
- `mustNotInclude` keyword 위반 여부
- minimum/maximum length 제약
- required JSON field presence

이 평가는 비용이 거의 없고 재현성이 높으므로, phase-1 기본 score 의 일부로 반드시 포함한다.

#### rubric-based judge evaluator

OpenAI judge 를 이용해 `promptText + actualOutput + optional reference/rubric` 를 보고 아래 항목을 0~1 또는 1~5 scale 로 평가한다.

- `instructionFollowing`
- `answerRelevancy`
- `correctness`
- `completeness`
- `clarity`

judge 는 phase-1 에서 `summary-catalog.mjs` 와 유사한 provider config 를 따르는 별도 eval provider config 로 두는 것이 맞다. 초기 기본 judge provider 는 `openai`, 기본 judge model 은 `gpt-5.4-2026-03-05` 로 둔다.

#### pairwise comparison evaluator

동일 `caseId` 에 대해 left/right 답변을 동시에 읽고 `어느 답변이 더 나은지` 와 그 이유를 판단한다.

이 평가가 필요한 이유는 `절대 점수` 와 실제 운영 decision 이 다를 수 있기 때문이다. phase-1 pairwise output 은 아래 세 값을 남긴다.

- `winner`: `left`, `right`, `tie`
- `confidence`: 0~1
- `reason`: 비교 근거 한두 문장

### 4.3 metric 수는 5개 이하로 시작한다

DeepEval/Ragas 권장과 현재 도구 목적을 함께 보면, phase-1 에서 metric 을 과도하게 늘리는 것은 좋지 않다. 초기 metric set 은 아래 다섯 개로 고정하는 편이 맞다.

- `instructionFollowing`
- `answerRelevancy`
- `correctness`
- `completeness`
- `pairwisePreference`

`clarity` 는 judge reason 에 포함하되, top-level aggregated KPI 로는 immediate 필수 항목이 아니다. 먼저 위 다섯 개로 score pipeline 을 닫고, 필요하면 phase-2 에서 확장한다.

### 4.4 prompt category 별로 평가 기준을 다르게 둔다

모든 prompt 를 같은 rubric 한 장으로 평가하면 noise 가 커진다. phase-1 에서는 category 기준으로 evaluator profile 을 분기한다.

- `operations`
  - 절차 설명, 핵심 포인트 누락, 운영 용어 적합성 위주 평가
- `analysis`
  - 질문 적합성, 비교 포인트 누락, reasoning coherence 위주 평가
- `logic`
  - 절차적 정확성, 단계 completeness, contradiction 여부 위주 평가
- `programming`
  - code block 존재, 언어/함수 형식, 핵심 API/예외 처리 포함 여부를 deterministic + judge 혼합 평가

즉, `programming` 은 code-shape deterministic check 비중이 더 높고, `operations/analysis` 는 rubric judge 비중이 더 높다.

### 4.5 prompt set metadata 를 평가 친화적으로 확장한다

현재 prompt set entry 는 아래 필드만 갖는다.

- `id`
- `category`
- `language`
- `difficulty`
- `text`

phase-1 에서는 아래 필드를 선택적으로 추가한다.

- `expectedLanguage`
- `mustInclude`
- `mustNotInclude`
- `formatRules`
- `referenceAnswer`
- `judgeRubric`
- `pairwiseFocus`

각 필드의 의도는 아래와 같다.

- `expectedLanguage`: 답변 언어 검사에 사용
- `mustInclude`: 반드시 포함돼야 하는 개념/용어/키워드
- `mustNotInclude`: 금지 표현 또는 피해야 할 포맷
- `formatRules`: 예를 들면 `codeBlockRequired`, `listRequired`, `maxParagraphs`, `jsonShape`
- `referenceAnswer`: reference-based correctness 평가에 사용 가능한 canonical answer
- `judgeRubric`: prompt 개별 judge prompt 보강 설명
- `pairwiseFocus`: 두 답변 비교 시 어떤 축을 더 중시할지 명시

모든 prompt 에 `referenceAnswer` 를 강제하지는 않는다. referenceless judge 를 허용하되, deterministic rule 이 최대한 많이 걸리도록 `mustInclude`, `formatRules` 를 우선 채운다.

### 4.6 CLI surface 는 `run-id` 기준 `evaluate` 와 `evaluate-compare` 로 분리한다

phase-1 command surface 는 아래 두 개로 고정하는 편이 맞다.

- `./llmtesttool evaluate <run-id> --model <lfm2|qwen36> --prompt-set <id> [--judge-provider openai] [--judge-model ...]`
- `./llmtesttool evaluate-compare <run-id> --prompt-set <id> [--judge-provider openai] [--judge-model ...]`

`evaluate` 는 run 디렉터리 아래 model-level stress artifact 를 입력으로 받아 case별 단독 점수를 만든다.

출력:

- per-case rule score
- per-case judge score
- aggregated score by category/language/difficulty
- fail/pass counts by threshold

`evaluate-compare` 는 같은 `run-id` 아래 left/right model stress row 를 `caseId` 기준으로 맞춰 pairwise evaluation 을 수행한다.

출력:

- case별 pairwise winner
- aggregated pairwise win counts
- category/language/difficulty 별 winner skew
- left/right model 별 answer-quality score summary

phase-1 에서는 `bundle --with-eval` 같은 inline auto-eval 은 열지 않는다. 이유는 benchmark 실행과 품질 평가를 분리해야 디버깅과 비용 통제가 쉽기 때문이다.

보조 정책은 아래처럼 둔다.

- `evaluate` 는 phase-1 에서 `stress` lane 전용으로 시작한다.
- `evaluate-compare` 는 기본적으로 suite/bundle run 의 `run-id` 를 받는다.
- legacy compare run ref 를 직접 받는 fallback 은 두되, README 와 help 에서는 `run-id` 흐름을 먼저 설명한다.

### 4.7 결과 저장은 `results/runs/<run-id>/evals/` 아래에 붙인다

성능 결과와 품질 평가 결과는 run 내부에서 계층을 분리하는 편이 맞다.

권장 경로:

- model eval JSON: `results/runs/<run-id>/evals/<prompt-set>/<model>/evaluation.json`
- model eval Markdown: `results/runs/<run-id>/evals/<prompt-set>/<model>/evaluation.md`
- model eval metadata: `results/runs/<run-id>/evals/<prompt-set>/<model>/evaluation-metadata.json`
- compare eval JSON: `results/runs/<run-id>/evals/<prompt-set>/compare/pairwise-evaluation.json`
- compare eval Markdown: `results/runs/<run-id>/evals/<prompt-set>/compare/pairwise-evaluation.md`
- compare eval metadata: `results/runs/<run-id>/evals/<prompt-set>/compare/evaluation-metadata.json`
- top-level flat report: `results/reports/report-eval-<left>_vs_<right>-<runTimestamp>-<prompt-set>.md`
- latest eval pointer: `results/reports/latest-eval-<left>_vs_<right>.md`

`summary-metadata.json` 과 유사하게, 평가에도 `evaluation-metadata.json` 을 둔다.

최소 포함 항목:

- source artifact path
- judge provider/model
- deterministic evaluator version
- rubric version
- total case count
- pass/fail threshold
- token usage / judge cost reference
- generatedAt

### 4.8 phase-1 judge provider 는 OpenAI external judge 우선

현재 local model 은 benchmark 대상이므로 judge 로 쓰면 evaluator drift 가 커질 수 있다. phase-1 에서는 summary 와 동일하게 external judge 를 우선 기본값으로 둔다.

권장 기본값:

- provider: `openai`
- model: `gpt-5.4-2026-03-05`
- env key: `OPEN_API_KEY`, fallback `OPENAI_API_KEY`

단, provider selection abstraction 은 summary path 와 분리하는 것이 좋다. summary judge 와 answer eval judge 가 추후 다른 model/provider 를 쓸 수 있기 때문이다.

## 5. 현재 코드 anchor

현재 기준 직접 영향을 받는 위치는 아래다.

- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/llmtesttool.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/stress-lane.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/prompt-sets.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/reporting.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/comparison-summary.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/summary-catalog.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/external-summary.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/result-layout.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/suite-artifacts.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/report-entrypoints.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/runs/*/run-manifest.json`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/runs/*/models/*/stress-*-results.jsonl`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/runs/*/compare/comparison.json`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/compare/*/*/comparison.json`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/prompt-sets/*.json`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/README.md`

새 작업공간 기준 추가 대상은 아래로 본다.

- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/eval-catalog.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/eval-rules.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/eval-judge.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/eval-pairwise.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/eval-reporting.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/lib/eval-layout.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/tests/eval-*.test.mjs`
- `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/runs/*/evals/`

## 6. 작업 분해

### Step 1. evaluation metadata schema 와 artifact 경로 고정

- `evaluation.json`, `evaluation.md`, `pairwise-evaluation.json`, `pairwise-evaluation.md`, `evaluation-metadata.json` 구조 정의
- run-level eval 과 compare-level eval 의 path rule 정의
- `results/reports/report-eval-...` / `latest-eval-...` flat report 규칙 정의

상태: 완료

### Step 2. deterministic evaluator 추가

- 언어 일치, 포맷 규칙, 필수/금지 키워드 검사 추가
- `programming` category 에 대한 code block / shape check 추가
- prompt set metadata 가 없는 구형 prompt 에 대한 fallback 정책 정의

상태: 완료

### Step 3. judge evaluator 추가

- judge provider/model config 분리
- rubric prompt template 정의
- case별 score, reason, pass/fail 계산 추가

상태: 부분 완료
OpenAI judge provider config, single-answer judge prompt, case별 judge score 와 summary-level judgeAverageScore, metadata token usage 까지는 반영됐다. 다만 richer rubric 관리와 cost estimation 은 아직 남아 있다.

### Step 4. pairwise evaluator 추가

- same `caseId` left/right alignment
- case별 winner, confidence, reason 저장
- aggregated winner count 와 grouped insight 생성

상태: 부분 완료
현재는 deterministic score 기반 pairwise winner/confidence/reason 과 judge-enriched model score 반영까지는 구현됐고, explicit pairwise judge prompt 로 winner/reason 을 뽑는 흐름은 아직 없다.

상태 업데이트: explicit pairwise judge prompt 로 `winner`, `confidence`, `reason` 을 직접 생성하는 흐름까지 반영됐고, bundled prompt set 전체에 실제 `referenceAnswer` / `judgeRubric` 가 채워졌다. 또한 evaluation metadata 와 markdown report 모두에 cost/provenance 가 들어가며, report 는 `Judge Interpretation` 문구로 pricing source, coverage completeness, pairwise judge 전략을 바로 읽을 수 있고 `Outcome Interpretation` 문구로 pass threshold 안정성, score gap, pairwise split 을 바로 해석할 수 있다. 기본 `openai/gpt-5.4-2026-03-05` judge 는 catalog fallback pricing 을 사용한다. 남은 부분은 pricing catalog 범위를 넓히고 interpretation 규칙을 운영 문서 수준으로 정리하는 일이다.

### Step 5. CLI command surface 추가

- `evaluate <run-id> --model <key> --prompt-set <id>` 구현
- `evaluate-compare <run-id> --prompt-set <id>` 구현
- help, REPL alias, JSON output 정리

상태: 부분 완료
`evaluate`, `evaluate-compare`, help, JSON output, `reports latest-eval`, `--judge-provider`, `--judge-model` 까지는 반영됐다.

### Step 6. prompt set metadata 확장

- `bilingual-mixed`, `standard`, `logic-intensive`, `programming-intensive` JSON 을 새 필드 구조로 확장
- category별 sample rubric 과 format rule 추가
- README 와 문서 예시 갱신

상태: 부분 완료
deterministic evaluator 가 바로 읽는 `expectedLanguage`, `mustInclude`, `formatRules`, `pairwiseFocus` 는 반영됐다. `referenceAnswer`, `judgeRubric` 의 실제 세트별 채움은 judge slice 와 함께 이어서 보는 편이 맞다.

### Step 7. reporting/documentation 추가

- evaluation markdown renderer 추가
- compare 품질 report 와 benchmark summary 를 연결하는 링크/경로 설명 추가
- planning/status-summary 에 인용 가능한 결과 포맷 정의

상태: 부분 완료
evaluation markdown renderer, eval flat report, latest pointer, README/help 반영은 완료됐다. status-summary 연결과 judge 설명 강화는 후속 항목이다.

## 7. 실제 코드 작업 backlog

### 7.1 eval catalog slice

- judge provider config 를 담는 `eval-catalog.mjs` 추가
- summary provider 와 eval judge provider 차이를 문서로 고정
- default judge model 과 env key resolution 전략 정리

### 7.2 eval rules slice

- `expectedLanguage`, `mustInclude`, `mustNotInclude`, `formatRules` 파서 추가
- deterministic score calculator 구현
- case별 failure detail 구조 정의

### 7.3 eval judge slice

- OpenAI judge request helper 추가
- rubric prompt builder 추가
- score normalization 과 threshold pass/fail 처리 추가

### 7.4 pairwise slice

- left/right case aligner 추가
- pairwise prompt template 추가
- `winner`, `confidence`, `reason` 집계와 grouped aggregation 추가

### 7.5 command slice

- `llmtesttool.mjs` 에 `evaluate`, `evaluate-compare` command 추가
- interactive help 와 history/replay 에 새 surface 반영
- `--judge-provider`, `--judge-model`, `--threshold`, `--model`, `--prompt-set` 옵션 추가

### 7.6 prompt-set schema slice

- prompt JSON field schema 확장
- 기존 prompt loader fallback 유지
- `programming` / `bilingual` / `logic` 대표 rule 추가

### 7.7 eval reporting slice

- per-case evaluation markdown table 추가
- aggregated quality summary 와 pairwise summary renderer 추가
- `report-eval-<left>_vs_<right>-<timestamp>-<prompt-set>.md` writer 추가
- `latest-eval-<left>_vs_<right>.md` pointer writer 추가

### 7.8 docs slice

- README 에 eval command, artifact path, judge provider 설명 추가
- planning/execution-backlog index 에 새 slice 연결
- 필요 시 `docs/guide` 또는 `operations/status-summary` 인용 규칙 보강

## 8. 검증 계획

권장 검증 순서는 아래다.

1. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && npm test`
2. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool evaluate <run-id> --model lfm2 --prompt-set standard`
3. `cd /Volumes/ExtData/AI_Project/LLMTestTool_v2 && ./llmtesttool evaluate-compare <run-id> --prompt-set standard`
4. `results/runs/<run-id>/evals/<prompt-set>/<model>/evaluation.json` 과 `evaluation.md` 확인
5. `results/runs/<run-id>/evals/<prompt-set>/compare/pairwise-evaluation.json` 과 `pairwise-evaluation.md` 확인
6. `results/reports/report-eval-<left>_vs_<right>-<timestamp>-<prompt-set>.md` 확인
7. `results/reports/latest-eval-<left>_vs_<right>.md` 확인
8. prompt set metadata 가 deterministic evaluator 에 실제 반영되는지 sample case 로 검증
9. `--judge-provider openai` 흐름에서 single-answer judge score 와 metadata token usage 확인
10. explicit pairwise judge slice 반영 이후 judge reason 기반 winner 를 별도 검증

focused 검증은 아래를 포함한다.

- bilingual prompt 에서 expected language mismatch 가 deterministic fail 로 잡히는지 확인
- programming prompt 에서 code block required rule 이 동작하는지 확인
- `mustInclude` 누락 시 failure detail 이 남는지 확인
- judge evaluator 가 case별 `score`, `reason`, `isPass` 를 남기는지 확인
- pairwise evaluator 가 `winner`, `confidence`, `reason` 를 남기는지 확인
- grouped aggregation 이 `category/language/difficulty` 별로 생성되는지 확인
- `evaluation-metadata.json` 에 judge provider/model/usage/source artifact path 가 기록되는지 확인

## 9. 완료 기준

1. `LLMTestTool_v2` 는 성능 benchmark 결과와 별도로 answer quality evaluation artifact 를 생성할 수 있다.
2. `evaluate` command 로 model-level answer quality score 를 계산할 수 있다.
3. `evaluate-compare` command 로 left/right pairwise 품질 승자를 계산할 수 있다.
4. deterministic rule-based 평가와 judge-based rubric 평가가 함께 저장된다.
5. prompt set JSON 에 평가용 metadata 를 선언할 수 있고, evaluator 가 이를 실제 사용한다.
6. 평가 결과는 `results/runs/<run-id>/evals/...` 와 `results/reports/report-eval-...` 경로에 저장된다.
7. README 와 planning 문서가 새 answer evaluation 흐름을 설명한다.

## 10. 다음 단계 연결

- 구현 시작 전 `09-nanobot-llmtesttool-v2-sequential-benchmark-phase1.md` 와 `11-nanobot-llmtesttool-v2-usability-results-reporting-phase1.md` 를 함께 prerequisite 문서로 본다.
- phase-1 구현 후 첫 실측 결과는 `results/runs/<run-id>/evals/...` artifact 와 함께 `operations/status-summary/` 날짜형 메모에 기록한다.
- phase-2 에서는 human review workflow, multi-turn judge, routing policy feedback loop, LangSmith 또는 OpenAI Evals 외부 연계를 별도 slice 로 연다.
