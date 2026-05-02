# 04. 일정 자동화와 승인 흐름

## 일정 자동화란

Nanobot 은 n8n webhook 을 통해 calendar 작업을 실행할 수 있다.

현재 정리된 기능은 아래와 같다.

- 오늘 일정 요약
- 특정 시간대 충돌 확인
- 일정 생성 요청
- 일정 생성 전 승인
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

```text
/calendar create --title "치과" --start 2026-05-02T15:00:00+09:00 --end 2026-05-02T16:00:00+09:00
```

일정 생성은 바로 확정되지 않고 승인 흐름을 탄다.

이유는 일정 생성이 실제 캘린더를 바꾸는 작업이기 때문이다.

## 승인하기

생성 요청이 맞다면 아래 명령을 입력한다.

```text
/calendar approve
```

승인되면 pending event 가 실제 생성 경로로 넘어간다.

## 취소 또는 반려하기

요청을 취소하려면 아래 중 하나를 쓴다.

```text
/calendar cancel
/calendar deny
```

최근 보강된 내용:

- pending 요청이 없을 때도 안전하게 처리한다.
- no-pending 상태에서 실수로 create/approve 로 이어지지 않는다.
- Telegram/WebUI 어디에서 입력해도 결과가 session history 에 남는다.

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

- 일정 삭제와 일정 이동은 별도 공식 command 로 아직 정리되지 않았다.
- 기존 일정 자체를 삭제하려면 calendar delete interface 가 추가로 필요하다.
- n8n webhook 과 캘린더 인증 상태가 정상이어야 실제 일정 작업이 된다.

## 추천 사용 흐름

1. `/calendar` 로 설정과 pending 상태를 확인한다.
2. `/calendar today` 로 오늘 일정을 본다.
3. 새 일정을 만들기 전 `/calendar check ...` 로 충돌을 본다.
4. 충돌이 없으면 `/calendar create ...` 를 입력한다.
5. 내용이 맞으면 `/calendar approve`, 아니면 `/calendar cancel` 을 입력한다.
