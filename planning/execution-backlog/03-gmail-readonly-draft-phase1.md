# 03 Gmail Read-Only Plus Draft Phase 1

## 1. 문서 목적

이 문서는 execution backlog 의 3순위 첫 단계인 `Gmail read-only + draft flow`를 phase-1 범위로 정리한 실행 문서다.

이번 단계의 목표는 mailbox domain 에 대해 `조회 -> 요약 -> 초안`까지를 assistant action 으로 연결해, approval 전 단계까지의 개인비서 경험을 구현하는 것이다.

## 2. 왜 지금 이 단계인가

- control surface 가 먼저 강화되어야 상태를 보여줄 수 있다.
- owner/task backbone 이 먼저 정리되어야 draft 와 summary 가 session-local 임시 정보로 흩어지지 않는다.
- Gmail은 개인비서 가치가 큰 첫 domain 이다.

## 3. phase-1 범위

포함할 것:

- `mail.list_important_threads`
- `mail.summarize_threads`
- `mail.create_draft`
- direct API-backed 또는 MCP-backed seam 결정
- normalized thread / draft summary shape
- WebUI mail status 최소 연결

제외할 것:

- send action
- approval resume loop
- mailbox rule engine
- full mailbox management UI

## 4. 핵심 작업

### Step 1. mail task contract 최소안 확정

- input/output schema 정리
- result / failure summary vocabulary 정리

### Step 2. executor seam 선택

- direct adapter vs MCP adapter
- auth / token / provider failure 책임 경계 정리

### Step 3. normalized summary shape 정의

- thread list summary
- per-thread summary
- draft preview summary

### Step 4. WebUI status 연결

- lookup running
- summary ready
- draft ready
- action failed

### Step 5. focused validation 계획

- backend task contract tests
- failure normalization tests
- frontend status exposure tests

## 5. 완료 기준

1. 중요한 메일 요약과 답장 초안 생성이 assistant action 으로 표현된다.
2. draft 결과가 thread 친화적 summary 로 WebUI 에 노출된다.
3. send approval 단계로 자연스럽게 이어질 수 있는 contract 가 준비된다.

## 6. 참고 연결

- `docs/archive/assistant-direction-a/workplans/05-3-gmail-pilot-readonly-draft-phase1-plan.md`
- [04-gmail-send-approval-phase1.md](./04-gmail-send-approval-phase1.md)