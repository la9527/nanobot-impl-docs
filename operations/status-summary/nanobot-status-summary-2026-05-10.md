# Nanobot 작업 상태 정리 - 2026-05-10

## 문서 목적

이 문서는 2026-05-10 기준 Nanobot 의 observable reasoning / WebUI reasoning visibility 작업, 새로고침 후 reasoning 유지, action result row 축소와 상세 토글 보강, live 검증 결과를 정리한 운영 메모다.

## 현재 상태 요약

현재 확인된 상태는 다음과 같다.

- source repo `/Volumes/ExtData/Nanobot/source` 는 현재 `feature/nanobot-fork-runtime` 브랜치 기준이며 `origin/feature/nanobot-fork-runtime` 와 sync 상태다.
- source repo working tree 는 현재 clean 상태다.
- Nanobot gateway 는 launchd label `com.nanobot.gateway` 로 실행 중이다.
- Nanobot API 는 launchd label `com.nanobot.api` 로 실행 중이다.
- local llama runtime 은 launchd label `com.nanobot.llama-lfm2-server` 로 실행 중이다.
- live gateway health 는 `http://127.0.0.1:18790/health` 에서 `{"status": "ok"}` 로 확인했다.
- live WebUI bootstrap 은 인증 없이 호출하면 `401 Unauthorized` 가 반환되며, 이는 현재 운영 설정상 정상 동작이다.
- live WebUI 에서는 assistant 답변 앞에 `생각` 한 줄이 history row 로 표시되고, 새로고침 후에도 같은 순서가 유지되는 것을 확인했다.
- live WebUI 에서는 `일정 결과` compact row 자체를 클릭해 상세를 열 수 있다.

## 이번에 반영한 주요 작업

### 1. observable reasoning 설계 문서를 정리했다

- `docs/archive/observable-reasoning-plan/README.md` 를 기준 entry 로 유지했다.
- `docs/archive/observable-reasoning-plan/telegram-v2-and-reasoning-visibility-plan-2026-05-10.md` 에 정의된 `reasoning_visibility = off | status_only | summary | debug_trace` 정책을 구현 기준으로 삼았다.
- 현재 구현 상태를 계획 항목 기준으로 추적할 수 있도록 별도 checklist 문서를 추가했다.

### 2. WebUI reasoning row 를 실제 history 순서에 맞게 정리했다

- live `tool_hint` / `progress` 가 있는 경우 assistant placeholder 뒤가 아니라 assistant 답변 앞에 들어오도록 정리했다.
- live trace 가 없는 세션에서는 synthetic reasoning trace row 를 history 배열에 주입해 `생각 -> 답변` 순서를 맞췄다.
- gray tone reasoning surface 와 `생각`, `생각 중` 라벨을 유지하면서 summary card 가 아니라 history row 중심으로 동작하게 맞췄다.

### 3. 새로고침 후 reasoning 유지 경로를 구현했다

- `nanobot/agent/loop.py` 의 `_save_turn()` 에서 final assistant message 저장 시 localized safe string `visible_reasoning` 을 함께 기록하게 했다.
- 이 값은 raw `reasoning_content` 를 그대로 노출하지 않고, UI-safe 한 한 줄 상태만 저장하는 방향으로 제한했다.
- `webui/src/hooks/useSessions.ts` 의 history hydration 에서 `visible_reasoning` 을 assistant 답변 앞 trace row 로 재구성하게 했다.
- live websocket turn 검증에서 session history JSON 의 마지막 assistant message 에 `visible_reasoning` 이 실제로 persisted 되는 것을 확인했다.

### 4. action result row 를 더 compact 하게 줄이고 클릭 표면을 넓혔다

- `ThreadInlineActionResult` 는 기본 상태에서 한 줄 compact summary 로만 보이게 했다.
- preview / conflict / thread detail 은 `세부 정보` 를 눌렀을 때만 보이게 유지했다.
- 이후 사용성 보정으로 trailing `세부 정보` 버튼뿐 아니라 row summary 영역 자체도 클릭하면 상세가 열리게 연결했다.
- 이 이슈는 `docs/planning/todo.md` 에 후속 점검 항목으로도 기록했다.

## 이번에 확인된 운영 포인트

### 1. reasoning persistence 는 final answer 재요약이 아니라 persisted field 로 푸는 편이 안정적이다

- 사용자 요구는 "최종 답변 요약" 이 아니라 "생각 한 줄이 답변 앞에 남는 것" 이었다.
- 따라서 새로고침 후 복원은 final answer text 를 다시 줄이는 방식보다, assistant message 자체에 safe visible field 를 저장하는 편이 더 안정적이었다.

### 2. live reasoning 확인은 websocket history 와 브라우저 둘 다 봐야 한다

- unit test 와 build 만으로는 실제 turn 저장 순서를 완전히 확인하기 어렵다.
- 이번에는 live websocket turn 을 실제로 보내고 `/api/sessions/{key}/messages` 응답에서 `visible_reasoning` 필드를 직접 확인했다.
- 이후 브라우저에서 같은 session 을 열어 `생각 -> 답변` 순서와 action result 클릭 동작까지 함께 확인했다.

### 3. launchd 재기동 직후 health 는 한 번 튈 수 있다

- restart 직후 `curl -fsS http://127.0.0.1:18790/health` 가 일시적으로 실패할 수 있었다.
- 하지만 launchd list 와 즉시 재시도한 health check 에서는 정상 복구를 확인했다.

### 4. broader `thread-shell` test file 은 이번 slice 의 직접 gate 로 쓰기 어려웠다

- `webui/src/tests/thread-shell.test.tsx` 전체 파일에는 이번 변경과 직접 연결되지 않은 approval/status expectation failure 가 섞여 있었다.
- 따라서 이번 slice 는 `useSessions`, `ThreadInlineActionResult`, backend `_save_turn()` 중심의 좁은 테스트로 검증했다.

## 검증 결과

source-tree 기준 focused 검증:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_loop_save_turn.py -q
# 27 passed

cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/useSessions.test.tsx
# 10 passed

npm test -- src/tests/thread-inline-action-result.test.tsx
# 2 passed

npm run build
# 통과
```

live runtime 기준 검증:

```bash
/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh

launchctl list | grep nanobot
curl -fsS http://127.0.0.1:18790/health
curl -i http://127.0.0.1:8765/webui/bootstrap
```

live websocket / browser 기준 확인:

- authenticated WebUI bootstrap token 으로 websocket turn 을 생성한 뒤 session history JSON 을 다시 읽어 `visible_reasoning` persisted 여부 확인
- 결과: 마지막 assistant message 에 `visible_reasoning = "답변 방향을 정리한 뒤 응답했습니다."` 저장 확인
- live 브라우저 `i18n 확인 일정` session 에서 `일정 결과` summary row 클릭 시 상세가 펼쳐지는 것 확인

## 현재 남아 있는 제약과 리스크

- `telegram_v2` / Mini App 방향은 현재 설계 문서 단계이며 runtime/channel 구현은 아직 시작하지 않았다.
- reasoning contract 는 final assistant turn persistence 까지는 구현됐지만, checkpoint 복구나 historical backfill 을 별도 정책으로 더 정리할 여지는 남아 있다.
- action result detail surface 는 현재 inline expand 형태이며, 이후 side sheet 또는 별도 detail surface 로 바꿀 가능성이 남아 있다.
- `/Volumes/ExtData/Nanobot/docs` 와 `/Volumes/ExtData/Nanobot/.github` 는 source repo 밖 경로라, source commit 과 문서/설정 변경은 별도 git 경계로 관리된다.

## 현재 기준 권장 운영 명령

```bash
git -C /Volumes/ExtData/Nanobot/source status --short --branch

cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/agent/test_loop_save_turn.py -q

cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/useSessions.test.tsx
npm test -- src/tests/thread-inline-action-result.test.tsx
npm run build

/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
launchctl list | grep nanobot
curl -fsS http://127.0.0.1:18790/health
curl -i http://127.0.0.1:8765/webui/bootstrap
```

## 다음 작업 후보

- broader `webui/src/tests/thread-shell.test.tsx` failure 를 이번 reasoning/action-result 구조와 분리해 다시 정리한다.
- action result detail surface 를 inline expand 가 아니라 sheet / inspector 패턴으로 옮길지 결정한다.
- `telegram_v2` / Mini App 범위를 첫 runnable slice 수준으로 줄여 prototype 을 시작한다.
- reasoning persistence 를 checkpoint / recovery 경로까지 같은 contract 로 확장할지 판단한다.

## 결론

2026-05-10 작업의 핵심은 observable reasoning 을 "보여줄까 말까" 수준이 아니라, live WebUI 에서 실제로 `생각 -> 답변` 순서로 남고 새로고침 후에도 유지되며, action result 도 compact 하게 다듬는 쪽으로 구체화한 것이다. 현재는 설계 문서, focused test, production build, launchd live runtime, websocket history, 브라우저 클릭 검증까지 이어진 상태이며, 다음 남은 과제는 broader thread-shell 정리와 `telegram_v2` 의 실제 runnable slice 정의다.

## 추가 작업 메모 - WebUI 상태/용어/i18n 정리

### 현재 상태 갱신

- source repo `/Volumes/ExtData/Nanobot/source` 는 현재 `feature/nanobot-fork-runtime` 브랜치 기준이며 `origin/feature/nanobot-fork-runtime` 와 sync 상태다.
- source repo 에는 이번 작업을 `869a7a24` (`webui: polish thread status, reasoning, and i18n surfaces`) 커밋으로 반영했고, 이후 origin 으로 push 했다.
- source repo working tree 는 현재 clean 상태다.
- 다만 `/Volumes/ExtData/Nanobot/docs/operations/status-summary/*` 와 `/Volumes/ExtData/Nanobot/.github/instructions/*` 는 source repo 밖 경로이므로, 이번 status-summary 갱신과 i18n 기본 instruction 추가는 같은 source git commit 에 포함되지 않는다. `/Volumes/ExtData/Nanobot/docs` 자체는 별도 git repo 이므로 docs 변경은 별도 commit 으로 추적할 수 있다.

### 이번에 반영한 추가 작업

#### 1. WebUI thread/status surface 를 더 단순하게 정리했다

- bridged session 에서는 삭제 affordance 를 숨기고, websocket session 에서만 삭제 UI 를 노출하도록 정리했다.
- `ThreadViewport` 의 중복 하단 여백을 제거해 composer 아래 빈 공간이 과하게 남지 않도록 보정했다.
- header 아래 status rail 을 한 줄 요약 + `세부 정보` 버튼 구조로 단순화해 `연결 6건` 같은 애매한 표현을 `관련 세션 6개 · 채널 Telegram · 최근 활동 ...` 형태로 읽히게 맞췄다.

#### 2. backend/frontend task summary 와 action result 표면을 같이 다듬었다

- backend `task_summary` 기본 문구와 `/new` 응답을 locale 기반으로 정리했다.
- generic completed next-step hint 가 상단 상태 UI 를 오염시키지 않도록 frontend normalization 과 backend continuity 생성 경로를 함께 보정했다.
- inline action result 는 compact row + 필요 시 detail disclosure 패턴으로 더 일관되게 유지했다.

#### 3. WebUI i18n 잔여분과 한국어 용어를 정리했다

- settings 화면의 `Account`, `Sign out`, footer 저장 문구 같은 영어 fallback 을 locale key 로 옮기고 `계정`, `로그아웃`, `저장` 계열 한국어 표현으로 정리했다.
- 한국어 locale 에 남아 있던 `provider/model/runtime`, `assistant`, `proactive update`, `Reasoning` 혼용 표현을 현재 UI 의미에 맞게 `공급자`, `모델`, `런타임`, `작업`, `선제 업데이트`, `추론 표시 수준` 등으로 맞췄다.
- 세부 정보 패널에서도 `현재 작업`, `사용자 기본값`, `메모리 도구` 같은 표현으로 정리했다.

#### 4. WebUI 용어 기준 문서를 따로 고정했다

- `/Volumes/ExtData/Nanobot/docs/guide/10-webui-terminology.md` 를 추가해 `대시보드`, `설정`, `세부 정보`, `대화`, `작업`, `타깃` 같은 현재 WebUI 기준 표현을 정리했다.
- user-guide 본문과 dated UX/backlog 문서에는 이 용어집을 기준으로 보거나, 상단 note 로 현재 기준을 따라야 한다는 연결을 추가했다.
- `/Volumes/ExtData/Nanobot/.github/instructions/i18n-default.instructions.md` 를 추가해 Nanobot source 작업에서는 i18n 을 기본 요구사항으로 다루도록 project instruction 도 보강했다.

### 추가 검증 결과

source-tree 기준 focused 검증:

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/app-layout.test.tsx
npm test -- src/tests/thread-shell.test.tsx -t "renders an owner-aware summary block from linked sessions and pending approvals"
npm test -- src/tests/i18n.test.tsx
npm run build
```

추가 focused gate:

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-status.test.ts src/tests/thread-status-block.test.tsx
```

확인 결과:

- settings/account/footer 관련 테스트는 통과했다.
- owner-aware summary / details panel 관련 좁은 `thread-shell` 케이스는 통과했다.
- locale smoke test 와 production build 는 통과했다.

live WebUI 기준 확인:

- shared browser page 를 reload 한 뒤 `대시보드`, `새 채팅`, `세부 정보`, `타깃 자동`, compact status rail 문구가 실제로 반영된 것을 확인했다.
- settings 화면에서 `기본값`, `기본 공급자`, `기본 모델`, `런타임에서 관리됨`, `추론 표시 수준`, `계정`, `로그아웃` 이 보이는 것을 확인했다.

### 현재 남아 있는 제약과 리스크 갱신

- `/Volumes/ExtData/Nanobot/docs` 와 `/Volumes/ExtData/Nanobot/.github` 는 source repo 밖이므로, 운영 메모와 instruction 갱신은 git history 상 source commit 과 분리된다.
- broader `webui/src/tests/thread-shell.test.tsx` 전체 파일은 여전히 이번 slice 와 직접 관계없는 expectation drift 를 섞어 가질 수 있으므로, 계속 focused gate 중심으로 다루는 편이 안전하다.
- historical 기획 문서에는 예전 `assistant` / `dashboard` / `thread` 표현이 남아 있을 수 있으며, 현재 기준 해석은 용어집 문서를 통해 보완하는 방식으로 유지하고 있다.

### 다음 작업 후보 갱신

- broader `thread-shell` 테스트를 현재 UI 용어와 상태 구조 기준으로 다시 정리한다.
- calendar-webui-reliability-review, conceptual-phase 같은 나머지 dated 문서에도 현재 용어 기준 note 를 넓힐지 판단한다.
- live WebUI 세부 정보 패널을 실제 metadata 가 풍부한 session 기준으로 한 번 더 검증해, 대화/작업/메모리 표현이 모두 의도대로 보이는지 확인한다.