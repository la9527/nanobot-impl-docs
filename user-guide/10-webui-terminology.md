# 10. WebUI 용어 기준

이 문서는 Nanobot WebUI i18n 작업에서 반복해서 쓰는 표현 기준을 짧게 고정해 두기 위한 용어집이다.

## 목적

- 화면에 보이는 표현과 사용자 가이드의 용어를 같은 기준으로 맞춘다.
- 영어 식별자와 사용자 노출 문구를 구분한다.
- 이후 `source/webui/src/i18n/locales/*/common.json` 수정 시 같은 의미의 표현이 흔들리지 않게 한다.

## 그대로 유지하는 기술 용어

- `nanobot`, `WebUI`, `Telegram`, `gateway`, `smart-router`
- 실제 명령, 경로, 파일명, 환경변수명
- 모델 공급자 고유명과 모델 ID

## 화면 용어 기준

- `Dashboard` 는 `대시보드` 로 쓴다.
- `Settings` 는 `설정` 으로 쓴다.
- `Details` 는 버튼과 섹션 모두 `세부 정보` 로 쓴다.
- `Task overview` 는 `작업 개요`, `Task details` 는 `작업 세부 정보` 로 쓴다.

## 대화 관련 기준

- 사이드바 목록, 새 채팅, 채팅 검색처럼 navigation 맥락은 `채팅` 으로 쓴다.
- 현재 열린 conversation 을 설명하는 본문 문구는 `대화` 로 쓴다.
- `thread` 의 일반 UI 문구는 `대화` 로 옮긴다.
- 단, 메일 도메인의 `mail thread` 처럼 도메인 용어일 때는 `스레드` 를 유지할 수 있다.
- `linked session` 은 `연결된 세션` 으로 쓴다.

## 상태와 작업 기준

- persona 의미의 `assistant` 대신 실제 기능을 드러내는 `작업`, `현재 작업`, `작업 상태` 를 우선한다.
- `approval pending` 은 `승인 대기`, `blocked task` 는 `막힌 작업` 으로 쓴다.
- `proactive update` 는 `선제 업데이트` 로 쓴다.

## 설정과 모델 기준

- 일반 설정 문구의 `provider` 는 `공급자` 로 쓴다.
- `model` 은 `모델`, `target` 은 `타깃` 으로 쓴다.
- `managed by runtime` 은 `런타임에서 관리됨` 으로 쓴다.
- `reasoning visibility` 는 `추론 표시 수준` 으로 쓴다.

## 사용자 정보와 메모리 기준

- `owner defaults` 는 `사용자 기본값` 으로 쓴다.
- `task metadata` 는 `작업 메타데이터` 로 쓴다.
- `memory tools` 는 `메모리 도구` 로 쓴다.

## 적용 메모

- 내부 key 이름이나 API 필드는 기존 계약을 유지하고, 사용자 노출 문자열만 이 기준으로 맞춘다.
- 한국어 문구에서 모든 영어를 없애는 것이 목표는 아니다. 제품명, 기술명, 실행 경로는 그대로 유지한다.