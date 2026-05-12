# Nanobot Observable Reasoning / Telegram v2 / WebUI Reasoning Visibility 설계안 - 2026-05-10

## 1. 문서 목적

이 문서는 Nanobot 에서 observable reasoning 을 사용자에게 어떻게 보여줄지에 대한 제품, UX, runtime 설계안을 정리한다.

이번 설계의 직접 목표는 아래 세 가지다.

1. WebUI 에 reasoning 노출 수준을 일관되게 제어할 수 있게 만든다.
2. 기존 Telegram 채널은 유지하면서, WebUI 에 가까운 richer experience 를 제공하는 `telegram_v2` 를 별도 채널로 추가한다.
3. reasoning 관련 runtime event, session 저장, channel rendering 이 서로 다른 기준으로 흩어지지 않게 하나의 contract 로 정리한다.

이 문서는 구현 전에 합의할 설계 기준 문서이며, 이후 코드 작업의 source of truth 로 사용한다.

현재 구현 진척도 체크는 아래 문서를 함께 본다.

- [implementation-checklist-2026-05-10.md](./implementation-checklist-2026-05-10.md)

## 2. 문제 정의

현재 Nanobot 는 내부적으로 progress, tool hint, approval, turn completion, reasoning-content 계열 정보를 이미 일부 보유하고 있지만, 사용자 경험은 채널별로 크게 다르다.

- WebUI 는 websocket 기반으로 progress 류 이벤트를 비교적 풍부하게 받을 수 있다.
- Telegram 은 typing indicator 와 message edit 중심이라 상태 전달 밀도가 낮고, WebUI 같은 조작 surface 가 없다.
- session replay 에는 reasoning 계열 데이터가 일부 남지만, 어떤 수준까지 다시 보여줄지 정책이 없다.
- raw chain-of-thought 를 그대로 드러내는 것은 privacy, prompt leakage, 과도한 인지부하 측면에서 기본 UX 로 적합하지 않다.

즉, 지금 필요한 것은 "더 많이 보여주기" 가 아니라, 사용자에게 실제로 도움이 되는 observable reasoning layer 를 설계하는 것이다.

## 3. 이번 설계의 핵심 판단

### 3.1 raw chain-of-thought 는 기본 노출 대상으로 삼지 않는다

이번 작업의 기본 원칙은 reasoning 원문 전체를 실시간으로 그대로 노출하지 않는 것이다.

대신 아래 계층으로 변환해 보여준다.

- status: 지금 무엇을 하고 있는지
- summary: 왜 이 답이나 작업 흐름으로 갔는지에 대한 짧은 설명
- debug trace: 고급 사용자가 필요할 때만 보는 구조화된 실행 흔적

이 방향을 택하는 이유는 아래와 같다.

- 채널 간 일관성을 유지하기 쉽다.
- Telegram 처럼 UI 제약이 큰 채널에서도 동일한 의미를 전달할 수 있다.
- 모델별 reasoning 포맷 차이를 그대로 사용자 UX 로 노출하지 않아도 된다.
- 내부 prompt, hidden context, intermediate speculation 을 기본 surface 에서 분리할 수 있다.

### 3.2 Telegram 개선은 기존 채널 수정이 아니라 `telegram_v2` 추가로 푼다

기존 Telegram 채널은 현재 운용 중인 동작을 유지한다.

그 이유는 아래와 같다.

- 이미 안정화된 현재 사용 흐름을 깨지 않는다.
- richer UI 실험이 실패해도 기존 채널을 rollback 없이 유지할 수 있다.
- 설정, 사용자 전환, 운영 검증을 단계적으로 진행할 수 있다.

따라서 이번 설계는 `telegram` 을 유지하고, 신규 `telegram_v2` 채널을 추가하는 방향을 기준으로 삼는다.

### 3.3 Telegram 에서 WebUI 같은 경험은 Mini App 중심으로 설계한다

공식 Telegram Bot API / Mini App 문서 기준으로 보면, chat-native message edit 와 inline keyboard 만으로는 WebUI 와 유사한 지속적 상태 패널, 상세 reasoning disclosure, 고정 action strip, 설정 토글 같은 경험을 충분히 만들기 어렵다.

따라서 `telegram_v2` 의 주 surface 는 아래처럼 분리한다.

- Telegram chat: 짧은 답변, 상태 요약, quick action, Mini App 진입점
- Telegram Mini App: thread view, reasoning panel, action log, visibility setting, task/detail disclosure

즉, Telegram v2 는 "채팅 안에서 모든 것을 해결"하는 채널이 아니라, "Telegram 안에 있는 경량 WebUI" 성격으로 설계한다.

## 4. 목표와 비목표

### 4.1 목표

- 채널별 제약과 무관하게 같은 reasoning contract 를 유지한다.
- 사용자가 reasoning 노출 수준을 직접 제어할 수 있게 한다.
- WebUI 와 Telegram v2 가 같은 상태 의미를 다른 surface 에 맞게 표현하게 한다.
- approval, automation, tool usage, fallback 같은 중요한 runtime 변화를 사람이 빠르게 이해할 수 있게 한다.
- 실행 중에는 정보가 보이고 완료 후에는 조용히 접히는 Copilot-style interaction rhythm 을 WebUI 와 Telegram v2 에 공통 적용한다.

### 4.2 비목표

- 모델 원본 thought token stream 을 있는 그대로 전 채널에 공개하지 않는다.
- 기존 Telegram 채널을 즉시 대체하지 않는다.
- 이번 1차 설계에서 Telegram v2 Mini App 을 풀기능 WebUI 복제본으로 만들지 않는다.
- reasoning UI 를 위해 session 저장 포맷 전체를 다시 설계하는 작업까지 한 번에 하지 않는다.

## 5. 사용자 경험 원칙

### 5.1 chat-first, reasoning-second

대화 자체가 중심이다. reasoning 은 답변을 가리는 큰 패널이 아니라, 필요할 때 펼쳐 볼 수 있는 보조 정보여야 한다.

### 5.2 의미 중심 노출

reasoning 은 "모델 내부 문장" 이 아니라 "사용자가 이해해야 하는 실행 의미" 로 노출한다.

예시:

- "일정 조회를 위해 calendar workflow 를 호출 중"
- "도구 응답이 늦어 fallback summary 로 전환함"
- "답변 전에 메일/캘린더 결과를 합치는 중"

### 5.3 모드별 일관성

`status_only`, `summary`, `debug_trace` 는 채널이 달라도 같은 의미를 가져야 한다. 단지 표현 밀도와 완료 후 남는 disclosure 수준만 달라져야 한다.

### 5.4 default-safe

기본값은 과노출보다 보수적으로 둔다. 추천 기본값은 아래와 같다.

- WebUI: `summary`
- Telegram: `status_only`
- Telegram v2 Mini App: `summary`

### 5.5 Copilot-style progressive disclosure

이번 설계는 사용자가 선호한 VS Code Copilot 의 task/reasoning surface 와 유사한 상호작용 리듬을 기준으로 삼는다.

핵심 패턴은 아래와 같다.

- 실행 중에는 reasoning card 가 펼쳐진 상태로 현재 진행 trace 를 보여준다.
- 작업이 끝나면 같은 card 가 자동으로 접히고, summary 만 보이는 compact 상태로 전환된다.
- 사용자는 필요할 때만 다시 펼쳐 자세한 trace 를 확인한다.
- 시각 톤은 강조색보다 neutral gray 계열을 기본으로 사용한다.

즉, reasoning UI 는 "계속 펼쳐진 로그창" 이 아니라, "진행 중에는 보이고 끝나면 조용히 접히는 작업 카드" 에 가깝게 설계한다.

### 5.6 Visual language

색과 밀도는 아래 원칙을 따른다.

- 기본 배경, 보더, 텍스트는 gray 톤 중심으로 구성한다.
- 완료 상태에서도 큰 성공색, 경고색, 브랜드색을 기본값으로 남발하지 않는다.
- running, done, blocked 차이는 색보다 layout, icon, label, text hierarchy 로 먼저 구분한다.
- trace line 은 채팅 본문보다 한 단계 낮은 시각 우선순위를 가져야 한다.
- summary collapsed state 는 화면을 방해하지 않는 compact disclosure 형태를 기본으로 한다.

## 6. 제안하는 reasoning contract

### 6.1 내부 개념 구분

runtime 에서 reasoning 관련 정보를 아래 4가지 계층으로 구분한다.

1. hidden reasoning
   - 모델 내부 thought, intermediate scratchpad, 원문 reasoning token
   - 사용자 기본 surface 에 직접 노출하지 않음

2. observable status
   - 현재 단계, 도구 호출 중 여부, 대기, 승인, 재시도, fallback 상태
   - 모든 채널에서 공통 사용

3. reasoning summary
   - 왜 이런 행동을 택했는지에 대한 짧은 human-readable 요약
   - WebUI 와 Telegram v2 중심으로 노출

4. debug trace
   - 구조화된 event timeline, tool step, route, fallback 정보
   - 고급 모드에서만 노출

### 6.2 제안 event 타입

기존 progress/tool hint 계열을 확장해 아래와 같은 공통 event 레이어를 정의한다.

- `reasoning_status`
  - 현재 단계 한 줄 요약
  - 예: `calendar lookup in progress`

- `reasoning_summary`
  - 특정 답변 구간 또는 최종 답변에 대한 짧은 설명
  - 예: `calendar 결과를 확인한 뒤 오늘 일정만 추려서 요약함`

- `reasoning_trace`
  - debug mode 에서만 쓰는 구조화된 trace item
  - 예: tool 시작, 종료, fallback, retry, provider switch, approval wait

- `reasoning_mode_hint`
  - 현재 채널 또는 세션에서 어떤 visibility 가 적용됐는지 전달

이 레이어는 websocket, Telegram v2 bridge, session replay 가 공통으로 사용할 수 있어야 한다.

## 7. `reasoning_visibility` 정책

### 7.1 모드 정의

이번 설계에서 mode 는 단순히 "어느 정보를 얼마나 보여주나" 뿐 아니라, 아래 두 가지를 함께 제어한다.

1. 실행 중 card 가 얼마나 자세히 펼쳐지는가
2. 완료 후 어떤 정보가 접힌 상태로 남는가

#### `off`

- reasoning 관련 추가 surface 를 보여주지 않는다.
- 일반 답변과 필요한 최소 오류, 승인 메시지만 노출한다.
- 사용자가 가장 단순한 채팅 경험을 원할 때 사용한다.

#### `status_only`

- 실행 중에는 한 줄 또는 두 줄 수준의 compact status card 만 보여준다.
- 단계명, 대기 상태, 승인 필요, 도구 실행 중, fallback 여부만 노출한다.
- 완료 후에는 접힌 상태의 짧은 status result 만 남긴다.
- summary 본문과 trace reopen 은 제공하지 않는다.
- Telegram 기본값으로 적합하다.

#### `summary`

- 실행 중에는 gray reasoning card 가 펼쳐져 현재 step 을 순차적으로 보여준다.
- live trace 는 coarse-grained step 중심으로 유지해 과도한 로그화를 피한다.
- 완료 후에는 같은 card 가 자동으로 접히고, 1~3문장 summary 만 보이는 disclosure 로 전환된다.
- 사용자가 펼치면 summary 와 마지막 주요 step 들을 다시 볼 수 있다.
- 일반 사용자용 권장 모드다.
- WebUI 기본값 후보로 둔다.

#### `debug_trace`

- 실행 중에는 gray reasoning card 가 펼쳐진 상태에서 더 자세한 step trace 를 실시간 표시한다.
- tool 호출, retry, provider fallback, approval wait, route change 같은 정보를 포함한다.
- 완료 후에는 기본 시야에서는 `summary` 와 같은 collapsed state 로 돌아간다.
- 다만 사용자가 다시 펼치면 full trace timeline 을 확인할 수 있다.
- 개발, 운영, 파워유저용 모드다.

### 7.2 채널별 표현 매핑

| visibility | WebUI | Telegram | Telegram v2 Mini App |
| --- | --- | --- | --- |
| `off` | reasoning card 없음 | 일반 답변만 전송 | reasoning card 숨김 |
| `status_only` | compact gray status card | 편집 가능한 상태 한 줄 또는 짧은 상태 카드 | compact gray status card |
| `summary` | 실행 중 expanded card, 완료 후 collapsed summary card | 실행 중 status bubble, 완료 후 collapsed summary message | 실행 중 expanded card, 완료 후 collapsed summary card |
| `debug_trace` | 실행 중 expanded detailed trace, 완료 후 collapsed summary + reopen trace | 실행 중 제한된 trace, 완료 후 summary + `Open details` 링크 | 실행 중 expanded detailed trace, 완료 후 collapsed summary + reopen trace |

Telegram 기본 채널에서는 `debug_trace` 를 직접 렌더링하기보다, 필요한 경우 WebUI 또는 Mini App deep link 로 넘기는 것을 원칙으로 한다.

즉, `summary` 와 `debug_trace` 의 가장 큰 차이는 완료 후 기본 외형이 아니라, 실행 중 trace 밀도와 완료 후 reopen 가능한 상세 수준에 있다.

## 8. WebUI 설계안

### 8.1 사용자 설정

WebUI 는 사용자 또는 세션 단위로 `reasoning_visibility` 를 설정할 수 있어야 한다.

우선순위는 아래처럼 둔다.

1. session override
2. user preference
3. channel default
4. system default

초기 구현에서는 user preference + session override 까지만 지원해도 충분하다.

### 8.2 UI 배치 원칙

- 답변 본문 위에 큰 reasoning 카드 묶음을 항상 고정하지 않는다.
- 현재 turn 과 강하게 연결된 단일 reasoning card 만 노출한다.
- 실행 중에는 card 가 펼쳐지고, 완료 후에는 자동으로 접힌 summary disclosure 로 바뀐다.
- summary 는 message stream 을 가리지 않는 compact disclosure 로 둔다.
- debug trace 는 collapsed state 에서 본문을 침범하지 않고, 필요 시만 펼쳐지는 inspector 성격을 유지한다.

### 8.3 카드 구조 제안

WebUI reasoning card 는 아래 구조를 기본형으로 삼는다.

1. header
   - 현재 작업 제목 또는 짧은 상태 라벨
   - 선택적으로 step count, running/done indicator

2. body
   - 실행 중일 때만 보이는 trace line 목록 또는 step description

3. collapsed summary
   - 완료 후 기본으로 보이는 한두 줄 summary

4. disclosure action
   - `자세히 보기`, `trace 보기`, `접기` 같은 토글

이 구조는 Copilot task card 와 유사하게, 실행 중 정보 밀도와 완료 후 compactness 를 동시에 만족시키는 형태를 목표로 한다.

### 8.4 모드별 UI 동작

#### `off`

- progress stripe, reasoning summary, trace panel 을 모두 숨긴다.
- approval, blocking 상태만 별도 시스템 상태로 유지한다.

#### `status_only`

- 현재 turn 상단 또는 메시지 사이에 compact gray status card 를 노출한다.
- 예: `Calendar 확인 중`, `메일 검색 중`, `승인 대기 중`
- 완료 후에는 `일정 확인 완료` 같은 짧은 결과 라벨만 남기고, 본문 reasoning summary 는 숨긴다.

#### `summary`

- 실행 중에는 expanded reasoning card 를 노출한다.
- card body 에는 현재 처리 단계와 최근 주요 step 을 gray 톤 목록으로 보여준다.
- 답변 완료 후에는 같은 card 가 자동으로 접히고, summary disclosure 만 남긴다.
- summary 는 1~3문장 내 human-readable 텍스트를 원칙으로 한다.

#### `debug_trace`

- 실행 중에는 `summary` 와 같은 card 구조를 쓰되, body 에 더 촘촘한 trace line 과 metadata 를 포함한다.
- 완료 후 기본 상태는 여전히 collapsed summary 이다.
- 사용자가 disclosure 를 열면 full trace timeline 또는 inspector panel 을 볼 수 있다.
- developer debugging 정보가 아니라 사용자에게 의미 있는 실행 흔적만 남긴다.

### 8.5 WebUI 구현 범위 제안

1차에서는 websocket event 확장과 thread surface 반영까지만 한다.

- event type 추가
- visibility setting state 추가
- thread message 근처 reasoning card UI 추가
- 실행 중 expanded, 완료 후 collapsed 전환 추가
- debug trace reopen panel 추가

dashboard 나 historical replay 완성도는 2차 범위로 둔다.

## 9. `telegram_v2` 설계안

### 9.1 채널 모델

`telegram_v2` 는 현재 `telegram` 과 별개의 채널 타입 또는 별도 channel mode 로 취급한다.

최소 요구사항은 아래와 같다.

- 기존 `telegram` 동작 무변경
- 별도 설정 플래그 또는 별도 bot token, route 가능성 고려
- session, thread mapping 은 기존 Telegram bridge 와 최대한 재사용
- reasoning visibility 기본값은 `status_only`

### 9.2 UX 구성

`telegram_v2` 는 두 surface 를 동시에 가진다.

1. chat surface
   - 짧은 답변
   - live status, trace update
   - quick actions
   - Mini App 열기 버튼

2. Mini App surface
   - thread view
   - collapsed summary card
   - expanded trace card
   - task, result detail
   - visibility setting

### 9.3 chat surface 동작 원칙

- 최초 사용자 메시지 수신 후 즉시 neutral tone 의 reasoning message 를 하나 만든다.
- 진행 중에는 같은 메시지를 edit 하면서 현재 step 또는 trace line 을 갱신한다.
- 완료 시에는 같은 메시지를 collapsed summary 형태로 정리한다.
- 최종 답변은 별도 일반 message 로 마무리하거나, 필요 시 reasoning summary 아래에 이어 붙인다.
- 복잡한 상세 trace 는 inline keyboard 의 `Open details` 버튼으로 Mini App 에 넘긴다.

Telegram chat 은 색상 자유도가 제한되므로, gray tone 원칙은 아래처럼 해석한다.

- 과도한 emoji, status color 강조를 피한다.
- 짧은 label, 인용형 블록, code-style line, 절제된 아이콘으로 neutral hierarchy 를 만든다.
- 최종 상태는 긴 로그를 남기지 않고 compact summary 로 정리한다.

### 9.4 Mini App 동작 원칙

- Telegram 안에서 WebUI-lite 를 제공한다.
- session, thread 상태를 읽고 동일 turn 의 reasoning summary, trace 를 보여준다.
- write access 가 있을 때는 quick reply, retry, approve, reopen detail 같은 action 을 처리한다.
- theme, fullscreen, safe-area 대응을 기본 가정으로 둔다.
- visual hierarchy 는 WebUI 와 동일하게 neutral gray card 기준으로 맞춘다.
- 실행 중에는 expanded trace card, 완료 후에는 collapsed summary card 를 기본 상태로 둔다.

### 9.5 왜 chat-only 가 아니라 Mini App 인가

chat-only 방식의 한계는 아래와 같다.

- 긴 reasoning disclosure 가 채팅 로그를 과도하게 오염시킨다.
- 상태 패널, trace timeline, 설정 토글을 안정적으로 표현하기 어렵다.
- 기존 답변과 reasoning 부가정보의 정보 계층을 분리하기 어렵다.

반면 Mini App 은 아래 요구를 충족하기 쉽다.

- WebUI 와 유사한 상태 패널
- reasoning summary, debug trace 분리
- session-linked detail view
- 버튼, 토글, 설정 변경, deep link

## 10. 설정 모델 제안

### 10.1 전역 설정

```json
{
  "reasoning_visibility": "summary"
}
```

### 10.2 채널 기본값

```json
{
  "channels": {
    "webui": {
      "reasoning_visibility": "summary"
    },
    "telegram": {
      "reasoning_visibility": "status_only"
    },
    "telegram_v2": {
      "enabled": true,
      "reasoning_visibility": "status_only",
      "mini_app_enabled": true
    }
  }
}
```

### 10.3 세션 override

세션 단위로 아래 같은 override 를 둘 수 있다.

```json
{
  "session": {
    "reasoning_visibility": "debug_trace"
  }
}
```

## 11. 구현 단계 제안

### Phase 1. reasoning contract 정리

- runtime 내부 event 레이어 정리
- `reasoning_visibility` enum 추가
- session, channel 에 visibility 전달 경로 추가
- 기존 progress, tool hint 를 새 contract 에 매핑

완료 기준:

- WebUI 와 Telegram 양쪽에서 같은 status event 의미를 공유한다.

### Phase 2. WebUI 적용

- visibility state 추가
- `off | status_only | summary | debug_trace` UI 매핑 구현
- Copilot-style reasoning card 추가
- 실행 중 expanded, 완료 후 collapsed 동작 구현
- summary disclosure, debug trace reopen panel 추가
- 최소 테스트 보강

완료 기준:

- WebUI 에서 각 visibility mode 가 실제로 차등 렌더링된다.
- 실행 중 expanded card 가 보이고 완료 후 collapsed summary 로 전환된다.

### Phase 3. `telegram_v2` chat surface 추가

- 기존 Telegram 과 분리된 채널 등록
- 단일 reasoning message 의 edit-based live update 구현
- quick action, Mini App 진입 버튼 구현
- 완료 시 collapsed summary message 형태 정리

완료 기준:

- 기존 Telegram channel untouched
- `telegram_v2` 로 Copilot-like progressive reasoning rhythm 제공

### Phase 4. Telegram Mini App 연결

- Mini App bootstrap
- thread, summary, trace, detail view 구현
- Telegram deep link, button 연동

완료 기준:

- Telegram 안에서 WebUI-lite reasoning surface 사용 가능

## 12. 검증 계획

### 12.1 WebUI

- 각 visibility mode 별 thread 렌더링 확인
- 실행 중 expanded card 가 보이고 완료 후 collapsed summary 로 전환되는지 확인
- approval pending, tool running, fallback 상황에서 노출 차이 확인
- 기존 thread UX 가 과도하게 무거워지지 않는지 확인

### 12.2 Telegram

- 기존 `telegram` 채널 regression 없음 확인
- `telegram_v2` 에서 edit-based live reasoning message, collapsed summary, final answer, quick action 동작 확인
- summary, link-to-detail 흐름 확인

### 12.3 Session / replay

- summary 와 trace 가 저장, 재생 시 일관된지 확인
- visibility 가 다른 세션에서 replay 오동작이 없는지 확인

## 13. 리스크와 대응

### 13.1 reasoning 데이터 과노출

대응:

- 기본값을 `summary` 또는 `status_only` 로 유지
- hidden reasoning 과 observable reasoning 을 분리
- debug trace 도 구조화된 의미 정보만 노출

### 13.2 UI 복잡도 증가

대응:

- WebUI 는 단일 reasoning card + collapsed summary 중심으로 시작
- trace 는 disclosure 또는 inspector 로 격리

### 13.3 Telegram 구현 범위 과대화

대응:

- `telegram_v2` 1차는 chat surface + Mini App 진입점만 우선 구현
- Mini App 을 WebUI 완전 복제품으로 만들지 않음

## 14. 최종 제안

이번 작업은 아래 순서로 진행하는 것이 가장 안전하다.

1. observable reasoning contract 를 먼저 정리한다.
2. WebUI 에 `reasoning_visibility` 4단계를 먼저 적용한다.
3. 기존 Telegram 을 유지한 채 `telegram_v2` chat surface 를 추가한다.
4. 마지막에 Telegram Mini App 을 붙여 richer surface 를 완성한다.

이 순서를 택하면 기존 운용 채널을 깨지 않으면서, WebUI 와 Telegram v2 가 같은 reasoning 의미 체계를 공유하게 만들 수 있다.
