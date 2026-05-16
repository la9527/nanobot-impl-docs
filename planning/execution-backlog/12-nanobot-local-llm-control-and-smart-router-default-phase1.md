# 12 Nanobot Local LLM Control And Smart Router Default Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 검증된 로컬 LLM 중 `qwen36` 을 phase-1 기본 local LLM 으로 두고, Nanobot에서 실행, 종료, 재시작, 상태확인, 기본 사용 대상으로 제어하고, WebUI/Telegram 설정 surface 에서 같은 기능을 제공하며, smart-router `local` tier 가 그 로컬 LLM을 사용하게 만든다.

**Architecture:** 기존 `infra/scripts/local-models/` launchd 제어 계층은 유지하고, 부족한 `restart` 와 `use/default` 작업만 더한다. Nanobot runtime 쪽에는 local LLM control service 를 두어 shell script 호출, 상태 payload, env backup/update 를 한 곳에서 처리하고 WebUI HTTP endpoint 와 Telegram slash command 가 같은 service 를 사용하게 한다. live `~/.nanobot/nanobot.env` 는 직접 수정하지 않고, local LLM 전용 override 파일 `~/.nanobot/local-llm.env` 를 만들어 gateway/API start wrapper 가 `nanobot.env` 다음에 source 하도록 한다. WebUI와 `/model` surface 는 기존 `smart-router-local` model target 을 재사용하되, Settings 에서는 local LLM lifecycle 과 기본 local route 변경을 별도 섹션으로 노출한다.

**Tech Stack:** zsh launchd scripts, Nanobot Python config/model target layer, WebUI React/Vite settings UI, Telegram slash commands, Pydantic config schema, pytest, Vitest, live `~/.nanobot/nanobot.env`, `~/.nanobot/config*.json`, OpenAI-compatible local model endpoints.

---

## 1. 문서 목적

이 문서는 기존 `08-local-model-runtime-unification-and-qwen36-phase1.md` 와 별개로, 이미 검증된 로컬 LLM을 Nanobot의 실제 응답 경로에 연결하기 위한 high-priority 후속 slice 다.

`08` 이 `local-models` 공용 lifecycle 계층과 LFM2/Qwen3.6 런타임 정리에 초점을 둔다면, 이 문서는 아래 product 운영 기능에 초점을 둔다.

- 로컬 LLM 실행
- 로컬 LLM 종료
- 로컬 LLM 재시작
- 로컬 LLM 상태확인
- 검증된 로컬 LLM을 Nanobot 기본 local target 으로 사용
- WebUI Settings 에서 local LLM target, status, action, default-use 를 제어
- Telegram 에서 허용된 사용자만 local LLM status, action, default-use 를 제어
- smart-router `local` tier 가 그 로컬 LLM으로 실제 요청을 보내도록 설정

이번 slice 는 컨펌 후 implementation 으로 넘어간다. 계획 단계에서는 source/infra 구현 파일을 수정하지 않고, 적용 방안과 검증 기준만 고정한다.

## 2. 현재 기준 확인

이미 있는 기반:

- `infra/scripts/local-models/local-models.sh`
- `infra/scripts/local-models/start-local-model-services.sh`
- `infra/scripts/local-models/stop-local-model-services.sh`
- `infra/scripts/local-models/status-local-model-services.sh`
- `infra/scripts/local-models/run-local-model-smoke.sh`
- `infra/scripts/local-models/run-local-model-benchmark.sh`
- `infra/launchd/com.nanobot.local-model-lfm2.plist`
- `infra/launchd/com.nanobot.local-model-qwen36.plist`
- `source/nanobot/model_targets.py`
- `source/nanobot/config/schema.py`
- `source/nanobot/channels/websocket.py`
- `source/nanobot/command/builtin.py`
- `source/nanobot/channels/telegram.py`
- `source/webui/src/components/settings/SettingsView.tsx`
- `source/webui/src/lib/api.ts`
- `source/webui/src/lib/types.ts`
- `source/tests/config/test_model_targets.py`
- `source/tests/channels/test_websocket_http_routes.py`
- `source/tests/command/test_builtin_model.py`
- `source/webui/src/tests/api.test.ts`
- `infra/scripts/start-nanobot-gateway.sh`
- `infra/scripts/start-nanobot-api.sh`
- `infra/nanobot/config.template.json`
- `infra/nanobot/config.api.template.json`
- `infra/nanobot/nanobot.env.example`
- `infra/nanobot/local-llm.env.example`

현재 확인된 빈틈:

- `local-models.sh` 에 `restart` action 이 없다.
- `local-models.sh` 에 검증된 모델을 Nanobot 기본 local target 으로 지정하는 `use` 또는 `default` action 이 없다.
- gateway/API start wrapper 는 현재 `~/.nanobot/nanobot.env` 만 source 하므로, local LLM 선택값을 별도 파일에서 override 할 수 없다.
- `infra/nanobot/config.template.json` 의 `smartRouter` 기본 예시는 `enabled: false` 이고, `local` tier 예시가 없다.
- runtime config 는 `smart-router-local` target 을 이미 만들 수 있지만, live env/template/guide 에서 어떤 로컬 모델이 smart-router local tier 로 쓰이는지 한 번에 드러나지 않는다.
- WebUI Settings 는 현재 provider/model/theme/chat readability 중심이고, local LLM lifecycle 설정 섹션이 없다.
- Telegram command menu 는 `/model`, `/restart`, `/status` 를 제공하지만, local model service 자체를 시작/중지/재시작/use 하는 명령은 없다.

## 3. 목표 UX

컨펌 후 구현할 사용자-facing UX 는 아래를 기준으로 한다.

```bash
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh list
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh status all
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh start qwen36
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh stop qwen36
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh restart qwen36
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh smoke qwen36
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh use qwen36
```

WebUI Settings UX 는 아래를 기준으로 한다.

```text
Settings > Local LLM

Current default local LLM
  qwen36
  mlx-community/Qwen3.6-35B-A3B-4bit
  http://127.0.0.1:1246/v1

Targets
  lfm2    stopped / available / Use as default
  qwen36  running / default / Restart / Stop / Smoke

Actions
  Start, Stop, Restart, Smoke, Use as default local LLM
```

Telegram UX 는 아래 command 를 기준으로 한다.

```text
/local-llm status
/local-llm start qwen36
/local-llm stop qwen36
/local-llm restart qwen36
/local-llm smoke qwen36
/local-llm use qwen36
```

Telegram 에서는 파괴적이거나 live env 를 바꾸는 명령에 confirmation step 을 둔다.

```text
/local-llm use qwen36
Local LLM default will change to qwen36 after smoke check.
Reply yes to continue.
```

`use qwen36` 의 기대 동작:

1. `qwen36` launchd service 가 떠 있지 않으면 먼저 start 한다.
2. `qwen36` endpoint 가 `/v1/models` 에 응답할 때까지 기다린 뒤 smoke test 를 수행한다.
3. `~/.nanobot/local-llm.env` 가 있으면 timestamp backup 으로 복사한다.
4. `~/.nanobot/local-llm.env` 에 `NANOBOT_LOCAL_LLM_TARGET=qwen36` 을 기록한다.
5. `~/.nanobot/local-llm.env` 에 `LOCAL_LLM_BASE_URL=http://127.0.0.1:1246/v1` 을 기록한다.
6. `~/.nanobot/local-llm.env` 에 `LOCAL_LLM_MODEL=mlx-community/Qwen3.6-35B-A3B-4bit` 을 기록한다.
7. `~/.nanobot/nanobot.env`, `~/.nanobot/config.json`, `~/.nanobot/config.api.json` 은 자동 덮어쓰기 대상에서 제외한다.
8. gateway/API start wrapper 는 `nanobot.env` 를 source 한 뒤 `local-llm.env` 를 source 하여 local LLM 값만 override 한다.
9. config template 과 guide 는 smart-router local tier 가 `${LOCAL_LLM_MODEL}` 을 보도록 정리한다.
10. Nanobot service 재시작은 사용자가 확인한 뒤 launchd script 로 수행한다.

기본 local target 은 아래 원칙으로 둔다.

- Nanobot 기본 대화 target: `smart-router`
- 강제 local target: `smart-router-local`
- phase-1 기본 local LLM: `qwen36`
- smart-router local tier: provider `vllm`, model `${LOCAL_LLM_MODEL}`
- provider endpoint: `providers.vllm.apiBase = ${LOCAL_LLM_BASE_URL}`

## 4. 비범위

이번 slice 에서는 아래를 하지 않는다.

- 여러 local LLM 을 동시에 자동 상주시켜 auto-balancing 하기
- LLMTestTool_v2 의 route policy 결과를 자동으로 Nanobot live config 에 반영하기
- remote mini/full tier 정책을 새로 튜닝하기
- `~/.nanobot/config*.json` 을 무조건 rewrite 하기
- WebUI에 Settings 와 분리된 독립 local model 관리 앱을 새로 만들기
- Telegram inline keyboard 기반 multi-step wizard 를 처음부터 만들기

## 5. 작업 분해

### Task 1: `local-models.sh` restart action 추가

**Files:**

- Modify: `infra/scripts/local-models/local-models.sh`
- Reference: `infra/scripts/local-models/stop-local-model-services.sh`
- Reference: `infra/scripts/local-models/start-local-model-services.sh`

- [ ] **Step 1: failing shell-level smoke 작성**

Run:

```bash
cd /Volumes/ExtData/Nanobot
zsh -n infra/scripts/local-models/local-models.sh
infra/scripts/local-models/local-models.sh help | grep 'restart <lfm2|qwen36>'
```

Expected before implementation:

```text
grep exits 1 because restart is not documented yet
```

- [ ] **Step 2: `restart` dispatch 추가**

구현 기준:

```zsh
cmd_restart() {
  require_target "${1:-}"
  reject_all_for_action restart "$1"
  _run stop-local-model-services.sh "$1" || true
  _run start-local-model-services.sh "$1"
  _run status-local-model-services.sh "$1"
}
```

`case` 에는 아래 alias 를 둔다.

```zsh
restart|reboot) cmd_restart "$@" ;;
```

- [ ] **Step 3: syntax/help 확인**

Run:

```bash
cd /Volumes/ExtData/Nanobot
zsh -n infra/scripts/local-models/local-models.sh
infra/scripts/local-models/local-models.sh help | grep 'restart <lfm2|qwen36>'
```

Expected:

```text
restart <lfm2|qwen36>      Restart one installed launchd service
```

### Task 2: model metadata를 control script 에서 재사용 가능하게 고정

**Files:**

- Modify: `infra/scripts/local-models/common.sh`
- Test by command: `infra/scripts/local-models/local-models.sh list`

- [ ] **Step 1: model별 endpoint/model id helper 추가**

구현 기준 함수:

```zsh
model_api_base() {
  case "$1" in
    lfm2) echo "http://127.0.0.1:1242/v1" ;;
    qwen36) echo "http://127.0.0.1:1246/v1" ;;
    *) return 1 ;;
  esac
}

model_id() {
  case "$1" in
    lfm2) echo "LiquidAI/LFM2-24B-A2B-GGUF:Q4_0" ;;
    qwen36) echo "mlx-community/Qwen3.6-35B-A3B-4bit" ;;
    *) return 1 ;;
  esac
}
```

- [ ] **Step 2: list/status/smoke 에서 같은 helper 를 쓰도록 정리**

Run:

```bash
cd /Volumes/ExtData/Nanobot
infra/scripts/local-models/local-models.sh list
```

Expected:

```text
lfm2   LiquidAI/LFM2-24B-A2B-GGUF:Q4_0   llama.cpp   port 1242
qwen36 mlx-community/Qwen3.6-35B-A3B-4bit mlx_lm.server port 1246
```

### Task 3: `use <target>` command 로 local LLM override 파일 지정

**Files:**

- Modify: `infra/scripts/local-models/local-models.sh`
- Create: `infra/scripts/local-models/use-local-model.sh`
- Create: `infra/nanobot/local-llm.env.example`
- Modify: `infra/README.md`
- Modify: `docs/guide/09-troubleshooting.md`

- [ ] **Step 1: `use-local-model.sh` 설계대로 생성**

동작 기준:

```bash
infra/scripts/local-models/use-local-model.sh qwen36
```

Expected behavior:

- target 을 `lfm2|qwen36` 로 제한한다.
- target service 가 실행 중이 아니면 `start-local-model-services.sh <target>` 으로 먼저 시작한다.
- `run-local-model-smoke.sh <target>` 을 수행해 endpoint 응답을 확인한다.
- `~/.nanobot/local-llm.env` 가 있으면 `~/.nanobot/local-llm.env.backup-YYYYMMDD-HHMMSS` 로 복사한다.
- `~/.nanobot/local-llm.env` 가 없으면 새로 만든다.
- `NANOBOT_LOCAL_LLM_TARGET=` 라인을 target 으로 갱신한다.
- `LOCAL_LLM_BASE_URL=` 라인을 target endpoint 로 갱신한다.
- `LOCAL_LLM_MODEL=` 라인을 target model id 로 갱신한다.
- 값이 없으면 파일 끝에 추가한다.
- `~/.nanobot/nanobot.env` 는 수정하지 않는다.
- 마지막에 Nanobot 재시작 명령을 안내한다.

출력 예시:

```text
Updated ~/.nanobot/local-llm.env for qwen36
NANOBOT_LOCAL_LLM_TARGET=qwen36
LOCAL_LLM_BASE_URL=http://127.0.0.1:1246/v1
LOCAL_LLM_MODEL=mlx-community/Qwen3.6-35B-A3B-4bit
Restart Nanobot with /Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh && /Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
```

- [ ] **Step 2: `local-models.sh use` dispatch 추가**

구현 기준:

```zsh
cmd_use() {
  require_target "${1:-}"
  reject_all_for_action use "$1"
  _run use-local-model.sh "$1"
}
```

`case` alias:

```zsh
use|default|set-default) cmd_use "$@" ;;
```

- [ ] **Step 3: `local-llm.env.example` 생성**

예시 파일 기준:

```bash
# Nanobot local LLM override loaded after ~/.nanobot/nanobot.env.
NANOBOT_LOCAL_LLM_TARGET=qwen36
LOCAL_LLM_BASE_URL=http://127.0.0.1:1246/v1
LOCAL_LLM_MODEL=mlx-community/Qwen3.6-35B-A3B-4bit
```

- [ ] **Step 4: dry-run 없이 live local LLM override 파일을 만지는 명령임을 help 에 명시**

Help 문구 기준:

```text
use <lfm2|qwen36>          Start and smoke test target, then set ~/.nanobot/local-llm.env as the default local LLM
```

### Task 4: gateway/API wrapper 가 local LLM override 파일을 source 하게 변경

**Files:**

- Modify: `infra/scripts/start-nanobot-gateway.sh`
- Modify: `infra/scripts/start-nanobot-api.sh`
- Modify: `infra/scripts/setup-nanobot.sh`
- Modify: `infra/scripts/validate-nanobot-setup.sh`

- [ ] **Step 1: source order 고정**

Wrapper 는 아래 순서로 env 를 로드한다.

```zsh
ENV_FILE="$NANOBOT_HOME/nanobot.env"
LOCAL_LLM_ENV_FILE="$NANOBOT_HOME/local-llm.env"

source_env_file "$ENV_FILE"
source_env_file "$LOCAL_LLM_ENV_FILE"
```

`local-llm.env` 는 `nanobot.env` 다음에 source 되어 `LOCAL_LLM_BASE_URL`, `LOCAL_LLM_MODEL`, `NANOBOT_LOCAL_LLM_TARGET` 만 override 한다.

- [ ] **Step 2: setup script 는 local-llm.env.example 을 복사**

`setup-nanobot.sh` 는 기존 `nanobot.env` 를 건드리지 않고, 없을 때만 아래 파일을 만든다.

```text
~/.nanobot/local-llm.env
```

- [ ] **Step 3: validation 에 local override 상태 표시**

`validate-nanobot-setup.sh` 는 local override 파일이 있으면 현재 target/model/base 를 보여준다. 파일이 없어도 실패로 보지 않고, `qwen36` 기본 예시를 안내한다.

### Task 5: smart-router local tier template 정리

**Files:**

- Modify: `infra/nanobot/config.template.json`
- Modify: `infra/nanobot/config.api.template.json`
- Modify: `infra/nanobot/nanobot.env.example`
- Test: `source/tests/config/test_smart_router_config.py`
- Test: `source/tests/config/test_model_targets.py`

- [ ] **Step 1: config template 에 local tier 명시**

`smartRouter` 또는 `plugins.smartrouter` 예시는 아래 의미를 가져야 한다.

```json
{
  "enabled": true,
  "allowLocalTools": false,
  "local": {
    "provider": "vllm",
    "model": "${LOCAL_LLM_MODEL}"
  },
  "mini": {
    "provider": "",
    "model": ""
  },
  "full": {
    "provider": "",
    "model": ""
  }
}
```

기준은 `smart-router-local` 이 항상 `${LOCAL_LLM_MODEL}` 을 설명에 표시할 수 있는 상태다.

- [ ] **Step 2: model target 테스트 보강**

추가할 assertion:

```python
assert targets["smart-router-local"].kind == "smart_router"
assert targets["smart-router-local"].smart_router_mode == "local"
assert "LiquidAI/LFM2" in targets["smart-router-local"].description
```

Run:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/config/test_model_targets.py tests/config/test_smart_router_config.py -q
```

Expected:

```text
passed
```

### Task 6: WebUI/bootstrap model target surface 재검증

**Files:**

- Test: `source/tests/channels/test_websocket_http_routes.py`
- Reference: `source/nanobot/channels/websocket.py`
- Reference: `source/nanobot/model_targets.py`

- [ ] **Step 1: bootstrap 이 env-backed smart-router-local 설명을 노출하는지 확인**

이미 있는 test anchor:

```python
assert rows["smart-router-local"]["description"] == (
    "smart-router forced local tier (LiquidAI/LFM2-24B-A2B-GGUF:Q4_0)"
)
```

Run:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/channels/test_websocket_http_routes.py -q
```

Expected:

```text
passed
```

- [ ] **Step 2: 필요 시 `/model smart-router-local` round trip 보강**

현재 `source/tests/command/test_builtin_model.py` 에 smart-router target coverage 가 있으므로, 부족하면 아래를 추가한다.

```python
assert "smart-router-local" in rendered_model_list
```

### Task 7: 공통 local LLM control service 와 HTTP API 추가

**Files:**

- Create: `source/nanobot/local_llm_control.py`
- Modify: `source/nanobot/channels/websocket.py`
- Test: `source/tests/channels/test_websocket_http_routes.py`

- [ ] **Step 1: backend payload contract 고정**

WebUI 와 Telegram 이 공유할 response shape 는 아래로 둔다.

```json
{
  "default_target": "qwen36",
  "default_model": "mlx-community/Qwen3.6-35B-A3B-4bit",
  "default_api_base": "http://127.0.0.1:1246/v1",
  "targets": [
    {
      "name": "lfm2",
      "label": "LFM2",
      "runtime": "llama.cpp",
      "model": "LiquidAI/LFM2-24B-A2B-GGUF:Q4_0",
      "api_base": "http://127.0.0.1:1242/v1",
      "launchd_label": "com.nanobot.local-model-lfm2",
      "running": false,
      "endpoint_ok": false,
      "is_default": false
    },
    {
      "name": "qwen36",
      "label": "Qwen3.6",
      "runtime": "mlx_lm.server",
      "model": "mlx-community/Qwen3.6-35B-A3B-4bit",
      "api_base": "http://127.0.0.1:1246/v1",
      "launchd_label": "com.nanobot.local-model-qwen36",
      "running": true,
      "endpoint_ok": true,
      "is_default": true
    }
  ]
}
```

- [ ] **Step 2: HTTP endpoints 추가**

Endpoint 기준:

```text
GET  /api/local-llm/status
GET  /api/local-llm/start/<target>
GET  /api/local-llm/stop/<target>
GET  /api/local-llm/restart/<target>
GET  /api/local-llm/smoke/<target>
GET  /api/local-llm/use/<target>
```

현재 WebSocket channel 의 embedded HTTP surface 는 existing settings/session action route 와 마찬가지로 GET 기반 action endpoint 를 사용한다. 외부 API 서버로 분리할 때만 POST variant 를 추가 검토한다.

`use` 응답은 env backup 정보를 포함한다.

```json
{
  "ok": true,
  "action": "use",
  "target": "qwen36",
  "backup_path": "~/.nanobot/local-llm.env.backup-20260516-210000",
  "requires_restart": true,
  "message": "qwen36 is now the default local LLM. Restart Nanobot to apply."
}
```

- [ ] **Step 3: command execution boundary 고정**

`local_llm_control.py` 는 shell command 문자열 조합을 외부 입력으로 받지 않는다. 허용된 action/target enum 만 받아 아래 script 로 매핑한다.

```python
ALLOWED_ACTIONS = {"status", "start", "stop", "restart", "smoke", "use"}
ALLOWED_TARGETS = {"lfm2", "qwen36", "all"}
SCRIPT = "/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh"
```

`start|restart|smoke|use` 는 `all` target 을 거부한다.

- [ ] **Step 4: backend focused test**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/channels/test_websocket_http_routes.py -q
```

Expected:

```text
passed
```

### Task 8: WebUI Settings Local LLM section 추가

**Files:**

- Modify: `source/webui/src/lib/types.ts`
- Modify: `source/webui/src/lib/api.ts`
- Modify: `source/webui/src/components/settings/SettingsView.tsx`
- Modify: `source/webui/src/i18n/locales/*/common.json`
- Test: `source/webui/src/tests/api.test.ts`
- Test: add or modify `source/webui/src/tests/settings-view.test.tsx` if present; otherwise add focused coverage near existing settings tests

- [ ] **Step 1: WebUI API client 추가**

Type 기준:

```ts
export interface LocalLlmTargetStatus {
  name: "lfm2" | "qwen36";
  label: string;
  runtime: string;
  model: string;
  api_base: string;
  launchd_label: string;
  running: boolean;
  endpoint_ok: boolean;
  is_default: boolean;
}

export interface LocalLlmStatusPayload {
  default_target: string | null;
  default_model: string | null;
  default_api_base: string | null;
  targets: LocalLlmTargetStatus[];
}

export interface LocalLlmActionResponse {
  ok: boolean;
  action: "start" | "stop" | "restart" | "smoke" | "use";
  target: string;
  message: string;
  backup_path?: string | null;
  requires_restart?: boolean;
}
```

API function 기준:

```ts
export async function fetchLocalLlmStatus(token: string, base = "") {
  return request<LocalLlmStatusPayload>(`${base}/api/local-llm/status`, token);
}

export async function runLocalLlmAction(token: string, action: string, target: string, base = "") {
  return request<LocalLlmActionResponse>(
    `${base}/api/local-llm/${encodeURIComponent(action)}/${encodeURIComponent(target)}`,
    token,
    { method: "POST" },
  );
}
```

- [ ] **Step 2: Settings UI 구성**

`SettingsView` 에 `Local LLM` section 을 추가한다.

UX 기준:

```text
Local LLM
Default local route: qwen36
Smart-router local tier uses LOCAL_LLM_MODEL after restart.

qwen36  Running  Endpoint OK  Default
[Restart] [Stop] [Smoke]

lfm2    Stopped  Endpoint unavailable
[Start] [Smoke] [Use as default]
```

상태 규칙:

- `running && endpoint_ok`: `Running` + positive tone
- `running && !endpoint_ok`: `Running, endpoint unavailable` + warning tone
- `!running`: `Stopped` + muted tone
- `is_default`: `Default` badge 표시
- `use` 성공 후 `requires_restart` 가 true 이면 기존 restart CTA 를 재사용한다.

- [ ] **Step 3: destructive action confirmation**

WebUI 에서는 아래 action 에 확인 prompt 를 둔다.

```text
stop: Stop qwen36 local model?
restart: Restart qwen36 local model?
use: Set qwen36 as the default local LLM?
```

`smoke` 와 `status refresh` 는 확인 없이 실행한다.

- [ ] **Step 4: WebUI focused tests**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/api.test.ts
```

Expected:

```text
pass
```

Settings component test 가 추가되면 아래도 실행한다.

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/settings-view.test.tsx
```

### Task 9: Telegram `/local-llm` command 추가

**Files:**

- Modify: `source/nanobot/command/builtin.py`
- Modify: `source/nanobot/channels/telegram.py`
- Test: `source/tests/command/test_builtin_model.py` 또는 새 `source/tests/command/test_builtin_local_llm.py`

- [ ] **Step 1: command spec 추가**

Command palette 기준:

```python
BuiltinCommandSpec(
    "/local-llm",
    "Manage local LLM",
    "Show or control local LLM runtime and default local route.",
    "bot",
    "[status|start|stop|restart|smoke|use] [lfm2|qwen36]",
)
```

Telegram command menu 기준:

```python
BotCommand("local_llm", "Manage local LLM runtime")
```

Telegram-safe alias 는 `/local_llm` 을 `/local-llm` 으로 normalize 한다.

- [ ] **Step 2: `/local-llm status` 출력**

출력 기준:

```text
## Local LLM
Default: qwen36 -> mlx-community/Qwen3.6-35B-A3B-4bit

* qwen36 — running, endpoint ok, default
- lfm2 — stopped

Use `/local-llm use qwen36` to change the default local LLM.
Use `/model smart-router-local` to force the local tier for this chat.
```

- [ ] **Step 3: action command 출력**

`/local-llm use qwen36` 성공 출력 기준:

```text
## Local LLM
qwen36 is now the default local LLM.
Restart Nanobot to apply this to new runtime config.
Use `/restart` when ready.
```

`stop|restart|use` 는 Telegram 에서 바로 실행하지 않고 confirmation 을 요구한다. phase-1 에서는 가장 단순하게 `yes` 답변을 같은 session pending action 으로 받는 방식을 쓴다. confirmation state 를 새로 크게 만들기 어렵다면 phase-1 fallback 으로 아래처럼 안내하고 WebUI/CLI 실행을 요구한다.

```text
This action changes local runtime state. Run `/local-llm confirm use qwen36` to continue.
```

- [ ] **Step 4: Telegram handler regex 확장**

현재 Telegram handler 는 `/new|stop|restart|model|status|usage|dream` 중심이다. 아래 command 를 forward 대상에 추가한다.

```text
local-llm|local_llm
```

- [ ] **Step 5: command focused tests**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/command/test_builtin_local_llm.py -q
```

Expected:

```text
passed
```

### Task 10: live 적용 검증 절차 고정

**Files:**

- Modify: `docs/guide/09-troubleshooting.md`
- Modify: `docs/operations/status-summary/nanobot-status-summary-2026-05-16.md` after implementation

- [ ] **Step 1: 로컬 모델 상태 확인**

Run:

```bash
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh status all
```

Expected:

```text
lfm2 또는 qwen36 중 선택 대상 endpoint 가 /v1/models 응답 가능
```

- [ ] **Step 2: 기본 local LLM 지정**

Run:

```bash
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh use qwen36
```

Expected:

```text
~/.nanobot/local-llm.env backup 생성 또는 신규 생성
NANOBOT_LOCAL_LLM_TARGET=qwen36
LOCAL_LLM_BASE_URL=http://127.0.0.1:1246/v1
LOCAL_LLM_MODEL=mlx-community/Qwen3.6-35B-A3B-4bit
```

- [ ] **Step 3: Nanobot launchd 재시작**

Run:

```bash
/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
```

Expected:

```text
com.nanobot.gateway 와 com.nanobot.api 가 다시 올라옴
```

- [ ] **Step 4: gateway health 확인**

Run:

```bash
curl -fsS http://127.0.0.1:18790/health
```

Expected:

```json
{"ok":true}
```

- [ ] **Step 5: smart-router local 강제 응답 확인**

권장 경로:

```text
WebUI model target 에서 Local 선택 또는 /model smart-router-local 실행 후 짧은 질문 전송
```

Expected:

```text
response metadata 또는 footer 에 smart-router local target/route 가 표시되고, 실제 model 이 선택한 LOCAL_LLM_MODEL 로 기록됨
```

## 6. 구현 순서 권장안

1. `local-models.sh restart` 부터 추가한다.
2. model metadata helper 를 `common.sh` 로 정리한다.
3. `use/default` command 를 추가하되, target start, smoke 선행, `~/.nanobot/local-llm.env` backup/update 를 반드시 포함한다.
4. gateway/API wrapper 가 `nanobot.env` 다음에 `local-llm.env` 를 source 하도록 한다.
5. smart-router local tier template 을 `${LOCAL_LLM_MODEL}` 기준으로 맞춘다.
6. local LLM control service 와 `/api/local-llm/*` endpoint 를 추가한다.
7. WebUI Settings 에 `Local LLM` section 을 붙인다.
8. Telegram `/local-llm` command 와 `/local_llm` alias 를 붙인다.
9. model target/bootstrap/API/WebUI/command tests 를 좁게 통과시킨다.
10. launchd live restart 와 WebUI 또는 Telegram 에서 `smart-router-local` 강제 route 를 확인한다.

## 7. 완료 기준

- `local-models.sh start|stop|restart|status|smoke|benchmark|use <target>` UX가 문서와 일치한다.
- `qwen36` 이 phase-1 기본 local LLM 으로 문서, template, WebUI/Telegram status 에 일관되게 표시된다.
- `use <target>` 은 target start 와 smoke 성공 후에만 `~/.nanobot/local-llm.env` 를 백업/갱신하고 local LLM 기본값을 바꾼다.
- gateway/API wrapper 는 `~/.nanobot/nanobot.env` 를 먼저 source 하고 `~/.nanobot/local-llm.env` 를 나중에 source 하여 local LLM 값만 override 한다.
- WebUI Settings 에서 local LLM target 목록, running/endpoint/default 상태, Start/Stop/Restart/Smoke/Use as default action 을 제공한다.
- Telegram 에서 `/local-llm status|start|stop|restart|smoke|use <target>` 또는 `/local_llm ...` alias 로 같은 기능을 제공한다.
- WebUI/Telegram 은 같은 backend local LLM control service 와 같은 response contract 를 사용한다.
- smart-router `local` tier 는 provider `vllm`, model `${LOCAL_LLM_MODEL}` 기준으로 설명과 실행이 맞는다.
- `smart-router-local` target 이 WebUI/bootstrap 과 `/model` 목록에 명확히 노출된다.
- Nanobot launchd 재시작 후 `GET /health` 와 실제 local route 응답이 확인된다.
- 변경된 운영 기준이 `infra/README.md`, `docs/guide/09-troubleshooting.md`, status summary 에 남는다.

## 8. 확정 사항과 남은 컨펌

2026-05-16 사용자 컨펌으로 아래는 확정한다.

1. phase-1 기본 local LLM 은 `qwen36` 으로 둔다.
2. `use <target>` 은 `~/.nanobot/nanobot.env` 를 직접 수정하지 않는다.
3. local LLM 선택값은 별도 파일 `~/.nanobot/local-llm.env` 에 기록하고, gateway/API wrapper 가 `nanobot.env` 다음에 source 해서 적용한다.
4. `use qwen36` 은 qwen36 service 를 먼저 시작하고 endpoint/smoke 성공 후 기본값을 갱신한다.
5. WebUI Settings 와 Telegram `/local-llm` command 를 모두 phase-1 범위에 포함한다.

남은 컨펌은 구현 전 blocking 이 아니라, 구현 중 선택지가 생길 때만 확인한다.

- Telegram confirmation UX 를 `/local-llm confirm ...` text command 로 먼저 둘지, inline keyboard 를 phase-1 에도 포함할지
- WebUI action confirmation 을 browser confirm 으로 시작할지, 기존 design system modal 이 있으면 그 pattern 을 따를지
