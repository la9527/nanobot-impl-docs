# Nanobot Status Summary

이 디렉터리는 날짜별 운영 메모와 live 검증 기록을 모아 두는 위치다.

## 목적

- source tree 변경, WebUI build, launchd 재기동, live 검증 결과를 날짜 기준으로 남긴다.
- `todo.md` 에는 유지하기 너무 긴 구현 이력과 운영 메모를 이쪽으로 분리한다.
- 특정 시점의 runtime 상태를 다시 추적해야 할 때 reference log 역할을 한다.

## 포함할 내용

- source/docs 변경과 관련된 live 반영 결과
- gateway/API/WebUI health 확인 결과
- 실사용 채널 검증 결과
- issue 조사와 원인 정리 메모

## 포함하지 않을 내용

- 다음 작업의 active checklist
- 장기 제품 우선순위
- conceptual design discussion 전체

## 작성 규칙

- 파일명은 `nanobot-status-summary-YYYY-MM-DD.md` 형식을 따른다.
- 문서 첫 부분에 목적과 현재 상태 요약을 짧게 둔다.
- 구현 메모보다 `무엇을 확인했고 현재 어떤 상태인지` 를 우선 적는다.