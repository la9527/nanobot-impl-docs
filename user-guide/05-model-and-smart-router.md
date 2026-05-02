# 05. 모델 선택과 smart-router

## 모델 target 이란

모델 target 은 Nanobot 이 어떤 LLM 실행 경로를 사용할지 정해 둔 이름이다.

예를 들어 아래처럼 나뉠 수 있다.

- local LLM 직접 사용
- smart-router 자동 선택
- smart-router local tier
- smart-router mini tier
- smart-router full tier

## 현재 모델 확인

채팅창에 입력한다.

```text
/model
```

또는 상태까지 함께 보고 싶으면 아래도 사용할 수 있다.

```text
/status
```

## 선택 가능한 모델 보기

```text
/model list
```

현재 운영 기준으로 smart-router 가 설정되어 있으면 아래 target 들이 보일 수 있다.

- `smart-router`: 자동 선택
- `smart-router-local`: local tier 우선
- `smart-router-mini`: mini tier 우선
- `smart-router-full`: full tier 우선
- `local-llm`: local direct baseline

## smart-router 란

smart-router 는 질문의 성격에 따라 적절한 모델 tier 를 고르는 기능이다.

쉽게 말하면 아래 판단을 대신한다.

- 간단한 작업이면 local 또는 가벼운 모델
- 조금 복잡하면 mini
- 더 복잡하고 정확도가 필요하면 full

이렇게 해서 비용, 속도, 품질 사이의 균형을 잡는다.

## target 변경하기

자동 선택을 쓰려면 아래처럼 입력한다.

```text
/model smart-router
```

local tier 를 우선 쓰고 싶으면 아래처럼 입력한다.

```text
/model smart-router-local
```

mini 또는 full 을 쓰고 싶으면 아래처럼 입력한다.

```text
/model smart-router-mini
/model smart-router-full
```

## 세션별 설정

`/model <target>` 은 현재 채팅 세션에만 적용되는 override 로 동작할 수 있다.

즉, 한 채팅에서 smart-router 를 선택해도 다른 채팅은 기존 기본값을 유지할 수 있다.

## 원래 기본값으로 되돌리기

```text
/model clear
```

이 명령을 쓰면 현재 세션의 override 를 지우고 startup 기본 target 으로 돌아간다.

## WebUI 에서 보는 방법

WebUI 상단에는 현재 target 이 표시된다.

예시:

```text
Target Auto
```

Settings 화면에서도 모델 관련 상태를 볼 수 있다.

## 초보자 추천

처음에는 아래처럼 쓰면 된다.

```text
/model
/model list
/model smart-router
```

특정 작업에서 답변이 느리거나 부족하면 `smart-router-mini` 또는 `smart-router-full` 을 직접 선택해 볼 수 있다.

## 주의사항

- target 이 목록에 보인다고 모든 remote fallback 이 live 검증됐다는 뜻은 아니다.
- API key, local runtime, smart-router plugin 설정이 모두 맞아야 정상 동작한다.
- 모델을 바꿨는데 이상하면 `/model clear` 로 되돌린 뒤 `/status` 를 확인한다.
