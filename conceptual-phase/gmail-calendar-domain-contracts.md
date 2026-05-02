# Nanobot Gmail And Calendar Domain Contracts Draft

이 문서는 [ConceptualPhase.md](../ConceptualPhase.md)에서 분리한 상세 초안으로, 개인비서 관점에서 우선순위가 높은 `Gmail`과 `Calendar` domain action contract를 표 형태로 정리한다.

목표는 API 세부 구현을 확정하는 것이 아니라 아래를 먼저 고정하는 것이다.

- assistant가 사용자에게 어떤 action 단위로 보일 것인가
- 각 action의 입력/출력/approval/visibility 기준은 무엇인가
- Gmail과 Calendar가 같은 task/result/policy 언어를 공유하도록 어떻게 맞출 것인가

## 1. 설계 원칙

이 문서는 아래 원칙을 따른다.

- channel이 아니라 automation domain으로 본다.
- domain action contract를 서비스 adapter보다 먼저 정의한다.
- 결과는 raw payload가 아니라 사용자 친화적 summary로 정규화한다.
- approval은 send/create/update 같은 side effect action에 붙인다.
- WebUI visibility와 task state model을 함께 고려한다.

## 2. 공통 contract shape

두 도메인 모두 아래 공통 필드를 재사용하는 것이 적절하다.

| 필드 | 의미 |
| --- | --- |
| `domain` | `mail` 또는 `calendar` |
| `action` | domain 안의 세부 action |
| `input` | structured input |
| `approval_policy` | none / review-visible / required |
| `user_summary` | 사용자에게 보여 줄 작업 설명 |
| `result_summary` | 완료 후 보여 줄 핵심 결과 |
| `failure_mode` | 예상 가능한 실패 분류 |
| `task_transition` | task state model에서의 대표 전이 |

approval policy 해석:

- `none`: read-only 또는 low-risk
- `review-visible`: 승인 없이 가능하지만 검토가 쉬워야 함
- `required`: side effect 전 승인 필수

## 3. Gmail domain contracts

### 3.1 Gmail domain 목표

Gmail은 개인비서 가치가 높은 첫 pilot domain이다.

핵심 사용자 루프:

1. 메일을 읽는다.
2. 중요한 내용을 요약한다.
3. 초안을 만든다.
4. 필요하면 승인 후 발송한다.

### 3.2 Gmail action table

| Action | 목적 | 주요 입력 | 정규화 출력 | Approval | 대표 task transition | 비고 |
| --- | --- | --- | --- | --- | --- | --- |
| `mail.list_important_threads` | 중요 메일 후보 조회 | mailbox scope, label filter, date range, limit | thread list summary | `none` | `queued -> running -> completed` | read-only |
| `mail.summarize_threads` | 선택 스레드 요약 | thread ids, summary style, urgency threshold | per-thread summary, urgency, recommended next action | `none` | `queued -> running -> completed` | read-only |
| `mail.load_thread_context` | 초안 전 문맥 확보 | thread id, message limit | normalized thread context | `none` | `queued -> running -> completed` | 내부 helper 성격 강함 |
| `mail.create_draft` | 답장 초안 생성 | target thread id, reply intent, tone preference, constraints | draft subject, draft body, follow-up notes | `review-visible` | `queued -> running -> completed` | draft는 low-risk |
| `mail.review_draft` | 초안 검토/수정 | draft id or normalized draft payload, review instruction | revised draft summary, change notes | `review-visible` | `queued -> running -> completed` | send 직전 품질 단계 |
| `mail.send_message` | 실제 발송 | draft id or normalized payload, approval context | send status, sent timestamp, recipient summary | `required` | `running -> waiting-approval -> running -> completed` | high-risk side effect |

### 3.3 Gmail action별 최소 입력/출력 메모

#### `mail.list_important_threads`

- 입력 최소안: `scope`, `filters`, `limit`
- 출력 최소안: `thread_id`, `subject`, `sender`, `updated_at`, `importance_hint`

#### `mail.summarize_threads`

- 입력 최소안: `thread_ids`, `summary_style`
- 출력 최소안: `thread_summary[]`, `urgency_flag`, `recommended_next_action`

#### `mail.create_draft`

- 입력 최소안: `thread_id`, `reply_intent`, `tone_preference`, `constraints`
- 출력 최소안: `draft_id?`, `subject`, `body`, `follow_up_notes`

#### `mail.send_message`

- 입력 최소안: `draft_ref`, `approval_context`
- 출력 최소안: `send_status`, `sent_at`, `recipient_summary`, `message_ref?`

### 3.4 Gmail failure modes

| Failure code | 의미 | 사용자 표시 방향 |
| --- | --- | --- |
| `mailbox_unavailable` | mailbox 접근 불가 | 인증/연결 상태 확인 유도 |
| `thread_not_found` | 대상 스레드 없음 | thread 재선택 유도 |
| `invalid_draft_input` | 초안 입력 부족/형식 오류 | 필요한 필드 재질문 |
| `approval_required` | 발송 전 승인 필요 | approval pending 상태로 전환 |
| `send_failed` | 발송 자체 실패 | 재시도/검토 유도 |

### 3.5 Gmail visibility 원칙

- thread 안에는 `조회 중`, `초안 작성 중`, `승인 대기`, `발송 완료` 같은 사용자 언어를 노출한다.
- WebUI에는 `pending send approval`, `recent mail actions`, `draft summary`를 노출한다.
- raw Gmail payload, internal auth detail, transport debug log는 직접 노출하지 않는다.

## 4. Calendar domain contracts

### 4.1 Calendar domain 목표

Calendar는 개인비서의 두 번째 핵심 pilot domain이다.

핵심 사용자 루프:

1. 현재 일정을 본다.
2. 충돌을 찾는다.
3. 새 일정을 만든다.
4. 필요하면 수정하거나 취소한다.

### 4.2 Calendar action table

| Action | 목적 | 주요 입력 | 정규화 출력 | Approval | 대표 task transition | 비고 |
| --- | --- | --- | --- | --- | --- | --- |
| `calendar.list_events` | 일정 조회 | date range, calendar scope, filters | event list summary | `none` | `queued -> running -> completed` | read-only |
| `calendar.find_conflicts` | 충돌 탐지 | candidate time range, attendees?, calendar scope | conflict summary, free slots, recommendation | `none` | `queued -> running -> completed` | read-only decision helper |
| `calendar.create_event` | 일정 생성 | title, start/end, timezone, participants, location, notes | created event summary | `required` or `review-visible` | `running -> waiting-input|waiting-approval -> running -> completed` | 기본은 approval 권장 |
| `calendar.update_event` | 일정 수정 | event ref, changed fields, reason | updated event summary, changed fields | `required` | `running -> waiting-approval -> running -> completed` | side effect |
| `calendar.cancel_event` | 일정 취소 | event ref, optional message | cancellation summary | `required` | `running -> waiting-approval -> running -> completed` | destructive |

### 4.3 Calendar action별 최소 입력/출력 메모

#### `calendar.list_events`

- 입력 최소안: `range_start`, `range_end`, `scope`
- 출력 최소안: `event_ref`, `title`, `time_range`, `participants_summary`

#### `calendar.find_conflicts`

- 입력 최소안: `candidate_range`, `scope`, `participants?`
- 출력 최소안: `conflict_items`, `free_slots`, `recommendation`

#### `calendar.create_event`

- 입력 최소안: `title`, `start_at`, `end_at`, `timezone`, `participants?`, `location?`
- 출력 최소안: `event_ref`, `title`, `scheduled_time`, `participants_summary`

#### `calendar.update_event`

- 입력 최소안: `event_ref`, `changed_fields`
- 출력 최소안: `event_ref`, `updated_fields`, `updated_time_summary`

### 4.4 Calendar failure modes

| Failure code | 의미 | 사용자 표시 방향 |
| --- | --- | --- |
| `calendar_unavailable` | calendar 접근 불가 | 인증/연결 상태 확인 유도 |
| `event_not_found` | 일정 참조 실패 | event 재선택 유도 |
| `time_conflict` | 일정 충돌 | 대체 시간 제안 |
| `insufficient_input` | 제목/시간/참석자 정보 부족 | `waiting-input` 전환 |
| `approval_required` | 생성/수정/취소 전 승인 필요 | `waiting-approval` 전환 |
| `update_failed` | 수정/생성 실패 | 재검토/재시도 유도 |

### 4.5 Calendar visibility 원칙

- event creation/update/cancel은 thread에 `일정 생성 준비`, `승인 대기`, `일정 반영 완료` 식으로 보인다.
- WebUI에는 `pending calendar action`, `recent scheduling summary`, `conflict hint`를 노출한다.
- 외부 provider raw payload는 직접 노출하지 않는다.

## 5. Gmail과 Calendar의 공통 차이점 정리

| 항목 | Gmail | Calendar |
| --- | --- | --- |
| 대표 read-only action | 중요 스레드 조회/요약 | 일정 조회/충돌 탐지 |
| 대표 low-risk action | draft 생성/수정 | 제한적 없음, 대부분 review 필요 |
| 대표 high-risk action | 실제 발송 | 생성/수정/취소 |
| approval 핵심 순간 | send 직전 | create/update/cancel 직전 |
| visibility 핵심 | draft, approval, sent summary | conflict, approval, scheduled summary |

## 6. task state model과의 연결

두 domain 모두 [personal-task-state-model.md](./personal-task-state-model.md)의 상태 언어를 재사용하는 것이 적절하다.

예시:

- read-only action: `queued -> running -> completed`
- draft generation: `queued -> running -> completed`
- send/create/update: `running -> waiting-approval -> running -> completed`
- 입력 부족: `running -> waiting-input`
- 외부 문제: `running -> blocked` 또는 `failed`

## 7. phase별 구현 우선순위

### Phase 1

- Gmail: `list_important_threads`, `summarize_threads`, `create_draft`
- Gmail send approval contract 정리
- Calendar: `list_events`, `find_conflicts`, `create_event` 최소안 정리

### Phase 2

- shared action result / failure shape와 맞추기
- WebUI status block과 task summary 연결
- approval summary를 domain action summary와 통일

### Phase 3

- `calendar.update_event`, `calendar.cancel_event` 확장
- notes/contacts/reminder domain까지 동일 패턴 적용

## 8. 한 줄 결론

Gmail과 Calendar는 Nanobot 개인비서화의 첫 핵심 domain이고, 지금 필요한 것은 각 서비스 API를 먼저 붙이는 것이 아니라 `사용자에게 설명 가능한 action contract`, `approval policy`, `task transition`, `visibility shape`를 먼저 닫는 일이다.