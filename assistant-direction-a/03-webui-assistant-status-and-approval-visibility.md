# Nanobot WebUI Assistant Status And Approval Visibility

## 상태 메모

- 이 문서는 WebUI assistant status/approval visibility 의 초기 설계 reference 다.
- 2026-05-01 기준 구현 및 검증 source of truth 는 `docs/todo.md` section 5.1 과 `docs/execution-backlog/01-owner-aware-control-surface-phase1.md` 다.
- 따라서 본문은 남은 구현 backlog 가 아니라, 현재 UI 구조가 어떤 문제 정의에서 나왔는지 설명하는 historical context 로 읽는다.

## 1. 문서 목적

이 문서는 방향 A 기준에서 WebUI에 어떤 assistant 상태를 보여줘야 하는지, 그리고 approval / action visibility를 어떻게 다룰지 정리한다.

목표는 단순히 UI를 더 많이 보여주는 것이 아니라, 사용자가 아래 질문에 즉시 답할 수 있게 만드는 것이다.

- 지금 어떤 assistant 상태로 대화 중인가?
- 어떤 모델과 target이 사용 중인가?
- 지금 멈춘 이유가 승인 때문인가?
- 자동화 작업이 진행 중인가, 대기 중인가, 실패했는가?

## 2. 문제 정의

현재 Nanobot는 대화와 모델 선택은 비교적 잘 제공하지만, assistant가 실제 action을 하려는 순간 아래 정보가 흐려질 수 있다.

- 현재 active target
- session override 여부
- approval pending 여부
- tool/action 실행 중 상태
- channel continuity 상태

방향 A에서는 assistant가 단순 채팅이 아니라 개인 비서 역할을 하므로, 이 정보들을 thread 흐름을 깨지 않는 방식으로 보여줘야 한다.

## 3. WebUI가 보여줘야 할 핵심 상태

### 3.1 Assistant runtime 상태

최소 노출 항목:

- active target
- provider
- model
- locked 여부
- response footer mode

의도:

- 사용자가 현재 local model인지, smart-router인지, 특정 remote model인지 즉시 알 수 있어야 한다.

### 3.2 Session 상태

최소 노출 항목:

- 현재 session key 또는 session label
- channel 종류
- 외부 채널 연동 여부
- 최근 activity 시간

의도:

- 지금 보고 있는 thread가 WebUI native인지, Telegram mirror인지, Slack/Email 연동 세션인지 구분 가능해야 한다.

### 3.3 Action 상태

최소 노출 항목:

- idle
- running
- waiting-approval
- blocked
- completed
- failed

의도:

- assistant가 단순 응답 생성 중인지, 실제 외부 action을 수행 중인지 사용자에게 명확히 보여줘야 한다.

### 3.4 Approval 상태

최소 노출 항목:

- approval 필요 여부
- approval 대상 action 요약
- 왜 승인이 필요한지
- 어디서 승인할 수 있는지

의도:

- assistant가 멈춘 이유가 시스템 오류인지, 사용자 승인 대기인지 혼동되지 않게 한다.

## 4. 권장 UI 표면

## 4.1 Thread header

Thread header에는 가볍고 지속적인 상태만 둔다.

권장 항목:

- active target badge
- channel badge
- approval pending badge
- automation active badge

두지 않는 것이 좋은 것:

- 장문의 설명
- 모든 tool 내부 상태
- debug용 세부 메타 전체

## 4.2 Thread inline status block

assistant action이 발생했을 때는 thread 안에 상태 블록을 삽입하는 방식이 적절하다.

예시 상태:

- `일정 생성 준비 중`
- `Gmail 초안 작성 중`
- `승인 대기: 메일 전송`
- `실패: 외부 API 인증 필요`

이 블록은 일반 assistant 메시지와 구분되되, 너무 독립된 시스템 로그처럼 보여서는 안 된다.

## 4.3 Settings assistant section

Settings에는 장기 설정만 둔다.

권장 항목:

- 기본 target / provider / model
- locked state 설명
- footer mode 기본값
- automation 기본 정책 on/off
- approval 정책 기본값

Settings에 두지 않는 것이 좋은 것:

- 현재 turn의 일시적 action 상태
- pending approval 상세 목록
- 특정 tool 실행 로그

## 4.4 Optional side panel

assistant 기능이 늘어나면 side panel 또는 drawer가 유용할 수 있다.

권장 용도:

- pending approvals 목록
- recent actions 목록
- current session continuity 요약
- active automation summary

단, 초기에는 본격적인 작업 패널보다 compact summary를 먼저 도입하는 편이 낫다.

## 5. Approval visibility 정책

### 5.1 approval이 필요한 순간

approval visibility는 아래 작업에서 우선 필요하다.

- 파일 수정 또는 삭제
- shell/exec 계열 실행
- 외부 메시지 발송
- Email/Gmail 발송
- Calendar 일정 생성/변경
- Kakao 응답 발송 또는 외부 서비스 변경
- macOS local automation 실행

### 5.2 approval 표시 최소 정보

WebUI는 approval 대기 시 최소한 아래를 보여줘야 한다.

- action 제목
- action 요약
- 이유
- 영향 범위
- 승인 / 거절 가능한 위치

### 5.3 approval 상태 전이

권장 상태 전이는 아래와 같다.

- proposed
- waiting-approval
- approved
- rejected
- expired
- resumed
- failed

초기 구현에서는 전부를 UI에 노출하지 않아도 되지만, runtime metadata는 이 수준을 염두에 두고 잡는 편이 좋다.

## 6. Action visibility 정책

### 6.1 일반 대화와 action의 구분

assistant의 응답과 assistant의 action은 같은 thread 안에 있더라도 구분 가능해야 한다.

권장 구분 방식:

- 일반 응답: 기존 메시지 bubble
- action status: compact inline block
- approval request: emphasized action block
- completion/failure: result summary block

### 6.2 action 상태의 수준

초기에는 아래 두 수준만 있어도 충분하다.

- user-facing summary
- debug/internal metadata

WebUI는 summary만 우선 보여주고, 필요 시 metadata는 개발용으로 숨긴다.

## 7. Single-user continuity와의 연결

방향 A에서는 approval과 상태 가시성이 single-user continuity와 연결된다.

예를 들면 아래가 중요하다.

- Telegram에서 시작한 task를 WebUI에서 볼 수 있는가
- WebUI에서 pending approval을 확인할 수 있는가
- 외부 채널 turn이 현재 assistant state와 어떻게 연결되는가

따라서 WebUI status 설계는 단순 UI 문제가 아니라 continuity 정책의 일부다.

## 8. 제안하는 구현 순서

### 8.1 1차 구현

- header badge 수준의 compact assistant status
- active target / channel / approval pending 표시
- waiting-approval inline block 정의

### 8.2 2차 구현

- recent actions summary 추가
- automation active state 표시
- session continuity 표시 보강

### 8.3 3차 구현

- pending approvals panel 또는 drawer
- multi-channel continuity 상태 요약
- domain별 action result card 정리

## 9. 비목표

이번 문서 범위에서 비목표는 아래다.

- full audit log viewer
- admin dashboard
- 조직용 task board
- multi-user approval inbox
- workflow builder UI

## 10. 구현 체크리스트

WebUI 변경 전 아래를 체크한다.

1. 이 상태가 사용자에게 실제 의사결정을 돕는가?
2. 이 정보는 Settings가 아니라 thread 안에 있어야 하는가?
3. 단일 사용자 기준에서 너무 과도한 시스템 UI가 아닌가?
4. channel continuity를 설명하는 데 도움이 되는가?
5. optional integration이 없어도 성립하는 UI인가?

## 11. 최종 요약

방향 A에서 WebUI는 단순 채팅 화면이 아니라, `내 assistant가 지금 무엇을 하려는지 보여주는 표면`이 되어야 한다.

우선순위는 아래다.

1. active target과 approval 상태를 명확히 보여준다.
2. action 진행 상태를 thread 안에서 이해 가능하게 만든다.
3. 외부 채널과의 continuity를 보조하는 최소 상태를 추가한다.
4. 초기에는 compact하고 단순하게 시작한다.

이 원칙을 따르면 WebUI는 복잡한 운영 콘솔이 아니라, 단일 사용자 assistant control surface로 자연스럽게 확장될 수 있다.