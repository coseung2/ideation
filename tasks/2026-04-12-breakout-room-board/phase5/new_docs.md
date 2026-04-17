# Phase 5 — 신규 생성 문서 목록

> task_id: `2026-04-12-breakout-room-board`
> seed_id: `seed_bb1d4eb1c442`
> integrator 실행: 2026-04-12

## 신규 `plans/*.md`

| # | 파일 | 요약 |
|---|---|---|
| 1 | `ideation/plans/breakout-room-roadmap.md` | Breakout Room 보드 로드맵 신규 생성. 섹션 구성: 0 핵심 명제 / 1 설계 전제(엔티티 최소화·배포·열람·기본값·복제 방식·Tier·템플릿 카탈로그) / 2 Prisma 데이터 모델(BreakoutTemplate·BreakoutAssignment·BreakoutMembership) / 3 작업 분할 BR-1~BR-9(+파킹 BR-A~D) / 4 의존 로드맵 상호 참조(T0-① / drawing-board / Seed 2 Tier) / 5 수용 기준(seed.yaml 그대로) / 6 리스크 완화 / 7 파킹 / 8 변경 로그 |

## 신규 `tasks/{task_id}/phase5/*.md`

| # | 파일 | 용도 |
|---|---|---|
| 1 | `tasks/2026-04-12-breakout-room-board/phase5/updated_docs.md` | 갱신 plan 목록 + 링크 검증 + 품질 바 점검 |
| 2 | `tasks/2026-04-12-breakout-room-board/phase5/new_docs.md` | 본 파일 — 신규 문서 목록 |

## 신규 `data/*.json` 시드 데이터

본 phase5에서는 시드 데이터 JSON을 직접 생성하지 않음. BR-2 작업 파이프라인에서 `ideation/data/breakout-template-seed.json`(시스템 템플릿 8종)을 실제 구현 단계에 생성하도록 `phase0-requests.md` BR-2 블록의 `seed_data_ref`로 지정함.
