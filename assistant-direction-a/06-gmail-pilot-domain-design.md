# Nanobot Gmail Pilot Domain Design

## 상태 메모

- 이 문서는 Gmail pilot domain 의 초기 설계 reference 다.
- 2026-05-01 기준 read-only, draft, send approval 의 실제 구현/검증 상태는 `docs/todo.md` section 5.3, 5.4 와 `docs/execution-backlog/03-gmail-readonly-draft-phase1.md`, `04-gmail-send-approval-phase1.md` 를 우선 본다.
- 본문은 도메인 경계와 pilot 범위 설정의 배경을 설명하는 historical context 로 유지한다.

## 1. 문서 목적

이 문서는 방향 A에서 assistant automation의 첫 pilot domain 후보로 `Gmail assistant workflow`를 어떻게 설계할지 정리한다.

여기서 목표는 Gmail 전체 플랫폼을 만드는 것이 아니다. 목표는 단일 사용자 local-first assistant가 아래 작업을 안전하게 수행할 수 있게 만드는 것이다.

- 중요한 메일 스레드 요약
- 답장 초안 작성
- 전송 전 검토
- 실제 발송 전 승인

이 문서는 기존 Email channel과의 관계도 함께 정리한다.

## 2. 왜 Gmail을 pilot으로 고르는가

방향 A 기준에서 Gmail pilot은 다음 이유로 적절하다.

- 개인 assistant 가치가 즉시 크다.
- `읽기 -> 요약 -> 초안 -> 승인 -> 발송` 흐름이 assistant action 구조를 검증하기 좋다.
- Nanobot에는 이미 Email channel 문맥이 있어 도메인 인접성이 있다.
- Calendar보다도 승인 흐름과 action visibility를 검증하기 쉽다.

즉, Gmail pilot은 assistant automation architecture의 핵심 요소를 작은 범위에서 검증하기 좋은 첫 도메인이다.

## 3. 범위 정의

### 3.1 1차 범위에 포함할 것

- inbox 또는 label 기준 중요 스레드 조회
- 스레드 요약
- draft 생성
- send 직전 사용자 승인
- send 결과 summary

### 3.2 1차 범위에서 제외할 것

- 메일함 전체 관리 UI
- 복잡한 필터/검색 DSL
- 자동 발송 rule engine
- 대량 발송
- 첨부파일 고급 처리
- 조직용 mailbox delegation

## 4. 기존 Email channel과의 관계

Gmail pilot은 기존 Email channel과 동일하지 않다.

구분은 아래와 같다.

### 4.1 Email channel

역할:

- Email을 하나의 채팅 채널처럼 취급
- inbound/outbound transport에 가까움

### 4.2 Gmail assistant domain

역할:

- assistant가 Gmail mailbox를 업무 도메인처럼 다룸
- 스레드 요약, 답장 초안, 승인 후 발송 같은 action 수행

따라서 Gmail pilot은 channel이 아니라 `automation domain`으로 보는 것이 맞다.

## 5. 대표 사용자 시나리오

### 5.1 중요한 메일 요약

사용자 요청:

- "오늘 중요한 Gmail 메일 요약해줘"

assistant 동작:

1. 중요 스레드 조회
2. 상위 N개 스레드 선택
3. 요약 생성
4. thread 안에 결과 표시

approval:

- 없음 또는 read-only

### 5.2 답장 초안 작성

사용자 요청:

- "이 메일에 답장 초안 만들어줘"

assistant 동작:

1. 대상 스레드 확인
2. 문맥과 tone 반영
3. draft 생성
4. WebUI에 초안 표시

approval:

- draft 생성은 read-only 또는 low-risk

### 5.3 승인 후 발송

사용자 요청:

- "이 초안 그대로 보내"

assistant 동작:

1. draft 확인
2. approval 필요 여부 표시
3. 사용자 승인 수집
4. 실제 발송
5. 결과 요약 표시

approval:

- 필수

## 6. 권장 task 분해

Gmail pilot은 아래 task로 나누는 것이 적절하다.

- `mail.list_important_threads`
- `mail.summarize_threads`
- `mail.load_thread_context`
- `mail.create_draft`
- `mail.review_draft`
- `mail.send_message`

이 분해는 automation architecture 문서의 `assistant automation task` 개념과 맞는다.

## 7. 입력/출력 contract 초안

### 7.1 list_important_threads

입력 예시:

- mailbox scope
- label filter
- date range
- limit

출력 예시:

- thread ids
- sender
- subject
- last update time
- importance hint

### 7.2 summarize_threads

입력 예시:

- selected thread ids
- summary style
- urgency threshold

출력 예시:

- per-thread summary
- recommended next action
- urgency flag

### 7.3 create_draft

입력 예시:

- target thread id
- reply intent
- tone preference
- constraints

출력 예시:

- draft subject
- draft body
- follow-up notes

### 7.4 send_message

입력 예시:

- draft id 또는 normalized message payload
- approval context

출력 예시:

- send status
- sent timestamp
- recipient summary

## 8. Approval 정책

### 8.1 read-only actions

아래는 기본적으로 approval 없이 가능하다.

- thread 조회
- thread 요약
- context 로드

### 8.2 draft actions

아래는 approval 없이 생성 가능하지만, 사용자가 쉽게 검토할 수 있어야 한다.

- draft 생성
- draft 수정

### 8.3 send action

아래는 반드시 approval이 필요하다.

- 실제 메일 발송

### 8.4 user-facing approval 문구 예시

- `이 초안을 실제로 Gmail에서 발송하려고 합니다.`
- `수신자와 내용 요약을 확인한 뒤 승인해주세요.`

## 9. Visibility 정책

Gmail pilot은 action visibility 검증에도 중요하다.

### 9.1 thread 안에서 보여줄 것

- 조회 중 / 요약 중 / 초안 작성 중 / 승인 대기 / 발송 완료 상태
- 초안 요약
- 실패 시 이유

### 9.2 WebUI에서 보여줄 것

- pending send approval
- 최근 mail action summary
- 대상 thread 요약

### 9.3 보여주지 않아야 할 것

- raw Gmail API payload
- 내부 인증 세부값
- 지나치게 긴 transport debug log

## 10. Executor 선택

Gmail pilot은 아래 두 경로 중 하나가 현실적이다.

### 10.1 direct API-backed pilot

장점:

- pilot 구현이 단순하다.
- assistant workflow 검증이 빠르다.

단점:

- Gmail 특화 코드가 Nanobot 쪽에 생길 수 있다.

### 10.2 MCP-backed pilot

장점:

- Gmail adapter를 분리하기 쉽다.
- 이후 Calendar/Notion과 비슷한 패턴으로 확장 가능하다.

단점:

- pilot 속도는 다소 느릴 수 있다.

방향 A 기준 초기 권장은 아래다.

- 가능하면 MCP-backed
- pilot 속도가 더 중요하면 direct API-backed로 시작 후 이후 분리

## 11. Memory 반영 원칙

Gmail pilot 결과는 전부 memory로 올리지 않는다.

memory 후보:

- 답장 tone 선호
- 자주 연락하는 상대에 대한 지속적 선호
- 메일 정리 습관

memory 비후보:

- 개별 메일 전문
- 일회성 발송 결과
- raw thread payload

## 12. 실패 유형

Gmail pilot은 실패를 아래처럼 구분하는 편이 좋다.

- auth needed
- mailbox unavailable
- thread not found
- invalid draft input
- approval required
- send rejected
- external API failure

이 분류는 WebUI action visibility와 연결된다.

## 13. 구현 순서 제안

### 13.1 Step 1

- `mail.list_important_threads`
- `mail.summarize_threads`

목표:

- read-only path 먼저 검증

### 13.2 Step 2

- `mail.create_draft`

목표:

- assistant action + draft visibility 검증

### 13.3 Step 3

- `mail.send_message`
- approval path 연결

목표:

- approval / action visibility / external side-effect 검증

## 14. 완료 기준

Gmail pilot 1차 완료 기준은 아래다.

1. 사용자가 중요한 메일 요약을 요청할 수 있다.
2. 특정 스레드에 대한 답장 초안을 생성할 수 있다.
3. 실제 발송은 반드시 approval 뒤에만 일어난다.
4. thread와 WebUI에서 상태를 이해할 수 있다.
5. integration이 없어도 Nanobot core는 그대로 동작한다.

## 15. 최종 판단

Gmail pilot은 방향 A에서 매우 좋은 첫 automation domain이다.

이유는 아래와 같다.

1. 개인 assistant 가치가 크다.
2. read-only, draft, send, approval을 모두 검증할 수 있다.
3. WebUI visibility와 continuity 설계에 직접 연결된다.
4. core를 무겁게 만들지 않고도 pilot을 진행할 수 있다.

따라서 방향 A 기준 첫 실제 automation pilot은 Gmail workflow로 시작하는 것이 합리적이다.