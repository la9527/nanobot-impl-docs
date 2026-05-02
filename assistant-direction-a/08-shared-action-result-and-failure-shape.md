# Nanobot Shared Action Result and Failure Shape

## 상태 메모

- 이 문서는 direction A 시기의 공통 vocabulary 정리 문서로 유지한다.
- 2026-05-01 기준 mail/calendar runtime 구현은 이미 존재하므로, 실제 field/source of truth 는 `source/nanobot/automation_results.py`, 관련 automation module, `docs/todo.md` section 5 진행 메모를 우선 본다.
- 따라서 이 문서는 새로운 공통 shape 변경을 직접 선언하는 spec 이라기보다, domain 간 naming 과 visibility 관점을 맞추는 reference 로 읽는 것이 맞다.

## 1. 문서 목적

이 문서는 방향 A 기준 assistant automation domain 들이 공통으로 재사용할 `action result / failure / visibility` 최소 shape 를 정리한 상위 설계 문서다.

대상 범위는 우선 아래 세 축이다.

- Gmail read-only + draft pilot
- Gmail send approval flow
- Kakao adapter 최소 범위

여기서 목표는 모든 domain 을 하나의 거대한 결과 타입으로 통합하는 것이 아니다. 목표는 `thread 친화적 결과`, `approval visibility`, `실패 구분`, `후속 action 힌트`를 공통 언어로 정리해 각 phase plan 과 실제 구현이 서로 다른 shape 로 흩어지지 않게 하는 것이다.

## 2. 왜 공통 shape 가 필요한가

현재 방향 A 문서들은 아래를 이미 전제로 둔다.

- assistant automation task 는 domain/action/input/output/approval policy/visibility summary 를 가져야 한다.
- 결과는 raw payload 가 아니라 사용자에게 설명 가능한 형식이어야 한다.
- approval pending, auth needed, external failure 같은 상태는 WebUI 상태 표면과 직접 연결된다.

하지만 5.3, 5.4, 5.5 를 각자 따로 구현하면 아래 문제가 생긴다.

- Gmail draft 결과와 Kakao action 결과가 서로 다른 성공/실패 vocabulary 를 쓸 수 있다.
- approval pending 이 domain 마다 다른 metadata key 나 다른 summary shape 로 흩어질 수 있다.
- WebUI status block, sidebar badge, linked session summary 가 domain 별 예외 처리로 늘어날 수 있다.

따라서 phase-1 에서는 `완전한 공통 런타임 타입`보다 먼저, `공통 설계 vocabulary` 와 `최소 필드 집합`을 고정하는 것이 맞다.

## 3. 설계 원칙

### 3.1 raw payload 우선이 아니라 thread 친화적 결과 우선

공통 shape 는 외부 API 응답 원문을 보존하는 구조가 아니라, thread 와 WebUI status 표면에 올릴 수 있는 설명 가능한 결과를 우선한다.

### 3.2 approval 과 result 를 분리하되 연결 가능해야 함

approval pending 은 최종 result 가 아니다. 하지만 approval lifecycle 이후의 completed / rejected / failed 결과와 같은 action id 또는 같은 summary 흐름으로 연결 가능해야 한다.

### 3.3 domain 특화 payload 는 summary 뒤로 숨김

Gmail thread, draft, Kakao payload 처럼 domain-specific detail 은 adapter 또는 domain layer 에 남기고, 공통 shape 는 summary 와 status 중심으로 제한한다.

### 3.4 session-local 과 assistant-global continuity 를 섞지 않음

공통 result shape 는 thread 와 status visibility 를 위한 표면이다. long-term memory 나 owner directory 를 직접 표현하지 않는다. continuity metadata 와 연결되더라도, 그것은 linked visibility 를 위한 보조 관계여야 한다.

## 4. 공통 개념 정리

### 4.1 action result

사용자 관점에서 assistant action 이 어떤 결과로 끝났는지 설명하는 정규화된 결과다.

최소 포함 요소:

- 어떤 action 이었는가
- 무엇을 하려고 했는가
- 실제로 무엇이 일어났는가
- 현재 상태가 무엇인가
- 다음에 사용자가 무엇을 할 수 있는가

### 4.2 action failure

실패는 단순 error text 가 아니라, user-facing summary 와 retry / approval / auth 연결이 가능한 종류 구분을 가진 결과다.

### 4.3 action visibility summary

thread inline status block, sidebar badge, linked session row, Kakao simple text 안내 같은 표면에서 재사용하는 짧은 상태 요약이다.

### 4.4 domain detail payload

Gmail thread id, Gmail draft body 전체, Kakao request payload 같은 세부 정보는 domain layer 에 남는다. 공통 shape 는 이 detail 을 직접 표준화하지 않고, 필요 시 reference 또는 normalized preview 만 들고 간다.

## 5. phase-1 공통 result shape 최소안

concept 수준 최소안:

```text
action_id
domain
action
status
title
summary
details
next_step
visibility
references
error
```

필드 의미:

- `action_id`
  - 하나의 action lifecycle 을 식별하는 id
- `domain`
  - `mail`, `kakao`, 이후 `calendar` 같은 domain 구분
- `action`
  - `list_important_threads`, `create_draft`, `send_message` 같은 action 이름
- `status`
  - 사용자에게 보일 현재 결과 상태
- `title`
  - thread 나 status block 에 바로 올릴 수 있는 짧은 제목
- `summary`
  - 1~3문장 수준의 사용자 친화적 요약
- `details`
  - preview 수준의 추가 정보
- `next_step`
  - 사용자가 다음으로 할 수 있는 제안 또는 시스템의 후속 요구
- `visibility`
  - 어떤 표면에 어떤 형태로 노출할지 힌트
- `references`
  - domain-specific id 나 linked session reference 의 얕은 참조
- `error`
  - 실패 또는 경고가 있을 때의 정규화된 정보

이 shape 는 런타임 객체, API payload, session metadata 를 지금 즉시 하나로 통합하자는 뜻이 아니다. phase-1 에서는 각 구현이 이 개념적 shape 를 공유 vocabulary 로 따르는 것이 목표다.

## 6. status vocabulary 최소안

phase-1 에서 공통으로 쓰는 상태는 아래 정도로 제한하는 것이 적절하다.

- `running`
- `waiting_approval`
- `completed`
- `failed`
- `rejected`
- `blocked`

domain 별 해석 예시:

- Gmail draft 생성 성공: `completed`
- Gmail send approval 대기: `waiting_approval`
- Gmail send 거절: `rejected`
- Gmail auth missing: `blocked`
- Kakao adapter unavailable: `failed`

`partial_success` 같은 추가 상태는 실제 요구가 생길 때만 도입한다.

## 7. failure code vocabulary 최소안

phase-1 에서 우선 공통으로 맞출 failure code 는 아래 정도면 충분하다.

- `approval_needed`
- `approval_rejected`
- `authentication_needed`
- `invalid_input`
- `not_found`
- `service_unavailable`
- `executor_unavailable`
- `rate_limited`
- `unknown_failure`

중요한 점은 raw exception class 가 아니라 user-facing policy 분기 기준이 되는 code 를 갖는 것이다.

## 8. visibility shape 최소안

공통 visibility 힌트는 아래 정도로 제한한다.

```text
surface
badge
inline_status
linked_summary
approval_summary
```

설명:

- `surface`
  - `thread`, `sidebar`, `linked_session`, `external_channel` 같은 대상 표면
- `badge`
  - 짧은 상태 badge 텍스트
- `inline_status`
  - thread 안에서 보여줄 상태 블록 정보
- `linked_summary`
  - 다른 session row 또는 linked summary 에서 보여줄 짧은 요약
- `approval_summary`
  - approval pending 과 approval 완료 흐름에서 재사용할 요약

phase-1 에서는 WebUI 를 primary visibility surface 로 두고, Kakao 같은 외부 채널은 `external_channel` surface 에 대한 simple text summary 정도만 맞춘다.

## 9. references shape 최소안

references 는 domain-specific id 를 공통 shape 안에 과도하게 평탄화하지 않기 위한 얕은 연결층이다.

예시:

- `thread_id`
- `draft_id`
- `message_id`
- `session_key`
- `canonical_owner_id`

원칙:

- reference 는 식별 및 링크용이다.
- raw payload, credential, 전체 request body 는 여기에 두지 않는다.

## 10. domain 적용 가이드

### 10.1 Gmail read-only + draft

대표 적용:

- `mail.list_important_threads`
- `mail.summarize_threads`
- `mail.create_draft`

권장 결과 해석:

- `title`: "Important mail summary ready", "Draft ready"
- `summary`: thread 요약 또는 초안 생성 요약
- `details`: sender / subject / draft preview 정도
- `next_step`: "Review draft", "Ask to send"
- `references`: `thread_id`, `draft_id`

### 10.2 Gmail send approval

대표 적용:

- `mail.send_message`

권장 결과 해석:

- approval 전: `waiting_approval`
- 승인 후 성공: `completed`
- 거절: `rejected`
- 외부 API 실패: `failed`

중요한 점은 `approval_summary` 와 최종 send result 를 같은 action lifecycle 아래에서 이어 보이게 하는 것이다.

### 10.3 Kakao adapter 최소 범위

대표 적용:

- Kakao inbound normalize
- Kakao outbound simple text summary
- approval 안내 문구

권장 결과 해석:

- Kakao 자체가 action executor 가 아니라 visibility surface 또는 channel adapter 인 경우, 공통 result 전체를 다 쓰지 않고 `title`, `summary`, `status`, `references.session_key` 정도만 소비해도 충분하다.

## 11. WebUI 연결 원칙

WebUI는 phase-1 에서 아래 규칙으로 공통 shape 를 소비하는 것이 적절하다.

- thread header / inline status block 은 `status`, `title`, `summary` 중심으로 본다.
- sidebar badge 는 `visibility.badge` 또는 `status` 기반의 짧은 표시만 사용한다.
- linked session summary 는 `linked_summary` 와 `references.session_key`, `canonical_owner_id` 중심으로 본다.
- approval pending 은 기존 `approval_summary` 표면을 재사용하되, 최종 action result 와 연결 가능한 `action_id` 관점을 유지한다.

## 12. session metadata 연결 원칙

phase-1 에서 session metadata 와 연결할 때는 아래 원칙을 따른다.

- continuity metadata 는 owner / channel identity 연결에만 쓴다.
- approval pending 은 기존 `session.metadata["approval_summary"]` 표면을 우선 재사용한다.
- final action result 전체를 session metadata 에 그대로 적재하지 않는다.
- session metadata 에 남길 것은 linked visibility 와 resume 에 필요한 최소 요약만 둔다.

즉, `result object 전체 저장`보다 `visibility / resume 에 필요한 최소 metadata 저장`이 기본 원칙이다.

## 13. 구현 시 주의점

- 공통 shape 를 이유로 domain detail 을 과도하게 평탄화하지 않는다.
- approval state 와 final result state 를 같은 필드에 뭉개지 않는다.
- WebUI 편의 때문에 외부 채널 formatter 를 복잡하게 만들지 않는다.
- 실패 code 를 너무 세분화해 phase-1 구현을 무겁게 만들지 않는다.

## 14. 완료 기준

이 문서 기준 완료는 아래를 뜻한다.

1. 5.3, 5.4, 5.5 가 공통으로 참조할 result / failure / visibility vocabulary 가 정의된다.
2. 최소 status / failure code / visibility / reference shape 가 정리된다.
3. WebUI 와 session metadata 연결 원칙이 정리된다.
4. domain-specific detail 과 공통 summary 의 경계가 정리된다.

## 15. 다음 단계 연결

이 문서가 생기면 다음 단계는 아래로 이어진다.

- 5.3 Gmail read-only + draft contract 구현
- 5.4 send approval visibility / result 연결
- 5.5 Kakao adapter simple text summary / approval 안내 연결

## 16. 현재 code draft 메모

현재 source 기준 code draft 는 아래 파일에 두었다.

- `source/nanobot/automation_results.py`

현재 반영 범위:

- 공통 `ActionResult`, `ActionFailure`, `ActionVisibility`, `ActionReferences` 초안
- 5.3용 `MailImportantThreadsResult`, `MailSummarizeThreadsResult` 초안
- focused test `source/tests/test_automation_results.py`

이 단계에서는 runtime orchestration 이나 API surface 연결까지는 하지 않고, domain contract 와 result vocabulary 를 코드 타입으로 먼저 고정하는 데 집중한다.