# 14 WebUI Smart Router Model Picker Alias Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Nanobot WebUI 채팅창 모델 선택기와 badge 가 live local runtime 상태와 어긋나지 않게 맞추고, smart-router 사용 시 채팅창에서는 `Auto`, `Local`, `Mini`, `Full` 네 가지 alias 만 선택되도록 정리한다.

**Architecture:** 현재 WebUI picker 는 `local-models.sh status` 결과를 직접 읽지 않고 `build_model_targets(config)` 가 만든 정적 model target catalog 를 사용한다. 이 계획은 backend model target 해석 단계에서 `~/.nanobot/local-llm.env` 기반 live local runtime 메타를 반영해 `local-llm` 과 `smart-router-local` 이 현재 default local runtime 을 가리키게 만들고, WebUI 채팅창 picker/badge 는 `display_name` 중심 alias UX 로 축소한다. 상세 concrete model 명은 Settings, `/local-llm`, status payload 같은 운영 surface 에 남기고, 채팅창 선택 UI 는 smart-router 중심으로 단순화한다.

**Tech Stack:** Nanobot Python runtime model target layer, websocket/bootstrap HTTP surface, WebUI React/Vite thread composer and badge rendering, local LLM env/status helpers, pytest, Vitest, launchd-backed `local-models.sh`.

---

## 1. 문제 정의와 root cause

현재 확인된 사실은 아래와 같다.

- `infra/scripts/local-models/local-models.sh status` 기준 실제 실행 local model 은 `com.nanobot.local-model-lfm2` 일 수 있다.
- 하지만 WebUI 채팅창 모델 선택기에서는 `smart-router-local` 과 `local-llm` 이 여전히 `mlx-community/Qwen3.6-35B-A3B-4bit` 같은 정적 config model 로 보일 수 있다.
- `source/nanobot/model_targets.py` 의 `_smart_router_variants()` 는 `config.plugins.smartrouter.local.model` 을 그대로 `smart-router-local` metadata 로 쓴다.
- `source/nanobot/model_targets.py` 의 configured target path 는 `agents.defaults.modelSelection.targets.local-llm` 의 provider/model 을 그대로 쓰므로 live `~/.nanobot/local-llm.env` override 와 자동 동기화되지 않는다.
- `source/nanobot/channels/websocket.py` 는 `_load_model_target_context()` 에서 `build_model_targets(config)` 결과를 그대로 WebUI bootstrap/session-model-target payload 로 넘긴다.
- `source/webui/src/components/thread/ThreadComposer.tsx` 는 picker 에서 `display_name` 이 있으면 그걸 쓰지만, subtitle 은 여전히 `provider -> model` 또는 raw model string 을 그대로 보여준다.
- `source/webui/src/components/thread/ThreadShell.tsx` 의 `toModelBadgeLabel()` 는 active target 이 smart-router 계열이어도 concrete `modelName` leaf 를 우선 노출하므로 alias 보다 raw model 이름이 앞에 나온다.

즉 root cause 는 아래 두 가지다.

1. backend model target catalog 가 live local runtime override 를 반영하지 않는다.
2. chat picker 와 badge UX 가 smart-router alias surface 가 아니라 concrete provider/model surface 에 가깝다.

## 2. 목표 UX 고정

이번 slice 에서 고정할 목표 UX 는 아래와 같다.

### 2.1 채팅창 모델 선택기

smart-router 가 활성이고 smart-router target group 이 존재하면 채팅창 picker 에는 아래 네 항목만 노출한다.

```text
Auto
Local
Mini
Full
```

선택 값은 내부적으로 아래 target name 을 유지한다.

```text
Auto  -> smart-router
Local -> smart-router-local
Mini  -> smart-router-mini
Full  -> smart-router-full
```

`default`, `local-llm`, 기타 direct provider target 은 chat picker 기본 목록에서 숨긴다.

### 2.2 채팅창 badge / current model label

채팅창 상단이나 composer 근처 badge 는 concrete model 이름 대신 alias 를 우선 노출한다.

```text
smart-router        -> Auto
smart-router-local  -> Local
smart-router-mini   -> Mini
smart-router-full   -> Full
local-llm           -> Local LLM
default             -> Default
```

concrete model 이름은 아래 surface 에 남긴다.

- Settings > Local LLM
- `/local-llm`
- `/model`
- status/context detail view

### 2.3 live local runtime 동기화

`smart-router-local` 과 `local-llm` 의 concrete provider/model metadata 는 current live local runtime 을 따라야 한다.

예시:

```text
~/.nanobot/local-llm.env => NANOBOT_LOCAL_LLM_TARGET=lfm2
```

이면 payload 는 아래처럼 해석된다.

```text
local-llm.model = LiquidAI/LFM2-24B-A2B-GGUF:Q4_0
smart-router-local.model = LiquidAI/LFM2-24B-A2B-GGUF:Q4_0
```

`qwen36` 이 live default local runtime 이 아니면 WebUI chat picker 나 badge 에 `Qwen3.6` 이 current local choice 인 것처럼 보이면 안 된다.

## 3. 비범위

이번 slice 에서는 아래를 하지 않는다.

- smart-router policy 자체의 route scoring 변경
- `mini/full` remote provider 변경
- local-models infra 를 smart-router plugin 과 완전히 합치기
- Settings 화면에서 direct provider target 전부 제거하기
- `/model` built-in command 의 모든 direct target 을 없애기
- Telegram model picker UX 재설계

## 4. 수정 대상 파일 구조

### Backend target resolution

- Modify: `source/nanobot/model_targets.py`
- Modify: `source/nanobot/local_llm_control.py`
- Modify: `source/nanobot/channels/websocket.py`
- Modify: `source/nanobot/command/builtin.py`
- Test: `source/tests/config/test_model_targets.py`
- Test: `source/tests/channels/test_websocket_http_routes.py`
- Test: `source/tests/command/test_builtin_model.py` 또는 local-llm 관련 builtin test file

### WebUI picker and badge

- Modify: `source/webui/src/components/thread/ThreadComposer.tsx`
- Modify: `source/webui/src/components/thread/ThreadShell.tsx`
- Modify: `source/webui/src/components/ChatList.tsx`
- Modify: `source/webui/src/lib/types.ts` 필요 시 picker visibility field 추가
- Test: `source/webui/src/tests/thread-composer.test.tsx`
- Test: `source/webui/src/tests/thread-shell.test.tsx`
- Test: `source/webui/src/tests/app-layout.test.tsx` 필요 시 bootstrap expectations 갱신

### Docs

- Modify: `docs/operations/status-summary/nanobot-status-summary-2026-05-16.md` 필요 시 후속 반영 메모
- Modify: `source/docs/MODEL_TARGETS.md`

## 5. 설계 원칙

### 원칙 1: chat picker 는 alias-first

chat picker 는 operator 가 concrete model ID 를 읽는 곳이 아니라 route intent 를 고르는 곳으로 본다.

따라서 picker primary label 은 아래를 사용한다.

```text
display_name > translated alias fallback > target.name
```

subtitle 은 smart-router group 에 대해서는 concrete `provider -> model` 을 기본 노출하지 않는다.

권장 표시:

```text
Auto   | smart-router automatic tier selection
Local  | smart-router forced local tier
Mini   | smart-router forced mini tier
Full   | smart-router forced full tier
```

### 원칙 2: live local runtime 은 `local-llm.env` 가 source of truth

chat picker 와 session target payload 에서 local tier concrete model 을 계산할 때 우선 순위는 아래다.

1. `~/.nanobot/local-llm.env` 의 `NANOBOT_LOCAL_LLM_TARGET`
2. `~/.nanobot/local-llm.env` 의 `LOCAL_LLM_MODEL` / `LOCAL_LLM_BASE_URL`
3. `LocalLlmController.status()` 의 default target
4. fallback: `DEFAULT_LOCAL_LLM_TARGET`

즉 smart-router local tier metadata 는 static `config.plugins.smartrouter.local.model` 을 무조건 신뢰하지 않는다.

### 원칙 3: smart-router 사용 시 chat picker 는 smart-router targets 만

`modelTargets` 중 `group == "smart-router"` 항목이 하나라도 있으면 chat picker 는 그 group 만 보여준다.

예상 구현 개념:

```ts
function visibleModelTargets(targets: ModelTargetOption[]): ModelTargetOption[] {
  const smartRouterTargets = targets.filter((target) => target.group === "smart-router");
  if (smartRouterTargets.length > 0) {
    return orderModelTargets(smartRouterTargets);
  }
  return orderModelTargets(targets.filter((target) => target.name !== "default"));
}
```

## 6. 작업 분해

### Task 1: live local runtime aware target metadata 고정

**Files:**

- Modify: `source/nanobot/local_llm_control.py`
- Modify: `source/nanobot/model_targets.py`
- Test: `source/tests/config/test_model_targets.py`

- [ ] **Step 1: failing test 작성**

테스트 의도:

```python
def test_build_model_targets_uses_live_local_llm_for_local_targets(tmp_path, monkeypatch):
    env_path = tmp_path / ".nanobot" / "local-llm.env"
    env_path.parent.mkdir(parents=True)
    env_path.write_text(
        "NANOBOT_LOCAL_LLM_TARGET=lfm2\n"
        "LOCAL_LLM_MODEL=LiquidAI/LFM2-24B-A2B-GGUF:Q4_0\n"
        "LOCAL_LLM_BASE_URL=http://127.0.0.1:1242/v1\n",
        encoding="utf-8",
    )

    monkeypatch.setattr("nanobot.local_llm_control.Path.home", lambda: tmp_path)
    targets = build_model_targets(load_test_config())

    assert targets["local-llm"].model == "LiquidAI/LFM2-24B-A2B-GGUF:Q4_0"
    assert targets["smart-router-local"].model == "LiquidAI/LFM2-24B-A2B-GGUF:Q4_0"
    assert targets["smart-router-local"].display_name == "Local"
```

- [ ] **Step 2: local runtime metadata helper 추가**

구현 방향:

```python
def resolve_live_local_llm_target() -> dict[str, str] | None:
    status = default_controller().status()
    default_target = status["default_target"]
    target_row = next((row for row in status["targets"] if row["name"] == default_target), None)
    if not target_row:
        return None
    return {
        "name": target_row["name"],
        "label": target_row["label"],
        "model": target_row["model"],
        "api_base": target_row["api_base"],
        "runtime": target_row["runtime"],
        "provider": "vllm" if target_row["runtime"] == "mlx_lm.server" else "llama.cpp",
    }
```

`LocalLlmController` 에는 최소한 아래 둘을 추가 검토한다.

```python
ALLOWED_TARGETS = {"lfm2", "qwen36", "qwen35-base-mlx-4bit", "all"}
LOCAL_LLM_TARGETS["qwen35-base-mlx-4bit"] = {...}
```

- [ ] **Step 3: `build_model_targets()` 에 live local override 주입**

구현 방향:

```python
live_local = resolve_live_local_llm_target()
if live_local and "local-llm" in targets:
    targets["local-llm"].model = live_local["model"]
    targets["local-llm"].display_name = "Local LLM"
    targets["local-llm"].description = f"현재 기본 local runtime ({live_local['label']})"

if live_local and "smart-router-local" in targets:
    targets["smart-router-local"].provider = live_local["provider"]
    targets["smart-router-local"].model = live_local["model"]
    targets["smart-router-local"].display_name = "Local"
    targets["smart-router-local"].description = f"smart-router forced local tier ({live_local['label']})"
```

- [ ] **Step 4: focused test 실행**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/config/test_model_targets.py -q
```

Expected:

```text
PASS
```

### Task 2: websocket/bootstrap payload 를 alias-friendly 하게 고정

**Files:**

- Modify: `source/nanobot/channels/websocket.py`
- Test: `source/tests/channels/test_websocket_http_routes.py`

- [ ] **Step 1: failing HTTP route test 작성**

테스트 의도:

```python
def test_webui_bootstrap_exposes_smart_router_aliases_with_live_local_model(...):
    body = webui_bootstrap_json(...)
    names = [target["name"] for target in body["model_targets"]]
    assert "smart-router-local" in names
    local = next(target for target in body["model_targets"] if target["name"] == "smart-router-local")
    assert local["display_name"] == "Local"
    assert local["model"] == "LiquidAI/LFM2-24B-A2B-GGUF:Q4_0"
```

- [ ] **Step 2: session-model-target payload 와 bootstrap payload 모두 같은 serializer 사용**

구현 방향:

```python
def _serialize_model_target(target: ResolvedModelTarget | None) -> dict[str, Any] | None:
    ...
```

중복 serializer 를 하나로 모아 `display_name`, `group`, `smart_router_mode` 가 항상 일관되게 나가게 한다.

- [ ] **Step 3: focused route test 실행**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/channels/test_websocket_http_routes.py -q -k model_target
```

Expected:

```text
PASS
```

### Task 3: chat picker 를 smart-router 4-option alias surface 로 축소

**Files:**

- Modify: `source/webui/src/components/thread/ThreadComposer.tsx`
- Modify: `source/webui/src/lib/types.ts` 필요 시 `hidden_in_picker` 같은 field 추가
- Test: `source/webui/src/tests/thread-composer.test.tsx`

- [ ] **Step 1: failing component test 작성**

테스트 의도:

```tsx
it("shows only Auto Local Mini Full when smart-router targets exist", async () => {
  render(
    <ThreadComposer
      ...
      activeTarget="smart-router"
      modelTargets={[
        { name: "default", kind: "provider_model", model: "openai/gpt-5.4" },
        { name: "local-llm", kind: "provider_model", model: "LiquidAI/LFM2-24B-A2B-GGUF:Q4_0", display_name: "Local LLM" },
        { name: "smart-router", kind: "smart_router", display_name: "Auto", group: "smart-router", smart_router_mode: "auto" },
        { name: "smart-router-local", kind: "smart_router", display_name: "Local", group: "smart-router", smart_router_mode: "local" },
        { name: "smart-router-mini", kind: "smart_router", display_name: "Mini", group: "smart-router", smart_router_mode: "mini" },
        { name: "smart-router-full", kind: "smart_router", display_name: "Full", group: "smart-router", smart_router_mode: "full" },
      ]}
    />,
  );

  expect(screen.queryByText("Local LLM")).not.toBeInTheDocument();
  expect(screen.queryByText("openai/gpt-5.4")).not.toBeInTheDocument();
  expect(screen.getByText(/^Auto$/i)).toBeInTheDocument();
  expect(screen.getByText(/^Local$/i)).toBeInTheDocument();
  expect(screen.getByText(/^Mini$/i)).toBeInTheDocument();
  expect(screen.getByText(/^Full$/i)).toBeInTheDocument();
});
```

- [ ] **Step 2: visible target filtering 구현**

구현 방향:

```ts
function visibleModelTargets(targets: ModelTargetOption[]): ModelTargetOption[] {
  const smartRouterTargets = targets.filter((target) => target.group === "smart-router");
  if (smartRouterTargets.length > 0) {
    return orderModelTargets(smartRouterTargets);
  }
  return orderModelTargets(targets.filter((target) => target.name !== "default"));
}
```

smart-router group subtitle 은 raw `provider -> model` 을 숨기고 description 만 보여준다.

- [ ] **Step 3: focused WebUI test 실행**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-composer.test.tsx
```

Expected:

```text
PASS
```

### Task 4: badge 와 current target label 을 alias-first 로 고정

**Files:**

- Modify: `source/webui/src/components/thread/ThreadShell.tsx`
- Modify: `source/webui/src/components/ChatList.tsx`
- Test: `source/webui/src/tests/thread-shell.test.tsx`
- Test: `source/webui/src/tests/app-layout.test.tsx` 필요 시 갱신

- [ ] **Step 1: failing badge test 작성**

테스트 의도:

```tsx
it("prefers Local alias over concrete model name for smart-router-local", async () => {
  renderThreadShell({
    active_target: "smart-router-local",
    model_name: "mlx-community/Qwen3.6-35B-A3B-4bit",
    model_targets: [
      {
        name: "smart-router-local",
        kind: "smart_router",
        display_name: "Local",
        group: "smart-router",
        smart_router_mode: "local",
        model: "LiquidAI/LFM2-24B-A2B-GGUF:Q4_0",
      },
    ],
  });

  expect(screen.getByText("Local")).toBeInTheDocument();
  expect(screen.queryByText(/Qwen3\.6/)).not.toBeInTheDocument();
});
```

- [ ] **Step 2: target metadata lookup 기반 badge label 구현**

구현 방향:

```ts
function resolveTargetDisplayName(activeTarget: string | null, modelTargets: ModelTargetOption[]): string | null {
  const match = modelTargets.find((target) => target.name === activeTarget);
  const display = match?.display_name?.trim();
  if (display) return display;
  return null;
}
```

`toModelBadgeLabel()` 는 `modelName` raw leaf 를 보기 전에 위 alias lookup 을 먼저 수행한다.

- [ ] **Step 3: focused tests 실행**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-shell.test.tsx src/tests/app-layout.test.tsx
```

Expected:

```text
PASS
```

### Task 5: `/local-llm` 과 `/model` 운영 문구를 live state 기준으로 정리

**Files:**

- Modify: `source/nanobot/command/builtin.py`
- Modify: `source/docs/MODEL_TARGETS.md`
- Test: `source/tests/command/test_builtin_model.py` 또는 관련 builtin/local-llm tests

- [ ] **Step 1: failing builtin text test 작성**

테스트 의도:

```python
def test_local_llm_status_message_mentions_current_default_target(monkeypatch):
    monkeypatch.setattr(default_controller(), "status", lambda: {
        "default_target": "lfm2",
        "default_model": "LiquidAI/LFM2-24B-A2B-GGUF:Q4_0",
        ...
    })
    message = run_local_llm_status_command(...)
    assert "Default: `lfm2`" in message.content
    assert "Use `/local-llm use lfm2`" in message.content
```

- [ ] **Step 2: hard-coded qwen36 운영 문구 제거**

현재 고정 문자열:

```python
"Use `/local-llm use qwen36` to change the default local LLM."
```

이를 live status 기반 포맷으로 바꾼다.

- [ ] **Step 3: focused pytest 실행**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/command -q -k 'local_llm or model'
```

Expected:

```text
PASS
```

### Task 6: production validation

**Files:**

- Validate only, no new files unless bugs require it

- [ ] **Step 1: backend focused tests**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/pytest tests/config/test_model_targets.py tests/channels/test_websocket_http_routes.py -q
```

Expected:

```text
PASS
```

- [ ] **Step 2: WebUI focused tests + build**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-composer.test.tsx src/tests/thread-shell.test.tsx src/tests/app-layout.test.tsx
npm run build
```

Expected:

```text
PASS
vite build success
```

- [ ] **Step 3: live route 확인**

Run:

```bash
cd /Volumes/ExtData/Nanobot/source
PYTHONPATH=$PWD ./.venv/bin/python - <<'PY'
from nanobot.local_llm_control import default_controller
print(default_controller().status())
PY
curl -fsS http://127.0.0.1:18790/health
```

Expected:

```text
status payload shows lfm2 when local-llm.env points to lfm2
{"status": "ok"}
```

- [ ] **Step 4: authenticated bootstrap/manual UI 확인**

검증 기준:

1. WebUI 새로고침 후 채팅창 picker 에 `Auto`, `Local`, `Mini`, `Full` 만 보인다.
2. `Local` subtitle 에 raw `mlx-community/...` 가 primary label 로 나오지 않는다.
3. `smart-router-local` 선택 시 badge 는 `Local` 로 보인다.
4. `lfm2` 가 current default local runtime 이면 `Settings > Local LLM` 에서는 `lfm2` concrete model 이 보이고, chat picker 는 여전히 `Local` alias 만 보인다.

## 7. 기대 결과 요약

이 계획이 끝나면 아래가 성립해야 한다.

- `local-models.sh status` 와 WebUI local tier concrete metadata 가 서로 어긋나지 않는다.
- 채팅창 model picker 는 smart-router 사용 시 4개 alias 선택지로 단순화된다.
- chat picker 와 badge 는 alias-first 이고, concrete model ID 는 운영 상세 surface 에서만 본다.
- `smart-router-local` 은 더 이상 stale `qwen3.6` 정적 문구를 보여주지 않고 현재 default local runtime 을 가리킨다.
- `local-llm` direct target 은 backend/status surface 에는 남더라도 chat picker 기본 surface 에서는 숨겨진다.

## 8. 구현 후 상태 메모 반영

구현이 끝나면 아래를 함께 반영한다.

- `docs/operations/status-summary/nanobot-status-summary-2026-05-16.md` 또는 해당 날짜 memo 에 이번 UI alias 정리와 live local target sync 결과 기록
- 필요 시 repository memory 에 "WebUI picker 는 smart-router group only" 규칙 추가