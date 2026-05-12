---
applyTo: "source/nanobot/**"
description: "Use when editing Nanobot Python runtime code such as gateway, API, channels, agent loop, commands, plugins, and model target logic."
---

# Nanobot Runtime Instructions

이 instruction 은 Nanobot Python runtime 파일군에만 적용한다.

## 적용 범위

- `source/nanobot/**`

## 기본 원칙

- 현재 운영 runtime 은 source tree 를 editable install 로 참조하는 구조를 유지한다.
- source foreground 검증과 launchd live runtime 검증을 분리해서 다룬다.
- gateway, API, plugins, smart-router, model target 변경은 실제 live config 와 충돌하지 않게 유지한다.

## 작업 방식

- 경로 등록, 세션 메타데이터, provider 선택, plugin enablement 처럼 실제 동작을 결정하는 지점을 우선 수정한다.
- WebUI 연동 변경이 있으면 websocket/http surface 까지 함께 점검한다.
- 실운영 파일 `~/.nanobot/config.json`, `~/.nanobot/config.api.json`, `~/.nanobot/nanobot.env` 는 명시 요청 없이는 자동 수정하지 않는다.

## 검증 기준

- 가능하면 source tree 기준 pytest 또는 좁은 실행 검증을 먼저 수행한다.
- import 경로가 애매하면 `PYTHONPATH=/Volumes/ExtData/Nanobot/source` 를 명시한다.
- runtime surface 변경 후에는 관련 health 또는 route endpoint 를 실제 호출해 본다.

## 권장 명령

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/channels/test_websocket_http_routes.py -q
```

## 주의사항

- 수동 foreground 프로세스 결과를 launchd live 상태와 동일시하지 않는다.
- runtime 변경이 launchd 운영 경로에 영향을 주면 상태 메모 갱신도 고려한다.
