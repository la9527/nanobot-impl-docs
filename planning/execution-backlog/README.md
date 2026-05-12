# Nanobot Personal Assistant Execution Backlog

> 용어 메모: 이 디렉터리의 초기 기획 문서에는 `assistant`, `Dashboard`, `Settings`, `thread` 같은 예전 WebUI 표현이 남아 있을 수 있다. 현재 사용자 노출 용어 기준은 [guide/10-webui-terminology.md](../../guide/10-webui-terminology.md) 를 따른다.

상태:

- 이 디렉터리는 `새 구현 slice 를 정의하는 workplan index` 로 유지한다.
- 현재 `01` 부터 `06` 까지의 phase-1 문서는 구현 완료 범위를 설명하는 reference 로 본다.
- 다음 active 제품 작업을 새로 열면 `07-*` 이후 번호로 추가하고, 먼저 [../todo.md](../todo.md) 에서 우선순위를 명시한다.

이 디렉터리는 [concepts/overview.md](../concepts/overview.md)와 `docs/planning/concepts/*.md` 에서 정리한 내용을 실제 구현 순서로 내리기 위한 실행 백로그 문서 모음이다.

목표는 두 가지다.

- 이미 문서화된 개인비서 제품 레이어를 `무엇부터 구현할지` 우선순위 기준으로 고정하기
- 각 우선순위별로 바로 구현 착수 가능한 phase-1 workplan 을 준비하기

## 1. 선정 원칙

이번 실행 백로그는 아래 원칙으로 대상을 선정한다.

1. 이미 완료된 기반 위에 바로 붙일 수 있는가
2. WebUI control surface, continuity, approval 같은 기존 완료 항목을 재사용하는가
3. 사용자 체감 가치가 빠르게 나오는가
4. 새로운 대규모 인프라보다 product layer 정리가 먼저 가능한가
5. Kakao 같은 후순위 채널보다 core assistant experience를 먼저 강화하는가

## 2. 현재 정리된 phase-1 구현 대상

현재 문서로 정리된 phase-1 구현 대상은 아래다.

1. owner-aware control surface aggregate
2. owner memory + personal task backbone
3. Gmail read-only + draft flow
4. Gmail send approval flow
5. Calendar read/conflict/create flow
6. proactive briefing + quiet hours phase-1

## 3. 작업 순서

아래 순서는 최초 implementation order 기록으로 보존한다. 현재는 `01`~`06` 이 모두 completed reference 이며, 새 작업을 다시 열 때는 이 순서를 그대로 active queue 로 간주하지 않는다.

### 3.1 1순위

- [01-owner-aware-control-surface-phase1.md](./01-owner-aware-control-surface-phase1.md)

이 단계는 이미 구현된 WebUI status / continuity / approval visibility를 `assistant control surface` 수준으로 끌어올리는 작업이다.

### 3.2 2순위

- [02-owner-memory-and-task-backbone-phase1.md](./02-owner-memory-and-task-backbone-phase1.md)

이 단계는 owner profile, task state model, continuity aggregate를 실제 runtime/product layer에서 읽을 수 있는 최소 backbone을 정리한다.

### 3.3 3순위

- [03-gmail-readonly-draft-phase1.md](./03-gmail-readonly-draft-phase1.md)
- [04-gmail-send-approval-phase1.md](./04-gmail-send-approval-phase1.md)

이 단계는 개인비서 체감 가치가 큰 mailbox workflow를 action contract와 approval 흐름으로 구현한다.

### 3.4 4순위

- [05-calendar-read-conflict-create-phase1.md](./05-calendar-read-conflict-create-phase1.md)

Calendar는 Gmail 다음으로 value가 크지만, 입력 부족과 충돌 해석이 섞여 있어 mail pilot 다음으로 두는 편이 안정적이다.

### 3.5 5순위

- [06-proactive-briefing-and-quiet-hours-phase1.md](./06-proactive-briefing-and-quiet-hours-phase1.md)

proactive는 체감 가치는 크지만 annoyance risk도 커서, owner/task/action 기반이 잡힌 뒤에 제한적으로 올리는 편이 맞다.

## 4. 이번 백로그에서 바로 후순위로 둔 항목

아래는 필요하지만 이번 first execution backlog 에서는 후순위로 둔다.

- Kakao adapter 본격 구현
- contacts / notes 전용 domain 확장
- full owner directory 또는 별도 중앙 task database
- rich timeline / inbox / dashboard 급 독립 WebUI 뷰

## 5. 공통 완료 기준

각 workplan 은 아래 공통 기준을 가진다.

- phase-1 범위와 비범위가 분명하다.
- runtime, WebUI, metadata, approval, test 관점의 작업이 같이 정리된다.
- live 운영 원칙과 충돌하지 않는다.
- 다음 단계 문서와 자연스럽게 연결된다.

## 6. 한 줄 결론

이 디렉터리는 `무엇을 어떻게 구현했는지` 를 설명하는 product-layer workplan 기록이며, 현재 active queue 자체는 [../todo.md](../todo.md) 를 기준으로 읽는다.