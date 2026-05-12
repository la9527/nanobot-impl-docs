---
applyTo: "source/webui/**"
description: "Use when editing the Nanobot WebUI React/Vite frontend, including components, tests, bootstrap wiring, and production bundle validation."
---

# Nanobot WebUI Instructions

이 instruction 은 Nanobot WebUI 파일군에만 적용한다.

## 적용 범위

- `source/webui/**`

## 기본 원칙

- WebUI 는 React 18 + TypeScript + Vite 기반이다.
- 운영 반영 기준은 dev server 가 아니라 production bundle 이다.
- UI 변경은 가능하면 관련 테스트와 함께 검증한다.
- 기존 디자인을 크게 재편하기보다 현재 대화형 assistant UI 흐름을 유지한다.

## 작업 방식

- 컴포넌트 동작 변경 시 직접 제어하는 파일부터 좁게 수정한다.
- 상태 표시, 스크롤, composer 동작 같은 UX 변경은 관련 테스트를 함께 추가하거나 갱신한다.
- WebUI API 기본 연결 경로는 local gateway 연동을 전제로 유지한다.

## 검증 기준

- 좁은 범위 테스트를 우선 실행한다.
- bundle 변경이 있으면 `npm run build` 로 dist 를 갱신한다.
- 운영 반영 확인은 gateway 재기동 후 authenticated `GET /webui/bootstrap` 또는 실제 브라우저 새로고침으로 본다.

## 권장 명령

```bash
cd /Volumes/ExtData/Nanobot/source/webui
npm test -- src/tests/thread-composer.test.tsx
npm run build
```

## 주의사항

- `vite dev` 는 로컬 개발 확인용일 수는 있지만, live 반영 경로로 취급하지 않는다.
- build 없이 source 만 수정하고 운영 반영이 끝났다고 판단하지 않는다.
