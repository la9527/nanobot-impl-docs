# WebUI Context Window Indicator Workplan

이 문서는 WebUI 각 채팅창 상단 오른쪽의 `context window indicator` 기능에 대한 현재 구현 기준과, 마지막 refinement 결과를 정리한 implementation note 이다.

현재 상태는 아래로 본다.

- `session.metadata.context_window` 계약은 websocket native 와 linked Telegram session 에 공통 반영되어 있다.
- header 우측 indicator 는 `textless circular graph` 로 축소되었고 detail disclosure 와 연결되어 있다.
- WebUI 첫 진입 시 summary 가 비어 있으면 session route 에서 lazy hydrate 한다.
- detail disclosure 는 `상태 pill + usage card + secondary metadata` 중심으로 간결화되었다.
- healthy 상태에서는 설명성 guidance 를 숨기고, warning / critical 일 때만 경고 문구를 강조해서 노출한다.

## 1. 현재 목표

사용자가 각 채팅창에서 아래를 거의 즉시 파악할 수 있게 만든다.

- 현재 대화가 컨텍스트를 얼마나 점유했는가
- 다음 응답을 위해 예약된 여유가 어느 정도인가
- 곧 잘림 또는 aggressive truncation 위험이 있는가
- 이 값이 어떤 target / model 기준으로 계산된 것인가

이번 구현의 핵심은 `더 많이 보여주는 것` 이 아니라 `더 덜 방해하면서 정확한 상태를 먼저 보여주는 것` 이다.

## 2. 현재 코드 앵커

- `source/webui/src/components/thread/ThreadHeader.tsx`
  - indicator 를 theme/settings 옆에 배치하는 실제 header slot 이다.
- `source/webui/src/components/thread/ThreadShell.tsx`
  - session metadata 와 live metadata 를 합쳐 header 에 주입하는 지점이다.
- `source/webui/src/components/thread/ThreadContextWindowIndicator.tsx`
  - collapsed / expanded UI 를 직접 렌더링하는 주 컴포넌트다.
- `source/nanobot/agent/loop.py`
  - turn 완료 후 `session.metadata.context_window` 를 갱신하는 runtime owner 다.
- `source/nanobot/channels/websocket.py`
  - WebUI 의 `/api/sessions` / `/api/sessions/{key}/messages` 응답을 제어하므로, 첫 진입 시 lazy hydrate 를 넣기 가장 좋은 지점이다.

## 3. UX 방향 재정리

### 3.1 collapsed state

header 우측 control 은 `텍스트 chip` 이 아니라 `icon-only circular graph` 로 간다.

- 시각 형태는 작은 원형 ring indicator
- 기본 크기는 대략 32px~36px
- 텍스트는 header 에서 직접 노출하지 않는다
- ring 바깥쪽 arc 로 `사용량`, 옅은 secondary arc 로 `응답 예약분` 을 구분한다
- 색상만으로 healthy / warning / critical 신호를 준다
- 상세 수치는 클릭 후 detail panel 에서 확인한다

즉, header 에서는 `숫자 읽기` 보다 `위험도 glance` 가 우선이다.

### 3.2 expanded state

detail disclosure 는 작은 compact panel 을 유지한다.

- 상단 설명 문구와 별도 status chip 을 제거한다
- `컨텍스트 사용량`, `점유율`, `응답 예약`, `남은 여유` 를 핵심 수치로 둔다
- status text 는 usage graph 옆에 붙여 같은 정보 블록 안에서 읽히게 한다
- `타깃`, `모델`, `출처`, `갱신 시각` 은 secondary metadata 로 내려서 묶는다
- `max window` 를 따로 반복해서 카드로 늘리기보다 `used / max` headline 안에 포함한다
- `warning` / `critical` 일 때만 경고성 guidance 를 노출한다
- `updated_at` 는 detail 에서 local time `YYYY-MM-DD HH:mm:ss` 형식으로 보여준다

desktop 에서는 compact popover, mobile 에서는 bottom sheet fallback 을 유지한다.

## 4. 용어 기준

이번 iteration 에서는 `예산` 표현을 쓰지 않는다.

권장 용어:

- `컨텍스트 사용량`
- `점유율`
- `사용 입력`
- `응답 예약`
- `남은 여유`

피해야 할 표현:

- `예산`
- `reserve` 원문 노출
- 같은 의미를 box 안에서 반복하는 중복 문구

## 5. 데이터 계약 기준

기본 contract 는 그대로 `session.metadata.context_window` 를 사용한다.

권장 shape 예시:

```json
{
  "context_window": {
    "max_tokens": 65536,
    "used_input_tokens": 27900,
    "reserved_output_tokens": 8192,
    "available_tokens": 29444,
    "usage_ratio": 0.551,
    "status": "healthy",
    "source": "estimated",
    "active_target": "smart-router",
    "resolved_model": "smart-router",
    "updated_at": "2026-05-13T11:29:04Z"
  }
}
```

이번 refinement 에서 추가로 중요한 점은 아래다.

- 새 turn 이 없더라도, WebUI 가 session detail 을 처음 읽을 때 summary 가 비어 있으면 lazy hydrate 한다.
- 즉 `turn 완료 후 저장` 과 `첫 진입 시 보강` 을 같이 가져간다.

## 6. 구현 결과 요약

### Phase A. initial-load hydrate

첫 진입 시 `컨텍스트 --` 로 남아 있는 시간을 줄인다.

- `/api/sessions/{key}/messages` 응답 직전에 `context_window` 가 없으면 lazy summary 생성
- 필요 시 같은 helper 를 session list metadata 에도 재사용 가능하게 유지
- source 는 여전히 `estimated` 로 둔다

### Phase B. header indicator slimming

`ThreadContextWindowIndicator` 를 icon-only circular graph 로 교체한다.

- pill width 제거
- visible text 제거
- 현재 버튼 footprint 를 거의 절반 수준으로 축소
- focus / keyboard / status color semantics 는 유지

### Phase C. detail panel condensation

detail panel 을 재배치했다.

- width / height 를 줄였다
- duplicated cards 를 제거했다
- numeric summary 3~4개와 secondary metadata 만 남겼다
- 상태 pill 은 usage graph 옆으로 옮겼다
- healthy 상태의 기본 설명 문구는 제거했다
- warning / critical 상태에서만 경고 문구를 보여준다
- `updated_at` 는 local time 포맷으로 변환한다

### Phase D. i18n and tests

- `en` / `ko` locale key 를 새 용어 기준과 경고 노출 정책에 맞춰 정리했다
- component test expectation 을 `textless header indicator + condensed detail panel + local timestamp` 기준으로 수정했다
- backend 는 session route lazy hydrate 를 focused test 로 고정했다

## 7. 완료 기준

아래를 이번 refinement 완료 기준으로 유지한다.

- header indicator 가 icon-only circular graph 로 보인다
- 이전 pill UI 보다 footprint 가 확실히 줄어든다
- detail panel 이 더 작고, 중복 수치가 줄어든다
- `예산` / `reserve` 같은 어색한 표현이 UI 에 남지 않는다
- 기존 세션도 첫 진입 후 새 turn 없이 context summary 를 받을 수 있다
- websocket native 와 linked Telegram 이 같은 metadata contract 를 유지한다
- healthy 상태에서는 상단 설명형 guidance 가 보이지 않는다
- warning / critical 에서만 경고 문구가 강조 노출된다
- 갱신 시각이 local time `YYYY-MM-DD HH:mm:ss` 형식으로 보인다

## 8. 후속 제안

- indicator hover 시 desktop tooltip 에 간단한 status text 노출 검토
- 향후 dashboard / session list row 에 같은 circular indicator 재사용 검토
- 장기적으로는 `최근 turn 증가량` 이나 `compaction 여부` 를 power-user detail 로 추가 검토