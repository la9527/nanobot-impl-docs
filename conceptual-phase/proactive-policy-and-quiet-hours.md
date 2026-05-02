# Nanobot Proactive Policy And Quiet Hours Draft

이 문서는 [ConceptualPhase.md](../ConceptualPhase.md)에서 분리한 상세 초안으로, Nanobot를 `개인비서`로 발전시킬 때 필요한 proactive behavior 정책과 quiet hours 기준을 정리한다.

핵심 목적은 세 가지다.

- 현재 Nanobot에 이미 있는 heartbeat, periodic task, proactive delivery 기반을 개인비서형 proactive surface로 어떻게 확장할지 정의하기
- 유용한 선제 알림과 과한 autonomy를 구분하는 정책 기준 만들기
- owner profile, personal task state, approval, channel continuity와 연결되는 운영 원칙 정하기

이 문서는 `아무 때나 먼저 말하는 assistant`를 정당화하는 문서가 아니다. 오히려 `언제 먼저 말하면 유용하고, 언제는 침묵해야 하는가`를 제한하는 문서다.

## 1. 현재 Nanobot에 이미 있는 proactive 기반

현재 문서와 기능 기준으로 Nanobot에는 아래 proactive 기반이 이미 있다.

- `HEARTBEAT.md` 기반 periodic task 실행
- heartbeat service를 통한 주기적 wake-up
- 가장 최근 활성 채널로 결과 전달
- 일부 채널에서 proactive send 허용
- idle 시 proactive auto-compact

즉, proactive capability의 바닥은 이미 있다. 다만 현재는 `개인비서 경험`보다는 `기술적 scheduled execution`에 더 가깝다.

## 2. proactive의 목표 정의

개인비서에서 proactive behavior의 목적은 단순 알림 수를 늘리는 것이 아니다.

좋은 proactive의 목적은 아래다.

- 사용자가 묻기 전에 가치 있는 요약을 제공한다.
- 놓치기 쉬운 follow-up을 상기시킨다.
- pending approval이나 blocked state를 적절한 시점에 다시 보여 준다.
- 사용자의 시간대와 루틴에 맞춘다.

반대로 나쁜 proactive는 아래에 가깝다.

- 이미 알고 있는 사실을 반복적으로 푸시한다.
- 긴 raw log를 먼저 보낸다.
- quiet hours를 무시한다.
- owner가 원하지 않은 채널에 먼저 메시지를 보낸다.
- side effect action을 사실상 압박한다.

## 3. proactive 분류

proactive surface는 아래 4종류로 나누는 것이 적절하다.

### 3.1 briefing

예시:

- 아침 요약
- 오늘 일정 브리핑
- inbox triage 요약
- 오늘 follow-up 후보 요약

성격:

- 요약 중심
- 대체로 read-only
- 가장 개인비서다운 proactive

### 3.2 reminder

예시:

- 승인 대기 중인 작업 재알림
- 회의 15분 전 리마인드
- 답장 초안 검토 요청 재알림

성격:

- time-sensitive
- 특정 task나 일정에 연결됨

### 3.3 follow-up nudge

예시:

- 지난주 생성한 초안이 아직 미발송 상태
- blocked 상태가 계속 유지되는 task가 있음
- 특정 메일 thread에 응답이 장기간 없음

성격:

- task continuity 중심
- owner의 업무 흐름을 잇는 역할

### 3.4 system-level proactive

예시:

- idle auto-compact
- heartbeat-driven scheduled execution
- background health-related notify 후보

성격:

- 사용자 경험보다는 운영/성능 보조
- 직접 사용자 메시지로 노출되지 않을 수도 있음

## 4. 기본 정책 원칙

### 4.1 opt-in 우선

개인비서 proactive는 기본적으로 opt-in이 맞다.

특히 아래는 명시적 활성화가 필요하다.

- 아침 briefing
- inbox triage push
- 메일 follow-up 알림
- 반복 reminder

반면 아래는 기본 on이어도 무방할 수 있다.

- idle auto-compact
- approval pending visibility의 WebUI 내 재노출

### 4.2 summary-first

proactive 메시지는 raw detail보다 summary가 먼저여야 한다.

좋은 예:

- `오늘 확인할 중요한 메일 3개가 있습니다.`
- `승인 대기 중인 작업 1개가 있습니다.`
- `10분 뒤 회의가 있습니다. 일정 충돌은 없습니다.`

나쁜 예:

- raw payload 덤프
- 내부 evaluator reasoning 출력
- 긴 디버그 로그

### 4.3 owner-context-aware

proactive는 owner profile을 읽어야 한다.

핵심 항목:

- timezone
- briefing 선호 시간
- quiet hours
- 선호 채널
- reminder 허용 빈도

즉, 같은 이벤트라도 owner profile에 따라 보내거나 보내지 않을 수 있어야 한다.

### 4.4 no hidden side effects

proactive는 기본적으로 `알림/요약/재노출`이어야 한다. side effect action은 proactive 자체로 실행하면 안 된다.

예시:

- 가능: `보낼 초안이 있습니다. 검토할까요?`
- 불가: 사용자 확인 없이 메일 자동 발송

## 5. quiet hours 정책 초안

### 5.1 quiet hours의 목적

quiet hours는 assistant가 owner의 집중 시간이나 휴식 시간을 침범하지 않도록 제한하는 장치다.

quiet hours는 기능이 아니라 trust 장치에 가깝다.

### 5.2 권장 모델

초기에는 아래 정도의 단순 모델이 적절하다.

| 필드 | 의미 |
| --- | --- |
| `enabled` | quiet hours 사용 여부 |
| `start_local_time` | 시작 시각 |
| `end_local_time` | 종료 시각 |
| `timezone` | owner 기준 timezone |
| `allow_critical` | critical reminder 예외 허용 여부 |
| `allowed_channels` | 예외 허용 채널 |

예시:

```json
{
  "enabled": true,
  "start_local_time": "22:30",
  "end_local_time": "07:30",
  "timezone": "Asia/Seoul",
  "allow_critical": true,
  "allowed_channels": ["webui"]
}
```

### 5.3 quiet hours 중 허용할 수 있는 것

- WebUI 내부 배지/상태 표시
- 다음 날로 미뤄도 되는 briefing 준비
- critical 일정 리마인드 같은 제한적 예외

### 5.4 quiet hours 중 기본 차단할 것

- Telegram/Slack/Email proactive push
- inbox triage push
- 반복 follow-up nagging
- non-critical approval reminder

## 6. proactive severity 분류

proactive는 severity에 따라 다르게 다루는 편이 낫다.

| Severity | 예시 | quiet hours 중 처리 |
| --- | --- | --- |
| `low` | inbox triage, general brief | 보류 |
| `normal` | follow-up reminder, draft review reminder | 보류 또는 morning digest로 합침 |
| `high` | 곧 시작하는 일정, 당일 마감에 가까운 task | owner 설정에 따라 제한적 허용 |
| `critical` | 매우 드물게 허용되는 긴급 일정/안전 관련 | 별도 opt-in일 때만 예외 |

초기 제품에서는 `critical`을 거의 비워 두는 것이 낫다.

## 7. 채널 정책

### 7.1 권장 우선순위

개인비서형 proactive는 채널별로 다르게 다뤄야 한다.

권장 우선순위:

1. WebUI: 가장 안전한 기본 control surface
2. Telegram/Slack: 짧은 reminder나 summary에 적합
3. Email: 브리핑/리포트성 요약에 제한적으로 적합
4. Kakao: 향후 요약형 handoff에 적합하나 control surface는 약함

### 7.2 채널 선택 원칙

- 기본은 owner의 최근 활성 채널을 참조하되, owner profile의 preferred proactive channel이 있으면 우선 검토
- quiet hours 중에는 push channel보다 WebUI badge/summary를 우선
- approval 관련 proactive는 rich control surface가 있는 WebUI 우선

## 8. 대표 proactive 시나리오

### 8.1 morning briefing

조건:

- owner opt-in
- quiet hours 종료 후
- 최근 24시간 요약 가능

포함 항목 예시:

- 오늘 일정 3개
- 승인 대기 1개
- follow-up 후보 메일 2개
- blocked task 1개

### 8.2 pending approval reminder

조건:

- `waiting-approval` task 존재
- 최초 노출 후 일정 시간 경과
- quiet hours 아님 또는 WebUI-only 재노출

메시지 원칙:

- 승인 요청 이유와 대상만 짧게 요약
- 바로 action할 수 있는 surface로 연결

### 8.3 follow-up digest

조건:

- 미완료/blocked task 누적
- owner가 follow-up digests opt-in

메시지 원칙:

- 개별 nagging보다 digest 우선
- 같은 task를 과도하게 반복하지 않음

## 9. anti-spam / anti-annoyance 규칙

개인비서 proactive는 usefulness보다 annoyance risk가 더 크다. 따라서 아래 제한이 필요하다.

- 같은 category proactive는 짧은 시간에 반복 전송하지 않음
- 같은 task에 대해 연속 리마인드 횟수 제한
- low severity 항목은 digest로 묶기 우선
- 실패/blocked 상태를 매 tick마다 다시 보내지 않음
- owner가 dismiss한 항목은 일정 기간 재알림 금지

## 10. owner profile 및 task state와의 연결

### 10.1 owner profile에서 읽어야 할 것

- timezone
- quiet hours
- preferred proactive channel
- reminder frequency tolerance
- morning briefing preference

### 10.2 task state에서 읽어야 할 것

- `waiting-approval`
- `blocked`
- `scheduled`
- `completed` 이후 후속 필요 여부

즉, proactive는 단독 기능이 아니라 owner profile과 task state를 읽는 상위 behavior layer다.

## 11. 현재 기준 상태 판정

| 항목 | 현재 상태 | 판정 |
| --- | --- | --- |
| heartbeat periodic tasks | 구현 있음 | 기술적 기반 확보 |
| proactive auto-compact | 구현 있음 | 운영형 proactive 확보 |
| proactive delivery 기본 경로 | 일부 있음 | 제품 정책 보강 필요 |
| morning briefing | 없음 | 설계 필요 |
| quiet hours | 없음 | 설계 필요 |
| reminder frequency policy | 없음 | 설계 필요 |
| owner-aware proactive preference | 없음 | owner profile 연결 필요 |

## 12. phase별 제안

### Phase 1

- proactive category와 severity vocabulary 확정
- quiet hours 정책 초안 확정
- WebUI-first proactive 원칙 정리

### Phase 2

- owner profile에서 proactive preference field 확정
- waiting-approval / blocked digest를 WebUI와 연결
- morning briefing 최소안 설계

### Phase 3

- inbox triage / follow-up digest / schedule digest 확장
- Telegram/Slack/Kakao별 proactive delivery 정책 세분화

## 13. 한 줄 결론

Nanobot의 proactive는 `먼저 말할 수 있다`는 기술 능력보다 `언제는 말하고 언제는 침묵하는가`를 설계하는 정책 문제가 더 크다. 개인비서 제품화에서 quiet hours와 summary-first 원칙은 기능이 아니라 신뢰의 핵심이다.