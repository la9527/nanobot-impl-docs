# Nanobot Single-User Operating Policy

## 상태 메모

- 이 문서는 방향 A 시기의 단일 사용자 운영 원칙을 정리한 reference 문서다.
- 2026-05-01 기준 실제 구현 반영 상태는 `docs/todo.md` section 5.2 와 `docs/execution-backlog/02-owner-memory-and-task-backbone-phase1.md` 를 우선 본다.
- 본문은 여전히 유효한 운영 전제를 설명하지만, active task tracking 문서로 사용하지는 않는다.

## 1. 문서 목적

이 문서는 방향 A 기준에서 Nanobot를 `로컬 self-hosted`, `단일 사용자` assistant 런타임으로 운영하기 위한 정책을 정리한다.

핵심 목적은 두 가지다.

- 이후 assistant 기능 확장 시 multi-user 전제를 무의식적으로 넣지 않게 하기
- session, memory, approval, channel continuity를 단일 사용자 기준으로 일관되게 설계하기

이 문서는 구현 문서라기보다 운영과 설계의 기본 전제 문서다.

## 2. 기본 전제

이 정책은 아래 조건을 고정한다.

- 실제 사용자 주체는 한 사람이다.
- 여러 채널이 있어도 모두 같은 사용자의 입력 경로일 가능성이 높다.
- 권한 시스템보다 `내 assistant가 내 대신 무엇을 할 수 있는가`가 더 중요하다.
- 상태 저장은 우선 file/session/memory 기반으로 유지한다.
- 데이터베이스나 조직 계층은 기본 가정에서 제외한다.

## 3. 사용자 모델

방향 A 기준 Nanobot의 기본 사용자 모델은 아래처럼 본다.

- primary user: assistant의 실질 소유자이자 주 사용자
- channel endpoint: WebUI, Telegram, Slack, Email, 이후 Kakao 등 입력 채널
- session: 채널별 대화 단위
- assistant memory: primary user와 assistant의 장기 맥락

중요한 점은 `channel account != 서로 다른 사용자`로 간주하지 않는다는 것이다.

예를 들어 아래는 기본적으로 같은 사람으로 본다.

- WebUI local profile
- Telegram personal account
- Slack DM account
- 개인 Email 주소

즉, 기본 가정은 `한 사용자가 여러 문으로 assistant에 들어온다`이다.

## 4. 세션 정책

### 4.1 기본 원칙

- session은 여전히 채널별로 분리한다.
- 다만 session의 상위 개념은 단일 사용자 assistant context다.
- session 분리는 UI/대화 흐름을 위한 것이지, 서로 다른 사람을 뜻하지 않는다.

### 4.2 session 분리 이유

아래 이유로 session은 계속 유지한다.

- 채널별 응답 제약이 다르다.
- thread/history 표시 방식이 다르다.
- tool approval 재개 위치가 다를 수 있다.
- channel transport 오류나 reconnect를 분리해서 다뤄야 한다.

### 4.3 session 상위 규칙

단일 사용자 기준에서는 아래를 우선한다.

- memory는 session보다 assistant-global 해석을 우선 검토한다.
- 선호 설정은 per-session보다 global default가 우선이다.
- 특별히 충돌하지 않으면 approval 대상 작업은 same-owner action으로 본다.

## 5. Memory 정책

### 5.1 memory의 주체

memory는 채널별 사용자가 아니라 `primary user와 assistant 관계`를 기준으로 저장하는 것이 맞다.

즉, memory는 아래 우선순위로 본다.

1. assistant-global long-term memory
2. task/session-local temporary context
3. channel-specific transport metadata

### 5.2 channel-specific 데이터의 취급

아래 정보는 memory 본문보다 transport metadata에 가깝다.

- Telegram user id
- Slack DM id
- Email thread id
- Kakao user key

이 값은 동일 인물 여부를 추론하는 힌트는 될 수 있지만, 장기 메모리의 핵심 내용으로 다루지 않는다.

### 5.3 memory 반영 원칙

- 개인 선호, 장기 프로젝트, 지속적 습관은 assistant-global memory 후보
- 특정 채널의 일회성 제약은 session-local 문맥으로 처리
- channel-specific identifier는 continuity metadata로 분리

## 6. Approval 정책

### 6.1 기본 원칙

approval은 다중 사용자 결재 시스템이 아니라, `내 assistant가 내 대신 작업해도 되는지 확인하는 절차`로 본다.

### 6.2 approval의 의미

단일 사용자 기준 approval은 아래 의미를 가진다.

- destructive action 확인
- 외부 서비스 변경 확인
- 로컬 환경 실행 확인
- 민감 정보 접근 확인

### 6.3 approval 저장 방식

- 우선은 session/pending queue 중심 구조 유지
- DB 기반 approval ticket은 도입하지 않음
- 단, cross-channel 재개 수요가 반복되면 lightweight metadata 확장 정도는 검토 가능

## 7. Channel continuity 정책

### 7.1 목표

여러 채널을 쓰더라도 사용자는 하나의 assistant를 쓰고 있다고 느껴야 한다.

### 7.2 continuity 최소 기준

- assistant memory는 채널을 넘어 이어질 수 있어야 한다.
- active project 문맥은 다른 채널에서 재호출 가능해야 한다.
- approval 대기 상태는 최소한 WebUI에서 가시화할 수 있어야 한다.

### 7.3 continuity를 위해 필요한 최소 메타

필요 시 아래 수준의 lightweight mapping을 도입할 수 있다.

- canonical owner id: 로컬 단일 사용자 식별자
- channel kind
- external channel user id
- trust level 또는 수동 연결 여부
- last confirmed timestamp

중요한 점은 이것이 multi-user IAM이 아니라, 단일 사용자 continuity 보조 메타라는 것이다.

## 8. 설정 정책

단일 사용자 기준 설정은 아래 우선순위를 가진다.

1. global assistant defaults
2. session override
3. channel transport-specific constraints

예시:

- 기본 모델은 global default
- 특정 대화에서만 smart-router 사용은 session override
- Kakao 응답 길이 제한은 channel constraint

## 9. 비목표

이 정책 문서에서 의도적으로 제외하는 것은 아래다.

- 팀 단위 협업 계정 모델
- 조직별 권한 분리
- 역할 기반 접근 제어
- 다중 사용자 memory 격리 체계
- 중앙 DB 기반 user directory

## 10. 구현 시 체크리스트

새 기능을 추가할 때는 아래를 먼저 확인한다.

1. 이 기능이 실제로 multi-user를 요구하는가?
2. 단일 사용자 가정으로 단순화할 수 없는가?
3. channel-specific 정보와 assistant-global 정보를 분리했는가?
4. approval이 결재 흐름이 아니라 self-confirmation 흐름으로 유지되는가?
5. DB나 workflow engine 없이도 구현 가능한가?

## 11. 최종 정책 요약

방향 A 기준 Nanobot는 `여러 채널을 가진 단일 사용자 assistant`로 본다.

따라서 아래를 유지한다.

- session은 채널별로 유지
- memory는 assistant-global 우선
- approval은 self-confirmation 우선
- continuity는 lightweight metadata로 보강
- multi-user 구조는 후순위

이 원칙을 지키면 Nanobot는 복잡한 플랫폼이 아니라, 일관된 개인용 assistant 런타임으로 발전할 수 있다.