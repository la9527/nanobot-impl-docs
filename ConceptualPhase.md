# Nanobot LLM 개인비서 Conceptual Phase

이 문서는 `LLM 개인비서`를 제품 관점에서 어떻게 정의할지, 그리고 현재 Nanobot 및 관련 문서 기준으로 무엇이 이미 준비되어 있고 무엇을 더 보강해야 하는지 정리하는 초안이다.

문서 목적은 세 가지다.

- 개인비서로서 필요한 기능을 큰 축부터 빠짐없이 나열하기
- 현재 Nanobot runtime과 문서에 이미 반영된 내용, 부분 구현된 내용, 아직 없는 내용을 구분하기
- 이후 작업을 계속 보강할 수 있도록 gap과 우선순위를 살아 있는 문서로 유지하기

이 문서는 방향 A 문서군과 호환되도록 `로컬 self-hosted`, `단일 사용자`, `assistant-global memory`, `approval`, `channel continuity` 전제를 유지한다.

## 0. 현재 비교에 사용한 기준 문서

이번 정리에서 현재 상태를 판단할 때 주로 참고한 축은 아래다.

- `docs/assistant-direction-a/02-single-user-operating-policy.md`
- `docs/assistant-direction-a/04-channel-continuity-and-identity-mapping.md`
- `docs/assistant-direction-a/05-assistant-automation-architecture.md`
- `docs/todo.md`
- `source/README.md`

즉, 이번 문서는 runtime 코드의 세부 구현 목록보다 `이미 합의된 방향 문서 + 현재 운영/기능 문서`를 기준으로 gap을 정리한 conceptual summary라고 보면 된다.

## 1. 개인비서의 목표 정의

여기서 말하는 LLM 개인비서는 단순 질의응답 챗봇이 아니라 아래 성질을 가진 시스템이다.

- 여러 채널에서 같은 assistant처럼 동작한다.
- 사용자 장기 맥락을 기억하고 다음 행동에 반영한다.
- 일정, 메일, 문서, 로컬 작업, 외부 도구를 실제로 다룬다.
- 위험한 행동은 approval을 거쳐 안전하게 실행한다.
- 대화만 하는 것이 아니라 `상태 파악`, `초안 작성`, `실행`, `후속 관리`까지 이어진다.
- WebUI는 단순 채팅창이 아니라 control surface 역할을 한다.

즉, 개인비서의 핵심은 답변 품질 하나가 아니라 아래 루프를 안정적으로 수행하는 것이다.

1. 사용자의 현재 상태와 맥락을 이해한다.
2. 필요한 정보와 도구를 찾아 정리한다.
3. 실행 가능한 action으로 바꾼다.
4. 승인과 안전 기준을 통과하면 실제로 수행한다.
5. 결과와 후속 과제를 같은 assistant 맥락으로 이어 준다.

## 2. 개인비서 기능 맵

개인비서 기능은 아래 8개 축으로 보는 것이 적절하다.

### 2.1 대화와 채널 축

필요 기능:

- WebUI, Telegram, Slack, Email, 향후 Kakao 등 다채널 진입점
- 채널별 제약을 유지하면서 같은 assistant 경험 제공
- 대화 이어보기, linked session 확인, channel-specific formatting 대응
- 음성, 첨부파일, 이미지, 링크 등 멀티모달 입력 흡수

개인비서 관점 핵심 질문:

- 사용자가 어느 채널에서 말하든 같은 assistant를 쓰고 있다고 느끼는가?
- text-only 채널과 rich UI 채널의 역할 분리가 명확한가?

### 2.2 기억과 사용자 모델 축

필요 기능:

- assistant-global long-term memory
- session-local context와 long-term memory의 분리
- 사용자 선호, 반복 습관, 장기 프로젝트, 인간관계/연락처 맥락 저장
- memory 반영 기준, decay, correction, deletion 정책
- memory를 action과 연결하는 retrieval 구조

개인비서 관점 핵심 질문:

- assistant가 내 선호와 ongoing project를 누적해서 기억하는가?
- 잘못 저장된 기억을 쉽게 수정하거나 잊게 할 수 있는가?

### 2.3 계획과 작업 관리 축

필요 기능:

- active project / ongoing task 관리
- 오늘 할 일, 대기 중인 일, follow-up이 필요한 일의 구분
- long-running task 상태 추적
- draft, pending approval, blocked, completed 같은 상태 모델
- 대화에서 나온 실행 항목을 task로 승격하는 규칙

개인비서 관점 핵심 질문:

- assistant가 단순 답변 후 사라지지 않고 후속 과제를 계속 추적하는가?
- 중간 상태를 WebUI에서 한눈에 볼 수 있는가?

### 2.4 외부 세계 실행 축

필요 기능:

- Email/Gmail 읽기, 요약, 초안 작성, 발송
- Calendar 조회, 충돌 탐지, 일정 생성, 수정, 취소
- 문서/노트/파일 읽기와 초안 작성
- macOS 로컬 자동화, shell/exec, 브라우저 작업, 앱 제어
- MCP 또는 plugin 기반 외부 툴 연결

개인비서 관점 핵심 질문:

- assistant가 내 대신 무엇을 실제로 할 수 있는가?
- action이 tool call이 아니라 사용자에게 설명 가능한 작업 단위로 보이는가?

### 2.5 안전과 승인 축

필요 기능:

- destructive action, 외부 발송, 민감 정보 접근에 대한 approval
- approval 요청 이유와 결과의 표준화
- text channel과 WebUI에서 모두 승인 흐름 제공
- cross-session approval visibility
- action audit summary와 rollback 가능성 검토

개인비서 관점 핵심 질문:

- assistant가 지나치게 많은 일을 묵시적으로 실행하지 않는가?
- 승인해야 할 작업이 어디서든 보이고 재개 가능한가?

### 2.6 관측성과 운영 축

필요 기능:

- assistant status, active target, current channel, pending approval, automation active 상태 가시화
- 최근 action summary, 실패 원인, retry 상태 노출
- health check, launchd 운영, live runtime과 source tree 검증 분리
- action/result/log의 최소 추적 가능성

개인비서 관점 핵심 질문:

- 지금 assistant가 무엇을 하고 있는지 바로 알 수 있는가?
- 실패했을 때 어디가 문제인지 운영자가 추적할 수 있는가?

### 2.7 지식과 컨텍스트 축

필요 기능:

- 로컬 문서, 메모, 코드, 첨부파일, 대화 이력의 지식화
- RAG 또는 검색 기반 참조
- 프로젝트별 context pack 또는 workspace-aware retrieval
- 외부 정보 검색 결과와 개인 지식의 분리

개인비서 관점 핵심 질문:

- assistant가 일반 모델 지식이 아니라 내 자료를 기반으로 판단하는가?
- 정보 출처와 신뢰도를 구분해서 다루는가?

### 2.8 사전 대응과 루틴 축

필요 기능:

- daily brief, inbox triage, 일정 브리핑, reminder
- scheduled tasks와 event-driven automation
- 특정 조건에서 먼저 알려 주는 proactive behavior
- 단, 과한 autonomy를 막는 제한과 quiet hours 같은 정책

개인비서 관점 핵심 질문:

- assistant가 내가 먼저 물어봐야만 움직이는가?
- 도움이 되지만 귀찮지 않은 수준의 proactive behavior를 만들 수 있는가?

## 3. 현재 Nanobot에서 이미 강한 부분

현재 Nanobot runtime과 문서에서 이미 꽤 강한 기반은 아래다.

### 3.1 이미 확보된 기반

- 다채널 runtime 기반이 있다.
	- WebUI, Telegram, Slack, Email 등 여러 entrypoint를 이미 다룬다.
- plugin/MCP/tool 확장 구조가 있다.
	- core를 크게 깨지 않고 외부 capability를 확장할 수 있다.
- local-first 운영 기반이 있다.
	- launchd, local LLM, editable install, WebUI bundle, health check 흐름이 정리되어 있다.
- model target / smart-router 구조가 있다.
	- `local-llm`, `smart-router`, direct target 전환이 가능하다.
- 기본 memory와 long-running task 기반이 있다.
	- 완성형 personal memory system은 아니어도 runtime 차원의 memory와 task 운영 기반은 존재한다.
- 최소 approval 흐름이 있다.
	- 특히 `exec` 계열에서 approval pattern과 WebUI visibility가 이미 있다.
- WebUI control surface 방향이 잡혀 있다.
	- status badge, pending approval visibility, linked session summary가 문서와 구현 양쪽에 반영되기 시작했다.
- 단일 사용자 continuity 관점이 문서로 정리되어 있다.
	- multi-user IAM보다 owner continuity를 우선하는 방향이 명확하다.

### 3.2 이미 문서화된 방향

아래 내용은 아직 일부만 구현됐더라도 개념 방향은 이미 명확하다.

- single-user operating policy
- channel continuity / identity mapping metadata
- WebUI assistant status / approval visibility
- assistant automation architecture
- Gmail pilot domain
- Kakao adapter feasibility

즉, Nanobot는 `기반 runtime이 없는 상태`가 아니라, 이미 좋은 core를 가진 상태에서 `개인비서 제품 레이어`를 더 얹어야 하는 단계로 보는 것이 맞다.

## 4. 현재 비교 기준에서의 상태 판정

아래 표는 지금 시점의 rough assessment를 `현재 구현`, `문서만 존재`, `설계 미정` 기준으로 더 세밀하게 재분류한 것이다. 이후 구현과 문서가 바뀌면 계속 수정하면 된다.

| 기능 축 | 재분류 | 현재 상태 | 판단 |
| --- | --- | --- | --- |
| 다채널 대화 runtime | 현재 구현 | 강함 | 이미 강점 |
| WebUI control surface | 현재 구현 + 문서 보강 | 부분 구현 | 더 강화 필요 |
| assistant-global memory 정책 | 문서만 존재 + 일부 기반 | 문서화됨 | 구현 보강 필요 |
| session continuity metadata | 현재 구현 + 문서 보강 | 1차 구현 진행 | UI/조회 보강 필요 |
| approval visibility | 현재 구현 + 문서 보강 | 1차 구현 진행 | 범용 action으로 확장 필요 |
| long-running task 상태 모델 | 현재 구현 | 기반 있음 | task/product 레이어 보강 필요 |
| personal task state model | 문서만 존재 | 초안 문서화 | runtime/product 연결 필요 |
| Gmail/Email automation | 문서만 존재 + 일부 인접 기반 | 일부 기반 + 문서화 | 도메인 action 구현 필요 |
| Calendar automation | 문서만 존재 | 개념 설계 수준 | 구현 필요 |
| proactive briefing/routine | 현재 구현 + 설계 미정 혼합 | 일부 scheduled capability | 개인비서형 경험 설계 필요 |
| quiet hours policy | 설계 미정 | 없음 | 정책 정의 필요 |
| contact/relationship memory | 설계 미정 | 거의 없음 | 추가 필요 |
| personal knowledge organization | 현재 구현 인접 기반 + 설계 보강 필요 | 일부 가능 | retrieval/product 설계 필요 |
| 결과 요약/audit/history | 현재 구현 인접 기반 + 설계 보강 필요 | 약함 | 운영 및 UX 보강 필요 |

재분류 해석 기준:

- `현재 구현`: runtime 또는 UI 기준으로 실제 동작 기반이 있음
- `문서만 존재`: 방향 문서나 conceptual 문서로는 정의됐지만 runtime 제품 레이어는 아직 없음
- `설계 미정`: 필요성은 명확하지만 정책, shape, 저장 경계, UX가 아직 닫히지 않음

## 5. 추가해야 할 것

현재 Nanobot 위에 `개인비서 제품`을 세운다고 보면, 특히 아래 기능들은 추가 필요성이 높다.

### 5.1 Owner-centric profile layer

현재 continuity metadata는 있지만, 진짜 개인비서 수준으로 가려면 `primary user profile` 계층이 더 필요하다.

예시:

- 이름, 시간대, 기본 언어, 선호 응답 톤
- 자주 쓰는 연락처, 팀/가족/거래처 같은 관계 메모
- 자주 가는 장소, 루틴 시간대, 회의 선호 규칙
- 기본 도구 선호
	- Gmail 우선인지, 로컬 메일 우선인지
	- 어떤 calendar를 source of truth로 볼지

현재 문서에는 `canonical owner`가 있지만, 아직 `owner profile` 자체는 약하다.

### 5.2 Structured memory policy

현재 memory는 중요 축이지만, 개인비서 용도로는 아래 보강이 필요하다.

- memory type 분류
	- preference
	- project
	- people/contact
	- routine
	- sensitive temporary fact
- memory 생성 기준과 수정 기준
- memory TTL 또는 decay 정책
- 사용자 수정 명령
	- 기억해
	- 이건 잊어
	- 이건 내 기본 설정이 아냐
- memory retrieval가 action planning과 연결되는 구조

지금은 `memory가 중요하다`는 수준은 정리됐지만 `어떤 memory를 어떤 구조로 유지할지`가 더 명확해져야 한다.

### 5.3 Task and follow-up system

개인비서는 대화 자체보다 `후속 관리`에서 체감 가치가 크다.

추가 필요 기능:

- 대화에서 action item 추출
- pending / blocked / waiting-approval / scheduled / completed 상태 관리
- `내가 맡긴 일 목록` 보기
- 특정 task를 다른 채널에서 이어보기
- task 결과가 memory, thread, external action 중 어디에 귀속되는지 구분

현재 long-running task 기반은 있지만, user-facing personal task model은 더 필요하다.

### 5.4 Domain action contracts 확대

방향 A 문서에 이미 일부 정의되어 있지만, 개인비서 관점에서 우선 필요한 도메인은 아래다.

- mail.list_threads
- mail.summarize_threads
- mail.create_draft
- mail.send_message
- calendar.list_events
- calendar.find_conflicts
- calendar.create_event
- calendar.update_event
- notes.capture
- notes.organize
- contacts.lookup
- reminders.create
- daily_brief.generate

핵심은 API를 직접 붙이는 것이 아니라 `assistant action contract`부터 닫는 것이다.

### 5.5 Approval generalization

현재 approval은 일부 실행 경로에 강하지만, 개인비서 관점에서는 더 일반화가 필요하다.

- exec 뿐 아니라 메일 발송, 일정 생성/변경, 외부 업로드, 연락처 사용 등에도 동일한 approval model 적용
- approval request shape 표준화
- why this needs approval 문구 표준화
- 승인 후 재개, 거절 후 정리, timeout 처리
- channel별 승인 방식 차이 흡수

### 5.6 Personal knowledge ingestion

개인비서는 public web search만으로 충분하지 않다.

추가 필요 기능:

- 로컬 문서/노트/회의록/첨부파일 ingestion
- 폴더 또는 workspace 단위 색인
- 대화에서 참조한 파일을 project memory와 연결
- 지식과 memory를 분리 저장

즉, `내 자료를 아는 assistant`가 되어야 한다.

### 5.7 Proactive surface

개인비서 제품다움은 proactive layer에서 크게 갈린다.

추가 필요 기능:

- 아침 briefing
- 일정 직전 요약
- inbox triage 요약
- 미응답 메일 / follow-up 후보 알림
- 오늘 미완료 task 요약

다만 이 기능은 autonomy를 키우기 때문에 quiet hours, frequency limit, opt-in 정책이 같이 필요하다.

## 6. 나아져야 할 점

단순히 기능을 더 넣는 것보다, 아래 품질 축이 같이 좋아져야 실제 개인비서가 된다.

### 6.1 답변 중심에서 상태 중심으로

현재는 채팅 응답이 중심이지만, 개인비서는 아래가 더 중요해진다.

- 지금 무엇을 하고 있는가
- 무엇을 기다리고 있는가
- 무엇이 실패했는가
- 다음에 무엇을 해야 하는가

따라서 WebUI는 `대화창`보다 `assistant 상태판`에 더 가까워져야 한다.

### 6.2 Tool call 중심에서 action 중심으로

사용자 입장에서는 `tool이 호출됐다`보다 아래 형태가 더 중요하다.

- 메일 5개를 요약했다
- 회의 일정 충돌을 찾았다
- 답장 초안을 만들었다
- 승인을 기다리고 있다

즉, 내부 tool log가 아니라 사용자 설명 가능한 action/result 모델이 강화되어야 한다.

### 6.3 session 중심에서 owner continuity 중심으로

현재 session 분리는 맞는 설계지만, 개인비서 체감은 `내 일의 연속성`으로 만들어진다.

더 나아져야 할 점:

- 같은 owner 기준 recent activity 집계
- cross-session pending action visibility
- linked channel summary 강화
- 어떤 채널에서 시작했든 WebUI에서 전체 assistant 상태를 볼 수 있게 하기

### 6.4 memory 양보다 memory 품질 중심으로

기억을 많이 저장하는 것보다 아래가 중요하다.

- 장기적으로 가치 있는가
- 잘못 저장됐을 때 수정 가능한가
- action planning에 실제로 도움 되는가
- 민감 정보 취급이 안전한가

즉, memory는 단순 축적이 아니라 `정제된 개인 맥락`이어야 한다.

### 6.5 기능 추가보다 안전한 실행 체계 우선

개인비서에서 가장 위험한 구간은 실제 action이다.

우선 개선이 필요한 부분:

- action별 approval policy 표준화
- 실패/재시도/중복 실행 방지
- external side effect 기록
- 사용자가 나중에 "무슨 일이 있었는지" 이해할 수 있는 최소 audit trail

## 7. 우선순위 제안

현재 Nanobot 상태를 기준으로 하면, 개인비서 제품화를 위한 우선순위는 아래 순서가 적절하다.

### 7.1 1순위: control surface 완성

- WebUI assistant status 강화
- pending approval / recent action / linked sessions 표시 강화
- 현재 active task와 blocked task 가시화

이 단계가 먼저 필요한 이유는 이후 Gmail, Calendar, Kakao를 붙여도 사용자가 상태를 이해할 수 있어야 하기 때문이다.

### 7.2 2순위: owner memory + task 모델 정리

- owner profile 초안
- structured memory taxonomy
- ongoing task / follow-up 상태 모델
- cross-session continuity에서 보일 최소 aggregate 정의

이 단계가 개인비서의 뼈대다.

### 7.3 3순위: Gmail/Calendar pilot

- Gmail read-only + draft
- Gmail send approval
- Calendar list/create/update
- action result / failure / approval shape 통일

이 단계가 실제 개인비서 value를 가장 빨리 보여 준다.

### 7.4 4순위: proactive routine

- daily brief
- inbox triage
- schedule digest
- follow-up reminder

단, opt-in과 quiet hours가 함께 있어야 한다.

### 7.5 5순위: Kakao 및 외부 채널 확장

- Kakao는 control surface가 약하므로 summary/approval handoff 설계가 선행되어야 한다.
- Kakao 자체 구현보다 WebUI continuity와 action visibility가 더 선행 과제다.

## 8. 현재 기준 핵심 gap 요약

핵심 gap을 짧게 요약하면 아래와 같다.

1. Nanobot는 이미 훌륭한 multi-channel agent runtime이지만, 아직 `owner-aware personal assistant product layer`는 얇다.
2. memory, approval, continuity는 개념과 일부 구현이 있으나 아직 범용 personal workflow 수준으로 통합되지는 않았다.
3. Gmail/Calendar 같은 도메인 action contract를 실제로 닫아야 개인비서다운 가치가 나온다.
4. WebUI는 채팅창을 넘어 assistant control surface로 더 강해져야 한다.
5. 앞으로의 핵심은 기능 수를 늘리는 것보다 `상태`, `안전`, `후속 관리`, `owner continuity`를 일관되게 묶는 것이다.

## 9. 다음 보강 후보

분리 상세 문서:

- `docs/conceptual-phase/owner-profile-and-memory-taxonomy.md`
- `docs/conceptual-phase/personal-task-state-model.md`
- `docs/conceptual-phase/gmail-calendar-domain-contracts.md`
- `docs/conceptual-phase/proactive-policy-and-quiet-hours.md`

실행 백로그 문서:

- `docs/execution-backlog/README.md`
- `docs/execution-backlog/01-owner-aware-control-surface-phase1.md`
- `docs/execution-backlog/02-owner-memory-and-task-backbone-phase1.md`
- `docs/execution-backlog/03-gmail-readonly-draft-phase1.md`
- `docs/execution-backlog/04-gmail-send-approval-phase1.md`
- `docs/execution-backlog/05-calendar-read-conflict-create-phase1.md`
- `docs/execution-backlog/06-proactive-briefing-and-quiet-hours-phase1.md`

이 문서는 계속 확장할 예정이므로, 다음 보강 항목 후보를 남긴다.

- 기능 맵을 `현재 구현`, `문서만 존재`, `설계 미정`으로 더 세밀하게 재분류
- 기능 맵을 `현재 구현`, `문서만 존재`, `설계 미정`으로 더 세밀하게 재분류 완료
- owner profile / memory taxonomy 초안 별도 문서 분리 완료
- personal task state model 초안 추가 완료
- Gmail/Calendar domain contract 표 초안 정리 완료
- proactive policy와 quiet hours 정책 문서 추가 완료
- Kakao/WebUI 역할 분담 표 추가
- 개인비서 품질 지표
	- 응답 품질
	- action 성공률
	- 승인 후 재개 성공률
	- daily brief usefulness
	- follow-up 누락률

## 10. 한 줄 결론

현재 Nanobot는 `개인비서를 만들 수 있는 기반 runtime`은 이미 충분히 갖추고 있다. 앞으로 필요한 것은 새로운 범용 플랫폼을 다시 만드는 일이 아니라, `owner 중심 memory + task + approval + automation + control surface`를 일관된 제품 레이어로 올리는 작업이다.
