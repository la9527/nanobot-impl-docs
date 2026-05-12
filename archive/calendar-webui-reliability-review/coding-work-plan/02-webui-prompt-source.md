# 02. WebUI Prompt Source

## 목표

WebUI 가 prompt 선택 박스를 message tail 에서만 찾지 않고, session metadata 의 `calendar_pending_interaction` 을 우선 사용하도록 변경한다.

## 수정 대상 파일

1. `source/webui/src/lib/types.ts`
2. `source/webui/src/lib/sessionMetadata.ts`
3. `source/webui/src/components/thread/ThreadShell.tsx`
4. `source/webui/src/components/thread/AskUserPrompt.tsx`
5. `source/webui/src/tests/thread-shell.test.tsx`
6. `source/webui/src/tests/useSessions.test.tsx`

## Type 추가

`types.ts` 의 `ChatSummary.metadata` 에 아래 형태를 추가한다.

```ts
calendar_pending_interaction?: {
  id?: string;
  kind?: "collect_input" | "conflict_review" | "create_approval" | "delete_approval" | "reschedule_approval" | string;
  status?: "pending" | "completed" | "cancelled" | "failed" | string;
  question?: string;
  buttons?: string[][];
  request?: Record<string, unknown>;
  conflicts?: Array<Record<string, unknown>>;
  expected_field?: string | null;
  created_at?: string | null;
  updated_at?: string | null;
}
```

## Parser 추가

`sessionMetadata.ts` 에 `getPendingInteraction` 또는 calendar 전용 parser 를 추가한다.

```ts
export interface DerivedPendingInteraction {
  id: string | null;
  domain: "calendar" | "generic";
  kind: string | null;
  status: string | null;
  question: string | null;
  buttons: string[][];
}

export function getCalendarPendingInteraction(session: ChatSummary | null | undefined): DerivedPendingInteraction | null {
  ...
}
```

## ThreadShell 변경

현재 `pendingAsk` 는 `messages` 만 본다.

변경 후 우선순위:

1. `getCalendarPendingInteraction(session)`
2. calendar `approval_summary` fallback
3. latest assistant message buttons fallback

주의:

- metadata prompt 를 표시할 때도 `AskUserPrompt` 의 `onAnswer` 는 기존 경로를 사용한다.
- websocket session 은 `handleWebSocketSend`
- linked session 은 `handleBridgedSessionSend`

## UX 규칙

1. prompt 는 inline result 보다 아래, composer 보다 위에 둔다.
2. pending prompt 가 있으면 status rail 과 inline result 는 prompt 를 가리지 않는다.
3. prompt question 은 너무 길면 2줄 clamp 또는 compact preview 로 줄인다.
4. detail 은 inline result `Details` 나 assistant details sheet 로 보낸다.

## 테스트 추가

`thread-shell.test.tsx` 에 추가한다.

1. metadata conflict_review prompt renders without message buttons
2. metadata prompt survives when latest assistant message has no buttons
3. metadata prompt click uses websocket send for WebUI session
4. metadata prompt click uses session_message for Telegram linked session
5. message-tail buttons still work as fallback when metadata is absent

## 완료 기준

1. session metadata 에 pending interaction 이 있으면 prompt 가 항상 보인다.
2. hard reload 후에도 prompt 가 보인다.
3. proactive summary 또는 later assistant message 가 있어도 prompt 가 사라지지 않는다.
4. WebUI focused tests 가 통과한다.