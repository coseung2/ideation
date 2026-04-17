# Phase 5 new_docs — Parent Class Invite Refine

- **task_id**: `2026-04-13-parent-class-invite-refine`
- **seed_id**: `seed_6d7077aac472` (refinement of `seed_37b35654542f`)

## 신규 파일 생성 없음

본 refinement는 **기존 로드맵의 in-place 메이저 갱신**으로 처리되었다. 계약서 원칙 "시드는 별개, 살아있는 문서는 in-place" + "refinement 처리 시 supersede 표기만 적용, 원 plan은 갱신"에 따름.

신규 plan 파일·신규 research 파일·신규 data 파일을 만들지 않았다. 대신 기존 4개 문서를 갱신했다 (`updated_docs.md` 참조).

## phase5 산출 목록

| 파일 | 상태 | 설명 |
|---|---|---|
| `plans/parent-viewer-roadmap.md` | updated | v2 메이저 갱신 (섹션 15개 수정/신설) |
| `plans/seeds-index.md` | updated | Seed 7 supersede 표기 + Seed 7-v2 신규 행·섹션 |
| `plans/phase0-requests.md` | updated | PV-v2-BUNDLE 블록 신규 |
| `ideas-parking-lot.md` | updated | parent-viewer v2 refinement 파킹 6개 |
| `tasks/.../phase5/updated_docs.md` | **new (phase5 산출)** | 본 phase5의 갱신 범위 보고서 |
| `tasks/.../phase5/new_docs.md` | **new (phase5 산출)** | 이 파일 |

## 다음 phase

**phase6 handoff** — `handoff-writer` 에이전트 호출:
- `tasks/2026-04-13-parent-class-invite-refine/phase6/padlet_phase0_request.json` 생성 (`plans/phase0-requests.md`의 PV-v2-BUNDLE 블록을 독립 파일로 추출)
- `tasks/2026-04-13-parent-class-invite-refine/phase6/handoff_note.md` 생성
- destination: padlet (`parent-viewer v2` feature module, `padlet/apps/web/features/parent-access/`)

## 후속 phase 고려사항

**phase7 dispatcher** 시점에 결정할 항목 (참고):
- `seed_37b35654542f` (v1) → `destinations/archive/`로 이관 (supersede 완료된 원시드)
- `seed_6d7077aac472` (v2) → padlet INBOX로 배송
