# Nanobot Personal Task State Model Draft

이 문서는 [ConceptualPhase.md](../ConceptualPhase.md)에서 분리한 상세 초안으로, Nanobot를 `개인비서`로 발전시키기 위해 필요한 `personal task state model`을 정리한다.

핵심 목적은 세 가지다.

- 현재 Nanobot의 long-running task 기반과 user-facing personal task 개념을 구분하기
- owner continuity, approval, WebUI visibility와 자연스럽게 연결되는 task 상태 모델을 정의하기
- 이후 Gmail/Calendar/action contract가 같은 상태 언어를 재사용할 수 있게 만들기

이 문서는 background runtime task scheduler를 설계하는 문서가 아니다. 여기서 말하는 task는 `사용자가 assistant에게 맡긴 일`을 제품 관점에서 어떻게 표현할지에 대한 모델이다.

## 1. 왜 task state model이 필요한가

현재 Nanobot에는 이미 아래 기반이 있다.

- long-running task 실행 기반
- `/stop` 같은 task 제어 흐름
- approval pending visibility
- WebUI status block
- continuity metadata를 통한 cross-session visibility의 시작점

하지만 개인비서 관점에서 중요한 것은 단순 실행 태스크 수가 아니라 아래 질문에 답하는 것이다.

- 지금 assistant가 내 대신 처리 중인 일은 무엇인가?
- 어떤 일은 승인 대기 중인가?
- 어떤 일은 막혀 있고, 어떤 일은 끝났는가?
- 어떤 일은 메일/일정/문서 같은 외부 action과 연결되어 있는가?
- 이 작업을 다른 채널이나 다음 날에도 이어볼 수 있는가?

즉, runtime task와 personal task는 겹치지만 동일하지 않다.

## 2. 기본 개념

### 2.1 personal task의 정의

personal task는 assistant가 owner를 대신해 추적하거나 수행하는 `사용자 관점의 일 단위`다.

예시:

- 오늘 중요한 메일 5개를 요약한다.
- 회의 일정 충돌을 찾아본다.
- 특정 메일 답장 초안을 만든다.
- 초안 발송 승인을 기다린다.
- 내일 오전 brief를 준비한다.

중요한 점은 이 task가 내부 tool call보다 상위 개념이라는 것이다.

- tool call: 구현 수단
- runtime task: 실행 수명 단위
- personal task: 사용자에게 보이는 일 단위

### 2.2 task와 session의 관계

개인비서 기준에서는 task가 session보다 상위 의미를 갖는 경우가 많다.

예시:

- Email에서 초안 작성 시작
- WebUI에서 초안 검토
- Telegram에서 승인 여부 확인

따라서 task는 아래 성질을 가져야 한다.

- origin session이 있을 수 있음
- current visible session이 바뀔 수 있음
- canonical owner 기준으로 continuity를 가짐

## 3. 권장 task shape

### 3.1 최소 필드

초기 task 모델은 아래 필드를 가지는 것이 적절하다.

```text
task_id
canonical_owner_id
task_kind
title
summary
status
priority
origin_channel
origin_session_key
linked_sessions
created_at
updated_at
due_at
approval_state
action_refs
result_summary
failure_summary
next_step_hint
```

### 3.2 필드 의미

권장 해석:

- `task_id`: owner 기준으로 추적 가능한 식별자
- `canonical_owner_id`: 방향 A 기준 기본값은 `primary-user`
- `task_kind`: mail, calendar, research, local-action, reminder 같은 분류
- `title`: 사용자가 이해할 수 있는 짧은 작업명
- `summary`: 현재 작업 맥락 요약
- `status`: 아래 4장 상태 모델 참조
- `priority`: low, normal, high 정도의 lightweight 수준
- `origin_channel`: 작업이 처음 시작된 채널
- `origin_session_key`: 원 출발 세션
- `linked_sessions`: 현재 작업이 노출된 관련 세션 목록
- `approval_state`: approval 여부와 대기 상태
- `action_refs`: mail thread id, draft id, event id 같은 외부 action 참조
- `result_summary`: 완료 시 사용자 친화적 결과
- `failure_summary`: 실패 시 사용자 친화적 실패 요약
- `next_step_hint`: 다음 사용자 행동 또는 assistant follow-up 제안

## 4. 상태 모델

### 4.1 핵심 상태

초기 상태 집합은 아래 정도가 적절하다.

| 상태 | 의미 | 예시 |
| --- | --- | --- |
| `draft` | 아직 확정되지 않은 작업 준비 상태 | 어떤 메일에 답할지 정리 중 |
| `queued` | 실행 예정이나 아직 본격 실행 전 | batch 요약 대기 |
| `running` | assistant가 현재 처리 중 | 메일 스레드 요약 중 |
| `waiting-approval` | 승인 없이는 다음 단계로 못 감 | 초안 발송 승인 대기 |
| `waiting-input` | 사용자 정보 추가가 필요 | 참석자/시간 미확정 |
| `blocked` | 외부 조건이나 실패로 진행 중단 | API 인증 없음 |
| `scheduled` | 특정 시간 또는 이벤트를 기다림 | 내일 아침 briefing 예약 |
| `completed` | 사용자 관점의 목표 달성 | 일정 생성 완료 |
| `cancelled` | 사용자 또는 시스템이 중단 | `/stop` 또는 명시 취소 |
| `failed` | 종료됐지만 실패 | executor 오류로 메일 조회 실패 |

### 4.2 상태 전이 원칙

대표 전이는 아래를 기준으로 본다.

- `draft -> queued -> running -> completed`
- `running -> waiting-approval -> running -> completed`
- `running -> waiting-input -> running`
- `running -> blocked -> running`
- `running -> failed`
- `queued|running|waiting-approval|waiting-input -> cancelled`

중요한 점은 `blocked`, `failed`, `cancelled`를 하나로 뭉개지 않는 것이다.

- `blocked`: 재개 가능성이 높음
- `failed`: 현재 실행은 종료됐고 원인 설명이 필요함
- `cancelled`: 사용자 의사 또는 시스템 중단이 명확함

### 4.3 approval state와 status의 분리

approval은 상태 자체라기보다 상태를 보조하는 축으로 따로 두는 것이 낫다.

예시:

- `status=waiting-approval`, `approval_state=pending`
- `status=cancelled`, `approval_state=denied`
- `status=completed`, `approval_state=approved`

이렇게 분리하면 같은 task라도 승인 흐름을 더 정확히 설명할 수 있다.

## 5. task 분류

### 5.1 권장 task kind

초기에는 아래 정도로만 분류해도 충분하다.

- `mail`
- `calendar`
- `document`
- `research`
- `local-action`
- `reminder`
- `briefing`
- `follow-up`

이 분류는 domain action contract와 직접 연결된다.

### 5.2 action task와 tracking task 구분

task는 크게 두 종류로 나뉜다.

#### action task

- 실제 외부 실행이나 초안 생성을 동반
- 예: draft 생성, event 생성, 파일 정리

#### tracking task

- 즉시 실행보다 추적과 재노출이 핵심
- 예: 메일 follow-up 필요, 다음 주 재확인, briefing 준비

개인비서 경험에서는 tracking task 비중도 크므로 둘을 모두 모델링해야 한다.

## 6. 저장 경계

### 6.1 session과 분리해야 하는 것

아래는 session-local만으로 두기 아쉽다.

- owner 기준 ongoing task 목록
- cross-session approval pending
- 최근 완료된 중요한 action summary
- follow-up이 필요한 일

### 6.2 long-term memory와 분리해야 하는 것

아래는 장기 memory 본문보다 task layer에 더 가깝다.

- 현재 초안 상태
- 진행 중인 승인 대기
- 이번 주 안에 처리할 follow-up
- 외부 API 호출 실패 후 재시도 대기

즉, task는 memory taxonomy 안에서 `active task memory`로 해석될 수 있지만, 저장 위치는 `USER.md`나 `MEMORY.md`가 아니라 별도 task view 또는 session metadata aggregation에 가깝다.

### 6.3 phase-1 권장 방식

초기에는 별도 복잡한 DB보다 아래 접근이 현실적이다.

- session metadata 기반 task summary 유지
- WebUI에서 recent/active task aggregate view 노출
- approval summary, continuity metadata와 같이 lightweight aggregation부터 시작

## 7. WebUI 노출 원칙

WebUI는 personal task의 첫 control surface가 되어야 한다.

최소 노출 항목:

- 현재 active task 수
- waiting-approval task
- blocked task
- 최근 completed task
- 선택한 thread와 연결된 current task summary

권장 표시 항목:

- task title
- current status
- origin channel
- last updated time
- next step hint

초기에는 full kanban보다 `status block + sidebar summary + linked thread summary` 정도가 현실적이다.

## 8. approval, continuity, memory와의 연결

### 8.1 approval 연결

approval pending은 task 상태의 핵심 consumer다.

원칙:

- approval이 생기면 task는 `waiting-approval` 상태로 승격
- `approval_summary`는 task summary에서 파생
- deny/approve 결과는 task state에 반영

### 8.2 continuity 연결

continuity는 같은 owner가 여러 채널에서 같은 task를 이어보게 하기 위한 기반이다.

원칙:

- task는 channel-local이 아니라 owner-aware view를 가져야 함
- 단, raw execution은 여전히 session context를 존중

### 8.3 memory 연결

task 결과 중 일부만 장기 memory 후보다.

예시:

- 반복적으로 드러난 선호는 `USER.md` 후보
- 프로젝트 운영 규칙은 `MEMORY.md` 후보
- 개별 task의 상세 실행 로그는 memory보다 task history 쪽이 적절

## 9. Gmail/Calendar와의 연결 예시

### 9.1 Gmail draft flow

- `mail.create_draft` 실행
- task 생성: `status=running`
- draft 생성 완료 후 `status=completed` 또는 send 전환 시 별도 send task 생성
- `mail.send_message` 요청 시 send task는 `waiting-approval`

### 9.2 Calendar create flow

- `calendar.create_event` 준비 단계에서 입력 부족 시 `waiting-input`
- conflict 발견 시 `blocked` 또는 `waiting-input`
- 일정 생성 직전 approval 정책이면 `waiting-approval`
- 생성 완료 후 `completed`

이 예시는 domain contract와 task state model이 분리되지만 서로 맞물려야 함을 보여준다.

## 10. Phase별 제안

### Phase 1

- 상태 vocabulary 확정
- task와 session/memory 경계 문서화
- WebUI에 active/waiting-approval/blocked summary 기준 정리

### Phase 2

- Gmail/Calendar action contract에서 task transition 연결
- approval summary와 task summary를 같은 shape로 맞추기
- recent activity를 owner-aware task feed로 노출 검토

### Phase 3

- follow-up, briefing, reminder까지 task kind 확장
- 필요한 경우 lightweight task index 도입 검토

## 11. 한 줄 결론

Nanobot의 personal task model은 새 workflow engine이 아니라, 이미 있는 long-running task·approval·continuity 기반 위에 `사용자가 이해할 수 있는 일의 상태 언어`를 올리는 일이다.