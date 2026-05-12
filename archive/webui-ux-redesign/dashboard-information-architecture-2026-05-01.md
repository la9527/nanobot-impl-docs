# Nanobot Assistant Dashboard Information Architecture - 2026-05-01

> 용어 메모: 이 문서는 당시 정보구조 논의를 보존하기 위해 예전 표현을 유지한다. 현재 WebUI 용어 기준은 [docs/guide/10-webui-terminology.md](../../guide/10-webui-terminology.md) 를 따른다.

## 1. 문서 목적

이 문서는 thread 화면을 chat-first 로 되돌리는 대신, owner-wide aggregate 를 수용할 별도 home/dashboard surface 를 어떻게 설계할지 정보구조 수준에서 구체화한 문서다.

이번 문서의 목표는 아래 네 가지다.

- dashboard 에 무엇을 보여줄지 정의한다.
- 무엇을 dashboard 에서만 보고, 무엇을 thread 에 남길지 경계를 정한다.
- 1인 사용자 개인 assistant 기준으로 "매일 처음 보는 화면"의 우선순위를 정한다.
- 이후 WebUI 구현 시 바로 wireframe 수준으로 옮길 수 있는 구조를 남긴다.

## 2. dashboard 의 역할 정의

dashboard 는 새로운 채팅 화면이 아니다. dashboard 의 역할은 아래로 제한해야 한다.

1. 지금 당장 사용자가 처리해야 할 것의 우선순위를 보여준다.
2. assistant 가 여러 thread 와 channel 에 걸쳐 쌓아둔 상태를 한곳에 모은다.
3. 상세 처리 자체는 thread 로 넘기고, dashboard 는 진입과 판단을 돕는다.

즉, dashboard 는 execution cockpit 이지, message history viewer 가 아니다.

추가로 dashboard 는 reply surface 가 아니다. home 상태에서는 composer, message input, draft entry 를 같이 두지 않는다.

## 3. 핵심 제품 원칙

### 원칙 1. dashboard 는 "overview + next action" 중심이어야 한다

- 장황한 설명보다 지금 처리할 것 3개가 중요하다.
- summary 는 짧고 action 은 명확해야 한다.

### 원칙 2. 모든 정보를 한 번에 다 보여주지 않는다

- top priority queue
- recent context
- quick entry

이 세 묶음이면 충분하다.

### 원칙 3. dashboard 는 thread 를 대체하지 않는다

- reply 쓰기
- 세부 message 읽기
- 긴 결과 검토

이 행동은 thread 에서 한다.

### 원칙 4. cross-thread state 만 dashboard 에 올린다

- approvals
- blocked tasks
- proactive briefings
- linked external session health
- 최근 완료된 automation 결과

반대로 single-thread 내부 디테일은 dashboard 에서 요약만 하고 thread 로 보낸다.

## 4. 권장 화면 구조

dashboard 는 세로 스크롤 단일 컬럼을 기본으로 하고, desktop 에서만 일부 구역을 2단으로 확장하는 것이 맞다. 이유는 모바일과 desktop 모두에서 같은 우선순위를 유지하기 위해서다.

navigation 진입은 sidebar 상단의 고정 `Dashboard` 항목을 기준으로 한다. dashboard 는 채팅 리스트 안의 특수 row 나 thread header 의 숨은 버튼에 의존하지 않는다.

권장 구조는 아래 순서다.

1. hero summary
2. priority queue
3. today brief
4. recent outcomes
5. linked channels
6. quick actions

## 5. 각 섹션 상세 정의

## 5.1 hero summary

### 목적

dashboard 진입 직후 현재 assistant 상태를 한 문장으로 이해시키는 구역이다.

### 표시 정보

- 현재 가장 중요한 한 줄 요약
- pending approvals 수
- blocked tasks 수
- 오늘 proactive briefing 존재 여부

### 예시 문구

```text
오늘 바로 처리할 항목 2개가 있습니다. 메일 승인 1건과 일정 충돌 1건이 대기 중입니다.
```

### 주의점

- 여기서 linked session 수나 owner defaults 같은 보조 정보까지 넣지 않는다.
- hero 는 설명 패널이 아니라 orientation 용이다.

## 5.2 priority queue

### 목적

사용자가 실제로 눌러야 하는 항목을 우선순위대로 보여주는 메인 구역이다.

표현은 `mini card` 묶음보다 `compact action row` 에 가깝게 유지한다. dashboard 전체가 또 하나의 widget board 로 보이지 않게 하기 위함이다.

### 카드 유형

- Approval required
- Blocked task
- Waiting input
- Proactive review needed

### 카드 필드

- title
- status badge
- source channel
- last updated
- next step 한 줄
- primary CTA
- secondary CTA

### CTA 예시

- `승인 열기`
- `대화 이어가기`
- `WebUI에서 검토`
- `해당 thread 열기`

### 정렬 우선순위

1. approval pending
2. blocked but resumable
3. waiting input
4. proactive held
5. recent completed follow-up

## 5.3 today brief

### 목적

오늘 일정과 mail/calendar 요약을 operational context 수준으로 제공하는 구역이다.

### 포함 정보

- 오늘 일정 수와 첫 일정
- 중요 메일 요약 한 줄
- 오전/오후 기준 next best action

### 표현 방식

- 긴 상세 리스트를 그대로 보여주지 않는다.
- "오늘 일정 3개, 첫 일정 10:00 프로젝트 리뷰" 같은 compressed line 중심으로 간다.

### 비포함 정보

- 메일 thread 전문
- calendar event 상세 description

이것들은 thread 나 detail sheet 로 넘긴다.

## 5.4 recent outcomes

### 목적

assistant 가 최근 완료한 작업을 확인하고, 필요 시 후속 thread 로 들어가게 하는 구역이다.

### 포함 항목

- 최근 생성한 draft
- 최근 conflict check 결과
- 최근 event create 결과
- 최근 proactive briefing 결과

### 카드 형태

- 작은 timeline card 3~5개
- 각 카드에 result type, summary, opened-at, open-thread CTA

### 이유

현재 thread 상단에 고정된 action result preview 는 문맥을 밀어내지만, dashboard 의 recent outcomes 구역에 있으면 자연스럽다.

## 5.5 linked channels

### 목적

assistant 가 현재 어떤 외부 채널과 이어져 있는지 운영 관점으로 보여주는 구역이다.

### 포함 정보

- Telegram, WebUI 등 주요 channel badge
- 마지막 activity 시각
- trust/linked 상태
- quiet hours 로 hold 중인지 여부

### 주의점

- chat identity 세부값을 전면에 크게 드러내지 않는다.
- channel health/status 용으로 축약한다.

## 5.6 quick actions

### 목적

새로운 작업을 시작하거나 자주 쓰는 assistant entry point 로 빠르게 이동시키는 구역이다.

### 권장 버튼

- `오늘 일정 보기`
- `최근 메일 보기`
- `승인 대기 보기`
- `막힌 작업 이어가기`
- `새 채팅 시작`

### 포함하지 않을 것

- memory correction chip 전체 묶음

이것은 quick action 이 아니라 power tool 이므로 details drawer 또는 composer menu 쪽이 더 적합하다.

## 6. dashboard 와 thread 의 책임 분리

### dashboard 에 남길 것

- owner-wide aggregate
- multi-thread priority queue
- 최근 완료 결과 묶음
- channel health overview
- thread 진입 CTA

### thread 에 남길 것

- 현재 대화 메시지
- 현재 thread 의 얇은 status rail 또는 alert
- 최근 assistant action 의 inline result card
- reply 작성과 context reading

### drawer 에 남길 것

- current task detail
- owner defaults
- linked session detail
- memory tools
- metadata explanation

### dashboard 에 두지 않을 것

- sticky composer
- message history preview
- thread 내부 전용 approval response UI
- memory correction quick action 전체 노출

## 7. 추천 와이어프레임

```text
[Hero]
오늘 바로 처리할 항목 2개가 있습니다. 메일 승인 1건, 일정 충돌 1건.

[Priority Queue]
- 메일 승인 필요 / Telegram / 8분 전 / 승인 열기
- 일정 충돌 검토 / WebUI / 21분 전 / 대화 이어가기

[Today Brief]
- 오늘 일정 3개, 첫 일정 10:00 프로젝트 리뷰
- 중요 메일 2건, 답변 대기 1건

[Recent Outcomes]
- Draft ready / Budget follow-up / thread 열기
- Conflict found / 14:00-14:30 / thread 열기

[Linked Channels]
- Telegram 연결됨 / 8분 전 활동
- WebUI 연결됨 / quiet hours hold 가능

[Quick Actions]
[승인 대기 보기] [오늘 일정 보기] [최근 메일 보기] [새 채팅]
```

## 8. 진입 흐름 정의

### 흐름 A. 일반 진입

- 사용자가 WebUI 를 열면 dashboard/home 에 도착
- hero 와 priority queue 를 먼저 확인
- 필요한 항목을 눌러 해당 thread 로 이동

### 흐름 B. deep link 진입

- Telegram mirroring 또는 특정 session link 로 바로 thread 에 진입
- thread 는 최소 상태만 보여줌
- 필요 시 `Assistant details` 또는 `홈으로` 로 dashboard 이동

### 흐름 C. 새 작업 시작

- dashboard 의 quick actions 또는 sidebar 의 새 채팅에서 시작
- 새 thread 생성 후 작업 수행

## 9. sidebar 와의 관계

dashboard 를 넣더라도 sidebar 를 대체할 필요는 없다. 대신 sidebar 의 역할을 더 명확히 해야 한다.

- sidebar: thread 목록과 최근 대화 진입
- dashboard: 우선순위와 assistant 운영 개요

권장 보완:

- sidebar 최상단에 `홈` 항목 추가
- thread list 는 그대로 유지
- approval pending 같은 badge 는 sidebar 에 남기되, 본격적인 설명은 dashboard 에서 제공

## 10. 구현 우선순위

1. thread 상단 과밀 구조부터 정리
2. compact rail + drawer 체계 도입
3. 그 다음 dashboard/home 추가
4. 마지막으로 sidebar 와 dashboard 간 이동 polish

즉, dashboard 는 중요하지만 thread slimming 이 선행이다. 그래야 dashboard 가 보완 surface 가 되고, 또 하나의 과밀 화면이 되지 않는다.

## 11. 최종 판단

dashboard 는 Nanobot 에 꼭 맞는 방향이다. 다만 "정보를 더 넣을 곳"으로 생각하면 실패한다.

성공하는 dashboard 는 아래를 만족해야 한다.

- 사용자가 5초 안에 오늘 처리할 것을 안다.
- 상세 작업은 thread 로 자연스럽게 넘긴다.
- owner-wide aggregate 만 남기고 single-thread detail 은 욕심내지 않는다.

이 기준을 지키면 dashboard 는 thread 를 무겁게 만드는 대신, thread 를 가볍게 되돌리는 핵심 구조가 된다.