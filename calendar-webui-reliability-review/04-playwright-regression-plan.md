# Playwright Regression Plan

## 목적

이번 재적용은 live WebUI 에서 직접 확인해야 한다. 단위 테스트만으로는 prompt 가 사라지는 문제, dashboard 진입 실패, token 만료 후 polling 같은 문제를 잡기 어렵다.

## 공통 준비

1. WebUI bundle 을 build 한다.
2. launchd gateway 를 재기동한다.
3. `http://127.0.0.1:8765/webui/bootstrap` 응답을 확인한다.
4. 브라우저가 최신 `assets/index-*.js` 를 로드하는지 확인한다.
5. console 에 반복 401 이 있는지 기록한다.

## Scenario A. Dashboard navigation

목표: dashboard 가 role 기반 Playwright 로 안정적으로 열려야 한다.

절차:

1. WebUI 첫 화면 진입
2. sidebar 가 접혀 있으면 연다.
3. `getByRole('button', { name: /Dashboard/ })` 로 dashboard 클릭
4. dashboard heading 또는 priority queue 확인
5. composer 가 dashboard 에 보이지 않는지 확인

합격 기준:

- role 기반 selector 로 dashboard 진입 성공
- dashboard 는 overview surface 로 표시
- thread composer 없음

## Scenario B. Calendar list natural language

목표: 자연어 calendar list 요청이 일반 LLM 답변으로 빠지지 않는다.

절차:

1. 새 WebUI chat 생성
2. `오늘 캘린더 일정 보여줘` 입력
3. calendar automation 실행 결과 확인
4. action_result domain 이 `calendar`, action 이 `list_events` 인지 API 로 확인

합격 기준:

- assistant 가 “툴이 없다”라고 답하지 않음
- calendar result 또는 structured event summary 표시

## Scenario C. Calendar conflict prompt persistence

목표: conflict review 선택 박스가 refresh 와 polling 후에도 유지된다.

절차:

1. 기존 일정과 겹치는 create 요청 입력
2. conflict result 표시 확인
3. prompt 버튼 확인
4. hard reload
5. prompt 버튼이 다시 표시되는지 확인
6. session API 에서 `calendar_pending_interaction.kind = conflict_review` 확인

합격 기준:

- `그래도 생성 승인 요청`, `새 시간 다시 입력`, `취소` 버튼 유지
- message tail 에 buttons 가 없어도 metadata 로 표시

## Scenario D. Reschedule flow

목표: conflict 후 새 시간 입력이 안정적으로 이어진다.

절차:

1. conflict prompt 에서 `새 시간 다시 입력` 클릭
2. start time 입력 prompt 표시 확인
3. 직접 ISO start 입력
4. end time 입력 prompt 표시 확인
5. end time 입력
6. conflict 재검사 또는 approval 전환 확인

합격 기준:

- prompt 가 중간에 사라지지 않음
- stale conflict card 가 새 interaction 을 덮지 않음

## Scenario E. Force create approval

목표: conflict 를 감수하고 생성 승인 단계로 전환한다.

절차:

1. conflict prompt 에서 `그래도 생성 승인 요청` 클릭
2. create approval prompt 표시 확인
3. session API 에서 conflict review metadata 제거 확인
4. approval metadata 만 남는지 확인

합격 기준:

- conflict prompt 제거
- create approval prompt 표시
- stale `calendar_conflict_review` 없음

## Scenario F. Cancel and deny cleanup

목표: 취소 후 stale pending 상태가 남지 않는다.

절차:

1. conflict prompt 에서 `취소` 클릭
2. cancelled result 확인
3. refresh
4. prompt 와 pending badge 가 사라졌는지 확인
5. API metadata 에 pending interaction, approval summary 가 없는지 확인

합격 기준:

- `Approvals N` 이 줄어듦
- prompt 없음
- action_result 는 cancelled 또는 rejected 로 교체

## Scenario G. Delete event flow

목표: delete 기능이 추가된 뒤 기존 일정 삭제가 안전하게 동작한다.

절차:

1. calendar list 실행
2. 이벤트 선택
3. delete approval prompt 표시
4. 삭제 승인
5. list 재조회로 삭제 확인

합격 기준:

- event_id 기반 삭제
- 승인 없이는 삭제되지 않음
- 삭제 후 stale conflict 가 사라짐

## Scenario H. Telegram fallback

목표: Telegram 에서 native buttons off 상태도 동일하게 동작한다.

절차:

1. Telegram linked session 에서 conflict 생성
2. fallback text `[새 시간 다시 입력]` 또는 `새 시간 다시 입력` 입력
3. WebUI mirrored session 에서 상태 동기화 확인

합격 기준:

- bracketed reply normalize
- WebUI prompt 와 Telegram prompt 상태 일치

## 기록 방식

각 scenario 는 아래 형식으로 결과를 남긴다.

```text
Scenario:
Date:
Browser asset hash:
Session key:
Input:
Expected:
Actual:
API metadata:
Console errors:
Pass/Fail:
Follow-up:
```