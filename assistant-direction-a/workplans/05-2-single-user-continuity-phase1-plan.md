# 05-2 Single-User Continuity Phase 1 Plan

## 상태 메모

- 상태: 완료, active backlog 에서 흡수됨
- 현재 source of truth: `docs/todo.md` section 5.2, `docs/execution-backlog/02-owner-memory-and-task-backbone-phase1.md`
- 이 문서는 초기 phase-1 계획과 metadata 경계의 배경 설명을 남기는 archive reference 다.

## 1. 문서 목적

이 문서는 방향 A 구현 백로그의 `5.2 single-user continuity / identity mapping metadata`를 1차 구현 단위로 정리한 작업 진행 방안 문서다.

이번 1차 목표는 full identity system을 만드는 것이 아니라, 단일 사용자 assistant continuity를 보강할 수 있는 최소 metadata 경계를 정하는 것이다.

핵심 질문은 아래다.

- 여러 채널 세션을 같은 owner 관점으로 볼 최소 메타는 무엇인가?
- 이 메타를 어디에 저장해야 현재 구조를 해치지 않는가?
- approval visibility와 continuity를 어디까지 연결할 것인가?

## 2. 이번 1차 범위

1차 범위에 포함할 것:

- canonical owner 개념의 최소 정의
- channel identity metadata 최소 스키마 정의
- session-local state 와 assistant-global continuity 경계 정의
- approval pending visibility와 continuity의 최소 연결 규칙 정의
- metadata 저장 위치 결정

1차 범위에서 제외할 것:

- full user directory
- multi-user IAM
- cross-tenant auth model
- 완전한 linked sessions UI
- 모든 채널 자동 매칭

## 3. 1차에서 결정해야 할 핵심 사항

### 3.1 canonical owner 최소 정의

이번 단계에서는 owner를 아래처럼 단순화한다.

- `primary-user`

의미:

- WebUI, Telegram, Slack, Email, 이후 Kakao까지 모두 기본적으로 같은 사용자의 진입점으로 본다.

이번 단계에서 필요한 것은 이 owner를 복잡하게 관리하는 것이 아니라, continuity의 기준점으로 삼는 것이다.

### 3.2 channel identity metadata 최소 스키마

1차 최소안:

- channel kind
- external identity
- canonical owner id
- trust level
- last confirmed timestamp

선택 필드 후보:

- link source
- notes

이번 단계에서는 이 스키마를 실제 persistence format으로 확정하기보다, runtime에서 요구되는 최소 단위로 본다.

### 3.3 session-local 과 assistant-global 경계

이번 단계에서 명확히 나눌 것:

- session-local
  - thread history
  - channel transport state
  - temporary draft state
- assistant-global
  - long-term memory
  - approval visibility summary
  - recent action continuity hint

중요한 점은 continuity가 memory 확장 그 자체가 아니라는 점이다.

현재 1차 구현 기준으로는 아래처럼 해석한다.

- assistant-global continuity metadata
  - `session.metadata["continuity"]`
  - `session.metadata["approval_summary"]`
- session-local state
  - thread history
  - channel transport state
  - `session.metadata["pending_user_turn"]`
  - `session.metadata["runtime_checkpoint"]`

즉, phase-1 의 assistant-global continuity 는 별도 owner store 나 long-term memory 본문이 아니라, 여러 entrypoint 에서 같은 owner 문맥을 이어 보기 위한 최소 metadata layer 로 한정한다.

## 4. 작업 분해

### Step 1. continuity metadata 필요 지점 파악

해야 할 일:

- WebUI가 어떤 외부 session 정보를 필요로 하는지 정리
- approval visibility가 cross-session으로 보이려면 어떤 키가 필요한지 정리
- Gmail/Kakao 같은 future domain에서 continuity metadata가 어디에 쓰일지 정리

현재 phase-1 에서 확인된 소비 지점은 아래로 제한한다.

- WebUI linked external session summary
- WebUI sidebar session row 의 pending approval visibility
- 이후 Gmail pilot / Kakao adapter 에서 same-owner session 선택 또는 linked identity 힌트로 재사용 가능한 최소 metadata 기반

단, Gmail/Kakao 쪽의 실제 consumer contract 는 이번 5.2 phase-1 범위에서는 확정하지 않고 후속 도메인 작업에서 정한다.

완료 기준:

- continuity metadata 소비 지점이 문서화됨

### Step 2. 최소 metadata shape 정의

이번 단계에서 문서/코드 초안으로 정할 shape:

```text
canonical_owner_id
channel_kind
external_identity
trust_level
last_confirmed_at
```

결정 포인트:

- trust level 을 enum처럼 둘지 plain text로 둘지
- external identity 를 raw string으로 둘지 normalized field로 둘지

완료 기준:

- backend metadata 초안이 정의됨

### Step 3. 저장 위치 결정

후보:

- session metadata 안에 넣기
- 별도 lightweight store 만들기

판단 기준:

- 현재 구조를 가장 적게 흔드는가
- cross-session visibility 요구를 수용하는가
- debugging 이 쉬운가

1차 권장:

- 우선 lightweight 별도 store 또는 dedicated metadata file 방향 검토
- 단, 실제 첫 구현은 session metadata 기반 임시 연결도 허용

완료 기준:

- 1차 구현 저장 위치가 결정됨

### Step 4. approval visibility 와의 연결 규칙 정의

해야 할 일:

- pending approval 을 같은 owner 관점으로 재노출할 최소 규칙 정의
- 승인 위치와 실행 위치를 분리할지 여부 정리
- 우선순위를 `execute anywhere`가 아니라 `visible from WebUI`에 둠

완료 기준:

- approval continuity의 1차 규칙이 정의됨

### Step 5. WebUI 노출 후보 정의

이번 단계에서 실제 UI 구현까지 전부 가지 않아도 된다. 다만 아래를 정의한다.

- linked owner/session summary가 어디에 들어갈지
- channel badge와 연동 가능한지
- external pending action summary가 필요한지

완료 기준:

- 5-1 WebUI 작업과 연결 가능한 continuity UI 후보가 정리됨

### Step 6. focused backend test 계획 정의

테스트 후보:

- canonical owner metadata read/write
- session-local 과 assistant-global 구분 검증
- pending approval summary가 owner 기준으로 연결 가능한지 검증

현재 1차 구현에서는 마지막 항목을 아래 수준으로 닫는다.

- 같은 `canonical_owner_id=primary-user` 를 갖는 WebUI/Telegram 세션이 함께 노출될 때, pending approval 은 aggregate store 없이도 linked external row 에서 확인 가능해야 한다.
- 이번 단계에서는 owner-level aggregate index 나 unloaded session fan-in 까지는 요구하지 않는다.

완료 기준:

- 실제 구현 시 바로 테스트로 옮길 수 있는 체크리스트가 준비됨

## 5. 권장 구현 순서

1. continuity metadata 필요 지점 파악
2. 최소 metadata shape 정의
3. 저장 위치 결정
4. approval visibility 연결 규칙 정의
5. WebUI 노출 후보 정의
6. backend test 계획 정리

## 6. 구현 시 주의점

- 1차 단계에서는 full identity system으로 커지지 않게 한다.
- memory 본문과 continuity metadata를 섞지 않는다.
- approval ticket system으로 확대 해석하지 않는다.
- WebUI control surface와 연결되더라도 admin UI처럼 만들지 않는다.

## 7. 완료 기준

이번 1차 문서 기준 완료는 아래를 뜻한다.

1. canonical owner 와 channel identity metadata 최소안이 정의된다.
2. 저장 위치가 결정된다.
3. approval visibility 와 continuity 의 최소 연결 규칙이 정의된다.
4. 5-1 WebUI 상태 문서와 자연스럽게 연결된다.
5. 실제 backend 테스트 항목으로 옮길 수 있는 체크리스트가 준비된다.

## 8. 다음 단계 연결

이 작업이 끝나면 다음 단계는 아래로 이어진다.

- continuity metadata의 실제 runtime 반영
- WebUI에 linked session / external pending summary 노출
- Gmail pilot 및 Kakao adapter에서 same-owner continuity 재사용

## 9. 현재 1차 구현 메모

현재 source 기준 1차 구현은 아래처럼 잡는다.

- canonical owner 는 `primary-user` 로 고정한다.
- continuity metadata 저장 위치는 우선 `session.metadata["continuity"]` 로 둔다.
- 최소 필드는 아래로 고정한다.

```text
canonical_owner_id
channel_kind
external_identity
trust_level
last_confirmed_at
```

- `SessionManager` save/load/list/read 경로에서 continuity metadata 가 항상 보정되도록 한다.
- WebUI HTTP surface 는 `/api/sessions`, `/api/sessions/:key/messages` 에서 이 metadata 를 그대로 노출한다.
- WebUI linked external session summary 는 owner / linked identity / trust 수준을 읽어 보여주는 최소 표면까지만 연결한다.
- approval pending 의 cross-session visibility 는 이번 단계에서 `session.metadata["approval_summary"]` + WebUI sidebar badge + click-to-expand inline summary 까지 연결한다.
- 별도 lightweight store 는 아직 도입하지 않는다. 현재 단계에서는 session metadata 만으로 충분하고, same-owner 기준의 unloaded session aggregation 또는 dedicated owner index 가 실제 요구로 생길 때 분리한다.
- same-owner 관점 검증은 focused route test 로 우선 닫고, owner-level aggregate UX 는 실제 요구가 생길 때 다음 단계에서 분리한다.