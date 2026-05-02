# 05-1 WebUI Assistant Status Phase 1 Plan

## 상태 메모

- 상태: 완료, active backlog 에서 흡수됨
- 현재 source of truth: `docs/todo.md` section 5.1, `docs/execution-backlog/01-owner-aware-control-surface-phase1.md`
- 이 문서는 초기 phase-1 계획과 완료 전 설계 의도를 보존하는 archive reference 다.

## 1. 문서 목적

이 문서는 방향 A 구현 백로그의 `5.1 WebUI assistant status / approval visibility`를 1차 구현 단위로 쪼갠 작업 진행 방안 문서다.

이번 1차 목표는 큰 UI 재설계가 아니라, 사용자가 아래 질문에 즉시 답할 수 있도록 최소 상태 표면을 추가하는 것이다.

- 지금 어떤 target으로 대화 중인가?
- 현재 session은 어떤 채널과 연결돼 있는가?
- assistant가 지금 승인 대기 상태인가?
- assistant action이 돌아가고 있는가?

## 2. 이번 1차 범위

1차 범위에 포함할 것:

- thread header용 compact status badge
- waiting-approval 상태를 위한 inline status block 초안
- 기존 Settings를 깨지 않는 선에서 assistant 관련 설명 정리 포인트 정의
- WebUI 테스트와 build 검증 경로 마련

1차 범위에서 제외할 것:

- full side panel 또는 drawer
- recent actions timeline
- multi-channel continuity summary panel
- rich action result card
- approval inbox 같은 독립 뷰

## 3. 구현 목표

이번 단계에서 WebUI가 최소한 보여줘야 하는 것은 아래 네 가지다.

### 3.1 header status badge

최소 항목:

- active target
- channel kind
- approval pending 여부
- automation active 여부

원칙:

- 짧고 읽기 쉬워야 한다.
- 기존 thread header를 과도하게 복잡하게 만들지 않는다.
- 텍스트만으로도 의미가 전달돼야 한다.

### 3.2 waiting-approval inline block

최소 항목:

- action 제목
- 짧은 요약
- 왜 대기 중인지
- 사용자에게 다음에 무엇을 해야 하는지

원칙:

- 일반 assistant 답변 bubble과 시각적으로 구분된다.
- 시스템 로그처럼 길게 노출하지 않는다.
- 이후 richer approval card로 확장 가능한 구조여야 한다.

### 3.3 Settings assistant-related copy 정리

이번 단계에서는 Settings를 크게 뜯지 않는다. 다만 아래를 정리한다.

- target / provider / model 관련 라벨 정리
- locked state 설명 방식 정리
- assistant-related wording 정리

즉, 구현보다 문구/정보 구조 정리 수준에 가깝다.

### 3.4 continuity placeholder 문구

이번 1차에서는 real continuity metadata를 아직 넣지 않아도 된다. 다만 향후 5-2와 연결되도록 아래 수준의 placeholder를 고려한다.

- current channel label
- external session 여부 표시 가능성

## 4. 작업 분해

### Step 1. 현재 WebUI 상태 표면 파악

해야 할 일:

- thread header가 현재 무엇을 보여주는지 확인
- active target / model 관련 기존 표시 경로 확인
- settings에서 target/provider/model 관련 UI 확인
- approval 상태를 보여줄 기존 표면이 있는지 확인

완료 기준:

- 어떤 컴포넌트를 직접 수정해야 하는지 식별됨

### Step 2. status badge 데이터 모델 최소안 정의

필요 데이터:

- active target
- current channel kind
- approval pending boolean
- automation active boolean

결정 포인트:

- bootstrap payload에서 충분한지
- session payload 확장이 필요한지
- frontend derived state로 가능한지

완료 기준:

- header badge에 필요한 최소 데이터 소스가 결정됨

### Step 3. header UI 구현

해야 할 일:

- compact badge visual 정의
- active target badge 구현
- channel badge 구현
- approval pending / automation active badge 자리 확보

완료 기준:

- thread header에서 상태를 짧게 확인 가능

### Step 4. waiting-approval inline block 정의

해야 할 일:

- approval 상태 메시지 표현 방식 정의
- 기존 message list 안에 inline block 삽입 가능한 구조 확인
- 최소 문구 포맷 정의

예시:

- `승인 대기: 메일 발송`
- `승인 대기: 일정 생성`
- `외부 변경 작업이라 확인이 필요합니다.`

완료 기준:

- approval 상태를 시스템적이되 과하지 않게 thread 안에서 표현 가능

### Step 5. Settings copy 정리

해야 할 일:

- assistant 관련 라벨과 설명 재검토
- locked field 설명 방식 통일
- footer mode 설명이 assistant 상태 문맥과 어긋나지 않게 조정

완료 기준:

- 상태 표면과 Settings 문구가 같은 개념 체계를 사용함

### Step 6. 테스트 및 빌드 검증

해야 할 일:

- touched slice frontend 테스트 추가/수정
- 기존 layout/settings 테스트 영향 확인
- `npm run build` 검증

완료 기준:

- 1차 UI 변화가 테스트와 build 기준으로 안정적임

## 5. 권장 구현 순서

1. 현재 상태 표면 파악
2. status badge 데이터 모델 최소안 정의
3. header UI 구현
4. waiting-approval inline block 구현
5. Settings copy 정리
6. 테스트 + build

## 6. 구현 시 주의점

- 1차 단계에서는 side panel까지 욕심내지 않는다.
- 새 상태는 thread 흐름을 방해하지 않아야 한다.
- optional integration이 없어도 보이는 UI여야 한다.
- approval UX는 5-2 continuity 작업과 자연스럽게 이어져야 한다.

## 7. 완료 기준

이번 1차 문서 기준 완료는 아래를 뜻한다.

1. thread header에서 target / channel / approval 상태를 짧게 볼 수 있다.
2. approval 대기 상태를 thread 안에서 이해할 수 있다.
3. Settings 문구가 assistant 상태 개념과 어긋나지 않는다.
4. frontend 테스트와 build가 통과한다.

## 8. 다음 단계 연결

이 작업이 끝나면 다음 단계는 자연스럽게 아래로 이어진다.

- `05-2 single-user continuity metadata` 최소 구조 반영
- header badge에 linked session / external continuity 힌트 추가
- approval visibility를 cross-session summary와 연결