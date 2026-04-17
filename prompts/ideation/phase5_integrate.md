# Phase 5 — Integrate

살아있는 문서(`plans/`·`research/`·`data/`) 갱신 + `seeds-index.md` 등재.

## 입력

- `tasks/{task_id}/phase4/seed.yaml`
- `tasks/{task_id}/phase3/decisions.md`
- 기존 `plans/` 관련 문서(있는 경우)

## 출력

- `tasks/{task_id}/phase5/updated_docs.md` — 갱신 목록
- `tasks/{task_id}/phase5/new_docs.md` — 신규 생성 목록
- 실제 `plans/`·`research/`·`data/` 파일 수정 또는 생성
- `plans/seeds-index.md` 갱신 (해당 시드 등재)

## 절차

1. `seed.yaml`의 `constraints`·`acceptance_criteria`를 기존 plan 문서 형식으로 재작성
2. 관련 plan 문서가 있으면 **업데이트** (확정된 결정·수치로 교체)
3. 없으면 **신규 작성** (`plans/{topic}-roadmap.md`)
4. 필요한 경우 `data/` 에 seed data JSON 생성 (예: 식물 카탈로그)
5. `seeds-index.md`에 이번 시드 행 추가 + 핵심 결정 표 업데이트
6. `plans/phase0-requests.md` 에 padlet 진입 템플릿 블록 추가 (phase6와 공동 작성)
7. 관련 문서 간 상호 참조 링크 갱신

## 문서 갱신 규칙

- 기존 문서의 "미결 사항" 항목이 phase3에서 결정된 경우 → 해당 항목을 "확정 결정" 표로 이동
- "파킹됨(v2+)" 항목이 있으면 `ideas-parking-lot.md`에 섹션 추가
- 새 제약(예: 태블릿 예산 재조정)은 `tablet-performance-roadmap.md`에도 동기화

## 자율 진행 지침

- 기존 plan 문서와 결정 충돌 시 **새 결정(시드)이 우선**. 기존 문서는 덮어쓰고 변경 로그만 문서 하단에 표시.
- 동일 주제에 여러 시드가 쌓이면 최신 시드 기준으로 통합. 예전 시드는 `seeds-index.md`에 "superseded" 표시.

## 검증 게이트

- `seeds-index.md`에 새 시드 행 존재
- 관련 `plans/` 문서 ≥ 1개 업데이트되었거나 신규 생성됨
- `phase0-requests.md`에 진입 템플릿 블록 추가됨 (phase6에서 최종 확인)

## 핸드오프

`updated_docs.md` + `new_docs.md` phase6에 전달.
