# 05-5 Kakao Adapter Minimal Scope Phase 1 Plan

## 상태 메모

- 상태: 보류, reference-only
- 현재 active implementation backlog 로는 승격되지 않았고, section 5 의 선행 환경 준비와 Kakao feasibility 참고 문서만 유지 중이다.
- Kakao 구현을 다시 시작할 때는 먼저 `docs/todo.md` section 5 또는 새 execution backlog item 으로 active slice 를 만든 뒤 이 문서를 reference 로 갱신한다.

## 1. 문서 목적

이 문서는 방향 A 구현 백로그의 `5.5 Kakao adapter 최소 범위 준비`를 1차 구현 단위로 정리한 작업 진행 방안 문서다.

공통 result / failure / visibility vocabulary 는 `docs/assistant-direction-a/08-shared-action-result-and-failure-shape.md` 를 따른다.

이번 1차 목표는 Kakao 를 Nanobot core channel 로 깊게 넣지 않고, optional integration 으로 붙일 수 있는 최소 adapter 범위를 고정하는 것이다.

핵심 질문은 아래다.

- Kakao inbound / outbound / approval 안내를 core 오염 없이 어디까지 수용할 것인가?
- channel plugin, external adapter, MCP-backed connector 중 어느 경로가 1차 범위에 맞는가?
- single-user continuity 와 Kakao session 을 어떤 최소 metadata 로 연결할 것인가?

## 2. 이번 1차 범위

1차 범위에 포함할 것:

- Kakao inbound payload normalize shape 정의
- Kakao session key 정책 초안
- simple text 중심 outbound formatter 초안 정의
- approval 안내를 Kakao에서 어떻게 노출할지 1차 결정
- Kakao 와 WebUI continuity 연결 최소 규칙 정의
- channel plugin vs external adapter 경계 결정

1차 범위에서 제외할 것:

- 복잡한 card / quick reply 기반 full UX
- Kakao 안에서 모든 approval 을 끝내는 전용 시스템
- long-running workflow orchestration
- multi-user account directory
- Kakao 중심 core runtime 재구성

## 3. 1차에서 결정해야 할 핵심 사항

### 3.1 integration 형태 선택

후보는 아래다.

- channel plugin
- external adapter + normalized gateway bridge
- MCP-backed connector

1차 판단 기준:

- Kakao 특수성을 core 밖으로 충분히 밀어낼 수 있는가
- webhook / outbound credential / response formatting 을 격리할 수 있는가
- launchd 기반 현재 운영 흐름과 충돌하지 않는가

1차 권장:

- Kakao-specific logic 는 adapter 경계에 최대한 둔다.
- Nanobot core 는 normalized inbound / outbound shape 와 continuity metadata 만 이해한다.

### 3.2 inbound normalize shape

phase-1 최소 inbound shape 는 아래 정도면 충분하다.

- channel = `kakao`
- external user id 또는 동등 식별자
- utterance text
- request id 또는 delivery trace id
- raw payload reference 여부

이 단계에서는 Kakao raw payload 전체를 session history 나 generic message payload 로 승격하지 않는다.

### 3.3 session key 와 continuity 규칙

phase-1 에서는 Kakao session key 를 아래처럼 다루는 것이 적절하다.

- `kakao:<external-identity>` 기본형

continuity 연결 최소안:

- `canonical_owner_id=primary-user`
- `channel_kind=kakao`
- `external_identity=<kakao-user-key>`
- `trust_level` 은 별도 link 신뢰도 정책에 따라 결정

즉, Kakao 는 5.2 continuity metadata 를 소비하는 새 채널일 뿐, 별도 identity system 의 시작점이 아니다.

### 3.4 outbound formatter 범위

phase-1 에서 outbound 는 아래 범위로 제한한다.

- simple text 응답
- approval 안내 또는 WebUI 유도 문구
- action result summary

card / quick reply 는 필요성만 적고, 초기 구현 범위에는 넣지 않는다.

### 3.5 approval 안내 방식

phase-1 에서 먼저 정해야 할 것은 Kakao 안에서 approve/reject 를 완결하는지가 아니라, approval 을 어떻게 안내할지다.

1차 권장:

- Kakao 에서는 `승인은 WebUI에서 확인` 중심 안내를 우선한다.
- 필요 시 simple text 기반 `pending approval` 요약만 보낸다.
- 실제 승인 실행 표면은 WebUI 를 기준으로 유지한다.

## 4. 작업 분해

### Step 1. Kakao integration 경계 정리

해야 할 일:

- plugin / adapter / MCP 중 1차 경로 선택
- Kakao-specific webhook / formatter / auth 책임이 어디에 있는지 정리
- core runtime 이 무엇만 알면 되는지 정리

완료 기준:

- Kakao integration boundary 가 문서화됨

### Step 2. inbound normalize shape 정의

해야 할 일:

- utterance 추출 필드 정의
- session key 생성 기준 정의
- trace / debug 용 최소 메타 정의

완료 기준:

- Kakao inbound 를 공통 message shape 로 변환하는 기준이 정리됨

### Step 3. outbound formatter 최소안 정의

해야 할 일:

- simple text 응답 형식 정의
- action status / failure summary 길이 제한 기준 정리
- rich response 는 후속 범위로 분리

완료 기준:

- Kakao outbound 최소 formatter 기준이 정리됨

### Step 4. continuity / approval 연결 규칙 정의

해야 할 일:

- Kakao session 이 5.2 continuity metadata 를 어떻게 재사용하는지 정리
- pending approval 을 Kakao에서 어디까지 요약할지 정리
- WebUI approval surface 와의 역할 분리를 정리

완료 기준:

- Kakao 와 WebUI continuity / approval 연결 규칙이 정리됨

### Step 5. focused backend test 계획 정의

테스트 후보:

- Kakao payload normalize 검증
- session key 생성 규칙 검증
- simple text formatter 길이 / summary 검증
- continuity metadata 연결 검증
- approval 안내 문구 normalization 검증

완료 기준:

- 실제 구현 시 바로 테스트로 옮길 수 있는 체크리스트가 준비됨

## 5. 권장 구현 순서

1. Kakao integration 경계 정리
2. inbound normalize shape 정의
3. outbound formatter 최소안 정의
4. continuity / approval 연결 규칙 정의
5. focused backend test 계획 정리

## 6. 구현 시 주의점

- Kakao 요구 때문에 core runtime 구조를 바꾸지 않는다.
- rich response 를 초기에 약속하지 않는다.
- Kakao approval UX 를 WebUI approval surface 와 경쟁시키지 않는다.
- single-user continuity 를 multi-user account system 으로 확대 해석하지 않는다.

## 7. 완료 기준

이번 1차 문서 기준 완료는 아래를 뜻한다.

1. Kakao integration boundary 가 정리된다.
2. inbound normalize shape, session key, outbound formatter 최소안이 정의된다.
3. continuity / approval 연결 규칙이 정리된다.
4. simple text 중심 초기 scope 가 고정된다.
5. 실제 backend 테스트 항목으로 옮길 수 있는 체크리스트가 준비된다.

## 8. 다음 단계 연결

이 작업이 끝나면 다음 단계는 아래로 이어진다.

- Kakao adapter 또는 bridge 의 실제 구현
- continuity metadata 를 이용한 linked session 보강
- WebUI approval surface 와 Kakao 안내 문구 연결

## 9. 현재 1차 구현 메모

현재 source 기준 1차 구현은 아래처럼 잡는다.

- Kakao 는 optional integration 으로 다루고, core channel 로 고정하지 않는다.
- 초기 구현은 simple text 중심으로 제한한다.
- approval 은 Kakao 내부 완결보다 WebUI 유도형 안내를 우선한다.
- continuity 는 `session.metadata["continuity"]` 재사용 원칙 위에 얹는다.
- Kakao-specific payload / formatter / credential handling 은 adapter 경계에 가둔다.
- rich card / quick reply 는 feasibility 와 후속 단계로 넘긴다.