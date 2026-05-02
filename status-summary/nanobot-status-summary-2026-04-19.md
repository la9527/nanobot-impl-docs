# Nanobot 작업 상태 정리 - 2026-04-19

## 문서 목적

이 문서는 2026-04-19 기준으로 Nanobot 로컬 운영 환경에 대해 지금까지 진행한 작업, 현재 실행 상태, 확인된 이슈와 수정 사항, 이후 남은 작업을 한 번에 정리하기 위한 운영 메모다.

이번 작업의 중심 목표는 아래 4가지였다.

- 기존 ZeroClaw 운영 흔적에서 Telegram 설정을 복구해 Nanobot 에 재적용
- Nanobot 운영 스크립트를 단순화해 gateway/api/llama.cpp 상태를 쉽게 관리
- local llama.cpp 런타임을 Nanobot 기준으로 관리 가능하게 전환
- gateway tool-call 경로에서 발생하던 local LLM parse error 를 줄이기 위한 호환성 수정

## 현재 상태 요약

현재 확인된 상태는 다음과 같다.

- Nanobot gateway 는 launchd label `com.nanobot.gateway` 로 실행 중
- Nanobot API 는 launchd label `com.nanobot.api` 로 실행 중
- local llama.cpp 서버는 launchd label `com.nanobot.llama-lfm2-server` 로 실행 중
- local LLM endpoint `http://127.0.0.1:1242/v1` 는 정상 응답 확인
- Nanobot 기본 provider 는 `vllm` 설정을 사용하지만 실제 연결 대상은 local llama.cpp OpenAI-compatible endpoint
- Telegram 채널은 활성화되어 있으며 실제 수신 및 응답 로그 확인 완료
- active Nanobot runtime 은 `/Volumes/ExtData/Nanobot/source` 를 editable install 한 source tree 기준으로 동작 중

즉, 현재는 gateway, api, local llama.cpp 3개 축이 모두 Nanobot 기준으로 운영 가능한 상태다.

## 이번에 반영한 주요 작업

### 1. Telegram 설정 복구 및 재적용

기존 `~/.zeroclaw` 설정 파일만으로는 Telegram 토큰을 직접 복구할 수 없었고, legacy 로그에서 실제 동작하던 값을 추적해 적용했다.

반영 결과:

- `~/.nanobot/nanobot.env` 에 Telegram 관련 값 반영
- `~/.nanobot/config.json` 에 Telegram enabled 상태 반영
- 허용 사용자 ID 설정 반영
- gateway 로그 기준 실제 메시지 수신 및 응답 확인

정리하면 Telegram 은 현재 설정 문제가 아니라, 정상 동작하는 상태로 확인됐다.

### 2. Nanobot 통합 운영 스크립트 추가

기존에는 gateway/api 운영 스크립트가 분산되어 있어 상태 확인과 재시작 흐름이 번거로웠다. 이를 정리하기 위해 아래 통합 스크립트를 추가했다.

- `infra/scripts/nanobot.sh`

지원 기능:

- `start`
- `stop`
- `status`
- `install`
- `uninstall`
- `log`

대상:

- `all`
- `gateway`
- `api`

추가로 아래 사항도 반영했다.

- CLI 색상 헤더 정리
- macOS 에서 `tail` 인자 처리 문제 수정
- 로그 파일이 여러 개일 때도 `tail -F` 가 정상 동작하도록 개선

### 3. llama.cpp 운영 스크립트 및 LaunchAgent 추가

기존 local llama.cpp 는 legacy label `com.zeroclaw.llama-cpp` 로 살아 있었고, Nanobot 관점에서 직접 관리할 수 있는 상태가 아니었다. 이를 정리하기 위해 Nanobot 전용 llama 관리 스크립트를 추가했다.

추가 경로:

- `infra/scripts/llama/llama.sh`
- `infra/scripts/llama/start-llama-cpp-lfm2-server.sh`
- `infra/scripts/llama/start-llama-services.sh`
- `infra/scripts/llama/stop-llama-services.sh`
- `infra/scripts/llama/status-llama-services.sh`
- `infra/scripts/llama/install-launchd-services.sh`
- `infra/scripts/llama/uninstall-llama-services.sh`
- `infra/launchd/com.nanobot.llama-lfm2-server.plist`

이후 실제로 `llama.sh install` 을 통해 active label 을 아래와 같이 전환했다.

- 이전: `com.zeroclaw.llama-cpp`
- 현재: `com.nanobot.llama-lfm2-server`

전환 후에도 `127.0.0.1:1242/v1/models` 응답은 유지됐고, endpoint health 도 문제 없음을 확인했다.

### 4. local provider parse error 완화 수정

문제 증상은 gateway 에이전트 루프에서 tool-call 이 일어난 뒤, 후속 요청에서 local llama.cpp 쪽이 `500 Internal Server Error` 와 parse error 를 내는 것이었다.

확인한 핵심 포인트:

- 단순 chat completion 은 정상
- local llama.cpp 프로세스도 정상
- 문제는 gateway tool-call 후속 턴에서만 재현
- 로그상 `<|tool_call_end|>` 류 파싱 실패 흔적 존재

원인은 local OpenAI-compatible endpoint 에 대해, 이미 완료된 tool-call 이력을 구조화된 `assistant.tool_calls` 와 `tool` 메시지 그대로 다시 보내던 흐름 때문으로 판단했다.

이를 줄이기 위해 다음 파일을 수정했다.

- `source/nanobot/providers/openai_compat_provider.py`

수정 내용:

- local provider (`spec.is_local`) 인 경우 후속 요청 전에 완료된 tool-call history 를 평탄화
- 완료된 assistant tool-call 메시지는 텍스트 설명 형태로 변환
- tool result 메시지는 일반 user-side 대화 이력 형태로 변환
- 아직 완료되지 않은 pending tool-call 은 기존 구조를 유지

테스트도 추가했다.

- `source/tests/providers/test_litellm_kwargs.py`

검증 결과:

- 추가한 sanitize/flatten 테스트 통과
- active runtime 에 source tree 를 editable install 로 연결
- gateway/api 재시작 후 수정 코드가 실제 런타임에 반영됨을 확인
- flattened history 를 사용한 local endpoint follow-up call 이 parse error 없이 수용됨을 확인

## 현재 설정 및 운영 포인트

### LaunchAgent 기준 활성 label

- `com.nanobot.gateway`
- `com.nanobot.api`
- `com.nanobot.llama-lfm2-server`

### 주요 엔드포인트

- Gateway health: `http://127.0.0.1:18790/health`
- API health: `http://127.0.0.1:8900/health`
- LLM models: `http://127.0.0.1:1242/v1/models`

### 주 사용 스크립트

gateway/api 관리:

```bash
/Volumes/ExtData/Nanobot/infra/scripts/nanobot.sh status
/Volumes/ExtData/Nanobot/infra/scripts/nanobot.sh start all
/Volumes/ExtData/Nanobot/infra/scripts/nanobot.sh stop all
/Volumes/ExtData/Nanobot/infra/scripts/nanobot.sh log
```

llama.cpp 관리:

```bash
/Volumes/ExtData/Nanobot/infra/scripts/llama/llama.sh status
/Volumes/ExtData/Nanobot/infra/scripts/llama/llama.sh start
/Volumes/ExtData/Nanobot/infra/scripts/llama/llama.sh stop
/Volumes/ExtData/Nanobot/infra/scripts/llama/llama.sh log
```

### 현재 runtime 반영 방식

현재 `nanobot` 실행 바이너리는 `uv` tool 환경 아래에 있고, active runtime 은 source checkout 을 editable install 한 상태다.

운영상 의미는 다음과 같다.

- `/Volumes/ExtData/Nanobot/source` 수정 사항이 live runtime 에 반영 가능
- launchd 재시작 시에도 현재 연결 상태가 유지되는지 수시 확인 필요
- 향후 runtime 경로를 더 명시적으로 고정하는 작업이 있으면 운영 안정성이 더 좋아질 수 있음

## 확인된 리스크와 주의점

### 1. parse error 는 완전히 "모든 경우" 해결됐다고 단정하면 안 됨

이번 수정은 local llama.cpp 가 이미 종료된 tool-call history 를 파싱하다 실패하는 대표 경로를 줄이는 목적이다. 따라서 같은 유형의 500 parse error 재발 가능성은 낮아졌지만, 모델이 새로운 tool-call 을 어떤 품질로 생성하는지까지 보장하는 수정은 아니다.

### 2. active runtime 과 source tree 연결 상태를 의식해야 함

launchd 는 `nanobot` CLI 를 실행하고, 실제 실행 환경은 `uv` tool 환경이다. 따라서 source tree 만 수정하고 runtime 이 그 코드를 보지 않으면 수정이 반영되지 않는다.

현재는 editable install 로 연결해 두었지만, 이후 환경 재구성 시 이 연결이 끊기지 않는지 확인이 필요하다.

### 3. legacy 흔적은 일부 디스크에 남아 있을 수 있음

active label 은 Nanobot 기준으로 전환했지만, legacy plist 나 로그 파일 일부는 디스크에 남아 있을 수 있다. 실행 주체와 참고 파일을 혼동하지 않도록 상태 확인 시 active launchd label 기준으로 보는 것이 안전하다.

## 이번 작업으로 실제 좋아진 점

- Telegram 채널을 다시 사용할 수 있게 됨
- gateway/api/llama 운영 명령이 훨씬 단순해짐
- local llama.cpp 프로세스가 Nanobot 기준으로 관리 가능해짐
- parse error 원인을 단순 추정이 아니라 runtime/log/provider 레이어까지 좁혀서 수정함
- source tree 수정이 실제 live runtime 에 반영되도록 정리함

## 다음 작업 후보

나중에 이어서 볼 만한 작업은 아래와 같다.

1. 실제 Telegram 또는 gateway 실사용 턴 기준으로 parse error 재발 여부를 추가 검증
2. `nanobot` runtime 경로를 더 명시적으로 고정해 editable source 연결 의존성을 줄이기
3. local llama.cpp prompt/tool-call 호환성 정책을 더 정교하게 다듬기
4. 운영 문서에 install/recovery 절차를 더 체계적으로 분리 정리하기

## 결론

2026-04-19 기준 Nanobot 로컬 운영 환경은 이전보다 훨씬 정리된 상태다. Telegram, gateway, api, local llama.cpp 모두 Nanobot 기준으로 관리 가능한 구조를 갖췄고, 실제로 문제를 일으키던 local tool-call parse error 도 provider 레이어에서 완화 조치를 반영했다.

즉, 현재는 "기본 운영이 가능한 상태"까지는 확보됐고, 이후 작업은 안정화와 문서화, 추가 검증 중심으로 이어가면 된다.