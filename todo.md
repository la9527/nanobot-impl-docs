# Nanobot TODO

이 문서는 현재 Nanobot 다음 작업을 우선순위와 구현 단위로 정리한 작업 기준 문서다.

## 현재 원칙

- 문서 구조 정리와 실제 구현 작업을 분리해서 본다.
- OpenClaw 참고 기능은 그대로 복제하지 않고 Nanobot 구조에 맞는 최소 구현 단위로 재해석한다.
- smart-router 와 충돌하지 않게 모델 선택 기능과 응답 상태 노출 기능을 설계한다.
- 각 작업은 config, runtime, slash command, 테스트, 문서를 함께 본다.

## 우선순위

1. `/model smart-router` 실제 채널 inbound 검증
2. [x] Telegram WebUI websocket push mirror 검증 및 마무리

우선순위 상태 메모:

- Telegram WebUI websocket push mirror 는 이미 live 동작 확인이 끝났으므로 별도 open item 으로 유지하지 않는다.
- 현재 남은 top-priority active item 은 `/model smart-router` 실제 채널 inbound 검증이다.

## 1. status-summary 문서 구조 정리

목표:
기존 날짜형 운영 메모를 `docs/status-summary/` 아래로 모으고, 앞으로도 같은 위치를 기준으로 쓰게 만든다.

이번에 바로 반영할 항목:

- [x] `docs/status-summary/` 디렉터리 생성
- [x] 기존 `nanobot-status-summary-*` 문서 이동
- [x] `.github/skills/nanobot-status-memory/SKILL.md` 경로 설명 업데이트

후속 확인 항목:

- [x] 다른 문서나 안내문에서 이전 경로를 직접 참조하는지 재검색
- [x] 상태 메모 작성 시 새 위치를 기준으로 예시를 맞추기

완료 기준:

- 새 상태 메모는 모두 `docs/status-summary/` 아래에 생성된다.
- skill 문서가 새 경로를 source of truth 로 설명한다.

## 2. 모델 선택 기능과 `/model` 명령

목표:
Nanobot 에서 local model, remote model, smart-router 를 모두 선택 가능한 target 으로 관리하고, 플러그인 형태로 provider/model 후보를 확장할 수 있게 만든다. 채팅 중에는 `/model` 명령으로 현재 모델 확인과 변경이 가능해야 한다.

배경 참고:

- 현재 Nanobot 는 단일 `agents.defaults.provider` + `agents.defaults.model` 중심이다.
- OpenClaw 는 모델 선택과 상태 표시에 대해 `/status`, `/usage`, model picker 계열 흐름을 이미 갖고 있다.
- Nanobot 쪽은 smart-router 를 하나의 선택 가능한 target 으로 포함하는 “사용자 명시 선택 모델 target” 계층이 먼저 필요하다.

세부 작업 항목:

- [x] 현재 config schema 에서 모델 선택 확장 지점 정의
  - 예: `agents.defaults.modelSelection.targets` 같은 구조 후보 검토
  - local / remote / alias / default / smart-router 여부를 표현할 최소 스키마 설계
- [x] provider/model 후보를 플러그인으로 등록하는 계약 정의
  - runtime plugin 이 provider/model catalog 를 기여할 수 있는지 검토
  - smart-router 와 특정 provider/model target 이 같은 선택 표면에 올라오도록 역할 분리
- [x] active model override 저장 위치 결정
  - 전역 기본값
  - session 단위 override
  - slash command 로 바꾼 값을 어디에 저장할지 결정
- [x] `/model` 명령 설계
  - 현재 모델 보기
  - 사용 가능한 모델 목록 보기
  - 모델 변경
  - smart-router target 선택
  - 변경 실패 시 안내 메시지 정의
- [x] Nanobot API 와 gateway 양쪽에서 현재 모델 해석 경로 통일
- [x] 테스트 추가
  - config parsing
  - slash command parsing
  - session override persistence
  - provider/model resolution
- [x] 문서 추가
  - 모델 등록 방법
  - `/model` 사용 예시
  - smart-router 와 특정 provider/model target 을 함께 쓸 때의 우선순위

OpenClaw 참고 지점:

- `/status`, `/usage` 명령 설명: [README]( /Volumes/ExtData/OpenClaw/README.md#L312 )
- 모델 선택 관련 명령 모듈 후보: [model-picker.ts]( /Volumes/ExtData/OpenClaw/src/commands/model-picker.ts ), [models.ts]( /Volumes/ExtData/OpenClaw/src/commands/models.ts )

선행 결정 포인트:

- Nanobot 는 OpenClaw 처럼 풍부한 catalog 기반으로 갈지, 우선 local/remote 2-tier alias 관리부터 갈지 결정 필요
- smart-router 가 활성 plugin 이더라도 `/model` 에서는 일반 target 중 하나처럼 선택 가능해야 함

완료 기준:

- 사용자 입장에서 `/model` 로 현재 모델 확인과 변경이 가능하다.
- 구현이 특정 provider 하드코딩이 아니라 plugin 확장 구조를 가진다.

현재 판정:

- [x] 완료
- [x] source 기준 문서 `source/docs/MODEL_TARGETS.md` 추가
- [x] runtime plugin contract 에 `RuntimePlugin.build_model_targets(context)` 반영
- [x] smart-router 가 plugin-contributed named target 으로 `/model` 목록에 노출됨
- [x] live config 에 `local-llm` 과 `smart-router` target 반영
- [x] source venv 기준 active target, session override, plugin status, health endpoint 재검증 완료

현재 상태 메모:

- [x] named target helper, config schema, session override, `/model`, Telegram/Discord 명령 노출, API/gateway/CLI 실행 경로 연결, 기본 테스트 반영 완료
- [x] plugin catalog 계약과 사용자 문서 반영 완료
- [x] `~/.nanobot/config.api.json`, `~/.nanobot/config.json` 에 `activeTarget=local-llm` + selectable `smart-router` 반영 완료
- [x] real remote provider 를 붙인 `local -> mini -> full` fallback chain 실운영 검증 완료

후속 작업 항목:

- [x] real remote credential 을 붙여 `smart-router` 의 `local -> mini -> full` fallback 실검증
- [x] example plugin 또는 sample config 기준의 provider/model target catalog 운영 예시 보강
- [ ] `/model smart-router` 를 실제 채널 세션에서 end-to-end 로 한 번 더 검증

## 3. 채팅 응답 상태 정보 노출과 slash command

목표:
각 응답 마지막에 context 사용량, token, model 등의 간단한 상태 정보를 노출할 수 있게 한다. 이 기능은 기본 slash command 로 제어 가능해야 한다.

배경 참고:

- OpenClaw 는 `/status` 와 `/usage off|tokens|full` 로 응답 footer 와 상태 노출을 제공한다.
- Nanobot 는 현재 응답 본문 외에 이런 compact status/footer 레이어가 없다.

세부 작업 항목:

- [x] Nanobot 응답 객체에서 노출 가능한 usage metadata 수집 경로 파악
  - input tokens
  - output tokens
  - total tokens
  - model
  - context window 또는 현재 context size
- [x] provider 별 usage 데이터 표준화 계층 추가 여부 검토
- [x] 응답 footer 포맷 설계
  - 짧은 한 줄 compact 형식
  - 채널별 렌더링 안전성 검토
- [x] slash command 설계
  - `/status` 로 현재 세션 상태 보기
  - `/usage off|tokens|full` 처럼 footer 레벨 제어
  - 필요 시 `/model` 과 역할 경계 정리
- [x] 세션별 설정 저장 방식 정의
- [x] CLI, gateway, API surface 별 적용 범위 정의
- [x] 테스트 추가
  - footer on/off
  - provider usage 없음 fallback
  - 채널 포맷 안전성
- [x] 문서 추가
  - 사용자용 slash command 설명
  - 예시 출력 형식

OpenClaw 참고 지점:

- 채팅 명령 설명: [README]( /Volumes/ExtData/OpenClaw/README.md#L312 )
- usage 관련 타입/플러그인 seam 후보: [types.ts]( /Volumes/ExtData/OpenClaw/src/plugins/types.ts#L1001 )

선행 결정 포인트:

- Nanobot 에서 usage 메타가 없는 provider 는 어떤 fallback 문구를 보여줄지 결정 필요
- 상태 footer 를 기본 on 으로 둘지, slash command 로 opt-in 할지 결정 필요

완료 기준:

- 사용자가 slash command 로 상태 노출 수준을 제어할 수 있다.
- 최소한 model 과 token usage 는 응답 후 쉽게 확인 가능하다.

현재 판정:

- [x] 완료
- [x] `/usage off|tokens|full` built-in command 추가
- [x] `/status` 에 active target / reply footer mode 노출 추가
- [x] non-streaming chat reply 와 CLI 응답에 compact footer 추가
- [x] OpenAI-compatible API 는 본문 footer 대신 JSON `usage` 필드 유지
- [x] focused test `44 passed` 재검증 완료
- [x] streaming 채널에서도 `/usage tokens|full` footer 자동 부착 지원

현재 상태 메모:

- [x] session metadata key `response_footer_mode` 로 per-session footer 설정 저장
- [x] `source/nanobot/response_status.py` 에 usage normalization / footer formatting helper 추가
- [x] Discord slash command 에 `/usage` 노출 추가
- [x] Telegram bot command menu 에 `/usage` 노출 추가
- [x] live runtime 기준 `/model smart-router` -> `/status` -> `/usage tokens` -> 일반 응답 footer 경로 재검증 완료
- [ ] 실제 Telegram/Discord 등 외부 inbound 채널에서 `/model smart-router` 를 직접 한 번 더 검증하는 작업은 남아 있음

## 4. Telegram WebUI websocket push mirror

목표:
Telegram 에서 새 user turn 이 들어오거나 assistant 응답이 생성될 때, polling 을 기다리지 않고 현재 열려 있는 WebUI thread 에 즉시 반영되게 만든다.

배경 참고:

- 현재 WebUI 는 Telegram 세션을 읽고 답장할 수 있지만, history polling 없이는 외부 Telegram 입력이 즉시 보이지 않는다.
- session key 기반 websocket mirror 를 붙이면 현재 선택된 Telegram thread 를 거의 실시간으로 반영할 수 있다.

세부 작업 항목:

- [x] Telegram session 을 websocket subscription key 로 재사용하는 live mirror 경로 추가
- [x] remote Telegram user turn 을 WebUI 전용 websocket mirror event 로 전달
- [x] Telegram assistant response / delta 를 WebUI mirror stream 으로 전달
- [x] WebUI thread 가 Telegram session key 를 live subscription key 로 사용하도록 연결
- [x] focused backend / frontend 테스트 추가
- [ ] 실제 live Telegram inbound 한 턴을 보내서 WebUI 즉시 반영 여부를 재검증

완료 기준:

- Telegram 에서 입력한 user turn 이 manual refresh 없이 현재 열려 있는 WebUI thread 에 나타난다.
- 같은 Telegram 세션의 assistant 응답도 websocket mirror 로 즉시 반영된다.

## 권장 진행 순서

1. 실제 채널 inbound 에서 `/model smart-router` 한 번 더 검증
2. live Telegram inbound 한 턴으로 websocket push mirror 최종 확인

## Execution Backlog Sync

개인비서 제품 레이어 관련 작업은 이제 아래 execution backlog 순서를 기준으로 본다.

### 즉시 착수 우선순위

1. `docs/execution-backlog/01-owner-aware-control-surface-phase1.md`
2. `docs/execution-backlog/02-owner-memory-and-task-backbone-phase1.md`
3. `docs/execution-backlog/03-gmail-readonly-draft-phase1.md`
4. `docs/execution-backlog/04-gmail-send-approval-phase1.md`
5. `docs/execution-backlog/05-calendar-read-conflict-create-phase1.md`
6. `docs/execution-backlog/06-proactive-briefing-and-quiet-hours-phase1.md`

### 현재 상태 메모

- 1순위 control surface 의 기존 기반인 WebUI status / approval / continuity phase-1 은 완료 상태다.
- 다음 코드 착수 단위는 `owner-aware aggregate` 를 WebUI 와 `/api/sessions` 기준 payload 에 연결하는 작업이다.
- owner memory / personal task backbone 은 conceptual draft 는 완료됐고 runtime metadata 경계 정의가 다음 단계다.
- Gmail/Calendar/proactive 는 conceptual contract 는 준비됐고 runtime 구현은 아직 시작 전이다.
- [ ] WebUI 일정 action result strip 이 history 하단에 고정처럼 남는지 live 구조를 다시 점검하고, 기본은 한 줄 compact summary + `세부 정보` 클릭 시 상세 노출 형태로 정리

### backlog 와 기존 방향 A 문서의 관계

- `docs/assistant-direction-a/workplans/05-1-*`, `05-2-*` 는 완료된 기반 문서로 유지한다.
- `docs/execution-backlog/01-*` 부터는 그 위에 올라가는 다음 제품 레이어 실행 순서로 본다.
- 즉, 방향 A workplan 은 기반 구축, execution backlog 는 다음 코드 작업 순서다.

## 5. 개인비서 Execution Backlog Active Work Items

목표:
execution backlog 문서를 실제 착수 가능한 TODO 체크리스트로 내려, 현재부터 어떤 순서로 구현할지 한 문서에서 바로 확인할 수 있게 한다.

운영 원칙:

- 아래 항목은 `docs/execution-backlog/*.md` 의 실행 체크리스트를 `todo.md` 용으로 축약한 active queue 다.
- 기존 1~4번 항목은 live 운영 확인/마무리 작업으로 유지한다.
- 아래 5.x 항목은 개인비서 제품 레이어 구현용 중기 백로그다.
- 구현 착수 시에는 한 번에 하나의 priority slice 만 `in-progress` 로 본다.

### 5.1 owner-aware control surface aggregate

참고 문서:

- `docs/execution-backlog/01-owner-aware-control-surface-phase1.md`

현재 상태:

- [x] 진행 중

이번 phase-1 목표:

- WebUI 에 same-owner 기준 assistant summary block 을 추가한다.
- `/api/sessions` 또는 session-derived payload 기준으로 active / approval / blocked / linked 상태를 aggregate 한다.
- 이후 Gmail/Calendar action summary 가 꽂힐 UI 자리를 확보한다.

세부 작업 항목:

 [x] session summary aggregate helper 위치 결정
 [x] `approval_summary` 기반 pending count 계산 경로 정의
 [x] linked external session count 계산 경로 정의
- [x] recent completed / blocked summary 최소 shape 정의
 [x] `/api/sessions` payload 만으로 충분한지 판단
- [ ] 부족하면 route serializer 또는 aggregate payload 확장
 [x] `source/webui/src/components/thread/ThreadShell.tsx` 에 assistant summary block 초안 추가
- [x] `source/webui/src/components/ChatList.tsx` row wording 과 thread summary wording 통일
 [x] `source/webui/src/tests/thread-shell.test.tsx` aggregate rendering test 추가
- [x] `source/webui/src/tests/chat-list.test.tsx` badge alignment test 추가
- [x] `source/tests/channels/test_websocket_http_routes.py` aggregate serialization test 추가
 [x] `source/webui` 기준 `npm run build` 검증
- [x] live WebUI 에서 approval pending + linked external session 동시 표시 확인

이번 작업 반영:

- `App -> ThreadShell` 로 전체 session list 를 전달해 same-owner 기준 요약 계산 경로를 프론트에 우선 배치했다.
- 1차 assistant summary block 은 active now / approval pending / blocked / linked sessions / latest external activity / recent completion / next step hint 까지 표시한다.
- `ChatList` 와 `ThreadShell` 이 공통 session metadata helper 를 사용하도록 맞춰 channel badge 와 approval wording drift 를 줄였다.
- `/api/sessions` 응답만으로 same-owner aggregate 를 계산할 수 있다는 점을 route regression test 로 고정했다.
- launchd live runtime 이 shell `OPENAI_API_KEY` 를 자동 상속하지 않아 `~/.nanobot/nanobot.env` 에 실제 `OPENAI_API_KEY` 를 채운 뒤 gateway/api 를 정상 복구했다.
- live WebUI 에서 session refresh 이후 `Assistant summary` 의 `1 approval pending • 4 linked sessions` 와 sidebar `Approval pending` badge 동시 표시를 확인했고, 검증용 Telegram session metadata 주입은 즉시 원복했다.
- blocked 는 `pending_user_turn` 또는 `runtime_checkpoint.phase` 기반으로, recent completed 는 same-owner 최신 non-blocked/non-approval session 기준으로 우선 파생 계산한다.

완료 기준:

- 사용자가 WebUI에서 현재 active / approval / blocked / linked 상태를 same-owner 기준으로 한눈에 본다.

### 5.2 owner memory + personal task backbone

참고 문서:

- `docs/execution-backlog/02-owner-memory-and-task-backbone-phase1.md`

현재 상태:

- [x] 완료

이번 phase-1 목표:

- owner profile 최소 필드와 active task summary metadata 를 runtime/product layer 에서 읽을 수 있게 정리한다.
- `USER.md`, `memory/MEMORY.md`, session/task metadata, raw history 경계를 phase-1 수준에서 닫는다.

세부 작업 항목:

- [x] owner profile phase-1 최소 필드 고정
- [x] active task summary metadata 최소 shape 정의
- [x] `waiting-approval` / `blocked` / `scheduled` / `recent-completed` 분류 경계 정리
- [x] owner-aware aggregate read path 를 session metadata 기반으로 정의
- [x] 별도 owner/task store 없이 가능한 범위 문서화
- [x] memory correction vocabulary 초안 정리
- [x] metadata serialization / API exposure / WebUI consumption 검증 항목 정리

이번 작업 반영:

- `normalized_session_metadata()` 에서 `task_summary` 최소 shape 를 파생 생성하도록 연결했다.
- phase-1 최소 `task_summary` 는 `task_id`, `canonical_owner_id`, `title`, `status`, `origin_channel`, `origin_session_key`, `updated_at`, `next_step_hint` 를 포함한다.
- `approval_summary`, `pending_user_turn`, `runtime_checkpoint` 기준으로 `waiting-approval` / `blocked` / `completed` 를 우선 파생한다.
- `/api/sessions` route regression test 로 `task_summary` 노출 계약을 고정했고, WebUI owner-aware summary 는 `task_summary.status` / `task_summary.next_step_hint` 를 우선 읽도록 연결했다.
- `normalized_session_metadata()` 에 phase-1 `owner_profile` / `memory_boundary` 파생값을 추가해 `canonical_owner_id`, `preferred_language`, `timezone`, `response_tone`, `response_length` 와 저장 경계(`USER.md`, `memory/MEMORY.md`, `session.metadata`, `memory/history.jsonl`)를 함께 노출하도록 정리했다.
- `SessionManager` 가 workspace `USER.md` snapshot 을 함께 넘기도록 연결해서 `owner_profile` 기본값이 template placeholder 가 아니라 실제 `USER.md` 값(`Timezone`, `Language`, `Communication Style`, `Response Length`)을 우선 읽도록 정리했다.
- phase-1 `memory_correction` metadata contract 를 추가해 `기억해`, `잊어`, `이건 기본 선호가 아님`, `이 프로젝트는 끝났어` 를 action code와 저장 대상(`USER.md`, `memory/MEMORY.md`, `session.metadata`) 기준으로 노출하도록 고정했다.
- WebUI `ThreadShell` 에 `Current task` inline block 을 추가해 현재 session 의 `task_summary` title/status/origin/next_step_hint 와 owner defaults 를 바로 확인할 수 있게 했다.
- memory correction chip draft 를 구조화된 입력(`내용:` / `수정할 기본 선호:` / `프로젝트 메모:`)으로 바꿔서 send 직전 편집 여지를 남기고, websocket channel 에서 해당 draft 를 deterministic 하게 파싱해 `USER.md` 또는 `memory/MEMORY.md` 에 즉시 반영하도록 연결했다.
- direct websocket thread 는 local assistant confirmation message 를 바로 반환하고, linked session thread 는 기존 poll 경로로 assistant confirmation 이 보이도록 session file 에 user/assistant turn 을 함께 저장한다.
- memory correction reply 가 끝나면 `ThreadShell` 이 session list refresh 를 한 번 더 실행하도록 연결해서 `USER.md` 변경 직후 owner defaults 줄이 같은 thread 안에서 바로 갱신되게 했다.
- `잊어` 는 exact-match 만 지우던 규칙에서 벗어나 normalized partial match 도 허용하도록 보강해 `## Special Instructions` 나 `프로젝트 종료:` bullet 을 짧은 표현으로도 제거할 수 있게 했다.

완료 기준:

- Gmail/Calendar/proactive 가 읽을 최소 owner/task backbone 이 정의된다.

### 5.3~5.5 및 Kakao 선행 환경 준비

목표:

- Gmail, Calendar, Kakao 후속 구현에 들어가기 전에 이미 AI_Assistant 에서 검증된 credential, webhook, 공개 ingress 구성을 Nanobot 작업 기준 문서로 재사용 가능하게 정리한다.

source of truth 참고:

- `/Volumes/ExtData/AI_Project/AI_Assistant/.env`
- `/Volumes/ExtData/AI_Project/AI_Assistant/docs/gmail-integration.md`
- `/Volumes/ExtData/AI_Project/AI_Assistant/docs/google-calendar-integration.md`
- `/Volumes/ExtData/AI_Project/AI_Assistant/docs/kakao-integration.md`
- `/Volumes/ExtData/AI_Project/AI_Assistant/docs/architecture.md`

공통 준비 원칙:

- AI_Assistant 디렉터리의 `.env` 와 integration 문서는 reference-only source of truth 로 본다. 현재 작업 기준에서는 AI_Assistant stack 이 실행 중이라고 가정하지 않고, live 검증은 Nanobot local runtime 과 별도 권한 확인으로 처리한다.
- 실제 secret value 는 Nanobot 저장소 문서나 tracked file 에 복사하지 않는다. 값은 `~/.nanobot/nanobot.env` 또는 launchd runtime 이 읽는 로컬 비추적 secret source 에만 넣고, `todo.md` 에는 변수명과 접근 권한만 남긴다.
- Gmail/Calendar phase-1 은 AI_Assistant 에서 이미 연결된 `n8n` + Google OAuth 자산을 우선 참고한다. 즉, 새 provider 설계 전에 `http://127.0.0.1:5678` 편집기 접근, 동일 Google 계정, OAuth redirect URI, live workflow 존재 여부를 먼저 확인한다.
- Kakao 는 개인 계정 자동화가 아니라 AI_Assistant 와 동일하게 공식 채널 + OpenBuilder + 공식 webhook 기준으로만 진행한다.
- 외부 LLM fallback 또는 structured extraction 품질이 필요하면 AI_Assistant `.env` 의 `OPENAI_API_KEY`, `EXTERNAL_LLM_*` 설정명을 참조하되, 실제 값은 Nanobot local env 에만 주입한다.

공통 확인 항목:

- [x] AI_Assistant `.env` 기준으로 Nanobot 에서 재사용할 변수명 매핑 초안 작성
  - `N8N_BASE_URL`, `N8N_EDITOR_BASE_URL`, `N8N_GMAIL_WEBHOOK_PATH`, `N8N_CALENDAR_CREATE_WEBHOOK_PATH`, `N8N_CALENDAR_UPDATE_WEBHOOK_PATH`, `N8N_WEBHOOK_TOKEN`
  - `KAKAO_PUBLIC_BASE_URL`, `CLOUDFLARE_TUNNEL_TOKEN`, `TAILSCALE_HOSTNAME`
  - 필요 시 `OPENAI_API_KEY`, `EXTERNAL_LLM_*`, `LOCAL_LLM_*`
- 2026-05-01 확인 메모: Nanobot local env 에서는 현재 `OPENAI_API_KEY`, `LOCAL_LLM_BASE_URL` 만 확인됐고, `N8N_*`, `KAKAO_PUBLIC_BASE_URL`, `CLOUDFLARE_TUNNEL_TOKEN`, `TAILSCALE_HOSTNAME`, `EXTERNAL_LLM_*` 매핑은 아직 비어 있다.
- [x] `n8n` 편집기 `http://127.0.0.1:5678` 접근 가능 여부와 Google credential 현황 확인
- 2026-05-01 확인 메모: `http://127.0.0.1:5678` 자체는 응답하고, 이를 AI_Assistant live stack 으로 간주하지 않는 전제 아래 Docker `n8n` 컨테이너 metadata 기준으로 Google credential 존재 여부까지 확인했다.
- 2026-05-01 추가 확인 메모: Docker `n8n` 컨테이너의 PostgreSQL metadata 에서 `Gmail account`, `Google Calendar account` credential 존재를 확인했고, refresh token 재발급 후 Google API profile 조회 기준 두 credential 모두 동일한 Google 계정으로 연결돼 있음을 확인했다.
- [x] Gmail OAuth2 API credential, Google Calendar OAuth2 API credential 이 실제 Google 계정으로 연결돼 있는지 확인
- [x] Kakao 공식 채널 관리자센터, OpenBuilder, Cloudflare Tunnel 을 수정할 수 있는 운영 권한 보유 여부 확인
- 2026-05-01 확인 메모: AI_Assistant 참고용 `KAKAO_PUBLIC_BASE_URL` 공개 health endpoint 는 현재 `530` 응답으로 정상 상태가 아니었고, 로컬 docker runtime 에서 `cloudflared` 컨테이너는 보이지 않았다.
- 2026-05-01 추가 확인 메모: OpenBuilder `https://i.kakao.com/` 로그인 후 `AI비서` 챗봇이 보이고, 연결 채널 `@AI 개인비서`, 권한 `마스터`, `AI 챗봇 ON` 상태까지 확인했다.
- 2026-05-01 추가 확인 메모: Kakao 비즈니스 파트너센터 `https://business.kakao.com/dashboard/` 에서 `AI 개인비서` 채널 대시보드와 파트너센터 메뉴 접근도 확인했다.
- 2026-05-01 추가 확인 메모: AI_Assistant compose 의 `edge` profile 로 `cloudflared` 를 실제 기동했고, 공개 host `https://ai-assistant-kakao.la9527.cloud/assistant/api/health` 가 `200 OK` 로 복구되는 것까지 확인했다. 따라서 Cloudflare Tunnel 운영 가능 상태도 확인됐다.
- [x] Nanobot 구현 시작 전 secret 주입 위치를 `~/.nanobot/nanobot.env` 또는 별도 비추적 env 파일로 고정
- 2026-05-01 확인 메모: `~/.nanobot/nanobot.env` 파일이 이미 존재하므로, 후속 Gmail/Calendar/Kakao secret 도 같은 비추적 경로에만 추가하는 기준으로 유지한다.

완료 기준:

- 다음 Gmail/Calendar/Kakao 구현이 AI_Assistant 쪽 기존 credential 과 접근 권한을 참조해 바로 시작 가능한 상태가 된다.

### 5.3 Gmail pilot read-only + draft flow

참고 문서:

- `docs/execution-backlog/03-gmail-readonly-draft-phase1.md`
- `docs/assistant-direction-a/workplans/05-3-gmail-pilot-readonly-draft-phase1-plan.md`
- `/Volumes/ExtData/AI_Project/AI_Assistant/docs/gmail-integration.md`
- `/Volumes/ExtData/AI_Project/AI_Assistant/.env`

현재 상태:

- [x] 완료

이번 phase-1 목표:

- mailbox domain 에 대해 `조회 -> 요약 -> 초안` 까지를 assistant action 으로 구현한다.

세부 작업 항목:

- [x] AI_Assistant Gmail/n8n 환경 기준 재사용 범위 확정
  - `N8N_GMAIL_WEBHOOK_PATH`, `N8N_BASE_URL`, `N8N_EDITOR_BASE_URL`, `N8N_WEBHOOK_TOKEN`
  - 2026-05-01 구현 메모: phase-1 executor 는 MCP 가 아니라 direct n8n webhook adapter 로 먼저 고정하고, Nanobot runtime 에서는 `N8N_GMAIL_WEBHOOK_PATH` + 기본 `webhook/assistant-gmail-thread` + 기본 `webhook/assistant-gmail-draft` 조합으로 호출하는 `N8NGmailAutomationClient` 를 추가했다.
- [x] Google Cloud / n8n 접근 권한 확인
  - `Gmail OAuth2 API` credential 접근 가능 여부 확인
  - redirect URI `http://127.0.0.1:5678/rest/oauth2-credential/callback` 와 접속 host 를 `127.0.0.1` 로 통일할지 다시 확인
- [x] `mail.list_important_threads` contract 초안 구현
- [x] `mail.summarize_threads` contract 초안 구현
- [x] `mail.create_draft` contract 초안 구현
- [x] direct adapter vs MCP adapter seam 결정
- [x] normalized thread summary shape 정의
- [x] normalized draft preview shape 정의
- [x] auth needed / mailbox unavailable / thread not found failure shape 정의
- [x] 실제 user-triggered mail action path 연결
- [x] WebUI mail status 최소 연결
- [x] WebUI mail detail card 연결
- [x] backend contract / failure normalization test 추가
- [x] 필요 시 frontend status exposure test 추가

- 이번 작업 반영:
  - `source/nanobot/automation_results.py` 에 `MailDraftPreview`, `MailCreateDraftDetails`, `MailCreateDraftResult` 를 추가해 5.3 draft preview contract 를 action result vocabulary 안에 포함시켰다.
  - `source/nanobot/automation/mail.py` 에 direct n8n webhook 기반 `N8NGmailAutomationClient` 와 `MailDraftRequest` 를 추가해 `list_important_threads`, `summarize_threads`, `create_draft` 의 phase-1 runtime seam 을 만들었다.
  - summary/thread/draft 응답을 thread 친화적인 normalized shape 로 변환하고, HTTP auth / executor / not found 계열 실패를 action failure code 로 좁혀서 반환하도록 정리했다.
  - focused pytest `tests/test_automation_results.py tests/test_mail_automation.py -q` 기준 `8 passed` 를 확인했다.
  - `source/nanobot/session/manager.py` 에 latest `action_result` metadata 저장 helper 를 추가하고, `source/nanobot/session/continuity.py` 의 task summary 파생이 이 값을 읽어 현재 task title/status/next step 을 유지하도록 연결했다.
  - `source/nanobot/automation/mail.py` 에 `MailAutomationSessionRunner` 를 추가해 mail client 결과가 실제 Nanobot session file 의 assistant turn + `action_result` metadata 로 저장되는 최소 session/action flow 를 만들었다.
  - `source/nanobot/command/builtin.py` 에 `/mail` built-in command 를 추가해 `/mail list`, `/mail thread`, `/mail draft --to ... --subject ... --body ...` 형태로 실제 user-triggered path 에서 `MailAutomationSessionRunner` 를 호출하도록 연결했다.
  - `source/webui/src/components/thread/ThreadShell.tsx` 와 관련 type/helper 에 `action_result` metadata surface 를 추가해 draft ready 같은 mail action 결과가 Thread status block 으로 바로 보이고, draft preview / thread summary detail card 까지 같은 thread 안에서 확인되게 연결했다.
  - focused 검증으로 `PYTHONPATH=$PWD ./.venv/bin/pytest tests/test_mail_automation.py tests/agent/test_session_manager_history.py tests/test_automation_results.py tests/command/test_builtin_mail.py -q` 기준 `35 passed`, `npm test -- src/tests/thread-shell.test.tsx` 기준 `20 passed` 를 확인했다.

완료 기준:

- 중요한 메일 요약과 답장 초안 생성이 assistant action 으로 노출된다.

### 5.4 Gmail send approval flow

참고 문서:

- `docs/execution-backlog/04-gmail-send-approval-phase1.md`
- `docs/assistant-direction-a/workplans/05-4-gmail-send-approval-phase1-plan.md`
- `/Volumes/ExtData/AI_Project/AI_Assistant/docs/gmail-integration.md`
- `/Volumes/ExtData/AI_Project/AI_Assistant/.env`

현재 상태:

- [x] 완료

이번 phase-1 목표:

- `mail.send_message` 를 approval 기반 action 으로 닫고, WebUI control surface 와 연결한다.

세부 작업 항목:

- [x] AI_Assistant 메일 승인 경로 재사용 범위 확정
  - draft/send/reply workflow 와 approval ticket 경계를 Nanobot 에서 그대로 따를지, adapter 경계만 바꿀지 결정
  - `N8N_GMAIL_WEBHOOK_PATH` 외 send/reply 전용 webhook 또는 workflow naming 을 정리
- [x] 실제 발송 권한과 안전장치 확인
  - 같은 Google 계정으로 draft, send, reply 가 가능한 `Gmail OAuth2 API` credential 유지 여부 확인
  - Nanobot local env 에 필요한 secret key 는 변수명만 기록하고 값은 비추적 secret source 에만 주입
- [x] `mail.send_message` contract 초안 구현
- [x] approval policy shape 정의
- [x] approval 요청 문구 표준화
- [x] send success / denied / failure summary 정규화
- [x] thread inline / sidebar / owner-aware aggregate 에 approval visibility 연결
- [x] approve / deny / timeout 처리 규칙 정리
- [x] backend approval route / summary test 추가
- [x] frontend approval visibility test 추가

- 이번 작업 반영:
  - `source/nanobot/automation/mail.py` 에 `N8N_GMAIL_SEND_WEBHOOK_PATH` 기본값(`webhook/assistant-gmail-send`)과 `mail.send_message` contract 를 추가하고, `MailSendRequest` / `MailSendMessageResult` 를 direct n8n webhook adapter 위에 올렸다.
  - `MailAutomationSessionRunner` 가 draft 생성 시 최신 draft payload 를 세션 metadata 에 보존하고, `/mail send` 로 pending approval 을 만들고 `/mail approve`, `/mail deny` 로 이를 해제하도록 session-backed approval state 를 추가했다.
  - pending send approval 은 `approval_summary` + `action_result(status=waiting_approval)` 조합으로 저장해 기존 owner-aware aggregate 와 current task/status block/UI 흐름을 그대로 재사용하도록 맞췄다.
  - `source/nanobot/command/builtin.py` 에 `/mail send`, `/mail approve`, `/mail deny` 를 추가해 phase-1 send approval flow 를 실제 user-triggered command path 로 노출했다.
  - `source/webui/src/components/thread/ThreadShell.tsx` 는 waiting approval 상태의 mail result 에 `Approval pending` badge 를 노출하도록 보강했다.
  - live `~/.nanobot/nanobot.env` 에 `N8N_BASE_URL=http://127.0.0.1:5678`, `N8N_GMAIL_WEBHOOK_PATH`, `N8N_GMAIL_THREAD_WEBHOOK_PATH`, `N8N_GMAIL_DRAFT_WEBHOOK_PATH`, `N8N_GMAIL_SEND_WEBHOOK_PATH` 를 반영하고 launchd gateway/api 를 재기동했다.
  - direct webhook 기준 `assistant-gmail-draft` 가 실제로 초안을 만들고 `draft_id` 를 반환하는 것을 확인했고, `assistant-gmail-send` 도 실제 발송과 `message_id=19de24688848fe49` 반환을 확인했다.
  - live WebUI 기준 같은 websocket session 에서 `/mail draft --to la9527@daum.net ...` → `/mail send` → `/mail approve` 를 순서대로 실행해 draft 생성, approval pending, 실제 발송 완료까지 end-to-end 로 확인했다.
  - live session file `~/.nanobot/workspace/sessions/websocket_76a4e39c-4ca4-4a6d-ac41-9e884eafff42.jsonl` 기준 `action_result.action=send_message`, `details.message_id=19de2466fe1b86fa`, `references.draft_id=r8715010086581269631` 가 저장되는 것까지 확인했다.
    - focused 검증으로 `PYTHONPATH=$PWD ./.venv/bin/pytest tests/test_automation_results.py tests/test_mail_automation.py tests/command/test_builtin_mail.py tests/agent/test_session_manager_history.py -q` 기준 `41 passed`, `npm test -- src/tests/thread-shell.test.tsx` 기준 `21 passed`, `npm run build` 통과를 확인했다.

완료 기준:

- 실제 메일 발송은 approval 이후에만 가능하고, 대기/결과가 WebUI 에서 이해 가능하다.

### 5.5 Calendar read/conflict/create flow

참고 문서:

- `docs/execution-backlog/05-calendar-read-conflict-create-phase1.md`
- `/Volumes/ExtData/AI_Project/AI_Assistant/docs/google-calendar-integration.md`
- `/Volumes/ExtData/AI_Project/AI_Assistant/.env`

현재 상태:

- [x] 완료

이번 phase-1 목표:

- `조회 -> 충돌 탐지 -> 생성` 까지의 최소 calendar assistant workflow 를 연다.

세부 작업 항목:

- [x] AI_Assistant Calendar/n8n 환경 기준 재사용 범위 확정
  - `N8N_CALENDAR_CREATE_WEBHOOK_PATH`, `N8N_CALENDAR_UPDATE_WEBHOOK_PATH`, 필요 시 delete workflow naming 까지 mapping 정리
  - `assistant-calendar-create`, `assistant-calendar-update`, `assistant-calendar-delete` workflow 또는 동등 경로를 Nanobot 에서 어떻게 호출할지 결정
- [x] Google Calendar 접근 권한 확인
  - `Google Calendar OAuth2 API` credential 접근 가능 여부 확인
  - read-only 범위와 create/update/delete 범위를 phase-1 에서 어디까지 열지 문서로 고정
- [x] `calendar.list_events` contract 초안 구현
- [x] `calendar.find_conflicts` contract 초안 구현
- [x] `calendar.create_event` contract 초안 구현
- [x] waiting-input 질문 경로 정의
- [x] create approval 정책 정의
- [x] conflict summary normalization 정의
- [x] create ready / approval pending status 연결
- [x] contract / conflict / waiting-input focused test 추가

- 이번 작업 반영:
  - `source/nanobot/automation/calendar.py` 에 direct n8n webhook 기반 `N8NCalendarAutomationClient`, `CalendarAutomationSessionRunner`, `CalendarCreateRequest` 를 추가해 `calendar.list_events`, `calendar.find_conflicts`, `calendar.create_event` 의 phase-1 seam 을 만들었다.
  - 일정 조회는 AI_Assistant 의 `assistant-automation` webhook(`N8N_WEBHOOK_PATH`)를 재사용하고, 일정 생성은 `assistant-calendar-create` webhook(`N8N_CALENDAR_CREATE_WEBHOOK_PATH`)를 직접 호출하도록 맞췄다.
  - phase-1 conflict check 는 `assistant-automation` 의 `events[]` 와 `timeMin` / `timeMax` 입력을 사용해 day-window 기준 overlap를 계산하고, structured event data 가 비어 있는 구형 executor 응답에는 false negative 대신 `structured_conflict_data_unavailable` 로 막도록 했다.
  - `source/nanobot/command/builtin.py` 에 `/calendar today`, `/calendar check --start ... --end ...`, `/calendar create --title ... --start ... --end ...`, `/calendar approve`, `/calendar deny` 를 추가해 실제 user-triggered path 를 열었다.
  - `/calendar create` 에 필수 필드가 빠진 경우 session metadata + command interceptor 기반 waiting-input flow 로 `title -> start -> end` 를 순차적으로 수집하도록 확장했다.
  - `source/webui/src/components/thread/ThreadShell.tsx` 와 관련 type/test 를 보강해 calendar event preview 도 thread 안에서 바로 보이게 했다.
  - AI_Assistant 쪽 source-of-truth 도 함께 갱신해 `workflows/n8n/assistant-automation.json` 이 `events[]`, `count`, `timeMin`, `timeMax`, `window_label` 을 다루도록 바꿨고, `docs/google-calendar-integration.md` 의 응답 예시도 같은 shape로 동기화했다.
  - focused 검증으로 `PYTHONPATH=$PWD ./.venv/bin/pytest tests/test_automation_results.py tests/test_calendar_automation.py tests/command/test_builtin_calendar.py -q` 기준 `20 passed`, `npm test -- src/tests/thread-shell.test.tsx` 기준 `22 passed`, `npm run build` 통과를 확인했다.
  - live n8n 검증으로 running `assistant-automation` workflow 를 재import/publish 하고 webhook registration row 를 복구한 뒤, `POST http://127.0.0.1:5678/webhook/assistant-automation` 이 `{"reply":"오늘 일정은 14:00-14:30 Nanobot webui calendar validation입니다.","action":"calendar-summary","count":1,"events":[...]}` 형태의 structured payload 를 반환하는 것을 확인했다. `POST http://127.0.0.1:5678/webhook/assistant-calendar-create` 도 실제 `event_id=mngftbafpisks34cns5fdqdnjg` 를 반환했다.
  - live WebUI/launchd 검증으로 `~/.nanobot/nanobot.env` 에 `N8N_WEBHOOK_PATH`, `N8N_CALENDAR_CREATE_WEBHOOK_PATH` 를 추가하고 재기동한 뒤, `/calendar today` -> `오늘 등록된 일정이 없습니다.`, `/calendar create ...` -> `Calendar create approval required`, `/calendar approve` -> `Calendar event created` 흐름이 실제 session과 상단 status card에 반영되는 것을 확인했다.
  - 추가 live 검증으로 `/calendar create --details "waiting input live validation"` 뒤에 일반 답변으로 `Nanobot waiting-input validation` -> `2026-05-06T11:00:00+09:00` -> `2026-05-06T12:00:00+09:00` 를 순차 입력했을 때 최종 `Calendar create approval required` 로 이어지는 waiting-input flow 를 확인했다.
  - 후속 live overlap 검증으로 WebUI에서 `/calendar check --start 2026-05-04T14:00:00+09:00 --end 2026-05-04T14:30:00+09:00` 를 실행했을 때 `Conflicts found` 와 `Nanobot webui calendar validation` conflicting event preview 가 실제 session UI에 표시되는 것을 확인했다.

완료 기준:

- 일정 조회, 충돌 확인, 일정 생성 준비가 assistant action 으로 정규화된다.

### 5.6 proactive briefing + quiet hours

참고 문서:

- `docs/execution-backlog/06-proactive-briefing-and-quiet-hours-phase1.md`
- `docs/conceptual-phase/proactive-policy-and-quiet-hours.md`

현재 상태:

- [x] 완료

이번 phase-1 목표:

- heartbeat / periodic task 기반 위에 morning briefing, approval/blocked digest, quiet hours 정책을 제한적으로 올린다.

세부 작업 항목:

- [x] briefing / reminder / follow-up digest category 최소안 고정
- [x] quiet hours config 최소 모델 정리
- [x] active / waiting-approval / blocked / schedule summary source 결정
- [x] WebUI-first proactive delivery 원칙 정리
- [x] quiet hours 중 push suppression 규칙 정리
- [x] severity / frequency 제한 규칙 정리
- [x] heartbeat consumer contract / deliverability suppression test 후보 정리

- 이번 작업 반영:
  - `source/nanobot/heartbeat/proactive.py` 를 추가해 heartbeat 전용 proactive policy helper 를 만들고, `waiting-approval`, `blocked`, 최근 `calendar.list_events`, 최근 `mail.list_messages/list_threads` 결과를 digest source 로 묶는 `build_proactive_context(...)` 와 WebUI-first + quiet hours suppression 을 결정하는 `decide_heartbeat_target(...)` 를 구현했다.
  - `source/nanobot/config/schema.py` 의 `HeartbeatConfig` 에 `webui_first`, `max_digest_items`, `quiet_hours_*` 최소 설정을 추가해 quiet hours 시작/종료 시각, timezone, 허용 채널, critical 예외 모델을 runtime config 표면으로 올렸다.
  - `source/nanobot/cli/commands.py` 에서 heartbeat execution prompt 앞에 proactive context summary 를 주입하고, notify 시에는 quiet hours 밖에서는 최근 external channel fallback 을 우선하고, quiet hours 또는 external channel 부재 시에는 WebUI-first 로 남도록 policy 를 적용했다.
  - phase-1 기준 severity/frequency 는 별도 복잡한 scorer 대신 `max_digest_items` 로 digest fan-out 을 제한하고, quiet hours 중 non-WebUI push 를 기본 억제하는 쪽으로 먼저 고정했다.
  - `source/nanobot/templates/HEARTBEAT.md`, `source/docs/chat-commands.md`, live `~/.nanobot/workspace/HEARTBEAT.md` 에 morning briefing task 를 반영해 기본 heartbeat workspace 에서 바로 briefing prompt 가 seed 되도록 맞췄다.
  - `source/nanobot/session/manager.py`, `/api/sessions`, `source/webui/src/lib/sessionMetadata.ts`, `source/webui/src/components/thread/ThreadShell.tsx` 를 연결해 quiet hours 로 보류된 proactive update 를 `proactive_summary` metadata 와 owner-aware `Assistant summary` 의 `proactive held` 문구로 노출하도록 보강했다.
  - focused 검증으로 `PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_heartbeat_proactive.py tests/agent/test_heartbeat_service.py -q` 기준 `12 passed` 를 확인했다.
  - 추가 focused 검증으로 `PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_heartbeat_proactive.py tests/agent/test_heartbeat_service.py tests/channels/test_websocket_http_routes.py -q` 기준 `37 passed`, `npm test -- src/tests/thread-shell.test.tsx` 기준 `23 passed`, `npm run build` 통과를 확인했다.
  - live 반영 검증으로 launchd gateway/api 를 재기동한 뒤 `curl -fsS http://127.0.0.1:18790/health` 가 `{"status": "ok"}` 를 반환하고, `curl -i http://127.0.0.1:8765/webui/bootstrap | head -n 10` 이 HTTP 200 을 반환하는 것을 확인했다.
  - live runtime 검증으로 `~/.nanobot/nanobot.env` 를 source 한 상태에서 heartbeat와 같은 `AgentLoop` 경로로 morning briefing prompt preview 를 실행했고, `오늘은 보류 중인 승인 요청을 검토하고, 중단된 작업을 재개한 후 다음 작업으로 넘어가세요.` 응답을 확인했다.
  - live WebUI 검증으로 실제 linked Telegram session 에 `proactive_summary(status=suppressed, category=briefing)` 를 주입한 뒤 thread snapshot 에서 `Assistant summary` 의 `1 proactive held` 와 `Quiet hours held Morning briefing ready for Telegram ...` 문구가 노출되는 것을 확인했다.
  - 재기동 중 `commands.py` 의 `hb_cfg` 초기화 순서 오류로 gateway 가 한 번 실패했고, stderr traceback 확인 후 같은 slice 에서 즉시 수정해 재기동/health 검증까지 다시 닫았다.

완료 기준:

- proactive 가 trust-breaking automation 이 아니라 controlled assistant surface 로 정의된다.

## 6. 방향 A 문서군 재정리

상태:

- [x] 2026-05-01 기준 1차 재정리 완료

판정:

- 현재 active implementation source of truth 는 section 5 execution backlog 와 `docs/execution-backlog/*.md` 다.
- `docs/assistant-direction-a/**` 는 초기 방향 정리와 설계 배경을 보존하는 archive/reference 문서군으로 유지한다.
- 따라서 section 6 은 별도 active backlog 가 아니라, 기존 방향 A 문서가 section 5 에 어떻게 흡수됐는지 추적하는 review 메모로 유지한다.

흡수 상태 점검:

- `05-1 WebUI assistant status` 는 section 5.1 과 `docs/execution-backlog/01-owner-aware-control-surface-phase1.md` 로 흡수됐고 구현/검증도 완료됐다.
- `05-2 continuity metadata` 는 section 5.2 와 `docs/execution-backlog/02-owner-memory-and-task-backbone-phase1.md` 로 흡수됐고 구현/검증도 완료됐다.
- `05-3 Gmail read-only + draft` 는 section 5.3 과 `docs/execution-backlog/03-gmail-readonly-draft-phase1.md` 로 흡수됐고 구현/검증도 완료됐다.
- `05-4 Gmail send approval` 는 section 5.4 와 `docs/execution-backlog/04-gmail-send-approval-phase1.md` 로 흡수됐고 구현/검증도 완료됐다.
- `05-5 Kakao adapter` 는 아직 active execution backlog 로 승격되지 않았다. 현재는 section 5.3~5.5 선행 환경 준비와 `docs/assistant-direction-a/07-kakao-integration-feasibility.md`, 관련 workplan 문서를 reference 로만 유지한다.
- `docs/assistant-direction-a/08-shared-action-result-and-failure-shape.md` 는 공통 vocabulary 참고 문서로 유지하되, 실제 구현 source of truth 는 `source/nanobot/automation_results.py`, 관련 automation/runtime 코드, section 5 진행 메모다.

현재 section 6 이후 이슈에 대한 작업 진행 여부 검토:

- 방향 A 문서군에서 별도 active 작업으로 유지할 항목은 현재 없다.
- Kakao 는 운영 권한/ingress 준비 확인까지만 끝났고, 실제 구현은 새 execution backlog item 또는 section 5 active slice 로 다시 승격될 때 시작한다.
- 따라서 section 6 이하의 예전 체크리스트는 더 이상 진행 상태를 직접 추적하지 않고, archive/reference 상태와 흡수 위치만 관리한다.

문서 운영 원칙:

- `docs/assistant-direction-a/README.md` 와 `01-implementation-plan.md` 는 archive/source-of-truth 안내를 포함해야 한다.
- `docs/assistant-direction-a/workplans/05-1-*` ~ `05-5-*` 는 active checklist 가 아니라 `completed`, `absorbed`, `deferred` 상태를 먼저 밝힌다.
- 앞으로 새 구현을 다시 열 때는 section 5 또는 `docs/execution-backlog/*.md` 에서 먼저 backlog item 을 만들고, 필요한 경우에만 방향 A 문서를 reference 로 갱신한다.

## 메모

- 5.3~5.5 공통 action result / failure / visibility shape 상위 설계는 `docs/assistant-direction-a/08-shared-action-result-and-failure-shape.md` 기준으로 정리했다.
- Gmail/Calendar/Kakao 후속 구현의 environment source of truth 는 우선 `/Volumes/ExtData/AI_Project/AI_Assistant/.env` 와 해당 integration 문서들이며, Nanobot 문서에는 secret value 대신 변수명과 권한 요구만 유지한다.

- 현재 live 기본 target 은 `local-llm` 이고, session override 로 `smart-router` 를 선택할 수 있다.
- source venv 기준 재검증에서는 available targets 가 `default`, `smart-router`, `local-llm` 으로 확인됐다.
- `http://127.0.0.1:8900/health`, `http://127.0.0.1:18790/health`, `http://127.0.0.1:8900/v1/models` 는 현재 정상 응답을 반환한다.
- `~/.nanobot/nanobot.env` 는 이제 shell 에 이미 export 된 `OPENAI_API_KEY` 또는 `OPEN_API_KEY` 를 빈 값으로 덮어쓰지 않는다.
- live `~/.nanobot/config.json`, `~/.nanobot/config.api.json` 는 현재 `smartrouter.local=vllm/${LOCAL_LLM_MODEL}`, `smartrouter.mini=openai/gpt-5.4-mini-2026-03-17`, `smartrouter.full=openai/gpt-5.4-2026-03-05` 로 설정돼 있다.
- live 설정 백업은 `~/.nanobot/backups/2026-04-22-smart-router-openai/` 아래에 `nanobot.env.bak`, `config.json.bak`, `config.api.json.bak` 로 저장했다.
- `source/.venv` 기준 `openai` provider direct call 이 `OPENAI_API_KEY` 경로로 성공했고, smart-router 실검증에서도 `requestedTier/finalTier` 가 `local/local`, `mini/mini`, `full/full`, fallback fault injection 기준 `local/mini`, `mini/full` 로 확인됐다.
- live Telegram bot `getMyCommands` 재확인 결과 `/usage` command 가 실제 menu 에 등록된 상태다.