# 06. 상태, 작업, 메모리 관리

## 상태 확인

가장 기본적인 상태 확인 명령은 아래다.

```text
/status
```

또는 자연어로 물어볼 수도 있다.

```text
현재 상태를 알려줘
현재 시스템 상태를 확인해줘
```

보통 아래 정보를 확인할 수 있다.

- 현재 채널
- chat ID
- model 또는 target
- context 설정
- web 기능 여부
- exec 기능 여부
- 최근 작업 상태

## WebUI 상태 표시

WebUI 에서는 상태가 여러 위치에 표시된다.

- Dashboard: 전체 요약
- Thread header: 현재 채팅 target 과 채널
- Status rail: owner-aware summary, blocked 상태, action result
- Assistant details: 자세한 자동화 결과
- Sidebar badge: pending approval 또는 channel 정보

## 작업 상태란

Nanobot 은 단순히 한 번 답하고 끝나는 것이 아니라, 일부 작업 상태를 세션 metadata 로 남긴다.

예시:

- 현재 작업 요약
- 막힌 작업
- 승인 대기
- 최근 자동화 결과
- proactive hold 또는 quiet-hours 상태

이 정보는 WebUI Dashboard 와 채팅 화면에서 다시 볼 수 있다.

## 메모리란

메모리는 사용자의 선호, 프로젝트 상태, 반복되는 지시를 기억하는 기능이다.

현재 작업된 방향은 아래와 같다.

- owner profile
- task summary
- memory boundary
- memory correction
- project memory

## 기억시키기

자연어로 요청할 수 있다.

```text
이 내용을 기억해줘: 나는 오전 회의를 선호해
앞으로 일정 요약은 짧게 알려줘
```

## 기억 수정하기

잘못 기억한 내용은 수정 요청을 할 수 있다.

예시:

```text
그건 내 기본 선호가 아니야
이건 이 프로젝트에만 해당돼
방금 말한 내용은 잊어줘
```

## 프로젝트 종료 표시

프로젝트나 작업이 끝났으면 아래처럼 말할 수 있다.

```text
이 프로젝트는 끝났어
이 작업은 완료됐어
```

Nanobot 은 이를 바탕으로 memory correction 또는 task summary 를 정리할 수 있다.

## 주의사항

- 메모리 보정은 아직 모든 자연어를 완벽히 이해하는 단계는 아니다.
- 중요한 선호는 짧고 명확하게 말하는 것이 좋다.
- 민감한 개인정보는 저장 전에 꼭 필요 여부를 판단한다.

## 좋은 요청 예시

```text
이 내용을 기억해줘: 일정은 가능하면 오후 2시 이후로 잡아줘
이건 임시 작업이니까 장기 기억에 넣지 마
지난번 캘린더 테스트 작업은 완료됐어
```

## 문제가 생겼을 때

- 기억이 잘못 반영된 것 같으면 바로 정정한다.
- WebUI 에서 memory correction prompt 가 보이면 내용을 확인한다.
- 반복적으로 틀리면 어떤 문구가 잘못 저장됐는지 구체적으로 말한다.
