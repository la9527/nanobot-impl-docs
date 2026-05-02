# 09. 운영 확인과 문제 해결

## 먼저 확인할 것

Nanobot 이 이상해 보이면 아래 순서로 확인한다.

1. WebUI 가 열리는지 확인
2. gateway health 확인
3. bootstrap 응답 확인
4. launchd 서비스 상태 확인
5. 최근 변경 후 WebUI build 를 했는지 확인

## WebUI 주소

```text
http://127.0.0.1:8765/webui/
```

## health 확인

```bash
curl -fsS http://127.0.0.1:18790/health
```

정상이면 아래처럼 나온다.

```json
{"status":"ok"}
```

## WebUI bootstrap 확인

```bash
curl -i http://127.0.0.1:8765/webui/bootstrap
```

정상이면 `HTTP/1.1 200 OK` 와 token/model target 정보가 나온다.

## launchd 서비스 확인

```bash
launchctl list | grep nanobot
```

주요 서비스:

- `com.nanobot.gateway`
- `com.nanobot.api`
- `com.nanobot.llama-lfm2-server`

## 서비스 재시작

운영 반영은 launchd 스크립트 경로를 우선한다.

```bash
/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
```

재시작 후 다시 health 를 확인한다.

```bash
curl -fsS http://127.0.0.1:18790/health
```

## WebUI 변경 후 주의사항

WebUI source 를 수정했다면 production bundle 을 다시 만들어야 live 에 반영된다.

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm run build
```

`vite dev` 는 운영 반영 수단이 아니다.

## Telegram 에서는 보이는데 WebUI 에서 사라질 때

가능한 원인:

- WebUI history polling 이 이전 기록을 다시 읽음
- command 결과가 session file 에 저장되지 않음
- 브라우저가 오래된 bundle 을 보고 있음

현재 `/calendar` command 결과는 session 에 저장되도록 보강되어 있다.

확인 순서:

1. WebUI 를 새로고침한다.
2. 같은 Telegram 세션을 선택한다.
3. `/calendar` 를 입력한다.
4. 3~5초 뒤에도 결과가 남는지 본다.
5. 사라지면 session history 저장과 gateway 로그를 확인한다.

## 일정 생성이 막힐 때

가능한 원인:

- pending approval 이 남아 있음
- 기존 일정과 충돌함
- n8n calendar webhook 설정 문제
- webhook token 또는 base URL 문제

확인 명령:

```text
/calendar
/calendar status
/calendar config
/calendar cancel
```

## 모델 답변이 이상할 때

확인 명령:

```text
/model
/model list
/status
```

되돌리기:

```text
/model clear
```

## exec 명령이 안 될 때

가능한 원인:

- exec 기능 비활성화
- 승인 필요
- timeout
- workspace 밖 접근 제한
- 위험 명령 차단

안전한 테스트:

```text
exec로 pwd 명령을 한 번만 실행해줘
```

## 로그를 볼 때

launchd runtime 과 source foreground 검증은 구분한다.

- live 운영: launchd 서비스 기준
- 개발 검증: `/Volumes/ExtData/Nanobot/source` 기준
- 환경변수: `~/.nanobot/nanobot.env` 기준

## 커밋 대상에서 제외할 것

아래 같은 파일은 보통 커밋하지 않는다.

```text
webui/webui-dev.pid
```

## 가장 짧은 점검 루틴

```bash
launchctl list | grep nanobot
curl -fsS http://127.0.0.1:18790/health
curl -i http://127.0.0.1:8765/webui/bootstrap
```

이 세 가지가 정상이면 WebUI 와 gateway 기본 상태는 대체로 정상이다.
