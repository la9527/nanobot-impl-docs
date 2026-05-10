# Nanobot 사용자 설명서

## 이 문서 묶음의 목적

이 문서 묶음은 현재 로컬에서 운영 중인 Nanobot 을 처음 쓰는 사람이 빠르게 이해할 수 있도록 정리한 사용자 설명서다.

원래 Nanobot 기본 소스가 제공하는 기능 전체를 다시 길게 설명하기보다, 현재까지 이 환경에서 추가로 작업한 개인비서 기능과 WebUI 운영 흐름을 중심으로 정리한다.

## 먼저 알아둘 것

현재 Nanobot 은 단순 채팅봇이 아니라 아래 역할을 함께 한다.

- WebUI 로 대화하고 상태를 확인하는 개인비서
- Telegram 채팅과 WebUI 를 이어 주는 브리지
- 일정, 메일, 작업 상태를 다루는 자동화 입구
- 로컬 LLM 과 smart-router 를 오가며 답변하는 모델 라우팅 환경
- 반복 작업과 기억 정리를 도와주는 로컬 에이전트

## 빠른 시작

1. WebUI 를 연다.

```text
http://127.0.0.1:8765/webui/
```

2. 왼쪽 사이드바에서 원하는 채팅을 고른다.

- 일반 WebUI 채팅: Nanobot 과 새로 대화하는 공간
- Telegram 채팅: Telegram 앱에서 오간 대화가 WebUI 에도 보이는 공간

3. 메시지 입력창에 질문이나 명령을 입력한다.

예시:

```text
현재 상태를 알려줘
/calendar
/model
오늘 일정 알려줘
```

## 문서 목록

- [00. 현재 Nanobot 추가 기능 한눈에 보기](00-current-feature-map.md)
- [01. 화면 구성과 WebUI 사용법](01-webui-overview.md)
- [02. Telegram 연동 채팅 사용법](02-telegram-bridge.md)
- [03. 자주 쓰는 명령어](03-chat-commands.md)
- [04. 일정 자동화와 승인 흐름](04-calendar-automation.md)
- [05. 모델 선택과 smart-router](05-model-and-smart-router.md)
- [06. 상태, 작업, 메모리 관리](06-status-task-memory.md)
- [07. 반복 작업과 알림](07-periodic-tasks.md)
- [08. 메일 자동화 파일럿](08-mail-automation.md)
- [09. 운영 확인과 문제 해결](09-troubleshooting.md)
- [10. WebUI 용어 기준](10-webui-terminology.md)
- [11. 심화 참고 문서 안내](11-advanced-reference.md)

## 가장 많이 쓰는 명령

```text
/status
/calendar
/calendar today
/calendar cancel
/model
/model list
/model smart-router
/model clear
/help
```

## 현재 기준으로 중요한 주의사항

- WebUI 변경은 운영 반영 전에 production bundle build 가 필요하다.
- Telegram 앱에서 정상으로 보이는 답변도 WebUI history 에 저장되어야 WebUI 에 오래 남는다.
- 일정 생성은 승인 흐름을 거치며, 충돌이 있으면 승인 전에 충돌 안내가 나온다.
- 삭제나 실행 명령 같은 위험한 작업은 승인 또는 별도 도구가 필요할 수 있다.
- live 설정 파일인 `~/.nanobot/config.json`, `~/.nanobot/config.api.json`, `~/.nanobot/nanobot.env` 는 함부로 초기화하지 않는다.
