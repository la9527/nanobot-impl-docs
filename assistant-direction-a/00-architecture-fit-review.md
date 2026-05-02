# AI Assistant 초기 아키텍처의 Nanobot 수용성 검토

## 상태 메모

- 이 문서는 방향 A 착수 시점의 수용성 검토 기록이다.
- 2026-05-01 기준 active implementation source of truth 는 `docs/todo.md` section 5 와 `docs/execution-backlog/*.md` 다.
- 따라서 본문은 현재 실행 체크리스트가 아니라, 어떤 AI Assistant 요소를 Nanobot에 흡수할 수 있다고 판단했는지 설명하는 historical reference 로 읽는다.

## 1. 문서 목적

이 문서는 초창기 AI Assistant 구상 문서인 `AI_Assistant/docs/architecture.md`를 기준으로, 현재 Nanobot 구조가 그 요구를 얼마나 수용할 수 있는지 검토한 결과를 정리한다.

정리 범위는 아래 세 가지다.

- AI Assistant 초기 요구 중 Nanobot에 그대로 흡수 가능한 요소
- 비슷한 기능은 있으나 구현 방식이 달라 재해석이 필요한 요소
- 현재 Nanobot 구조에 없거나, 넣더라도 핵심 철학과 충돌할 가능성이 큰 요소

문서 목적은 단순 비교가 아니라, 앞으로 Nanobot 기준에서 어떤 방향으로 확장하는 것이 합리적인지 결정하는 데 있다.

## 2. 결론 요약

한 줄 결론부터 말하면, AI Assistant 초기 설계의 절반 이상은 Nanobot 위에 재구성 가능하다. 다만 `FastAPI + LangGraph + n8n + PostgreSQL/Redis + Kakao 중심 자동화 허브`라는 원안 전체를 Nanobot에 그대로 이식하는 방식은 적합하지 않다.

현재 Nanobot가 특히 잘 수용하는 영역은 아래다.

- 로컬 우선 + 필요 시 외부 LLM 사용
- 단일 사용자 중심의 WebUI / gateway / OpenAI-compatible API 구성
- Slack, Telegram, Email, WebSocket 기반 멀티채널 입력
- 세션 지속성, 장기 메모리, 압축 및 Dream 기반 memory 운영
- MCP 및 skill 기반 확장
- 실행 전 interactive approval
- macOS launchd, Docker, Tailscale 계열의 self-hosted 운영 방식

반대로 현재 Nanobot가 약하거나, 그대로 넣기 어려운 영역은 아래다.

- Kakao 공식 채널 / OpenBuilder 연동
- Gmail / Calendar / Notion 중심의 업무 자동화 orchestration
- n8n을 실행 중심 계층으로 두는 구조
- PostgreSQL / Redis 기반의 중앙 상태 저장 모델
- approval ticket, async task, session state를 API/DB 중심으로 다루는 구조
- AppleScript / Playwright / Notes 같은 macOS 자동화 계층을 1급 실행기로 다루는 구조
- extraction JSON을 모든 요청 앞단에 강제하는 skill-first orchestration

즉, Nanobot는 AI Assistant의 "개인용 self-hosted agent" 방향은 잘 수용하지만, "채널/업무 자동화 허브" 방향은 부분 재설계가 필요하다.

## 3. 항목별 수용성 평가

| AI Assistant 초기 요소 | Nanobot 현재 상태 | 수용성 | 판단 |
|---|---|---|---|
| 웹 UI 중심 운영 | 자체 WebUI + WebSocket gateway + embedded HTTP surface 보유 | 높음 | Open WebUI 대신 Nanobot WebUI로 대체하는 편이 자연스럽다. |
| OpenAI 호환 API | `nanobot serve`로 `/v1/chat/completions`, `/v1/models` 제공 | 높음 | FastAPI 호환층 일부를 대체 가능하다. |
| 로컬/외부 LLM 병행 | 다수 provider + local provider + env 기반 config 지원 | 높음 | AI Assistant의 local-first 원칙과 잘 맞는다. |
| 다중 채널 입력 | Telegram, Discord, Slack, Email, WeChat, WhatsApp 등 지원 | 높음 | Kakao만 빠져 있고 나머지는 오히려 더 넓다. |
| 공통 세션 지속성 | session manager + channel:key 구조 + WebUI bridge 보유 | 중간 이상 | 단일 사용자 기준으로는 충분하지만, identity mapping 테이블 구조는 아니다. |
| 장기 메모리 | session + `history.jsonl` + `SOUL.md`/`USER.md`/`MEMORY.md` + Dream | 높음 | AI Assistant 초안보다 더 구체적이고 운영 친화적이다. |
| 승인 흐름 | ask_user + tool approval + pending queue 보유 | 중간 이상 | interactive approval은 강하지만, DB 기반 approval ticket 모델은 없다. |
| Skill / MCP 확장 | built-in skills + MCP server 연결 + runtime tool 확장 지원 | 높음 | AI Assistant의 skill-first 확장 철학과 가장 잘 맞는 부분이다. |
| FastAPI 중심 core API | gateway + serve는 있으나 범용 REST orchestration 계층은 아님 | 중간 | 일부는 대체 가능하지만 구조 철학이 다르다. |
| LangGraph state router | agent loop / pending queue / memory / approval은 있으나 LangGraph는 없음 | 중간 | 기능은 비슷한 부분이 많지만 구현 방식은 다르다. |
| n8n automation layer | 현재 없음 | 낮음 | webhook or MCP executor로 일부 대체는 가능하나 core 구조는 아님. |
| PostgreSQL / Redis 상태 계층 | 기본은 file/workspace/session 중심 | 낮음 | 의도적으로 더 가벼운 구조다. |
| Kakao 공식 채널 | 현재 없음 | 낮음 | 새 channel plugin 또는 별도 adapter 개발이 필요하다. |
| Gmail/Calendar/Notion 비서 자동화 | Email channel은 있으나 SaaS automation workflow 계층은 약함 | 낮음 | 업무 자동화 허브로 쓰려면 새 integration 계층이 필요하다. |
| 브라우저 자동화 | web search / browse-read 계열은 있으나 full browser operator는 약함 | 중간 이하 | 조사/검색은 가능하지만 Playwright-driven action system과는 차이가 크다. |
| macOS 로컬 자동화 | 현재 소스 기준 1급 runtime 계층으로 정리되어 있지 않음 | 낮음 | MCP 또는 별도 host runner로 설계해야 맞다. |

## 4. 비슷한 기능 비교 검토

### 4.1 Open WebUI + FastAPI vs Nanobot WebUI + gateway + serve

AI Assistant 원안은 Open WebUI를 프론트로 두고 FastAPI를 core API로 두는 구조였다. 이 구조는 `UI`와 `비즈니스 로직`을 분리한다는 점에서는 명확하지만, 개인용 self-hosted agent 기준으로는 운영 레이어가 다소 무겁다.

Nanobot는 이미 아래 조합을 제공한다.

- WebUI: browser chat surface
- gateway: 실시간 채널 및 WebSocket surface
- serve: OpenAI-compatible API

따라서 Nanobot 기준에서는 Open WebUI를 다시 얹는 것보다, 현재 WebUI와 `serve`를 확장하는 편이 더 일관적이다.

권장 판단:

- Open WebUI를 직접 가져오기보다 Nanobot WebUI를 기준 UI로 유지
- 외부 연동이 필요하면 `serve` 또는 gateway의 HTTP surface를 늘리는 방식이 적절

### 4.2 LangGraph router vs Nanobot agent loop

AI Assistant 원안의 핵심은 상태 기반 router였다. Nanobot는 LangGraph를 쓰지 않지만, 실질적으로 비슷한 운영 요구를 아래 방식으로 해결하고 있다.

- session 기반 대화 지속
- pending queue 기반 mid-turn follow-up
- ask_user / tool approval
- memory consolidation + Dream
- tool / MCP 연결

다만 차이도 분명하다.

- AI Assistant는 state graph를 명시 구조로 두려 했다.
- Nanobot는 더 단순한 agent loop와 session metadata 쪽으로 구현되어 있다.

권장 판단:

- LangGraph 자체를 들여오는 것보다, Nanobot loop에 `structured extraction -> route intent -> tool policy` 단계를 보강하는 편이 현실적이다.

#### 4.2.1 LangGraph에서 기대했던 것 중 Nanobot에 아직 약한 부분

LangGraph를 도입하려던 목적이 단순히 "예전 정보를 기억하는 것"이 아니라, `상태 전이`, `분기`, `재개`, `장기 작업 제어`까지 포함한 workflow 제어였다면, Nanobot에는 아직 약한 부분이 있다.

| 관점 | LangGraph 쪽이 강한 부분 | Nanobot 현재 상태 | 판단 |
|---|---|---|---|
| 상태 전이 모델 | 노드/엣지 기반의 명시적 state graph | agent loop + session metadata 중심 | Nanobot는 흐름은 단순하지만 상태 전이가 명시적이지 않다. |
| 분기 제어 | 조건별 branching, subgraph, retry 경로 정의 용이 | tool 호출과 pending queue 수준의 분기 | 복잡한 workflow branching은 아직 약하다. |
| 장기 작업 재개 | checkpoint 기반 resume 설계에 유리 | pending queue와 session continuity 중심 | 간단한 재개는 가능하지만 durable workflow resume은 약하다. |
| 실패 복구 | 단계별 실패 처리와 재시도 경로가 구조적으로 표현됨 | 실패는 주로 turn/session 수준에서 처리 | 장기 task 관점의 복구 흐름은 미흡하다. |
| 가시성 | 왜 이 단계로 갔는지 graph 수준 추적 가능 | 현재는 thread, session, memory 관점 추적이 중심 | reasoning trace보다 operational trace가 약하다. |
| 다단계 automation | 여러 action을 하나의 workflow state로 묶기 쉬움 | assistant task를 쪼개서 순차 처리하는 쪽에 가깝다 | 현재 구조는 간단한 action에는 적합하지만 복합 automation은 별도 보강이 필요하다. |

즉, Nanobot는 `기억과 문맥 유지`에는 강하지만, `명시적 workflow/state machine`에는 상대적으로 약하다.

따라서 방향 A 기준에서는 아래 순서가 더 적절하다.

- 먼저 Nanobot의 memory, approval, session continuity를 활용한다.
- 그 다음 `structured extraction -> route intent -> tool policy -> assistant automation task` 계층을 보강한다.
- 정말 복잡한 long-running workflow가 필요해질 때만 별도 persistent orchestration을 검토한다.

### 4.3 n8n automation layer vs Nanobot skill/MCP/tool layer

AI Assistant 원안은 SaaS automation을 n8n에 위임하는 구조였다. Nanobot는 현재 n8n을 전제로 하지 않고, skills / tools / MCP를 통해 agent가 바로 실행하는 방향에 가깝다.

이 차이는 매우 중요하다.

- AI Assistant: workflow 중심
- Nanobot: agent runtime 중심

Nanobot에 n8n을 넣을 수는 있지만 core로 두기보다는 아래처럼 다루는 편이 맞다.

- 외부 webhook executor
- MCP server 뒤에 감춘 workflow backend
- 특정 업무 도메인용 custom tool

권장 판단:

- n8n을 Nanobot의 필수 중심 계층으로 넣는 것은 비권장
- 필요하면 `automation adapter` 또는 `MCP-backed workflow tool`로 한정 도입

### 4.4 Approval ticket 모델 vs Nanobot interactive approval

AI Assistant 원안은 승인 티켓을 별도 저장하고, API로 approve/reject 하며, 장기 작업 재개까지 염두에 두고 있었다.

Nanobot는 현재 아래 방향이다.

- high-risk command/tool에 대해 ask_user 또는 approval prompt 발생
- pending queue로 yes/no 응답 수집
- session metadata와 loop 안에서 처리

즉, 현재 Nanobot approval은 "대화형 승인"에는 충분하지만, "업무 시스템형 승인 티켓"에는 약하다.

권장 판단:

- 일반 개인 에이전트 범위에서는 현재 구조 유지가 낫다.
- Kakao/Slack/Web 간 비동기 승인 재개가 핵심이면, 그때만 persistent approval ticket 계층을 별도 추가하는 것이 맞다.

### 4.5 Session / memory 계층 비교

이 부분은 오히려 Nanobot가 AI Assistant 초안보다 더 앞서 있다.

AI Assistant 초안은 다음을 원했다.

- message history
- session state
- long-term memory
- cross-channel continuity

Nanobot는 현재 아래를 갖고 있다.

- session.messages
- JSONL session persistence
- history compaction
- Dream 기반 long-term memory editing
- GitStore 기반 memory versioning

부족한 것은 channel-agnostic identity mapping 테이블 정도다.

권장 판단:

- memory는 Nanobot 방식을 유지
- 필요 시 `channel identity map`만 얹는 편이 낫다.

#### 4.5.1 Nanobot에서 예전 정보를 다시 찾는 실제 흐름

LangGraph로 하려던 히스토리 관리 목적이 `예전 정보를 기억하고, 필요할 때 다시 찾아 쓰는 것`이었다면, Nanobot는 그 역할을 memory 계층으로 나눠서 해결한다.

기본 흐름은 아래와 같다.

1. 최근 대화는 `session.messages`에 살아 있는 short-term context로 남는다.
2. 오래된 대화는 `Consolidator`가 요약해서 `memory/history.jsonl`에 append-only로 쌓는다.
3. `Dream`이 그 요약과 `SOUL.md`, `USER.md`, `memory/MEMORY.md`를 읽고 장기 기억을 갱신한다.
4. 이후 assistant는 최근 문맥, 압축된 과거, 장기 기억을 조합해 예전 정보를 다시 참조한다.

즉, Nanobot는 하나의 giant conversation state를 계속 들고 가기보다 아래 계층을 나눠 쓴다.

- 최근 문맥: `session.messages`
- 압축된 과거 기록: `memory/history.jsonl`
- 장기 기억: `USER.md`, `memory/MEMORY.md`, `SOUL.md`
- 장기 기억 정리기: `Dream`

#### 4.5.2 시나리오 A: 사용자 선호 기억

예시:

- 사용자가 여러 번 "답변은 짧고 직접적으로 해달라"고 말한다.
- 초기에는 그 정보가 `session.messages`에만 있다.
- 시간이 지나면 오래된 대화 구간이 `history.jsonl`로 압축된다.
- 반복적으로 중요한 선호로 판단되면 `Dream`이 `USER.md`에 반영한다.
- 이후 새 대화에서는 최근 메시지에 그 말이 없어도 assistant가 그 선호를 다시 참조할 수 있다.

#### 4.5.3 시나리오 B: 프로젝트 결정 기억

예시:

- 대화 중 "n8n은 core가 아니라 optional executor로 둔다"는 결정을 내린다.
- 처음에는 최근 thread 문맥 안에 있다.
- 시간이 지나면 그 결정이 `history.jsonl`의 요약 기록으로 남는다.
- 충분히 durable한 프로젝트 사실이라면 `Dream`이 `memory/MEMORY.md`에 반영한다.
- 나중에 "왜 n8n을 core로 안 두기로 했지?"라고 물으면 assistant가 장기 기억을 근거로 다시 설명할 수 있다.

#### 4.5.4 시나리오 C: 승인 대기 작업과 최근 문맥

예시:

- 사용자가 메일 발송 같은 action을 요청한다.
- assistant가 approval을 요구하며 pending 상태가 된다.
- 이 정보는 우선 session과 pending queue에 남는다.
- 장기 기억으로 승격될 내용은 아니므로 보통 `USER.md`나 `MEMORY.md`에는 들어가지 않는다.
- 즉, approval 대기 작업은 `장기 memory`가 아니라 `recent session state`에 가까운 정보로 다뤄진다.

이 시나리오가 중요한 이유는, Nanobot가 `기억해야 할 것`과 `지금만 유지하면 되는 상태`를 분리한다는 점을 보여주기 때문이다.

#### 4.5.5 요약

정리하면, LangGraph에서 기대했던 "히스토리를 들고 가는 기능"은 Nanobot에서 아래처럼 대응된다.

- 최근 대화 유지: `session.messages`
- 과거 대화 압축 보관: `memory/history.jsonl`
- 장기 선호/사실 반영: `Dream` + `USER.md` + `memory/MEMORY.md` + `SOUL.md`
- 장기 기억 변경 이력: `GitStore`

따라서 `예전 정보를 기억하고 다시 찾는 목적` 자체는 Nanobot에서 이미 상당 부분 구현돼 있다고 보는 편이 맞다. 부족한 쪽은 memory 그 자체보다, 복잡한 workflow state를 명시적으로 다루는 계층이다.

## 5. Nanobot가 이미 더 나은 부분

초기 AI Assistant 설계와 비교했을 때, 현재 Nanobot가 더 성숙하거나 실전적인 부분도 있다.

### 5.1 Provider / local runtime 유연성

초기 설계는 Ollama, LM Studio, MLX를 고려했지만 추상화 계층이 아직 개념 수준이었다. Nanobot는 이미 다수 provider와 local provider를 실제 config/runtime로 다룬다.

### 5.2 Memory 운영 모델

AI Assistant 초안은 memory 후보 저장과 자동 반영 기준을 고민하는 단계였다. Nanobot는 Consolidator + Dream + versioned memory로 훨씬 구체적이다.

### 5.3 채널 폭과 gateway 운영

AI Assistant 초안은 Web/Slack/Kakao 재사용을 목표로 했지만, 실제 지원 채널 폭은 제한적이었다. Nanobot는 현재 Telegram, Discord, Slack, Email, WeChat, WhatsApp 등 훨씬 넓은 채널 기반을 가지고 있다.

### 5.4 MCP 수용성

AI Assistant 초안은 MCP를 확장 방향으로 언급했지만, Nanobot는 이미 MCP를 runtime 연결 구조 안에 넣고 있다.

## 6. 현재 빠진 것과 그대로는 구현하기 어려운 것

### 6.1 Kakao 공식 채널

현재 Nanobot에는 Kakao 공식 채널 / OpenBuilder 경로가 없다. 이 부분은 새 channel adapter를 만들어야 한다.

필요 작업:

- Kakao webhook 인증 / payload normalize
- Kakao card/simpleText 응답 format
- Kakao 채널 session key 설계
- gateway와의 연결 정책 정리

### 6.2 Gmail / Calendar / Notion 업무 자동화

Nanobot는 Email channel은 있지만, AI Assistant 원안의 Gmail summary/draft/send, Calendar create/update/delete, Notion page workflow 같은 업무 자동화 계층은 현재 없다.

중요 포인트:

- Email channel은 메일을 채널로 받는 구조이지, Gmail personal assistant workflow와는 다르다.
- Calendar / Notion 도메인은 현재 first-class skill/tool로 정리되어 있지 않다.

### 6.3 n8n 중심 orchestration

Nanobot는 workflow engine 없이도 돌아가는 것이 장점이다. 따라서 n8n을 core dependency로 삼으면 Nanobot의 경량성과 단순한 운영 모델이 약해질 수 있다.

### 6.4 PostgreSQL / Redis 기반 운영 상태

AI Assistant 원안은 message/state/task/approval을 DB와 cache로 나누려 했다. Nanobot는 workspace와 file/session 중심이다.

이건 단순히 "빠졌다"라기보다 철학 차이다.

- Nanobot 장점: 단순, 로컬 친화적, 운영 부담 적음
- AI Assistant 장점: 비동기 작업, 다중 사용자, 복잡한 상태 머신에 유리

개인용 self-hosted agent라면 Nanobot 기본값이 더 적합하다.

### 6.5 macOS 로컬 자동화 계층

AI Assistant 초안은 AppleScript / Playwright / Open Interpreter를 1급 도구로 보았다. 현재 Nanobot 소스 기준으로는 이 부분이 독립 계층으로 정리되어 있지 않다.

Nanobot에 이 기능을 넣으려면 가장 자연스러운 경로는 아래 중 하나다.

- MCP server로 감싸기
- custom tool plugin으로 추가
- host-side helper를 channel/tool runtime과 분리해 연결

### 6.6 구조화 extraction JSON 우선 파이프라인

AI Assistant 원안은 자유 답변 이전에 extraction JSON을 공통 envelope로 만들고, 그 결과를 기준으로 automation을 실행하려 했다. 현재 Nanobot는 이 철학을 전면 채택하고 있지 않다.

이는 현재 Nanobot의 강점과도 충돌할 수 있다.

- 장점: 도메인 자동화에는 좋음
- 단점: 일반 대화형 agent loop가 무거워질 수 있음

따라서 전역 강제보다 domain-specific preprocessor로 제한하는 것이 적합하다.

## 7. Nanobot 기준 권장 개발 방향

### 7.1 권장 방향: "Nanobot를 기반으로 AI Assistant 일부를 흡수"

가장 현실적인 방향은 AI Assistant 원안을 Nanobot 안으로 흡수하되, Nanobot의 핵심 구조는 유지하는 것이다.

권장 원칙:

- gateway / WebUI / serve / session / memory / skill / MCP는 그대로 유지
- 새 기능은 channel plugin, tool plugin, MCP server, docs-driven runtime policy로 확장
- n8n, DB, workflow engine은 core가 아니라 optional integration으로 둔다.

### 7.2 단계별 권장 순서

#### 1단계: low-risk 흡수

- Kakao 채널 도입 여부 검토
- structured extraction을 특정 도메인에만 도입
- WebUI에 assistant settings / action panel / automation status 보강
- channel identity continuity 규칙 정리

#### 2단계: automation adapter 도입

- Gmail / Calendar / Notion을 바로 n8n에 묶기보다 domain tool interface 먼저 설계
- executor는 MCP, webhook, direct API 중 교체 가능하게 유지
- approval을 tool metadata 기준으로 확장

#### 3단계: 필요 시 persistent orchestration 추가

- truly async 작업이 많아질 때만 persistent task store 도입
- approval ticket이 cross-channel resume 요구를 가질 때만 DB 계층 추가
- multi-user / team assistant로 확장할 때만 PostgreSQL / Redis를 검토

## 8. 권장하지 않는 방향

아래 방향은 현재 Nanobot의 장점을 약화시킬 가능성이 크므로 권장하지 않는다.

- Open WebUI를 다시 주 UI로 두는 것
- LangGraph를 core loop 위에 또 얹는 것
- n8n을 필수 중심 실행 계층으로 만드는 것
- 초기에 PostgreSQL / Redis를 core dependency로 도입하는 것
- 모든 일반 대화에 extraction JSON 강제 파이프라인을 넣는 것

## 9. 실무 판단

실무적으로 보면, AI Assistant 원안은 "개인 비서 자동화 허브"에 가깝고, Nanobot는 "경량 개인 에이전트 런타임"에 가깝다.

따라서 앞으로의 방향은 두 가지 중 하나로 명확히 정해야 한다.

### 방향 A: Nanobot를 유지하고 assistant 기능을 선택 흡수

이 방향이 현재 기준에서는 가장 추천된다.

적합한 경우:

- 단일 사용자 중심
- 로컬 self-hosted 우선
- 채널/메모리/도구 기반 agent를 선호
- 점진적으로 Kakao나 업무 자동화만 추가하고 싶음

### 방향 B: 업무 자동화 허브를 새로 구축하고 Nanobot는 참고만 사용

이 방향은 아래 조건일 때만 더 적합하다.

- Kakao 중심 비서가 최우선
- Gmail/Calendar/Notion automation이 핵심 제품 요구
- approval ticket / async task / workflow engine이 주축
- DB/cache 기반 상태 관리가 필수

즉, "Nanobot에 AI Assistant 전부를 얹는가"보다, "Nanobot를 core runtime으로 쓰고 AI Assistant의 일부 기능만 가져오는가"가 더 맞는 질문이다.

현재 판단으로는 후자가 맞다.

## 10. 최종 제안

현재 Nanobot 기준 최종 제안은 아래와 같다.

1. Nanobot를 기본 런타임으로 유지한다.
2. AI Assistant 초안 중 `channel-independent assistant UX`, `local/external model routing`, `approval`, `skill/MCP expansion`, `session continuity`만 우선 흡수한다.
3. Kakao와 SaaS automation은 별도 integration layer로 설계한다.
4. n8n은 필수 core가 아니라 optional executor로 둔다.
5. PostgreSQL / Redis는 초기 도입 대상에서 제외하고, multi-user or long-running workflow 요구가 생길 때만 검토한다.
6. macOS 자동화는 AppleScript 전용 core layer를 만들기보다 MCP 또는 host-side tool adapter로 넣는다.

이 방향이면 Nanobot의 장점인 경량성, 단순 운영, self-hosted 친화성은 유지하면서도, AI Assistant 원안의 실질 가치가 있는 부분만 점진적으로 가져올 수 있다.