# Nanobot 작업 상태 정리 - 2026-05-16

## 문서 목적

이 문서는 해당 날짜의 운영 변경과 검증 결과를 간단히 기록하기 위한 status summary 초안이다.

<!-- llmtesttool-status:start -->
## LLMTestTool_v2 비교 벤치마크 요약
- 비교 run: `lfm2__vs__qwen35-base-mlx-4bit/bundle-standard-r10-2026-05-16_214004`
- 실행 시각: `2026-05-16_214004`
- 요약 모델: `lfm2` (API 경유 전체 시간 기준)
- 가장 빠른 시작 시간: `lfm2`
- 가장 빠른 벤치마크 시간: `lfm2`
- 가장 빠른 전체 시간: `lfm2`
- 상세 비교 Markdown: `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/compare/lfm2__vs__qwen35-base-mlx-4bit/bundle-standard-r10-2026-05-16_214004/comparison.md`
- 최종 요약 Markdown: `/Volumes/ExtData/AI_Project/LLMTestTool_v2/results/reports/report-summary-lfm2_vs_qwen35-base-mlx-4bit-2026-05-16_214004.md`

### 모델 간 핵심 차이

| 지표 | lfm2 | qwen35-base-mlx-4bit | 차이 | 우세 모델 |
| --- | ---: | ---: | ---: | --- | 
| 직접 호출 시작 시간 | 18153 | 20226 | -2073 | lfm2 |
| 직접 호출 벤치마크 시간 | 192 | 3180 | -2988 | lfm2 |
| 직접 호출 전체 시간 | 18856 | 23686 | -4830 | lfm2 |
| quick.ttftMs | 176 | 3013 | -2837 | lfm2 |
| quick.promptTokens | 14 | 17 | -3 | lfm2 |
| quick.completionTokens | 2 | 1 | 1 | qwen35-base-mlx-4bit |
| quick.totalTokens | 16 | 18 | -2 | lfm2 |
| quick.completionTokensPerSec | 10.42 | 0.31 | 10.11 | lfm2 |
| API 경유 전체 시간 | 15855 | 132701 | -116846 | lfm2 |
| assistant.ttftMs | 3319 | 111829 | -108510 | lfm2 |
| assistant.promptTokens | 6798 | 6128 | 670 | qwen35-base-mlx-4bit |
| assistant.completionTokens | 5 | 1 | 4 | qwen35-base-mlx-4bit |
| assistant.totalTokens | 6803 | 6129 | 674 | qwen35-base-mlx-4bit |
| assistant.completionTokensPerSec | 1.46 | 0.01 | 1.45 | lfm2 |
| 부하 테스트 실패율 | 0 | 0 | 0 | 동점 |
| 부하 테스트 최악 지연 | 29077 | 170893 | -141816 | lfm2 |
<!-- llmtesttool-status:end -->

## Nanobot Local LLM phase-1 반영 상태

### 현재 상태 요약

- phase-1 기본 local LLM target 은 `qwen36` 으로 반영했다.
- live override 파일은 `~/.nanobot/local-llm.env` 이며, `~/.nanobot/nanobot.env` 를 직접 수정하지 않는 구조로 적용했다.
- gateway/API launchd wrapper 는 `nanobot.env` 를 먼저 source 한 뒤 `local-llm.env` 를 source 하므로, local LLM 변경값이 base env 를 덮어쓴다.
- live 기준 `com.nanobot.local-model-qwen36` 은 loaded 상태이고 listener / endpoint 모두 OK 로 확인했다.

### 이번에 반영한 주요 작업

- `infra/scripts/local-models/local-models.sh` 에 `restart`, `use` 계열 명령을 추가했다.
- `infra/scripts/local-models/use-local-model.sh` 는 target start, smoke test, `~/.nanobot/local-llm.env` 갱신을 한 흐름으로 처리한다.
- source runtime 에 `nanobot.local_llm_control` 을 추가해 WebUI route 와 slash command 가 같은 target metadata / action runner 를 사용하게 했다.
- WebUI Settings 에 Local LLM 섹션을 추가하고 qwen36 / lfm2 상태, endpoint 상태, start/restart/smoke/stop/use action 을 노출했다.
- Telegram command menu 에 `/local_llm` 을 추가하고 runtime command 는 `/local-llm` 로 정규화한다. `stop`, `restart`, `use` 는 `/local-llm confirm ...` 확인 흐름을 거치게 했다.

### 검증 결과

- source focused tests: `15 passed, 37 deselected`
	- `tests/test_local_llm_control.py`
	- `tests/command/test_builtin_local_llm.py`
	- `tests/config/test_model_targets.py`
	- `tests/channels/test_websocket_http_routes.py` 의 local/model-target 관련 subset
- WebUI tests: `src/tests/api.test.ts`, `src/tests/settings-view.test.tsx` 기준 `9 passed`.
- WebUI production bundle: `npm run build` 성공.
- Infra shell validation: local model scripts, gateway/API wrappers, setup/validate scripts `zsh -n` 통과.
- Live validation:
	- `infra/scripts/local-models/local-models.sh use qwen36` 성공.
	- `curl -fsS http://127.0.0.1:18790/health` 는 `{"status":"ok"}` 반환.
	- API health 는 `8900` 계열 endpoint 에서 `{"status":"ok"}` 반환.
	- authenticated `GET /api/local-llm/status` 는 `default_target=qwen36`, qwen36 `running=true`, `endpoint_ok=true` 를 반환.

### 남아 있는 제약과 리스크

- WebUI embedded HTTP surface 의 local LLM action 은 기존 settings/session action route 와 맞춰 GET 기반이다. 외부 API 서버로 분리하면 POST variant 를 별도 검토한다.
- 넓게 `tests/channels/test_websocket_http_routes.py` 전체를 실행했을 때 local LLM 과 무관한 locale expectation 실패가 1건 있었다. focused local LLM route 검증은 통과했다.
- Telegram destructive action 은 phase-1 에서 텍스트 confirm subcommand 로 보호한다. 이후 inline approval UI 가 필요하면 별도 phase 로 분리한다.

### 현재 기준 권장 운영 명령

```bash
/Volumes/ExtData/Nanobot/infra/scripts/local-models/local-models.sh use qwen36
/Volumes/ExtData/Nanobot/infra/scripts/stop-nanobot-services.sh
/Volumes/ExtData/Nanobot/infra/scripts/start-nanobot-services.sh
```
