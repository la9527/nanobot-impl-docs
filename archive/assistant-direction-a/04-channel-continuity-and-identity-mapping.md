# Nanobot Channel Continuity And Identity Mapping

## 상태 메모

- 이 문서는 continuity/identity mapping 을 multi-user directory 가 아니라 single-user owner-link metadata 로 제한한 배경을 설명하는 reference 문서다.
- 2026-05-01 기준 실제 구현 반영 상태는 `docs/planning/todo.md` section 5.2 와 `docs/planning/execution-backlog/02-owner-memory-and-task-backbone-phase1.md` 를 우선 본다.
- 본문은 active 구현 항목이 아니라 continuity 경계의 설계 rationale 을 보존하는 historical context 로 유지한다.

## 1. 문서 목적

이 문서는 방향 A 기준에서 Nanobot의 `channel continuity`와 `identity mapping`을 어떻게 해석하고, 어느 수준까지 구현할지 정리한다.

핵심 질문은 아래와 같다.

- WebUI, Telegram, Slack, Email, 이후 Kakao를 오갈 때 같은 assistant 경험을 어떻게 유지할 것인가
- 채널별 식별자를 어디까지 같은 사용자로 묶을 것인가
- 다중 사용자 플랫폼이 아니라 단일 사용자 assistant 구조를 유지하면서 continuity를 어떻게 보강할 것인가

## 2. 방향 A 기준 결론

방향 A에서 continuity의 목적은 `서로 다른 계정을 가진 여러 사용자 식별`이 아니다. 목적은 `한 사용자가 여러 채널을 통해 들어와도 같은 assistant를 쓰고 있다고 느끼게 하는 것`이다.

따라서 이 문서의 기본 결론은 아래와 같다.

- session은 채널별로 유지한다.
- continuity는 assistant-global 맥락을 보강하는 계층으로 둔다.
- identity mapping은 IAM이나 user directory가 아니라 lightweight owner-link metadata로 제한한다.
- multi-user 전제를 넣지 않는다.

## 3. 문제 정의

Nanobot는 이미 채널별 session을 잘 다룬다. 하지만 방향 A에서 assistant 기능이 커질수록 아래 문제가 생길 수 있다.

- Telegram에서 하던 대화를 WebUI에서 이어보고 싶다.
- Email에서 요청한 초안 작업을 WebUI에서 승인하고 싶다.
- Slack에서 시작한 프로젝트 문맥을 다음 날 WebUI에서 계속 쓰고 싶다.
- Kakao가 들어오면 그것도 같은 사용자 assistant 문맥에 연결하고 싶다.

즉, 문제는 `대화를 어디서 했는지`보다 `같은 assistant owner의 연속된 작업인지`를 유지하는 것이다.

## 4. 핵심 개념

### 4.1 session

session은 채널별 대화 단위다.

예시:

- WebUI thread
- Telegram DM thread
- Slack DM thread
- Email thread

session의 책임은 아래다.

- 채널 transport 유지
- thread/history 유지
- channel-local constraint 유지
- immediate approval resume 위치 유지

### 4.2 continuity

continuity는 서로 다른 session 위에서 `같은 사용자 assistant 맥락`을 느끼게 하는 상위 개념이다.

continuity가 다루는 것은 아래다.

- assistant-global memory
- 현재 활성 프로젝트나 ongoing task
- approval/action visibility의 재노출
- 특정 선호 설정의 재사용

### 4.3 identity mapping

identity mapping은 여러 채널 식별자를 하나의 canonical owner로 연결하는 최소 메타 계층이다.

여기서 중요한 점은 아래다.

- account management가 아니다.
- 팀 사용자 디렉터리가 아니다.
- authorization system이 아니다.
- single-user continuity 보조 수단이다.

## 5. 방향 A에서의 owner 모델

### 5.1 canonical owner

방향 A에서는 아래처럼 하나의 canonical owner를 상정하는 것이 맞다.

- `primary-user`

이 owner는 실제 사람 하나를 가리킨다. WebUI, Telegram, Slack, Email, Kakao에서 들어오더라도 기본적으로 같은 사람으로 본다.

### 5.2 channel identity

channel identity는 채널이 제공하는 외부 식별자다.

예시:

- WebUI local profile id
- Telegram user id
- Slack user id
- Email address 또는 thread sender
- Kakao user key

### 5.3 link record

canonical owner와 channel identity의 연결은 link record로 볼 수 있다.

개념적으로는 아래 속성을 갖는다.

```text
canonical_owner_id
channel_kind
external_identity
link_source
trust_level
last_confirmed_at
notes
```

## 6. continuity에서 이어져야 하는 것

### 6.1 이어져야 하는 것

- assistant-global long-term memory
- active project or ongoing task context
- 기본 모델/assistant 선호
- approval pending visibility
- recent action summary

### 6.2 굳이 이어지지 않아도 되는 것

- 채널별 읽음 상태
- 채널별 reply formatting 제약
- session-local draft state 일부
- transport retry state

### 6.3 이어지면 안 되는 것

- 채널 특수 payload raw data
- 외부 서비스 고유 토큰/민감 식별자
- debug log 전체

## 7. continuity 최소 구현 수준

방향 A 초기 구현에서는 아래 수준이면 충분하다.

### 7.1 Level 1: policy-only continuity

특징:

- 문서와 운영 규칙으로 continuity를 해석
- assistant-global memory 우선
- session은 독립 유지

적합 시점:

- 지금 즉시

### 7.2 Level 2: lightweight owner-link metadata

특징:

- canonical owner와 channel identity를 metadata로만 연결
- WebUI에서 연결 상태를 보조적으로 볼 수 있음
- approval / action summary 재노출 가능

적합 시점:

- Kakao, Email, Telegram, Slack 간 continuity가 실제로 중요해질 때

### 7.3 Level 3: continuity-aware assistant surfaces

특징:

- WebUI가 외부 채널 세션을 same-owner 관점으로 묶어 보여줌
- assistant action과 pending approval을 cross-session에서 쉽게 확인

적합 시점:

- WebUI를 assistant control surface로 강화할 때

## 8. approval과 continuity의 연결

방향 A에서 continuity가 필요한 가장 큰 이유 중 하나는 approval이다.

예시:

- Email에서 `초안 보내도 될까?`가 발생
- 실제 승인은 WebUI에서 하고 싶음
- assistant는 같은 owner의 승인으로 간주해야 함

따라서 continuity는 아래를 최소 지원해야 한다.

- pending approval이 다른 session에서도 보일 수 있음
- 승인 주체는 같은 canonical owner로 해석 가능
- 실행 위치와 승인 위치는 분리 가능

단, 초기에는 `어디서든 승인 가능`까지 바로 가지 않아도 된다. 우선은 `WebUI에서 visibility 확보`가 1순위다.

## 9. memory와 continuity의 연결

continuity는 memory를 더 넓게 쓰는 면허가 아니다.

원칙:

- 장기 memory는 owner 기준으로 본다.
- session-local 맥락은 계속 session 안에 둔다.
- continuity metadata는 memory 본문과 분리한다.

즉, continuity는 `같은 사람이다`를 보조할 뿐, 모든 channel-local 흔적을 장기 memory로 승격시키지 않는다.

## 10. WebUI에서의 권장 표면

WebUI는 continuity의 첫 번째 control surface가 되어야 한다.

권장 표시 항목:

- current session의 channel kind
- same-owner linked session 존재 여부
- pending approval 존재 여부
- recent external activity summary

초기에는 아래 정도면 충분하다.

- channel badge
- linked owner badge 또는 summary
- external pending action summary

## 11. Kakao와 continuity

Kakao를 넣을 경우 continuity 설계는 더 중요해진다.

이유:

- Kakao 안에서는 rich control surface가 약할 수 있다.
- approval과 task visibility를 WebUI로 넘기고 싶을 수 있다.
- Kakao session을 same-owner continuity에 연결해야 assistant 경험이 유지된다.

따라서 Kakao는 channel plugin 구현보다 continuity policy가 먼저 정리되어야 한다.

## 12. 구현 순서 제안

### 12.1 1차

- continuity policy 문서화
- canonical owner 개념 도입
- channel identity를 continuity metadata로 분리

### 12.2 2차

- WebUI status에 external continuity 요약 추가
- pending approval cross-session visibility 연결

### 12.3 3차

- lightweight owner-link metadata 저장 구조 도입
- Email, Telegram, Slack, Kakao 순으로 연동 검토

## 13. 비목표

이번 문서에서 의도적으로 하지 않는 것은 아래다.

- full identity provider 시스템
- multi-user IAM
- 조직/팀 user directory
- cross-tenant permission model
- centralized auth database

## 14. 최종 판단

방향 A에서 channel continuity는 `여러 채널을 쓰는 한 명의 사용자를 위해 assistant 경험을 이어주는 얇은 계층`이어야 한다.

핵심은 아래다.

1. session은 채널별로 유지한다.
2. continuity는 assistant-global 문맥을 보강한다.
3. identity mapping은 lightweight owner-link metadata로 제한한다.
4. approval visibility와 WebUI control surface를 우선 연결한다.
5. multi-user 구조로 확장하지 않는다.

이 원칙을 지키면 Nanobot는 방향 A의 핵심인 local-first single-user assistant 구조를 유지하면서도, 채널 이동에 덜 끊기는 assistant 경험을 제공할 수 있다.