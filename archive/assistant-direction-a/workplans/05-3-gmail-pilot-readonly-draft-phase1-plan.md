# 05-3 Gmail Pilot Read-Only + Draft Phase 1 Plan

## 상태 메모

- 상태: 완료, active backlog 에서 흡수됨
- 현재 source of truth: `docs/planning/todo.md` section 5.3, `docs/planning/execution-backlog/03-gmail-readonly-draft-phase1.md`
- 이 문서는 Gmail pilot 초기 계획과 seam 선택 배경을 남기는 archive reference 다.

## 1. 문서 목적

이 문서는 방향 A 구현 백로그의 `5.3 Gmail pilot domain: read-only + draft`를 1차 구현 단위로 정리한 작업 진행 방안 문서다.

공통 result / failure / visibility vocabulary 는 `docs/archive/assistant-direction-a/08-shared-action-result-and-failure-shape.md` 를 따른다.

이번 1차 목표는 Gmail 전체 제품을 만드는 것이 아니라, 단일 사용자 local-first assistant가 mailbox를 안전한 automation domain으로 다루는 최소 경로를 여는 것이다.

핵심 질문은 아래다.

- Gmail pilot의 첫 executor 경로를 direct API-backed 와 MCP-backed 중 어디에 둘 것인가?
- read-only 조회, 요약, draft 생성까지를 어떤 task contract 로 나눌 것인가?
- WebUI에서 mail action status 를 어디까지 드러내야 phase-1 검증으로 충분한가?

## 2. 이번 1차 범위

1차 범위에 포함할 것:

- `mail.list_important_threads` task contract 초안
- `mail.summarize_threads` task contract 초안
- `mail.create_draft` task contract 초안
- Gmail executor 경로 1차 선택
- normalized thread / draft summary shape 정의
- WebUI mail action status 최소 연결
- auth needed / mailbox unavailable / thread not found failure shape 정의
- focused backend test 계획 정의

1차 범위에서 제외할 것:

- 실제 send action
- approval resume loop
- mailbox rule engine
- multi-account mailbox directory
- 첨부파일 고급 처리
- full Gmail management UI

## 3. 1차에서 결정해야 할 핵심 사항

### 3.1 Gmail pilot executor 선택

후보는 아래 둘이다.

- direct API-backed pilot
- MCP-backed pilot

1차 판단 기준:

- local 운영 환경에서 재현이 쉬운가
- auth / token refresh / permission failure 를 좁은 범위에서 다룰 수 있는가
- Nanobot core runtime 을 Gmail 특화 로직으로 오염시키지 않는가

1차 권장:

- Gmail pilot 자체의 task contract 는 core 쪽에 둔다.
- 실제 mailbox access 는 direct adapter 또는 MCP adapter 뒤에 감춘다.
- phase-1 에서는 둘 중 하나를 고르되, 호출부는 `mail.*` task interface 기준으로 고정한다.

### 3.2 read-only / draft task contract 최소안

phase-1 최소 task 는 아래로 둔다.

- `mail.list_important_threads`
- `mail.summarize_threads`
- `mail.create_draft`

필요 시 보조 task 후보:

- `mail.load_thread_context`
- `mail.review_draft`

중요한 점은 thread transport 자체가 아니라, assistant action 이 mailbox domain 을 어떻게 다루는지 계약을 먼저 고정하는 것이다.

### 3.3 normalized mailbox shape

phase-1 에서 정규화할 최소 필드는 아래 정도면 충분하다.

- thread id
- subject
- sender summary
- last update time
- unread / importance hint
- draft id
- draft subject
- draft body preview

이 단계에서는 raw Gmail payload 전체를 runtime 전역 타입으로 끌어올리지 않는다.

### 3.4 visibility 와 status 연결 범위

phase-1 에서 WebUI에 보여줄 최소 상태는 아래다.

- mail thread lookup running
- thread summary ready
- draft ready
- action failed

approval pending 은 5.4 send 단계에서 본격 연결하고, 5.3 에서는 read-only / draft 결과를 thread 친화적 summary 로 보여주는 데 집중한다.

### 3.5 failure shape 기준

phase-1 에서 먼저 닫아야 할 대표 실패는 아래다.

- auth needed
- mailbox unavailable
- thread not found
- provider temporary failure

이 실패는 raw exception text 가 아니라 user-facing summary 와 internal error code 성격으로 정규화하는 쪽이 맞다.

## 4. 작업 분해

### Step 1. Gmail pilot consumer flow 정리

해야 할 일:

- 어떤 사용자 요청이 `list -> summarize -> draft` 흐름으로 들어오는지 정리
- WebUI thread 에 어떤 중간 상태와 결과가 남아야 하는지 정리
- 기존 Email channel 과 Gmail automation domain 의 경계를 다시 명시

완료 기준:

- Gmail pilot 의 대표 사용자 흐름이 문서화됨

### Step 2. task contract 초안 정의

해야 할 일:

- `mail.list_important_threads` 입력/출력 shape 정의
- `mail.summarize_threads` 입력/출력 shape 정의
- `mail.create_draft` 입력/출력 shape 정의

완료 기준:

- backend task interface 초안이 문서화됨

### Step 3. executor seam 선택

해야 할 일:

- direct adapter 와 MCP adapter 중 1차 경로 선택
- auth, token, failure handling 책임이 어느 계층에 있는지 정리
- Nanobot runtime 이 consumer contract 만 알도록 경계 정리

완료 기준:

- Gmail mailbox access seam 이 결정됨

### Step 4. normalized thread / draft shape 정의

해야 할 일:

- thread summary 표시용 최소 필드 정의
- draft 결과를 thread 친화적 형식으로 normalize 하는 기준 정의
- raw payload 를 노출하지 않을 규칙 정리

완료 기준:

- WebUI / runtime 이 공유할 최소 normalized shape 가 정의됨

### Step 5. WebUI 상태 노출 후보 정의

해야 할 일:

- mail action status 를 thread 안에 어떻게 표시할지 정리
- draft ready 상태를 approval 없이 검토 가능한 형태로 보여줄지 정리
- 최근 mail action summary 가 필요한지 정리

완료 기준:

- 5.1 status surface 와 자연스럽게 이어지는 mail status 노출 후보가 정리됨

### Step 6. failure shape 와 focused test 계획 정의

테스트 후보:

- task contract validation
- normalized thread / draft shape serialization
- auth needed / mailbox unavailable / thread not found failure normalization
- read-only / draft status emission 검증

완료 기준:

- 실제 구현 시 바로 테스트로 옮길 수 있는 체크리스트가 준비됨

## 5. 권장 구현 순서

1. Gmail pilot consumer flow 정리
2. task contract 초안 정의
3. executor seam 선택
4. normalized thread / draft shape 정의
5. WebUI 상태 노출 후보 정의
6. failure shape 와 backend test 계획 정리

## 6. 구현 시 주의점

- Gmail pilot 을 Email channel 확장으로 오해하지 않는다.
- raw Gmail API payload 를 WebUI 나 session metadata 표면에 그대로 노출하지 않는다.
- draft 생성은 low-risk 로 보되, send approval 과 섞지 않는다.
- mailbox auth / token 문제를 generic provider failure 와 구분한다.

## 7. 완료 기준

이번 1차 문서 기준 완료는 아래를 뜻한다.

1. Gmail pilot 의 대표 사용자 흐름이 정의된다.
2. `mail.list_important_threads`, `mail.summarize_threads`, `mail.create_draft` contract 초안이 정의된다.
3. executor seam 과 normalized shape 기준이 정리된다.
4. WebUI status 연결 기준과 failure shape 가 정리된다.
5. 실제 backend 테스트 항목으로 옮길 수 있는 체크리스트가 준비된다.

## 8. 다음 단계 연결

이 작업이 끝나면 다음 단계는 아래로 이어진다.

- Gmail task contract 의 실제 runtime 구현
- WebUI 에 mail action status 노출
- 5.4 send approval flow 연결

## 9. 현재 1차 구현 메모

현재 source 기준 1차 구현은 아래처럼 잡는다.

- Gmail pilot 은 channel 이 아니라 automation domain 으로 다룬다.
- runtime 호출 표면은 `mail.*` task contract 로 먼저 고정한다.
- mailbox access 구현은 direct adapter 또는 MCP adapter 뒤에 숨긴다.
- phase-1 의 user-visible 결과는 `important threads summary`, `thread summary`, `draft preview` 세 가지로 제한한다.
- failure 는 최소한 `auth_needed`, `mailbox_unavailable`, `thread_not_found` 정도의 구분을 갖는 normalized shape 로 정리한다.
- send approval 과 resume loop 는 이번 5.3 에서 다루지 않고 5.4 로 넘긴다.
- code draft 는 `source/nanobot/automation_results.py` 에 공통 action result 초안과 함께 `MailImportantThreadsResult`, `MailSummarizeThreadsResult` 까지 먼저 반영한다.