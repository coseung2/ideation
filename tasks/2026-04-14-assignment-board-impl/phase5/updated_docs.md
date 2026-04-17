# Phase 5 — Updated Docs

- **task_id**: `2026-04-14-assignment-board-impl`
- **seed**: `seed_38c34e91bf28`
- **실행일**: 2026-04-14

## 갱신된 plan 문서 (기존 파일 in-place 수정)

| # | 파일 | 변경 요약 |
|---|---|---|
| 1 | `plans/seeds-index.md` | (a) 헤더 "전체 Seed 11개 → 12개" 갱신. (b) 시드 목록표에 Seed 11 (assignment-board, `seed_38c34e91bf28`, ambiguity 0.083) 행 추가. (c) "Seed 11: 과제 배부 보드" 섹션 신규 (핵심 결정 14축). (d) 의존성 그래프에 Seed 11 관계 5건 추가 (Board.layout 패턴·Submission 네임스페이스 분리·자녀 범위 매트릭스·tablet 성능 §2a·Canva 시너지). |
| 2 | `plans/tablet-performance-roadmap.md` | §2a "30-카드 5×6 정형 격자 성능 예산" 신규 절 추가 — DOM ≤ 30 · 카드당 자식 ≤ 6 · 썸네일 160×120 WebP · CSS 상태 토글 · S-Pen 카드 금지 · iframe v1 금지 · WebSocket `board:${id}:assignment` 단일 채널 등 12개 지표 + phase9 QA 게이트 7개. 하단 변경 로그 추가. |
| 3 | `plans/event-signup-roadmap.md` | "Submission 엔티티 공유 — assignment-board (Seed 11)" 절 신규 추가 — `Submission.status`(event-signup 6값)와 `AssignmentSlot.submissionStatus`(assignment 6값)의 네임스페이스 분리 명시, Zod layout 기반 분기 합의, `SubmissionHistory` 공통 v2 파킹. 변경 로그 2026-04-14 행 추가. |
| 4 | `plans/parent-viewer-roadmap.md` | (a) §5 자녀 범위 매트릭스에 "과제 배부 보드 (Seed 11)" 행 추가 — `AssignmentSlot.studentId ∈ parent.children` 필터, 5×6 격자 미렌더, 자녀 전용 단일 카드 뷰. (b) §6 `/parent/*` PWA 라우트 트리에 `/parent/child/[id]/assignment` 추가. (c) §11 변경 로그 2026-04-14 행 추가 (구현은 PV-7 서버 필터에서 처리, 신규 작업 카드 없음). |
| 5 | `plans/phase0-requests.md` | "과제 배부 보드 (assignment-board-roadmap.md, Seed 11)" 섹션 신규 — 2개 JSON 블록(AB-1 마이그레이션 + AB-2~AB-10 통합 feature)을 padlet 파이프라인 진입 포맷으로 등재. context_refs에 seed.yaml·decisions.md·sketch.md·exploration.md·assignment-board-roadmap.md·tablet-performance §2a·event-signup Submission 공유 절·parent-viewer §5 매트릭스 링크 포함. |

## 갱신 건수: **5**

## 검증

- `seeds-index.md` Seed 11 행 열 수 = 헤더 열 수 (7열) ✓
- `phase0-requests.md` context_refs 링크가 실제 섹션 헤더와 일치 ✓
- Submission 네임스페이스 분리 합의가 event-signup 로드맵과 assignment-board 로드맵 양쪽에 동일 기술 ✓
- parent-viewer §5 매트릭스 행 열 수 (7열) = 기존 행 동일 구조 ✓

## 이전 시드 supersede 여부

- **Seed 11은 parent_seed_id = null** (신규 시드, 기존 시드 대체하지 않음).
- Seed 7 (parent-viewer v1 → v2) supersede 관계 재검토 → 영향 없음. 본 시드의 학부모 뷰 요건은 Seed 7-v2 `seed_6d7077aac472`의 §5 매트릭스에 편입되었으며, Seed 7-v2를 대체하지 않는다.
- Seed 3 (event-signup) supersede 아님. Submission 재사용은 **병존 확장**이며 충돌 없음.
- Seed 6 (Breakout) supersede 아님. `Board.layout` 확장 패턴만 승계.
- seeds-index의 supersede 관계 구조에 변경 없음.
