# Phase 2 Sketch — Canva Publisher Receiver (`/api/external/cards` + PAT)

> task_id: `2026-04-12-canva-publisher-receiver`
> 작성: 2026-04-12 · sketch-architect 에이전트
> 입력: phase0/request.json · phase1/exploration.md (§5 1순위 권고) · padlet/prisma/schema.prisma (읽기 전용) · aura-canva-app/src/intents/content_publisher/index.tsx

---

## 0. 전제 고정 (phase1 1순위 권고)

phase1 §5의 **1순위 아키텍처 조합을 본 스케치의 불변 전제**로 고정한다.

| 축 | 결정 | 근거 (phase1) |
|---|---|---|
| PAT 저장 | **SHA-256(secret‖PEPPER) + prefix lookup** | §1-A, §1-B — 요청당 <1ms (태블릿 p95 예산), GitHub/GitLab 업계 실채용 |
| 토큰 포맷 | `aurapat_{8자 tokenId base62}_{40자 secret base62}` | §1-B — prefix로 row 1개 조회 → timingSafeEqual |
| 평문 노출 | 생성 직후 **모달 1회만, 영구 재표시 불가** | §1-B, §4-B — 업계 공통, 감사 가능성 |
| 이미지 경로 | Canva가 송신한 data URL → 서버 decode → Buffer → `@vercel/blob.put()` | §2, §2-A — Canva 스펙 고정, presigned 불가 |
| 스토리지 | **Vercel Blob** (MVP) | §2-B — Aura 환경변수 자동 주입, `@vercel/blob` SDK 한 줄, Prisma 확장만으로 배포 |
| Runtime | Next.js App Router Route Handler, `export const runtime = 'nodejs'` | §2-A (3) — Buffer/Blob 처리, Edge는 2MB 페이로드로 부적합 |
| Body size | content-length 조기 413 가드 (4.0MB 컷; Vercel 4.5MB 하드 리밋의 안전 마진) | §2-A (2) — App Router는 bodyParser 설정 불가 |
| Rate limit | **Upstash `@upstash/ratelimit` sliding window** — 토큰/교사/IP 3축 | §3 — Edge/Node 양립, per-identifier |
| Tier 게이팅 | Content Publisher 수신은 Pro 전용 (Seed 2 승계). Free 교사는 **토큰 생성 단계**에서 disabled + 업그레이드 CTA, 수신 단계에서는 `403 tier_locked` 이중 방어 | Seed 2 `seed_8967b77d7759`, phase1 §4-B |
| UI 기준 단말 | 갤럭시 탭 S6 Lite 10.4", Chrome Android, `navigator.clipboard.writeText()` | tablet-performance-roadmap.md §0, phase1 §4-A |

**phase1 권고 중 본 task 스코프에 포함되지 않는 항목**:
- CRC32 checksum (phase1 §1-B 🟡 "1차 출시 후 옵션") — **v1 미포함**, 미결 질문 Q3에서 재검토
- Edge runtime 이전 — Node 고정, v1 스코프 외
- HMAC-peppered 변형은 PEPPER 환경변수 주입만으로 달성 (코드는 `hash = SHA-256(secret‖PEPPER)`)

---

## 1. Prisma 엔티티 — 기존 재사용 + 확장

### 1-A. 중요 발견: `ExternalAccessToken`이 이미 존재

padlet/prisma/schema.prisma **38~49행에 `ExternalAccessToken` 모델이 이미 정의됨**.

```prisma
model ExternalAccessToken {
  id         String    @id @default(cuid())
  userId     String
  name       String
  tokenHash  String    @unique
  lastUsedAt DateTime?
  revokedAt  DateTime?
  createdAt  DateTime  @default(now())
  user       User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@index([userId])
}
```

또한 `User.externalTokens ExternalAccessToken[]` 역관계도 선언되어 있음 (34행).

**결론**: phase1이 전제한 "신규 `PersonalAccessToken` 엔티티"는 만들지 **않는다**. 대신 **기존 `ExternalAccessToken`을 확장**해 prefix-lookup·scope·expiry를 추가하고, 모델명은 기존 이름을 유지한다.

이유:
1. 기존 Aura-board 스키마 단일 출처(single source of truth) 보존
2. User 관계·인덱스 재사용
3. 마이그레이션은 additive만 — 기존 데이터(있다면) 파괴 없음

### 1-B. 확장 제안 (additive 마이그레이션)

```prisma
model ExternalAccessToken {
  id            String    @id @default(cuid())
  userId        String
  name          String
  tokenHash     String    @unique              // 기존 유지. 저장값 = sha256(secret || PEPPER) hex
  lastUsedAt    DateTime?
  revokedAt     DateTime?
  createdAt     DateTime  @default(now())

  // ── 신규 ────────────────────────────────────────────────
  tokenPrefix   String    @unique              // 8자 base62, plaintext, lookup 키
  scopeBoardIds String[]                       // 빈 배열 = 교사 소유 전체 보드
  scopes        String[]  @default(["cards:write"]) // v1 단일 scope (미결 Q1)
  expiresAt     DateTime?                      // 기본 null = 무기한 (미결 Q2)
  lastUsedIp    String?                        // 감사용 (선택)

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([tokenPrefix])                       // lookup O(1)
  @@index([revokedAt])                         // 활성 토큰 필터
}
```

**마이그레이션 전략**:
- 기존 row(있다면)는 `tokenPrefix`가 NULL → 마이그레이션 시 `tokenPrefix`를 기존 `tokenHash`의 첫 8자로 backfill하거나, v1 출시 시 레거시 row 일괄 revoke 후 재발급 안내. (미결 질문 Q4 후보)

### 1-C. 기존 엔티티 재사용 매핑

| 신규 요구 | 기존 엔티티 재사용 |
|---|---|
| 토큰 발급 주체 | `User` (교사) — id/email/memberships |
| 토큰 권한 검증 | `BoardMember` (role: owner/editor/viewer) — 보드 소유 확인 |
| 카드 생성 | `Card` (imageUrl·title·authorId·boardId·sectionId?) — 신규 필드 추가 **불필요** |
| 섹션 타겟 | `Section` — Canva는 모름. 기본 sectionId=null (보드 최상단) |
| Classroom 스코프 | `Classroom` — 교사 소유 보드는 `classroomId` 경유로 조회 가능 |
| Tier 게이팅 | `User` + Seed 2 tier 저장소(별도 문서) — 본 task는 **게이팅 훅 포인트만** 정의, 실제 tier 스키마는 Seed 2 구현 때 합류 |

**따라서 신규 엔티티는 0개 (기존 `ExternalAccessToken` 확장만)**. 이는 sketch-architect contract의 "기존 Board/Card/Section/Classroom 재사용 우선" 기준을 통과한다.

---

## 2. API 계약 초안

### 2-A. `POST /api/external/cards` (Canva 호출 수신)

**계약은 Canva 클라이언트 코드(읽기 전용)로 고정**. `aura-canva-app/src/intents/content_publisher/index.tsx` line 99-110 참조.

```
POST /api/external/cards
Host: aura-board-app.vercel.app
Authorization: Bearer aurapat_<8id>_<40secret>
Content-Type: application/json
Content-Length: <4.0MB

Body:
{
  "boardId": "<cuid>",
  "title":   "<string, 1~200자>",
  "imageDataUrl": "data:image/png;base64,<base64>"
}

200 OK:
{
  "id":  "<Card.id cuid>",
  "url": "<절대 URL: https://aura-board-app.vercel.app/board/<slug>#c/<cardId>>"
}
```

**처리 파이프라인** (phase1 §5 그대로):

```
1. content-length > 4_000_000 → 413 payload_too_large (early reject)
2. Authorization 헤더 파싱 → `aurapat_` prefix 확인 → 401 invalid_token 실패 시
3. prefix split → ExternalAccessToken.findUnique({ tokenPrefix })
   - row 없음 → 401 invalid_token (timingSafeEqual dummy hash로 side-channel 차단)
   - revokedAt NOT NULL → 401 token_revoked
   - expiresAt < now → 401 token_expired
4. sha256(secret || PEPPER) 계산 → timingSafeEqual(row.tokenHash) → 실패 401 invalid_token
5. Upstash rate limit 3축:
   - perToken:   60 req / 60s  (phase1 §3)
   - perTeacher: 200 req / 600s
   - perIp:      300 req / 60s
   - 초과 → 429 rate_limited + Retry-After
6. boardId 권한: BoardMember where userId=tokenOwner and boardId=body.boardId
   - 없음 → 403 forbidden_board
   - scopeBoardIds 비어있지 않으면 boardId ∈ scopeBoardIds 재확인
7. Tier 게이팅: User.tier !== 'pro' → 403 tier_locked (Seed 2 훅)
8. zod schema: body.title 1-200자, imageDataUrl starts with "data:image/png;base64,"
   - 실패 → 422 invalid_data_url
9. imageDataUrl → strip prefix → Buffer.from(base64, 'base64')
10. @vercel/blob.put(`boards/${boardId}/cards/${cuid}.png`, buffer, {
      access: 'public', contentType: 'image/png'
    }) → { url }
11. prisma.card.create({
      data: { boardId, authorId: tokenOwner.id, title, imageUrl: blob.url,
              sectionId: null, content: "", width: 240, height: 160 }
    })
12. ExternalAccessToken.update({ lastUsedAt: now, lastUsedIp: hashed })
13. return { id: card.id, url: `${HOST}/board/${board.slug}#c/${card.id}` }
```

**에러 코드 테이블** (Canva UI 매핑 — phase1 §4-C 그대로 승계):

| HTTP | code | 원인 |
|---|---|---|
| 401 | invalid_token | 포맷 오류·prefix 미매치·hash 불일치 |
| 401 | token_revoked | soft-deleted |
| 401 | token_expired | expiresAt 경과 |
| 403 | forbidden_board | boardId 소유 아님 or scope 위배 |
| 403 | tier_locked | Free 교사 |
| 413 | payload_too_large | content-length > 4MB |
| 422 | invalid_data_url | zod 실패 / base64 파싱 실패 |
| 429 | rate_limited | 토큰·교사·IP 중 하나 초과 |
| 500 | internal | Blob put 실패 등 |

### 2-B. `POST /api/tokens` (PAT 발급 — 교사)

```
POST /api/tokens
Auth: NextAuth session (교사)

Body:
{
  "name":          "<string, 1~60자>",          // 라벨 e.g. "갤탭 3반용"
  "scopeBoardIds": ["<boardId>", ...] | [],    // 빈 배열 = 전체 허용
  "expiresInDays": 30 | 90 | 365 | null        // null = 무기한 (미결 Q2)
}

201 Created:
{
  "id":        "<ExternalAccessToken.id>",
  "prefix":    "aurapat_<8id>",                // 목록 표시용
  "rawToken":  "aurapat_<8id>_<40secret>",    // ⚠️ 이 응답에서만 반환. 재조회 불가
  "name":      "...",
  "scopeBoardIds": [...],
  "expiresAt": "<ISO>|null",
  "createdAt": "<ISO>"
}
```

**구현 요지**:
- `tokenId` = crypto.randomBytes → base62 8자
- `secret` = crypto.randomBytes(30) → base62 40자 (≈238bit)
- `tokenHash` = sha256(secret || PEPPER) hex 64자
- `tokenPrefix` = tokenId (plaintext, DB unique)
- Tier 게이팅: Free → **400 tier_locked** (UI는 버튼 disabled 1차, API 이중 방어)

### 2-C. `GET /api/tokens` (목록)

```
GET /api/tokens
Auth: NextAuth session

200:
{
  "tokens": [
    {
      "id": "...", "prefix": "aurapat_ab12cd34", "name": "갤탭 3반용",
      "scopeBoardIds": [...], "expiresAt": "...",
      "lastUsedAt": "...", "createdAt": "...", "revokedAt": null
    },
    ...
  ]
}
```
**rawToken·tokenHash는 절대 응답 포함 금지**.

### 2-D. `DELETE /api/tokens/:id` (철회)

```
DELETE /api/tokens/:id
Auth: NextAuth session (소유자 일치 검증)

204 No Content
```
- hard delete 금지. `revokedAt = now` soft delete (감사 로그 보존). phase1 §1-C 근거.

---

## 3. UI 스케치 (갤럭시 탭 S6 Lite 친화)

### 3-A. 경로

`/(teacher)/settings/external-tokens` — 교사 설정 메뉴 하위. NextAuth session 필수.

### 3-B. 목록 화면 (기본 진입)

```
┌─────────────────────────────────────────────────────────────┐
│  외부 앱 연결 (PAT)               [＋ 새 토큰 발급] (48×48) │
├─────────────────────────────────────────────────────────────┤
│  💡 Canva Content Publisher 앱에서 붙여넣을 키를 발급합니다. │
│     분실 시 재발급이 필요합니다(다시 볼 수 없음).             │
├─────────────────────────────────────────────────────────────┤
│  🟢 갤탭 3반용                                                │
│     aurapat_ab12cd34…                                         │
│     범위: 3반 수업 보드·발표 보드    만료: 2026-07-11         │
│     마지막 사용: 2시간 전                    [🗑 철회]          │
│  ─────────────────────────────────────────────────────────── │
│  ⚪ 데스크톱 테스트 (2026-03-10 철회됨)                        │
│     aurapat_ef56gh78…                                         │
│     만료: —    마지막 사용: 3주 전                              │
└─────────────────────────────────────────────────────────────┘
```

- Revoke 버튼 = 44×44pt (Android tap target 권장). 탭 → 확인 다이얼로그 "다시 되돌릴 수 없습니다" → 204.
- monospace로 prefix만 표시. 전체 토큰은 이미 노출 불가.

### 3-C. 생성 모달

```
┌─ 새 외부 토큰 발급 ─────────────────────────────────────────┐
│                                                              │
│  이름(라벨)  [갤탭 3반용                              ]      │
│              (예: "3반 Canva 연동", "내 데스크톱")          │
│                                                              │
│  범위       ◉ 내 모든 보드                                    │
│             ○ 선택한 보드만                                   │
│               □ 3-1반 메인 보드                              │
│               □ 3-1반 발표 보드                              │
│               □ 방과후 실험 보드                              │
│                                                              │
│  유효기간    [▼ 90일 (2026-07-11)        ]                   │
│              • 30일 • 90일(기본) • 1년 • 무기한              │
│                                                              │
│  ⚠️  발급된 키는 즉시 다음 화면에서 한 번만 표시되며,         │
│     그 후에는 어떤 방법으로도 다시 볼 수 없습니다.            │
│     복사해 Canva에 바로 붙여넣으세요.                         │
│                                                              │
│              [  취소  ]    [  발급하기  ] (56×200)           │
└──────────────────────────────────────────────────────────────┘
```

- Free 교사: "발급하기" 버튼 disabled + 상단 배너 "Pro 플랜에서 사용 가능 [업그레이드]"

### 3-D. 발급 직후 1회 노출 모달

```
┌─ 토큰 발급 완료 ────────────────────────────────────────────┐
│                                                              │
│  ✅ "갤탭 3반용" 토큰이 발급되었습니다.                       │
│                                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ aurapat_ab12cd34_xXyZ...(40자)                     │     │
│  │                                       [📋 복사] (48×48)  │
│  └────────────────────────────────────────────────────┘     │
│                                                              │
│  ⚠️  이 창을 닫으면 다시 볼 수 없습니다.                     │
│     지금 복사해서 Canva 앱 "API 키" 입력란에 붙여넣으세요.   │
│                                                              │
│  [ 📱 Canva 앱으로 이동 ]   [  완료  ]                       │
└──────────────────────────────────────────────────────────────┘
```

- 복사 버튼 탭 → `navigator.clipboard.writeText(rawToken)` → tooltip "복사됨!" 1.5s
- monospace·12자 단위 시각적 띄어쓰기(실제 복사값 공백 無) — phase1 §4-B
- "Canva 앱으로 이동" = Canva Apps 딥링크(구체 URL은 Canva 앱 프로젝트 확인 — 미결 Q5)
- 닫기 확인 다이얼로그: "토큰을 복사하셨나요? 지금 닫으면 다시 볼 수 없습니다."

---

## 4. 역할별 사용자 흐름

### 4-A. 교사 흐름 (PAT 발급·사용)

```
교사 (갤럭시 탭 S6 Lite, 수업 전 5분)
  │
  ├─(1) Aura-board 로그인 → 설정 → 외부 앱 연결
  ├─(2) [새 토큰 발급] → 라벨·범위·유효기간 입력 → [발급하기]
  ├─(3) 1회 모달에서 🟢 [복사] 버튼 탭 (44×44 target)
  │     → clipboard.writeText(rawToken), tooltip "복사됨!"
  ├─(4) [Canva 앱으로 이동] 딥링크 → Canva "내 앱" → Publisher Intent 설정
  ├─(5) Canva 앱 설정 화면에 PAT 붙여넣기 + boardId 드롭다운 선택 + 제목 입력
  ├─(6) Canva 에디터에서 수업자료 디자인 → [Publish] 클릭
  │     → Canva가 `POST /api/external/cards` 호출
  ├─(7) 반응: Canva UI Toast "Aura-board에 게시됨" + externalUrl 링크 표시
  └─(8) 교사가 externalUrl 클릭 → Aura-board 보드 열람 (카드 확인)

오류 경로(예시):
  (6)에서 Content-Length 5MB → 413 → Canva UI "이미지가 너무 큽니다"
  (6)에서 Free 교사 → 403 tier_locked → Canva UI "Pro 플랜 기능입니다"
  (6)에서 토큰 revoke 후 사용 → 401 token_revoked → Canva UI "폐기된 키"
```

### 4-B. 학생 흐름 (수동적 — v1 범위 밖이지만 영향 표시)

```
학생 (태블릿, 수업 중)
  │
  └─ 교사가 Canva에서 발행한 카드를 Aura-board에서 열람 (이미 존재하는 Card 뷰)
     v1에서 학생이 Canva Publisher를 직접 사용하지 않음 — Seed 5 결정 승계
```

### 4-C. 학부모 흐름 (Seed 7 범위 밖)

```
학부모가 Canva 발행 카드를 보는지? Seed 7 자녀 범위 매트릭스에 Card는 명시 없음.
본 task는 Parent Viewer에 카드 노출 로직을 건드리지 않음. 기존 board-scope 규칙 그대로.
```

---

## 5. 태블릿 성능 체크리스트 (tablet-performance-roadmap.md §2 예산)

본 기능은 **교사 설정 UI + 서버 엔드포인트** 조합. 학생 동시접속 스파이크 영향은 간접(Card 증가). 직접 영향 축:

| 예산 (tablet-performance-roadmap.md §2) | 본 기능 영향 | 검증 방법 |
|---|---|---|
| 초기 TTI < 3s (30카드 보드) | 🟢 영향 없음 (설정 페이지는 보드 무관) | — |
| 드래그 60fps | 🟢 무관 | — |
| **설정 페이지 TTI < 2s 갤탭** | 🟡 목록 쿼리 O(n) — n=교사당 토큰 수, 보통 1~5개 | Prisma findMany + `@@index([userId, revokedAt])` |
| **PAT 발급 API p95 < 500ms** | 🟡 sha256 + randomBytes + Blob put 없음 → 가볍다. rate limit 체크 포함해도 여유 | Upstash REST latency 측정 |
| **`/api/external/cards` p95 < 2000ms** (이미지 업로드 포함) | 🔴 Blob put 네트워크 왕복 + base64 decode. 3MB 기준 Vercel Node 함수 2~4s 관측 사례. 예산 초과 가능 | 로드 테스트: 30 동시 req × 3MB PNG |
| 메모리 < 500MB/1h | 🟡 Buffer는 요청 스코프 즉시 해제. 누수 위험은 Blob SDK 재사용 클라이언트 캐시 정도 | Vercel function memory graph |
| iframe 동시 ≤ 3 | 🟢 무관 | — |
| WebSocket 채널 분리 | 🟢 무관 (본 기능은 REST only) | — |
| **이미지 원본 노출 금지** (T0-④) | 🟡 Blob은 public URL. 공개 CDN이지만 예측 불가 경로(`boards/<cuid>/cards/<cuid>.png`). 그래도 `next/image`로 래핑해 Vercel Image Optimization 경유 필수 | Card 렌더 시 `<Image src={imageUrl}>` 강제 |
| 3G에서 썸네일 < 500KB | 🟡 저장된 PNG는 원본(최대 3MB). 뷰어 쪽은 Vercel Image Optimization width/quality 자동 | 보드 렌더 검증 |

**고위험 결론**: `POST /api/external/cards` p95 예산이 유일한 적색 항목. 완화책은 §7 리스크에 기재.

---

## 6. Canva/Tier/기존 기능 시너지

### 6-A. Seed 5 (Canva 통합 재검토)
- Content Publisher v1 = 교사 주체 프라이빗 앱 결정 **직접 구현**. 서버 측 착륙 = 본 task.
- 미래 P2-⑤ Canva Webhook 수신 엔드포인트와 `/api/external/*` 네임스페이스 공유. 토큰 시스템 재사용 가능 (scope 확장: `webhooks:receive`).

### 6-B. Seed 2 (Tier)
- Content Publisher 수신 = **Pro 전용** (phase0 known_constraints + phase1 §4-B).
- 본 task는 tier 검증을 "게이팅 훅 포인트"로만 남김: `user.tier === 'pro'` 체크 함수 호출. Seed 2 실제 스키마(예: `User.tier`, `Subscription`)는 별도 구현. 훅 미정의 시 v1은 전 교사 허용으로 임시 배포하고, Seed 2 머지 시점에 연결.

### 6-C. 기존 기능
- `Card` 엔티티 재사용 — 필드 추가 없음. `authorId = tokenOwner.id` (교사) → 기존 author 표시 UI 그대로.
- `Board` slug → externalUrl 생성. 기존 `/board/[slug]` 라우트 재사용.
- `next.config.ts` `images.remotePatterns`에 Vercel Blob 도메인 추가 필요 (tablet-performance-roadmap.md T0-④ 지침 연속).
- `BoardMember`(role≥editor) 검증 — 기존 RBAC lib 재사용.

### 6-D. T0-④ 이미지 파이프라인
- Blob 저장 후 Card.imageUrl은 CDN URL. 뷰어 측은 `next/image` 필수(원본 노출 금지 원칙).
- `boards/{boardId}/cards/{cardId}.png` 키 규약 = onDelete cascade와 맞물려 향후 카드 삭제 시 Blob 정리 Cron(별도 task) 근거가 됨.

---

## 7. 리스크 표

| # | 리스크 | 영향 | 완화 |
|---|---|---|---|
| R1 | p95 예산 초과 (3MB PNG 업로드 → Blob put 2~4s) | 🔴 수업 중 교사 Canva UI 대기 시간 증가 → 불만 | (a) Canva 앱 쪽 프리-리사이즈 강제 (aura-canva-app에 별도 PR) (b) sharp로 서버측 다운사이즈 후 저장 (MVP 이후) (c) Blob put 결과 대기 중 Canva 쪽 timeout 설정 확인 |
| R2 | 레거시 `ExternalAccessToken` row (schema에 이미 선언됨)에 `tokenPrefix`가 NULL | 🟡 마이그레이션 실패 / 기존 토큰 즉시 무효화 | (a) 프로덕션에 해당 row 존재 여부 선확인 (phase3 확인) (b) 없으면 NOT NULL로 즉시 추가 (c) 있으면 v1 출시 시 일괄 revoke 공지 + 재발급 안내 |
| R3 | Vercel Blob 4.5MB 하드 리밋 초과 시 리커버리 불가 | 🟡 Canva Export 해상도가 서버 예상보다 큰 경우 | 조기 413 + Canva UI 메시지 + 재-Export 가이드. 송신측 프리-리사이즈는 aura-canva-app task |
| R4 | PAT 평문 유출(스크린샷·로그) | 🔴 타인 교실에 카드 쏟아짐 | (a) 로그에 `Authorization` 헤더 마스킹 필수 (b) prefix 포맷으로 secret scanner 탐지 가능 (c) 교사에게 1회 노출 경고 명확 (d) 즉시 revoke UI |
| R5 | timing side-channel로 유효한 prefix 탐지 | 🟡 이론적 — 대량 rand prefix 시도로 row 존재 탐지 | (a) prefix miss 시에도 dummy hash timingSafeEqual 수행 (b) per-IP rate limit (c) prefix 공간 62^8 ≈ 2e14 충분 |
| R6 | Upstash Redis 장애 시 rate limit fail-open vs fail-close | 🟡 장애 중 무제한 호출 또는 전면 차단 | fail-open 권고 (phase1 §3 묵시) + Upstash healthz 모니터 (장애 시 Slack 알림). 의도: 교실 수업 중단보다 악용 한시적 감내 |
| R7 | Free 교사가 Pro 게이팅 우회(토큰 발급 후 tier 강등) | 🟡 revoke 누락 시 수신 계속 동작 | 수신 단계에서도 user.tier 재검증(이중 방어). tier 강등 cron이 관련 토큰 revoke까지 책임 |
| R8 | Canva 송신측의 deprecated fileUrl signed URL 만료 | 🟡 Canva 측 이슈지만 Aura가 debug 요청받음 | 에러 응답에 최대한 원인 코드 명시. Canva 앱 프로젝트에 재시도 로직 권고(별도 task) |
| R9 | Tier 훅이 Seed 2 구현 전에 필요 | 🟡 임시 허용 정책 필요 | v1은 "모든 로그인 교사 허용" + 설정 페이지 상단에 "베타: Pro 제한은 추후 적용" 배너 |

---

## 8. 미결 질문 (phase3 인터뷰 재료)

본 스케치만으로 결정 불가한 7개. 에이전트가 즉결 가능한 건 이미 본문에서 결정했다.

1. **Q1 — Scope 체계 세분화 범위**
   - v1은 `cards:write`만 충분한가? 아니면 `submissions:write`, `webhooks:receive`까지 미리 예약해 둘까? 지금 `scopes String[]` 배열을 두면 확장은 가능하지만 기본값·검증 로직 복잡도 증가. (관련: Seed 5 P2-⑤ Webhook 미래 계획)

2. **Q2 — 유효기간 기본값·무기한 허용 여부**
   - phase1 §1-C는 "기본 90일" 권고. UI 드롭다운에 "무기한" 옵션을 둘지, 아니면 최대 365일로 강제할지. "무기한"은 GitHub도 2023년 이후 권장 폐지 방향. 교사 UX(재발급 귀찮음) vs 보안 회전 유도 트레이드오프.

3. **Q3 — 생성 후 1회 노출 시 다운로드(txt 파일) 제공?**
   - 태블릿에서 clipboard만 의존하면 "앱 이동 중 복사 내용 유실" 리스크. `.txt` 다운로드 버튼을 추가할지 (보안↓ vs UX↑), 또는 QR 코드로 Canva 앱에 전달(기술 허들)하는 옵션도. 기본은 clipboard only.

4. **Q4 — 레거시 `ExternalAccessToken` 처리**
   - 프로덕션 DB에 이미 발급된 row가 있는가? 있다면 `tokenPrefix` backfill 전략(tokenHash 앞 8자로 접두어 역산 불가 → **일괄 revoke + 재발급 공지**가 유일) vs 없으면 NOT NULL로 즉시 추가. 선확인 필요.

5. **Q5 — "Canva 앱으로 이동" 딥링크 URL**
   - Canva Apps가 제공하는 교사용 deeplink 스펙이 있는지 `aura-canva-app` 리포 또는 Canva 문서 확인 필요. 없으면 `https://www.canva.com/apps` 일반 URL로 폴백.

6. **Q6 — Card 생성 시 sectionId 지정 가능 여부**
   - Canva는 섹션 개념을 모름. v1은 `sectionId=null`(보드 최상단) 고정 제안. 교사가 "Breakout 모둠1에 바로 발행"을 원하면 Canva 앱 설정 UI에 sectionId 드롭다운을 추가해야 함. v1 범위로 제외하되 API 스펙에 `sectionId?` optional 필드 미리 예약할지 결정.

7. **Q7 — CRC32 checksum 도입 시점**
   - phase1 §1-B 🟡 "1차 출시 후 옵션". GitHub 로그 스캐너 호환성을 위해 v1부터 넣을지, v1.1로 미룰지. 구현 비용은 낮음(0.5일)이지만 토큰 포맷이 바뀌므로 나중 도입 시 기존 토큰과 공존 로직 필요.

**미결 질문 수: 7개** (contract 권장 7개 이하 충족)

---

## 9. Phase 2 → Phase 3 핸드오프

- 다음 단계: phase3 interview로 Q1~Q7 결정
- 결정 후 phase4 planner가 padlet 리포의 feature 파이프라인 task로 등록 (phase0 request.json 작성)
- padlet task는 본 sketch의 §1(스키마)·§2(API)·§3(UI) 그대로 구현 가능한 수준으로 구체화됨
- 참고 파일(구현 시):
  - `/home/coseung2/aura-canva-app/src/intents/content_publisher/index.tsx` — 클라이언트 계약 (읽기 전용)
  - `/mnt/c/Users/심보승/Desktop/Obsidian Vault/padlet/prisma/schema.prisma` — 기존 스키마
  - `ideation/plans/tablet-performance-roadmap.md` §2·§11 — QA 게이트
