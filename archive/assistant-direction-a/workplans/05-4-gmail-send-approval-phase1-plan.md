# 05-4 Gmail Send Approval Phase 1 Plan

## 상태 메모

- 상태: 완료, active backlog 에서 흡수됨
- 현재 source of truth: `docs/planning/todo.md` section 5.4, `docs/planning/execution-backlog/04-gmail-send-approval-phase1.md`
- 이 문서는 mail send approval 초기 계획과 lifecycle 정리 배경을 남기는 archive reference 다.

## 1. 문서 목적

이 문서는 방향 A 구현 백로그의 `5.4 Gmail pilot domain: send approval flow`를 1차 구현 단위로 정리한 작업 진행 방안 문서다.

공통 result / failure / visibility vocabulary 는 `docs/archive/assistant-direction-a/08-shared-action-result-and-failure-shape.md` 를 따른다.

이번 1차 목표는 Gmail send action 을 assistant automation 에 연결하되, 실제 발송은 반드시 approval 이후에만 일어나도록 하는 최소 안전 경로를 정의하는 것이다.

핵심 질문은 아래다.

- `mail.send_message` 를 어떤 approval contract 위에 올릴 것인가?
- approval 요청, 승인 후 resume, reject 후 정리까지를 어떤 상태 전이로 볼 것인가?
- WebUI 와 linked session 에서 send approval 을 어디까지 보이게 할 것인가?

## 2. 이번 1차 범위

1차 범위에 포함할 것:

- `mail.send_message` task contract 초안
- send action approval policy 연결
- approval 요청 문구 표준화
- WebUI pending approval visibility 연결
- 승인 후 send resume 최소 경로 정의
- send success / send rejected / external API failure 결과 형식 정의
- focused backend test 계획 정의

1차 범위에서 제외할 것:

- Kakao/Telegram 안에서 approval 을 직접 끝내는 rich UX
- 다중 승인자 모델
- scheduled send
- mass send / bulk queue
- mailbox rule driven auto-send

## 3. 1차에서 결정해야 할 핵심 사항

### 3.1 send action approval 기준

phase-1 에서 `mail.send_message` 는 기본적으로 high-risk action 으로 본다.

따라서 아래를 고정한다.

- 실제 발송 전 approval 필수
- approval 전에는 draft preview 와 recipient summary 만 노출
- approval 거절 시 send 는 절대 실행되지 않음

### 3.2 approval payload 최소안

approval 요청에 최소한 아래 정보는 포함되어야 한다.

- recipient summary
- subject summary
- body preview 또는 핵심 요약
- action type = `mail.send_message`
- originating session / channel summary

이 단계에서는 full MIME payload 나 raw Gmail request body 를 approval 표면에 올리지 않는다.

### 3.3 resume / reject 상태 전이

phase-1 에서 필요한 최소 상태 전이는 아래다.

- draft ready
- waiting approval
- send in progress
- send completed
- send rejected
- send failed

중요한 점은 approval lifecycle 을 generic runtime approval 흐름과 맞추되, mail send 결과는 mailbox domain summary 로 다시 정리해야 한다는 것이다.

### 3.4 visibility 연결 범위

phase-1 에서는 `execute anywhere` 보다는 `visible from WebUI` 를 우선한다.

즉, 아래 정도를 우선 보장한다.

- active thread 에서 pending send approval 확인 가능
- linked external session row 에서 approval pending badge 확인 가능
- 승인/거절 후 summary 가 thread 에 남음

### 3.5 failure / result shape 기준

phase-1 에서 먼저 닫아야 할 결과 범주는 아래다.

- send success
- send rejected
- external API failure
- approval expired or unavailable

이 결과는 runtime generic error 가 아니라 mail action result summary 로 표준화해야 한다.

## 4. 작업 분해

### Step 1. send action approval contract 정리

해야 할 일:

- `mail.send_message` 입력/출력 shape 정의
- approval required 조건과 approval payload 필드를 정리
- runtime approval system 과의 연결 지점을 정리

완료 기준:

- send action approval contract 초안이 문서화됨

### Step 2. approval user-facing copy 정의

해야 할 일:

- 발송 전 확인 문구 표준안 정의
- recipient / subject / preview 를 어떤 수준까지 보여줄지 정리
- reject 시 안내 문구와 completed summary 차이를 정리

완료 기준:

- WebUI 및 linked session 에서 재사용할 approval copy 기준이 정리됨

### Step 3. resume / reject 경로 정의

해야 할 일:

- 승인 후 send resume 호출 순서 정리
- reject 시 draft 보존 여부와 summary 정리
- approval summary cleanup 시점 정리

완료 기준:

- send action lifecycle 이 최소 상태 전이로 정리됨

### Step 4. WebUI visibility 규칙 정의

해야 할 일:

- thread inline status block 과 approval card 연결 규칙 정리
- sidebar / linked session row 의 pending badge 연결 규칙 정리
- send 완료 또는 실패 후 남길 결과 summary 정리

완료 기준:

- 5.1 / 5.2 와 자연스럽게 이어지는 visibility 규칙이 정리됨

### Step 5. result / failure shape 정의

해야 할 일:

- success / rejected / failed 결과 shape 정리
- external API failure 와 approval rejection 을 구분하는 user-facing summary 정의
- retry 가능 여부를 포함할지 결정

완료 기준:

- send 결과와 실패 형식이 정리됨

### Step 6. focused backend test 계획 정의

테스트 후보:

- approval required send contract 검증
- approval reject 시 send 미실행 검증
- approval accept 후 resume / success summary 검증
- external API failure normalization 검증
- pending approval visibility metadata 검증

완료 기준:

- 실제 구현 시 바로 테스트로 옮길 수 있는 체크리스트가 준비됨

## 5. 권장 구현 순서

1. send action approval contract 정리
2. approval user-facing copy 정의
3. resume / reject 경로 정의
4. WebUI visibility 규칙 정의
5. result / failure shape 정의
6. focused backend test 계획 정리

## 6. 구현 시 주의점

- send approval 을 draft 생성 단계와 섞지 않는다.
- rejection 과 external failure 를 같은 결과로 뭉개지 않는다.
- raw recipient list / message payload 를 과도하게 노출하지 않는다.
- approval visibility 와 actual send authority 를 혼동하지 않는다.

## 7. 완료 기준

이번 1차 문서 기준 완료는 아래를 뜻한다.

1. `mail.send_message` approval contract 가 정의된다.
2. approval copy, resume / reject 경로, visibility 규칙이 정리된다.
3. success / rejected / failed 결과 형식이 정리된다.
4. 5.1 / 5.2 status 및 continuity 설계와 연결된다.
5. 실제 backend 테스트 항목으로 옮길 수 있는 체크리스트가 준비된다.

## 8. 다음 단계 연결

이 작업이 끝나면 다음 단계는 아래로 이어진다.

- Gmail send approval flow 의 실제 runtime 구현
- approval summary 와 mail send result 의 WebUI 반영
- Kakao/외부 채널에서 approval 안내를 어떻게 우회 또는 연결할지 검토

## 9. 현재 1차 구현 메모

현재 source 기준 1차 구현은 아래처럼 잡는다.

- `mail.send_message` 는 approval required action 으로 고정한다.
- approval payload 는 recipient summary, subject summary, body preview 정도의 안전한 요약 중심으로 둔다.
- approval pending 상태는 기존 `approval_summary` 표면을 재사용해 WebUI 와 linked row 에 노출한다.
- 승인 후 실제 send 실행과 결과 summary 생성은 하나의 resume 경로로 묶는다.
- reject 시 draft 는 유지 가능하되, send action 은 종료된 것으로 정리한다.
- external failure 는 `send_rejected` 와 구분되는 별도 failure shape 로 유지한다.