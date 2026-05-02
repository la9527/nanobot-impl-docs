# 01. 화면 구성과 WebUI 사용법

## WebUI 주소

현재 Nanobot WebUI 는 아래 주소에서 사용한다.

```text
http://127.0.0.1:8765/webui/
```

같은 Mac 안에서 접속하는 로컬 주소다. 외부 공개용 주소가 아니다.

## 기본 화면

WebUI 는 크게 세 영역으로 나뉜다.

- 왼쪽 사이드바: 채팅 목록, Dashboard, Settings
- 가운데 채팅 영역: 선택한 세션의 대화 내용
- 상단/상태 영역: 현재 target, 채널, 연결 상태, assistant details

## Dashboard

Dashboard 는 전체 상태를 한눈에 보는 첫 화면이다.

주로 아래 내용을 확인하는 용도다.

- 최근 채팅 흐름
- 진행 중이거나 막힌 작업
- 승인 대기 상태
- 자동화 결과 요약
- 현재 assistant 상태

채팅을 시작하려면 사이드바에서 기존 채팅을 고르거나 새 채팅을 만든다.

## 채팅 목록

왼쪽 목록에는 WebUI 채팅과 Telegram 채팅이 함께 보일 수 있다.

각 항목에는 보통 아래 정보가 붙는다.

- 최근 메시지 미리보기
- 채널 표시
- model target 표시
- 승인 대기 badge

Telegram 채팅은 일반 WebUI 채팅과 달리 실제 Telegram 앱의 대화와 연결된다.

## 메시지 입력

아래 입력창에 메시지를 쓰고 Enter 를 누르면 전송된다.

```text
Enter: 전송
Shift+Enter: 줄바꿈
```

일반 질문과 명령어를 모두 입력할 수 있다.

예시:

```text
현재 상태를 알려줘
/calendar
/model list
```

## Settings

Settings 에서는 WebUI 표시와 모델 관련 상태를 확인한다.

현재 작업된 기능 기준으로 중요한 항목은 아래와 같다.

- theme 전환
- 채팅 글자 크기 조절
- 현재 model target 확인
- 환경변수로 잠긴 설정 확인

## Assistant details

Assistant details 는 현재 답변이나 자동화 결과의 세부 정보를 확인하는 공간이다.

예를 들어 calendar conflict, action result, blocked 상태 같은 내용을 자세히 볼 수 있다.

## 초보자용 사용 순서

1. WebUI 를 연다.
2. 왼쪽에서 채팅을 선택한다.
3. 먼저 `/status` 또는 `현재 상태를 알려줘` 를 입력한다.
4. 일정이 궁금하면 `/calendar` 또는 `/calendar today` 를 입력한다.
5. 모델을 바꾸고 싶으면 `/model list` 를 입력한다.

## 알아두면 좋은 점

- WebUI 는 단순 표시 화면이 아니라 Telegram 세션에도 답장을 보낼 수 있다.
- Telegram 채팅에서 보낸 slash command 결과도 WebUI 에 남도록 보강되어 있다.
- WebUI 가 이상해 보이면 브라우저 새로고침 전에 gateway health 와 bootstrap 을 확인하는 것이 좋다.
