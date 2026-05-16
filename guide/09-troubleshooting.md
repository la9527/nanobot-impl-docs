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

중요한 점은 현재 운영 설정에서 인증 없이 호출하면 `401 Unauthorized` 가 나올 수 있다는 것이다. 이것만으로 곧바로 장애라고 판단하지 않는다.

확인 기준은 아래처럼 나눠서 본다.

- 인증 없이 `401 Unauthorized`: 현재 운영 설정에서는 정상일 수 있다.
- 인증 헤더 또는 실제 브라우저 세션 기준 `200 OK`: bootstrap 동작 정상

예시:

```bash
curl -fsS -H 'X-Nanobot-Auth: <bootstrap-secret>' http://127.0.0.1:8765/webui/bootstrap
```

## launchd 서비스 확인

```bash
launchctl list | grep nanobot
```

주요 서비스:

- `com.nanobot.gateway`
- `com.nanobot.api`
- `com.nanobot.local-model-lfm2`
- `com.nanobot.local-model-qwen36`

로컬 모델 상태를 더 직접 보려면 아래 명령을 우선 사용한다.

```bash
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh list
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh status all
```

메모리 보호를 위해 `start all`, `install all` 은 허용하지 않는다. 필요한 모델만 개별적으로 올린다.

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

## Telegram linked session 에서 `/status` 가 계속 진행중으로 남을 때

이 경우는 backend inline slash-command 저장 경로와 WebUI linked-session pending 정리 둘 다 확인해야 한다.

확인 순서는 아래와 같다.

1. `cd /Volumes/ExtData/Nanobot/source/webui && npm run build` 를 다시 실행했는지 확인한다.
2. `/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh` 와 `start-nanobot-services.sh` 로 launchd 서비스를 재기동한다.
3. `curl -fsS http://127.0.0.1:18790/health` 와 authenticated `GET /webui/bootstrap` 이 정상인지 확인한다.
4. WebUI 를 새로고침한 뒤 같은 Telegram linked thread 에서 `/status` 를 다시 실행한다.
5. 결과 bubble 은 보이는데 `연결된 외부 세션의 응답을 기다리고 있습니다.` 문구가 남으면 오래된 bundle 또는 stale browser state 가능성을 먼저 본다.

현재 기준으로 `/status` 는 linked Telegram session 에서도 session history 에 저장되도록 보강돼 있다.
따라서 Telegram 에는 답이 오지만 WebUI 에만 계속 pending 이면, 먼저 build/restart/reload 순서가 실제로 반영됐는지 확인하는 편이 빠르다.

## reasoning 줄이 안 보이거나 순서가 이상할 때

확인 포인트는 아래와 같다.

1. 설정에서 `추론 표시 수준` 이 꺼져 있지 않은지 확인한다.
2. 새로고침 후에도 `생각` 또는 `생각 중` 줄이 답변 앞에 남는지 본다.
3. 오래된 bundle 을 보고 있을 수 있으니 `npm run build` 이후 새로고침했는지 확인한다.
4. 필요하면 WebUI session history 와 live 브라우저 표시를 함께 본다.

현재 기준으로 observable reasoning 은 answer 앞의 안전한 요약 줄을 남기는 방향으로 정리돼 있다.

## action result 상세가 안 보일 때

최근 UI 는 action result 를 기본적으로 compact row 로 접어 보여준다.

따라서 아래 순서로 확인한다.

1. result row 자체를 클릭한다.
2. 또는 `세부 정보` 버튼을 눌러 detail 을 연다.
3. 새로고침 후에도 같은 result 가 남는지 확인한다.

calendar/mail 결과가 본문 위에서 사라진 것처럼 보여도, compact row 로만 접혀 있는 경우가 있다.

## Telegram 채팅에 삭제 버튼이 안 보일 때

이 경우는 현재 정책상 정상 동작일 수 있다.

- WebUI 내부 websocket 채팅은 삭제 affordance 를 노출할 수 있다.
- Telegram 같은 bridged 채팅은 WebUI 에서 삭제 UI 를 숨긴다.

즉, 삭제 버튼이 안 보인다고 바로 오류로 보지 않는다.

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

더 깊은 운영 배경은 [11. 심화 참고 문서 안내](11-advanced-reference.md) 와 `docs/operations/status-summary/` 날짜형 메모를 함께 보는 편이 좋다.
