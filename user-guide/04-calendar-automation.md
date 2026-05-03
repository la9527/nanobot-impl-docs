# 04. 일정 자동화와 승인 흐름

## 일정 자동화란

Nanobot 은 n8n webhook 을 통해 calendar 작업을 실행할 수 있다.

현재 정리된 기능은 아래와 같다.

- 오늘 일정 요약
- 특정 시간대 충돌 확인
- 자연어 일정 생성 요청
- 자연어 일정 수정 요청
- 자연어 일정 삭제 요청
- 일정 생성 요청
- 일정 변경 전 승인
- 승인 대기 요청 취소
- WebUI 에 calendar 결과 표시

## 현재 설정 확인

채팅창에서 아래 명령을 입력한다.

```text
/calendar
```

그러면 보통 아래 정보를 볼 수 있다.

- automation configured 여부
- n8n base URL
- summary webhook path
- create webhook path
- timezone
- webhook token 설정 여부
- pending 상태

## 오늘 일정 보기

```text
/calendar today
```

오늘 등록된 일정이 없으면 “오늘 등록된 일정이 없습니다” 같은 답변이 나온다.

## 시간대 충돌 확인

```text
/calendar check --start 2026-05-02T15:00:00+09:00 --end 2026-05-02T16:00:00+09:00
```

결과는 보통 아래 중 하나다.

- 충돌 없음
- 충돌 있음
- 설정 오류
- webhook 호출 실패

충돌이 있으면 WebUI 에서도 conflict result 로 표시된다.

## 일정 생성 요청

일정 생성은 자연어로 요청할 수 있다.

```text
5월 2일 오후 3시에 치과 일정 1시간 잡아줘
```

또는 아래처럼 말할 수 있다.

```text
내일 오후 3시에 치과 일정 잡아줘
다음 화요일 오후 2시에 병원 일정 30분 잡아줘
2026-05-02 오후 3시에 치과 일정 1시간 잡아줘
```

Nanobot 이 제목, 시작 시간, 종료 시간을 추출하고, 부족한 값이 있으면 이어서 질문한다.

예를 들어 종료 시간이 없으면 종료 시간을 다시 묻는다.

```text
Calendar create needs an end time.
```

정확한 값을 직접 지정해야 할 때는 slash command 도 계속 사용할 수 있다.

```text
/calendar create --title "치과" --start 2026-05-02T15:00:00+09:00 --end 2026-05-02T16:00:00+09:00
```

일정 생성은 바로 확정되지 않고 승인 흐름을 탄다.

이유는 일정 생성이 실제 캘린더를 바꾸는 작업이기 때문이다.

## 승인하기

생성, 수정, 삭제 요청이 맞다면 아래 명령을 입력한다.

```text
/calendar approve
승인해줘
좋아, 진행해줘
```

승인되면 pending event 가 실제 생성, 수정, 삭제 경로로 넘어간다.

## 취소 또는 반려하기

요청을 취소하려면 아래 중 하나를 쓴다.

```text
/calendar cancel
/calendar deny
취소해줘
반려해줘
```

최근 보강된 내용:

- pending 요청이 없을 때도 안전하게 처리한다.
- no-pending 상태에서 실수로 create/approve 로 이어지지 않는다.
- Telegram/WebUI 어디에서 입력해도 결과가 session history 에 남는다.

## 일정 수정 요청

일정 수정도 자연어로 요청할 수 있다.

```text
5월 2일 오후 3시 치과 일정을 오후 4시로 변경해줘
```

Nanobot 은 제목, 날짜, 현재 시작 시간, 새 시작 시간을 기준으로 기존 일정을 먼저 찾는다.

후보가 하나로 확정되면 바로 수정하지 않고 승인 대기 상태로 둔다.

```text
Calendar update approval required
```

내용이 맞으면 `/calendar approve`, 아니면 `/calendar cancel` 을 입력한다.

## 일정 삭제 요청

일정 삭제도 자연어로 요청할 수 있다.

```text
5월 2일 오후 3시 치과 일정 삭제해줘
```

Nanobot 은 제목, 날짜, 시간이 일치하는 기존 일정을 찾는다.

후보가 하나로 확정되면 바로 삭제하지 않고 승인 대기 상태로 둔다.

```text
Calendar delete approval required
```

내용이 맞으면 `/calendar approve`, 아니면 `/calendar cancel` 을 입력한다.

후보가 없거나 여러 개면 삭제하지 않고 제목과 시간을 더 구체적으로 알려 달라고 응답한다.

## WebUI 에서 보이는 상태

WebUI 에서는 calendar 관련 상태가 다음 위치에 보일 수 있다.

- 채팅 본문: 명령 결과
- 사이드바: approval pending badge
- Assistant details: action result 또는 conflict details
- Dashboard: blocked/pending 작업 요약

## 충돌이 있을 때

충돌이 있으면 새 일정을 바로 만들지 않고 충돌 정보를 보여준다.

예시로 아래처럼 보일 수 있다.

```text
Conflicts found
The requested slot overlaps with Nanobot webui calendar validation.
```

이 경우 선택지는 보통 아래와 같다.

- 새 일정 요청 취소
- 다른 시간으로 다시 요청
- 기존 일정을 따로 검토

## 현재 한계

현재 사용자 설명서 작성 시점 기준으로 중요한 한계는 아래와 같다.

- 자연어 일정 수정은 현재 시작 시간과 새 시작 시간이 모두 있어야 안정적으로 처리된다.
- 자연어 일정 삭제는 후보가 하나로 확정될 때만 승인 흐름으로 넘어간다.
- 반복 일정, 참석자 변경, 복잡한 제목 변경은 아직 별도 공식 command 로 정리되지 않았다.
- n8n webhook 과 캘린더 인증 상태가 정상이어야 실제 일정 작업이 된다.

## 추천 사용 흐름

1. `/calendar` 로 설정과 pending 상태를 확인한다.
2. `/calendar today` 로 오늘 일정을 본다.
3. 새 일정을 만들기 전 `/calendar check ...` 로 충돌을 본다.
4. 충돌이 없으면 자연어로 `내일 오후 3시에 치과 일정 1시간 잡아줘` 처럼 입력한다.
5. 기존 일정을 바꿀 때는 `5월 2일 오후 3시 치과 일정을 오후 4시로 변경해줘` 처럼 입력한다.
6. 기존 일정을 지울 때는 `5월 2일 오후 3시 치과 일정 삭제해줘` 처럼 입력한다.
7. 내용이 맞으면 `승인해줘`, 아니면 `취소해줘` 를 입력한다.
