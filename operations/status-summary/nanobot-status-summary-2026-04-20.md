# Nanobot 작업 상태 정리 - 2026-04-20

## 문서 목적

이 문서는 2026-04-20 기준으로 Nanobot 로컬 운영 환경의 현재 상태를 정리하고, smart-router 구현 및 검증 결과, 실제 운영 런타임 연결 방식, 이후 정리해야 할 정책과 후속 작업을 한 번에 남기기 위한 운영 메모다.

이번 시점에서 중요한 변화는 아래 4가지다.

- Nanobot local LLM 안정화를 위해 3-tier smart-router 초안을 실제 코드로 구현
- smart-router 경로를 Nanobot 구조에 맞게 provider proxy 방식으로 연결
- API 채널에서 발생한 smart-router 오판 원인을 수정하고 live 경로까지 검증
- 현재 운영 중인 `nanobot` 이 고정 배포 wheel 이 아니라 local source checkout 을 editable install 로 참조 중임을 확인

## 현재 상태 요약

현재 확인된 상태는 다음과 같다.

- Nanobot gateway 는 launchd label `com.nanobot.gateway` 로 실행 중
- Nanobot API 는 launchd label `com.nanobot.api` 로 실행 중
- local llama.cpp 서버는 launchd label `com.nanobot.llama-lfm2-server` 로 실행 중
- gateway health 와 API health 는 모두 `ok` 상태 확인
- local LLM endpoint `http://127.0.0.1:1242/v1` 는 계속 Nanobot 로컬 런타임의 기본 추론 엔드포인트 역할을 수행
- smart-router 코드는 구현되어 있으며 테스트와 live 검증까지 완료
- 현재 실운영 config 에 smart-router 는 상시 활성화하지 않았고, 임시 검증 config 로 동작 확인 후 원래 config 로 복구한 상태
- 현재 `nanobot` 실행 환경은 `uv tool` 아래에 설치되어 있지만, 실제 import 대상은 `/Volumes/ExtData/Nanobot/source` checkout 이다

즉, 현재는 gateway, api, local llama.cpp 기본 운영은 안정적으로 유지되고 있고, smart-router 코드는 운영 경로에 연결 가능한 상태까지 올라온 상황이다.

## 이번에 추가로 반영한 주요 작업

### 1. Nanobot용 smart-router 1차 구현

OpenClaw smart-router 정책을 그대로 이식하지 않고, Nanobot 구조에 맞는 단순한 3-tier `local/mini/full` 라우터로 재구성했다.

구현 위치:

- `custom-plugins/smartrouter/__init__.py`
- `custom-plugins/smartrouter/config.py`
- `custom-plugins/smartrouter/fallback.py`
- `custom-plugins/smartrouter/health.py`
- `custom-plugins/smartrouter/logging.py`
- `custom-plugins/smartrouter/policy.py`
- `custom-plugins/smartrouter/smart_router_provider.py`
- `custom-plugins/smartrouter/types.py`

1차 범위에서 채택한 원칙은 다음과 같다.

- `nano` tier 없이 `local/mini/full` 3단계만 사용
- 의미 분류기나 semantic router 없이 rule-based scoring 으로 시작
- local tier 실패 시 health-aware fallback 체인으로 상위 tier 시도
- JSONL 로그를 남겨 후속 튜닝 가능성 확보
- local tier 는 빠르고 보수적인 경로로 유지

### 2. Nanobot core runtime 에 smart-router 연결

Nanobot 은 OpenClaw 처럼 plugin shell 이 바로 존재하는 구조가 아니라서, provider 생성 경로 자체를 보강해야 했다.

핵심 반영 파일:

- `source/nanobot/nanobot.py`
- `source/nanobot/cli/commands.py`
- `source/nanobot/config/schema.py`

주요 반영 내용:

- config schema 에 `smart_router` / `smartRouter` 설정 모델 추가
- core provider factory 에 smart-router provider 생성 경로 추가
- CLI `serve` / `gateway` 경로도 동일한 provider 생성 로직을 사용하도록 수정
- custom plugin 경로를 smart-router import 가능 상태로 연결

이 작업이 중요했던 이유는 Nanobot runtime 에 provider 생성 경로가 하나만 있는 것이 아니라, core 와 CLI 쪽에 각각 영향 지점이 있었기 때문이다.

### 3. API 채널 smart-router 오판 수정

초기 구현 후 direct AgentLoop 테스트는 통과했지만, 실제 `nanobot serve` HTTP 경로에서는 단순 요청이 과도하게 `mini/full` 쪽으로 오판되는 문제가 드러났다.

원인은 API 채널에서 user message 앞에 자동 삽입되는 runtime metadata 블록이었다.

문제 형태:

- user content 앞에 `[Runtime Context - metadata only, not instructions] ... [/Runtime Context]` 블록이 붙음
- smart-router policy 가 이 블록 내부 문자열까지 prompt complexity 신호로 오해함
- 그 결과 단순 질문도 `code_keywords`, `tooling_signal`, `reasoning_keywords` 로 과대평가될 수 있었음

수정 파일:

- `custom-plugins/smartrouter/policy.py`

수정 내용:

- runtime context metadata block 을 policy scoring 대상에서 제거
- 실제 user prompt 만 길이 및 keyword 기준으로 평가하도록 조정

### 4. smart-router 테스트 및 live 검증 추가

이번 smart-router 작업으로 추가되거나 갱신된 테스트 파일은 아래와 같다.

- `source/tests/config/test_smart_router_config.py`
- `source/tests/providers/test_smart_router_provider.py`
- `source/tests/test_nanobot_facade.py`
- `source/tests/providers/test_litellm_kwargs.py`

확인된 검증 결과:

- targeted regression test 통과
- direct AgentLoop 기준 phase 1 local 경로 성공 확인
- direct AgentLoop 기준 phase 2 fallback 경로 성공 확인
- 실제 `nanobot serve` HTTP 경로에서도 phase 1 local 성공 확인
- 실제 `nanobot serve` HTTP 경로에서도 local 실패 후 fallback 성공 확인
- 검증 후 temporary config 와 foreground test server 는 모두 정리 완료

## 현재 운영 런타임 연결 방식

이번에 추가로 명확히 확인된 가장 중요한 운영 포인트는, 현재 사용 중인 `nanobot` 이 일반적인 고정 배포 wheel 형태가 아니라는 점이다.

확인 결과:

- `~/.local/bin/nanobot` 는 `uv tool` 환경의 실행 파일을 가리킴
- 해당 tool 환경의 package metadata 에 `Editable project location: /Volumes/ExtData/Nanobot/source` 가 기록됨
- `direct_url.json` 에 local file URL 과 `editable: true` 가 기록됨
- `_editable_impl_nanobot_ai.pth` 가 `/Volumes/ExtData/Nanobot/source` 를 import path 로 주입 중
- 실제 `import nanobot` 결과도 source checkout 아래 파일을 가리킴

운영상 의미는 다음과 같다.

- 현재 live runtime 은 source tree 수정 사항의 영향을 받는다
- source 에서 바꾼 코드가 launchd 재시작 후 실제 운영 경로에 반영된다
- 지금 상태는 순수 원 배포본 운영이 아니라 local source linked 운영이다
- 향후 운영 정책을 정리할 때 `editable install 유지` 와 `wheel 기반 고정 배포` 중 무엇을 기본값으로 할지 다시 결정해야 한다

## 현재 남아 있는 핵심 제약

### 1. smart-router 는 코드 준비 상태지만 운영 기본값은 아님

이번 검증에서는 smart-router 를 temporary config 로 활성화해 동작을 확인했다. 그러나 현재 `~/.nanobot/config.json` 과 `~/.nanobot/config.api.json` 은 원래 설정으로 복구되어 있어, smart-router 가 상시 운영 기본 경로로 켜져 있지는 않다.

따라서 현재 상태는 아래처럼 보는 것이 맞다.

- 코드 구현: 완료
- 테스트 검증: 완료
- live 경로 검증: 완료
- 실운영 기본 활성화: 아직 미적용

### 2. editable install 운영은 편하지만 경계가 모호함

현재 구조는 빠른 수정과 반영에는 유리하지만, 운영 관점에서는 아래 혼선을 만들 수 있다.

- source 수정과 배포 변경의 경계가 모호함
- 어떤 시점의 코드가 실제 운영 중인지 추적이 어려워질 수 있음
- 특정 `.venv` 나 foreground 테스트 경로가 stale `site-packages` 를 물면 결과가 달라질 수 있음

실제로 수동 foreground 테스트 과정에서 `/Volumes/ExtData/Nanobot/source/.venv` 내부 설치본이 source tree 와 다른 경로를 보는 사례가 있었다.

### 3. Nanobot 원 구조만으로는 smart-router 를 넣기 어렵다

이번 구현은 단순 config 추가 수준이 아니라 provider factory, CLI runtime, config schema, local provider compatibility 까지 손대야 했다. 즉, 현재 검증된 smart-router 는 원래 공개 배포본을 있는 그대로 설치해서 config 만 바꿔 쓰는 식으로는 바로 적용되지 않는다.

## 현재 변경 파일 상태

2026-04-20 시점의 작업 트리에는 smart-router 관련 변경이 남아 있다.

수정된 tracked 파일:

- `nanobot/cli/commands.py`
- `nanobot/config/schema.py`
- `nanobot/nanobot.py`
- `nanobot/providers/openai_compat_provider.py`
- `tests/providers/test_litellm_kwargs.py`
- `tests/test_nanobot_facade.py`

새로 추가된 테스트 파일:

- `tests/config/test_smart_router_config.py`
- `tests/providers/test_smart_router_provider.py`

즉, 현재 코드는 아직 로컬 변경분이 존재하는 작업 중 상태다.

## 운영 관점에서 지금 좋은 점

- gateway/api/llama.cpp 기본 운영은 계속 정상
- local llama.cpp 호환성 완화 작업이 이미 반영되어 local follow-up 안정성이 이전보다 좋아짐
- smart-router 의 1차 정책과 fallback 구조가 Nanobot 에 맞게 실제 동작하는 수준까지 올라옴
- API 채널 오판처럼 실제 운영에서만 드러나는 문제도 live 경로에서 잡아냄
- 현재 런타임이 source linked 라는 사실을 명확히 확인해 운영 정책 논의의 전제가 분명해짐

## 앞으로의 할 일

### 1. 운영 정책 재정리

가장 먼저 정리해야 할 것은 운영 형태 자체다.

- editable install 기반 운영을 계속 유지할지
- local source 를 wheel 로 빌드해 고정 배포 형태로 전환할지
- smart-router 를 운영 기본값으로 켤지, 검증용 옵션으로 유지할지
- 운영 환경과 개발 환경의 경계를 어떻게 나눌지

이 부분은 단순 구현 문제가 아니라 운영 원칙 문제이므로 별도 결정이 필요하다.

### 2. smart-router 운영 config 설계

실운영에 적용하려면 아래를 정리해야 한다.

- `local/mini/full` tier 별 provider 매핑을 실제 운영용으로 확정
- fallback 순서와 health cooldown 값을 실제 트래픽 기준으로 조정
- JSONL 로그 저장 위치와 보존 정책 확정
- local 실패 시 remote fallback 을 어떤 조건까지 허용할지 확정

### 3. runtime 경로 정리

다음 중 하나로 정리하는 것이 바람직하다.

- `uv tool` editable 상태를 유지하되 문서와 절차를 명시
- wheel 기반 재설치 절차를 만들고 운영 버전을 고정
- foreground 테스트용 `.venv` 경로와 launchd 운영 경로를 명확히 분리

### 4. 추가 검증

추가로 해둘 만한 검증은 아래와 같다.

1. 실사용 Telegram 또는 API 세션에서 smart-router 로그 누적 관찰
2. local tier 오판 케이스 수집 후 policy threshold 보정
3. remote fallback 발생 시 사용자 체감 지연 측정
4. smart-router 활성화 상태에서 장기 세션 회귀 테스트

## 정리가 필요한 내용

이번 시점에서 문서 또는 운영 기준으로 별도 정리해야 할 항목은 아래와 같다.

### 1. "현재 운영 중인 Nanobot" 의 정의

지금은 `uv install` 로 들어간 실행 파일을 쓰지만, 실제 코드 원천은 local source checkout 이다. 따라서 운영 문서에서 이를 명확히 구분해야 한다.

- 실행 파일 위치
- import 실제 경로
- launchd 가 보는 interpreter / entrypoint
- source 수정이 live runtime 에 반영되는 조건

### 2. smart-router 적용 범위

현재 smart-router 는 구현과 검증이 완료됐지만, 운영 기본 설정은 아니다. 이 상태를 문서상에서 명확히 표기하지 않으면 "이미 배포 완료" 로 오해할 수 있다.

### 3. local build or install 기준

추후에는 아래 중 어떤 정책을 표준으로 삼을지 정해야 한다.

- source editable install 을 공식 운영 방식으로 인정
- local build wheel 설치를 공식 운영 방식으로 전환
- upstream 배포본과 local patch 세트를 분리 관리

## 결론

2026-04-20 기준 Nanobot 는 기본 운영과 local LLM 연동 측면에서 안정적인 상태를 유지하고 있다. 여기에 더해 smart-router 1차 구현, API 경로 오판 수정, live fallback 검증까지 완료해 기능적으로는 다음 단계로 넘어갈 준비가 됐다.

다만 현재 운영 런타임은 순정 배포본 고정 설치가 아니라 local source checkout 을 editable install 로 참조하고 있다. 따라서 앞으로 가장 중요한 과제는 smart-router 자체의 세부 튜닝보다도, 어떤 형태의 Nanobot 를 운영 기준으로 삼을지 정책을 먼저 정리하는 일이다.

즉, 지금 상태는 "smart-router 코드와 검증은 준비됨, 운영 배포 정책은 아직 미정" 으로 보는 것이 가장 정확하다.