---
name: nanobot-status-memory
description: Create or update dated Nanobot operation summary memos and closely related guide docs when runtime, plugin, smart-router, launchd, editable-install, API, gateway, Telegram linked WebUI, or local llama.cpp behavior changes. Use when work needs to be captured as a daily status note or operational memory under /Volumes/ExtData/Nanobot/docs/operations/status-summary, and when nearby troubleshooting or advanced-reference docs should be kept in sync.
---

# Nanobot Status Memory

이 skill 은 현재 Nanobot 프로젝트 전체를 위한 GitHub Copilot project skill 이다.

적용 범위:

- `/Volumes/ExtData/Nanobot/docs`
- `/Volumes/ExtData/Nanobot/infra`
- `/Volumes/ExtData/Nanobot/source`
- `/Volumes/ExtData/Nanobot/.github/skills`

즉, source repo 내부 런타임 skill 이 아니라, Nanobot 작업 전체를 날짜형 운영 메모로 남길 때 쓰는 project-level skill 이다.

## 언제 사용할지

다음 중 하나라도 해당하면 이 skill 을 사용한다.

- smart-router, runtime plugin, provider, launchd, API, gateway, local llama.cpp 관련 변경이 있었을 때
- Telegram linked WebUI, bridged session, duplicate rendering, slash command pending, bootstrap/auth 같은 운영 이슈를 날짜형 메모로 남겨야 할 때
- editable install 여부, 실제 runtime import 경로, foreground 검증 경로를 정리해야 할 때
- `/Volumes/ExtData/Nanobot/docs/operations/status-summary/nanobot-status-summary-YYYY-MM-DD.md` 문서를 새로 만들거나 갱신할 때
- status summary 와 함께 `docs/guide/09-troubleshooting.md`, `docs/guide/11-advanced-reference.md` 같은 인접 가이드 문서도 같이 갱신해야 할 때
- 현재 상태, 완료된 작업, 남은 리스크, 다음 작업 후보를 운영 메모 형태로 남겨야 할 때

## 기본 원칙

- 문서는 한글 우선으로 쓴다.
- 기술 식별자, 경로, 명령, 환경변수는 영어 원문을 유지한다.
- 단순 changelog 가 아니라 운영 메모처럼 쓴다.
- 실제로 확인한 내용과 아직 정책 미정인 내용을 구분한다.
- 검증한 내용과 검증하지 못한 내용을 섞어 쓰지 않는다.

## 문서 위치와 파일명

- 위치: `/Volumes/ExtData/Nanobot/docs/operations/status-summary`
- 파일명: `nanobot-status-summary-YYYY-MM-DD.md`
- 기존 문서가 있으면 먼저 읽고 톤과 구조를 맞춘다.

현재 참고 대상 문서:

- `/Volumes/ExtData/Nanobot/docs/operations/status-summary/nanobot-status-summary-2026-05-09.md`
- `/Volumes/ExtData/Nanobot/docs/operations/status-summary/nanobot-status-summary-2026-05-10.md`
- `/Volumes/ExtData/Nanobot/docs/operations/status-summary/nanobot-status-summary-2026-05-13.md`

상태 메모와 함께 자주 같이 확인할 문서:

- `/Volumes/ExtData/Nanobot/docs/guide/09-troubleshooting.md`
- `/Volumes/ExtData/Nanobot/docs/guide/11-advanced-reference.md`
- `/Volumes/ExtData/Nanobot/docs/operations/status-summary/README.md`

## 권장 문서 구조

기본 구조는 아래 순서를 따른다.

1. 문서 목적
2. 현재 상태 요약
3. 이번에 반영한 주요 작업
4. 이번에 확인된 운영 포인트
5. 현재 남아 있는 제약과 리스크
6. 현재 기준 권장 운영 명령
7. 다음 작업 후보
8. 결론

필요하면 아래 항목을 중간에 추가한다.

- 현재 변경 파일 상태
- runtime 연결 방식
- 검증 결과
- 정책 미정 항목
- live browser 확인 메모
- source repo 와 docs repo commit 경계

## 반드시 구분해서 적을 것

### 1. 구현 완료 vs 운영 기본 적용

항상 아래를 분리해서 적는다.

- 코드 구현이 끝났는지
- 테스트가 통과했는지
- live foreground 검증이 끝났는지
- 실제 운영 기본 config 에 상시 적용됐는지

### 2. source-tree 검증 vs live runtime 검증

항상 아래를 분리해서 적는다.

- `source/.venv` 기준 foreground 테스트
- launchd 기준 live runtime
- `uv tool` editable install 여부
- 실제 import 경로와 entrypoint 경로

특히 source 디렉터리 밖에서 source venv 로 `python -m nanobot.cli.commands` 를 실행할 때는 `PYTHONPATH=/Volumes/ExtData/Nanobot/source` 를 함께 주는 편이 안전하다는 점을 운영 포인트로 남긴다.

### 3. 검증 범위 경계

예를 들어 smart-router 검증에서는 아래를 분리해서 적는다.

- local tier route 확인
- JSONL 로그 생성 확인
- `plugins list/status` 에서 enabled 확인
- `local -> mini -> full` fallback 실검증 여부
- real remote credential 사용 여부

### 4. source repo 와 docs repo 경계

항상 아래를 구분해서 적는다.

- `/Volumes/ExtData/Nanobot/source` 의 코드/테스트 변경인지
- `/Volumes/ExtData/Nanobot/docs` 의 운영 메모/가이드 변경인지
- 두 영역이 서로 다른 git repo 인지
- commit/push 가 한 repo 에만 반영됐는지, 둘 다 반영됐는지

루트 `/Volumes/ExtData/Nanobot` 는 git repo 가 아닐 수 있으므로, 상위 디렉터리에서 `git status` 를 기준처럼 적지 않는다.

## 상태 메모 작성 체크리스트

문서를 만들기 전 최소한 아래를 확인한다.

- 현재 launchd / gateway / api / llama 상태
- 실제 runtime import 경로 또는 editable install 상태
- 현재 브랜치와 커밋 반영 여부
- 관련 focused test 결과
- live 검증 성공 여부 또는 실패 이유
- 남아 있는 open question
- 관련 guide 문서 동기화 필요 여부
- source repo 와 docs repo commit/push 상태

## 명령 예시

```bash
git -C /Volumes/ExtData/Nanobot/source status --short --branch

git -C /Volumes/ExtData/Nanobot/docs status --short --branch

PYTHONPATH=/Volumes/ExtData/Nanobot/source \
  /Volumes/ExtData/Nanobot/source/.venv/bin/python -m nanobot.cli.commands \
  plugins list --config /Users/byoungyoungla/.nanobot/config.api.json

PYTHONPATH=/Volumes/ExtData/Nanobot/source \
  /Volumes/ExtData/Nanobot/source/.venv/bin/python -m nanobot.cli.commands \
  plugins status --config /Users/byoungyoungla/.nanobot/config.api.json
```

API foreground 검증 예시:

```bash
set -a && source /Users/byoungyoungla/.nanobot/nanobot.env && set +a
PYTHONPATH=/Volumes/ExtData/Nanobot/source \
  /Volumes/ExtData/Nanobot/source/.venv/bin/python -m nanobot.cli.commands serve \
  --config /Users/byoungyoungla/.nanobot/config.api.json \
  --host 127.0.0.1 \
  --port 8911
```

WebUI / launchd / linked Telegram 검증 예시:

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-shell.test.tsx
npm run build

/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh

launchctl list | grep nanobot
curl -fsS http://127.0.0.1:18790/health
curl -fsS -H 'X-Nanobot-Auth: <bootstrap-secret>' http://127.0.0.1:8765/webui/bootstrap
```

## 작성 스타일

- 현재 시점 기준으로 단정할 수 있는 것만 단정한다.
- 추정이나 정책 제안은 별도 소제목으로 분리한다.
- 같은 사실을 문단마다 반복하지 않는다.
- 다음 작업 후보는 우선순위가 드러나게 쓴다.

## 하지 말아야 할 것

- 단순 diff 나 commit 목록만 나열하지 않는다.
- 아직 검증하지 않은 내용을 "완료" 라고 쓰지 않는다.
- launchd live runtime 과 source foreground 결과를 동일시하지 않는다.
- temporary config, temporary log, foreground test artifact 를 문서의 기준 상태처럼 쓰지 않는다.

## 권장 후속 행동

새 상태 문서를 만들었다면 필요 시 아래도 같이 한다.

- 바로 이전 날짜 문서와 비교해 새로 달라진 점만 강조
- 현재 unresolved risk 를 다음 작업 후보와 연결
- smart-router / runtime plugin 관련 문서와 모순 없는지 확인
- Telegram linked WebUI 나 bootstrap/auth 관련 이슈라면 `docs/guide/09-troubleshooting.md` 와 `docs/guide/11-advanced-reference.md` 도 같이 갱신할지 확인
- commit/push 요청이 있었다면 source repo 와 docs repo 를 각각 따로 점검하고, 어느 repo 까지 반영했는지 문서와 답변에 분리해서 남긴다
