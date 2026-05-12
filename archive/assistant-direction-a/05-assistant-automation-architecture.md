# Nanobot Assistant Automation Architecture

## 상태 메모

- 이 문서는 방향 A 시기의 assistant automation 상위 아키텍처 reference 다.
- 2026-05-01 기준 실제 구현 source of truth 는 `source/nanobot/automation*.py`, 관련 runtime module, `docs/planning/todo.md` section 5.3~5.6, `docs/planning/execution-backlog/*.md` 다.
- 따라서 본문은 workflow engine 비도입, tool-contract 중심 분리 원칙의 배경 설명으로 읽고, 세부 구현 상태 추적은 active backlog 쪽에서 한다.

## 1. 문서 목적

이 문서는 방향 A 기준에서 Nanobot에 assistant automation 기능을 어떤 구조로 추가할지 정리한 아키텍처 문서다.

여기서 말하는 assistant automation은 아래 범위를 뜻한다.

- Gmail / Email 요약, 초안 작성, 발송 보조
- Calendar 조회, 일정 생성, 일정 변경
- Notion 또는 문서/노트 계열 연동
- browser 기반 조사 또는 제한적 액션
- macOS local helper 기반 로컬 작업
- 이후 Kakao 같은 외부 채널과 연결되는 assistant action

핵심 목표는 automation 기능을 많이 넣는 것이 아니라, `Nanobot core를 무겁게 만들지 않고 assistant action을 안전하게 확장하는 구조`를 정의하는 것이다.

## 2. 방향 A 기준 결론

방향 A에서 assistant automation은 Nanobot core 안에 workflow engine처럼 들어가면 안 된다.

대신 아래 원칙으로 설계하는 것이 맞다.

- core runtime은 유지
- automation은 assistant tool layer로 분리
- domain마다 contract를 먼저 정의
- executor는 교체 가능하게 설계
- approval과 visibility를 필수 요소로 포함
- optional integration이 꺼져 있어도 core 기능은 그대로 유지

즉, 이 문서는 `workflow-first`가 아니라 `assistant runtime + tool contract + optional executor` 구조를 택한다.

## 3. 설계 원칙

### 3.1 core와 integration의 경계

Nanobot core는 아래 역할만 가진다.

- 사용자 요청 해석
- session / memory / approval 관리
- tool invocation orchestration
- result를 thread와 상태 표면에 반영

반대로 외부 서비스별 특수성은 아래 계층으로 내린다.

- domain tool contract
- service adapter
- MCP server
- direct API helper
- local host helper

### 3.2 전역 workflow 엔진 비도입

이 문서는 n8n이나 별도 state graph를 중심 계층으로 두지 않는다.

이유는 아래와 같다.

- 방향 A는 local-first single-user assistant가 우선이다.
- 대화형 assistant 흐름 전체를 workflow engine으로 바꾸면 과도하게 무거워진다.
- automation은 모든 turn에 필요한 것이 아니라 일부 도메인에서만 필요하다.

따라서 automation은 `필요한 작업에만 붙는 action layer`로 제한한다.

### 3.3 domain-first contract

assistant automation은 서비스별 구현보다 도메인별 contract가 먼저다.

예를 들면 아래 순서가 맞다.

1. `calendar.create_event` 같은 도메인 action 정의
2. 입력/출력 schema 정의
3. approval policy 정의
4. visibility policy 정의
5. executor 구현 선택

즉, 처음부터 `Google Calendar API를 어떻게 부를까`가 아니라, `assistant가 일정 생성이라는 액션을 어떻게 표현할까`가 먼저다.

## 4. 권장 아키텍처

방향 A 기준 권장 구조는 아래와 같다.

```text
User Request
  -> Nanobot agent loop
  -> intent/tool decision
  -> assistant automation task
  -> approval check
  -> executor dispatch
  -> result normalization
  -> thread/status/memory update
```

이를 계층별로 나누면 아래와 같다.

### 4.1 Layer 1: Assistant runtime layer

책임:

- 자연어 요청 해석
- session 문맥 유지
- memory 참조
- approval 필요성 판단
- 어떤 automation task를 호출할지 결정

이 계층은 기존 Nanobot agent loop, tool calling, memory 구조에 가장 가깝다.

### 4.2 Layer 2: Automation task layer

책임:

- assistant action을 domain 단위로 표현
- 서비스 독립적인 요청/응답 구조 제공
- 결과 상태를 표준화

예시 action:

- `mail.summarize_threads`
- `mail.create_draft`
- `mail.send_message`
- `calendar.list_events`
- `calendar.create_event`
- `notes.create_page`
- `browser.collect_sources`
- `macos.run_local_helper`

### 4.3 Layer 3: Approval and policy layer

책임:

- destructive 여부 판단
- 외부 변경 여부 판단
- approval 필요 사유 생성
- allowed / blocked / require-approval 판정

이 계층은 단순 boolean보다 아래 정보를 반환하는 것이 낫다.

- decision
- reason
- user-facing summary
- expected side effects

### 4.4 Layer 4: Executor layer

책임:

- 실제 외부 호출 실행
- MCP / direct API / local helper 중 적절한 경로 사용
- 결과와 에러를 표준 형식으로 반환

executor 유형:

- MCP-backed executor
- direct API-backed executor
- webhook-backed executor
- local host helper-backed executor

### 4.5 Layer 5: Result normalization and visibility layer

책임:

- 실행 결과를 thread 친화적 형식으로 변환
- WebUI 상태 블록과 approval 상태에 반영
- memory 반영 여부 판정
- 실패/재시도 메시지 생성

## 5. 핵심 개념: assistant automation task

### 5.1 정의

assistant automation task는 Nanobot가 외부 세계에 대해 실행하려는 의미 있는 동작 단위다.

이 task는 단순 tool call과 다르게 아래를 포함해야 한다.

- domain
- action name
- structured input
- expected outcome
- approval requirement
- execution strategy
- visibility summary

### 5.2 권장 형태

개념적으로는 아래 정보를 갖는 것이 적절하다.

```text
task_id
domain
action
input
approval_policy
executor_kind
user_summary
result_schema
failure_mode
```

### 5.3 task와 일반 tool call의 차이

일반 tool call은 내부 도구 실행에 가깝고, assistant automation task는 `사용자 관점에서 설명 가능한 action 단위`에 가깝다.

예를 들면:

- 일반 tool call: 특정 함수 호출
- automation task: "내일 3시에 회의 일정을 생성한다"

## 6. 도메인 분류

방향 A 기준으로 automation domain은 아래 정도로 분류하는 것이 적절하다.

### 6.1 Communication domain

범위:

- Gmail / Email 요약
- draft 생성
- 발송 전 검토
- 실제 발송

대표 action:

- `mail.list_important_threads`
- `mail.summarize_threads`
- `mail.create_draft`
- `mail.send_message`

### 6.2 Calendar domain

범위:

- 일정 조회
- 일정 생성
- 일정 변경
- 일정 취소

대표 action:

- `calendar.list_events`
- `calendar.find_conflicts`
- `calendar.create_event`
- `calendar.update_event`

### 6.3 Notes and knowledge domain

범위:

- Notion 페이지 생성
- 노트 업데이트
- 회의 요약 저장

대표 action:

- `notes.create_page`
- `notes.append_block`
- `notes.update_page`

### 6.4 Browser research domain

범위:

- 자료 수집
- 정보 비교
- 링크 요약
- 필요 시 제한적 browser action

대표 action:

- `browser.collect_sources`
- `browser.summarize_results`
- `browser.capture_page_state`

### 6.5 Local host helper domain

범위:

- macOS local action
- 파일 열기/저장 전 확인
- 앱 제어 helper

대표 action:

- `macos.run_helper`
- `local.open_app`
- `local.prepare_file_action`

## 7. Executor 전략

### 7.1 MCP-backed executor

적합한 경우:

- 이미 MCP 서버가 존재함
- local 또는 remote helper를 표준화해서 붙이고 싶음
- Nanobot core가 서비스 SDK를 직접 알 필요가 없음

장점:

- core와 integration 분리 용이
- 도구 교체 가능성 높음

단점:

- MCP 운영 구성 추가 필요

### 7.2 direct API-backed executor

적합한 경우:

- 단일 외부 서비스 연동이 단순함
- 인증과 호출이 비교적 안정적임
- pilot 구현을 빨리 하고 싶음

장점:

- 구현 단순
- 디버깅 쉬움

단점:

- 서비스 종속 코드가 늘어날 수 있음

### 7.3 webhook-backed executor

적합한 경우:

- 외부 automation backend를 두고 싶음
- n8n이나 별도 service를 optional하게 연결하고 싶음

장점:

- 외부 워크플로와 연결 쉬움

단점:

- 운영 경로 하나 증가

### 7.4 local host helper-backed executor

적합한 경우:

- macOS local automation
- browser helper
- 파일/앱 등 로컬 환경 작업

장점:

- local-first 방향과 잘 맞음

단점:

- 안전성, approval, 환경 의존성 설계가 중요함

## 8. Approval 아키텍처

assistant automation은 approval을 나중에 붙이면 안 되고, task 정의 단계에 포함해야 한다.

### 8.1 approval이 필요한 조건

- 외부 상태 변경
- 메시지 발송
- 일정 생성/수정/삭제
- 로컬 파일/앱 조작
- shell 또는 helper 실행

### 8.2 approval policy 예시

- `read-only`: 승인 없음
- `confirm-on-change`: 변경 작업만 승인
- `always-confirm`: 항상 승인
- `blocked-by-default`: 특별 허용 전까지 실행 금지

### 8.3 approval 결과 반영

approval 결과는 아래 표면에 연결되어야 한다.

- thread inline status
- WebUI approval visibility
- pending queue
- optional continuity metadata

## 9. Result and visibility 아키텍처

assistant automation의 결과는 단순 raw JSON이 아니라 사용자에게 설명 가능한 결과여야 한다.

### 9.1 사용자 친화적 결과 형식

최소 구성:

- 무엇을 하려 했는지
- 실제로 무엇을 했는지
- 결과가 성공/실패/부분 성공인지
- 다음에 무엇을 할 수 있는지

### 9.2 실패 형식

실패도 아래처럼 구분해야 한다.

- approval needed
- authentication needed
- invalid input
- external service failure
- executor unavailable

이 구분은 WebUI 상태 설계와 바로 연결된다.

## 10. Memory와의 연결 원칙

automation 결과가 모두 memory로 들어가면 안 된다.

기본 원칙:

- 장기 선호나 지속 프로젝트만 memory 후보
- 일회성 task 결과는 thread/session 중심
- 외부 식별자와 raw payload는 memory보다 metadata로 보관

예시:

- "매주 수요일 오전 미팅 선호"는 memory 후보
- "오늘 3시에 일정 생성 성공"은 session event에 가까움

## 11. 구현 순서 제안

### 11.1 1차: contract 정의

- assistant automation task 개념 정리
- domain별 action 목록 초안 작성
- approval policy 정의
- result schema 초안 작성

### 11.2 2차: pilot domain 하나 선택

권장 pilot:

1. mail draft/summarize
2. calendar read/create

이 둘은 방향 A와 잘 맞고, 로컬 self-hosted assistant 가치가 높다.

### 11.3 3차: visibility 연결

- WebUI status block 연결
- approval visibility 연결
- session continuity 표시와 연결

### 11.4 4차: optional executor 확대

- MCP executor 추가
- local helper 추가
- Kakao / browser / notes domain 확장

## 12. 비목표

이 문서에서 의도적으로 하지 않는 것은 아래다.

- 범용 workflow DSL 정의
- 중앙 task database 도입
- 모든 automation을 n8n으로 통일
- multi-user task ownership 설계
- enterprise integration platform 설계

## 13. 최종 판단

방향 A에서 assistant automation은 `Nanobot 위에 덧붙는 선택형 action layer`여야 한다.

핵심은 아래다.

1. action을 domain task로 정의한다.
2. approval을 task 구조에 포함한다.
3. executor는 교체 가능하게 둔다.
4. 결과는 사용자 친화적으로 정규화한다.
5. core runtime은 가볍게 유지한다.

이 구조를 따르면 Nanobot는 workflow engine이 아니라, local-first 개인 assistant runtime으로서 필요한 automation만 안정적으로 수용할 수 있다.