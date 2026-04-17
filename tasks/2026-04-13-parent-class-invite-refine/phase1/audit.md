# Phase 1 감사 보고 — Parent Class Invite Refine

- **task_id**: `2026-04-13-parent-class-invite-refine`
- **target seed**: `seed_37b35654542f` (학부모 읽기 전용 뷰어 액세스, ambiguity 0.074)
- **감사 대상**: `plans/parent-viewer-roadmap.md` (전 섹션) + `seeds-index.md#Seed-7` + `phase3/decisions.md` 17건 + `phase4/seed.yaml`
- **change_trigger (고정)**: 학생별 개별 코드 발급·전달의 교사 운영 부담을 줄이고 학부모 셀프 온보딩(이메일 가입 + 학급 코드 + 자녀 셀프매칭)을 도입하되 사칭 리스크를 **교사 승인 게이트**로 차단한다.
- **새 흐름 (비교 기준)**:
  1. 학급 단위 단일 초대 코드 (학생별 개별 코드 폐지)
  2. 학부모 이메일 셀프 가입 → 학급 코드 입력 → 학생 명단에서 본인 자녀 선택
  3. `ParentChildLink.status = "pending"` 상태로 대기 → 교사 승인(`approved`) 후 `active` 전이

---

## 1. 결정 표 판정 (parent-viewer-roadmap.md §1.1 ~ §1.4 전 행)

범례: ✅ 유지 / ⚠️ 재검토(조정·확장 필요) / ❌ 변경(폐기·교체)

### §1.1 페어링 · 인증 (roadmap L25~L36)

| # | 행 | 현 결정 내용 | 판정 | 근거 (한 줄) |
|---|---|---|---|---|
| 1.1-a | 페어링 방식 | **Crockford Base32 6자리 + QR** (교사 발급 → 학생 카드 드롭다운 → 부모폰 스캔) | ❌ | 학생 카드 드롭다운 발급 방식이 폐기되고 학급 설정 단일 발급으로 완전 대체됨. |
| 1.1-b | 코드 엔트로피 | 32⁶ ≈ 1×10⁹, O/0·I/1·L 제외 대문자, `crypto.randomBytes` | ⚠️ | CSPRNG·Crockford 원칙은 유지하되 학급 코드는 노출 면적·수명↑ → 8자리(32⁸≈10¹²) 승격 또는 회전 주기 필수. |
| 1.1-c | 코드 수명 | 48h 만료 또는 maxUses 3 소진 | ❌ | 학급 단위는 학부모 온보딩 기간(학기 초 수 주) 지속 필요 → TTL·maxUses 정책 전면 재설계(예: 학기말까지 + 무제한 maxUses + 교사 수동 회전). |
| 1.1-d | brute-force 방어 | IP 5회/15분 잠금 + 코드 10회 실패 즉시 만료 | ⚠️ | IP 잠금은 유지. 그러나 "코드 10회 실패 즉시 만료"는 학급 코드에 쓰면 학급 전체 학부모 가입 봉쇄 위험 → 코드 단위 만료 트리거 재설계(예: 시도 실패는 IP 잠금만, 코드는 교사 수동 회전). |
| 1.1-e | 최초 인증 | 매직 링크(이메일 OTP) 전용 v1, 유효 15분 | ⚠️ | 매직 링크 수단은 유지. 단 가입(signup) 단계와 매칭 신청(match) 단계가 분리되므로 이메일 인증 시점·링크 용도(verify vs approve-notify) 재정의 필요. |
| 1.1-f | 세션 | `ParentSession` 7일, 만료 후 이메일만 재인증 | ✅ | 세션 정책은 페어링 단위 변경과 독립 — 유지. |
| 1.1-g | 동일 이메일 재가입 | 기존 `ParentChildLink`에 새 세션만 추가 (중복 Parent 방지) | ⚠️ | 단일 Parent 레코드 재사용 원칙 유지. 그러나 셀프매칭 흐름에서 **다른 학급 코드 추가 입력 → 추가 pending 링크 생성** 경로 신규 → 정책 문서 보강 필요. |
| 1.1-h | 학부모 1인당 자녀 수 | 5명 상한 (tier 무관) | ✅ | 상한 자체는 유지. 단 pending 포함 상한인지 active만 카운트인지 delta에서 명시 필요(⚠️ 경계). |

### §1.2 Revoke · 격리 (roadmap L40~L51)

| # | 행 | 현 결정 내용 | 판정 | 근거 |
|---|---|---|---|---|
| 1.2-a | Revoke SLA | ≤ 60초 (SWR 60s에 얹음) | ⚠️ | Revoke SLA 자체는 유지되나, **승인(approve) SLA**·**pending 자동 만료 TTL** 신규 SLA 항목 추가 필요. |
| 1.2-b | Revoke 경로 | 교사 1-click (학급 설정 → 학부모 액세스 탭) / 자발 탈퇴 | ⚠️ | 경로 유지. 다만 "reject(승인 거부)" 신규 경로 + "pending auto-expire" 추가로 action 종류가 3 → 5로 확장. |
| 1.2-c | 세션 차단 | `revokedAt IS NOT NULL` 또는 `deletedAt IS NOT NULL` → 401 | ⚠️ | 유지하되 **pending 상태 링크는 세션 발급 자체 금지 또는 세션은 있되 자녀 데이터 API가 403** — pending 처리 분기 신규. |
| 1.2-d | 클라이언트 401 처리 | 자동 로그아웃 + "접근이 해제되었습니다" | ⚠️ | "접근 해제" 메시지 외에 "승인 대기 중입니다" 상태 화면 신규 필요(401이 아닌 별도 상태 코드 또는 payload flag). |
| 1.2-e | 교사 UI 문구 "최대 1분 내 차단" | — | ✅ | Revoke 메커니즘 자체는 변동 없음 — 유지. |
| 1.2-f | 타 학생 API 격리 → 403 | parent 토큰으로 타 학생 studentId 호출 차단 | ✅ | 격리 원칙 유지 — 변경 없음. |
| 1.2-g | 타 학부모 데이터 격리 → 404 | parentA → parentB의 링크 조회 시도 | ✅ | RLS 단방향 유지. |
| 1.2-h | DOM 마스킹 + presigned 썸네일 | API 1차 + DOM 보조 | ✅ | 매칭 후 열람 범위는 변동 없음. |
| 1.2-i | 자녀 이름 본명 기본 노출 | 자녀 본인 업로드 사진만 표시 | ⚠️ | 매칭 이후는 유지. 그러나 **매칭 신청 단계(pending 이전)에 학부모가 학급 학생 명단을 보게 됨** → 명단 표시 필드 범위(이름만? 출석번호? 사진?) 신규 결정 필요. |
| 1.2-j | 학부모 간 격리 (BCC 금지 등) | 동일 자녀 부/모/조부모 상호 비노출 | ✅ | 격리 정책은 유지. |

### §1.3 알림 · Tier (roadmap L55~L63)

| # | 행 | 현 결정 내용 | 판정 | 근거 |
|---|---|---|---|---|
| 1.3-a | v1 알림 범위 | 인앱 배지 + 주간 이메일(Pro) | ⚠️ | 유지하되 **"승인 대기 N건" 교사 인앱 배지 + 학부모에게 "승인 완료/거부" 알림** 신규 이벤트 추가 필요. |
| 1.3-b | 주간 이메일 | 월 09:00 KST, 활동 0건 스킵 | ✅ | 주간 이메일 정책은 변동 없음. |
| 1.3-c | 이메일 콘텐츠 (헤더·집계·썸네일·피드백) | — | ✅ | 내용 구성은 매칭 활성화 이후이므로 유지. |
| 1.3-d | 이메일 수신 거부 `emailSummaryOptOut` | — | ✅ | 유지. |
| 1.3-e | **Free tier 발급 한도 자녀당 2** | 인앱 배지만 | ❌ | "자녀당 N" 개념이 학급 코드로 이동 시 의미 붕괴 — 학급당? 교사당? 학교당? 또는 폐지 후 다른 축(승인 건수·코드 회전 빈도)으로 대체 필요. |
| 1.3-f | **Pro tier 발급 한도 자녀당 5** | 주간 이메일 + 향후 푸시 | ❌ | 1.3-e와 동일 — 재정의 필수. |
| 1.3-g | v1 순수 read-only | 응원·좋아요·댓글 v2+ 파킹 | ✅ | 변경 트리거와 무관 — 유지. |

### §1.4 탈퇴 · 감사 (roadmap L67~L74)

| # | 행 | 현 결정 내용 | 판정 | 근거 |
|---|---|---|---|---|
| 1.4-a | Soft delete 고정 | hard delete 금지 | ✅ | 탈퇴 정책은 유지. |
| 1.4-b | 탈퇴 시퀀스 (`deletedAt` + session 무효 + link revoked + self_withdraw) | — | ⚠️ | 기본 시퀀스 유지하되 **pending 상태 링크 탈퇴 처리**는 경로 하나 추가 필요(거부 대기 중 학부모 자발 취소). |
| 1.4-c | 90일 익명화 Cron | email SHA-256, displayName "탈퇴한 학부모" | ✅ | 유지. |
| 1.4-d | 90일 내 재가입 | `deletedAt=null` 복구, 링크는 미복원 → 교사 재발급 필요 | ⚠️ | "교사 재발급"이 학생별 코드 전제 → 새 흐름에서는 **학급 코드 재입력 + 재승인** 경로로 치환. |
| 1.4-e | 감사 필드 `issuedById` + `revokedAt` + `revokedReason`(teacher_action/self_withdraw) | — | ⚠️ | `issuedById`는 학급 코드 발급 교사로 의미가 **승인 교사(approvedById)**와 분리되어야 함. `revokedReason` 유니언에 `rejected_by_teacher` + `auto_expired_pending` 추가 필요. |
| 1.4-f | 교사 UI 탈퇴 표시 "연결 해제됨 (탈퇴)" | — | ✅ | 유지. |

---

## 2. 데이터 모델 감사 (roadmap §2, L80~L154)

### 2.1 Prisma 엔티티별 영향

| 모델 | 현 스키마 위치 | 판정 | 변경 요지 |
|---|---|---|---|
| `Parent` (L81~L96) | 상태 필드 `revokedAt`·`deletedAt`·`emailSummaryOptOut` | ✅ 유지 | Parent 단독으로는 변경 없음. 단 신규 가입 경로(셀프 signup)는 Parent INSERT 시점만 이동. |
| `ParentChildLink` (L98~L116) | `status: "active"\|"revoked"` (L102), `revokedReason` (L104) | ❌ | **status 유니언 확장 필수**: `"pending" \| "active" \| "rejected" \| "revoked"`. 신규 필드: `requestedAt`, `approvedAt`, `approvedById`, `rejectedAt`, `rejectedById`, `rejectedReason`. `@@unique([parentId, studentId])` 제약은 유지(동일 자녀 중복 신청 방지) — 단 재신청 흐름 시 기존 rejected 레코드 덮어쓰기 정책 결정 필요. |
| `ParentInviteCode` (L118~L133) | `studentId` FK 필수 (L121) + `maxUses: Int @default(3)` (L124) | ❌ | **전면 교체**: 신규 `ClassInviteCode` 엔티티 도입 또는 `studentId` → `classroomId`로 전환. `maxUses` 삭제 또는 대폭 확대. `failedAttempts` 의미 재고(학급 코드는 정상 사용 실패 다수 예상). 신규 필드 후보: `rotatedAt`, `rotatedById`, `revokedAt`. |
| `ParentSession` (L135~L148) | 토큰·만료·revokedAt | ✅ 유지 | 세션 모델은 변동 없음. 단 pending 상태에서 세션 발급 허용 여부 delta 결정 필요. |
| `BoardMember` (L150~L154) | `role: "parent"` 유니언 | ⚠️ | 현 유니언은 유지되나, pending 상태를 BoardMember에 표현할지(`role="parent-pending"`) 또는 ParentChildLink.status에만 표현할지 결정 필요. 권장: **BoardMember INSERT는 approve 시점**으로 지연. |

### 2.2 RLS 정책 (roadmap L156~L159)

- `ParentChildLink` SELECT: `parent_id = auth.parent_id()` → **⚠️** `status='active'` 조건 추가 필수. pending·rejected 레코드는 학부모 자신 본인 신청 상태 조회용으로만 노출.
- `Parent` SELECT 자기 레코드만 → ✅ 유지.
- 콘텐츠 테이블(StudentAsset·PlantObservation·EventSignup·BreakoutMembership·Submission) `studentId ∈ parent.children` → **⚠️** `parent.children`의 정의를 `status='active' linkedChildren`으로 제한하는 조건 추가(pending·rejected는 자녀로 간주 금지).

### 2.3 신규 엔티티 제안 (delta가 다룰 대상)

- `ClassInviteCode` (별도 엔티티 안) — `{ id, code, classroomId, issuedById, createdAt, expiresAt, revokedAt, rotatedAt }`
- `ParentMatchRequest` (선택 안, 옵션) — pending 상태를 별도 테이블로 분리 vs `ParentChildLink.status='pending'`로 단일 테이블 유지 중 택1 (delta 결정 사항).

---

## 3. 시퀀스 다이어그램 감사 (roadmap §3, L165~L213)

### 3.1 §3.1 페어링 시퀀스 (L165~L191)

**판정: ❌ 전면 재작성.**

현 흐름(학생 카드 드롭다운 → 교사 코드 발급 → 학부모 코드+이메일 입력 → `ParentChildLink UPSERT` → 매직링크)은 3단계 중 2단계가 완전 교체됨.

신 흐름(스케치안 — delta에서 확정):
```
[교사] 학급 설정 → "학부모 초대 코드" 발급 (학급당 1개)
         └─ POST /api/classrooms/:id/invite-codes
         └─ ClassInviteCode INSERT {classroomId, code, expiresAt=EoT/학기말}

[학부모 스마트폰] /parent/signup → 이메일 + 표시 이름 입력
         └─ POST /api/parent/signup  ← 신규 엔드포인트
         └─ Parent INSERT + 매직 링크 발송(이메일 인증 15분)

[학부모] 매직 링크 클릭 → /parent/verify → ParentSession 발급

[학부모] /parent/match → 학급 코드 입력
         └─ POST /api/parent/match/code  ← 신규
         └─ ClassInviteCode 검증(IP rate limit) → 학급 학생 명단 반환(마스킹 규칙 적용)

[학부모] 명단에서 자녀 선택(복수 가능, 5명 상한 내)
         └─ POST /api/parent/match/request { studentIds[] }  ← 신규
         └─ ParentChildLink INSERT {status:"pending", requestedAt}

[교사] 학급 설정 "승인 대기" 인박스(N 배지)
         └─ GET /api/parent-child-links?status=pending
         └─ [승인] POST /:id/approve → status="active", approvedAt, approvedById, BoardMember INSERT
         └─ [거부] POST /:id/reject  → status="rejected", rejectedReason

[학부모] 승인 통보(인앱/이메일) → /parent/home 진입
[pending 7일 초과] Cron auto-expire → status="rejected", revokedReason="auto_expired_pending"
```

### 3.2 §3.2 Revoke 시퀀스 (L194~L213)

**판정: ⚠️ 분기 추가.** 기본 Revoke·탈퇴 경로는 유지되며, **승인 거부(reject) 경로** + **pending 자동 만료 경로** 2개가 신규 추가됨.

---

## 4. parentScopeMiddleware 감사 (roadmap §4, L217~L245)

**판정: ⚠️ 수정.**

- 현 미들웨어(L234): `prisma.parentChildLink.findFirst({ where: { parentId, studentId, status: "active" }})` — `status: "active"` 조건이 이미 있어 pending/rejected 방어는 일단 자동 적용됨 (**다행**).
- 단, **신규 엔드포인트 3종**(`/api/parent/signup`, `/api/parent/match/code`, `/api/parent/match/request`)은 **parentScopeMiddleware의 `studentId ∈ parent.children` 검증을 우회**해야 함 — 매칭 전 단계이므로 별도 미들웨어(`parentAuthOnlyMiddleware`) 도입 필요.
- `req.children` 계산(L241)에 `status='active'` 필터 명시 필요.

---

## 5. 자녀 범위 매트릭스 감사 (roadmap §5, L249~L266)

**판정: ✅ 매트릭스 자체는 유지 (out_of_scope로 target.json에서 제외됨)**.

- 모든 행의 "학부모 열람 범위" · "서버 필터" · "DOM 마스킹"은 **매칭 활성화(status='active') 이후의 규칙**이므로 완전 유지.
- 단 **공통 원칙 4 "privacy 토글 무관"**(L266)은 pending 동안에는 자녀 본인 콘텐츠도 열람 불가로 명시 보강 필요 (⚠️ 한 줄 추가).

---

## 6. 작업 카드 PV-1~PV-12 영향 매핑 (roadmap §7, L298~L315)

| # | 작업 | 판정 | 영향 요지 |
|---|---|---|---|
| PV-1 | 스키마 마이그레이션 (4종 모델 + RLS) | ❌ | `ParentInviteCode → ClassInviteCode` 치환 또는 studentId → classroomId 전환, `ParentChildLink.status` 유니언 확장, 신규 필드(requestedAt·approvedAt·approvedById·rejectedAt·rejectedById) 추가, RLS 정책에 `status='active'` 조건 명시. 공수 **+1일** 예상. |
| PV-2 | Crockford Base32 + 학생 카드 드롭다운 + QR | ❌ | 학생 카드 드롭다운 UI 폐기. 학급 설정 화면에 **단일 코드/QR + 회전 버튼 + 만료 정책 표시** UI로 전면 개편. 공수 유지. |
| PV-3 | `POST /api/parent/redeem` (코드 + 매직링크) | ❌ | 단일 엔드포인트가 **3개**로 분리: `POST /api/parent/signup` + `POST /api/parent/match/code` + `POST /api/parent/match/request`. 공수 **+1일**. |
| PV-4 | `GET /parent/verify` (매직링크 소비) | ✅ | 세션 발급 로직 유지. 단 verify 직후 리다이렉트가 `/parent/home`(기존) → `/parent/match`(자녀 미매칭 시) 또는 `/parent/pending`(승인 대기 시) 분기 필요. |
| PV-5 | `parentScopeMiddleware` | ⚠️ | 기존 미들웨어 유지 + `parentAuthOnlyMiddleware`(매칭 전 엔드포인트용) 신규 작성. 공수 +0.5일. |
| PV-6 | `/parent/*` PWA 쉘 | ⚠️ | 라우트 구조에 `/parent/signup`·`/parent/match`·`/parent/pending` 3개 신규. 홈 카드 구조는 유지. |
| PV-7 | 자녀 범위 서버 필터 일괄 | ✅ | 매칭 활성화 이후 규칙은 변동 없음 — 유지. |
| PV-8 | 교사 관리 UI (학급 설정 "학부모 액세스" 탭) | ❌ | 전면 개편: **"승인 대기 인박스" 섹션 신설**(신청별 학부모 이메일·선택 자녀·신청 시각·Approve/Reject 버튼 + 일괄 승인). 기존 연결 리스트는 유지. 공수 **+1~2일**. |
| PV-9 | Revoke SLA ≤ 60s | ⚠️ | Revoke 구현 유지 + **승인 SLA(24h 권고) + pending auto-expire Cron(7일)** 신규 작업 추가. 공수 +0.5일. |
| PV-10 | 주간 이메일 요약 | ✅ | 매칭 활성화 후 이메일 — 변동 없음. |
| PV-11 | 학부모 탈퇴 플로우 | ⚠️ | pending 상태에서 탈퇴 처리 경로 추가. 공수 +0.5일. |
| PV-12 | E2E 보안 게이트 테스트 | ❌ | 신규 케이스 대량 추가: pending 상태에서 자녀 데이터 접근 401/403, reject 후 접근 차단, pending auto-expire, 학급 코드 brute-force·회전, 명단 조회 시 마스킹 범위, 사칭 신청 감지. 공수 **+2일**. |

**총 공수 증분 예상**: 기존 27일 → **+6~7일 → 33~34일** (delta에서 정밀화).

**신규 작업 카드 후보 (delta 필수)**:
- **PV-13** (신규): 학급 코드 발급/회전 교사 UI + `ClassInviteCode` CRUD
- **PV-14** (신규): 학부모 셀프 가입/매칭 플로우 (`/parent/signup` + `/parent/match`)
- **PV-15** (신규): 교사 승인 인박스 + Approve/Reject + 일괄 승인
- **PV-16** (신규): pending auto-expire Cron + 알림(교사·학부모 양쪽)

---

## 7. 연쇄 영향 다이어그램

```
[change_trigger: 학생별 → 학급별 + 교사 승인 게이트]
                │
    ┌───────────┼─────────────────────────────┬──────────────────────┐
    ↓           ↓                             ↓                      ↓
[Prisma 스키마] [API/미들웨어]             [교사 UI]              [학부모 UI]
    │           │                             │                      │
 ParentInvite   /api/parent-invite-codes   학생 카드 드롭다운     /parent/enter
 Code(studentId)  (student 발급)              "학부모 초대"          (코드 입력)
    │           │                             │                      │
    ↓ 치환       ↓ 분리                        ↓ 이동                  ↓ 분리
 ClassInvite   signup + match/code +       학급 설정 단일 코드    /parent/signup
 Code(classroomId) match/request              + QR + 회전           /parent/match
                                              + 승인 대기 인박스     /parent/pending
    ↓           ↓                             ↓                      ↓
 ParentChildLink.status:                    승인/거부 액션          매칭 신청 + 상태
 "pending"|"active"|"rejected"|"revoked"   (approve/reject/bulk)    추적
 + requestedAt, approvedAt, approvedById,
   rejectedAt, rejectedById, rejectedReason
    │
    ↓
[RLS]  parent_id=auth.parent_id() AND status='active'
       (pending·rejected는 본인 신청 조회만)
    │
    ↓
[parentScopeMiddleware]  parent.children := active links only
[parentAuthOnlyMiddleware] (신규 — 매칭 전 엔드포인트용)
    │
    ↓
[자녀 범위 매트릭스(§5)] 매칭 활성화 이후 규칙 — ✅ 변동 없음
    │
    ↓
[알림] "승인 대기 N건" 교사 배지 + "승인 완료/거부" 학부모 알림 신규
    │
    ↓
[작업 카드] PV-1·PV-2·PV-3·PV-8·PV-12 전면 수정 + PV-13~16 신규
```

---

## 8. 뒤집힐 후보 결정 (≥ 5, 구체)

delta가 결정해야 할 **뒤집힐 후보** — 각 후보마다 현 값 → 신규 옵션 제시:

### 후보 A — 페어링 단위 (cardinality)
- **현**: 학생 1 : 코드 1 (`ParentInviteCode.studentId` unique not enforced but issued per student)
- **신 옵션 A1**: 학급 1 : 코드 1 (별도 엔티티 `ClassInviteCode`)
- **신 옵션 A2**: 학급 1 : 코드 N (세대별 회전 — active 1 + rotating spare)
- **권장**: A1 (단순성 우선)

### 후보 B — 코드 엔트로피 / 수명 / 회전
- **현**: 6자리(32⁶≈10⁹) + 48h + maxUses 3
- **신 옵션 B1**: 8자리(32⁸≈10¹²) + 학기말 TTL + 무제한 maxUses + 교사 수동 회전 버튼
- **신 옵션 B2**: 6자리 유지 + 30일 자동 회전 Cron
- **권장**: B1 (학급 단위는 유출 임팩트가 크므로 엔트로피 우선 + 교사 통제권 유지)

### 후보 C — brute-force 정책
- **현**: IP 5회/15분 + 코드 10회 실패 즉시 만료
- **신 옵션 C1**: IP 5회/15분 유지 + **코드 자체 만료 트리거 제거** (학급 전체 봉쇄 방지) + 교사 알림만 발송
- **신 옵션 C2**: 코드당 실패 임계 상향(예: 50회) + 임계 초과 시 교사에 "의심 시도" 알림
- **권장**: C1 + C2 혼합 (코드 자동 만료 제거, 교사 수동 회전 + 알림)

### 후보 D — revoke_reason 사유 코드 유니언
- **현**: `teacher_action` | `self_withdraw`
- **신 옵션 D1**: 위 2개 + `rejected_by_teacher` + `auto_expired_pending`
- **신 옵션 D2**: D1 + `code_rotated` (학급 코드 회전 시 해당 코드로 매칭한 pending 일괄 처리)
- **권장**: D2 (회전 시나리오까지 포괄)

### 후보 E — 교사 UI 배치
- **현**: 학생 카드 드롭다운 "학부모 초대" + 학급 설정 "학부모 액세스" 탭(리스트)
- **신 옵션 E1**: 학급 설정 "학부모 액세스" 탭 내부에 **(a) 초대 코드 섹션 (b) 승인 대기 인박스 (c) 연결된 학부모 리스트** 3-섹션 구조
- **신 옵션 E2**: 별도 "학부모 관리" 최상위 메뉴 신설
- **권장**: E1 (단일 탭 3-섹션, 기존 IA 최소 변경)

### 후보 F — 명단 표시 범위 (매칭 신청 시 학부모가 보는 학급 학생 명단)
- **현**: 해당 없음 (신규 결정)
- **신 옵션 F1**: 이름만 (본명, 자녀 본명 노출 기존 정책 승계)
- **신 옵션 F2**: 이름 + 출석번호
- **신 옵션 F3**: 이름 + 사진(프로필)
- **권장**: F1 (유출 면적 최소화, 자녀 식별에는 이름만으로 충분, 동명이인은 출석번호 토글로 확장 가능)

### 후보 G — pending 상한 & 자동 만료 TTL
- **현**: 해당 없음
- **신 옵션 G1**: pending TTL 7일, 학부모당 동시 pending 3건 상한
- **신 옵션 G2**: pending TTL 48h, 학부모당 pending 5건 상한
- **권장**: G1 (교사 업무 주기 = 주 단위 현실 반영)

### 후보 H — 승인 SLA
- **현**: 해당 없음
- **신 옵션 H1**: 권고 24h, hard SLA 없음(교사 주의 배지만)
- **신 옵션 H2**: 권고 24h + 72h 초과 시 학급장·관리자 에스컬레이션 배지
- **권장**: H1 (교사 자율, v1 단순성)

### 후보 I — 동일 자녀 복수 학부모 중복 신청 처리
- **현**: `@@unique([parentId, studentId])` 로 중복 방지(수용 단위)
- **신 옵션 I1**: 기존 유지(동일 parent가 같은 자녀 재신청 불가) + 다른 parent는 독립 신청 허용
- **신 옵션 I2**: 자녀당 active 학부모 상한(예: 4명 = 부/모/조부/조모) 도입
- **권장**: I1 (현 5명 상한은 학부모 기준, 자녀 기준 상한 없음 유지 — v1 단순성)

### 후보 J — Free/Pro 발급 한도 재정의
- **현**: Free 자녀당 2 / Pro 자녀당 5 (L61~L62)
- **신 옵션 J1**: 폐지 (학급 코드는 교사 단위, 학부모 구독이 학교 단위와 무관)
- **신 옵션 J2**: 학급당 승인 가능 학부모 수 (Free 학급당 20 / Pro 학급당 무제한)
- **신 옵션 J3**: 학부모 Pro만 주간 이메일 수신 (현 Pro tier 혜택 유지) + 발급 한도 개념 폐지
- **권장**: J3 (tier 연계는 이메일 기능에만 집중, 발급 한도 폐지)

### 후보 K — pending 상태 세션/BoardMember 발급 시점
- **현**: 해당 없음
- **신 옵션 K1**: ParentSession은 signup 시점 발급 + BoardMember는 approve 시점 발급(분리)
- **신 옵션 K2**: 둘 다 approve 시점으로 통일
- **권장**: K1 (학부모가 pending 상태 조회·취소를 위해 세션 필요)

**총 11개** 후보(A~K) 도출, 요구 최소 5개 상회.

---

## 9. 유지되어야 할 결정 (본 변경과 무관 — 변경 금지 명시)

target.json `out_of_scope` 기반 + 감사 결과 보강:

| 영역 | 유지 결정 |
|---|---|
| 인증 수단 | 매직 링크 이메일 OTP 전용, 유효 15분 |
| 세션 | 7일 TTL + 재인증 시 이메일만 |
| 학부모 상한 | 학부모 1인당 자녀 5명 |
| 격리 정책 | API 필터링(1차) + DOM 마스킹(보조) + RLS(3중), parentA→parentB 404, 타 학생 403 |
| 탈퇴 | Soft delete 고정 + 90일 익명화 + hard delete 금지 |
| 주간 이메일 | 월 09:00 KST + 활동 0건 스킵 + BCC 금지 개별 발송 |
| Revoke SLA | ≤ 60s (SWR 60s 폴링 기반) |
| 성능 예산 | TTI < 2s LTE / < 3s 3G, 첫 뷰포트 < 500KB, 썸네일 < 200KB |
| iframe 금지 | proxy thumbnail만 |
| WebSocket 비활성 | SWR polling 60s |
| 자녀 범위 매트릭스 (§5) | 매칭 활성화 이후 열람 범위 — 완전 유지 |
| Crockford 규칙 | O/0·I/1·L 제외 대문자, CSPRNG — 길이만 재검토 |

---

## 10. 새로 필요한 결정 항목 (delta가 다룰 대상 — 집약)

| # | 결정 항목 | 후보 참조 | 우선순위 |
|---|---|---|---|
| N1 | 학급 코드 엔티티 구조 (`ClassInviteCode` 신설 vs `ParentInviteCode.classroomId` 전환) | A | P0 |
| N2 | 학급 코드 자릿수·TTL·maxUses·회전 정책 | B | P0 |
| N3 | 학급 코드 brute-force 방어 재설계 (봉쇄 위험 제거) | C | P0 |
| N4 | `ParentChildLink.status` 유니언 확장 + 전이 머신 | 2.1 | P0 |
| N5 | 신규 감사 필드 (`requestedAt`, `approvedAt`, `approvedById`, `rejectedAt`, `rejectedById`, `rejectedReason`) | 2.1 | P0 |
| N6 | `revokedReason` 유니언 확장 (`rejected_by_teacher`, `auto_expired_pending`, 선택적 `code_rotated`) | D | P0 |
| N7 | 신규 API 엔드포인트 3종 명세 (`/signup`, `/match/code`, `/match/request`) | 3.1 | P0 |
| N8 | `parentAuthOnlyMiddleware` 도입 vs `parentScopeMiddleware` 분기 처리 | §4 | P1 |
| N9 | 교사 UI 3-섹션 배치 (`invite-code` / `pending-inbox` / `linked-list`) | E | P1 |
| N10 | 매칭 신청 시 학급 학생 명단 표시 범위 (이름만/출석번호/사진) | F | P1 |
| N11 | pending 상한·TTL·학부모당 동시 pending 수 | G | P1 |
| N12 | 승인 SLA 문구·에스컬레이션 여부 | H | P2 |
| N13 | Free/Pro 발급 한도 재정의 또는 폐지 | J | P1 |
| N14 | pending 상태 세션/BoardMember 발급 시점 정책 | K | P1 |
| N15 | 학급 코드 유출 시 회전 절차(SOP) + 사칭 시도 감지(다수 거부 신청 패턴) 알림 | 신규 리스크 | P2 |
| N16 | `/parent/pending` 상태 화면 UX + HTTP 응답 방식(401 vs 200 with payload flag) | 1.2-c,d | P1 |
| N17 | 90일 내 재가입 시 학급 코드 재입력 경로 (§1.4-d 대체) | 1.4-d | P2 |
| N18 | PV 작업 카드 재구성 (PV-1·2·3·8·12 개정 + PV-13~16 신설) | §6 | P0 |
| N19 | 공수 재산정 (기존 27일 → 약 33~34일) | §6 | P1 |

---

## 11. 감사 요약

- **✅ 유지** 결정: 1.1-f, 1.2-e·f·g·h·j, 1.3-b·c·d·g, 1.4-a·c·f, Parent·ParentSession 모델, RLS 기본 단방향성, 자녀 범위 매트릭스(§5) 전체, parentScopeMiddleware 기본 구조, PV-4·7·10 작업 카드
- **⚠️ 재검토** 결정: 1.1-b·d·e·g·h, 1.2-a·b·c·d·i, 1.3-a, 1.4-b·d·e, ParentSession 발급 시점, RLS status 필터, parentScopeMiddleware의 children 정의, PV-5·6·9·11 작업 카드
- **❌ 변경** 결정: **1.1-a·c, 1.3-e·f, ParentInviteCode 모델 전체, 페어링 시퀀스 §3.1 전면, PV-1·2·3·8·12 작업 카드**
- **🆕 신규 필요** 결정: 11개(N1~N19 중 P0 = 7건)
- **뒤집힐 후보**: 11개 (A~K) 도출 — delta 필수 의사결정 대상

**다음 phase2 delta**가 다룰 범위: N1~N19 전 항목 + PV 작업 카드 재구성 + 시퀀스 다이어그램 재작성 + 공수 재산정.
