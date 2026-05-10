# upstream/main -> local main 반영 요약

## 비교 기준

- 이전 local `main`: `e3bca929`
- 현재 local `main` = `upstream/main`: `451d7408`
- 범위: `e3bca929..451d7408`
- 규모: `211 files changed, 22895 insertions(+), 2344 deletions(-)`

## 한 줄 요약

이번 sync 는 단순 hotfix 수준이 아니라, WebUI UX, image generation, provider 확장, session/memory 안정화, 채널별 thread/reply 보정, 보안/네트워크 가드 강화가 한 번에 들어온 대형 업데이트다.

## 추가된 기능

### 1. Image generation 기능군 추가

아래 항목이 새로 들어왔다.

- image generation tool 및 provider 계층 추가
- WebUI 에 image mode 와 관련 문서 추가
- image generation intent / artifacts / titles 보조 유틸 추가
- `docs/image-generation.md` 신규 문서 추가
- 관련 skill 추가: `nanobot/skills/image-generation/SKILL.md`

의미:

- 텍스트 채팅 중심이던 runtime 에서 이미지 생성 요청을 별도 tool/provider 흐름으로 다룰 수 있게 됐다.
- WebUI 는 일반 채팅과 이미지 생성 모드를 분리해 안내할 수 있게 됐다.

### 2. Provider 확장

주요 추가 항목:

- Bedrock native provider 추가
- LongCat OpenAI-compatible provider 추가
- Hugging Face inference provider 추가
- GitHub Copilot provider logout 지원 추가
- transcription retry / malformed response guard 보강

의미:

- provider 선택지가 늘었고, long request fallback 및 reasoning-content 정합성 문제가 다수 보완됐다.
- 운영 환경에 따라 local / LAN / hosted provider 조합을 더 유연하게 가져갈 수 있다.

### 3. WebUI 신규 UX 기능

새로 들어온 사용자 기능:

- ask_user choices 렌더링
- localized slash commands
- model settings runtime refresh
- video media attachment 렌더링
- chat layout / titles polish
- turn completion / streaming UX 개선
- delete dialog / sidebar toggle polish

의미:

- 단순 message viewport 에서 벗어나 assistant status, quick action, choice prompt, localized command UX 가 더 명확해졌다.
- stream_end 와 turn_end 사이 동작을 더 엄격히 처리하게 됐다.

### 4. Agent / session 기능 확장

새 기능:

- `ask_user` tool 추가
- `/history` command 추가
- `sender_id` 를 LLM runtime context 에 주입
- `toolHintMaxLength` config 추가
- configurable consolidation ratio 추가
- session replay/file-cap invariant 강화
- proactive delivery 의 channel session 기록 보강

의미:

- agent 가 사용자 확인을 요구하는 상호작용 패턴이 늘었고, session history 운용 방식이 더 보수적으로 바뀌었다.
- tool hint 와 history replay 는 now stricter / more configurable 하다.

## 개선된 내용

### 1. Memory / history 안정화

이 범위에서 특히 많이 보강된 영역이다.

- replay overflow 와 history trimming 정렬
- raw archive / hidden history / recent history cap 보정
- `history.jsonl` atomic write 및 directory fsync 보강
- file-state 를 session 단위로 격리
- model prompt 에 불필요한 `[Message Time: ...]` 류 메타가 새지 않도록 수정

영향:

- history corruption 가능성이 줄었고, 장기 세션에서 메모리 오염이나 context bloat 가 덜해졌다.
- 대신 기존 fork 가 session/history internals 를 건드리고 있다면 merge 충돌 가능성이 높아진다.

### 2. Channel thread / reply 정합성 개선

영향이 큰 채널 변경:

- Discord: full thread support, session isolation, allowlist enforcement
- Telegram: inline keyboard, video, silent unauthorized handling, attachment naming 보정
- Feishu: group thread / top-level session isolation, reply target 정교화, streaming buffer scope 개선
- Slack: thread context, slash command reply, proactive reply continuity 개선
- Matrix / Dingtalk / Weixin / WhatsApp: auth, SSRF, fetch, media, bridge 안정성 보강
- WebSocket: SSE / streaming / turn completion 처리 개선

영향:

- 멀티채널 thread continuity 가 전반적으로 강화됐다.
- 반면 channel adapter 를 별도 커스터마이즈했다면 merge 후 회귀 검증 범위가 넓어진다.

### 3. 보안 및 운영 안정성 강화

주요 보안/운영 변화:

- ExecTool path append shell injection 방지
- `allow_patterns` 우선순위 명확화
- outbound media fetch SSRF guard 강화
- local/LAN endpoint detection 및 keepalive 정책 조정
- non-localhost / LAN WebUI bootstrap 에 `token_issue_secret` 요구 강화
- unauthorized inbound 를 side effect 전에 차단

영향:

- 운영은 더 안전해졌지만, 기존에 느슨한 LAN bootstrap 이나 exec allow/deny 규칙에 기대던 환경은 설정 점검이 필요하다.

### 4. CLI / setup / docs 개선

- provider logout command 추가
- `_retry_wait` interactive routing 개선
- update-setup wizard skill 추가
- macOS launchd setup 문서 정리
- config / deployment / README news 갱신

## 삭제된 항목 또는 실질적으로 사라진 동작

이번 sync 에서 눈에 띄는 대규모 기능 삭제는 확인되지 않았다. 다만 아래는 기존보다 더 엄격해져서 체감상 "이전처럼 안 되는" 변화로 볼 수 있다.

- LAN / non-localhost WebUI bootstrap 은 `token_issue_secret` 없이 동작하지 않게 됨
- unauthorized inbound message 는 이전보다 더 이른 단계에서 drop 됨
- progress / stream completion 처리에서 느슨한 fallback 이 줄어듦
- session/history replay 는 파일 cap / replay cap 규칙을 더 엄격히 따름
- ExecTool 정책은 `allow_patterns` / `deny_patterns` 적용 순서가 바뀌어 기존 예외 규칙 체감이 달라질 수 있음

## local fork 입장에서 중요하게 볼 변경 포인트

특히 `feature/nanobot-fork-runtime` 같은 local fork 와 충돌하기 쉬운 축은 아래다.

- `nanobot/agent/loop.py`, `nanobot/agent/memory.py`, `nanobot/agent/runner.py`
- `nanobot/session/manager.py`
- `nanobot/channels/websocket.py`
- `nanobot/api/server.py`
- `webui/src/App.tsx`
- `webui/src/components/thread/ThreadComposer.tsx`
- `webui/src/components/thread/ThreadShell.tsx`
- `webui/src/hooks/useNanobotStream.ts`
- `webui/src/i18n/locales/*/common.json`

이유:

- upstream 는 ask_user / image generation / stricter turn lifecycle / auth bootstrap / localized command UX 를 넣었다.
- local fork 는 model target, continuity, owner-aware summary, automation 결과, smart-router 표시, custom status block 을 유지하고 있었다.
- 두 방향 모두 동일한 핵심 surface 를 수정했기 때문에 conflict density 가 높을 수밖에 없다.

## 권장 후속 확인

- WebUI live bootstrap / auth 경로 재확인
- image generation provider 설정값 유무 점검
- LAN 접근 환경이면 `token_issue_secret` 운영값 점검
- session/history trimming 관련 기존 로컬 회귀 시나리오 재검증
- channel 별 threaded reply 가 중요한 통합 경로를 좁게 재검증
