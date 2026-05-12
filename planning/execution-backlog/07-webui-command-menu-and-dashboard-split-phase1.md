# 07 WebUI Command Menu And Dashboard Split Phase 1

> 용어 메모: 이 문서는 현재 WebUI 용어 기준을 따른다. 사용자 노출 표현은 [docs/guide/10-webui-terminology.md](../../guide/10-webui-terminology.md) 를 우선 기준으로 본다.

## 1. 문서 목적

이 문서는 다음 WebUI 정리 작업을 하나의 phase-1 slice 로 묶어, 바로 착수 가능한 작업 방안으로 정리한 실행 문서다.

이번 단계의 목표는 세 가지다.

- 채팅 입력창에서 `/` 입력 시 보이는 slash command 안내가 현재 runtime 에 추가된 명령 집합과 어긋나지 않게 맞추기
- 사이드바의 `대시보드` 와 `새 채팅` 이 사실상 같은 화면으로 보이는 현재 구조를 분리하기
- 대시보드 feed box 제목 앞에 의미에 맞는 아이콘을 붙여 scanability 를 높이기

## 2. 왜 지금 필요한가

현재 WebUI 는 아래 두 문제가 동시에 있다.

1. slash command palette 는 존재하지만, 최근 추가된 명령 항목이 palette/i18n 쪽에 충분히 반영되지 않아 사용자가 `/` 입력만으로 새 기능을 발견하기 어렵다.
2. `대시보드` 와 `새 채팅` 이 모두 `session 없음` 상태를 공유해 같은 화면처럼 보인다. 그 결과 대시보드에 있어야 할 카드와 새 채팅 진입용 quick action strip 이 한 surface 에 섞인다.

추가로 현재 대시보드 feed box 는 제목이 텍스트만 있어 정보 종류를 빠르게 구분하기 어렵다. 따라서 이번 slice 는 `명령 발견성`, `home/new-chat 정보 구조`, `dashboard scanability` 를 함께 정리하는 것이 맞다.

## 3. phase-1 범위

포함할 것:

- slash command palette 의 source-of-truth 와 WebUI 표시 항목 정렬
- `대시보드` 와 `새 채팅` 의 view state 분리
- `새 채팅` 에서만 composer 아래 quick action card strip 노출
- `대시보드` 에서는 composer 아래 quick action strip 제거
- dashboard section title 앞 아이콘 추가
- touched slice 테스트, i18n, build 검증

제외할 것:

- dashboard 전체 정보구조 재설계
- command backend contract 자체의 대규모 개편
- 새 dashboard data source 추가
- sidebar/navigation 전체 redesign
- mobile 전용 별도 IA 재설계

## 4. 핵심 구현 대상

### 4.1 slash command palette 동기화

최소 목표:

- WebUI 가 `/api/commands` 또는 현재 slash command payload 기준 최신 명령 집합을 빠뜨리지 않게 반영
- 신규 명령의 title/description 이 palette 에서 바로 보이게 i18n registry 정리
- command metadata 와 palette fallback wording drift 방지

현재 코드 anchor:

- `source/webui/src/lib/api.ts`
- `source/webui/src/components/thread/ThreadComposer.tsx`
- `source/webui/src/i18n/locales/en/common.json`
- `source/webui/src/i18n/locales/ko/common.json`
- `source/webui/src/tests/api.test.ts`
- `source/webui/src/tests/thread-composer.test.tsx`

### 4.2 대시보드와 새 채팅 surface 분리

최소 목표:

- `대시보드` 는 owner-wide summary / feed card 전용 surface 로 유지
- `새 채팅` 은 빈 대화 시작 surface 로 분리
- 새 채팅 진입 시 empty composer + 하단 quick action strip 만 보이고, dashboard card 는 보이지 않게 함
- Home/Dashboard 와 New Chat 이 같은 state (`activeKey = null`) 를 공유하지 않도록 명시적 view state 추가

현재 코드 anchor:

- `source/webui/src/App.tsx`
- `source/webui/src/components/Sidebar.tsx`
- `source/webui/src/components/thread/ThreadShell.tsx`
- `source/webui/src/tests/app-layout.test.tsx`
- `source/webui/src/tests/thread-shell.test.tsx`

### 4.3 composer 하단 quick action strip 분리

현재 quick action card strip 은 session 없는 상태에서 dashboard 와 함께 섞여 보일 수 있다. phase-1 에서는 아래처럼 분리한다.

- `대시보드`: feed box 만 노출, composer 하단 quick action strip 제거
- `새 채팅`: composer + quick action strip 유지
- 기존 quick action prompt 및 image mode 진입 흐름은 유지

현재 코드 anchor:

- `source/webui/src/components/thread/ThreadShell.tsx`
- `source/webui/src/i18n/locales/en/common.json`
- `source/webui/src/i18n/locales/ko/common.json`
- `source/webui/src/tests/thread-shell.test.tsx`

### 4.4 dashboard feed title iconography

최소 목표:

- `우선순위 대기열`, `오늘 브리프`, `빠른 작업`, `최근 결과`, `연결된 채널` 제목 앞에 의미상 맞는 아이콘 추가
- 아이콘은 title text 를 대체하지 않고 보조만 수행
- dark/light theme 에서 과하지 않은 시각적 대비 유지
- mobile 과 desktop 모두 density 를 해치지 않게 작은 leading icon 패턴으로 통일

현재 코드 anchor:

- `source/webui/src/components/home/AssistantDashboard.tsx`
- `source/webui/src/tests/app-layout.test.tsx`

## 5. 작업 분해

### Step 1. slash command source-of-truth 재확인

- runtime slash command endpoint 가 현재 어떤 명령 metadata 를 내리는지 다시 확인
- palette 가 endpoint 항목을 누락하는지, 아니면 i18n key 만 빠진 상태인지 구분
- 신규 명령 추가 시 fallback title/description 전략을 정리

### Step 2. explicit home mode 분리

- `dashboard` 와 `new-chat` 를 구분하는 명시적 state 설계
- `Sidebar` 의 Home 버튼은 `dashboard`, New chat 버튼은 `new-chat` 으로 연결
- `onNewChat()` 가 더 이상 단순 `activeKey = null` 과 같은 의미가 아니게 정리

### Step 3. ThreadShell empty-state 분리

- `session 없음` 이더라도 `dashboard` 와 `new-chat` 렌더링을 분기
- `dashboard` 에서는 `AssistantDashboard` 만 노출
- `new-chat` 에서는 hero composer + quick action strip 만 노출

### Step 4. dashboard title icon 추가

- section title 공통 helper 또는 small title row component 도입 여부 판단
- icon set 은 lucide 기존 사용 범위 안에서 고정
- 아이콘이 CTA, badge 와 시각적으로 충돌하지 않는지 확인

### Step 5. 테스트 및 live 검증 계획 정리

권장 검증 순서:

1. `source/webui/src/tests/api.test.ts`
2. `source/webui/src/tests/thread-composer.test.tsx`
3. `source/webui/src/tests/app-layout.test.tsx`
4. 필요 시 `source/webui/src/tests/thread-shell.test.tsx`
5. `npm --prefix /Volumes/ExtData/Nanobot/source/webui run build`

## 6. 실제 코드 작업 backlog

### 6.1 slash palette slice

- commands endpoint payload 와 palette filter 기준 정렬
- 신규 명령 i18n key 보강
- palette selection / insertion 테스트 갱신

### 6.2 shell state slice

- `App.tsx` 에 `dashboard` 와 `new-chat` 분리 state 추가
- `Sidebar.tsx` 의 Home / New Chat action 분리
- blank start page 관련 테스트 기대값 갱신

### 6.3 ThreadShell empty surface slice

- dashboard surface 와 new-chat surface 를 분리 렌더링
- quick action strip 을 new-chat 전용으로 제한
- 기존 첫 메시지 전송 시 lazy chat creation 흐름 유지

### 6.4 dashboard icon slice

- feed section title icon map 추가
- title row UI 정리
- icon + title alignment regression 확인

### 6.5 validation slice

- desktop/mobile narrow width 에서 dashboard 와 new-chat 이 시각적으로 섞이지 않는지 확인
- slash palette 에 신규 명령이 실제로 보이는지 확인
- dashboard title icon 이 feed density 를 해치지 않는지 live 확인

## 7. 완료 기준

1. `/` 입력 시 WebUI slash command palette 에 현재 추가된 명령 항목이 빠지지 않고 노출된다.
2. 사이드바 `대시보드` 와 `새 채팅` 은 서로 다른 surface 로 보인다.
3. `새 채팅` 에서만 composer 아래 quick action strip 이 나타난다.
4. `대시보드` feed box 제목 앞에 의미에 맞는 아이콘이 노출된다.

## 8. 다음 단계 연결

- 구현 시작 전 `planning/todo.md` 에 active slice 로 반영
- 구현 후 live 결과는 `operations/status-summary/` 날짜형 메모에 기록