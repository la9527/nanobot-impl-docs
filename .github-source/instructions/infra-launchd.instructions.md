---
applyTo: "infra/**"
description: "Use when editing Nanobot infrastructure scripts, launchd helpers, environment loading, and operational restart or health-check workflows."
---

# Nanobot Infra And Launchd Instructions

이 instruction 은 Nanobot 인프라와 운영 스크립트 파일군에만 적용한다.

## 적용 범위

- `infra/**`

## 기본 원칙

- 현재 운영 서비스는 launchd 기반 경로를 기본값으로 유지한다.
- 재기동, 중지, health-check 스크립트는 `~/.nanobot/nanobot.env` 를 읽는 실제 운영 흐름과 맞아야 한다.
- gateway 와 API 는 운영 중 포트 충돌 없이 함께 관리되어야 한다.

## 작업 방식

- launchd wrapper, start/stop script, env sourcing, health-check 순서를 보존한다.
- 운영 스크립트 변경 시 수동 foreground 명령보다 launchd 경로 검증을 우선한다.
- WebUI 반영 작업이 들어가더라도 운영 반영은 infra restart 흐름으로 검증한다.

## 검증 기준

- 변경 후에는 가능하면 launchd 상태와 health endpoint 를 함께 확인한다.
- 필요한 경우 `launchctl list | grep nanobot`, `curl -fsS http://127.0.0.1:18790/health`, authenticated `GET /webui/bootstrap` 조합으로 확인한다.

## 권장 명령

```bash
/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
launchctl list | grep nanobot
curl -fsS http://127.0.0.1:18790/health
```

## 주의사항

- 직접 `nanobot gateway --config ...` 를 실행하면 live env 와 다른 상태로 뜰 수 있으므로 운영 반영 수단으로 우선 사용하지 않는다.
