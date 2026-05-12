# 04 Gmail Send Approval Phase 1

## 1. 문서 목적

이 문서는 execution backlog 의 3순위 두 번째 단계인 `Gmail send approval flow`를 phase-1 범위로 정리한 실행 문서다.

이번 단계의 목표는 `mail.send_message` 를 approval 기반 action 으로 닫고, approval pending visibility 와 resume 흐름을 WebUI/control surface 와 연결하는 것이다.

## 2. 선행 조건

아래가 먼저 준비되어야 한다.

- Gmail read-only + draft contract
- owner-aware control surface aggregate
- approval summary 와 task summary 의 최소 연결

## 3. phase-1 범위

포함할 것:

- `mail.send_message` contract 초안 구현
- approval policy 연결
- approval request wording 표준화
- send success / denied / failure summary 정리
- WebUI pending approval summary 연결

제외할 것:

- scheduled send
- mailbox rules
- multi-recipient advanced policies
- post-send analytics

## 4. 핵심 작업

### Step 1. send action contract 정의

- input/output schema
- recipient summary
- send status normalization

### Step 2. approval policy shape 정의

- required 로 고정할지
- approval context 에 어떤 정보가 필요한지 정리

### Step 3. approval visibility 연결

- thread inline
- sidebar summary
- owner-aware aggregate 반영

### Step 4. resume / deny flow 정리

- approve 시 resume
- deny 시 cancelled 또는 denied 정리
- timeout 시 처리 규칙

### Step 5. focused validation 계획

- approval pending route exposure
- send success / denied / failure normalization
- WebUI approval visibility test

## 5. 완료 기준

1. 실제 메일 발송은 approval 이후에만 가능하다.
2. approval 대기와 결과가 WebUI 에서 이해 가능하다.
3. send action 이 generic approval model 확장의 첫 사례가 된다.

## 6. 참고 연결

- `docs/archive/assistant-direction-a/workplans/05-4-gmail-send-approval-phase1-plan.md`
- [05-calendar-read-conflict-create-phase1.md](./05-calendar-read-conflict-create-phase1.md)