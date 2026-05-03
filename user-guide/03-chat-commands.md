# 03. 자주 쓰는 명령어

## 명령어란

명령어는 채팅창에 `/` 로 시작해서 입력하는 짧은 기능 호출이다.

예시:

```text
/status
/calendar
/model
/help
```

일반 자연어로 물어봐도 되지만, 정확한 기능을 바로 실행하고 싶을 때 명령어가 편하다.

## 기본 명령어

| 명령어 | 용도 |
| --- | --- |
| `/help` | 사용할 수 있는 명령어 보기 |
| `/status` | 현재 Nanobot 상태 확인 |
| `/new` | 현재 작업을 멈추고 새 대화 시작 |
| `/context` | 현재 채팅에 저장된 context 상태 확인 |
| `/context clear` | 현재 채팅의 저장된 대화 context 비우기 |
| `/clear` | `/context clear` 단축 명령 |
| `/stop` | 현재 작업 중지 |
| `/restart` | bot 재시작 요청 |

`/context clear` 는 현재 Telegram 또는 WebUI 채팅 세션에 쌓인 대화 기록과 pending 상태 표시를 비운다. 모델 선택, reply footer 설정, 장기 메모리는 유지한다.

## 일정 명령어

| 명령어 | 용도 |
| --- | --- |
| `/calendar` | calendar 자동화 도움말과 현재 설정 보기 |
| `/calendar today` | 오늘 일정 요약 |
| `/calendar status` | calendar 상태 확인 |
| `/calendar config` | calendar webhook 설정 확인 |
| `/calendar check --start ... --end ...` | 특정 시간대 충돌 확인 |
| `/calendar create --title ... --start ... --end ...` | 일정 생성 요청 |
| `/calendar approve` | pending 일정 생성 승인 |
| `/calendar deny` | pending 일정 생성 반려 |
| `/calendar cancel` | pending 일정 생성 취소 |

가장 자주 쓰는 것은 아래 3개다.

```text
/calendar
/calendar today
/calendar cancel
```

## 모델 명령어

| 명령어 | 용도 |
| --- | --- |
| `/model` | 현재 세션의 모델 target 보기 |
| `/model list` | 선택 가능한 target 목록 보기 |
| `/model smart-router` | smart-router 자동 선택 사용 |
| `/model smart-router-local` | local tier 우선 사용 |
| `/model smart-router-mini` | mini tier 우선 사용 |
| `/model smart-router-full` | full tier 우선 사용 |
| `/model clear` | 세션별 모델 선택 해제 |

## 메모리 관련 자연어

현재 메모리 보정은 일부 자연어 표현으로도 동작한다.

예시:

```text
이 내용을 기억해줘: 나는 오전 회의를 선호해
이건 기본 선호가 아님
이 프로젝트는 끝났어
그 내용은 잊어줘
```

정확한 동작은 문장 형태와 현재 세션 상태에 따라 달라질 수 있다.

## 실행 명령 관련 자연어

exec 기능이 활성화되어 있으면 Nanobot 에게 로컬 명령 실행을 요청할 수 있다.

예시:

```text
exec로 pwd 명령을 한 번 실행해줘
Downloads 목록을 확인해줘
현재 시스템 상태를 확인해줘
```

주의:

- 삭제나 위험 명령은 승인 또는 차단이 필요할 수 있다.
- workspace 밖 파일 접근은 설정과 권한에 따라 제한될 수 있다.

## 초보자 추천 사용법

처음에는 아래 순서로 써보는 것이 좋다.

1. `/status`
2. `/calendar`
3. `/calendar today`
4. `/model`
5. `/model list`
6. 일반 질문 입력

## 명령이 기대대로 안 될 때

- 명령어 앞에 `/` 가 있는지 확인한다.
- Telegram 세션인지 WebUI 세션인지 확인한다.
- 일정 명령은 n8n calendar webhook 설정이 필요하다.
- 모델 명령은 설정된 target 이 있어야 선택 가능하다.
- 계속 이상하면 `/status` 로 현재 상태부터 확인한다.
