# 06 Proactive Briefing And Quiet Hours Phase 1

## 1. 문서 목적

이 문서는 execution backlog 의 5순위 작업인 `proactive briefing + quiet hours`를 phase-1 범위로 정리한 실행 문서다.

이번 단계의 목표는 이미 있는 heartbeat / periodic task 기반을 개인비서형 proactive surface 로 제한적으로 확장하되, quiet hours 와 opt-in 정책으로 annoyance risk 를 통제하는 것이다.

## 2. 왜 마지막인가

proactive는 체감 가치가 크지만, owner profile, task state, Gmail/Calendar action summary 가 먼저 있어야 유용한 메시지를 만들 수 있다.

즉, proactive는 단독 기능이 아니라 앞선 단계들이 만든 상태와 summary 를 소비하는 상위 behavior layer다.

## 3. phase-1 범위

포함할 것:

- morning briefing 최소안
- pending approval / blocked digest 최소안
- quiet hours 기본 모델
- severity / frequency 제한 규칙
- WebUI-first proactive 원칙

제외할 것:

- aggressive push automation
- auto-send / auto-create 같은 hidden side effect
- channel별 고급 personalization

## 4. 핵심 작업

### Step 1. proactive category 최소안 확정

- briefing
- reminder
- follow-up digest

### Step 2. quiet hours config / policy 최소안 정리

- enabled
- start/end local time
- timezone
- allow critical 여부

### Step 3. summary source 결정

- active tasks
- waiting-approval tasks
- blocked tasks
- today schedule / inbox summary

### Step 4. delivery policy 정리

- WebUI-first when push should stay in-product
- recent external active channel fallback outside quiet hours
- quiet hours 중 push suppression

### Step 5. focused validation 계획

- heartbeat consumer contract
- deliverability suppression
- digest summary formatting 검증

## 5. 완료 기준

1. morning briefing 과 최소 digest 가 owner/task/action summary 기반으로 정의된다.
2. quiet hours 와 anti-spam 원칙이 runtime 정책으로 연결될 준비가 된다.
3. proactive가 trust-breaking behavior 가 아니라 controlled assistant surface 로 해석된다.

## 6. 참고 연결

- `docs/planning/concepts/proactive-policy-and-quiet-hours.md`