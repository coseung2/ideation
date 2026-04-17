# Phase 5 Updated Docs — Canva Publisher Receiver

> task_id: `2026-04-12-canva-publisher-receiver`
> seed: `seed_26af361e92b7`
> 작성: 2026-04-12 · integrator 에이전트

---

## 갱신된 문서

### 1. `ideation/plans/seeds-index.md`
- 헤더 "전체 Seed 7개" → **"전체 Seed 8개"**
- 표에 Seed 8 행 추가 (Canva Publisher 수신 엔드포인트 & PAT, `seed_26af361e92b7`, ambiguity 0.121, `canva-publisher-receiver-roadmap.md`)
- "🎯 Seed 8" 핵심 결정 표 신규 섹션 — D1~D16 16건 요약(엔티티·PAT 포맷·1회 노출·유효기간·scope·tier·rate limit·body 스키마·boardId 검증·응답·업로드 한도·저장소·스트리밍·마이그·sectionId·CRC32·딥링크·분실 정책·교사 UI·Card 기본값)
- "🔗 인접 시드 간 의존성" 그래프에 Seed 8 네 줄 추가 (Seed 2·Seed 5·v2 확장점·tablet-performance 연결)

### 2. `ideation/plans/phase0-requests.md`
- 말미 "사용 지침" 섹션 직전에 **"Canva Publisher 수신 (canva-publisher-receiver-roadmap.md)"** 신규 섹션 삽입
- CR-1~CR-10 JSON 블록 10개 추가:
  - CR-1 스키마 & 마이그 Stage 1
  - CR-2 external-auth.ts PAT 유틸
  - CR-3 POST /api/external/cards 핵심 엔드포인트
  - CR-4 Upstash 3축 rate limit
  - CR-5 Vercel Blob 스트리밍 업로드
  - CR-6 /api/tokens CRUD
  - CR-7 교사 UI 갤탭 S6 Lite
  - CR-8 마이그 Stage 2·3 (legacy revoke + NOT NULL)
  - CR-9 /api/external/healthz 모니터
  - CR-10 E2E 보안 게이트 10종

### 3. `ideation/plans/implementation-roadmap.md`
- "## P0-② Content Publisher Intent 앱" 섹션 도입부에 인용 블록 추가:
  - **"서버 수신 구현 (padlet 측)은 Seed 8(`canva-publisher-receiver-roadmap.md`)에서 처리"** 명시
  - 상세 범위 분할 — implementation-roadmap은 Canva 앱 프로젝트(`content-publisher-app/`)의 SSOT, Seed 8은 padlet 측 수신 인프라 SSOT

---

## 변경 로그 (요약)

| 문서 | 변경 종류 | 핵심 내용 |
|---|---|---|
| seeds-index.md | 확장 | Seed 8 등재 + 16건 결정 표 + 의존성 그래프 갱신 |
| phase0-requests.md | 확장 | CR-1~CR-10 블록 신규 섹션 |
| implementation-roadmap.md | 삽입 | P0-② 수신측 Seed 8 크로스 참조 |
