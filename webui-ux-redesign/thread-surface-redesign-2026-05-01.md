# Nanobot WebUI Thread Surface Redesign 검토 - 2026-05-01

> 용어 메모: 이 문서는 2026-05-01 당시의 UX 판단 기록이라 `thread`, `dashboard`, `Assistant details` 같은 예전 표현을 포함한다. 현재 WebUI 용어 기준은 [docs/user-guide/10-webui-terminology.md](../user-guide/10-webui-terminology.md) 를 따른다.

## 1. 문서 목적

이 문서는 Nanobot WebUI 의 현재 thread 화면이 owner-aware summary, current task, linked session, action result preview, thread status block 을 동시에 노출하면서 채팅 자체가 압박받는 문제를 UX 관점에서 다시 판단하기 위한 작업안이다.

이번 문서는 구현 명세가 아니라 제품 판단 문서다. 즉, 지금 당장 어떤 화면을 만들어야 하는지보다 아래 질문에 답하는 데 목적이 있다.

- 어떤 정보가 thread 안에서 항상 보여야 하는가
- 어떤 정보는 접거나 다른 surface 로 옮겨야 하는가
- 1인 사용자 개인비서라는 전제에서 dashboard 가 thread 보다 더 적합한 정보는 무엇인가
- 다음 구현에서 가장 작은 비용으로 체감 품질을 크게 올릴 수 있는 안은 무엇인가

## 2. 이번 검토에서 확인한 근거

### 2.1 직접 확인한 현재 화면 특징

live WebUI 와 스냅샷 기준으로 현재 thread 화면은 아래 순서로 구성된다.

1. header badge
2. Assistant summary 카드
3. Current task 카드
4. 경우에 따라 action result 카드
5. linked session 안내 카드
6. thread status block
7. 실제 대화 메시지
8. composer

즉, 사용자가 thread 에 들어왔을 때 가장 먼저 보는 것이 대화가 아니라 상태 카드 묶음이 된다.

### 2.2 현재 화면의 구체적 UX 문제

1. 정보 계층이 뒤집혀 있다.
thread 의 1차 목적은 대화 읽기와 응답인데, 현재는 시스템 상태 카드가 대화보다 우선 배치된다.

2. 같은 의미의 정보가 여러 레이어에 중복된다.
approval pending, blocked, next step, latest update 같은 메시지가 assistant summary, current task, status block, action result 에 분산 반복된다.

3. 요약 카드가 너무 정적이다.
지금 카드는 항상 펼쳐져 있고, 사용자가 그 순간에 필요하지 않아도 세로 공간을 지속 점유한다.

4. memory correction quick action 이 과하게 전면 배치돼 있다.
이 기능은 보조적 생산성 도구인데, thread 상단 주요 카드 내부에서 상시 노출되며 시선 우선순위를 과도하게 가져간다.

5. cross-thread aggregate 와 current-thread state 가 한 덩어리로 섞여 있다.
linked session 수, recent completion, pending approval 같은 owner-wide 상태와 현재 thread 의 실제 task 가 구분되지 않는다.

6. 채팅 시작점이 너무 아래로 밀린다.
특히 노트북 화면이나 모바일에서는 첫 메시지 위치가 fold 아래로 밀리기 쉬워서, thread 진입 즉시 맥락을 읽기 어렵다.

7. action result preview 가 message 가 아니라 control surface 처럼 보인다.
mail/calendar preview 는 사실 최근 작업의 결과물인데, thread 본문 내부 메시지로 읽히기보다 별도 대시보드 위젯처럼 떠 있어 흐름이 끊긴다.

8. action result 내부 detail 이 항상 fully expanded 라서 세로 공간을 과점유한다.
특히 calendar conflict 결과는 requested slot, reason, conflicting event card 들이 한 번에 모두 펼쳐지며, 바로 아래 failed status block 까지 겹치면 실제 대화보다 결과 패널이 더 큰 surface 가 된다.

## 3. 왜 이런 문제가 생겼는가

현재 설계는 backlog 상으로는 합리적이었다. owner backbone, approval visibility, proactive summary, action preview 를 최소 경로로 WebUI 에 노출하는 목적에는 맞았다.

문제는 최소 구현 경로가 그대로 최종 UX 처럼 굳어졌다는 점이다.

- summary 를 노출해야 하니 카드 하나를 추가
- approval 을 노출해야 하니 badge 와 task block 추가
- action result 를 보여야 하니 preview 카드 추가
- memory correction 을 쉽게 써야 하니 quick action chip 추가

이런 방식은 기능 단위로는 맞지만, 제품 화면 차원에서는 누적 부하를 만든다. 지금은 기능이 많아서 불편한 것이 아니라, 각 기능이 모두 같은 시각적 우선순위로 올라와 있기 때문에 불편하다.

## 4. 정보 구조를 다시 나누는 원칙

### 원칙 1. thread 에서는 대화가 주인공이어야 한다

- 사용자가 thread 에 들어오면 가장 먼저 읽어야 하는 것은 최근 메시지와 현재 응답 흐름이다.
- 상태 정보는 대화를 보조해야지, 대화를 밀어내면 안 된다.

### 원칙 2. 항상 보여야 하는 정보는 매우 적어야 한다

- 현재 thread 에 바로 영향을 주는 경고 1개
- pending approval 같은 즉시 행동 필요 상태 1개
- 현재 채널/target 정도의 얇은 메타 정보

이 정도만 always-visible 영역에 둘 수 있다.

### 원칙 3. owner-wide aggregate 는 thread 밖으로 빼는 것이 맞다

- linked session 수
- 다른 채널 recent completion
- blocked session count
- proactive held count

이 정보는 현재 대화보다 한 단계 위의 운영 개요다. 따라서 thread 상단 카드보다 home/dashboard surface 가 더 자연스럽다.

### 원칙 4. 결과물은 가능하면 메시지처럼 읽혀야 한다

mail draft preview, conflict check, create approval preview 같은 것은 "현재 시스템 상태"보다 "방금 수행한 작업의 결과"에 가깝다. 이런 정보는 thread 상단 고정 패널보다 최근 assistant message 에 인접한 결과 card 나 expandable message pattern 이 더 자연스럽다.

추가로 inline result 는 기본적으로 compact 해야 한다. 기본 상태에서는 title, summary, 한 줄짜리 핵심 정보만 보여주고, 충돌 이벤트 목록이나 draft/body preview 같은 상세는 사용자가 열었을 때만 드러나는 disclosure 패턴이 적합하다.

### 원칙 5. power action 은 감춰진 진입점이 더 낫다

memory correction quick action 은 유용하지만 고빈도 1차 행동은 아니다. slash command, composer menu, details drawer 안 action row 로 이동하는 편이 전체 균형이 낫다.

## 5. 대안 비교

## Option A. 현재 thread 유지 + 상단을 단일 compact status rail 로 축소

### 개념

지금 상단 2~4개 카드를 없애고, header 아래에 높이 40~56px 수준의 compact rail 하나만 둔다.

예시:

```text
[승인 1] [막힘 1] [Telegram 연결] [최근 업데이트 8분 전] [세부정보 보기]
```

세부정보를 누르면 drawer 또는 popover 로 아래 정보가 열린다.

- current task
- owner aggregate
- linked session detail
- memory correction action

### 장점

- 가장 빠르게 개선 가능하다.
- thread 화면의 세로 공간을 즉시 회복한다.
- 기존 metadata 모델을 거의 그대로 재사용할 수 있다.

### 단점

- owner-wide 정보가 여전히 thread 안에 남는다.
- 정보량이 많은 날에는 compact rail 이 다시 과밀해질 수 있다.
- 제품 성격이 chat-first 인지는 살리지만 assistant 운영 개요를 별도 surface 로 승격시키지는 못한다.

### 적합도

- 단기 개선안으로 매우 적합
- 최종 형태로는 다소 아쉬움

## Option B. thread 는 chat-first 로 정리하고, 세부 상태는 우측 inspector 또는 mobile sheet 로 이동

### 개념

thread 본문에서는 아래만 남긴다.

- header badge
- 필요 시 1줄짜리 alert banner
- 최근 action result 는 메시지형 inline card

나머지는 "Assistant details" 패널로 이동한다.

- current task
- owner defaults
- linked sessions
- memory correction tools
- proactive summary

desktop 에서는 우측 inspector, mobile 에서는 bottom sheet / full-screen panel 로 보여준다.

### 장점

- chat 영역과 control 영역이 분리되어 사용성이 좋아진다.
- 현재 thread 에 집중할 때 방해가 거의 없다.
- 고급 정보가 완전히 사라지지 않고 필요할 때 열 수 있다.

### 단점

- responsive 설계 비용이 조금 있다.
- inspector 를 닫아둔 상태에선 일부 사용자가 기능 존재를 잊을 수 있다.
- global summary 와 thread detail 이 여전히 같은 패널 안에 혼재할 가능성이 있다.

### 적합도

- 중기 redesign 안으로 적합
- chat-first 원칙을 가장 자연스럽게 반영

## Option C. owner-wide dashboard/home 를 별도 1차 화면으로 두고, thread 는 최소 상태만 유지

### 개념

사용자가 WebUI 에 들어오면 먼저 "Assistant dashboard" 를 본다.

dashboard 에서 보여줄 것:

- 오늘 처리할 승인 대기
- blocked session
- 최근 proactive briefing
- mail/calendar 최근 결과
- linked external session 상태
- 빠른 진입 버튼
  - 승인 보기
  - 오늘 일정 보기
  - 최근 메일 보기
  - 막힌 작업 이어가기

thread 로 들어가면 thread 에는 아래 정도만 둔다.

- target/channel 메타
- pending approval 또는 blocked 여부 1줄 banner
- 최근 action result 는 본문 inline card
- 나머지 owner summary 는 노출하지 않음

### 장점

- 1인 사용자 개인비서 제품과 가장 잘 맞는다.
- cross-thread aggregate 의 의미가 가장 선명해진다.
- thread 를 원래 목적에 맞게 가볍게 만들 수 있다.

### 단점

- 구현 범위가 가장 크다.
- dashboard 를 만들더라도 thread 내부 최소 상태 설계는 별도로 필요하다.
- 사용자가 곧바로 특정 thread 로 들어오는 흐름과 home 진입 흐름을 함께 설계해야 한다.

### 적합도

- 제품 방향성 관점에서는 가장 설득력 있음
- 단기 한 번에 전환하기보다는 단계적 도입이 적합

## 6. dashboard 가 맞는가에 대한 판단

사용자 의견인 "1인 사용자이니 dashboard 형태가 맞지 않나"는 상당히 타당하다. 다만 정확히는 아래처럼 나눠서 보는 것이 맞다.

### 맞는 부분

- owner-aware aggregate 는 본질적으로 dashboard 정보다.
- approvals, blocked tasks, linked sessions, proactive held 상태는 현재 thread 의 일부라기보다 assistant 운영 개요다.
- 한 명의 사용자가 자신의 assistant 전체 상태를 훑는 용도로는 dashboard 가 thread 보다 훨씬 자연스럽다.

### 보완이 필요한 부분

- dashboard 가 있다고 thread 안 정보 구조 문제가 자동으로 해결되지는 않는다.
- thread 에도 최소 상태 표현은 필요하다. 예를 들어 pending approval, blocked, linked external continuation 정도는 현재 대화 문맥상 여전히 중요하다.
- 따라서 "dashboard 로 완전히 이동"보다 "global summary 는 dashboard 로, current-thread urgency 는 thread 에 최소한만"이 맞다.

### 최종 판단

dashboard 는 만들어야 한다. 다만 thread 를 dashboard 처럼 만들면 안 된다.

즉, 제품 구조는 아래가 바람직하다.

- home/dashboard: assistant 전체 운영 상태
- thread: 특정 대화와 실행 흐름
- inspector/drawer: 상세 메타와 power action

추가로 navigation 도 이 구조를 따라야 한다. dashboard 진입점이 현재 thread header 안 숨은 동작처럼 보이면 안 되고, sidebar 상단의 고정 항목으로 존재해야 한다.

## 7. 선택된 진행 방향

이번 UX 개선은 Option B 와 Option C 의 hybrid 를 기준안으로 확정하고 진행한다.

즉, 이 문서에서의 추천안은 더 이상 비교 후보가 아니라 실제 추진 기준이다.

### 확정 구조

1. thread 상단 카드 묶음을 제거한다.
2. thread 에는 compact status rail 또는 1줄 alert 만 남긴다.
3. current task, owner defaults, memory correction, linked session detail 은 details drawer 로 이동한다.
4. owner-wide aggregate 는 별도 dashboard/home 에서 본다.
5. action result preview 는 상단 고정 카드가 아니라 recent assistant message 와 붙은 inline result card 로 재배치한다.
6. inline result card 는 compact summary 를 기본으로 하고, structured detail 은 `View details` 로 접어 둔다.
7. action result card 가 이미 blocked/completed 상태를 설명할 때는 같은 의미의 thread status block 을 중복 노출하지 않는다.
8. dashboard 는 sidebar 상단의 고정 `Dashboard` 항목으로 진입한다.
9. dashboard/home 는 assistant overview surface 이며, message composer 나 reply box 를 포함하지 않는다.
10. thread header 의 제목은 현재 thread 식별용이며, home navigation 동작을 맡지 않는다.

### dashboard 와 thread 전환 규칙

1. sidebar 의 `Dashboard` 는 항상 보이는 1차 navigation 이다.
2. thread header title 은 informational label 로만 유지한다.
3. dashboard 에서 할 수 있는 일은 overview 확인과 thread 진입까지다.
4. 실제 reply 작성, 긴 결과 검토, 승인 응답은 thread 안에서 한다.
5. 따라서 no-session 상태의 main panel 은 dashboard 정보만 렌더하고, 하단 chat composer 는 제거한다.

dashboard 구조 상세안은 [dashboard-information-architecture-2026-05-01.md](./dashboard-information-architecture-2026-05-01.md) 를 따른다.

화면이 넓고 느슨하게 보이는 문제에 대한 별도 compact/density 기준안은 [visual-density-simplification-plan-2026-05-02.md](./visual-density-simplification-plan-2026-05-02.md) 를 따른다.

### 이 방향을 확정한 이유

- 지금 가장 큰 문제인 "한 화면이 꽉 차 보이는 문제"를 즉시 완화한다.
- 현재 구현된 metadata 자산을 버리지 않고 재배치만으로 품질을 올릴 수 있다.
- 장기적으로 dashboard 방향과도 충돌하지 않는다.
- 1인 사용자 제품의 운영 개요와 대화 흐름을 명확히 분리할 수 있다.

## 8. 정보별 권장 배치안

### Assistant summary

현재:
- thread 상단 고정 카드

권장:
- 기본은 dashboard 로 이동
- thread 에서는 summary 전체를 노출하지 않고, 필요한 경우 compact rail 로 축약

### Current task

현재:
- thread 상단 큰 카드

권장:
- pending approval / blocked 인 경우만 얇은 alert 또는 rail badge 로 노출
- 상세 title, next step, owner defaults 는 details drawer 로 이동

### Linked external session

현재:
- 별도 카드 또는 summary 문장으로 반복 노출

권장:
- header meta badge 에 축약
- 세부 identity/trust 설명은 drawer 로 이동

### Action result preview

현재:
- thread 상단 고정 카드

권장:
- 최근 assistant message 아래 inline result card 로 붙이기
- 오래된 결과는 message history 안에 남기고 상단에서는 제거

### Memory correction actions

현재:
- current task 카드 내부 chip 상시 노출

권장:
- composer 좌측 plus/menu 안으로 이동
- 또는 details drawer 의 "Memory tools" 섹션으로 이동
- 정말 자주 쓰는 1개만 composer shortcut 으로 남기고 나머지는 숨김

## 9. 단계별 실행안

## Phase 1. 가장 작은 비용으로 답답함 제거

- Assistant summary + Current task + Action result 상단 카드 제거 또는 기본 collapsed 처리
- thread 상단에는 compact status rail 1개만 유지
- memory correction chip 은 drawer 또는 composer menu 로 이동

기대 효과:
- 첫 메시지가 위로 올라온다.
- 사용자가 thread 진입 직후 바로 대화를 읽을 수 있다.

## Phase 2. action result 의 위치를 본문 친화적으로 재배치

- mail/calendar/conflict preview 를 최근 assistant response 근처 inline result component 로 이동
- thread status block 과 action result 의 역할 중복을 제거

기대 효과:
- 결과물이 시스템 패널이 아니라 작업 결과처럼 읽힌다.
- 상단 chrome 면적이 줄어든다.

## Phase 3. Assistant dashboard/home 도입

- home 에 global queue 성격의 dashboard 추가
- approval, blocked, proactive, recent mail/calendar, linked session 상태를 모아서 보여줌
- sidebar 목록도 단순 thread list 에서 "작업 중심 진입" 구조로 일부 확장 가능
- 구체적인 home 섹션 구조는 `Hero summary -> Priority queue -> Today brief -> Recent outcomes -> Linked channels -> Quick actions` 순서를 기본으로 한다.

기대 효과:
- owner-aware aggregate 의 올바른 집이 생긴다.
- thread 와 운영 개요의 역할이 분리된다.

## 10. 구현 전 체크리스트

1. mobile 에서 compact rail 과 drawer 가 어떻게 접히는지 먼저 결정할 것
2. thread 상단 always-visible 정보는 최대 3개 신호만 남길 것
3. pending approval, blocked, proactive held 중 동시에 여러 개가 있어도 한 줄로 압축 가능한지 검토할 것
4. action result 와 thread status 중 source of truth 를 하나로 정리할 것
5. dashboard 도입 시 home 진입과 direct thread deep-link 진입 모두 자연스럽게 유지할 것

## 11. 최종 제안

Nanobot 는 chat app 안에 assistant 기능을 얹는 수준이 아니라, 1인 사용자 운영 assistant 에 가깝다. 그래서 dashboard 자체는 맞는 방향이다. 다만 그 판단이 곧바로 "thread 안에 많은 summary 를 유지해도 된다"는 뜻은 아니다.

오히려 반대다. dashboard 가 맞기 때문에 thread 는 더 가벼워져야 한다.

따라서 다음 구현의 제품 판단은 아래처럼 가져가는 것이 가장 좋다.

1. thread 는 chat-first 로 되돌린다.
2. owner-wide aggregate 는 dashboard/home 로 이동한다.
3. 세부 메타와 power action 은 drawer 로 이동한다.
4. 결과물은 message 흐름 가까이 붙인다.
5. dashboard 는 overview 와 triage 에 집중하고, 실제 처리와 긴 검토는 thread 로 넘긴다.

이 방향이 현재의 답답한 화면 문제를 가장 직접적으로 해소하면서도, Nanobot 를 개인 assistant product 로 키우는 장기 구조와도 가장 잘 맞는다.