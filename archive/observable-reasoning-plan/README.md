# Nanobot Observable Reasoning Plan

상태:

- 현재는 targeted feature design/reference 디렉터리로 유지한다.
- 새 active backlog 항목은 이 디렉터리에서 직접 관리하지 않고 `docs/planning/todo.md` 또는 별도 execution backlog 문서로 승격해서 연다.

## 목적

이 디렉토리는 Nanobot 에서 observable reasoning 을 사용자 경험으로 안전하게 노출하기 위한 설계 문서를 모아두는 공간이다.

이번 주제는 단순히 모델의 "생각"을 더 많이 보여주는 문제가 아니라, 아래 세 가지를 함께 정리하는 작업이다.

- WebUI 에서 reasoning 을 어느 수준까지 보여줄지
- Telegram 에서 기존 채널을 깨지 않고 더 나은 상호작용 화면을 어떻게 추가할지
- runtime, session, channel, WebUI 가 같은 reasoning contract 를 공유하도록 어떻게 정리할지

## 현재 핵심 결정

- raw chain-of-thought 전체를 그대로 노출하는 방향은 기본값으로 채택하지 않는다.
- 사용자에게는 observable reasoning 을 `status`, `summary`, `debug trace` 중심으로 노출한다.
- 선호 UI 패턴은 VS Code Copilot 류의 neutral gray reasoning card 이며, 실행 중에는 trace 가 펼쳐지고 완료 후에는 summary 가 접힌 상태로 남는 progressive disclosure 를 기본 구조로 삼는다.
- 기존 Telegram 채널은 유지하고, richer UX 는 신규 `telegram_v2` 채널로 분리한다.
- `telegram_v2` 는 Telegram Mini App 중심으로 설계하고, chat message / inline keyboard 는 보조 surface 로 둔다.
- WebUI 는 `reasoning_visibility = off | status_only | summary | debug_trace` 4단계 정책을 사용한다.

## 문서 구성

- [telegram-v2-and-reasoning-visibility-plan-2026-05-10.md](./telegram-v2-and-reasoning-visibility-plan-2026-05-10.md): observable reasoning, `telegram_v2`, WebUI `reasoning_visibility` 의 통합 설계안
- [implementation-checklist-2026-05-10.md](./implementation-checklist-2026-05-10.md): 현재 구현 완료 / 미완료 항목을 plan 기준으로 추적하는 체크리스트
