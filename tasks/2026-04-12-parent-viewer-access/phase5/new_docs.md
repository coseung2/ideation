# Phase 5 — 신규 생성 plan 문서 목록

- **task_id**: `2026-04-12-parent-viewer-access`
- **seed_id**: `seed_37b35654542f`
- **작성일**: 2026-04-12

---

## 신규 생성 plan 문서 (1개)

| # | 파일 | 섹션 구성 | 주요 산출 |
|---|---|---|---|
| 1 | `ideation/plans/parent-viewer-roadmap.md` | §0 핵심 명제 · §1 확정 결정 (1.1 페어링/인증 · 1.2 Revoke/격리 · 1.3 알림/Tier · 1.4 탈퇴/감사) · §2 데이터 모델 Prisma 4종 + RLS · §3 페어링 흐름 (교사 발급, Revoke) · §4 parentScopeMiddleware 설계 · **§5 Cross-cutting 자녀 범위 매트릭스 (single source of truth)** · §6 /parent/* PWA 구조 · §7 작업 분할 PV-1~PV-12 · §8 수용 기준 · §9 리스크 · §10 파킹 · §11 변경 로그 | Parent·ParentChildLink·ParentInviteCode·ParentSession 4종 Prisma 스키마 + BoardMember.role += "parent" + RLS parent_id=auth.parent_id() 단방향 정책. Crockford Base32 6자리 코드 + 이중 rate limit(IP 5회/15분 + 코드 10회 즉시 만료). 매직 링크 15분 + ParentSession 7일. Revoke SLA ≤ 60s. Soft delete + 90일 익명화 Cron. §5 매트릭스는 Seed 1(StudentAsset)·Seed 3(EventBoard·Submission)·Seed 4(PlantObservation)·Seed 6(BreakoutMembership) 전부의 학부모 열람 규칙을 통합 단일 진실로 확정. PV-1~PV-12 12개 작업(스키마·코드·redeem·verify·middleware·PWA·자녀 범위 필터·교사 UI·revoke·주간 이메일·탈퇴·E2E). 1차 배포 PV-1~PV-9+PV-12, 2차 PV-10+PV-11. |

---

## Seed 생성물과의 매핑

- **seed.yaml**: 모든 ontology_schema.fields(parentId·parentEmail·displayName·revokedAt·deletedAt·emailSummaryOptOut·parentChildLinkId·status·revokedReason·issuedById·inviteCode·expiresAt·maxUses·usedCount·failedAttempts·sessionToken·sessionExpiresAt·sessionRevokedAt·boardMemberRole) → Prisma 스키마 §2 1:1 반영
- **acceptance_criteria**: seed 13개 전체 → 본 로드맵 §8 수용 기준 13개 그대로 인용
- **constraints**: 스마트폰 PWA 예산·iframe 금지·SWR 60s·자녀 5명 상한·RLS 단방향 등 15개 → §1 및 §6 분산 반영
- **evaluation_principles**: security(0.3)·privacy_isolation(0.25)·performance(0.2)·auditability(0.15)·ux_frictionless(0.1) → §9 리스크 테이블 및 §12 E2E 게이트에서 QA 검증 포인트와 연결

---

## 후속 작업 (phase6 handoff 대상)

1. **handoff-writer**: 본 로드맵 + seeds-index + phase0-requests를 phase6 로드맵 통합문서로 요약.
2. **`ooo run seed_37b35654542f`** (phase7 실행) 진입 시 padlet 파이프라인 `tasks/2026-04-12-parent-viewer-access-impl/phase0/request.json`에 PV-1 블록 복사 → 스키마 마이그레이션 착수.
3. PV-5 middleware · PV-7 자녀 범위 필터는 feature 로드맵 간 cross-cutting이므로 통합 PR 권장.
