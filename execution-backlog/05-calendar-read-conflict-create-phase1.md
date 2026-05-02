# 05 Calendar Read Conflict Create Phase 1

## 1. 문서 목적

이 문서는 execution backlog 의 4순위 작업인 `Calendar read/conflict/create`를 phase-1 범위로 정리한 실행 문서다.

이번 단계의 목표는 calendar domain 을 `조회 -> 충돌 탐지 -> 생성`까지의 최소 assistant workflow 로 여는 것이다.

## 2. 왜 Gmail 다음인가

Calendar는 개인비서 핵심 domain 이지만, 메일보다 입력 부족과 conflict resolution 이 더 자주 발생한다. 따라서 mail pilot 에서 contract / approval / visibility vocabulary 를 먼저 검증한 뒤 적용하는 편이 낫다.

## 3. phase-1 범위

포함할 것:

- `calendar.list_events`
- `calendar.find_conflicts`
- `calendar.create_event`
- waiting-input / waiting-approval state 연결
- conflict summary normalization

제외할 것:

- update/cancel
- participant response sync
- full calendar management UI

## 4. 핵심 작업

### Step 1. calendar task contract 최소안 확정

- read/conflict/create input-output 정리

### Step 2. waiting-input 경로 정의

- 제목/시간/참석자 부족 시 어떤 질문을 할지 정리

### Step 3. create approval 정책 정의

- 기본 required 인지 review-visible 예외를 둘지 정리

### Step 4. conflict summary 와 WebUI status 연결

- free slot suggestion
- conflict summary
- create ready / approval pending 표시

### Step 5. focused validation 계획

- contract tests
- conflict normalization tests
- waiting-input / approval visibility tests

## 5. 완료 기준

1. 사용자가 일정 조회, 충돌 확인, 일정 생성 준비를 assistant action 으로 사용할 수 있다.
2. 입력 부족과 충돌이 사용자 언어로 정규화된다.
3. update/cancel 단계로 확장 가능한 contract 가 준비된다.

## 6. 다음 단계 연결

- [06-proactive-briefing-and-quiet-hours-phase1.md](./06-proactive-briefing-and-quiet-hours-phase1.md)