# 02 Owner Memory And Task Backbone Phase 1

## 1. 문서 목적

이 문서는 execution backlog 의 2순위 작업인 `owner memory + personal task backbone`을 phase-1 범위로 정리한 실행 문서다.

이번 단계의 목표는 conceptual 문서로 정리된 owner profile, memory taxonomy, personal task state model을 runtime/product layer에서 읽을 수 있는 최소 backbone 으로 내리는 것이다.

## 2. 왜 2순위인가

Gmail/Calendar 같은 domain action을 붙이려면 먼저 아래가 안정되어야 한다.

- 어떤 정보가 owner profile 인가
- 어떤 정보가 long-term memory 인가
- 어떤 정보가 active task 인가
- 어떤 상태를 WebUI aggregate 에서 보여줄 것인가

이 backbone 이 없으면 domain action은 붙더라도 금방 session-local 임시 상태로 흩어진다.

## 3. phase-1 범위

포함할 것:

- owner profile 핵심 필드 최소안 확정
- memory taxonomy 와 저장 경계 확정
- active task summary metadata 최소안 정의
- assistant-global aggregate 읽기 경로 정리
- user-facing 수정/정정 vocabulary 초안 정리

제외할 것:

- full owner database
- 별도 중앙 task database
- Dream 전체 재설계
- full CRUD memory UI

## 4. 핵심 구현 대상

### 4.1 owner profile minimal fields

- timezone
- preferred language
- response tone/length preference
- proactive preference 일부
- tool/policy defaults 일부

### 4.2 active task backbone

- active task summary shape
- waiting-approval / blocked / scheduled / recent-completed 분류
- owner-aware aggregate 에 필요한 최소 필드

### 4.3 memory boundary

- `USER.md` 로 갈 것
- `memory/MEMORY.md` 로 갈 것
- session/task metadata 로 남길 것
- raw history 에만 남길 것

## 5. 작업 분해

### Step 1. owner profile field phase-1 최소안 고정

- immediate UX 가치가 큰 필드만 우선 선정
- Gmail/Calendar/proactive가 재사용할 필드 우선

### Step 2. active task summary metadata 초안

- task id
- title
- status
- next step hint
- action refs
- updated_at

### Step 3. aggregate read path 정의

- session metadata 기반으로 충분한지 확인
- owner-aware aggregate 를 별도 store 없이 만들 수 있는지 우선 검토

### Step 4. memory correction vocabulary 정리

- 기억해
- 잊어
- 이건 기본 선호가 아님
- 이 프로젝트는 끝났어
- phase-1 metadata contract 에서는 각 phrase 를 `remember`, `forget`, `not-default`, `project-complete` action code 로 노출한다.
- `remember` / `forget` 은 `USER.md` 또는 `memory/MEMORY.md`, `project-complete` 는 `memory/MEMORY.md` 또는 `session.metadata` 를 우선 target 으로 본다.

### Step 5. focused test/validation 항목 정리

- metadata serialization
- API route exposure
- WebUI aggregate consumption

## 6. 완료 기준

1. owner profile, memory, task state 의 저장 경계가 phase-1 수준에서 닫힌다.
2. Gmail/Calendar/proactive가 읽을 최소 owner/task backbone 이 정의된다.
3. 별도 DB 없이도 owner-aware aggregate 를 확장할 수 있는 최소 metadata 경로가 정리된다.

## 7. 다음 단계 연결

- [03-gmail-readonly-draft-phase1.md](./03-gmail-readonly-draft-phase1.md)
- [06-proactive-briefing-and-quiet-hours-phase1.md](./06-proactive-briefing-and-quiet-hours-phase1.md)