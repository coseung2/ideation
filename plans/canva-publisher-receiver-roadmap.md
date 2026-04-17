# Aura-board Canva Content Publisher 수신 엔드포인트 & PAT 로드맵

> 작성일: 2026-04-12
> Seed: `seed_26af361e92b7` (task `2026-04-12-canva-publisher-receiver`)
> Interview: `interview_20260412_124111` (ambiguity 0.12)
> 전제 로드맵 / 링크:
> - `ideation/plans/implementation-roadmap.md#p0-②-content-publisher-intent-앱-canva--aura-board` (상위 기획 — 수신 서버측 본 로드맵이 담당)
> - `ideation/plans/seeds-index.md` Seed 2 (Tier 정책 승계 — `cards:write`는 Pro 전용)
> - `ideation/plans/seeds-index.md` Seed 5 (P0-② 프라이빗 교사 앱 한정)
> - `ideation/plans/tablet-performance-roadmap.md#2-성능-예산` (갤탭 S6 Lite 교사 UI 예산)
> - 클라이언트 계약 원본: `aura-canva-app/src/intents/content_publisher/index.tsx` (읽기 전용)

---

## 0. 핵심 명제

> **Canva 에디터에서 "Aura-board에 게시" 버튼을 누른 교사가, Pro 인증된 PAT로 4MB 이하 PNG를 스트리밍 업로드해 2초 이내에 보드 카드로 꽂는다. 토큰은 평문으로 저장되지 않고, 1회만 노출된다.**

aura-canva-app(Canva 측) 클라이언트는 이미 `{boardId,title,imageDataUrl,sectionId?}` 페이로드와 `200 {id,url}` 응답 계약을 고정했다. 본 로드맵은 그 수신측 — **PAT 시스템 확장 + `/api/external/cards` 라우트 + 교사 토큰 관리 UI** — 구현 사양을 확정한다.

---

## 1. 설계 전제 (Phase 3 확정 결정)

### 1.1 엔티티 — 신규 테이블 없음, `ExternalAccessToken` 확장만

Seed 5 P2-⑤(Webhook) / Seed 2(Tier)와 일관되게, **신규 테이블 금지**. 기존 `ExternalAccessToken` 엔티티에 필드 추가로 처리.

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | cuid PK | 기존 유지 |
| `tokenPrefix` | String @unique | **신규**. 8-char base62. O(1) DB 조회용. 마이그레이션 3단계 후 NOT NULL |
| `tokenHash` | String | **신규(또는 기존 재정의)**. `SHA-256(secret ‖ PEPPER)`. 평문 저장 금지 |
| `scopes` | String[] | v1 고정 `['cards:write']`. v2 확장 (`submissions:write`·`webhooks:receive`) 여지 유지 |
| `scopeBoardIds` | String[] | **신규**. 허용 boardId CUID allowlist. 빈 배열 = 교사 소유 모든 보드 허용 |
| `teacherId` | String FK User | 기존 유지. 발행 교사 = Card.authorId |
| `expiresAt` | DateTime? | null = 무기한. 기본 90일 |
| `revokedAt` | DateTime? | 기존 유지. soft revoke |
| `label` | String | **신규**. 교사가 인지 가능한 토큰 이름(예: "Canva 3-2반") |
| `lastUsedAt` | DateTime? | **신규**. 감사·정리용 |
| `createdAt` / `updatedAt` | DateTime | 기존 유지 |

**PAT 포맷**: `aurapat_{8-char base62 id}_{40-char base62 secret}`
- prefix는 tokenPrefix 필드에 저장, secret은 SHA-256 후 폐기
- GitHub 스타일 secret scanner 호환 접두 정규식 `^aurapat_[0-9a-zA-Z]{8}_[0-9a-zA-Z]{40}$`
- **1회 노출**: 발급 직후 모달에서 한 번만. DB 재조회 불가.

### 1.2 3-stage 마이그레이션 (D4 / R2)

SHA-256 단방향이라 기존 `tokenHash`에서 `tokenPrefix`를 역산 불가. 따라서 레거시는 일괄 revoke + 재발급 공지.

| Stage | 작업 |
|---|---|
| **1** | `tokenPrefix String? @unique` (nullable) 추가 + `label`·`lastUsedAt`·`scopeBoardIds String[]` 추가. 마이그 1회성. |
| **2** | 레거시 row 전체 `revokedAt=now()` + 교사별 이메일 공지 "PAT 재발급 필요" (Resend). 공지 발송 증거 기록. |
| **3** | `tokenPrefix String @unique` NOT NULL 전환. 남은 row가 모두 신규 포맷인지 검증 후 제약 추가. |

Stage 2와 3 사이에 **최소 7일 유예**. 각 stage는 별도 마이그레이션 PR(MG-A·MG-B·MG-C).

### 1.3 Tier 게이팅 (D11)

| 축 | Free | Pro |
|---|---|---|
| 토큰 **발급** UI 진입 | ✅ (설정 페이지 접근 가능) | ✅ |
| `cards:write` **scope 선택** 가능 | ❌ (UI 잠금 배지 "Pro 필요") | ✅ |
| `cards:write` 토큰 **수신** 시 200 | ❌ **402 Payment Required** + 업그레이드 링크 | ✅ |
| 수신 단계 재검증 | `user.tier !== "pro"` 이중 방어 (R7) |

Free 강등 우회 리스크(R7) 완화: 발급 시점 체크 + 매 수신 시점 `tokenOwner.tier` 재조회 → 발급 후 Free 강등 계정도 차단.

### 1.4 Rate Limit 3축 (D12)

Upstash Redis sliding window. 모든 축 OR 판정 — 하나라도 초과 시 429.

| 축 | 한도 | Redis key |
|---|---|---|
| per-token | 60 req / 1 min | `rl:pat:{tokenId}:1m` |
| per-teacher | 300 req / 1 hour | `rl:teacher:{teacherId}:1h` |
| per-IP | 300 req / 1 min | `rl:ip:{ipHash}:1m` |

429 응답 헤더 `Retry-After: <seconds>`. Upstash 장애 시 fail-open(R6) + `/api/external/healthz` 모니터링.

### 1.5 p95 < 2000ms — 스트리밍 업로드 (D13 / R1)

- `imageDataUrl` 파싱: base64 prefix 제거 후 `Readable.from()`로 chunk 스트림 생성 (전체 버퍼 X)
- `@vercel/blob`의 `put()` 스트리밍 호출 (`multipart: true` 옵션)
- 4.0MB 하드 가드: `Content-Length` 헤더 선검사 → 초과 시 본문 파싱 전 413 (Vercel 4.5MB 천장 대비 여유)
- aura-canva-app 측 프리-리사이즈는 **별도 task**로 파킹 (§9)

### 1.6 요청/응답 계약 (D14·D15·D16)

**Request — Zod strict**:
```ts
const bodySchema = z.object({
  boardId: z.string().cuid(),
  title: z.string().min(1).max(200),
  imageDataUrl: z.string().regex(/^data:image\/png;base64,/),
  sectionId: z.string().cuid().nullable().optional(),
}).strict();
```

- strict() → unknown 필드 거부 (`422 invalid_data_url`)
- `boardId`는 body 필수. 서버가 `scopeBoardIds` allowlist 검증 (빈 배열 = 교사 소유 전체 허용)
- `sectionId` 누락/null → 보드 기본 섹션 (freeform null)

**Response 200 OK** (201 아님 — 기존 계약):
```json
{ "id": "<Card.cuid>", "url": "https://aura-board-app.vercel.app/board/<slug>#c/<cardId>" }
```
- `imageUrl`·내부 필드·전체 card 객체 **응답 비포함**
- Canva 앱 Toast "Aura-board에 게시됨" + externalUrl 링크로 교사에게 결과 확인

**Error 포맷**:
```json
{ "error": { "code": "string", "message": "string" } }
```

### 1.7 에러 코드 표

| HTTP | code | 트리거 |
|---|---|---|
| 400 | `invalid_body` | JSON 파싱 실패 |
| 401 | `invalid_token` | 포맷 위반 / prefix 미존재 / hash 불일치 (timing-safe) |
| 402 | `tier_required` | Free 교사 토큰으로 `cards:write` 호출 (D11) |
| 403 | `forbidden_board` | body.boardId ∉ scopeBoardIds (빈 배열 제외) |
| 403 | `forbidden_scope` | 토큰 scopes에 `cards:write` 없음 |
| 404 | `board_not_found` | boardId는 있지만 DB 미존재 / revoked |
| 410 | `token_revoked` / `token_expired` | revokedAt·expiresAt 도달 |
| 413 | `payload_too_large` | Content-Length > 4.0MB |
| 422 | `invalid_data_url` | Zod strict 실패 (unknown key, 형식 오류 포함) |
| 429 | `rate_limited` | 3축 중 하나 초과 |
| 500 | `internal_error` | Blob 업로드 실패 등 서버 예외 |
| 503 | `rate_limit_unavailable` | Upstash 장애 시 fail-open이면 200, fail-close 옵션은 503 |

### 1.8 Card 기본값 (acceptance)

```ts
{
  width: 240,
  height: 160,
  content: "",
  authorId: token.teacherId,
  sectionId: body.sectionId ?? null,
  imageUrl: blobUrl, // 내부 저장, 응답 비공개
  kind: "image",
}
```

---

## 2. 신규·변경 파일 맵

| 경로 | 변경 유형 | 역할 |
|---|---|---|
| `prisma/schema.prisma` | 수정 | `ExternalAccessToken` 필드 추가 (tokenPrefix·label·lastUsedAt·scopeBoardIds) |
| `prisma/migrations/<ts>_add_token_prefix/` | 신규 | Stage 1 마이그레이션 |
| `prisma/migrations/<ts>_revoke_legacy_tokens/` | 신규 | Stage 2 데이터 마이그레이션 |
| `prisma/migrations/<ts>_token_prefix_not_null/` | 신규 | Stage 3 NOT NULL 전환 |
| `src/lib/external-auth.ts` | 신규 | PAT 생성·해싱(pepper)·검증(timing-safe)·resolve by prefix |
| `src/lib/rate-limit.ts` | 신규 또는 확장 | Upstash sliding window 3축 헬퍼 |
| `src/app/api/external/cards/route.ts` | 신규 | POST 핸들러. 인증→rate-limit→tier→Zod→boardId 권한→Blob 스트리밍→Card INSERT→응답 |
| `src/app/api/external/healthz/route.ts` | 신규 | Upstash·Blob·DB 라이브니스 |
| `src/app/api/tokens/route.ts` | 신규 | GET(list) · POST(issue) |
| `src/app/api/tokens/[id]/route.ts` | 신규 | DELETE(revoke) · PATCH(label 변경) |
| `src/app/(teacher)/settings/external-tokens/page.tsx` | 신규 | 갤탭 S6 Lite 최적화 교사 UI |
| `src/app/(teacher)/settings/external-tokens/TokenRevealModal.tsx` | 신규 | 1회 노출 모달 (Copy + Download .txt) |
| `src/components/TierLockBadge.tsx` | 신규 또는 재사용 | "Pro 필요" 배지 (Seed 2 공통) |
| `next.config.ts` | 수정 | `/api/external/*` body size config (4.5MB default 유지 + 라우트별 4MB 가드) |
| `.env.example` | 수정 | `PAT_PEPPER`, `UPSTASH_REDIS_REST_URL/TOKEN`, `BLOB_READ_WRITE_TOKEN` |

---

## 3. 교사 UI — `/(teacher)/settings/external-tokens` (갤탭 S6 Lite)

### 3.1 목록 페이지

| 컬럼 | 내용 |
|---|---|
| label | 교사가 지정한 이름 |
| prefix | `aurapat_xxxxxxxx_••••…` (secret 마스킹) |
| scopes | 배지 (v1은 `cards:write` 단일, Free는 잠금 아이콘) |
| scopeBoardIds | "모든 보드" 또는 N개 칩 |
| expiresAt | "2026-07-11 · D-90" 형식 |
| lastUsedAt | "3시간 전" 또는 "사용 이력 없음" |
| 액션 | [Revoke] 버튼 (즉시 `revokedAt=now()`) |

**레이아웃**: 세로 리스트. 터치 타겟 ≥ 44px. 우측 sticky "+ 새 토큰 발급" FAB.

### 3.2 발급 모달

| 입력 | UX |
|---|---|
| Label | 필수. 1~40자 |
| Scope | v1 고정 `cards:write`. Free 계정은 잠금 배지 |
| 보드 범위 | 라디오: ○ "내 모든 보드" (scopeBoardIds=[]) / ○ "특정 보드" → 다중 선택 체크박스(교사 소유 보드만) |
| 유효기간 | 드롭다운: 1일(테스트) / 30일 / **90일 (기본, 권장)** / 365일 / 무기한. "권장: 90일 회전" 주석 |

### 3.3 토큰 공개 모달 (1회 only · D3)

- 가운데 큰 monospace 블록에 `aurapat_…` 표시
- **[복사] 버튼** (navigator.clipboard)
- **[다운로드 .txt] 버튼** — 파일명 `aura-token-{label}-{YYYYMMDD}.txt`, 본문에 경고 + 토큰 + 발급 정보
- 하단 경고 배너: "이 창을 닫으면 토큰을 다시 볼 수 없습니다. 잃어버리면 재발급만 가능합니다."
- 닫기 시 확인 팝업 "복사하셨나요?"

### 3.4 Tier 잠금 UX (Free 계정)

- 목록 상단에 "Pro 업그레이드하면 Canva 연동 가능" 배너 + CTA
- 발급 모달에서 `cards:write` scope 체크박스 disabled + 잠금 배지
- Free 계정은 토큰을 발급해도 수신 단계에서 402 (이중 방어)

---

## 4. 작업 분할 (CR-1 ~ CR-10)

각 작업은 padlet `feature` 파이프라인에 `tasks/{YYYY-MM-DD-slug}/phase0/request.json`으로 별도 등재. phase0-requests.md의 **"Canva Publisher 수신"** 섹션 참조.

### CR-1 — 스키마 & 마이그 Stage 1 (nullable prefix)
- `prisma/schema.prisma` 필드 추가 (`tokenPrefix String? @unique` 등)
- `prisma migrate` 생성
- 수용: CI 통과, 기존 row 영향 없음

### CR-2 — `src/lib/external-auth.ts` (PAT 생성·검증)
- `generatePat()`: base62 id(8) + secret(40) 생성, `PAT_PEPPER` 환경변수 사용
- `hashPat(secret)`: `sha256(secret ‖ PEPPER)`
- `resolvePatByPrefix(prefix)`: O(1) DB lookup + timingSafeEqual 더미 해시 (R5)
- 수용: unit test — 유효/무효/존재하지 않는 prefix 모두 일정 시간

### CR-3 — `POST /api/external/cards` (핵심 엔드포인트)
- 13단계 체인: Content-Length 4MB 가드 → PAT 파싱 → prefix resolve → hash 검증 → scope 체크 → tier 재검증 → 3축 rate-limit → Zod strict → boardId allowlist → Blob 스트리밍 put → Card INSERT → lastUsedAt 갱신 → 200 응답
- 모든 분기에 에러 코드 (§1.7)
- 수용: 통합 테스트 — 유효/각 에러/p95 < 2000ms

### CR-4 — Upstash Redis 3축 rate limit (`src/lib/rate-limit.ts`)
- sliding window — `@upstash/ratelimit` 사용
- fail-open on 장애 + healthz 라인
- 수용: 60·300·300 한도 초과 시 429 + Retry-After

### CR-5 — Vercel Blob 스트리밍 업로드
- `imageDataUrl` → base64 decode stream → `put(key, stream, { multipart: true })`
- key 규약: `external-cards/{boardId}/{cardId}.png`
- 수용: 3MB 업로드 p95 < 2000ms (phase3 D13)

### CR-6 — `GET/POST /api/tokens` & `PATCH/DELETE /api/tokens/[id]`
- teacher 인증 필수 (NextAuth 세션)
- POST 응답에 **1회만** secret 포함, 이후 DB에서 조회 불가
- DELETE는 soft revoke (`revokedAt=now()`)
- 수용: 타 교사 토큰 조회 → 404

### CR-7 — 교사 UI `/(teacher)/settings/external-tokens/page.tsx`
- 목록 + FAB + 발급 모달 + 공개 모달
- 갤탭 S6 Lite 1200×800 포트레이트/가로 모두 대응
- Tier 잠금 배지
- 수용: Playwright 터치 흐름 · 모달 1회 노출 · Copy/Download 동작

### CR-8 — 마이그 Stage 2·3 (legacy revoke + NOT NULL)
- Stage 2: 레거시 row `revokedAt=now()` + Resend로 교사 이메일 발송 ("PAT 재발급 필요")
- Stage 3: 7일 유예 후 `tokenPrefix String @unique` NOT NULL 전환
- 수용: 운영 DB 검증 완료, 미발송 교사 0명

### CR-9 — `/api/external/healthz` 모니터
- Upstash ping + Blob head + DB `SELECT 1`
- Vercel cron 1분 체크 + 실패 시 알림
- 수용: 각 의존성 down 시 503

### CR-10 — E2E 보안 게이트 (phase9 QA 필수)
- 1) 무효 PAT 401 · 2) Free 토큰 402 · 3) boardId 범위 위반 403 · 4) 4MB 초과 413 · 5) 3축 rate-limit 429 · 6) p95 < 2000ms · 7) 1회 모달 재표시 불가 · 8) secret scanner 정규식 매칭 · 9) prefix miss timing 일정 · 10) Upstash 장애 fail-open
- 수용: 10개 시나리오 통과

---

## 5. 수용 기준 (seed.yaml acceptance_criteria)

- [ ] POST /api/external/cards가 Zod strict body 검증 (boardId cuid / title 1–200 / imageDataUrl data:image/png;base64 / sectionId? cuid | null)
- [ ] 성공 시 200 `{ id, url: "https://aura-board-app.vercel.app/board/<slug>#c/<cardId>" }`
- [ ] 모든 에러는 `{ error: { code, message } }` 포맷
- [ ] PAT는 tokenPrefix(8자) DB 조회 후 SHA-256(secret‖PEPPER) 검증
- [ ] boardId가 token.scopeBoardIds allowlist 통과(빈 배열 = 교사 전체 허용)
- [ ] Vercel Blob 스트리밍 업로드, p95 < 2000ms
- [ ] rate limit 429 + Retry-After
- [ ] Free 토큰 `cards:write` 호출 → 402 + upgrade link
- [ ] Content-Length > 4.0MB → body parse 이전 413
- [ ] `/(teacher)/settings/external-tokens` — 갤탭 S6 Lite 최적화
- [ ] 발급 1회 모달 — Copy + Download(.txt)
- [ ] 유효기간 드롭다운 1/30/90(기본)/365/무기한 + 90일 회전 권장 주석
- [ ] 3-stage 마이그 (nullable → revoke+공지 → NOT NULL) 운영 DB에서 성공
- [ ] Card 기본값: width=240, height=160, content="", authorId=tokenOwner.id, sectionId=body.sectionId ?? null
- [ ] 알 수 없는 필드 → 422 invalid_data_url

---

## 6. 리스크 & 완화

| # | 리스크 | 완화 |
|---|---|---|
| R1 | p95 업로드 예산 초과 (3MB) | **스트리밍** put + 4MB 하드 가드 + aura-canva-app 프리-리사이즈 task(§9) |
| R2 | 레거시 tokenPrefix NULL | 3-stage 마이그 (CR-1 · CR-8) + 교사 이메일 공지 |
| R3 | Vercel Blob 4.5MB 초과 | 4.0MB 조기 413 + Canva UI 재-Export 가이드 (클라이언트 토스트) |
| R4 | PAT 평문 유출 | 1회 모달 + prefix secret scanner 호환 + 즉시 revoke UI |
| R5 | timing side-channel | prefix miss 시 **더미 hash** timingSafeEqual + per-IP rate limit |
| R6 | Upstash 장애 | **fail-open** 기본 + healthz 모니터 + 옵션 `RL_FAIL_MODE=close` 환경변수 |
| R7 | Free 강등 우회 | 발급 시점 + 수신 시점 이중 tier 재검증 |
| R8 | Canva 송신측 이슈 | 에러 응답에 원인 코드 명시 — Canva Toast에 노출 |
| R9 | Tier 훅 Seed 2 구현 전 | v1 임시 허용 + "베타: Pro 제한 추후" 배너 (Seed 2 도입 후 자동 활성) |

---

## 7. 의존 그래프

```
[Seed 2 Tier 매트릭스] ────┐
                         ├─→ Tier 게이팅 (D11) — Free 402 / Pro 발급 가능
[aura-canva-app 클라이언트] ─→ Request/Response 계약 고정 (D14·D15·D16)
                         │
[tablet-performance-roadmap §2 예산] ─→ 갤탭 S6 Lite 교사 UI 터치 타겟·TTI
                         │
[Vercel Blob 저장소] ──────┴─→ 스트리밍 put + 4MB 가드

[본 로드맵 CR-1~CR-10]
       │
       ├─→ (미래) Seed 5 P2-⑤ Webhook receive — `scopes: webhooks:receive` 확장 지점 공유
       ├─→ (미래) 다른 외부 앱(Slack/Miro) — `metadata JSON` 일반화 (D8)
       └─→ (미래) Canva 앱 deeplink 조사 task (D5 후속)
```

---

## 8. 학부모 열람 범위

Canva Publisher로 생성된 카드는 **정적 PNG 카드**. Seed 7 `parent-viewer-roadmap.md` §5 매트릭스상 일반 `Card` 범주로 편입된다.

- 학부모는 자녀가 속한 보드·섹션의 카드만 열람 (server filter)
- Publisher 출처라는 메타는 parent viewer에 노출 불필요
- **별도 구현 작업 없음** — Seed 7 PV-7(자녀 범위 서버 필터)이 `Section → Card` 경로로 자연 처리

---

## 9. 파킹 (v2+)

| # | 파킹 항목 | 근거 |
|---|---|---|
| 1 | aura-canva-app 프리-리사이즈 강제 task | R1 완화 b조건 — 별도 리포 task. 3MB 초과 시 export 전 해상도 스케일다운 |
| 2 | Canva Apps 교사용 deeplink 스펙 조사 | D5 후속 — `https://www.canva.com/apps/intent/...` 형식 v1.1 적용 |
| 3 | Card 삭제 시 Vercel Blob 정리 Cron | sketch §6-D — onDelete cascade와 물리 삭제 동기화. padlet 리포 별도 task |
| 4 | Webhook 수신 `scopes: webhooks:receive` | D1 v2 확장 — Seed 5 P2-⑤ 계승 |
| 5 | `metadata JSON` 필드 일반화 | D8 — Slack/Miro 등 다른 외부 앱 합류 시점에 스키마 확정 |
| 6 | CRC32 checksum (`aurapatc_` 신포맷) | D7 — v1.1 — 구현 0.5일이지만 v1 복잡도 회피 |
| 7 | sectionId UI — Canva 앱 section 드롭다운 | D6 — v1은 API만 수용, 클라이언트 UI는 v2 |
| 8 | Enterprise v2+ S3 마이그레이션 | D9 — Vercel Blob 한계 도달 시 |
| 9 | 학교 일괄 결제·세금계산서 | Seed 2 연계 Enterprise v2+ |
| 10 | Token rotation 자동화 (90일 만료 7일 전 알림) | D2 연계 — v1.1 |

---

## 10. 변경 로그

| 날짜 | 시드 ID | 변경 |
|---|---|---|
| 2026-04-12 | `seed_26af361e92b7` | 초안 생성 — ExternalAccessToken 확장(prefix·label·lastUsedAt·scopeBoardIds), `/api/external/cards` + `/api/tokens` CRUD + 교사 UI + 3-stage 마이그 + Tier 게이팅(Free 402) + 3축 rate limit + Blob 스트리밍 업로드 + 1회 노출 모달(Copy+Download) 확정. CR-1~CR-10 작업 분할. |
