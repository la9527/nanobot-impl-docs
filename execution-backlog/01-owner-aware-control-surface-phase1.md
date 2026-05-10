# 01 Owner-Aware Control Surface Phase 1

> 용어 메모: 이 문서는 당시 설계 표현을 유지한 기록 문서다. 현재 WebUI 용어 기준은 [docs/user-guide/10-webui-terminology.md](../user-guide/10-webui-terminology.md) 를 따른다.

## 1. 문서 목적

이 문서는 개인비서 execution backlog 의 1순위 작업인 `owner-aware control surface`를 phase-1 범위로 정리한 실행 문서다.

이번 단계의 목표는 이미 구현된 status badge, approval visibility, continuity summary를 `assistant control surface` 수준으로 묶어, 사용자가 WebUI에서 현재 assistant 상태를 owner 기준으로 이해할 수 있게 만드는 것이다.

## 2. 왜 1순위인가

현재 Nanobot는 아래 기반을 이미 갖고 있다.

- WebUI status badge
- approval pending visibility
- linked session summary
- continuity metadata

하지만 아직 아래 질문에는 한 번에 답하기 어렵다.

- 지금 내 assistant가 처리 중인 일은 무엇인가?
- 어떤 일이 approval 대기 중인가?
- 어떤 외부 채널 activity가 같은 owner 문맥으로 이어지는가?
- 최근 완료된 action과 막힌 action은 무엇인가?

즉, control surface를 먼저 강화해야 이후 Gmail, Calendar, proactive가 붙어도 사용자가 상태를 이해할 수 있다.

## 3. phase-1 범위

포함할 것:

- owner-aware assistant summary block
- active / waiting-approval / blocked / recent-completed aggregate
- linked sessions / external activity summary 보강
- approval summary 와 task summary 의 최소 연결
- touched slice frontend/backend 테스트 계획

제외할 것:

- full dashboard
- timeline 전용 페이지
- cross-owner multi-user view
- rich kanban / inbox 화면

## 4. 핵심 구현 대상

### 4.1 assistant summary block

최소 항목:

- active target
- current channel
- active task count
- approval pending count
- blocked count

### 4.2 linked session aggregate

최소 항목:

- same-owner linked sessions 존재 여부
- 가장 최근 external activity summary
- linked approval pending 존재 여부

### 4.3 recent action summary

최소 항목:

- 최근 완료된 action 1~3개
- 실패 또는 blocked 상태 1~2개
- 다음 step hint

## 5. 작업 분해

### Step 1. 현재 WebUI 노출 표면 inventory 재확인

- 현재 header / sidebar / thread inline status가 가진 데이터 정리
- aggregate view에 재사용 가능한 metadata 확인

현재 코드 anchor:

- `source/webui/src/components/thread/ThreadShell.tsx`
- `source/webui/src/components/ChatList.tsx`
- `source/webui/src/lib/types.ts`
- `source/tests/channels/test_websocket_http_routes.py`
- `source/webui/src/tests/thread-shell.test.tsx`
- `source/webui/src/tests/chat-list.test.tsx`

### Step 2. owner-aware aggregate shape 정의

- active tasks
- approval pending summaries
- blocked summaries
- recent completed summaries

phase-1 최소 shape 후보:

```text
assistant_summary:
	active_target
	current_channel
	active_task_count
	approval_pending_count
	blocked_count
	recent_completed[]
	recent_blocked[]
	linked_session_count
	recent_external_activity
```

backend 1차 위치 후보:

- `/api/sessions` listing payload 확장
- session metadata aggregation helper 추가
- WebUI bootstrap payload 재사용 가능 여부 확인

### Step 3. WebUI 최소 summary block 설계

- 어디에 둘지 결정
- 텍스트 우선 compact layout 유지
- existing header/shell 과 충돌하지 않게 배치

실제 코드 작업 단위:

- `ThreadShell.tsx` 에 assistant summary block 자리 추가
- 기존 status badge 아래 또는 sidebar 상단 중 한 곳으로 최소 배치 결정
- 모바일/데스크톱 모두 width contract 를 깨지 않는지 확인

### Step 4. linked session / external activity 보강

- linked channel summary 확장
- 최근 external activity 한 줄 요약 설계

실제 코드 작업 단위:

- `continuity` + `approval_summary` 만으로 만들 수 있는 summary 우선 구현
- `ChatList.tsx` row badge 와 `ThreadShell.tsx` linked summary 문구를 같은 vocabulary 로 맞춤
- owner-aware aggregate store 없이도 session list 기반 derived summary 로 가능한지 우선 확인

### Step 5. 테스트 및 live 검증 계획 정리

- thread-shell 테스트
- session list / route focused test
- WebUI build 및 live 확인

권장 검증 순서:

1. `source/webui/src/tests/thread-shell.test.tsx`
2. `source/webui/src/tests/chat-list.test.tsx`
3. `source/tests/channels/test_websocket_http_routes.py`
4. `source/webui` 기준 `npm run build`

## 6. 실제 코드 작업 backlog

아래는 문서 수준이 아니라 실제 코드 착수 단위로 더 쪼갠 backlog 다.

### 6.1 backend aggregation slice

- session summary aggregate helper 위치 결정
- `approval_summary` pending count 계산 경로 정의
- linked external session count 계산 경로 정의
- recent completed / blocked summary 를 metadata 기반으로 만들 수 있는 최소 shape 정의

### 6.2 API exposure slice

- `/api/sessions` payload 만으로 충분한지 판단
- 부족하면 route serializer 에 assistant summary block 추가
- 기존 WebUI session fetch contract 와 호환성 확인

### 6.3 ThreadShell UI slice

- assistant summary block 컴포넌트 초안 추가
- active / approval / blocked / linked 상태 표시
- recent external activity 또는 next step hint 표시 후보 추가

### 6.4 Sidebar row alignment slice

- `ChatList.tsx` 의 approval badge 와 thread 내부 summary wording 통일
- linked session / blocked / active 상태와 충돌하지 않는 row density 유지

### 6.5 test slice

- thread-shell aggregate rendering test
- chat-list badge alignment test
- websocket route aggregate serialization test

### 6.6 live validation slice

- local WebUI 에서 approval pending session + linked external session 동시 표시 확인
- deny/resolve 후 aggregate 정리 확인
- build / launchd live runtime 반영 확인

## 7. 완료 기준

1. 사용자가 WebUI에서 same-owner 기준 현재 상태를 한눈에 파악할 수 있다.
2. active / approval / blocked / recent action 이 최소 aggregate로 노출된다.
3. 이후 Gmail/Calendar action summary가 꽂힐 자리가 확보된다.

## 8. 다음 단계 연결

- [02-owner-memory-and-task-backbone-phase1.md](./02-owner-memory-and-task-backbone-phase1.md)
- [03-gmail-readonly-draft-phase1.md](./03-gmail-readonly-draft-phase1.md)