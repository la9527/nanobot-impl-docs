# Nanobot Personal Assistant Execution Backlog

> 용어 메모: 이 디렉터리의 초기 기획 문서에는 `assistant`, `Dashboard`, `Settings`, `thread` 같은 예전 WebUI 표현이 남아 있을 수 있다. 현재 사용자 노출 용어 기준은 [guide/10-webui-terminology.md](../../guide/10-webui-terminology.md) 를 따른다.

상태:

- 이 디렉터리는 `새 구현 slice 를 정의하는 workplan index` 로 유지한다.
- 현재 `01` 부터 `06` 까지의 phase-1 문서는 구현 완료 범위를 설명하는 reference 로 본다.
- 다음 active 제품 작업을 새로 열면 `07-*` 이후 번호로 추가하고, 먼저 [../todo.md](../todo.md) 에서 우선순위를 명시한다.

이 디렉터리는 [concepts/overview.md](../concepts/overview.md)와 `docs/planning/concepts/*.md` 에서 정리한 내용을 실제 구현 순서로 내리기 위한 실행 백로그 문서 모음이다.

목표는 두 가지다.

- 이미 문서화된 개인비서 제품 레이어를 `무엇부터 구현할지` 우선순위 기준으로 고정하기
- 각 우선순위별로 바로 구현 착수 가능한 phase-1 workplan 을 준비하기

## 1. 선정 원칙

이번 실행 백로그는 아래 원칙으로 대상을 선정한다.

1. 이미 완료된 기반 위에 바로 붙일 수 있는가
2. WebUI control surface, continuity, approval 같은 기존 완료 항목을 재사용하는가
3. 사용자 체감 가치가 빠르게 나오는가
4. 새로운 대규모 인프라보다 product layer 정리가 먼저 가능한가
5. Kakao 같은 후순위 채널보다 core assistant experience를 먼저 강화하는가

## 2. 현재 정리된 phase-1 구현 대상

현재 문서로 정리된 phase-1 구현 대상은 아래다.

1. owner-aware control surface aggregate
2. owner memory + personal task backbone
3. Gmail read-only + draft flow
4. Gmail send approval flow
5. Calendar read/conflict/create flow
6. proactive briefing + quiet hours phase-1

현재 다음 active candidate 로는 아래 slice 를 열어 둔다.

- [07-webui-command-menu-and-dashboard-split-phase1.md](./07-webui-command-menu-and-dashboard-split-phase1.md)
- [08-local-model-runtime-unification-and-qwen36-phase1.md](./08-local-model-runtime-unification-and-qwen36-phase1.md)
- [09-nanobot-llmtesttool-v2-sequential-benchmark-phase1.md](./09-nanobot-llmtesttool-v2-sequential-benchmark-phase1.md)
- [10-nanobot-llmtesttool-v2-answer-evaluation-phase1.md](./10-nanobot-llmtesttool-v2-answer-evaluation-phase1.md)
- [11-nanobot-llmtesttool-v2-usability-results-reporting-phase1.md](./11-nanobot-llmtesttool-v2-usability-results-reporting-phase1.md)
- [12-nanobot-local-llm-control-and-smart-router-default-phase1.md](./12-nanobot-local-llm-control-and-smart-router-default-phase1.md)

## 3. 작업 순서

아래 순서는 최초 implementation order 기록으로 보존한다. 현재는 `01`~`06` 이 모두 completed reference 이며, 새 작업을 다시 열 때는 이 순서를 그대로 active queue 로 간주하지 않는다.

### 3.1 1순위

- [01-owner-aware-control-surface-phase1.md](./01-owner-aware-control-surface-phase1.md)

이 단계는 이미 구현된 WebUI status / continuity / approval visibility를 `assistant control surface` 수준으로 끌어올리는 작업이다.

### 3.2 2순위

- [02-owner-memory-and-task-backbone-phase1.md](./02-owner-memory-and-task-backbone-phase1.md)

이 단계는 owner profile, task state model, continuity aggregate를 실제 runtime/product layer에서 읽을 수 있는 최소 backbone을 정리한다.

### 3.3 3순위

- [03-gmail-readonly-draft-phase1.md](./03-gmail-readonly-draft-phase1.md)
- [04-gmail-send-approval-phase1.md](./04-gmail-send-approval-phase1.md)

이 단계는 개인비서 체감 가치가 큰 mailbox workflow를 action contract와 approval 흐름으로 구현한다.

### 3.4 4순위

- [05-calendar-read-conflict-create-phase1.md](./05-calendar-read-conflict-create-phase1.md)

Calendar는 Gmail 다음으로 value가 크지만, 입력 부족과 충돌 해석이 섞여 있어 mail pilot 다음으로 두는 편이 안정적이다.

### 3.5 5순위

- [06-proactive-briefing-and-quiet-hours-phase1.md](./06-proactive-briefing-and-quiet-hours-phase1.md)

proactive는 체감 가치는 크지만 annoyance risk도 커서, owner/task/action 기반이 잡힌 뒤에 제한적으로 올리는 편이 맞다.

### 3.6 다음 active candidate

- [07-webui-command-menu-and-dashboard-split-phase1.md](./07-webui-command-menu-and-dashboard-split-phase1.md)

### 3.7 현재 active local model candidate

- [08-local-model-runtime-unification-and-qwen36-phase1.md](./08-local-model-runtime-unification-and-qwen36-phase1.md)

이 단계는 기존 `infra/scripts/llama/` 운영 경로를 `local-models` 공용 계층으로 재정리하고, Qwen3.6 MLX runtime 과 focused benchmark 흐름을 함께 붙이는 인프라 중심 slice 다.

이 단계는 slash command 발견성과 `대시보드` / `새 채팅` surface 분리를 함께 정리하는 WebUI 중심 slice 다.

### 3.8 다음 benchmark harness candidate

- [09-nanobot-llmtesttool-v2-sequential-benchmark-phase1.md](./09-nanobot-llmtesttool-v2-sequential-benchmark-phase1.md)

이 단계는 기존 `LLMTestTool` 의 OpenClaw 결합을 걷어내고, `LLMTestTool_v2` 를 Nanobot `local-models` control plane 위에서 순차 benchmark 와 lifecycle timing 비교를 수행하는 도구로 다시 세우는 benchmark 전용 slice 다.

### 3.9 다음 answer evaluation candidate

- [10-nanobot-llmtesttool-v2-answer-evaluation-phase1.md](./10-nanobot-llmtesttool-v2-answer-evaluation-phase1.md)

이 단계는 `LLMTestTool_v2` 가 현재 저장하는 `results/runs/<run-id>/...` artifact, 특히 stress row 와 run manifest 를 기반으로 answer quality evaluation layer 와 pairwise judge flow 를 추가하는 benchmark 후처리/eval slice 다. 2026-05-16 기준으로 `11` usability/reporting slice 의 core 구현이 대부분 반영됐으므로, 이제 이 문서는 그 run 중심 구조 위에 eval 계층을 붙이는 next active candidate 로 읽는 편이 맞다.
현재는 deterministic eval, pairwise eval, prompt-set metadata, `latest-eval` entrypoint, OpenAI judge 기반 single-answer eval, explicit pairwise judge flow, bundled prompt-set rubric/reference 보강, pricing catalog fallback, markdown provenance 노출, interpretation 문구 생성까지 core 범위가 반영됐다. 최근에는 `Outcome Interpretation` 까지 추가돼 pass stability 와 pairwise split 해석이 report 본문에 바로 노출된다. pricing catalog 범위 확대와 interpretation 규칙 정교화는 현재 기준 필수 backlog 가 아니라 선택적 고도화로 둔다.

### 3.10 high-priority LLMTestTool UX candidate

- [11-nanobot-llmtesttool-v2-usability-results-reporting-phase1.md](./11-nanobot-llmtesttool-v2-usability-results-reporting-phase1.md)

이 단계는 `LLMTestTool_v2` 의 결과 디렉터리 구조, 실행 progress log, interactive launcher, 전체 suite 실행 명령, 최종 report chart 를 정리하는 high-priority usability slice 다. 2026-05-16 기준으로 core phase-1 구현은 대부분 반영됐고, planning 문서는 완료/부분 완료 상태와 검증 범위를 함께 기록하는 reference 로 본다. answer evaluation 보다 먼저 적용해 이후 평가 결과도 단순한 run 중심 구조에 얹히게 하는 편이 맞다.

### 3.11 high-priority local LLM control and default route candidate

- [12-nanobot-local-llm-control-and-smart-router-default-phase1.md](./12-nanobot-local-llm-control-and-smart-router-default-phase1.md)

이 단계는 기존 `local-models` 공용 계층을 Nanobot 실제 응답 경로와 WebUI/Telegram 설정 surface 에 연결하는 후속 slice 다. `qwen36` 을 phase-1 기본 local LLM 으로 두고, `~/.nanobot/nanobot.env` 대신 `~/.nanobot/local-llm.env` override 파일로 선택값을 관리하며, smart-router `local` tier 가 선택된 로컬 LLM을 사용하도록 설정하는 것을 목표로 한다. 2026-05-16 기준으로 implementation 진행 기준이 확정된 high-priority candidate 로 둔다.

## 4. 이번 백로그에서 바로 후순위로 둔 항목

아래는 필요하지만 이번 first execution backlog 에서는 후순위로 둔다.

- Kakao adapter 본격 구현
- contacts / notes 전용 domain 확장
- full owner directory 또는 별도 중앙 task database
- rich timeline / inbox / dashboard 급 독립 WebUI 뷰

## 5. 공통 완료 기준

각 workplan 은 아래 공통 기준을 가진다.

- phase-1 범위와 비범위가 분명하다.
- runtime, WebUI, metadata, approval, test 관점의 작업이 같이 정리된다.
- live 운영 원칙과 충돌하지 않는다.
- 다음 단계 문서와 자연스럽게 연결된다.

## 6. 한 줄 결론

이 디렉터리는 `무엇을 어떻게 구현했는지` 를 설명하는 product-layer workplan 기록이며, 현재 active queue 자체는 [../todo.md](../todo.md) 를 기준으로 읽는다.