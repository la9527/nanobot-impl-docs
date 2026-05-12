# Nanobot WebUI UX Redesign

> 용어 메모: 이 디렉터리의 검토 문서는 당시 표현을 그대로 보존한다. 현재 WebUI 용어 기준은 [docs/guide/10-webui-terminology.md](../../guide/10-webui-terminology.md) 를 따른다.

상태:

- 이 디렉터리는 WebUI 정보 구조 재설계 검토 기록을 보관하는 reference archive 다.
- 현재 active 구현 기준은 `docs/guide/`, `docs/planning/todo.md`, 필요한 경우 새 execution backlog 문서로 이어서 본다.

## 목적

이 디렉토리는 Nanobot WebUI 의 thread 화면에서 owner-aware summary, task summary, action result preview, memory correction quick action 이 한 화면에 과도하게 누적되는 문제를 재정리하고, 선택된 개선 방향과 dashboard 정보 구조를 구체화한 문서를 모아두는 공간이다.

현재 기준 핵심 판단은 아래와 같다.

- thread 화면은 어디까지나 대화가 주인공이어야 한다.
- cross-thread 성격의 owner aggregate 는 thread 상단 고정 카드보다 dashboard/home 성격의 surface 가 더 적합하다.
- thread 안에서는 항상 보이는 정보와 필요할 때만 꺼내 보는 정보를 분리해야 한다.
- memory correction 과 action preview 는 기능적으로 중요하지만, 항상 펼쳐진 카드로 두면 채팅 몰입을 해친다.
- action result 는 항상 펼쳐진 큰 결과 패널이 아니라 compact summary + 필요 시 detail disclosure 패턴이어야 한다.
- action result 가 이미 상태를 설명할 때는 같은 의미의 failed/completed status block 을 바로 아래 중복 노출하지 않는다.

## 문서 구성

- [thread-surface-redesign-2026-05-01.md](./thread-surface-redesign-2026-05-01.md): 현재 화면 문제 진단, 대안 비교, 선택된 진행 방향, 단계별 작업안
- [dashboard-information-architecture-2026-05-01.md](./dashboard-information-architecture-2026-05-01.md): Assistant dashboard/home 구조, 섹션 정의, 진입 흐름, 카드 우선순위, thread 와의 역할 분리안