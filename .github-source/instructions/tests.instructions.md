---
applyTo: "source/tests/**"
description: "Use when editing or adding Nanobot source-tree tests for channels, commands, config, plugins, and runtime behavior."
---

# Nanobot Test Instructions

이 instruction 은 Nanobot source test 파일군에만 적용한다.

## 적용 범위

- `source/tests/**`

## 기본 원칙

- 테스트는 touched behavior 를 직접 검증하는 좁은 범위를 우선한다.
- source tree 기준 검증과 live 운영 검증의 역할을 분리한다.
- smart-router, model target, websocket/http route, built-in command 변경은 회귀가 생기기 쉬우므로 관련 테스트를 유지한다.

## 작업 방식

- 새 동작을 넣었으면 그 동작을 가장 직접적으로 검증하는 테스트부터 추가한다.
- broad suite 대신 focused test 세트를 먼저 사용한다.
- flaky 하거나 운영 의존적인 검증은 단위 테스트와 분리한다.

## 검증 기준

- 변경한 테스트 파일 또는 인접 테스트를 우선 실행한다.
- 실패가 나면 같은 슬라이스에서 먼저 수정하고 다시 같은 테스트를 재실행한다.

## 권장 명령

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/channels/test_websocket_http_routes.py -q
```
