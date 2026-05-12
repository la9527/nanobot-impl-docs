# Nanobot Conceptual Drafts

상태:

- 이 디렉터리는 active checklist 가 아니라 conceptual reference 묶음이다.
- 현재 구현 우선순위와 validation 은 [../todo.md](../todo.md) 와 [../execution-backlog/README.md](../execution-backlog/README.md) 를 기준으로 본다.

## 목적

이 디렉터리는 [./overview.md](./overview.md) 에서 한 번에 다루기 어려운 주제를 도메인별 상세 초안으로 분리한 공간이다.

현재 문서들은 아래 역할로 유지한다.

- `owner-profile-and-memory-taxonomy.md`: owner profile, memory boundary, correction vocabulary 같은 개인 메모리 축 개념 정리
- `personal-task-state-model.md`: personal task 상태 언어와 runtime task 의 구분
- `gmail-calendar-domain-contracts.md`: mail/calendar 도메인 contract 초안과 action vocabulary 기준
- `proactive-policy-and-quiet-hours.md`: proactive, quiet hours, delivery policy 기준

## 운영 원칙

- 이 디렉터리 문서는 구현 완료 체크리스트로 사용하지 않는다.
- 구현이 시작되면 필요한 범위만 `../execution-backlog/*.md` 또는 `../../operations/status-summary/*.md` 로 내려 적는다.
- 개념 설명이 낡았더라도 삭제보다 `현재 source of truth 가 어디인지` 를 링크로 분명히 남기는 쪽을 우선한다.