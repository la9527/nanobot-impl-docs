# 07. 반복 작업과 알림

## 반복 작업이란

Nanobot gateway 는 주기적으로 workspace 의 `HEARTBEAT.md` 를 확인해 할 일을 실행할 수 있다.

이 기능은 매번 사용자가 직접 말하지 않아도 정해 둔 작업을 확인하게 만드는 용도다.

예시:

- 아침 브리핑
- 오늘 일정 확인
- pending approval 확인
- 막힌 작업 확인
- 중요한 메일 확인
- 날씨 요약

## 기본 파일 위치

반복 작업 파일은 보통 아래 위치에 있다.

```text
~/.nanobot/workspace/HEARTBEAT.md
```

## 예시 형식

```markdown
## Periodic Tasks

- [ ] Morning briefing: review today's calendar, pending approvals, blocked tasks, and important inbox follow-ups
- [ ] Check weather forecast and send a summary
- [ ] Scan inbox for urgent emails
```

## 자연어로 추가하기

사용자가 직접 파일을 열지 않고 Nanobot 에게 말할 수도 있다.

예시:

```text
매일 아침 오늘 일정과 중요한 메일을 확인하는 반복 작업을 추가해줘
```

## proactive heartbeat

최근 작업으로 Nanobot 은 heartbeat 결과를 어느 채널에 보낼지 더 똑똑하게 판단하도록 보강됐다.

현재 정책 요약:

- quiet hours 밖이면 최근 사용한 external channel 을 우선한다.
- Telegram 이 최근 활성 채널이면 Telegram 쪽으로 보낼 수 있다.
- quiet hours 이거나 외부 전송이 적절하지 않으면 WebUI 쪽에 hold/suppression 상태를 남긴다.

## quiet hours 란

quiet hours 는 사용자를 방해하지 않도록 외부 알림을 줄이는 시간대다.

이 시간대에는 Telegram 같은 외부 채널로 바로 보내지 않고 WebUI 에 상태만 남기는 방식이 더 안전하다.

## WebUI 에서 보이는 정보

반복 작업이나 proactive 상태는 아래 위치에 나타날 수 있다.

- Dashboard
- thread status rail
- assistant details
- chat list badge

## 추천 반복 작업

초보자에게 추천하는 작업은 아래다.

```markdown
- [ ] Morning briefing: 오늘 일정, 승인 대기, 막힌 작업, 중요한 메일 확인
- [ ] Calendar check: 오늘 일정 충돌이나 빈 시간 확인
- [ ] Inbox check: 중요한 메일 초안 후보 확인
- [ ] Task cleanup: 완료된 작업과 남은 작업 요약
```

## 주의사항

- gateway 가 실행 중이어야 반복 작업이 동작한다.
- Nanobot 이 어느 채널로 보낼지 알 수 있도록 한 번 이상 대화한 채널이 있어야 한다.
- 반복 작업은 실제 외부 행동을 만들 수 있으므로, 처음에는 요약/확인 위주로 두는 것이 좋다.
