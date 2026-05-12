# 06. Dashboard Reliability

## 목표

Dashboard 를 단순 home 화면이 아니라 pending approval, blocked task, linked session 을 안전하게 해결하는 운영 surface 로 만든다.

## 수정 대상 파일

1. `source/webui/src/App.tsx`
2. `source/webui/src/components/Sidebar.tsx`
3. `source/webui/src/components/home/AssistantDashboard.tsx`
4. `source/webui/src/components/thread/ThreadShell.tsx`
5. `source/webui/src/tests/app-layout.test.tsx`
6. `source/webui/src/tests/thread-shell.test.tsx`

## 현재 관찰

Playwright 에서 snapshot 에는 `Dashboard` 텍스트가 보였지만, role 기반 클릭은 timeout 됐다.

가능한 원인:

1. sidebar collapsed/hidden state
2. button accessible name 이 불안정함
3. mobile/desktop sidebar 중 다른 surface 가 활성화됨
4. Playwright page focus 또는 stale overlay

## 구현 작업

### 1. Dashboard button accessibility 고정

`Sidebar` 의 Dashboard 버튼에 명시적 aria-label 을 둔다.

```tsx
<Button aria-label="Dashboard" ...>
  ...
  <span>Dashboard</span>
</Button>
```

테스트 selector 는 아래 둘 중 하나로 통일한다.

```ts
page.getByRole("button", { name: "Dashboard" })
```

또는 다국어 대응이 필요하면:

```ts
page.getByTestId("dashboard-nav")
```

단, 우선 role selector 를 통과시키는 것이 목표다.

### 2. Dashboard active state 정리

Dashboard 클릭 시:

1. `activeKey = null`
2. `view = chat` 또는 dashboard route 에 해당하는 내부 상태
3. mobile sidebar 닫기
4. ThreadShell 에 `session = null` 전달

### 3. Pending queue action 연결

AssistantDashboard 의 pending item click 은 해당 session 으로 이동한다.

다음 단계에서는 session 이동 후 pending interaction prompt 가 즉시 보여야 한다.

### 4. Dashboard density 정리

기능 신뢰성 확보 후 visual density pass 를 적용한다.

우선순위:

1. Priority queue 는 유지
2. Recent outcomes 는 compact list
3. Linked channels 는 compact list
4. Quick actions 는 thin button row
5. dashboard 에 composer 없음 유지

## 테스트 계획

### App layout tests

1. Dashboard button exists by role
2. Dashboard click clears active session
3. Dashboard has no composer textbox
4. New chat still opens composer

### ThreadShell tests

1. no session renders AssistantDashboard
2. priority item click calls `onOpenSession`
3. pending approval item label visible

### Playwright tests

1. `Dashboard` role click works on live page
2. dashboard heading visible
3. composer absent
4. pending item click opens target thread

## 완료 기준

1. Dashboard 는 Playwright role selector 로 항상 진입 가능하다.
2. dashboard 에서 pending item 은 actionable 하다.
3. dashboard 는 composer 를 노출하지 않는다.
4. visual density 문서와 충돌하지 않는다.