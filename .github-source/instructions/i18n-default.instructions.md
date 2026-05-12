---
applyTo: "source/**"
description: "Use when editing Nanobot source code that adds or changes user-visible text, locale handling, backend response copy, session metadata copy, or WebUI labels. Treat i18n as a default requirement, not a later cleanup."
---

# Nanobot I18n Default Instructions

이 instruction 은 Nanobot source 작업 전반에 기본 적용한다.

## 적용 범위

- `source/**`

## 기본 원칙

- 사용자에게 보이는 문자열은 기본적으로 i18n 대상이라고 간주한다.
- UI 문구, backend 응답 문구, slash command 응답, 상태 배지, 오류 문구, session metadata 기반 안내 문구를 하드코딩한 채로 두지 않는다.
- i18n 은 마무리 단계의 후처리가 아니라 기능 구현의 일부로 함께 반영한다.

## 구현 규칙

- WebUI 에서 새 문자열을 추가하거나 바꾸면 `source/webui/src/i18n/locales/en/common.json` 과 `source/webui/src/i18n/locales/ko/common.json` 를 함께 갱신한다.
- Python runtime 에서 새 문자열을 추가하거나 바꾸면 `source/nanobot/i18n.py` catalog 를 함께 갱신하고, 호출부는 `_t(...)` 또는 `translate(...)` 경로를 사용한다.
- backend 가 생성한 metadata 가 WebUI 에서 그대로 보일 수 있으면, 그 metadata 에 들어가는 기본 문구도 i18n 대상에 포함한다.
- WebUI locale 문자열은 plain i18next interpolation 기준으로 작성한다. ICU/select 문법은 사용하지 않는다.
- 조건에 따라 달라지는 접미어, 상대시간 보조문구 같은 조합형 표현은 TypeScript 또는 Python 쪽에서 단순 문자열 조합으로 만든 뒤 locale placeholder 에 주입한다.

## 작업 방식

- 기능 변경 중 사용자 노출 텍스트가 생기면 같은 작업 묶음 안에서 locale key 와 번역을 같이 추가한다.
- 영어 fallback 이 우연히 보이는 상태를 허용하지 않는다. 최소한 `en` 과 `ko` 는 함께 맞춘다.
- 테스트가 문구나 상태 요약을 검증한다면 변경된 locale/표시 방식에 맞춰 테스트도 함께 갱신한다.
- 문구를 재사용할 수 있으면 비슷한 의미의 key 를 중복으로 늘리기보다 기존 key 구조를 우선 확인한다.

## 검증 기준

- 새 locale key 를 추가했으면 양쪽 catalog 에 모두 존재하는지 확인한다.
- WebUI 문구를 바꿨으면 관련 좁은 테스트와 `npm run build` 를 통해 반영 여부를 확인한다.
- runtime 문구를 바꿨으면 관련 pytest, route 호출, 또는 최소 import/compile 검증으로 키 누락과 오타를 먼저 확인한다.
- live 확인이 가능하면 한국어 UI 기준으로 영어 fallback 이 남지 않았는지 함께 확인한다.

## 주의사항

- 보이는 문자열만 번역하고 실제 의미를 바꾸지 않는다. 상태값과 내부 식별자는 기존 계약을 유지한다.
- locale 정리를 이유로 unrelated 문구까지 넓게 갈아엎지 않는다. 현재 작업 범위에 직접 연결된 문자열만 다룬다.