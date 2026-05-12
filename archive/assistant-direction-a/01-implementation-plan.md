# Nanobot 방향 A 구현 진행 방안

## 상태 메모

- 이 문서는 초기 방향 A 구현 계획의 상위 설계 문서다.
- 2026-05-01 기준 active backlog source of truth 는 `docs/planning/todo.md` section 5 와 `docs/planning/execution-backlog/*.md` 로 이동했다.
- 본 문서는 남은 active checklist 를 추적하지 않고, 초기 구현 스트림이 어떤 논리로 나뉘었는지 설명하는 reference 로 유지한다.
- 현재 흡수 상태는 아래와 같다.
  - Assistant UX / continuity / Gmail read-only+draft / Gmail send approval 은 실행 backlog 로 흡수되어 구현 완료
  - Kakao 는 feasibility 및 경계 설계 reference 로만 남아 있고 active implementation 으로는 승격되지 않음

## 1. 문서 목적

이 문서는 [AI Assistant 초기 아키텍처의 Nanobot 수용성 검토](./00-architecture-fit-review.md)에서 정리한 `방향 A: Nanobot를 유지하고 assistant 기능을 선택 흡수`를 실제 작업 기준으로 풀어낸 실행 계획 문서다.

이 문서는 아래 전제를 고정한다.

- 로컬 self-hosted 우선
- 단일 사용자 기준
- Nanobot 기존 runtime 철학 유지
- Open WebUI, LangGraph, n8n, PostgreSQL, Redis를 core dependency로 두지 않음
- AI Assistant 초기 구상 중 실질 가치가 높은 부분만 Nanobot 구조에 맞게 흡수

즉, 이 문서는 "새 시스템을 다시 만드는 계획"이 아니라, "현재 Nanobot를 중심으로 assistant 기능을 점진 확장하는 계획"이다.

## 2. 목표 상태

방향 A 기준 최종 목표는 아래와 같다.

1. 사용자는 Nanobot를 단일 개인 assistant 런타임으로 계속 사용한다.
2. WebUI, gateway, OpenAI-compatible API, memory, MCP, skill 구조는 유지한다.
3. 모델 선택, 세션 지속성, approval, personal automation, channel continuity 같은 assistant 경험은 더 강화한다.
4. Kakao 및 SaaS automation은 optional integration으로 넣는다.
5. 복잡한 workflow engine이나 DB 기반 state machine 없이도 개인 비서 수준의 자동화가 가능하도록 만든다.

## 3. 설계 원칙

### 3.1 유지할 것

- `source/nanobot/**` runtime 구조
- WebUI + gateway + `nanobot serve` 삼각 구조
- file/session 중심 상태 관리
- Dream 기반 memory 운영
- MCP 및 built-in skill 확장 방식
- interactive approval

### 3.2 새로 넣을 때의 원칙

- 새 기능은 core rewrite가 아니라 작은 확장 단위로 넣는다.
- 가능한 경우 `channel plugin`, `tool plugin`, `MCP server`, `helper module`로 구현한다.
- 일반 대화 경로를 무겁게 만드는 전역 orchestration은 피한다.
- 단일 사용자 기준에서 필요한 것만 먼저 넣고, multi-user 전제 기능은 뒤로 미룬다.

### 3.3 하지 않을 것

- Open WebUI를 새 메인 UI로 도입하지 않음
- LangGraph를 core agent loop 위에 재구축하지 않음
- n8n을 필수 runtime dependency로 두지 않음
- PostgreSQL / Redis를 초기 필수 계층으로 도입하지 않음
- 모든 요청에 extraction JSON 강제 파이프라인을 두지 않음

## 4. 우선 구현 범위

방향 A에서 우선 흡수할 범위는 아래 다섯 축이다.

### 4.1 Assistant UX 축

- WebUI에서 assistant 설정과 상태를 더 명확히 노출
- 모델 선택, active target, automation 상태, approval 대기 상태를 일관되게 표시
- 채널이 달라도 같은 assistant라는 감각을 유지할 수 있게 session continuity를 보강

### 4.2 Model routing 축

- local / remote / smart-router 기반 모델 선택을 더 안정화
- 단일 사용자 기준의 기본 모델 정책을 명확히 문서화
- session override와 global default의 충돌 규칙을 정리

### 4.3 Approval 축

- 현재 interactive approval을 유지하되, assistant 작업에 필요한 metadata를 늘림
- 어떤 작업이 왜 승인 대상인지 더 명확히 노출
- WebUI에서 승인 대기 작업을 더 확인하기 쉽게 만든다.

### 4.4 Personal automation 축

- Gmail / Calendar / Notion / browser / macOS automation은 core가 아니라 optional tool integration으로 설계
- domain-specific tool contract를 먼저 만들고, 실제 executor는 MCP 또는 외부 adapter로 연결

### 4.5 Channel continuity 축

- 단일 사용자 기준으로 여러 채널 세션을 더 잘 이어서 볼 수 있게 함
- 필요한 경우 `channel identity map` 또는 그에 준하는 lightweight mapping 규칙을 추가
- 다중 사용자용 권한/조직 모델은 도입하지 않음

## 5. 작업 스트림

## 5.1 스트림 A: Assistant UX 정리

목표:
WebUI와 설정 표면을 assistant 중심으로 정리해, 모델/상태/자동화/승인 흐름이 사용자에게 명확하게 보이도록 만든다.

세부 작업:

- Settings에 assistant 관련 섹션 구조 재정리
- active target, provider, model, locked state 표시 일관화
- approval pending 상태 표시 검토
- automation enabled/disabled 상태 표시 검토
- session continuity 관련 상태 문구 정리

완료 기준:

- 사용자는 WebUI만 보고 현재 assistant 상태를 이해할 수 있다.
- 설정 변경 가능/불가 이유가 UI에 드러난다.

## 5.2 스트림 B: 단일 사용자 운영 정책 고정

목표:
로컬 self-hosted, 단일 사용자 기준에서 불필요한 다중 사용자 복잡도를 배제하고 runtime 기본값을 명확히 한다.

세부 작업:

- docs에 단일 사용자 운영 원칙 명시
- session identity 기본 규칙 정의
- local memory와 personal profile 경계 정리
- approval과 automation의 소유자 개념을 단일 사용자 기준으로 단순화

완료 기준:

- 구현 문서와 설정 문서에서 다중 사용자 전제를 제거하거나 후순위로 명시한다.
- 신규 기능 설계 시 user/team/account abstraction을 억지로 넣지 않는다.

## 5.3 스트림 C: Assistant automation 도구 계층 설계

목표:
Gmail, Calendar, Notion, browser, macOS automation을 Nanobot core가 아닌 assistant tool layer로 정리한다.

세부 작업:

- 공통 `assistant automation task` 개념 정의
- domain별 tool contract 초안 작성
- executor 종류 구분
  - MCP-backed
  - direct API-backed
  - local host helper-backed
- approval 필요 조건 정의
- 실패 시 fallback 응답 형식 정의

완료 기준:

- 실제 integration 구현 전에도 tool contract 문서가 존재한다.
- 각 automation 도메인이 core runtime 변경 없이 확장 가능함이 명확하다.

## 5.4 스트림 D: Channel continuity 강화

목표:
단일 사용자가 WebUI, Slack, Telegram, 이후 Kakao 등 여러 채널을 써도 assistant 경험이 끊기지 않게 한다.

세부 작업:

- channel session key 정책 정리
- channel identity continuity를 위한 lightweight mapping 설계
- WebUI에서 외부 채널 세션 표시 방식 검토
- approval / memory / status가 채널별로 어떻게 이어지는지 규칙화

완료 기준:

- 같은 사용자라는 전제 아래 채널 이동 시 기억/상태가 덜 끊긴다.
- session continuity 규칙이 문서화된다.

## 5.5 스트림 E: Kakao integration 준비

목표:
Kakao를 당장 core에 넣기보다, 방향 A에 맞는 optional channel integration으로 준비한다.

세부 작업:

- Kakao webhook payload 정규화 요구 정리
- Kakao session key 정책 초안 작성
- 응답 format 제약 정리
- approval과 quick reply 상호작용 방식 검토
- channel plugin vs 별도 adapter 경계 결정

완료 기준:

- Kakao를 언제든 다음 단계에서 구현할 수 있는 설계 문서가 준비된다.
- Nanobot core가 Kakao 전용 구조에 잠식되지 않는다.

## 6. 단계별 실행 순서

## 6.1 1단계: 구조 고정과 문서 선행

목표:
방향 A 기준에서 무엇을 유지하고 무엇을 optional로 둘지 먼저 고정한다.

이번 단계 작업:

- 방향 A 구현 계획 문서 작성
- 단일 사용자 / 로컬 self-hosted 운영 원칙 문서화
- assistant automation 범위를 문서 기준으로 분리
- Kakao 및 SaaS automation을 optional integration으로 선언

산출물:

- 본 문서
- 후속 세부 설계 문서

## 6.2 2단계: WebUI / 상태 표면 보강

목표:
사용자 관점에서 assistant 상태와 실행 가능 기능을 잘 보이게 만든다.

작업 후보:

- Settings assistant 섹션 보강
- active target / approval / automation status UI 추가
- session continuity 관련 UI 보강

## 6.3 3단계: automation tool contract 도입

목표:
Gmail, Calendar, Notion, browser, macOS 계열을 도메인별 tool contract로 정리한다.

작업 후보:

- assistant automation base interface 정의
- domain별 request/result schema 정의
- approval metadata 정의
- MCP 또는 helper 실행 경로 설계

## 6.4 4단계: 특정 도메인 1개씩 실제 연결

목표:
모든 integration을 한 번에 넣지 말고, 가장 가치가 높은 도메인부터 하나씩 연결한다.

권장 순서:

1. Gmail 또는 Email assistant workflow
2. Calendar assistant workflow
3. browser automation helper
4. Kakao channel integration
5. Notion integration

## 7. 즉시 진행할 작업 제안

방향 A로 실제 작업을 시작한다면, 다음 순서를 권장한다.

### 7.1 바로 시작할 문서 작업

- assistant automation architecture 초안 작성
- channel continuity / identity mapping 메모 작성
- Kakao integration requirement 문서 작성
- single-user operating policy 문서 작성

### 7.2 바로 시작할 구현 작업

- WebUI assistant status 표면 정의
- approval pending 상태 노출 범위 정리
- automation task metadata 구조 초안 작성

### 7.3 첫 구현 대상으로 적합한 항목

- Gmail summary/draft 수준의 assistant tool
- Calendar read/create 수준의 assistant tool
- WebUI approval/status 패널

이 세 가지는 방향 A와 잘 맞고, n8n/DB 없이도 단계적으로 넣기 쉽다.

## 8. 비목표

이번 방향 A 진행에서는 아래를 목표로 잡지 않는다.

- multi-user team workspace
- 조직용 RBAC
- 중앙 DB 기반 approval ticket 시스템
- 범용 workflow engine 구축
- Open WebUI 재도입
- LangGraph 중심 재구현

## 9. 검증 기준

방향 A 관련 작업은 아래 기준으로 검증한다.

### 9.1 문서 검증

- 새로운 기능이 Nanobot core 철학과 충돌하지 않는지 확인
- optional integration과 core runtime 경계가 문서에 명확한지 확인

### 9.2 구현 검증

- source tree 기준 focused test 우선
- WebUI 변경 시 touched slice 테스트 + build
- backend 변경 시 focused pytest 우선
- live runtime 반영이 필요한 경우 launchd 기준 검증

### 9.3 운영 검증

- 단일 사용자 기준 UX가 단순해졌는지 확인
- 설정과 실제 runtime behavior가 어긋나지 않는지 확인
- optional integration이 비활성일 때 core 기능에 영향이 없는지 확인

## 10. 최종 방향 정리

방향 A에서 중요한 것은 기능을 많이 넣는 것이 아니라, Nanobot를 assistant로 더 일관되게 만드는 것이다.

정리하면 아래 순서를 따른다.

1. Nanobot core는 유지한다.
2. 단일 사용자 / 로컬 self-hosted 기준을 먼저 고정한다.
3. assistant UX, approval, session continuity, model routing을 먼저 다듬는다.
4. automation은 tool/MCP/helper 계층으로 분리한다.
5. Kakao와 SaaS integration은 optional로 넣는다.
6. DB/workflow engine은 정말 필요해질 때만 뒤에 검토한다.

이 방향이면 초기 AI Assistant의 목적을 잃지 않으면서도, 현재 Nanobot의 강점인 경량성, 운영 단순성, self-hosted 친화성을 유지할 수 있다.