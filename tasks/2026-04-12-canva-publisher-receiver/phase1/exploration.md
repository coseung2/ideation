# Phase 1 Exploration — Canva Publisher Receiver (`/api/external/cards` + PAT)

> task_id: `2026-04-12-canva-publisher-receiver`
> 작성: 2026-04-12 · explorer 에이전트
> 기준 단말: 갤럭시 탭 S6 Lite / Chrome Android / 학교 Wi-Fi 50 Mbps

---

## 0. Scope

Canva Content Publisher 앱(이미 완성)이 `POST /api/external/cards`로 보내는 `Authorization: Bearer {pat}` + `{ boardId, title, imageDataUrl(PNG base64 data URL) }` 계약을 Aura-board 서버가 수용하기 위한 세 하위 의사결정:

1. **PAT 발급·저장 전략** (해시 알고리즘, 포맷, 룩업 방식)
2. **이미지 업로드 경로** (data URL 인입 후 서버→Blob / 클라이언트 직접 업로드 / presigned URL)
3. **Rate limit + 태블릿 PAT 발급 UX**

---

## 1. PAT 발급·저장 전략 비교

### 1-A. 해시 알고리즘

| 후보 | 라이선스 | 검증 비용 | GPU 저항 | PAT 적합성 | Aura 적합성 |
|---|---|---|---|---|---|
| **bcrypt** (cost=10~12) | Public Domain | ~50–100ms | 중 | 암호용 설계 — **token 매 요청 hash 재계산 시 과중** | ❌ 요청당 50ms = p95 오염 |
| **Argon2id** (m=64MB,t=3) | CC0 / Apache-2 | ~80–200ms | 상 | 메모리 과다, Vercel Function cold start 악화 | ❌ 태블릿 동시 30명 시 비용 폭발 |
| **SHA-256 + prefix lookup** | Public Domain | < 1ms | N/A (고엔트로피 전제) | **고엔트로피 랜덤 토큰이면 충분** (GitHub/GitLab 실채용) | 🟢 권고 |
| **HMAC-SHA-256 (peppered)** | Public Domain | < 1ms | N/A | SHA-256 + 서버 비밀키 pepper — 환경변수 유출 내성 추가 | 🟢 강화 옵션 |

**핵심 근거**: 사용자 비밀번호(저엔트로피)와 PAT(128bit+ 랜덤)은 위협 모델이 다르다. 비밀번호는 무차별 대입 가능해 slow hash 필요, PAT은 난수 공간이 $2^{128}$ 이상이면 SHA-256 pre-image 공격이 불가능. GitLab/GitHub가 실제로 PAT을 bcrypt가 아닌 **SHA-256 lookup index**로 저장하는 이유([참조 2]).

### 1-B. 토큰 포맷·룩업 방식

| 패턴 | 예시 | 장점 | 단점 | Aura 권고 |
|---|---|---|---|---|
| **단일 opaque 문자열** | `a1b2c3...` | 간단 | 전체 테이블 hash 비교 필요, 실패해도 timing side-channel | ❌ |
| **Prefix + secret** (GitHub식) | `aurapat_<8char-id>_<32char-secret>` | prefix로 DB row 즉시 조회 → SHA-256 한 번만 검증, 로그 스캐너 탐지 용이 | 포맷 고정 | 🟢 **권고** |
| **JWT** | `eyJ...` | stateless, 만료 포함 | **폐기(revoke) 어려움** — PAT은 즉시 폐기 요건 필수 | ❌ |
| **Prefix + checksum** (GitHub 2021+) | `aurapat_...<CRC32>` | 가짜 토큰 DB 조회 전 차단, 로그 유출 탐지 가능 | CRC32 계산 추가 | 🟡 옵션 (1차 출시 후) |

**권고 포맷**:
```
aurapat_{8자 tokenId: base62}_{40자 secret: base62}
```
- `tokenId` = DB PK (plain, 비밀 아님) → `SELECT ... WHERE token_id = ?` O(1)
- `secret` = 고엔트로피 랜덤 (40 base62 ≈ 238bit)
- 저장: `hashed_secret = SHA-256(secret || PEPPER)` 한 컬럼
- 검증: prefix split → `tokenId`로 row 1개 조회 → SHA-256 재계산 → `timingSafeEqual`
- **평문은 생성 직후 단 1회만 반환** (GitHub·Notion·Linear·Vercel 공통)

### 1-C. 메타 필드 (참고 구현 공통분모)

| 필드 | GitHub | GitLab | Linear | Vercel | Aura 결론 |
|---|---|---|---|---|---|
| `name` (라벨) | ✅ | ✅ | ✅ | ✅ | **필수** — "내 교실용" 같은 메모 |
| `scopes`/`boardIds` | ✅ | ✅ | ✅ | ✅ | **필수** — 교사 소유 board만 선택 가능 |
| `expires_at` | 선택 (권장 최대 1년) | 필수 | 선택 | 선택 | **필수 + 기본 90일** (교실 연간 회전) |
| `last_used_at` | ✅ | ✅ | ✅ | ✅ | **필수** — 유령 토큰 회수 판단 |
| `revoked_at` | soft delete | soft | hard | hard | **soft delete** (감사 로그 보존) |

---

## 2. 이미지 업로드 경로 비교

Canva 클라이언트 스펙이 **`imageDataUrl` (base64 data URL)** 로 고정되어 있어 선택지가 제약됨.

| 경로 | 스펙 수용 | Vercel 4.5MB 서버 제한 | 태블릿 영향 | Aura tier/CORS 게이팅 | 권고 |
|---|---|---|---|---|---|
| **A. 서버에서 data URL 수신 → decode → `put()` Vercel Blob** | 🟢 스펙 그대로 | ⚠️ base64는 원본의 ~1.37배 — 3.2MB PNG부터 413 위험 | 🟢 태블릿 쪽 로직 無 (Canva 데스크톱/아이패드가 송신) | 🟢 Bearer로 게이팅 | 🟢 **MVP 권고** |
| **B. 서버에서 presigned URL 발급 → 클라이언트 직접 S3/Blob 업로드** | ❌ **Canva 클라가 이미 data URL 송신 — 스펙 위반** | — | — | — | ❌ (재협상 필요) |
| **C. multipart/form-data** | ❌ 스펙 위반 | 4.5MB | — | — | ❌ |
| **D. Route Handler 대신 Server Action** | ❌ Canva 앱은 fetch 호출 — Server Action 불가 | `bodySizeLimit` 설정 가능 | — | — | ❌ |

### 2-A. 413 한도 내 운용 전략 (경로 A 채택 전제)

Next.js 15 App Router **Route Handler는 body size 설정 불가** ([참조 1] 공식 문서). Vercel Function은 **4.5MB 하드 리밋**([참조 3] Vercel Blob 공식 문서).

**대응**:
1. **Canva 앱 쪽 프리-리사이즈 강제** — PNG를 보내기 전 `createImageBitmap` + canvas로 최대 **1600×1200** 리사이즈(수업 썸네일 충분). base64 인플레이션 1.37× 반영 시 ≈ 2.5MB raw → 3.4MB body. 안전 마진.
2. **서버 early-reject**: `content-length > 4_000_000` 즉시 413 반환 (메모리 낭비 차단).
3. **data URL 파서**: `req.text()` → `data:image/png;base64,` 분리 → `Buffer.from(..., 'base64')` → `put(key, buffer, { access: 'public', contentType: 'image/png' })`.
4. **키 정책**: `boards/{boardId}/cards/{cardId}.png` (카드 삭제 시 cascade 용이).
5. **Runtime**: **Node.js runtime 고정** (Buffer/Blob 처리, Edge는 2MB 페이로드 한도로 더 빡빡). `export const runtime = 'nodejs'`.

> ⚠️ 현실 게이트: Canva 송신측(완성본)이 **이미 리사이즈**하고 있는지 exploration 다음 단계(phase2 sketcher)에서 `~/aura-canva-app/src/intents/content_publisher/index.tsx` 확인 필요. 미리사이즈면 서버 측에서 sharp로 다운사이즈 후 Blob 저장.

### 2-B. 스토리지 비교

| 스토리지 | 라이선스 | 가격 (10GB/월) | 리전 | Aura 적합성 |
|---|---|---|---|---|
| **Vercel Blob** | Proprietary (S3 backed) | 저장 $0.023/GB + 대역 $0.03/GB | Auto | 🟢 MVP — 환경변수 자동 주입, `@vercel/blob` SDK 단순 |
| **Supabase Storage** | Apache-2 (self-host 가능) | 포함 (2GB 프리) + $0.021/GB | Region lock | 🟡 DB 이미 Supabase면 RLS 재사용 이득. 단 Aura는 Prisma 사용 여부 확인 필요 |
| **AWS S3 direct** | Proprietary | $0.023/GB | Region lock | 🟡 CloudFront 추가 구성 필요 — MVP 과투자 |
| **Cloudflare R2** | Proprietary | $0.015/GB, 대역 무료 | Global | 🟢 장기적 저렴 — 대역 무료가 교실 30명×이미지에 유리, 그러나 Vercel Image Optimization 연동 간접 |

---

## 3. Rate Limiting

| 옵션 | 라이선스 | 저장소 | 정밀도 | Aura 적합성 |
|---|---|---|---|---|
| **Upstash `@upstash/ratelimit`** | Apache-2 | Upstash Redis (REST) | sliding window | 🟢 **권고** — Edge/Node 양립, per-token identifier 지원 |
| **Vercel KV + custom** | Proprietary (Upstash 래핑) | Redis | 수동 구현 | 🟡 동일 엔진, 상위 래퍼 직접 구현 오버헤드 |
| **In-memory LRU** | MIT | 메모리 | 프로세스당 | ❌ Vercel serverless 인스턴스 분산 시 부정확 |
| **Supabase `pg_stat`** | Apache-2 | Postgres | 낮음 | ❌ DB round-trip 비용 |

### 권고 설정

```ts
// per-token: 60 req / min (교사 1명이 분당 60장 발행은 비현실)
const perToken = new Ratelimit({ limiter: slidingWindow(60, '60 s'), ... });
// per-teacher (token 우회 방지): 200 req / 10min
const perTeacher = new Ratelimit({ limiter: slidingWindow(200, '600 s'), ... });
// per-IP (Canva CDN 공유 IP 고려해 느슨)
const perIp = new Ratelimit({ limiter: slidingWindow(300, '60 s'), ... });
```

초과 시 `429 Too Many Requests` + `Retry-After` 헤더 + Canva UI에 "잠시 후 다시" 토스트.

---

## 4. 태블릿(갤럭시 탭 S6 Lite) PAT 발급 UX

### 4-A. 제약

- 갤럭시 탭 S6 Lite 화면 10.4인치, Chrome Android
- 교사가 **수업 중 5분 내** PAT 발급 → Canva 앱에 붙여넣기
- `navigator.clipboard.writeText()` — Android Chrome 66+ 지원 ✅
- 40+자 랜덤 문자열은 손으로 옮기기 불가 → **원터치 copy 필수**

### 4-B. 설계 결정

| 요소 | 결정 | 근거 |
|---|---|---|
| 평문 표출 | **모달 1회만, 재표시 절대 불가** | GitHub/GitLab/Notion/Linear/Vercel 공통 ([참조 5]) |
| Copy 버튼 | **44×44pt 이상, 토큰 문자열 바로 옆**, 눌렀을 때 tooltip "복사됨!" 1.5s | Android Developers 가이드 + PatternFly clipboard-copy |
| 토큰 시각화 | `monospace` 폰트, 12자 단위 띄어쓰기 표시 (실제 복사 값에는 공백 無) | 가독성 + 오독 방지 |
| 만료 선택 | 드롭다운 (30/90/365일), **기본 90일** | 학기 단위 회전 유도 |
| 보드 스코프 | 교사 소유 보드 체크박스 목록 (전체 선택 기본) | 최소 권한 원칙 완화 — 교사는 1~3반 소유 |
| 발급 후 화면 | "Canva 앱으로 이동" 딥링크 버튼 + 설치 가이드 | 태블릿 교사 문서 없이 완주 |
| 목록 화면 | `name`·`last_used_at`·`expires_at`·`Revoke` | 고스트 토큰 식별 |
| Revoke | 탭 → 확인 다이얼로그 → soft delete, 이후 `401 token_revoked` | 감사 로그 보존 |
| Tier 게이팅 | Free 교사는 생성 버튼 disabled + "Pro 플랜" 배지 | Seed 2 Tier 정책 |

### 4-C. 에러 코드 매핑 (Canva UI 가독성)

| 코드 | 사유 | Canva UI 문구 가이드 |
|---|---|---|
| 401 `invalid_token` | PAT 포맷 틀림 / 없음 | "API 키가 유효하지 않아요. Aura-board 설정에서 재발급하세요." |
| 401 `token_revoked` | soft-deleted | "폐기된 키입니다. 새 키를 발급하세요." |
| 401 `token_expired` | 만료 | "키 만료. 재발급 필요." |
| 403 `forbidden_board` | boardId 소유 아님 | "이 보드에 발행 권한이 없습니다." |
| 403 `tier_locked` | Free 교사 | "Pro 플랜 기능입니다." |
| 413 `payload_too_large` | body > 4MB | "이미지가 너무 큽니다. 캔바에서 축소 후 재시도." |
| 422 `invalid_data_url` | data URL 파싱 실패 | "이미지 형식 오류." |
| 429 `rate_limited` | 한도 초과 | "잠시 후 다시 시도하세요." |

---

## 5. 아키텍처 1순위 권고 (요약)

```
Canva Publisher 앱 (완성본)
      │ POST /api/external/cards
      │ Authorization: Bearer aurapat_<id>_<secret>
      │ body: { boardId, title, imageDataUrl }
      ▼
┌─────────────────────────────────────────────┐
│ Next.js Route Handler (runtime = 'nodejs')  │
│ export const runtime = 'nodejs'             │
│ 1. content-length 조기 413 가드 (4MB)        │
│ 2. Upstash rate limit (token / teacher / IP)│
│ 3. PAT 검증: prefix split → DB lookup       │
│    → SHA-256(secret||PEPPER) timingSafeEq   │
│    → last_used_at 갱신                       │
│ 4. tier / boardId 권한 체크                  │
│ 5. data URL decode → Buffer                 │
│ 6. @vercel/blob put() → URL                 │
│ 7. Prisma: cards.create({ imageUrl, ... })  │
│ 8. return { id, url: `/b/{slug}#c/{id}` }   │
└─────────────────────────────────────────────┘
```

### DB 스키마 추가 (Prisma 가정)

```prisma
model PersonalAccessToken {
  id            String   @id @default(cuid()) // = tokenId (prefix)
  teacherId     String
  name          String
  hashedSecret  String   // SHA-256 hex of (secret||PEPPER)
  scopeBoardIds String[] // 빈 배열 = 전체 교사 소유 보드
  expiresAt     DateTime?
  lastUsedAt    DateTime?
  revokedAt     DateTime?
  createdAt     DateTime @default(now())
  teacher       Teacher  @relation(fields: [teacherId], references: [id])
  @@index([teacherId])
}
```

### 1순위 권고 근거

**Aura-board의 제약(갤럭시 탭 성능 예산·GPL 격리·tier)에 가장 적합한 조합은 `SHA-256 prefix-lookup PAT + Vercel Blob 서버 업로드 + Upstash sliding-window rate limit`** 이다. 이유:

1. **태블릿 성능 예산**: Route Handler는 p95 < 500ms 요구. bcrypt/Argon2는 요청당 50~200ms 즉 쿼터의 40%를 해시에 소모하지만, **SHA-256 prefix-lookup은 < 1ms**로 여유. 30명 교실에서 동시 publish 스파이크 흡수 가능.
2. **스펙 수용**: Canva 송신측은 data URL 고정 — presigned URL은 선택지 아님. Vercel Blob 서버 업로드가 `@vercel/blob.put()` 한 줄로 해결되며 MVP 속도 최대.
3. **보안·감사**: prefix 포맷은 로그 유출 시 secret scanner가 탐지 가능(GitHub 2021 이후 업계 표준). soft delete + last_used_at으로 유령 토큰 식별 루틴 제공.
4. **태블릿 UX**: GitHub/Linear/Notion/Vercel 공통 패턴(1회 노출 + 원터치 copy)은 교사가 5분 내 Canva까지 완주하는 shortest path. 평문 재표시 불가 정책은 감사 가능성을 보장.
5. **Aura 스택 시너지**: Vercel Blob은 환경변수 자동 주입·Prisma 스키마 확장만으로 배포 가능해 P0-② 로드맵의 빠른 착륙과 일치. Upstash는 Edge·Node 양쪽 호환으로 향후 Edge 이전 옵션 열어둠.

### Phase 2 Sketch (다음 단계)

- Route Handler 스펙 JSON + zod 스키마
- PAT 발급 설정 화면 와이어프레임 (갤럭시 탭 스케치)
- 에러 코드 테이블 → Canva 앱 intent UI 매핑 스펙
- DB 마이그레이션 SQL 초안
- Canva 송신측 이미지 리사이즈 여부 확인 TODO (`~/aura-canva-app/src/intents/content_publisher/index.tsx`)

---

## 참조 링크

1. [Next.js: App Router Route Handlers — body size limit is not configurable](https://github.com/vercel/next.js/issues/57501) — bodyParser 설정은 Pages API 전용, App Router는 설정 불가 확인.
2. [Behind GitHub's new authentication token formats (GitHub Blog)](https://github.blog/engineering/platform-security/behind-githubs-new-authentication-token-formats/) — prefix `ghp_` + checksum + SHA-256 lookup 설계의 원본 논리.
3. [Vercel Blob — Server Uploads (공식 문서, WebFetch 검증)](https://vercel.com/docs/vercel-blob/server-upload) — 서버 4.5MB 하드 리밋, `put()` API, 대용량은 client upload 권고.
4. [Upstash `@upstash/ratelimit` (npm 공식)](https://www.npmjs.com/package/@upstash/ratelimit) — sliding window, per-identifier rate limit, Edge runtime 호환.
5. [GitHub Docs: Managing personal access tokens (WebFetch 검증)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) — 1회 노출 + copy icon + 만료 선택 + 즉시 revoke UX 기준선.
6. [Password Hashing Guide 2025 — Argon2 vs Bcrypt vs SHA-256](https://guptadeepak.com/the-complete-guide-to-password-hashing-argon2-vs-bcrypt-vs-scrypt-vs-pbkdf2-2026/) — 고엔트로피 API 토큰과 저엔트로피 비밀번호의 위협 모델 차이.
