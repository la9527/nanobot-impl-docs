# Nanobot Assistant Direction A Docs

상태:

- 이 디렉터리는 2026-05-01 기준 `historical design / reference archive` 로 유지한다.
- 현재 active implementation source of truth 는 `docs/todo.md` section 5 와 `docs/execution-backlog/*.md` 다.
- 따라서 이 디렉터리 문서는 진행 체크리스트보다 배경 설계와 흡수 전 의도를 설명하는 참고 문서로 읽는다.

이 디렉터리는 방향 A, 즉 `Nanobot를 유지하고 assistant 기능을 선택 흡수`하는 초기 작업을 읽기 순서대로 정리한 문서 모음이다.

권장 읽기 순서:

1. `00-architecture-fit-review.md`
2. `01-implementation-plan.md`
3. `02-single-user-operating-policy.md`
4. `03-webui-assistant-status-and-approval-visibility.md`
5. `04-channel-continuity-and-identity-mapping.md`
6. `05-assistant-automation-architecture.md`
7. `06-gmail-pilot-domain-design.md`
8. `07-kakao-integration-feasibility.md`
9. `08-shared-action-result-and-failure-shape.md`

문서 역할 요약:

- `00`: Nanobot 수용성 검토와 방향 A 선택 배경
- `01`: 전체 구현 진행 방안과 우선순위
- `02`: 단일 사용자 운영 원칙
- `03`: WebUI 상태/승인 가시성 설계
- `04`: channel continuity와 identity mapping 설계
- `05`: assistant automation 아키텍처
- `06`: Gmail pilot domain 상세 설계
- `07`: Kakao optional integration 검토
- `08`: Gmail/Kakao 공통 action result / failure / visibility shape

현재 읽기 가이드:

- 구현 진행 상태를 확인할 때는 먼저 `docs/todo.md` section 5 와 `docs/execution-backlog/README.md` 를 본다.
- 이 디렉터리 문서는 `왜 이 순서와 경계가 나왔는지` 를 이해할 때만 참고한다.
- `workplans/05-1` ~ `05-5` 는 일부 완료, 일부 보류 상태가 이미 반영된 archive 메모다.