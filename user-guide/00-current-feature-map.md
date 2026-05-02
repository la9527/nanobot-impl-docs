# 00. 현재 Nanobot 추가 기능 한눈에 보기

## 이 문서의 범위

이 문서는 현재 로컬 운영 환경에서 원래 Nanobot 기본 기능 위에 추가로 정리하거나 보강한 기능을 사용자 관점에서 요약한다.

복잡한 내부 구조보다 “무엇을 할 수 있고, 어디서 쓰는지”를 먼저 이해하는 데 목적이 있다.

## 전체 기능 요약

| 영역 | 할 수 있는 일 | 어디서 쓰나 |
| --- | --- | --- |
| WebUI Dashboard | 전체 상태, 막힌 작업, 승인 대기 요약 보기 | WebUI 첫 화면 |
| Telegram bridge | Telegram 대화를 WebUI 에서 보고 답장 | WebUI 사이드바의 Telegram 채팅 |
| Calendar automation | 일정 요약, 충돌 확인, 생성 요청, 승인/취소 | `/calendar` 명령 |
| Model target | 세션별 모델 선택과 해제 | `/model` 명령, WebUI target 표시 |
| smart-router | 요청 성격에 따라 local/mini/full 선택 | `/model smart-router` |
| Owner-aware status | 사용자 중심 작업 상태와 action result 표시 | Dashboard, thread status |
| Memory correction | 잘못된 기억 수정, 프로젝트 완료 표시 | 자연어 요청, WebUI quick action |
| Proactive heartbeat | 반복 작업과 quiet-hours 기반 알림 제어 | `HEARTBEAT.md`, WebUI/Telegram |
| Mail pilot | 메일 요약, 초안, 승인 기반 전송 흐름 준비 | `/mail`, 자연어 요청 |
| Live operations | gateway/api/WebUI bundle 확인과 재시작 | launchd scripts, health endpoint |

## 초보자에게 가장 중요한 5가지

1. WebUI 는 `http://127.0.0.1:8765/webui/` 에서 연다.
2. 왼쪽 사이드바에서 WebUI 채팅과 Telegram 채팅을 고를 수 있다.
3. 일정은 먼저 `/calendar` 로 상태를 확인한다.
4. 모델은 `/model` 과 `/model list` 로 확인한다.
5. 문제가 생기면 `/status` 와 gateway health 를 먼저 본다.

## 현재 가장 안정적으로 쓸 수 있는 기능

아래 기능은 focused test 와 live 확인이 된 상태다.

- WebUI 기본 접속과 bootstrap
- Telegram 세션에서 `/calendar` 결과 유지
- `/calendar` 상태/설정/pending 표시
- `/calendar cancel`, `/calendar deny` 안전 처리
- WebUI calendar conflict/action result 표시
- `/model` 기반 model target 확인과 선택
- WebUI production bundle build 및 launchd 재시작 흐름

## 파일럿 또는 주의가 필요한 기능

아래 기능은 사용할 수 있지만, 환경 설정과 실제 credential 상태에 따라 동작 범위가 달라진다.

- mail automation
- calendar 실제 생성 end-to-end
- calendar event 삭제/이동
- exec 로컬 명령 실행
- smart-router remote fallback
- proactive heartbeat 외부 채널 dispatch

## 추천 학습 순서

처음 보는 사용자는 아래 순서로 읽으면 된다.

1. [화면 구성과 WebUI 사용법](01-webui-overview.md)
2. [Telegram 연동 채팅 사용법](02-telegram-bridge.md)
3. [자주 쓰는 명령어](03-chat-commands.md)
4. [일정 자동화와 승인 흐름](04-calendar-automation.md)
5. [운영 확인과 문제 해결](09-troubleshooting.md)

## 핵심 개념 한 줄 정리

Nanobot 은 “로컬에서 돌아가는 개인비서”이고, WebUI 와 Telegram 을 함께 쓰면서 일정, 메일, 모델 선택, 반복 작업, 기억 관리를 점진적으로 자동화하는 구조다.
