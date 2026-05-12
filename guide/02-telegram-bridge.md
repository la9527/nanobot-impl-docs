# 02. Telegram 연동 채팅 사용법

## Telegram 연동이란

Telegram 연동은 Telegram 앱에서 오간 Nanobot 대화를 WebUI 에서 보고, WebUI 에서 같은 Telegram 채팅으로 답장을 보내는 기능이다.

즉, 아래 두 화면이 같은 세션을 공유한다.

- Telegram 앱
- Nanobot WebUI

## 어디에서 보이나

WebUI 왼쪽 사이드바에 Telegram 채팅이 나타난다.

보통 아래처럼 표시된다.

- 채널: Telegram
- 채팅 ID 또는 미리보기
- 최근 메시지
- 승인 또는 blocked 상태 badge

## WebUI 에서 Telegram 채팅에 답장하기

1. WebUI 왼쪽에서 Telegram 채팅을 선택한다.
2. 메시지 입력창에 답장을 입력한다.
3. Enter 를 누른다.
4. 답장은 Nanobot 을 거쳐 실제 Telegram 채팅에도 반영된다.

## slash command 도 사용할 수 있나

사용할 수 있다.

예시:

```text
/calendar
/calendar today
/calendar cancel
/status
/model
```

최근 보강으로 Telegram 세션에서 `/calendar` 같은 명령을 WebUI 로 입력해도 결과가 잠깐 보였다가 사라지지 않고 대화 기록에 유지된다.

## 왜 예전에는 결과가 사라졌나

문제 원인은 다음과 같았다.

- Telegram 앱은 Nanobot 의 즉시 답변을 바로 받는다.
- WebUI 는 세션 기록을 주기적으로 다시 읽는다.
- 그런데 slash command 결과가 세션 파일에 저장되지 않으면, WebUI 가 새로고침하면서 이전 기록으로 화면을 덮었다.

현재는 command 결과도 user/assistant 메시지로 저장되도록 보강했다.

## WebUI 와 Telegram 앱의 차이

Telegram 앱:

- 실제 사용자가 보는 외부 채널
- 답변이 바로 도착하면 정상으로 보일 수 있음

WebUI:

- 저장된 session history 를 다시 읽음
- 기록 저장이 빠지면 화면에서 사라질 수 있음
- 현재는 이 부분을 보강해 slash command 결과가 유지됨

## 주의사항

- WebUI 에서 Telegram 채팅을 다룰 때는 선택한 세션이 맞는지 확인한다.
- 같은 사용자의 Telegram 채팅이라도 topic/thread 가 있으면 세션이 나뉠 수 있다.
- WebUI 가 오래 켜져 있었다면 새로고침 후 다시 확인하는 것이 좋다.

## 문제가 생겼을 때

아래 순서로 확인한다.

1. Telegram 앱에서는 답변이 보이는지 확인한다.
2. WebUI 에서 같은 Telegram 세션을 선택했는지 확인한다.
3. `/calendar` 같은 명령 결과가 몇 초 뒤에도 남아 있는지 확인한다.
4. 계속 사라지면 gateway 로그와 session history 저장 여부를 확인한다.
