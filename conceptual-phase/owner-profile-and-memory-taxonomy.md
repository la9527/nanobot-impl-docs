# Nanobot Owner Profile And Memory Taxonomy Draft

이 문서는 [ConceptualPhase.md](../ConceptualPhase.md)에서 분리한 상세 초안으로, Nanobot를 `단일 사용자 LLM 개인비서`로 발전시키기 위해 필요한 `owner profile`과 `memory taxonomy`를 구체화한다.

핵심 목적은 세 가지다.

- `canonical owner = primary-user` 개념을 실제 개인비서 제품 레이어에서 어떤 정보 구조로 확장할지 정의하기
- 현재 Nanobot memory 구조와 충돌하지 않도록 `무엇을 어디에 저장할지` 구분하기
- 이후 task, approval, Gmail/Calendar action, proactive routine이 참조할 수 있는 공통 분류 기준을 만들기

이 문서는 multi-user user directory를 만들기 위한 문서가 아니다. 방향 A 기준에서 `한 명의 owner와 assistant 관계`를 더 정교하게 다루기 위한 문서다.

## 1. 왜 owner profile이 필요한가

현재 continuity 문서와 구현은 아래 수준까지는 이미 다룬다.

- canonical owner id
- channel kind
- external identity
- trust level
- last confirmed timestamp

이 구조는 `같은 사람인지 연결`하는 데는 충분하지만, 개인비서가 실제로 도움이 되기 위해 필요한 아래 정보까지 담지는 못한다.

- 이 사용자는 어떤 톤과 길이의 답변을 선호하는가
- 일정이나 메일을 다룰 때 기본 기준은 무엇인가
- 자주 등장하는 사람, 프로젝트, 루틴은 무엇인가
- 어떤 정보는 장기 기억으로 올리고 어떤 정보는 임시로만 다뤄야 하는가

즉, continuity metadata는 `누구인가`를 최소한으로 연결하는 층이고, owner profile은 `이 사람을 어떻게 도와야 하는가`를 설명하는 층이다.

## 2. 현재 Nanobot memory 구조와의 관계

현재 Nanobot는 이미 layered memory를 갖고 있다.

- `session.messages`: 현재 대화의 living context
- `memory/history.jsonl`: 오래된 대화의 압축 요약
- `USER.md`: 사용자에 대한 장기 지식
- `memory/MEMORY.md`: 프로젝트/업무/지속 맥락
- `SOUL.md`: assistant의 voice와 style

따라서 owner profile / memory taxonomy 설계는 새로운 중앙 데이터베이스를 추가하는 방향보다, 위 구조의 의미를 더 명확하게 나누는 방향이 적절하다.

핵심 원칙은 아래다.

1. continuity metadata는 여전히 lightweight metadata로 유지한다.
2. owner profile은 `USER.md`와 관련 structured metadata의 책임을 명확히 하는 쪽으로 확장한다.
3. project/task/domain 지식은 `memory/MEMORY.md` 또는 별도 domain memory 후보로 분리한다.
4. raw history는 `history.jsonl`에 남기되, 장기 기억은 curated form으로만 승격한다.

## 3. Owner Profile 초안

### 3.1 정의

owner profile은 `primary-user`를 assistant가 장기적으로 이해하기 위해 유지하는 안정적인 사용자 모델이다.

이 profile은 아래를 위한 기준점이 된다.

- 답변 스타일 결정
- action planning 기본값 결정
- Gmail/Calendar 같은 domain action의 기본 규칙 제공
- proactive briefing 구성
- memory update 시 우선순위 판단

### 3.2 owner profile의 속성 범주

owner profile은 아래 6개 범주로 나누는 것이 적절하다.

#### A. Identity and defaults

예시 항목:

- display name
- preferred language
- timezone
- locale
- default response tone
- default response length

이 범주는 거의 모든 turn에 간접적으로 영향을 준다.

#### B. Communication preferences

예시 항목:

- 채널별 응답 선호
  - Telegram에서는 짧게
  - WebUI에서는 길게
- 메일 초안 톤 선호
- 일정 설명 방식 선호
- 요약 시 bullet 선호 여부

이 범주는 response generation과 draft generation에 직접 연결된다.

#### C. Work and project context

예시 항목:

- 현재 중요한 장기 프로젝트
- 자주 다루는 작업 종류
- 반복되는 운영 환경
- 자주 참조하는 repo, 문서, 시스템

이 범주는 `USER.md`와 `MEMORY.md` 경계에서 가장 자주 충돌하므로 별도 기준이 필요하다.

#### D. People and relationship context

예시 항목:

- 자주 연락하는 사람
- 메일에서 자주 등장하는 이해관계자
- 일정 조율 상대
- 관계별 말투/우선순위 힌트

이 범주는 개인비서 가치가 크지만 민감도도 높으므로 보수적으로 저장해야 한다.

#### E. Routine and behavioral defaults

예시 항목:

- 아침 briefing 선호 시간
- 미팅 전 리마인드 선호
- inbox triage 빈도
- focus time 또는 quiet hours

이 범주는 proactive routine 설계의 기반이 된다.

#### F. Tool and policy defaults

예시 항목:

- 기본 model target 선호
- 기본 mail/calendar source of truth
- approval이 항상 필요한 action 유형
- 업로드/발송/수정 계열의 보수성 수준

이 범주는 assistant automation과 approval policy에 직접 연결된다.

### 3.3 owner profile에 넣지 말아야 하는 것

아래 내용은 owner profile 본문에 직접 넣지 않는 것이 낫다.

- Telegram user id, Slack user id, Email thread id 같은 transport identifier
- 일회성 감정 상태나 하루짜리 TODO
- raw external payload
- access token, password, app secret
- session-local draft 전문

이 정보는 continuity metadata, session state, secure config, raw history 등 다른 계층으로 남겨야 한다.

## 4. Owner Profile 저장 원칙

### 4.1 권장 저장 경계

현재 구조 기준 권장 분리는 아래와 같다.

| 정보 종류 | 권장 저장 위치 | 이유 |
| --- | --- | --- |
| 사용자 기본 선호 | `USER.md` | 장기적이고 owner 중심 |
| assistant voice/style | `SOUL.md` | user가 아니라 assistant 자체 성격 |
| 프로젝트/업무 지속 사실 | `memory/MEMORY.md` | owner보다 work context 성격이 강함 |
| 오래된 대화 요약 | `memory/history.jsonl` | curated memory의 재료 |
| linked channel metadata | `session.metadata["continuity"]` | lightweight continuity layer |
| pending approval visibility | `session.metadata["approval_summary"]` | cross-session visibility용 metadata |

### 4.2 future structured layer 후보

장기적으로는 `USER.md`만으로 충분하지 않을 수 있다. 다만 지금 단계에서는 별도 DB보다 아래 정도의 lightweight structured layer를 검토하는 편이 적절하다.

예시:

```text
memory/
  owner-profile.json
```

이 파일은 source of truth가 아니라, `USER.md`의 특정 핵심 항목을 기계가 안정적으로 읽기 쉽게 정규화한 캐시 또는 보조 구조로 보는 편이 낫다.

즉, 1차 기본 원칙은 `문서형 memory 우선`, `기계 보조 구조는 최소화`다.

## 5. Memory Taxonomy 초안

### 5.1 목적

memory taxonomy는 어떤 사실을 어떤 등급의 기억으로 볼지 정하는 분류 체계다.

이 taxonomy가 필요한 이유는 아래와 같다.

- Dream이 무엇을 장기 기억으로 올릴지 기준이 필요하다.
- task, approval, proactive routine이 같은 분류 기준을 참조해야 한다.
- `기억해야 할 것`과 `그냥 기록으로 남길 것`을 구분해야 한다.

### 5.2 권장 분류

#### 1. Preference memory

정의:

- 사용자의 지속적 선호나 기본값

예시:

- 답변은 짧고 직설적으로 선호
- 일정 제안은 한국 시간 기준으로 설명 선호
- 메일 초안은 공손하지만 과장되지 않은 톤 선호

권장 위치:

- `USER.md`

승격 기준:

- 반복적으로 나타남
- 다음 turn이나 다음 주에도 유효할 가능성이 높음

#### 2. Project memory

정의:

- 특정 프로젝트, 운영 환경, 지속적 결정 사항

예시:

- Nanobot는 launchd 기준으로 운영한다.
- WebUI build 후 gateway 재기동 검증이 필요하다.
- 특정 저장소는 reference-only다.

권장 위치:

- `memory/MEMORY.md`

승격 기준:

- 반복되는 작업 판단에 실질적으로 도움 됨
- owner 개인 성향보다 work context 성격이 강함

#### 3. People and contact memory

정의:

- 자주 등장하는 사람과 관계적 맥락

예시:

- 특정 동료는 메일로 초안 검토를 자주 요청함
- 특정 상대는 일정 변경에 민감함

권장 위치:

- 기본은 `USER.md`
- 민감도 높으면 raw 저장보다 요약형만 유지

승격 기준:

- 반복되는 실제 상호작용이 존재함
- 이후 action planning에 도움이 됨
- 민감도와 보관 필요성을 함께 만족함

#### 4. Routine memory

정의:

- 반복 일정, 행동 패턴, 브리핑 습관 같은 주기적 정보

예시:

- 평일 오전 briefing 선호
- 회의 15분 전 미리 요약 받기 선호

권장 위치:

- `USER.md`

승격 기준:

- 명시적 요청이 있거나 반복 관찰이 충분함

#### 5. Active task memory

정의:

- 현재 진행 중이라 장기기억보다 `ongoing state`에 가까운 정보

예시:

- 답장 초안 대기 중
- approval pending 상태
- 오늘 안에 처리할 follow-up

권장 위치:

- 장기 memory가 아니라 session/task layer

중요한 점:

- 이것은 memory taxonomy에는 포함되지만, 저장 위치는 long-term memory가 아닐 수 있다.

#### 6. Ephemeral fact

정의:

- 한 번 쓰고 수명이 짧은 사실

예시:

- 오늘 오후 2시에 잠깐 외출 예정
- 이번 한 번만 특정 톤으로 메일을 써 달라

권장 위치:

- session context 또는 raw history

원칙:

- 장기 기억으로 자동 승격하지 않는다.

#### 7. Sensitive restricted memory

정의:

- 도움은 되지만 잘못 저장하면 위험한 정보

예시:

- 건강 상태
- 개인 관계 이슈
- 금융/계정 관련 세부 정보

원칙:

- 기본은 장기 저장에 보수적으로 접근
- 필요하면 요약형 힌트만 남기고 raw detail은 저장하지 않음

## 6. Memory 승격 규칙 초안

아래 질문을 통과하는 정보만 장기 memory 후보로 보는 것이 적절하다.

1. 이 정보가 다음 주나 다음 달에도 도움이 되는가?
2. 이 정보가 owner 개인의 안정적 특성이나 지속 프로젝트와 연결되는가?
3. session을 넘어 assistant continuity에 실제로 도움 되는가?
4. 잘못 저장됐을 때 사용자 피해가 크지 않은가?
5. raw detail이 아니라 정제된 요약으로 표현 가능한가?

위 질문 중 1, 2, 3이 약하면 `history.jsonl` 또는 session-local에 머무는 쪽이 맞다.

## 7. Memory 수정과 삭제 규칙 초안

개인비서 memory는 저장만큼 수정과 삭제가 중요하다.

### 7.1 수정이 필요한 경우

- 사용자가 명시적으로 정정함
- 과거 preference가 더 이상 맞지 않음
- project context가 종료되었거나 바뀜
- 사람/관계 정보가 오래되어 오히려 오판을 유도함

### 7.2 권장 사용자 표현

향후 user-facing command 또는 자연어 패턴은 아래를 지원하는 편이 좋다.

- 이건 기억해
- 이건 잊어
- 이건 내 기본 선호가 아니야
- 앞으로는 이렇게 처리해
- 이 프로젝트는 끝났어

### 7.3 삭제보다 강등이 나은 경우

모든 정보를 즉시 삭제하기보다 아래 방식이 더 나을 수 있다.

- stable preference에서 ephemeral note로 강등
- active project에서 archived project로 이동
- 사람 관련 raw fact를 high-level summary로 축약

## 8. Task, Approval, Proactive와의 연결

### 8.1 Task와의 연결

task layer는 아래 memory를 자주 참조하게 된다.

- preference memory
- project memory
- people/contact memory
- routine memory

반대로 active task state 자체는 memory 본문보다 task store 또는 session metadata에 더 가깝다.

### 8.2 Approval과의 연결

approval policy는 owner profile의 아래 요소와 연결될 수 있다.

- 어떤 action은 항상 승인이 필요한가
- 어떤 채널에서 승인을 선호하는가
- 메일 발송/일정 생성 같은 외부 변경 작업에 어느 정도 보수성을 둘 것인가

단, approval result 그 자체를 전부 memory로 올리는 것은 바람직하지 않다. 장기적으로 의미 있는 policy change만 memory 후보가 된다.

### 8.3 Proactive와의 연결

proactive surface는 owner profile이 없으면 오히려 방해가 되기 쉽다.

필수 연결 항목:

- briefing 선호 시간
- quiet hours
- reminder 빈도 허용치
- 어떤 종류의 proactive alert를 가치 있게 느끼는가

즉, proactive는 단순 scheduler 기능이 아니라 owner profile을 기반으로 조정되는 assistant behavior여야 한다.

## 9. 현재 기준 구현 우선순위

### Phase 1

- `USER.md`, `memory/MEMORY.md`, `history.jsonl`, continuity metadata의 책임 경계 문서화
- owner profile 핵심 범주 확정
- memory taxonomy 초안 확정

### Phase 2

- Dream이 memory type을 더 구분해서 반영하도록 기준 추가 검토
- owner profile에서 task/approval/proactive가 읽을 핵심 항목 정의
- WebUI에 owner-aware summary 또는 assistant preference view 노출 검토

### Phase 3

- Gmail/Calendar domain action이 실제로 참조할 preference field 연결
- task 모델과 active task memory 경계 확정
- 필요 시 lightweight structured metadata 보조층 도입

## 10. 한 줄 결론

`canonical owner`는 continuity의 시작점이고, `owner profile + memory taxonomy`는 개인비서 제품 레이어의 핵심 뼈대다. 지금 Nanobot에 필요한 것은 거대한 user system이 아니라, 이미 있는 memory 구조 위에 `무엇을 어떤 등급의 기억으로 보고 어디에 둘지`를 명확히 세우는 일이다.