# Phase 5 — 갱신된 plan 문서 목록

- **task_id**: `2026-04-12-parent-viewer-access`
- **seed_id**: `seed_37b35654542f`
- **interview_id**: `interview_20260412_111153`
- **ambiguity**: 0.074 (phase4 seed) / 0.15 (phase3 측정)
- **작성일**: 2026-04-12

---

## 갱신된 plan 문서 (5개)

| # | 파일 | 핵심 변경 |
|---|---|---|
| 1 | `ideation/plans/seeds-index.md` | Seed 7 행 추가(테이블 6→7 확장) · Seed 7 핵심 결정 표 신규 섹션(페어링·격리·알림·탈퇴 + cross-cutting 자녀 범위 매트릭스 요약) · 의존성 다이어그램에 Seed 7 노드 4개 추가 · 성능 예산 주석에 스마트폰 PWA 별도 예산 명시 |
| 2 | `ideation/plans/phase0-requests.md` | PV-1~PV-12 padlet 진입 JSON 블록 12개 신규 추가 (스키마·코드 생성·redeem·verify·middleware·PWA·자녀 범위 필터·교사 관리 UI·revoke SLA·주간 이메일·탈퇴·E2E 보안 게이트) |
| 3 | `ideation/plans/drawing-board-library-roadmap.md` | §확정 결정 "공유 범위" 행 학부모 문구 → Seed 7 매트릭스 참조. §파킹 "학부모 포트폴리오 열람" → "PDF 내보내기만 v2"로 축소(v1은 Seed 7에서 구현). §학부모 열람 범위 절 신규 추가(StudentAsset studentId ∈ parent.children, presigned 썸네일). 변경 로그 추가 |
| 4 | `ideation/plans/plant-journal-roadmap.md` | §3.4 학부모 뷰 "v2 예정" → "v1 Seed 7 PV-7 이관". PJ-8 작업을 Seed 7로 이관 완료 표시. 미결 "학부모 인증" → Parent·ParentChildLink·ParentInviteCode·ParentSession 4종 + 매직 링크로 해소. §11 변경 로그 신규 추가 |
| 5 | `ideation/plans/event-signup-roadmap.md` | §미결 "학부모 대리 신청" → Seed 7 범위 밖(read-only)으로 해소. §파킹 "카카오 알림톡 학부모 연결" → Seed 7 v2 파킹과 연계. §학부모 열람 범위 절 신규 추가(학급 이벤트 메타 허용, 자녀 본인 Submission만, 참가자 명단·득점 마스킹). 변경 로그 추가 |
| 6 | `ideation/plans/breakout-room-roadmap.md` | §8 학부모 열람 범위 절 신규 추가 — BreakoutMembership studentId ∈ parent.children, teacher-pool 제외, peek-others 무관 자녀 모둠만. §변경 로그에 Seed 7 행 추가. §8 → §9로 변경 로그 섹션 번호 이동 |

---

## 누적 변경 요약

- **단일 진실 원천 확립**: Seed 1·3·4·6에서 단편적으로 언급되던 "학부모 열람 범위"를 Seed 7의 cross-cutting 자녀 범위 매트릭스(`parent-viewer-roadmap.md#5`)로 일원화. 모든 feature 로드맵이 Seed 7을 참조하는 방향으로 정정됐다.
- **중복 방지**: 각 feature 로드맵은 학부모 뷰 로직을 직접 구현하지 않고 Seed 7의 PV-7에 데이터 소스만 제공하도록 작업 경계를 재정의.
- **v1 범위 확장**: plant-journal에서 "v2 예정"으로 밀려 있던 학부모 뷰가 Seed 7을 통해 v1 초기 배포 범위에 포함.
- **seeds-index 의존성 다이어그램**: Seed 7이 Seed 1·3·4·6의 "학부모 열람" single source of truth임을 명시. T0-④ 이미지 파이프라인 · Seed 2 Tier · BoardMember.role 승계 링크 추가.
