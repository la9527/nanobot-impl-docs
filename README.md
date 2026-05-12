# Nanobot Docs Map

이 디렉터리는 Nanobot 의 구현 계획, 운영 메모, 사용자 가이드, 과거 설계 문서를 함께 보관한다. 문서 수가 늘어나면서 같은 주제를 여러 위치에서 반복 설명하는 경우가 생겼기 때문에, 이제는 `지금 바로 봐야 하는 문서`와 `배경/기록용 문서`를 분리해서 읽는 것을 기본 원칙으로 둔다.

## 먼저 볼 문서

1. [planning/todo.md](./planning/todo.md)
   - 현재 active validation, 다음 착수 후보, deferred item 을 짧게 보는 top-level queue
2. [planning/execution-backlog/README.md](./planning/execution-backlog/README.md)
   - 새 구현 slice 를 열 때 기준이 되는 workplan index
3. [guide/README.md](./guide/README.md)
   - 현재 live Nanobot 사용/운영 기준 설명
4. [operations/status-summary/README.md](./operations/status-summary/README.md)
   - 날짜별 운영 메모와 검증 기록 위치

## 현재 디렉터리 구조

- `.github-source/`
  - Nanobot workspace instruction, file instruction, skill 의 실제 버전관리 위치
- `planning/`
  - active queue, conceptual draft, execution backlog
- `operations/`
  - 날짜형 status summary 와 운영 검증 메모
- `guide/`
  - 현재 사용자/운영 기준 설명서
- `archive/`
  - 보존이 필요한 historical design, review, sync 기록

## 디렉터리 역할 정리

### Active source of truth

- `.github-source/`
  - Nanobot workspace instruction, file instruction, skill 의 versioned source
- `planning/todo.md`
  - 지금 열려 있는 작업, validation, defer 상태를 짧게 유지하는 문서
- `planning/execution-backlog/`
  - 새 제품 구현 slice 를 정의하는 실행 workplan 디렉터리
- `guide/`
  - 현재 사용자/운영자 기준 설명 문서
- `operations/status-summary/`
  - 날짜형 운영 메모와 live 검증 결과

### Concept and design reference

- `planning/concepts/overview.md`
  - 개인비서 제품 축을 큰 그림에서 정리한 개념 요약
- `planning/concepts/`
  - 개별 도메인별 상세 conceptual draft

### Historical design and review archive

- `archive/assistant-direction-a/`
  - 초기 방향 A 설계와 흡수 전 배경 문서
- `archive/webui-ux-redesign/`
  - WebUI 정보 구조 재설계 검토 기록
- `archive/observable-reasoning-plan/`
  - observable reasoning 및 관련 UI/채널 설계 기록
- `archive/calendar-webui-reliability-review/`
  - Calendar/WebUI reliability 재점검 기록
- `archive/upstream-main-sync-2026-05-08/`
  - 특정 sync/merge 시점의 운영 기록

## 문서 정리 원칙

- active queue 는 `planning/todo.md` 에만 남긴다.
- 구현 slice 의 범위와 완료 기준은 `planning/execution-backlog/*.md` 에서 관리한다.
- 날짜형 작업 로그와 live 검증 결과는 `operations/status-summary/` 로 보낸다.
- Nanobot 루트 `.github` 는 로컬 호환용 symlink 이고, 실제 tracked source 는 이 repo 안의 `.github-source/` 로 유지한다.
- 이미 끝난 설계 토론은 삭제보다 `reference archive` 로 남기고, README 에 현재 상태를 명시한다.
- 새 구현을 다시 열 때는 기존 archive 문서를 직접 TODO 로 되살리지 말고, `planning/todo.md` 와 새 backlog 문서로 승격해서 다시 시작한다.