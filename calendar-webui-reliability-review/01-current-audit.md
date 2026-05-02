# Current Audit - 2026-05-02

## 점검 범위

이번 점검은 live WebUI `http://127.0.0.1:8765/webui/` 를 Playwright 로 직접 확인하고, WebUI bootstrap token 을 사용해 session API 응답을 함께 확인한 결과를 기준으로 한다.

확인 대상은 아래와 같다.

1. thread 화면의 실제 렌더링 상태
2. sidebar 와 dashboard 진입 동작
3. calendar result, approval, prompt 선택 박스의 표시 여부
4. session summary metadata 와 persisted message history 의 불일치
5. 최근 채팅 히스토리에서 calendar 요청이 실제 automation 으로 연결됐는지 여부

## Playwright 관찰 결과

### Thread 화면

현재 thread 화면에서는 아래 요소가 확인되었다.

- header: `Target Auto`, `Channel Telegram`, `Linked session`
- status rail: `Approvals 2`, `Blocked 2`, `Linked 4`, `Linked Telegram`
- inline result: `Calendar result`, `Blocked`, `Conflict check`
- composer: 입력창은 보이지만 전송 버튼이 disabled 상태로 보인 순간이 있었다.
- action prompt 선택 박스는 보이지 않았다.

즉, 현재 화면은 pending approval 과 blocked 상태를 보여주지만, 사용자가 다음 액션을 선택할 수 있는 prompt surface 를 안정적으로 보여주지 못한다.

### Dashboard 진입

Playwright 에서 `Dashboard` 버튼을 role 기반으로 클릭하려 했지만 timeout 이 발생했다.

관찰된 현상은 아래와 같다.

- snapshot 에는 `Dashboard` 텍스트가 보인다.
- 그러나 `getByRole('button', { name: 'Dashboard' })` 는 클릭 가능한 요소로 안정적으로 찾지 못했다.
- 화면은 dashboard 로 전환되지 않고 기존 thread 화면이 유지되었다.

이 문제는 dashboard navigation 의 accessibility name, clickable target, collapsed sidebar 상태 중 하나와 관련될 가능성이 있다. dashboard 는 정보 구조뿐 아니라 Playwright regression 기준에서도 안정적인 진입점을 가져야 한다.

### Console 상태

브라우저 console 에 `401 Unauthorized` resource error 가 반복적으로 관찰되었다.

가능한 원인은 아래와 같다.

- bootstrap token 만료 후 polling 또는 stale fetch 가 계속 실행됨
- session refresh, media URL, settings API 중 일부가 reauth 없이 실패함
- WebSocket 재연결 또는 history polling 이 token 갱신과 어긋남

이 현상은 prompt 가 사라지는 직접 원인일 수도 있고, 적어도 live 검증을 불안정하게 만드는 요인이다.

## API 응답 관찰 결과

session API 에서 현재 thread 와 관련된 metadata 에 아래 값들이 동시에 남아 있었다.

- `action_result.status = blocked`
- `action_result.domain = calendar`
- `approval_summary.status = pending`
- `calendar_create_approval` payload 존재
- `task_summary.status = waiting-approval`

하지만 persisted message tail 에는 button-bearing assistant message 가 없었다.

이 조합은 현재 WebUI 구조에서 치명적이다. WebUI 는 마지막 assistant message 의 `buttons` 를 찾아 prompt 를 만들기 때문이다. metadata 는 pending 상태인데 message tail 에 buttons 가 없으면 사용자는 선택 박스를 볼 수 없다.

## 채팅 히스토리 관찰 결과

최근 대화에서 사용자는 아래와 같은 자연어 요청을 했다.

- `구글 캘린더 일정 리스트를 먼저 보여줘`
- `이거 지워줘 "Nanobot webui calendar validation - 5. 4. 14:00~14:30"`
- `내부 툴이 있는 것으로 알고 있는데 안되나?`

응답은 calendar automation 이 아니라 일반 LLM 답변으로 처리되었다.

즉, 현재 calendar 기능은 `/calendar list`, `/calendar create`, `/calendar approve` 같은 slash command 로는 존재하지만, 자연어 calendar 요청은 command/automation 으로 routing 되지 않는다.

## 코드 구조상 확인된 원인

### Prompt source of truth 문제

`ThreadShell` 의 `pendingAsk` 는 message 배열을 뒤에서부터 훑는다.

동작은 아래와 같다.

1. 마지막 assistant message 에 buttons 가 있으면 prompt 표시
2. 그 뒤 user message 가 있으면 prompt 없음
3. 그 뒤 일반 assistant message 가 있으면 prompt 없음

이 방식은 transient prompt 에는 간단하지만, refresh, polling, proactive message, action result update 가 끼면 쉽게 깨진다.

### Calendar metadata cleanup 문제

calendar conflict review 는 `calendar_conflict_review` metadata 를 사용하고, create approval 은 `calendar_create_approval` 과 `approval_summary` 를 사용한다. input collection 은 `calendar_create_input` 을 사용한다.

그러나 cancel, reschedule, force create, approve, deny 경로에서 이 세 가지 상태가 항상 같이 정리되지 않는다. 그 결과 이전 conflict 또는 approval 정보가 남아 dashboard 와 thread 에 stale 상태를 계속 노출할 수 있다.

### Delete/reschedule 기능 부재

현재 calendar automation 은 list, conflict check, create, approve, deny 중심이다. delete event 와 update/reschedule event 는 구현되어 있지 않다.

따라서 conflict 가 났을 때 사용자가 기존 일정을 제거하려 해도, 선택지와 실행 경로가 없다.

## 우선순위 판단

1. `pending interaction` 을 metadata 기반으로 재정의한다.
2. WebUI prompt 는 metadata 를 우선 표시한다.
3. calendar state cleanup 을 backend helper 로 통합한다.
4. 자연어 calendar intent 를 command bridge 로 연결한다.
5. delete/reschedule automation 을 별도 approval flow 로 추가한다.
6. dashboard navigation 과 401 반복 오류를 Playwright regression 에 포함한다.