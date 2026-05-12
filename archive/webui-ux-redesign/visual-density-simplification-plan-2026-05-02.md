# Nanobot WebUI Visual Density Simplification Plan - 2026-05-02

> 용어 메모: 이 문서는 2026-05-02 기준 density 검토 기록이라 `dashboard`, `thread` 같은 당시 표현을 유지한다. 현재 WebUI 용어 기준은 [docs/guide/10-webui-terminology.md](../../guide/10-webui-terminology.md) 를 따른다.

## 1. 문서 목적

이 문서는 현재 진행 중인 thread/dashboard 개선을 계속 밀기 전에, 화면이 넓고 느슨하게 보이는 문제를 먼저 줄이기 위한 `visual density pass` 기준안을 정리한다.

단, calendar prompt, dashboard navigation, pending approval 상태처럼 기능 신뢰성 자체가 흔들리는 문제는 별도 재검토 문서인 [calendar-webui-reliability-review](../calendar-webui-reliability-review/README.md) 를 우선 기준으로 삼는다. 이 문서는 그 위에 얹는 visual density 기준이다.

핵심 목표는 아래 두 가지다.

1. 정보량을 줄이는 것과 별개로, 같은 정보를 더 짧고 더 촘촘하게 배치한다.
2. chat-first 방향을 유지하되, 여백과 카드 부피 때문에 화면이 허전해 보이는 현상을 줄인다.

이 문서는 구현 명세 초안이며, 다음 UI polish 작업의 우선순위와 판단 기준으로 사용한다.

## 2. 현재 문제를 spacing 관점에서 다시 보면

이번까지의 구조 개선으로 상단 카드 묶음은 줄었지만, 화면이 여전히 크게 느껴지는 이유는 정보 구조보다 레이아웃 밀도에 있다.

현재 확인된 대표 원인은 아래와 같다.

1. thread empty/home 상태의 세로 시작 여백이 크다.
   - `ThreadViewport` 는 no-session 상태에서 `pt-14`, `pb-16`, `gap-5`, `max-w-[40rem]` 조합을 사용한다.
   - 결과적으로 첫 콘텐츠가 화면 중간쯤에서 시작하면서 빈 공간이 크게 느껴진다.

2. dashboard 섹션 카드가 모두 큰 카드 규격으로 통일돼 있다.
   - hero, queue, recent outcomes, linked channels가 모두 `rounded-[18px~24px]`, `p-3~p-5`, `space-y-3~4`를 사용한다.
   - 정보 우선순위는 달라졌는데 카드 밀도는 거의 같아서, 화면 전체가 위젯 보드처럼 퍼져 보인다.

3. width 는 넓은데 content line-length 는 짧아서 공백이 많다.
   - `max-w-[58rem]`, `max-w-[64rem]` 기준에서 실제 텍스트는 짧게 끝난다.
   - 특히 home 상태에서는 각 카드 안 문장이 짧아서 좌우 여백이 더 크게 체감된다.

4. status rail 과 inline result 도 card 스타일이 약간 과하다.
   - 둘 다 정보량은 작아졌지만 border, padding, shadow, round 값이 여전히 “카드” 느낌을 강하게 만든다.
   - 그래서 slimmed thread 라기보다 “작은 위젯 여러 개”처럼 보일 수 있다.

5. composer 상단 여백과 하단 sticky wrap 이 넉넉하다.
   - dashboard 아래에서 composer가 별도 panel처럼 떠 보이면서 vertical rhythm 이 두꺼워진다.

6. home/dashboard 상태에 composer 가 남아 있으면 surface 책임이 섞인다.
   - dashboard 는 overview 인데 하단 입력창이 보이면 새 채팅 시작 화면처럼 읽힌다.
   - 이 경우 정보 구조가 정리되어도 제품 인상이 다시 흐려진다.

## 3. 이번 density pass 의 목표

이번 단계는 기능 추가가 아니라 `덜 크고 덜 많은 것처럼 보이게 만드는 것` 이 목표다.

정확히는 아래 상태를 만들고 싶다.

1. thread 는 첫 인상에서 거의 메신저처럼 보여야 한다.
2. dashboard 도 “운영 개요”는 유지하되, full card board 보다 compact briefing 에 가까워야 한다.
3. 한 screen 안에서 서로 경쟁하는 box 수를 줄여야 한다.
4. 같은 정보라도 badge, inline row, compact list를 우선 사용해야 한다.

## 4. 기본 원칙

### 원칙 1. 큰 박스보다 얇은 줄이 우선이다

- 가능하면 카드 1개 대신 row 2개로 푼다.
- section 전체를 boxed panel 로 만들기보다 heading + compact list 를 우선한다.

### 원칙 2. vertical spacing 을 먼저 줄인다

- 이번 체감 문제는 width 보다 height 에서 더 크게 발생한다.
- 우선 `pt`, `pb`, `gap`, `space-y`, `p` 값을 한 단계씩 내린다.

### 원칙 3. radius 와 shadow 를 줄여서 panel 느낌을 약하게 만든다

- 지금은 구조는 slimmed 되었지만 시각 언어는 여전히 large card 중심이다.
- round 값과 shadow 를 줄이면 같은 컴포넌트도 더 가볍게 느껴진다.

### 원칙 4. home 은 “dashboard”보다 “briefing”에 가깝게 만든다

- 1인 사용자 assistant 에서는 대형 dashboard 보다 compact briefing board 가 더 자연스럽다.
- hero + queue + recent 정도만 분명하고, 나머지는 압축된 보조 블록이면 충분하다.

## 5. 권장 방향

## Direction A. Thread 는 compact messenger density 로 맞춘다

### 권장 변경

1. no-session / home 진입 영역 상단 여백 축소
   - `pt-14` 계열을 더 짧게 줄인다.
   - hero 와 composer 사이 간격도 한 단계 줄인다.

2. status rail 을 “mini info row”로 축소
   - outer padding 축소
   - caption line-height 축소
   - shadow 제거 또는 약화
   - badge 개수는 3개를 기본 상한으로 보고 나머지는 `+N`으로 묶는 방안 검토

3. inline result card 는 detail-first 가 아니라 summary-first
   - title, 상태, 한 줄 요약만 기본 노출
   - preview detail 은 기본 접힘 또는 한 줄 요약화
   - mail/calendar/thread summary 도 full preview 대신 compact disclosure pattern 검토

4. thread viewport container 폭 미세 조정
   - 메시지 영역은 지금보다 약간 좁혀도 됨
   - 대신 바깥 padding 을 줄여 실제 usable area 를 늘리는 방향이 더 낫다.

### 기대 효과

- thread 첫 인상이 훨씬 가벼워진다.
- rail 과 result 가 남아 있어도 “panel stack”처럼 보이지 않는다.

## Direction B. Dashboard 는 card board 가 아니라 briefing stack 으로 바꾼다

### 권장 변경

1. hero section 높이 축소
   - title, one-line summary, count 정도만 남긴다.
   - 소개성 문장은 짧게 줄인다.

2. `Priority queue` 를 가장 강하게, 나머지는 약하게
   - queue 만 card 느낌 유지
   - `Today brief`, `Recent outcomes`, `Linked channels`, `Quick actions` 는 lighter section 또는 list style 로 전환

3. 2-column block 을 유지하더라도 section 내부는 compact list로 바꾼다.
   - 지금은 각 칸이 또 카드라서 박스 수가 많다.
   - heading 아래 2~3 row summary 로 바꾸면 훨씬 단정해진다.

4. `Recent outcomes` 와 `Linked channels` 는 badge + one-line summary 위주로 압축
   - “해당 thread 열기” 버튼은 hover 또는 secondary action 수준으로 낮출 수 있다.

5. `Quick actions` 는 button bar 를 더 얇게
   - outline button 높이와 padding 축소
   - 4개를 모두 같은 강조도로 두지 않는다.

6. dashboard 는 dashboard 정보만 보여준다.
   - home 상태의 하단 composer 는 제거한다.
   - 새 채팅 시작은 sidebar 고정 `Dashboard` / `New chat` navigation 과 dashboard 내부 CTA 로 분리한다.

### 기대 효과

- home 이 큰 위젯 모음처럼 보이지 않고, assistant briefing board 처럼 보인다.
- 빈 공간이 줄어들고, 스크롤 길이도 짧아진다.

## 6. 가장 현실적인 1차 적용안

우선은 전체 재설계보다 아래 5개만 먼저 적용하는 것이 가장 비용 대비 효과가 크다.

1. `ThreadViewport` home/no-session vertical spacing 축소
2. `AssistantDashboard` hero 높이와 section gap 축소
3. dashboard 보조 section 4개를 card 에서 compact list 스타일로 낮춤
4. `ThreadStatusRail` 의 padding, shadow, radius 축소
5. `ThreadInlineActionResult` 기본 높이를 줄이고 detail 노출량 축소

이 5개만 해도 화면이 훨씬 덜 넓고 덜 무겁게 보일 가능성이 높다.

## 7. 구현 시 구체 체크포인트

### thread

- first content block 이 fold 상단 가까이 시작하는가
- rail + status + composer 가 한 화면 안에서 너무 많은 높이를 차지하지 않는가
- action result 가 message 흐름을 밀어내지 않는가

### home

- 첫 화면에서 2번 스크롤 없이 핵심 정보가 보이는가
- queue 외 section 이 모두 같은 무게로 보이지 않는가
- button row 가 control panel 처럼 두껍지 않은가

### visual language

- round 값은 한 단계 줄였는가
- shadow 는 정보 우선순위가 높은 surface 에만 남았는가
- 한 컴포넌트 안에서 `p-5`, `p-4`, `space-y-4` 가 과도하지 않은가

## 8. 다음 작업 권장 순서

1. `AssistantDashboard` compact briefing pass
2. `ThreadStatusRail` / `ThreadInlineActionResult` compact pass
3. `ThreadViewport` no-session spacing pass
4. live browser re-check

## 9. 결론

지금 필요한 것은 새로운 정보 구조를 더 추가하는 것이 아니라, 이미 줄여놓은 구조를 더 작고 더 촘촘하게 보이게 만드는 것이다.

따라서 다음 단계는 `기능 확장`보다 `density reduction` 이 우선이다.

한 줄로 정리하면 아래와 같다.

`dashboard 는 briefing 처럼, thread 는 messenger 처럼, rail/result 는 mini surface 처럼 보이게 줄인다.`