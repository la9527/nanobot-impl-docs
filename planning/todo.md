# Nanobot TODO

이 문서는 현재 Nanobot 에서 `지금 해야 할 일`만 빠르게 보기 위한 top-level queue 다.

상세 구현 범위는 `docs/planning/execution-backlog/*.md`, 날짜형 운영 기록은 `docs/operations/status-summary/*.md`, 배경 설계는 `docs/archive/assistant-direction-a/` 와 `docs/planning/concepts/` 를 본다.

## 문서 운영 원칙

- 이 문서에는 active item, next candidate, deferred item 만 남긴다.
- 구현이 끝난 slice 의 긴 체크리스트는 여기에서 제거하고, 관련 backlog 문서와 status summary 위치만 남긴다.
- 새 제품 작업을 열 때는 먼저 여기에서 우선순위를 정한 뒤, 필요하면 `planning/execution-backlog/07-*` 이후 번호로 새 workplan 을 만든다.

## 1. Active Now

### P0. 실제 채널 validation

- [ ] `/model smart-router` 를 실제 inbound 채널 세션에서 한 번 더 end-to-end 검증
- [ ] live Telegram inbound 한 턴으로 WebUI websocket push mirror 재검증

### P1. live UI follow-up

- [ ] WebUI 일정 action result strip 이 history 하단에 고정처럼 남는지 다시 점검
- [ ] 기본 표시를 `한 줄 compact summary + 세부 정보 disclosure` 패턴으로 유지하는지 live 기준 확인

## 2. Next Candidates

- owner-aware aggregate 가 `/api/sessions` payload 만으로 계속 충분한지 다시 확인하고, 부족하면 새 backlog slice 로 route serializer 확장 검토
- Gmail/Calendar/proactive 후속 phase 는 새 요구가 생길 때 `planning/execution-backlog/07-*` 이후 문서로 다시 승격
- Kakao 는 현재 reference/deferred 상태로 유지하고, 운영 ingress 와 권한 확인 결과만 참고 문서로 유지

## 3. Completed And Moved Out

아래 항목은 구현 자체는 완료됐고, 이 문서에서는 상세 체크리스트를 제거했다.

- `모델 선택 /model`, `응답 footer /status /usage`, Telegram WebUI websocket mirror 기반 작업
- owner-aware control surface, owner memory/task backbone, Gmail, Calendar, proactive phase-1
- status-summary 문서 구조 이동과 관련 운영 정리

상세 범위와 근거 문서는 아래를 본다.

- [execution-backlog/README.md](./execution-backlog/README.md)
- [operations/status-summary/README.md](../operations/status-summary/README.md)
- [guide/README.md](../guide/README.md)

## 4. Directory Roles

- `docs/README.md`: 전체 문서 맵과 읽기 순서
- `planning/execution-backlog/`: 구현 slice 정의와 완료 기준
- `operations/status-summary/`: 날짜형 운영 메모
- `guide/`: 현재 live 사용/운영 기준
- `archive/assistant-direction-a/`: historical design archive
- `planning/concepts/`: conceptual reference
- `archive/observable-reasoning-plan/`, `archive/webui-ux-redesign/`, `archive/calendar-webui-reliability-review/`, `archive/upstream-main-sync-2026-05-08/`: 특정 주제 review/archive

## 5. 현재 한 줄 판단

- 구현 backlog 의 큰 덩어리는 1차로 정리됐고, 지금 남은 핵심은 `실채널 validation` 과 `live UI 확인` 이다.
- 새 기능을 더 벌리기보다, 이미 구현된 slice 를 운영 기준으로 다시 확인하고 필요한 경우에만 다음 backlog 문서를 여는 상태로 본다.