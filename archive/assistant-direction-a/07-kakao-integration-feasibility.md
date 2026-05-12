# Nanobot Direction A Kakao Integration Feasibility

## 상태 메모

- 이 문서는 Kakao 를 active implementation backlog 가 아니라 optional integration feasibility 로 다뤘던 판단을 보존하는 reference 문서다.
- 2026-05-01 기준 Kakao 는 아직 active execution backlog 로 승격되지 않았고, section 5 에서도 구현 slice 가 열려 있지 않다.
- 따라서 본문은 현재 구현 계획서가 아니라, Kakao 를 다시 열 때 참고할 제약/경계 정리 문서로 읽는다.

## 1. 문서 목적

이 문서는 방향 A 기준에서 Kakao를 Nanobot core 안에 강하게 묶지 않고, optional channel integration으로 도입할 수 있는지 검토하기 위한 요구 정리 문서다.

핵심 질문은 아래다.

- Kakao를 Nanobot에 넣어도 방향 A 원칙을 깨지 않는가?
- 넣는다면 core 변경보다 adapter/plugin 경로가 가능한가?
- 어떤 제약 때문에 Kakao는 다른 채널보다 별도 설계가 필요한가?

## 2. 방향 A 기준 결론

방향 A에서는 Kakao를 `필수 core channel`이 아니라 `optional integration channel`로 다루는 것이 맞다.

이 판단의 이유는 아래와 같다.

- Nanobot의 현재 강점은 local-first single-user assistant runtime이다.
- Kakao는 채널 제약, 응답 format 제약, webhook 연동, 세션 식별 방식이 강한 외부 조건을 가진다.
- 이를 core에 직접 녹이면 Nanobot runtime이 Kakao 중심 구조로 오염될 수 있다.

따라서 Kakao는 아래 중 하나로 다루는 것이 적절하다.

- channel plugin
- external adapter + normalized gateway bridge
- MCP-backed channel connector

## 3. Kakao 도입 시 필요한 최소 요구

### 3.1 inbound 요구

- webhook endpoint 수신
- 요청 검증
- Kakao payload 정규화
- user utterance 추출
- channel session key 생성

### 3.2 outbound 요구

- simple text 응답
- card/quick reply 대응 가능성 검토
- 채널 제약에 맞는 길이 제한
- approval 상태와 action 결과를 Kakao-friendly하게 요약

### 3.3 session 요구

- Kakao user key 또는 동등 식별자 저장
- single-user continuity와 연결 가능한 metadata 정의
- channel-specific session과 assistant-global continuity의 경계 정의

## 4. Kakao가 다른 채널보다 까다로운 이유

### 4.1 응답 format 제약

WebUI나 Telegram보다 structured response 제약이 강하다. 따라서 assistant의 일반 응답과 action status를 그대로 보내기 어렵다.

### 4.2 approval interaction 제약

Kakao에서 approval을 처리할 때는 아래를 별도로 고민해야 한다.

- quick reply만으로 충분한가
- WebUI로 넘겨서 승인하게 할 것인가
- Kakao 안에서 approve/reject UX를 끝낼 수 있는가

### 4.3 session continuity 설계 필요

방향 A는 단일 사용자 continuity를 지향하지만, Kakao는 외부 채널 특성상 assistant-global context와 channel-local session을 어떻게 연결할지 더 명시적으로 설계해야 한다.

## 5. 권장 구조

방향 A 기준 Kakao 권장 구조는 아래다.

### 5.1 1차 권장안

- Kakao adapter가 inbound payload를 수신
- Nanobot 내부 공통 message shape로 normalize
- gateway/session manager로 전달
- outbound는 Kakao formatter가 변환

장점:

- Kakao 특수성 격리
- core runtime 오염 최소화
- 다른 채널과 공통 shape 유지 가능

### 5.2 2차 권장안

- Kakao 연동부를 아예 외부 service로 두고 Nanobot에는 bridge만 둠

장점:

- 배포와 인증을 분리 가능
- channel 정책 변경 대응이 상대적으로 쉬움

단점:

- 운영 구성요소가 하나 늘어난다.

## 6. Direction A와의 정합성 판단

Kakao integration이 방향 A와 맞으려면 아래 조건을 만족해야 한다.

1. Kakao가 없어도 Nanobot core는 완전히 동작해야 한다.
2. Kakao-specific response format은 adapter에서 처리해야 한다.
3. approval / automation / memory 정책이 Kakao 때문에 전역적으로 바뀌면 안 된다.
4. single-user continuity는 보강하되 multi-user platform 구조로 커지면 안 된다.

## 7. 선행 문서/설계 항목

Kakao 구현 전에 아래 문서가 먼저 있으면 좋다.

- single-user operating policy
- channel continuity / identity mapping 메모
- WebUI approval visibility 설계
- assistant automation adapter 원칙 문서

## 8. 구현 전 결정 포인트

구현 전에 아래를 먼저 결정해야 한다.

1. Kakao를 channel plugin으로 넣을지, 외부 adapter로 둘지
2. approval을 Kakao 안에서 처리할지 WebUI로 넘길지
3. Kakao 세션을 assistant-global continuity에 어느 수준까지 연결할지
4. rich response를 어느 수준까지 지원할지
5. 초기 scope를 simple text 중심으로 제한할지

## 9. 권장 초기 범위

초기 Kakao 도입 범위는 아래처럼 좁게 잡는 것이 맞다.

- inbound text 수신
- outbound simple text 응답
- session key 매핑
- 최소 approval 안내 문구
- WebUI에서 continuity 보조 확인 가능

초기 범위에서 제외할 것:

- 복잡한 카드 UX
- 광범위한 rich interaction
- long-running workflow orchestration
- Kakao 전용 approval system

## 10. 최종 제안

방향 A에서 Kakao는 중요할 수 있지만, Nanobot의 중심 구조가 되어서는 안 된다.

따라서 아래 순서를 권장한다.

1. single-user continuity 정책부터 정리한다.
2. approval/action visibility를 WebUI에서 먼저 정리한다.
3. Kakao는 adapter/plugin 방식으로 얇게 붙인다.
4. 초기에는 simple text 중심으로 시작한다.
5. richer interaction은 core 안정화 이후 단계적으로 검토한다.

이 방식이면 Kakao 요구를 수용하면서도, Nanobot 방향 A의 핵심인 local-first single-user assistant 구조를 유지할 수 있다.