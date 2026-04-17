# Phase 5 New Docs — Canva Publisher Receiver

> task_id: `2026-04-12-canva-publisher-receiver`
> seed: `seed_26af361e92b7`
> 작성: 2026-04-12 · integrator 에이전트

---

## 신규 생성 문서

### `ideation/plans/canva-publisher-receiver-roadmap.md`

**목적**: implementation-roadmap의 P0-② Content Publisher Intent 중 **서버(padlet) 측** 수신 인프라 구현의 single source of truth. Seed 8 `seed_26af361e92b7`의 D1~D16 결정과 seed.yaml acceptance_criteria를 설계 언어로 변환.

**섹션 구성 (10개)**

| # | 섹션 | 내용 |
|---|---|---|
| 0 | 핵심 명제 | "Pro 인증 교사가 PAT로 4MB 이하 PNG를 스트리밍 업로드해 2초 이내 카드 생성. 토큰 1회 노출." |
| 1 | 설계 전제 (Phase 3 확정) | 1.1 엔티티 확장만(ExternalAccessToken에 tokenPrefix·label·lastUsedAt·scopeBoardIds 추가), 1.2 3-stage 마이그, 1.3 Tier 게이팅, 1.4 3축 rate limit, 1.5 스트리밍 업로드, 1.6 Zod strict 계약, 1.7 에러 코드 표 12종, 1.8 Card 기본값 |
| 2 | 신규·변경 파일 맵 | prisma/schema · migrations × 3 · src/lib/external-auth.ts · rate-limit.ts · app/api/external/cards · app/api/tokens × 2 · teacher UI × 2 · next.config.ts · .env.example |
| 3 | 교사 UI | `/(teacher)/settings/external-tokens` 갤탭 S6 Lite 최적화 — 목록·발급 모달·1회 공개 모달(Copy+Download)·Tier 잠금 UX |
| 4 | 작업 분할 CR-1~CR-10 | 스키마, PAT 유틸, 수신 엔드포인트, rate limit, Blob 스트리밍, /api/tokens CRUD, 교사 UI, 마이그 Stage 2·3, healthz, E2E 보안 게이트 |
| 5 | 수용 기준 | seed.yaml acceptance_criteria 완전 반영 (15개 체크리스트) |
| 6 | 리스크 & 완화 | R1~R9 — p95·legacy migration·4.5MB 초과·평문 유출·timing side-channel·Upstash 장애·Free 강등·Canva 이슈·Seed 2 도입 전 |
| 7 | 의존 그래프 | Seed 2·aura-canva-app·tablet-performance·Vercel Blob 전제 + v2 확장 포인트 |
| 8 | 학부모 열람 범위 | Seed 7 매트릭스 일반 Card 범주 편입 (별도 구현 작업 없음) |
| 9 | 파킹 v2+ | 10항목 — 프리-리사이즈·deeplink·Blob 정리 Cron·Webhook receive·metadata JSON·CRC32·sectionId UI·S3·학교 일괄결제·토큰 회전 알림 |
| 10 | 변경 로그 | 2026-04-12 초안 등록 |

**연결점**
- 상위 기획: `implementation-roadmap.md#p0-②` (Canva 앱 측과의 2-산출물 분리 명시)
- Tier 승계: `seeds-index.md` Seed 2
- 교사 UI 예산: `tablet-performance-roadmap.md §2`
- 학부모 열람: `parent-viewer-roadmap.md §5`
- 작업 카드: `phase0-requests.md` CR-1~CR-10 블록

---

## 총계

- 신규 plan 문서: **1개** (`canva-publisher-receiver-roadmap.md`)
- 갱신 plan 문서: **3개** (seeds-index·phase0-requests·implementation-roadmap)
- 작업 카드: **10개** (CR-1 ~ CR-10)
- 적용된 decisions: **16건** (D1~D16 전건)
