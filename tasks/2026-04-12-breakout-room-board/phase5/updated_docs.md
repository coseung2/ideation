# Phase 5 — 갱신된 문서 목록

> task_id: `2026-04-12-breakout-room-board`
> seed_id: `seed_bb1d4eb1c442`
> integrator 실행: 2026-04-12

## 갱신 대상 `plans/*.md`

| # | 파일 | 변경 요약 |
|---|---|---|
| 1 | `ideation/plans/seeds-index.md` | Seed 6 행 추가(전체 5 → 6), 연관 plan 컬럼 추가, Seed 6 핵심 결정 표 신설(Section 재활용·템플릿 8종·Free 3/Pro 5·own-only 기본·v1 학생이동 불허·복사 방식·teacher-pool 단일·isPublic=true 등), 의존성 그래프에 Seed 6 ↔ T0-① / Seed 1 / Seed 2 크로스 엣지 추가 |
| 2 | `ideation/plans/phase0-requests.md` | "Breakout Room 보드" 섹션 신설, BR-1~BR-9 총 9개 padlet 진입 JSON 블록 추가. 모든 블록에 `context_refs`로 `breakout-room-roadmap.md`·`tablet-performance-roadmap.md`(T0-①)·`drawing-board-library-roadmap.md` 링크 포함 |
| 3 | `ideation/plans/tablet-performance-roadmap.md` | §5 T0-① 섹션 격리 Breakout 뷰 상단에 크로스 참조 블록 추가 — "이 작업은 breakout-room-roadmap.md(Seed 6)의 구현 전제" 명시, Section.accessToken 선행 마이그레이션 중복 금지 규칙 명문화 |
| 4 | `ideation/plans/drawing-board-library-roadmap.md` | 하단에 "재사용 포인트 (Seed 6 참조)" 섹션 신설 — AssetAttachment 복사 패턴·"템플릿 고르기" 모달 UX가 Breakout Room BR-3/BR-4로 승계됨을 명시 |

## 상호 참조 링크 검증

- `seeds-index.md` → `breakout-room-roadmap.md` (신규 plan 링크) ✅
- `phase0-requests.md` BR-1~BR-9 → `breakout-room-roadmap.md` 앵커 ✅
- `phase0-requests.md` BR-5/BR-6 → `tablet-performance-roadmap.md#5-별도-작업-t0-①` ✅
- `tablet-performance-roadmap.md` T0-① → `breakout-room-roadmap.md` ✅
- `drawing-board-library-roadmap.md` 하단 → `breakout-room-roadmap.md` ✅
- `breakout-room-roadmap.md` §4 → `tablet-performance-roadmap.md` T0-① / `drawing-board-library-roadmap.md` / Seed 2 ✅

## 품질 바 점검

- [x] 기존 plan의 "미결 사항" 중 phase3 결정 사항 → 본 시드는 신규 주제라 해당 없음
- [x] 변경 로그 섹션 — `breakout-room-roadmap.md` §8에 2026-04-12 초안 생성 행 추가
- [x] seeds-index에 새 행 + 핵심 결정 표 반영
- [x] 이전 시드와 상충하는 결정 없음 (Seed 2 Tier 정책·Seed 1 복사 패턴 승계)
- [x] phase0-requests.md에 padlet 진입 JSON 블록(context_refs 포함) 추가
