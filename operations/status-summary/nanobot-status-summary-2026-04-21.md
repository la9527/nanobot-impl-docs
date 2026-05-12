# Nanobot 작업 상태 정리 - 2026-04-21

## 문서 목적

이 문서는 2026-04-21 기준으로 Nanobot 로컬 운영 환경과 최근 runtime plugin 확장 작업의 현재 상태를 한 번에 정리해 두기 위한 운영 메모다.

이번 시점에서 중요한 변화는 아래 9가지다.

- smart-router 를 source repo 안쪽 `custom-plugins/smartrouter` 기준으로 정리
- Nanobot 에 일반 runtime plugin registry 를 도입하고 smart-router 를 그 경로로 연결
- smart-router 설정 경로를 `plugins.smartrouter` 중심으로 수렴
- runtime plugin 에 provider/hook 외에 tool/init/status seam 을 추가
- CLI 에 `plugins list` 와 `plugins status` 기반 runtime plugin 가시성을 강화
- 실제 source-tree 기준 API 서버에서 smart-router 동작과 JSONL 로그를 다시 검증
- named model target 구조를 추가해 direct model 과 smart-router 를 같은 선택 표면으로 노출
- `/model` built-in command 와 Telegram/Discord command 노출을 추가
- `~/.nanobot/config.api.json`, `~/.nanobot/config.json` 에 live model target 설정을 반영하고 API/gateway 재기동으로 확인

## 현재 상태 요약

현재 확인된 상태는 다음과 같다.

- Nanobot gateway 는 launchd label `com.nanobot.gateway` 로 실행 중인 운영 경로를 유지
- Nanobot API 는 launchd label `com.nanobot.api` 기준 운영 경로를 유지
- local llama.cpp 서버는 launchd label `com.nanobot.llama-lfm2-server` 기준 운영 경로를 유지
- active `nanobot` runtime 은 여전히 `/Volumes/ExtData/Nanobot/source` 를 editable install 로 참조 중
- smart-router 는 더 이상 ad hoc source patch 가 아니라 runtime plugin registry 위에서 동작
- smart-router 의 주 설정 경로는 이제 `plugins.smartrouter` 이고, legacy `smartRouter` / `smart_router` 입력은 호환용으로만 유지
- CLI 에서 `nanobot plugins list --config ...`, `nanobot plugins status --config ...` 로 runtime plugin 상태를 직접 확인 가능
- named model target 은 `agents.defaults.modelSelection` 아래에서 관리되고, `/model` 로 세션별 override 가능
- live `config.api.json` 과 `config.json` 모두 `activeTarget=local-llm` + selectable `smart-router` 구조로 맞춤
- source-tree 기준 target 해석 결과는 `default`, `smart-router`, `local-llm` 세 항목으로 확인 완료
- source venv 재검증에서도 두 live config 모두 `TARGETS=default,smart-router,local-llm`, `ACTIVE=local-llm` 으로 확인 완료
- session metadata 에 `model_target=smart-router` 를 넣으면 active target 이 `smart-router` 로 바뀌고, direct target `local-llm` 적용 시에는 `plugins.smartrouter.enabled=False` 로 해석되는 것까지 확인 완료
- focused test 세트는 `92 passed` 로 확인 완료
- 관련 변경은 브랜치 `feature/nanobot-fork-runtime` 에 커밋 `439801b` 로 푸시 완료

즉, 현재는 smart-router 가 Nanobot core patch 덩어리가 아니라 runtime plugin 으로 정리된 상태이며, 설정, CLI, 테스트, 운영 문서까지 한 세트로 정돈된 시점이다.

## 이번에 반영한 주요 작업

### 1. smart-router 를 runtime plugin 구조로 일반화

기존에는 smart-router 가 Nanobot 에 억지로 연결된 특수 경로에 가까웠다. 이번에는 아래 구조를 중심으로 일반 runtime plugin surface 로 재정리했다.

핵심 경로:

- `source/nanobot/plugins/__init__.py`
- `source/nanobot/plugins/types.py`
- `source/nanobot/plugins/registry.py`
- `source/custom-plugins/smartrouter/__init__.py`
- `source/custom-plugins/smartrouter/config.py`

주요 결과:

- `smartrouter` 는 `register_plugin()` 으로 등록되는 일반 runtime plugin 이 됨
- provider 생성은 plugin registry 를 통해 수행
- enablement 판단도 plugin registry 가 담당
- status row 도 plugin 쪽에서 설명 가능해짐

### 2. smart-router 설정을 `plugins.smartrouter` 중심으로 수렴

이전까지는 `smartRouter` 루트 블록이 사실상 주 경로였다. 이번에는 `plugins.smartrouter` 를 주 경로로 바꾸고, 기존 설정 입력은 호환용으로만 남겼다.

반영 내용:

- `Config.plugins.smartrouter` 추가
- root `smart_router` / `smartRouter` 는 load 시 plugin 설정으로 동기화
- save 시에는 `plugins.smartrouter` 기준으로 직렬화
- smart-router 로더는 `plugins.smartrouter` 우선, legacy root 경로 fallback 순으로 해석

운영상 의미:

- 이후 runtime plugin 들이 같은 설정 구조를 따르기 쉬워짐
- smart-router 가 예외 케이스가 아니라 일반 plugin 패턴 쪽으로 이동함

### 3. runtime plugin seam 확장

기존 runtime plugin 은 사실상 provider builder 와 hook builder 정도만 가능한 최소 구조였다. 이번에는 아래 seam 을 추가했다.

- `build_tools`
- `initialize`
- `describe_status`

이로 인해 가능한 범위는 아래처럼 넓어졌다.

- runtime plugin 이 `AgentLoop.tools` 에 직접 tool 등록 가능
- loop 생성 직후 lightweight initialization 가능
- CLI status 표에서 plugin 이 자기 상태를 설명 가능

현재 smart-router 는 이 중 `describe_status` 를 사용한다. tool/init 은 아직 smart-router 에 직접 쓰지 않았지만, 후속 runtime plugin 확장 기반은 마련됐다.

### 4. CLI runtime plugin 관측성 보강

이번 작업으로 CLI 쪽에도 runtime plugin 중심 운영 확인 경로를 추가했다.

반영 명령:

```bash
nanobot plugins list --config <config-path>
nanobot plugins status --config <config-path>
```

현재 CLI 에서 확인 가능한 정보:

- plugin name
- kind
- source
- config path
- enabled 여부
- 짧은 status/detail 문자열

이 변화는 multi-instance 또는 source-tree 검증 시 어떤 config 가 실제로 plugin 을 켜고 있는지 빠르게 확인하는 데 의미가 크다.

### 5. SDK / CLI runtime plugin 초기화 경로 정리

provider 생성만 plugin registry 를 쓰고 loop 초기화가 빠지면 tool/init seam 이 무의미해진다. 이를 막기 위해 아래 경로에 runtime plugin initialization 을 연결했다.

- `source/nanobot/nanobot.py`
- `source/nanobot/cli/commands.py`

적용 대상:

- SDK `Nanobot.from_config()`
- CLI `serve`
- CLI `gateway`
- CLI `agent`

즉, source 기준으로 쓰는 주요 진입점은 모두 같은 runtime plugin initialization 흐름을 타게 됐다.

### 6. 테스트 및 live 검증 보강

이번 단계에서 보강된 테스트 영역은 아래와 같다.

- `tests/config/test_smart_router_config.py`
- `tests/config/test_config_migration.py`
- `tests/plugins/test_runtime_plugins.py`
- `tests/cli/test_commands.py`

최종 focused test 결과:

- `92 passed, 1 warning`

추가 live 검증 내용:

- `plugins.smartrouter` 임시 config 로 `plugins list/status --config` 확인
- source-tree 기준 API server 기동 확인
- `/health` 응답 확인
- `/v1/chat/completions` 실요청 확인
- smart-router JSONL 로그에 `requestedTier`, `finalTier`, `reasonCodes`, `attempts` 기록 확인

### 7. named model target catalog 와 `/model` 추가

이번 단계에서는 smart-router 를 hidden provider swap 이 아니라 selectable target 으로 취급하도록 구조를 넓혔다.

핵심 경로:

- `source/nanobot/model_targets.py`
- `source/nanobot/command/builtin.py`
- `source/nanobot/agent/loop.py`
- `source/nanobot/plugins/types.py`
- `source/nanobot/plugins/registry.py`
- `source/custom-plugins/smartrouter/__init__.py`

주요 결과:

- `RuntimePlugin` 은 이제 `build_model_targets` seam 으로 named target catalog 를 기여할 수 있음
- `smartrouter` plugin 은 `smart-router` target 을 직접 catalog 에 올림
- `AgentLoop` 는 session metadata 기준으로 active target 을 해석하고 turn 실행 직전에 provider/model 을 동적으로 결정함
- `/model` 은 현재 target 보기, 목록 보기, target 변경, session override 해제를 처리함
- `/status` 는 effective target 기준으로 model 정보를 보여주게 됨

즉, smart-router 와 direct provider/model target 이 같은 사용자 표면에 놓이기 시작한 시점이다.

### 8. 문서와 테스트 추가

이번 model target 작업과 함께 아래도 반영했다.

- `source/docs/MODEL_TARGETS.md` 추가
- `source/README.md` 에 `Model Targets` 링크와 `/model` 명령 설명 추가
- `tests/config/test_model_targets.py` 에 plugin-contributed target merge 테스트 추가
- `tests/plugins/test_runtime_plugins.py` 에 runtime plugin model target catalog 테스트 추가
- `tests/command/test_builtin_model.py` 로 `/model` 기본 동작 검증 유지

focused test 재검증 결과:

- `tests/plugins/test_runtime_plugins.py`
- `tests/config/test_model_targets.py`
- `tests/command/test_builtin_model.py`

위 세트는 `15 passed` 로 통과했다.

### 9. live config 반영 및 재기동 확인

이번에는 구현만 끝낸 것이 아니라 실제 사용자 config 에도 최소 구성을 넣었다.

반영 파일:

- `/Users/byoungyoungla/.nanobot/config.api.json`
- `/Users/byoungyoungla/.nanobot/config.json`

반영 방식:

- `agents.defaults.modelSelection.activeTarget=local-llm`
- named direct target `local-llm` 추가
- `plugins.smartrouter` 에 local/mini/full 을 모두 현재 `${LOCAL_LLM_MODEL}` 기준으로 채워 selectable `smart-router` target 이 보이게 구성

의도적으로 현재 기본 동작은 바꾸지 않았다.

- active target 은 여전히 local direct target
- `smart-router` 는 `/model smart-router` 로 명시 선택할 때만 사용

실제 확인 결과:

- `plugins list/status --config /Users/byoungyoungla/.nanobot/config.api.json` 기준 `smartrouter` enabled 확인
- 같은 방식으로 `config.json` 기준도 `smartrouter` enabled 확인
- source-tree Python 로 target 해석 시 `default`, `smart-router`, `local-llm` 확인
- source venv 기준 재검증에서도 API/gateway config 모두 `activeTarget=local-llm` 확인
- source venv 기준 session override 재검증에서도 `model_target=smart-router` 저장 시 active target 이 `smart-router` 로 전환되는 것 확인
- `infra/scripts/start-nanobot-services.sh` 후 `api: ok` 확인
- gateway 는 초기 확인 시 MCP/Telegram 재연결 중이라 `down` 이었지만, 곧이어 `gateway: ok` 로 회복됨
- gateway log 에 MCP tool 등록, Telegram bot 연결, port `18790` 시작 로그 확인
- live health endpoint 재검증에서 `http://127.0.0.1:8900/health`, `http://127.0.0.1:18790/health` 모두 `{"status":"ok"}` 반환 확인
- `http://127.0.0.1:8900/v1/models` 는 `LiquidAI/LFM2-24B-A2B-GGUF:Q4_0` 단일 model id 를 정상 반환

## 이번에 확인된 운영 포인트

### 1. source-tree 기준 CLI 검증 시 `PYTHONPATH` 가 중요함

이번 검증 과정에서 가장 실제적인 운영 포인트는, source checkout 바깥 디렉터리에서 source venv 의 `python -m nanobot.cli.commands` 를 실행할 때 import 경로가 섞일 수 있다는 점이었다.

실제로 아래 형태는 실패 또는 잘못된 설치본 참조를 만들 수 있었다.

```bash
/Volumes/ExtData/Nanobot/source/.venv/bin/python -m nanobot.cli.commands ...
```

안정적으로 source tree 를 보게 한 경로는 아래였다.

```bash
PYTHONPATH=/Volumes/ExtData/Nanobot/source \
  /Volumes/ExtData/Nanobot/source/.venv/bin/python -m nanobot.cli.commands ...
```

즉, source 기준 실운영 검증 문서에는 `PYTHONPATH` 포함 경로를 기본 예시로 남기는 편이 안전하다.

### 2. local provider 검증과 API key 검사 경계

smart-router 검증용 임시 config 에서는 모두 local vLLM endpoint 만 써도 provider 생성 시 API key 검사에 걸릴 수 있었다. 최종적으로는 source-tree import 경로를 명확히 맞춘 뒤 실제 live 검증을 통과시켰다.

운영상 의미:

- local endpoint 라도 provider 선택 경로가 정확히 어떤 spec 을 타는지 의식해야 함
- foreground 실험 config 는 운영 config 와 분리해서 짧게 쓰고 정리하는 편이 안전함

### 3. smart-router 는 이제 "구현 완료 + 기본 운영 미적용" 단계를 넘음

2026-04-20 메모 시점에는 smart-router 가 구현/검증은 되었지만 운영 기본 설정은 아니었다. 2026-04-21 시점에는 한 단계 더 진전되었다.

현재 상태는 아래처럼 보는 것이 맞다.

- smart-router 구현: 완료
- plugin registry 일반화: 완료
- `plugins.smartrouter` 설정 수렴: 완료
- CLI status 가시화: 완료
- source-tree live 검증: 완료
- remote credential 포함 full fallback 운영 검증: 아직 제한적

이번 model target 작업까지 포함하면 상태는 아래처럼 보는 것이 맞다.

- target catalog 구현: 완료
- `/model` built-in command: 완료
- live config 반영: 완료
- API/gateway 재기동 확인: 완료
- real remote multi-tier routing: 아직 placeholder local-only tier 값으로 검증

## 현재 남아 있는 제약과 리스크

### 1. full remote fallback 실운영 검증은 아직 제한적

이번 검증은 local-only tier 매핑으로도 smart-router 호출과 JSONL logging 자체는 확인할 수 있었다. 하지만 실제 `local -> mini -> full` fallback chain 을 운영 credential 과 함께 장기적으로 검증한 것은 아니다.

남은 확인 항목:

- real remote provider auth 연결 상태
- timeout / connection failure 시 mini 승격
- mini 실패 시 full 승격
- 체감 latency 및 fallback rate 기록

현재 live config 는 `smart-router` 를 selectable target 으로 노출하기 위해 local/mini/full 을 모두 같은 `${LOCAL_LLM_MODEL}` 로 채운 상태다. 즉, target surface 와 runtime wiring 검증은 끝났지만 실제 remote tier 품질 검증은 아직 아니다.

### 2. editable install 운영 경계는 여전히 중요함

active runtime 이 source checkout 을 editable install 로 참조하는 구조는 유지 중이다. 이 덕분에 빠른 수정은 가능하지만, 운영 기준선이 흐려질 수 있다는 점은 여전하다.

즉, 아래는 계속 관리 포인트다.

- launchd 가 보는 runtime 과 source tree 의 관계
- foreground `.venv` 검증 경로와 live runtime 경로 차이
- source 변경이 언제 실제 운영에 반영되는지에 대한 기준

### 3. plugin status 는 관측성이지 health probing 은 아님

이번에 추가한 `plugins status` 는 매우 유용하지만, 이것만으로 provider health 나 fallback readiness 를 보장하지는 않는다. 현재 status 는 설정/등록/초기화 관점의 상태 표시에 가깝다.

향후 필요 작업:

- health probe 기반 detail 추가
- logging file rotation / retention 기준
- route decision 분석용 통계 명령 또는 별도 스크립트

## 현재 기준 권장 운영 명령

source-tree runtime plugin 상태 확인:

```bash
PYTHONPATH=/Volumes/ExtData/Nanobot/source \
  /Volumes/ExtData/Nanobot/source/.venv/bin/python -m nanobot.cli.commands \
  plugins list --config /Users/byoungyoungla/.nanobot/config.api.json

PYTHONPATH=/Volumes/ExtData/Nanobot/source \
  /Volumes/ExtData/Nanobot/source/.venv/bin/python -m nanobot.cli.commands \
  plugins status --config /Users/byoungyoungla/.nanobot/config.api.json
```

source-tree API foreground 검증:

```bash
set -a && source /Users/byoungyoungla/.nanobot/nanobot.env && set +a
PYTHONPATH=/Volumes/ExtData/Nanobot/source \
  /Volumes/ExtData/Nanobot/source/.venv/bin/python -m nanobot.cli.commands serve \
  --config /Users/byoungyoungla/.nanobot/config.api.json \
  --host 127.0.0.1 \
  --port 8911
```

health / completion / log 확인:

```bash
curl -s http://127.0.0.1:8911/health

curl -s http://127.0.0.1:8911/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "LiquidAI/LFM2-24B-A2B-GGUF:Q4_0",
    "messages": [
      {"role": "user", "content": "Reply with exactly SMART_ROUTER_PLUGIN_OK"}
    ]
  }'

tail -n 5 ~/.nanobot/logs/smart-router.jsonl
```

## 다음 작업 후보

지금 시점에서 후속 우선순위는 아래 순서가 적절하다.

1. 실제 remote credential 을 붙인 `local -> mini -> full` fallback 검증
2. smart-router JSONL 로그의 집계/분석 스크립트 또는 runbook 정리
3. `plugins.smartrouter` 기준 example config/template 추가

## 추가 업데이트 - usage footer / runtime command 검증

2026-04-21 후속 작업으로 응답 상태 footer 와 `/usage` command 도 source tree 기준으로 반영하고 재검증했다.

이번에 추가로 확인된 사항은 아래와 같다.

- `source/nanobot/response_status.py` 를 추가해 usage normalization 과 compact footer formatting 을 공용화
- `source/nanobot/command/builtin.py` 에 `/usage off|tokens|full` 추가
- `/status` 는 이제 active target 과 current reply footer mode 도 함께 표시
- `source/nanobot/agent/loop.py` 는 non-streaming 채널/CLI 응답에만 compact footer 를 부착하고, OpenAI-compatible API 응답은 본문 대신 JSON `usage` 필드만 유지
- `source/nanobot/channels/discord.py` 에 Discord app command `/usage` 노출 추가
- `source/README.md`, `source/docs/MODEL_TARGETS.md` 에 footer scope 와 운영 예시를 반영

focused test 재검증 결과:

- `tests/command/test_builtin_usage.py`
- `tests/command/test_builtin_model.py`
- `tests/cli/test_restart_command.py`
- `tests/test_openai_api.py`
- `tests/test_build_status.py`

위 세트는 source `.venv` 기준 `44 passed` 로 통과했다.

추가 live runtime 검증 결과:

- `Nanobot.from_config('/Users/byoungyoungla/.nanobot/config.json')` 기준 session `runtime:model-e2e` 에서 `/model smart-router` 실행 성공
- 이어서 `/status` 에서 `Target: smart-router`, `Reply footer: off` 확인
- `/usage tokens` 실행 후 일반 질의 응답 말미에 `Status: model=smart-router | tokens=3179 in/2 out` footer 부착 확인

추가로 2026-04-22 기준 live smart-router remote tier 설정도 실제 OpenAI 기준으로 반영했다.

- `~/.nanobot/backups/2026-04-22-smart-router-openai/` 아래에 `nanobot.env.bak`, `config.json.bak`, `config.api.json.bak` 백업 생성
- live `config.json`, `config.api.json` 의 `plugins.smartrouter.mini/full` 을 각각 `openai/gpt-5.4-mini-2026-03-17`, `openai/gpt-5.4-2026-03-05` 로 변경
- `providers.openai.apiKey=${OPENAI_API_KEY}` 를 live config 두 곳에 반영
- `stop-nanobot-services.sh && start-nanobot-services.sh` 후 health endpoint `18790`, `8900` 이 둘 다 `{"status":"ok"}` 로 복구 확인

같은 날 이어서 smart-router real remote fallback 검증도 마쳤다.

- live config 그대로 `Nanobot.from_config('/Users/byoungyoungla/.nanobot/config.json')` 기준 `/model smart-router` 세션에서 local/mini/full 유도 프롬프트를 각각 실행했고, `~/.nanobot/logs/smart-router.jsonl` 에서 `requestedTier/finalTier` 가 `local/local`, `mini/mini`, `full/full` 로 확인됐다.
- 저장된 config 는 그대로 둔 채 메모리 복사본으로만 fault injection 을 넣어 `local -> mini`, `mini -> full` fallback 도 확인했다.
- `local -> mini` 는 `vllm apiBase` 를 invalid localhost port 로 바꿔 local 실패를 유도했고, `mini -> full` 은 mini model 을 접근 불가한 `gpt-4.1-mini` 로 바꿔 유도했다.
- 이 세션 기준 remote fallback 검증 경로는 `openrouter` 없이 OpenAI 만 사용했다.

추가 확인 완료 항목은 아래 두 가지다.

- `~/.nanobot/nanobot.env` 의 빈 `OPENAI_API_KEY=` 대입이 shell env 를 지우던 문제는 수정했고, 현재는 `.zshrc` 에 export 된 `OPENAI_API_KEY` 또는 `OPEN_API_KEY` alias 를 유지한 채 source venv 검증이 가능하다
- `openai` provider 직접 호출은 `gpt-5.4-mini-2026-03-17` 기준 `Reply with exactly OK -> OK` 로 성공했고, 이 세션 기준 remote 검증 경로는 `openrouter` 없이 OpenAI 만 사용한다

현재 남은 blocker 는 아래 한 가지다.

- 실제 Telegram/Discord inbound 채널에서 `/model smart-router` 를 직접 입력해 확인하는 end-to-end 검증은 이번 세션에서 수행하지 못함

같은 후속 작업 중 Telegram live command readiness 는 추가로 확인했다.

- `source/nanobot/channels/telegram.py` 의 bot command menu 에 `/usage` 를 추가
- `tests/channels/test_telegram_channel.py` 포함 focused test `69 passed` 재검증 완료
- gateway 재시작 후 Telegram Bot API `getMyCommands` 응답에서 `/usage` 가 실제 등록된 것 확인

즉, Telegram 쪽은 실제 사용자 inbound 로 `/model smart-router` 를 한번 보내 보는 마지막 검증만 남은 상태다.

## 추가 업데이트 - gateway 재시작 오류 정리

같은 날 후속 확인에서 gateway 재시작 직후 로그에 아래 문제가 함께 보였다.

- 기존 gateway 프로세스가 health port `18790` 을 아직 점유한 상태에서 새 인스턴스가 올라오며 `OSError: [Errno 48] address already in use`
- bind 실패 뒤 shutdown 경로에서 MCP cleanup 과 Telegram stop 이 연쇄적으로 돌며 noisy error 가 추가 발생
- Telegram channel 은 초기화가 끝나기 전에 종료되면 `This Updater is not running!` 또는 `This Application is not running!` 류 예외가 섞일 수 있었음

이번에 반영한 정리는 아래와 같다.

- `source/nanobot/cli/commands.py`
  - gateway run loop 를 단순 `asyncio.gather` 에서 first-exception 기반 cancellation 정리 흐름으로 변경
  - health port bind 실패 시 `Gateway crashed unexpectedly` traceback 대신 `Health endpoint port <port> is already in use.` 형태로 명확한 메시지 출력
  - shutdown 순서를 `task cancel -> channel stop -> MCP close` 쪽으로 정리해 bind 실패 뒤 cleanup noise 를 줄임
- `source/nanobot/channels/telegram.py`
  - partially started updater/application 종료 시 `not running` 계열 RuntimeError 를 shutdown noise 로 보지 않고 흡수

focused test 재검증 결과:

- `tests/cli/test_commands.py`
  - `test_gateway_health_endpoint_binds_and_serves_expected_responses`
  - `test_gateway_port_conflict_reports_clean_error_and_stops_runtime`
- `tests/channels/test_telegram_channel.py`
  - `test_start_respects_custom_pool_config`
  - `test_stop_ignores_updater_not_running_runtime_error`

위 세트는 source `.venv` 기준 `4 passed` 로 통과했다.

live 재기동 확인 결과:

- `infra/scripts/stop-nanobot-services.sh` + `start-nanobot-services.sh` 이후 launchd 기준 gateway/API 프로세스가 다시 올라옴
- `status-nanobot-services.sh` 에서 gateway/api 모두 `ok`
- `http://127.0.0.1:18790/health`, `http://127.0.0.1:8900/health` 모두 `{"status":"ok"}` 반환
- 최근 gateway log 에서는 transient port overlap 흔적은 남아 있지만, 최종 상태는 MCP reconnect, Telegram bot 연결, health endpoint 복구까지 정상 확인
4. editable install 운영과 고정 배포 운영 중 표준 경로 결정
5. plugin status 에 health / probe 개념을 넣을지 별도 명령으로 분리할지 결정

## 결론

2026-04-21 기준 Nanobot 는 "smart-router 를 포함한 runtime plugin 구조가 실제로 정리된 상태" 라고 볼 수 있다. 이전에는 smart-router 가 동작하는지와 source-linked runtime 인지 확인하는 단계였다면, 지금은 plugin registry, 설정 구조, CLI 가시성, 테스트, live 검증, 문서까지 한 번 정리된 단계다.

즉, 현재 상태를 한 줄로 정리하면 아래와 같다.

- smart-router 는 Nanobot runtime plugin 구조 위에 안착함
- `plugins.smartrouter` 가 주 설정 경로가 됨
- runtime plugin 확장은 provider/hook 수준을 넘어 tool/init/status seam 까지 열림
- 운영 관점에서 다음 핵심 과제는 remote fallback 실검증과 운영 배포 기준 정리다