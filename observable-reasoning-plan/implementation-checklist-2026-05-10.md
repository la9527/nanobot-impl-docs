# Nanobot Observable Reasoning 구현 체크리스트 - 2026-05-10

## 문서 목적

이 문서는 `telegram-v2-and-reasoning-visibility-plan-2026-05-10.md` 기준으로 현재까지 실제로 반영된 항목과 아직 남아 있는 항목을 구분해서 추적하기 위한 구현 체크리스트다.

## 1. 문서와 방향 정리

- [x] `docs/observable-reasoning-plan/README.md` 를 plan entry 로 정리
- [x] `telegram_v2`, WebUI reasoning visibility, safe observable reasoning 원칙을 한 문서로 정리
- [x] raw chain-of-thought 기본 비노출 원칙 명시
- [x] 진행 상태를 체크 가능한 별도 checklist 문서 추가

## 2. WebUI reasoning visibility surface

- [x] `reasoning_visibility = off | status_only | summary | debug_trace` 타입과 기본 UI surface 연결
- [x] gray tone reasoning/status surface 유지
- [x] live `tool_hint` / `progress` row 가 assistant 답변 앞에 오도록 정렬
- [x] trace 가 없는 세션에 synthetic reasoning row 를 history 에 주입
- [x] `생각`, `생각 중` 라벨로 one-line reasoning row 표시
- [x] 새로고침 후에도 same turn reasoning row 가 history 에 남도록 persistence 연결
- [x] live browser 에서 `prompt -> 생각 -> 답변` 순서 확인
- [ ] 완료 후 collapsed summary disclosure 를 plan 문서와 동일한 수준으로 정교화
- [ ] `debug_trace` reopen inspector 를 별도 panel 수준으로 정리

## 3. Runtime / session reasoning contract

- [x] final assistant turn 저장 시 UI-safe `visible_reasoning` field persisted
- [x] raw `reasoning_content` 를 그대로 replay 하지 않고 safe one-line field 만 사용
- [x] WebUI session hydration 에서 `visible_reasoning` 을 assistant 앞 trace row 로 재구성
- [x] localized visible reasoning string 적용
- [x] live websocket turn 으로 session history JSON 에 `visible_reasoning` 저장 확인
- [ ] checkpoint / recovery 경로가 같은 field 를 명시적으로 재사용하도록 정리
- [ ] 예전 session history 에 대한 server-side backfill 정책 정의

## 4. Action result / supplemental UI 정리

- [x] `ThreadInlineActionResult` 기본 표시를 한 줄 compact row 로 축소
- [x] preview / conflict / thread detail 은 기본 숨김 유지
- [x] trailing `세부 정보` 버튼으로 detail toggle 유지
- [x] summary row 자체를 클릭해도 detail toggle 가능하게 연결
- [x] `일정 결과가 고정처럼 남는 구조` 점검 항목을 `docs/todo.md` 에 기록
- [x] live 브라우저에서 `일정 결과` row 클릭 시 detail expand 확인
- [ ] detail surface 를 side sheet / inspector 형태로 재구성

## 5. Telegram / `telegram_v2`

- [x] 기존 `telegram` 채널은 유지하고 richer UX 는 `telegram_v2` 로 분리한다는 방향 고정
- [x] Telegram Mini App 중심 설계 방향 문서화
- [ ] `telegram_v2` channel/runtime skeleton 구현
- [ ] Mini App thread / reasoning panel / action log 첫 runnable slice 구현
- [ ] Telegram 기본 채널과 Mini App deep-link 연결 정책 구현

## 6. 검증

- [x] backend focused test: `tests/agent/test_loop_save_turn.py`
- [x] frontend focused test: `src/tests/useSessions.test.tsx`
- [x] frontend focused test: `src/tests/thread-inline-action-result.test.tsx`
- [x] WebUI production build 통과
- [x] launchd restart 후 live gateway health 복구 확인
- [x] authenticated bootstrap + websocket turn + session history JSON 재조회 검증
- [x] live browser 에서 action result row click 확인
- [ ] broader `src/tests/thread-shell.test.tsx` 전체 파일 green 정리

## 7. 현재 판단

- [x] WebUI observable reasoning 의 핵심 사용자 요구였던 `생각 -> 답변` 순서와 refresh persistence 는 현재 구현 범위 안에서 달성
- [x] action result compact row 와 details-on-demand 는 현재 live UI 에 반영
- [ ] plan 문서의 모든 progressive disclosure / `debug_trace` inspector / `telegram_v2` 실행 항목이 완료된 상태는 아님

## 8. 다음 우선순위 제안

- [ ] `thread-shell` broader regression 정리
- [ ] action result detail surface 를 inspector 패턴으로 이동할지 결정
- [ ] `telegram_v2` 의 최소 runnable slice 를 정의하고 첫 구현 시작