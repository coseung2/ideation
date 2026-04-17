# Aura-board 학부모 읽기 전용 뷰어 액세스 로드맵

> 작성일: 2026-04-12 (v1) → **2026-04-13 v2 갱신 (학급 코드 + 셀프매칭 + 교사 승인)**
> Seed (active): `seed_6d7077aac472` (task `2026-04-13-parent-class-invite-refine`, parent_seed_id=`seed_37b35654542f`)
> Seed (archived): `seed_37b35654542f` — v1 학생별 코드 모델, v2에 의해 supersede (읽기 전용 보존)
> Interview (v2): `interview_20260413_075525` (ambiguity 0.10)
> Interview (v1): `interview_20260412_111153` (ambiguity 0.074 / phase3 측정 0.15)
> 관련 로드맵:
> - `ideation/plans/drawing-board-library-roadmap.md` (StudentAsset 자녀 열람 범위)
> - `ideation/plans/plant-journal-roadmap.md` (PlantObservation 자녀 열람 범위)
> - `ideation/plans/event-signup-roadmap.md` (EventBoard 자녀 학급 범위)
> - `ideation/plans/breakout-room-roadmap.md` (BreakoutAssignment 자녀 세션 범위)
> - `ideation/plans/tablet-performance-roadmap.md` (T0-④ 이미지 파이프라인 — thumbnail presigned URL 승계)

---

## 0. 핵심 명제

> **스마트폰 포트레이트 PWA(`/parent/*`)에서 학부모가 학급 단위 Crockford Base32 8자리 코드로 가입한 뒤, 학급 학생 명단(반·번호+이름 마스킹)에서 자녀를 셀프매칭으로 신청하고, 교사 승인을 통과한 경우에 한해 자녀의 그림보드·식물관찰일지·행사 보드·Breakout·숙제만 읽기 전용으로 열람한다. 타 학생 PII는 API 필터링 1차 + DOM 마스킹 보조 + RLS 정책 3중으로 완전 격리한다.**

Seed 1·3·4·6에서 개별적으로 "학부모 열람" 범위를 단편적으로 언급했던 내용을 본 로드맵이 **single source of truth**로 통합한다. 각 feature 로드맵은 본 문서의 "자녀 범위 매트릭스"(§5)를 단일 참조한다.

> **v2 변경 트리거**: 학생별 개별 코드 발급(v1)은 학생 수에 비례한 교사 운영 부담을 유발했다. v2에서는 학급 단위 단일 코드 + 학부모 셀프 온보딩 + 교사 승인 게이트로 전환해 발급 부담을 학생 수 → 1로 줄이고, 사칭 차단은 승인 게이트로 유지한다. v1은 미배포 상태였으므로 데이터 마이그레이션 없이 완전 폐기 후 즉시 전환한다.

---

## 1. 확정 결정 (v2 phase3 72건 요약 — seed_6d7077aac472)

### 1.1 페어링 · 인증 (v2)

| 결정 | 내용 |
|---|---|
| 페어링 단위 | **학급 1 : 코드 1** — `ClassInviteCode` 엔티티 신설. 학생별 코드 발급(v1) **폐기** (D-01·D-02) |
| 코드 포맷 | **Crockford Base32 8자리** (32⁸ ≈ 10¹²). O/0·I/1·L 제외, 대문자 고정. `crypto.randomBytes` CSPRNG (D-09·D-15) |
| 코드 수명 | **학기말 자동 만료** + 교사 수동 회전 버튼 + 무제한 `maxUses` (D-10) |
| 코드 brute-force 방어 | **3축 rate limit**: IP 5회/15분 잠금 + 코드당 50회/일 + **학급당 100회/일** (명단 조회와 풀 공유). 코드 자동 만료(실패 10회) 트리거 **제거** — 학급 단위에서는 DoS 벡터 (D-11·D-12·D-13) |
| 교사 회전 알림 | 거부율 임계 초과 시 교사 알림 배지만 v1 제공. 세부 SOP는 운영 단계 문서 이관 (D-14·D-48) |
| 최초 인증 | **매직 링크(이메일 OTP) 전용 v1**. 비밀번호·Kakao OAuth는 v2+ 파킹. 유효시간 15분 (D-38) |
| 세션 | `ParentSession` 토큰, 만료 **7일**. 만료 후 재인증은 이메일만 (학급 코드 재입력 불필요) (D-38) |
| ParentSession 발급 시점 | **signup 시점** (pending 상태 조회용). BoardMember는 **approve 시점** 발급(RLS 오염 방지) (D-25·D-26) |
| pending HTTP 응답 | **200 OK + `{"status":"pending"}` payload flag** → 클라이언트 `/parent/pending` 렌더 (D-27) |
| 동일 이메일 재가입 | 90일 내 재가입은 **학급 코드 재입력 + 재승인**으로 치환 (v1의 학생별 코드 재발급 경로 폐기) (D-47) |
| 학부모 1인당 자녀 수 | **5명 상한** (tier 무관). 다자녀 + 친척 돌봄 현실 반영 (유지) |

### 1.2 상태 전이 · Revoke · 격리 (v2)

| 결정 | 내용 |
|---|---|
| `ParentChildLink.status` 유니언 | **4-value**: `"pending" \| "active" \| "rejected" \| "revoked"` (D-03) |
| 상태 전이 머신 | `(none) → pending` (학부모 signup + 자녀 선택) → `active` (교사 approve) \| `rejected` (교사 reject / Cron auto-expire / code rotate) → `active → revoked` (교사 revoke / 학년 종료) (D-04) |
| `revokedReason` enum (확장) | **신규 4종 추가**: `rejected_by_teacher` + `auto_expired_pending` + `code_rotated` (D-05) + `classroom_deleted` (D-53, 학급 삭제 cascade). 기존 유지: `teacher_revoked` / `year_end`(수동 사유로만 존속, 자동 Cron 미구현 — D-56) / `parent_self_leave` |
| `rejectedReason` enum (신규) | `wrong_child` \| `not_parent` \| `other` (자유 텍스트 없음, D-06). 능동 거부 시 교사가 드롭다운 선택 (D-28) |
| 감사 필드 (신규 6종) | `requestedAt` · `approvedAt` · `approvedById` · `rejectedAt` · `rejectedById` · `rejectedReason` (D-07) |
| Revoke SLA | **≤ 60초** (SWR 60s 폴링 주기에 얹음). 즉시(< 1s)는 Redis 블랙리스트 필요 → v2 파킹 |
| Revoke 경로 | 교사 1-click (학급 설정 → 학부모 액세스 탭) / 학부모 자발 탈퇴 / **학급 코드 회전 시 pending 일괄 rejected** (D-39) / **Classroom 삭제 시 해당 학급 전 active 링크 cascade revoke** (D-52, `classroom_deleted`) |
| Active 링크 수명 정책 (D-51) | **학급(Classroom) 존재 기간과 1:1** — 시간 기반 자동 만료 없음. 담임이 학급을 삭제할 때까지 유지 |
| Classroom 삭제 UX (D-55) | 삭제 확인 모달에 "이 작업은 학부모 N명의 액세스를 해제합니다" 경고 + 학급명 재입력 확인 |
| Cascade 학부모 안내 (D-54) | `"[Aura-board] 연결된 학급이 종료되어 액세스가 해제되었습니다"` (교사 이름·사유 비노출, D-31 격리 정책 승계) |
| Rotate vs active | 코드 회전 시 **pending만 일괄 rejected**. 기존 active 링크는 **유지** (active는 링크 객체이며 코드 참조 X) (D-40) |
| 세션 차단 | `ParentSession.revokedAt IS NOT NULL` 또는 `Parent.deletedAt IS NOT NULL` → 모든 `/parent/*` API 미들웨어 **401**. pending 상태는 예외(200 + payload flag) |
| 클라이언트 | 401 수신 시 자동 로그아웃 + "접근이 해제되었습니다" 전환. pending 응답 수신 시 `/parent/pending` 렌더 |
| 교사 UI 문구 | "revoke 후 최대 1분 내 차단됩니다" 기대치 명시 |
| 타 학생 API 격리 | parent 토큰으로 타 학생 `studentId` 직접 호출 → **403** (E2E 필수). active 링크 없는 학생은 모두 거부 |
| 타 학부모 데이터 격리 | parentA 토큰으로 parentB의 `ParentChildLink` 조회 → **404** (존재 불인식, RLS `parent_id = auth.parent_id()` 단방향) |
| DOM 마스킹 | API 응답 필터링(1차) + DOM 마스킹(보조). 썸네일 URL은 **presigned 또는 RLS-scoped query** (URL 추측 차단) |
| 자녀 본명 노출 | **승인(active) 이후에만** 본명 노출 (가족 맥락). pending 동안은 자녀 본인 콘텐츠도 열람 불가 |
| 학급 명단 마스킹 (§1.5에서 상술) | 셀프매칭 화면에서 학생 명단은 `반+번호+성 + "O" + 끝글자` 마스킹 형식("3반 12번 김O민"). 프로필 사진 비노출 (D-16·D-17) |
| 학부모 간 격리 | 동일 자녀의 부/모/조부모는 상호 이름·이메일 비노출. 주간 이메일도 **개별 발송(BCC 금지)**. 교사 UI에서만 전체 학부모 이름 노출 |
| 거부 이메일 격리 | 거부/만료 알림 이메일에 **교사 이름·이메일·전화번호 미노출** ("담임 교사" 표현만 사용. 학교 대표 연락처만) (D-31) |

### 1.3 알림 · Tier (v2 재정의)

| 결정 | 내용 |
|---|---|
| 학부모 알림 범위 | **인앱 배지(Free/Pro 공통) + 주간 이메일 요약(Pro 전용)**. 실시간 push·카톡 알림톡은 v2+ |
| 주간 이메일 | 매주 월요일 00:00 UTC (= KST 09:00) Vercel Cron. 활동 0건 주는 **발송 스킵** (`lastSummarySkippedAt` 기록) |
| 이메일 콘텐츠 | 헤더 → 집계("그림 N건·관찰 N건·행사 N건·숙제 피드백 N건", 0건 항목 숨김) → 대표 썸네일 1장(presigned 7일) → 교사 피드백 1~3 bullet → CTA 딥링크 |
| 이메일 수신 거부 | `Parent.emailSummaryOptOut: Boolean @default(false)` — 계정 삭제 없이 이메일만 중단 |
| 교사 승인 인박스 알림 | **D+0 배지 / D+3 리마인더 이메일 / D+6 최종 경고 / D+7 자동 만료 요약** (D-22·E-04) — 채널: 인박스 배지 + 이메일만 (푸시/SMS 미사용) |
| **발급 한도 (v2 재정의)** | "자녀당 N 학부모" 개념은 학급 코드 체제에서 **의미 붕괴 → 폐지** (D-50 선행: J3). Pro 혜택은 **주간 이메일 수신(월 09:00 KST)** 전용으로 재집중 |
| Free tier | 인앱 배지만. 학급 코드 발급/사용 자체는 제한 없음 |
| Pro tier | 주간 이메일 + 향후 푸시 |
| v1 범위 | **순수 read-only**. 응원 이모지·좋아요·댓글 모두 v2+ 파킹 |
| 교사 자유 메시지 입력 | **v1 미제공** (v2 파킹) — 거부 이메일에 커스텀 문구 첨부 불가 (D-29) |
| 에스컬레이션 경로 | **v1 미제공** (1인 개발자 운영) — v2 파킹 (D-23) |

### 1.4 탈퇴 · 감사 (v2)

| 결정 | 내용 |
|---|---|
| 탈퇴 정책 | **Soft delete 고정** (hard delete 금지). 감사 로그 법적 증빙 + GDPR/개보법은 익명화로 충족 |
| 탈퇴 시퀀스 | 요청 → `Parent.deletedAt = now()` + 모든 `ParentSession` 즉시 무효 → `ParentChildLink.status="revoked"`, `revokedReason="parent_self_leave"`, `deletedAt=now()` |
| 90일 익명화 | Cron 자동: `email → SHA-256 hash` 또는 `deleted-{id}@anonymized.local`, `displayName = "탈퇴한 학부모"`. `ParentChildLink`는 감사 보존 |
| 90일 내 재가입 | 동일 이메일로 계정 복구(`deletedAt=null`). 기존 링크는 **복원 X** — **학급 코드 재입력 + 재승인** 필요 (D-47) |
| 감사 필드 (v2 확장) | **승인자·발급자 분리**: `ClassInviteCode.issuedById`(코드 발급 교사) + `ParentChildLink.approvedById`(승인 교사) + 신규 6종 (requestedAt·approvedAt·approvedById·rejectedAt·rejectedById·rejectedReason) + `revokedAt` + `revokedReason`(teacher_revoked / year_end / parent_self_leave / rejected_by_teacher / auto_expired_pending / code_rotated) |
| 교사 UI 표시 | 탈퇴 학부모는 "연결 해제됨 (탈퇴)"로 표시, 이름·이메일 비표시 |
| 거부 이메일 쿨다운 | 동일 학부모 이메일 거부 **3회 초과 시 24시간 쿨다운** (사칭 시도 방어, D-33·E-01) |

---

### 1.5 승인 게이트 흐름 (v2 신규)

v2의 핵심 차이. 학부모 셀프 온보딩 + 교사 승인으로 학생별 코드 발급 부담을 제거하면서 사칭 차단을 유지한다.

#### 1.5.1 셀프매칭 흐름

```
[교사] 학급 설정 → "학부모 액세스" 탭 → 초대 코드 섹션 → 현재 학급 코드 확인(QR·링크 복사)
  ↓ 코드를 학부모에게 공유 (알림장·카톡·인쇄 등)
[학부모] /parent/enter → 학급 코드 입력 + 이메일 입력
  ↓ 학급 코드 검증(IP 5회/15분, 코드 50회/일, 학급 100회/일)
[학부모] 매직 링크 이메일 수신 → 15분 내 클릭
  ↓ ParentSession 발급 (signup 시점)
[학부모] 학급 학생 명단 화면 → 반·번호 정렬, 이름 마스킹("3반 12번 김O민")
  ↓ 자녀 선택 (1건) + 매칭 신청
[시스템] ParentChildLink INSERT {status: "pending", requestedAt: now()}
  ↓ 동시 pending 3건 상한 검증
[학부모] /parent/pending 화면 (200 OK + {"status":"pending"})
[교사] 학급 설정 → "학부모 액세스" 탭 → "승인 대기 인박스" 배지 +1
```

#### 1.5.2 학생 명단 노출 범위 (D-16·D-17)

| 항목 | 규칙 |
|---|---|
| 표시 필드 | 반(`class_no`) · 번호(`student_no`) · 마스킹 이름(`masked_name`) |
| 마스킹 규칙 | 성 + "O" + 끝글자 (예: "김O민", "박O") — 2자 이름은 "김O" |
| 비노출 필드 | 본명 전체 · 프로필 사진 · 전화번호 · 주소 · 생년월일 · 모든 PII |
| 정렬 | 반 오름차순 → 번호 오름차순 |
| 필터 | 현재 학급만 (타 학급 학생 불가시) |
| Rate limit | 학급당 조회 **100회/일** (코드 brute-force 방어와 동일 풀) |
| pending 후 노출 | 승인(active) 이전에는 자녀 본명·자녀 본인 콘텐츠 열람 불가 |

#### 1.5.3 승인 인박스 · SLA · 자동 만료

| 항목 | 값 | 근거 |
|---|---|---|
| pending TTL | **7일** (D+7 자동 만료, Cron 강제) | D-19 |
| 학부모당 동시 pending 상한 | **3건** | D-20 |
| 권고 응답 SLA | **24h** (hard SLA 없음, 교사 안내 문구) | D-21 |
| D+0 배지 | 정상(회색) — 교사 승인 인박스 뱃지 +1 | D-22 |
| D+3~5 배지 | 주의(노랑) |
| D+3 리마인더 | 교사 이메일 `"[Aura-board] N명의 학부모가 승인 대기 중입니다"` | D-22·E-04 |
| D+6 경고 | 배지(빨강) + 교사 이메일 `"[Aura-board] 24시간 후 자동 만료 예정 (N건)"` | D-22 |
| D+7 만료 실행 | Vercel Cron (KST 02:00, UTC 17:00, 일 1회)로 pending 7일 초과 스캔 → `auto_expired_pending` rejected 처리 | D-24·E-03 |
| D+7 교사 요약 이메일 | 만료 실행 후 `"[Aura-board] N건이 자동 만료되었습니다"` 요약 발송 | D-22 |

#### 1.5.4 거부 사유 · 안내 톤 (교사 능동 거부)

교사는 pending 항목에 대해 **사전 정의 사유 3종 드롭다운** 중 하나를 선택해 거부한다 (D-28). 자유 텍스트 입력은 v1 미제공 (D-29, v2 파킹).

| `rejectedReason` | 학부모 이메일 본문 문구 |
|---|---|
| `wrong_child` | "선택한 학생이 해당 학부모의 자녀가 아닌 것으로 확인되었습니다. 자녀 정보를 다시 확인 후 재신청해 주세요." |
| `not_parent` | "신청자가 법적 보호자가 아닌 것으로 확인되었습니다. 보호자 본인이 직접 신청해 주세요." |
| `other` | "승인이 어려워 거부되었습니다. 자세한 사유는 학교로 문의해 주세요." |

**공통 규칙** (D-31·D-32·D-33):
- 교사 이름·이메일·전화번호 **미노출** (학교 대표 연락처만 표기)
- 재신청 deep link 포함 (signup 초기화 flow)
- 선택된 사유 문구만 본문에 삽입 (자유 메시지 없음)
- 동일 이메일 거부 누적 **3회 초과** 시 24시간 재신청 차단

#### 1.5.5 자동 만료 이메일 (D+7)

고정 문구 (D-30):

> "승인 요청이 7일간 처리되지 않아 자동 만료되었습니다. 다시 신청하시거나 담임 교사에게 직접 문의해 주세요."

재신청 deep link 포함. 교사 개입 없음.

#### 1.5.6 code_rotated 처리 (D-39·D-40)

교사가 학급 초대 코드를 회전하면:
- 해당 학급의 **pending** 건 일괄 `rejected` (reason=`code_rotated`) 처리
- **active** 링크는 **유지** (링크 객체는 코드 참조가 아닌 parentId×studentId 기반)
- pending 학부모에게 `code_rotated` 사유 이메일 발송 (재신청 deep link 포함)

### 1.6 교사 UI (v2) — 학급 설정 "학부모 액세스" 탭 (D-34·D-35)

**위치**: 학급 설정 > "학부모 액세스" 탭 (v1의 학생 카드 드롭다운 "학부모 초대" 항목은 **제거** — D-35)

**3-섹션 배치**:

| 섹션 | 위젯 | 배지 |
|---|---|---|
| **(a) 초대 코드** | 현재 코드 표시 · QR/링크 복사 · 회전 버튼 · 회전 히스토리 | — |
| **(b) 승인 인박스** | pending 리스트(D+N 일자 배지) · 일괄 승인 · 개별 승인/거부(사유 드롭다운) · 검색 | D+0~2 회색 / D+3~5 노랑 / D+6 빨강(+24h 만료 예고) |
| **(c) 연결된 학부모** | 학부모-자녀 쌍 목록 · revoke 버튼 · 최근 접속일 | — |

**v1 대비 제거**:
- 학생 카드 드롭다운의 "학부모 초대" 항목 (D-35)
- 학생별 코드 발급 UI (v1 경로는 읽기 전용 history 페이지로만 잔존, 신규 발급 불가)

---

## 2. 데이터 모델 (Prisma 최종 확정)

```prisma
model Parent {
  id                   String   @id @default(cuid())
  email                String   @unique
  displayName          String
  emailSummaryOptOut   Boolean  @default(false)
  revokedAt            DateTime?   // 교사 철회 (관리상 이 필드는 링크 수준이 실질이나 조회 성능상 유지)
  deletedAt            DateTime?   // 학부모 자발 탈퇴 (soft delete)
  createdAt            DateTime @default(now())
  updatedAt            DateTime @updatedAt

  childLinks ParentChildLink[]
  sessions   ParentSession[]

  @@index([email])
  @@index([deletedAt])
}

model ParentChildLink {
  id             String   @id @default(cuid())
  parentId       String
  studentId      String                              // RLS 필터 기준 키
  status         String   @default("pending")        // "pending" | "active" | "rejected" | "revoked"

  // v2 감사 필드 (D-07)
  requestedAt    DateTime @default(now())            // signup + 자녀 선택 시점
  approvedAt     DateTime?                            // 교사 승인 시점
  approvedById   String?                              // 승인 교사 ID
  rejectedAt     DateTime?                            // 거부/만료/회전 시점
  rejectedById   String?                              // 거부 교사 ID (auto_expired_pending/code_rotated는 null)
  rejectedReason String?                              // "wrong_child" | "not_parent" | "other" (능동 거부만)

  // revoke 필드 (active → revoked)
  revokedAt      DateTime?
  revokedReason  String?                             // "teacher_revoked" | "year_end" | "parent_self_leave" | "rejected_by_teacher" | "auto_expired_pending" | "code_rotated" | "classroom_deleted"

  deletedAt      DateTime?                           // 학부모 탈퇴 시
  createdAt      DateTime @default(now())

  parent  Parent  @relation(fields: [parentId], references: [id], onDelete: Cascade)
  student Student @relation(fields: [studentId], references: [id], onDelete: Cascade)

  @@unique([parentId, studentId])                    // 동일 자녀에 부·모 각각 별도 승인 허용
  @@index([studentId])
  @@index([status])
  @@index([approvedById])
  @@index([requestedAt])                              // Cron D+7 스캔
}

// v2 신규 — v1 ParentInviteCode 완전 대체 (D-01·D-02·D-42)
model ClassInviteCode {
  id              String   @id @default(cuid())
  code            String   @unique                   // Crockford Base32 8자리 (D-09)
  classroomId     String
  issuedById      String                              // 발급 교사
  expiresAt       DateTime?                           // 학기말 자동 만료 (D-10). null이면 학기 설정에서 자동 계산
  maxUses         Int?     @default(null)             // 무제한 (D-10)
  usedCount       Int      @default(0)
  failedAttempts  Int      @default(0)                // 통계용 (자동 만료 트리거 X — D-13)
  rotatedAt       DateTime?                           // 교사 수동 회전 시점
  createdAt       DateTime @default(now())

  classroom Classroom @relation(fields: [classroomId], references: [id], onDelete: Cascade)

  @@index([classroomId])
  @@index([expiresAt])
}

model ParentSession {
  id          String   @id @default(cuid())
  parentId    String
  token       String   @unique
  expiresAt   DateTime                                 // 발급 + 7일
  revokedAt   DateTime?                                // revoke 즉시 SET
  createdAt   DateTime @default(now())
  lastSeenAt  DateTime @default(now())

  parent Parent @relation(fields: [parentId], references: [id], onDelete: Cascade)

  @@index([parentId])
  @@index([expiresAt])
}

model BoardMember {
  // ... 기존 ...
  role String   // "teacher" | "student" | "parent" — parent는 read-only
}
```

**RLS 정책 요약** (v2 갱신):
- `ParentChildLink` SELECT: `parent_id = auth.parent_id()` (단방향)
- `Parent` SELECT: `id = auth.parent_id()` (자기 레코드만)
- `ClassInviteCode` SELECT: 학부모는 검증 엔드포인트 경유만 (직접 조회 불가, 교사만 RLS 허용)
- 모든 콘텐츠 테이블(`StudentAsset`, `PlantObservation`, `EventSignup`, `BreakoutMembership`, `Submission`): `studentId ∈ parent.children WHERE status='active'` 조인 검증 (pending·rejected·revoked는 차단)

---

## 3. 페어링 흐름 (v2 시퀀스)

### 3.1 학급 코드 발급 → 학부모 signup → 셀프매칭 → 교사 승인

```
[교사] 학급 설정 → "학부모 액세스" 탭 → 초대 코드 섹션
  ↓
[POST /api/class-invite-codes { classroomId }]
  ↓ crypto.randomBytes → Crockford Base32 8자리
[ClassInviteCode INSERT {code, expiresAt=학기말, maxUses=null}]
  ↓
[교사 UI: 코드 + QR + 링크 복사 CTA]
  ↓ 학부모에 공유 (알림장·카톡·인쇄)
[학부모] /parent/enter → 학급 코드 + 이메일 입력
  ↓
[POST /api/parent/signup { code, email }]   (※ parentAuthOnlyMiddleware 적용)
  ↓ 검증: expiresAt > now
  ↓ IP rate limit 5회/15분 + 코드 50회/일 + 학급 100회/일
  ↓ 실패 시 코드 자동 만료 트리거 X (학급 단위 DoS 벡터 방지, D-13)
[매직 링크 이메일 발송 (Resend + React Email)]
  ↓ 유효 15분
[학부모] /parent/verify?token=...
  ↓
[ParentSession INSERT {expiresAt=now+7d}]   (※ signup 시점 발급, D-25)
  ↓ BoardMember는 아직 생성하지 않음 (RLS 오염 방지, D-26)
[학부모] /parent/match/select → 학급 학생 명단 조회
  ↓
[POST /api/parent/match/code { code } + GET /api/parent/match/students?classroomId=...]
  ↓ 응답: [{classNo, studentNo, maskedName}...] (반·번호 정렬, PII 제거)
[학부모] 자녀 1건 선택 → 매칭 신청
  ↓
[POST /api/parent/match/request { studentId }]
  ↓ 검증: parent의 동시 pending 3건 상한
[ParentChildLink INSERT {status: "pending", requestedAt: now()}]
  ↓
[응답 200 OK + {"status":"pending"}]        (D-27)
[학부모] /parent/pending 렌더
  ↓
[교사] 학급 설정 → "학부모 액세스" 탭 → 승인 인박스 배지 +1
  ↓ (D+0 배지 / D+3 이메일 리마인더 / D+6 경고 / D+7 자동 만료)
[교사] 승인 or 거부(사유 드롭다운)

  (a) 승인 경로:
    [POST /api/parent-child-links/:id/approve]
    [ParentChildLink.status="active", approvedAt=now(), approvedById=teacher.id]
    [BoardMember INSERT {role: "parent"}]    (※ approve 시점 생성, D-26)
    [학부모] SWR 폴링에서 200 수신 → /parent/home 자동 진입

  (b) 거부 경로:
    [POST /api/parent-child-links/:id/reject { reason: wrong_child|not_parent|other }]
    [ParentChildLink.status="rejected", rejectedAt=now(), rejectedById=teacher.id, rejectedReason=...]
    [revokedReason="rejected_by_teacher"]
    [학부모] 거부 이메일 수신 (교사 정보 비노출, 재신청 deep link 포함)

  (c) 자동 만료 경로 (Vercel Cron KST 02:00, 일 1회, D-24·E-03):
    [SELECT * FROM ParentChildLink WHERE status="pending" AND requestedAt < now()-7d]
    [UPDATE status="rejected", rejectedAt=now(), revokedReason="auto_expired_pending"]
    [학부모에 자동 만료 이메일 + 교사에 D+7 요약 이메일]

  (d) 코드 회전 경로 (D-39·D-40):
    [교사 "회전" 버튼 클릭]
    [ClassInviteCode.rotatedAt=now() + 신규 코드 생성]
    [해당 학급 pending 건 일괄 rejected (revokedReason="code_rotated")]
    [active 링크는 유지]
```

### 3.2 Revoke 흐름

```
교사 경로 (active → revoked):
[교사 학급 설정 "학부모 액세스" 탭 → 연결된 학부모 섹션]
  ↓
[POST /api/parent-child-links/:id/revoke { reason: "teacher_revoked" }]
  ↓
[ParentChildLink.status="revoked", revokedAt=now(), revokedReason="teacher_revoked"]
  ↓
[ParentSession 해당 parentId 전체 revokedAt=now() SET]
  ↓ ≤ 60s 내 SWR 폴링 도달
[/parent/* 미들웨어 401 → 클라이언트 자동 로그아웃]

학부모 자발 탈퇴:
[/parent/settings "계정 탈퇴"]
  ↓
[Parent.deletedAt=now() + 모든 ParentSession 무효 + 모든 링크 revokedReason="parent_self_leave" 처리]
  ↓
[90일 후 Cron anonymize-parent job 실행]
```

---

## 4. parent 계층 미들웨어 (v2 — 2종)

v2는 매칭 전/후 경로가 분리되므로 미들웨어를 2종으로 분기한다.

### 4.1 `parentAuthOnlyMiddleware` (v2 신규, D-37)

**위치**: `src/middleware/parentAuthOnlyMiddleware.ts`
**적용**: 매칭 전 엔드포인트 — `POST /api/parent/signup`, `POST /api/parent/match/code`, `GET /api/parent/match/students`, `POST /api/parent/match/request`, `/parent/pending`

**역할**: 세션 존재·유효성만 검증. `studentId ∈ parent.children` 전제 **없음** (아직 링크가 pending 또는 없음). `parentScopeMiddleware`로는 우회 불가(후자는 active 링크를 요구).

```ts
// 의사 코드
export async function parentAuthOnlyMiddleware(req) {
  const session = await getParentSession(req);
  if (!session || session.revokedAt || session.expiresAt < now()) return 401;

  const parent = await getParent(session.parentId);
  if (parent.deletedAt) return 401;

  req.parentId = parent.id;
  // children 조회 없음 — pending 포함 모든 링크 허용
}
```

### 4.2 `parentScopeMiddleware` (기존, v2 유지)

**위치**: `src/middleware/parentScopeMiddleware.ts`
**적용**: 매칭 후 자녀 콘텐츠 엔드포인트 — `/parent/child/*`, `/parent/home`, `/parent/settings` 등

```ts
// 의사 코드
export async function parentScopeMiddleware(req) {
  const session = await getParentSession(req);
  if (!session || session.revokedAt || session.expiresAt < now()) return 401;

  const parent = await getParent(session.parentId);
  if (parent.deletedAt) return 401;

  // studentId 포함 요청 검증 — status="active"만 통과
  const studentId = extractStudentId(req);  // path param, query, body
  if (studentId) {
    const link = await prisma.parentChildLink.findFirst({
      where: { parentId: parent.id, studentId, status: "active" }
    });
    if (!link) return 403;  // 타 학생 + pending/rejected/revoked 차단
  }

  req.parentId = parent.id;
  req.children = await getParentChildren(parent.id);  // status="active"만
}
```

**RLS 보완**: Prisma 쿼리에 `auth.parent_id()` 세팅 (supabase RLS) → 미들웨어 우회 시도 방어. RLS `status='active'` 조건 동시 적용.

**ESLint 룰** (v2 확장): `/parent/match/*` 엔드포인트는 반드시 `parentAuthOnly`로, `/parent/child/*`·`/parent/home`·`/parent/settings`는 `parentScope`로 래핑 — 혼용 금지.

---

## 5. Cross-cutting 자녀 범위 매트릭스 — single source of truth

**본 표가 학부모 열람 범위의 single source of truth이다. 각 feature 로드맵은 이 표를 참조한다.**

| Feature | Seed | 콘텐츠 | 학부모 열람 범위 | 서버 필터 조건 | DOM 마스킹 보조 | v2 파킹 |
|---|---|---|---|---|---|---|
| **그림보드 × 학생 라이브러리** | Seed 1 `seed_91bcd4e99efb` | `StudentAsset` | **자녀 본인 자산 전체** (isSharedToClass 무관, isPrivate 무관) | `StudentAsset.studentId ∈ parent.children` | 타 학생 attribution 필드 제거 | "자녀 포트폴리오 내보내기 PDF" (drawing-board §파킹) |
| **식물관찰일지** | Seed 4 `seed_a79a953f188c` | `PlantObservation` · `PlantObservationImage` · `StudentPlant` | **자녀 본인 식물 전체 단계·관찰** (isPrivate 무관) + 교사 코멘트 | `StudentPlant.studentId ∈ parent.children` → `PlantObservation` 조인 | 타 학생 반 공개 관찰 DOM 숨김 | 응원 이모지(plant §v2) |
| **행사 신청 보드** | Seed 3 `seed_43fdf181262f` | `Board(layout=event-signup)` · `Submission` | **자녀 학급 전체 이벤트 메타** (포스터·일정·설명) + **자녀 본인 Submission·피드백만** | EventBoard: `classroomId = child.classroomId` 허용. EventSignup 응답에서 `studentId ≠ child` 레코드 제거 | 참가자 명단·득점 필드 마스킹 | 학부모 대리 신청(event §미결) |
| **Breakout Room** | Seed 6 `seed_bb1d4eb1c442` | `BreakoutAssignment` · `BreakoutMembership` · Section/Card | **자녀 본인 세션 결과·제출물만**. 교사-pool 섹션 제외. peek-others 설정이어도 타 모둠 노출 X | `BreakoutMembership.studentId ∈ parent.children` → `session.studentId ∈ parent.children` 검증 | Section `role="teacher-pool"` 필터 | Jigsaw 양 모둠 열람(Seed 6 §파킹) |
| **숙제/카드 피드백** | 공통 Card·Submission | 자녀 본인 제출 + 교사 코멘트 | `Submission.userId = child.userId` 또는 `studentId ∈ parent.children` | 타 학생 제출 제거 | — |
| **과제 배부 보드** | Seed 11 `seed_38c34e91bf28` | `Board(layout=assignment)` · `AssignmentSlot` · Card · Submission · `returnReason` | **자녀 본인 slot + Card/Submission/반려 사유만**. 5×6 격자 자체 미노출 → 자녀 전용 **단일 카드 뷰**로 축약. 교사 가이드 텍스트(`assignmentGuideText`)는 viewer-readable | `AssignmentSlot.studentId ∈ parent.children` 필터. 타 slot·card·submission 응답에서 제거 | 격자 컴포넌트 자체 미렌더 (DOM 트리에 타 학생 노드 0개) | matrix/grid 뷰(Seed 11 §11 v2 파킹 — owner+데스크톱 전용 별도 라우트에서도 viewer 제외 유지) |
| **Quiz 점수** | Seed 5 (P1-④) | QuizScore | **v1 비노출** (주간 이메일·PWA 뷰 모두) | — | — | 사후 요약 리포트 v2+ |

**공통 원칙**:
1. **API 필터링 1차 방어선** (parentScopeMiddleware + Prisma where 절). DOM 마스킹은 보조일 뿐 신뢰하지 않는다.
2. **썸네일 URL은 presigned 또는 RLS-scoped query**. URL 추측 공격 차단. T0-④ 이미지 파이프라인 승계.
3. **타 학부모 식별 정보는 응답에 포함 불가**. 동일 studentId를 보는 다른 parentId 목록 노출 금지.
4. **privacy 토글 무관 — 자녀 본인 콘텐츠는 항상 열람**. (isPrivate=true인 자녀 관찰일지도 학부모는 본다. 단, 타 학생의 반 공개 자료는 학부모 뷰에서 제외.)
5. **v2 pending 동안 자녀 본인 콘텐츠도 열람 불가**. `ParentChildLink.status='active'`인 경우에만 §5 매트릭스가 활성화된다. pending·rejected·revoked는 403.

---

## 6. /parent/* PWA 구조

```
/parent/
├── enter/               — 학급 코드 입력 + 이메일 입력 (v2: QR 대신 코드 공유 링크 중심)
├── verify?token=...     — 매직 링크 수신 페이지 (ParentSession 발급 — signup 시점)
├── (authed-preActive)/   — parentAuthOnlyMiddleware 적용
│   ├── match/
│   │   ├── select       — 학급 학생 명단 (반·번호+마스킹 이름) + 자녀 선택 UI
│   │   └── pending      — /parent/pending (200 + {"status":"pending"} 수신 시 렌더)
│   └── rejected         — 거부 이메일 deep link 착지점 (재신청 CTA)
├── (authed-active)/      — parentScopeMiddleware 적용 (ParentChildLink.status='active' 필수)
│   ├── home             — 자녀 카드 N개 (최대 5) + 최근 활동 배지
│   ├── child/[id]/
│   │   ├── drawings     — 그림보드 자산 그리드
│   │   ├── plant        — 식물관찰 노선도 (읽기 전용)
│   │   ├── events       — 자녀 학급 이벤트 + 본인 신청 상태
│   │   ├── breakout     — 자녀 참여 세션 요약
│   │   ├── assignment   — 과제 배부 보드: 자녀 slot 단일 카드 + 교사 가이드 + 반려 사유 배너 (Seed 11)
│   │   └── homework     — 숙제 카드 + 교사 피드백
│   └── settings         — 이메일 수신 거부 · 탈퇴
└── manifest.json        — PWA 설치
```

**성능 예산 (seed constraints)**:
- TTI < 2s LTE / < 3s 3G
- 첫 뷰포트 < 500KB
- 썸네일 < 200KB (T0-④ 파이프라인 경유)
- 뷰포트 320~430px iOS Safari / Android Chrome
- **iframe 금지** — Canva oEmbed는 proxy thumbnail만
- WebSocket 비활성 — **SWR polling 60s**

---

## 7. 작업 분할 (v2 — PV-1 ~ PV-16, 총 33~34일)

**v2 변경 영향** (D-49): PV-1·2·3·5·8·12 **개정** + PV-4·6·7·9·10·11 **유지** + PV-13·14·15·16 **신설** (총 16개 카드, 27일 → 33~34일).

| # | 작업 | v2 상태 | 의존 | 공수 |
|---|---|---|---|---|
| **PV-1** | 스키마 마이그레이션 — `Parent` / `ParentChildLink`(status 4-value + 감사 6종) / `ClassInviteCode`(v1 `ParentInviteCode` 대체) / `ParentSession` + `BoardMember.role` 유니언에 `"parent"` 추가 + RLS(`status='active'`) | **개정** | — | 2일 |
| **PV-2** | Crockford Base32 **8자리** 코드 생성기 + `POST /api/class-invite-codes` (교사) + 학급 설정 "학부모 액세스" 탭 내 초대 코드 섹션(회전·QR·링크 복사) | **개정** | PV-1 | 2일 |
| **PV-3** | `POST /api/parent/signup` — 학급 코드 검증 + 3축 rate limit (IP 5회/15분 · 코드 50회/일 · 학급 100회/일) + 매직 링크 발송(React Email + Resend, 15분). 코드 자동 만료 트리거 제거 | **개정** | PV-1, PV-2 | 2일 |
| **PV-4** | `GET /parent/verify?token=...` — 매직 링크 소비 + `ParentSession` 발급(signup 시점, D-25) + 쿠키 세팅 → `/parent/match/select` 리다이렉트 | 유지(경로 변경) | PV-3 | 1일 |
| **PV-5** | `parentAuthOnlyMiddleware`(신규 매칭 전) + `parentScopeMiddleware`(매칭 후, `status='active'` 조건 추가). ESLint 룰 2종 분기 | **개정** | PV-1 | 2일 |
| **PV-6** | `/parent/*` PWA 쉘 — manifest.json + match/select + match/pending + home(active) + child 라우트 4종 + SWR 60s 폴링 | 유지(+라우트) | PV-5 | 4일 |
| **PV-7** | 자녀 범위 서버 필터 일괄 적용 — §5 매트릭스 구현. `status='active'` 조건 필수 | 유지 | PV-5, PV-6 | 4일 |
| **PV-8** | 교사 관리 UI — 학급 설정 "학부모 액세스" 탭 **3-섹션** (a)초대 코드 (b)승인 인박스 (c)연결된 학부모. 학생 카드 드롭다운 "학부모 초대" **제거** (D-35) | **개정** | PV-1 | 3일 |
| **PV-9** | Revoke SLA ≤ 60s 구현 — 교사 revoke → `ParentSession.revokedAt` 일괄 SET + 클라이언트 401 시 자동 로그아웃 + "접근이 해제되었습니다" 전환 | 유지 | PV-5, PV-8 | 1일 |
| **PV-10** | 주간 이메일 요약 (Pro 전용) — Vercel Cron 월 09:00 KST + 집계 + 대표 썸네일 presigned 7일 + 교사 피드백 bullet + CTA 딥링크 + 활동 0건 스킵 | 유지 | PV-7 | 3일 |
| **PV-11** | 학부모 탈퇴 플로우 — `/parent/settings` "계정 탈퇴" + soft delete 즉시 반영 + 90일 익명화 Cron (`anonymize-parents` job) | 유지 | PV-4, PV-9 | 2일 |
| **PV-12** | E2E 보안 게이트 테스트 — 403(타 학생 API)·404(타 학부모 링크)·≤60s revoke·rate limit(IP 15분·코드 50/일·학급 100/일)·썸네일 직접 접근 403·90일 익명화 + **pending 차단**·**회전 시 pending 일괄 rejected**·**자동 만료 Cron** | **개정** | PV-1 ~ PV-11, PV-13~16 | 3일 |
| **PV-13 (신규)** | 학부모 셀프매칭 학생 명단 API + 화면 — `GET /api/parent/match/students?classroomId=...` (마스킹 이름 + 반·번호 정렬) · `POST /api/parent/match/request` (pending 생성, 3건 상한) · `/parent/match/select` UI | **신규** | PV-4, PV-5 | 3일 |
| **PV-14 (신규)** | 교사 승인 인박스 UI + API — `POST /api/parent-child-links/:id/approve`·`/reject { reason }` · pending 리스트 + D+N 배지(회색/노랑/빨강) + 일괄 승인 + 개별 승인/거부(드롭다운 3종) + 검색 | **신규** | PV-8 | 3일 |
| **PV-15 (신규)** | pending 자동 만료 Cron + 교사 알림 스케줄 — Vercel Cron KST 02:00 일 1회 pending 7일 초과 스캔 → `auto_expired_pending` rejected + D+3 리마인더 · D+6 경고 · D+7 요약 이메일 + 거부/만료 학부모 이메일 (교사 정보 비노출 + 재신청 deep link + 3회 초과 24h 쿨다운) | **신규** | PV-14 | 3일 |
| **PV-16 (신규)** | 학급 코드 회전 기능 — 교사 "회전" 버튼 → 신규 코드 발급 + 기존 코드 `rotatedAt` SET + 해당 학급 pending 일괄 `code_rotated` rejected + 학부모 재신청 이메일 발송 | **신규** | PV-2, PV-14 | 1일 |
| **PV-17 (신규)** | **Classroom 삭제 cascade revoke** — 학급 삭제 확인 모달(학급명 재입력 + "학부모 N명 액세스 해제" 경고) → 단일 트랜잭션으로 해당 학급 학생의 전 active `ParentChildLink` revoke (`classroom_deleted`) + `ParentSession` 즉시 차단 + 학부모 cascade 안내 이메일 (교사 정보 비노출) | **신규** | PV-8, PV-14 | 1.5일 |

**총 공수**: 34.5일 (PV-6 최대 분할 시 35.5일). v1 27일 → v2 34.5~35.5일 (증분 +7.5~8.5일, D-50 + D-51~56).

**1차 배포 범위**: PV-1 ~ PV-9 + PV-13 ~ PV-15 + PV-12 (승인 게이트 핵심 경로 포함)
**2차 배포**: PV-10 (주간 이메일) + PV-11 (탈퇴) + PV-16 (회전)
**v2+ 파킹**: 실시간 push · 카카오 알림톡 · 응원 이모지/댓글 · Kakao/Google OAuth · Redis 즉시 revoke · 비밀번호 로그인 · Enterprise SSO · Native 앱 · **교사 자유 메시지** (D-29) · **에스컬레이션 경로** (D-23) · **월드카페 프리셋** · **학부모 앱 네이티브**

---

## 8. 수용 기준 (v2 seed_6d7077aac472 acceptance_criteria)

- [ ] 교사가 학급 설정 "학부모 액세스" 탭에서 학급 단위 `ClassInviteCode`(Crockford Base32 8자리) 발급·회전 가능
- [ ] 학부모가 학급 코드로 signup 후 매직 링크로 ParentSession 발급(signup 시점)
- [ ] 학부모가 학급 명단(반·번호+마스킹 이름 "3반 12번 김O민")에서 자녀 선택 → pending 생성 가능
- [ ] `ParentChildLink.status`가 `pending`→`active`→`rejected`→`revoked` 4-value 전이 머신으로 동작
- [ ] 교사 UI "학부모 액세스" 탭에 초대 코드/승인 인박스/연결된 학부모 3-섹션 표시
- [ ] 동일 자녀에 부·모 각각 별도 승인 가능(`@@unique([parentId, studentId])` 제약만 적용)
- [ ] ParentSession은 signup 시점 생성, BoardMember는 approve 시점 생성(RLS 오염 방지)
- [ ] pending 상태 응답 HTTP **200 OK + `{"status":"pending"}`** payload 반환
- [ ] Vercel Cron KST 02:00 일 1회 pending 7일 초과 건 `auto_expired_pending` 자동 만료 처리
- [ ] D+3 교사 리마인더 이메일 + D+6 최종 경고 이메일 + D+7 요약 이메일 발송
- [ ] 교사 능동 거부 시 `wrong_child`/`not_parent`/`other` 3종 드롭다운 선택 가능
- [ ] `rejected_by_teacher` 이메일에 선택된 사유 문구 포함 + 재신청 deep link 포함
- [ ] `auto_expired_pending` 이메일에 고정 문구 + 재신청 deep link 포함
- [ ] 코드 회전 시 해당 학급 pending 일괄 `rejected` (revokedReason=`code_rotated`) 처리 + active 링크 유지
- [ ] `/parent/*` 라우트에서 자녀 그림보드·식물관찰·행사·Breakout·숙제 열람 가능 (**status='active' 전제**)
- [ ] pending 동안 자녀 본인 콘텐츠도 열람 불가 (403)
- [ ] `parentAuthOnlyMiddleware` 매칭 전 엔드포인트 적용 + `parentScopeMiddleware` 매칭 후 `status='active'` 적용
- [ ] parent 토큰으로 타 학생 studentId API 직접 호출 → **403** (E2E 필수)
- [ ] parentA 토큰으로 parentB의 `ParentChildLink` 조회 → **404** (E2E 필수)
- [ ] 교사 1-click revoke → ≤ 60s 내 학부모 세션 차단
- [ ] Classroom 삭제 시 해당 학급 전 active `ParentChildLink` cascade revoke (`classroom_deleted`) + ParentSession 즉시 차단
- [ ] Classroom 삭제 확인 모달에 "학부모 N명 액세스 해제" 경고 + 학급명 재입력 확인 표시
- [ ] Cascade 학부모 안내 이메일에 교사 이름·사유 비노출, 학급명만 포함
- [ ] 학부모 401 수신 시 자동 로그아웃 + "접근이 해제되었습니다" 화면
- [ ] Pro 학부모에게 매주 월 09:00 KST 주간 요약 이메일 발송 (활동 0건 주 스킵)
- [ ] 학부모 탈퇴 → soft delete → 90일 후 PII 익명화
- [ ] PWA 설치 가능 (/parent/* manifest 포함)
- [ ] EventBoard classroomId 조회 허용 + 타 학생 EventSignup 응답 제거
- [ ] Breakout `session.studentId ∈ parent.children WHERE status='active'` 검증
- [ ] 390px 세로 Lighthouse Mobile ≥ 90
- [ ] 3G fast TTI < 3s
- [ ] iframe 마운트 0건 (DOM snapshot)
- [ ] IP 5회 실패/15분 잠금 + 코드 50회/일 + 학급 100회/일 rate limit
- [ ] 거부 이메일에 교사 이름·이메일·전화번호 비노출 (학교 대표 연락처만)
- [ ] 동일 학부모 이메일 거부 3회 초과 시 24h 재신청 차단
- [ ] 공수 증분 +6~7일(총 33~34일) 범위 내 완료

---

## 9. 리스크 & 완화

| 리스크 | 완화 |
|---|---|
| parentScopeMiddleware 우회 (개발자 실수) | RLS 이중 방어 + PV-12 E2E 게이트 필수 + `/parent/*` 엔드포인트 ESLint 룰("must wrap in parentScope or parentAuthOnly") — v2에서는 2종 분기 |
| 학급 코드 brute-force (v2 noise ↑) | IP 15분 잠금 + 코드 50회/일 + **학급 100회/일** + CSPRNG. 32⁸ ≈ 10¹² 조합. 자동 만료는 DoS 벡터이므로 **제거** (D-13) |
| 사칭 시도 (학부모가 남의 자녀 선택) | 교사 승인 게이트 + 거부 3회 초과 24h 쿨다운 + 거부율 임계 교사 배지 |
| pending 적체 (교사 응답 지연) | D+0 배지 + D+3 리마인더 + D+6 경고 + D+7 자동 만료 Cron 4단 스케줄 |
| 학부모 간 식별 누출 (동일 자녀) | RLS `parent_id = auth.parent_id()` 단방향 + 주간 이메일 BCC 금지 개별 발송 + 교사 UI에서만 학부모 이름 노출 |
| Revoke 60s 지연 = 양육권 분쟁 민감 | v1 SLA 문서 명시 + 즉시 revoke는 v2 Redis 블랙리스트 |
| 썸네일 URL 추측 | presigned URL (7일) 또는 RLS-scoped query. 원본 Vercel Blob 직접 접근 차단 |
| 탈퇴 학부모 감사 소실 | soft delete + 90일 PII 익명화, `ParentChildLink`는 감사 보존 영구 |
| 다자녀 상한 5명 초과 | 앱 레벨 enforce, 6번째 자녀는 친척 학부모 별도 계정 유도 |
| iframe 금지 규칙 위반 | ESLint 룰 + DOM snapshot E2E (PV-12) |

---

## 10. 파킹 (v2+ · v3+)

- 실시간 push 알림 (Pro 확장 후보)
- 카카오톡 알림톡
- 학부모 코멘트·좋아요·응원 이모지
- Kakao/Google OAuth (매직 링크만 v1)
- Quiz 점수 열람 (사후 요약 리포트)
- 학부모 셀프 자녀 이동·연결 변경 (교사 재발급 경유)
- Redis 블랙리스트 기반 즉시 revoke (< 1s)
- 비밀번호 기반 로그인
- Enterprise 학교 SSO (G Suite/네이버웍스)
- Parent 앱 native (Flutter/React Native)
- "반 대표 학부모" 공식 역할 (v3+)
- 자녀 포트폴리오 내보내기 PDF
- **교사 자유 메시지 입력** (거부 이메일에 커스텀 문구 첨부, v2 파킹 — D-29)
- **에스컬레이션 경로** (승인 SLA 초과 시 학급장/관리자 알림, 1인 개발자 운영 전제 해소 후, v2 파킹 — D-23)
- **사칭 감지 SOP 고도화** (거부율 임계 알고리즘·학급 코드 회전 SOP·패턴 감지 룰, P2 파킹 — D-48)

---

## 11. 변경 로그

| 날짜 | 시드 ID | 변경 |
|---|---|---|
| 2026-04-12 | `seed_37b35654542f` | 초안 생성 — Parent·ParentChildLink·ParentInviteCode·ParentSession 4종 스키마, Crockford Base32 6자리, 매직 링크 인증, 7일 세션, revoke ≤60s, 학부모 1인당 자녀 5명, Free 2/Pro 5 초대 한도, soft delete + 90일 익명화, 주간 이메일 Pro 전용, `/parent/*` PWA, iframe 금지, PV-1~PV-12 작업 분할. Seed 1·3·4·6의 개별 "학부모 열람" 언급을 §5 매트릭스로 통합. |
| 2026-04-13 (amendment) | `seed_6d7077aac472` + D-51~56 | **Active 링크 수명 정책 확정** — 학급(Classroom) 존재 기간과 1:1 (시간 기반 자동 만료 없음, `year_end` Cron 폐지 확정). Classroom 삭제 시 cascade revoke(`classroom_deleted`), 삭제 확인 모달 + 학부모 cascade 안내 이메일(교사 정보 비노출). PV-17 신규(+1.5일). 총 공수 34.5~35.5일. |
| 2026-04-14 | `seed_38c34e91bf28` | **§5 매트릭스에 과제 배부 보드(Seed 11) 행 추가** — `AssignmentSlot.studentId ∈ parent.children` 필터로 자녀 slot + Card/Submission/반려 사유만 열람. 5×6 격자 자체 미렌더(DOM 트리에 타 학생 노드 0), 자녀 전용 단일 카드 뷰로 축약. `assignmentGuideText`는 viewer-readable. §6 PWA 라우트 `/parent/child/[id]/assignment` 추가. matrix 뷰는 viewer 제외 유지. 구현은 PV-7 자녀 범위 서버 필터에서 처리(작업 카드 신설 없음). |
| 2026-04-16 | `seed_0badf1e571bc` | **수행평가 성적 탭 편입 미결 메모 추가** (§관련 주제 참조). 본 변경은 §5 매트릭스·§7 PV-* 작업 카드에 **영향 없음** (v1에서는 assessment-autograde 로드맵 안의 별도 뷰로 진행, 본 로드맵에 탭 편입은 v1.5 integrate 시점에 결정). |
| 2026-04-13 | `seed_6d7077aac472` (parent_seed_id=`seed_37b35654542f`, supersede) | **v2 학급 단위 코드 + 셀프매칭 + 교사 승인 게이트** 도입 — `ParentInviteCode` → `ClassInviteCode` 치환(학급 1:코드 1, Crockford Base32 **8자리**, 학기말 만료 + 수동 회전). `ParentChildLink.status` 4-value 전이(`pending`/`active`/`rejected`/`revoked`) + 감사 필드 6종(requestedAt·approvedAt·approvedById·rejectedAt·rejectedById·rejectedReason). `revokedReason` enum에 `rejected_by_teacher`·`auto_expired_pending`·`code_rotated` 추가. `rejectedReason` 신규 enum(`wrong_child`/`not_parent`/`other` 드롭다운, 자유 텍스트 v2 파킹). 승인 게이트 흐름 §1.5 신규 — pending TTL **7일** / 권고 SLA **24h** / 동시 pending 상한 **3건** / D+0 배지 → D+3 리마인더 → D+6 경고 → D+7 자동 만료 Vercel Cron(KST 02:00). 셀프매칭 명단은 반·번호 + 성+O+끝글자 마스킹("김O민") · 프로필 사진 비노출 · 학급당 100회/일 rate limit. 거부 이메일은 교사 정보 비노출 · 재신청 deep link · 동일 이메일 3회 초과 24h 쿨다운. ParentSession=signup 시점, BoardMember=approve 시점(RLS 오염 방지). 교사 UI 3-섹션화(학생 카드 드롭다운 "학부모 초대" 제거). Free/Pro 발급 한도 개념 폐지(학급 코드 체제 의미 붕괴), Pro는 주간 이메일 전용. `parentAuthOnlyMiddleware` 신규(매칭 전 전용). PV-1·2·3·5·8·12 개정 + PV-13(셀프매칭) · PV-14(승인 인박스) · PV-15(자동 만료 Cron + 알림) · PV-16(코드 회전) 신설. 총 공수 27일 → **33~34일**. 마이그레이션: v1 미배포 상태이므로 완전 폐기 즉시 전환. |


---

## 관련 주제

- [`./assessment-autograde-roadmap.md`](./assessment-autograde-roadmap.md) — 수행평가 자동채점 파이프라인 (Seed 12, `seed_0badf1e571bc`, 2026-04-16).
  - **수행평가 성적 탭 편입 미결 (분기 3.1)**. parent-viewer v2의 자녀 단일 뷰에 "성적" 탭을 **편입**할지, 별도 route `/aura-web/gradebook`을 **신설**할지 미결. 현재 합의는 "parent-viewer v2와 동일 Aura 웹앱·동일 RLS·동일 PWA 셸"까지. UI 트리(탭 vs 페이지) 결정권은 **v1.5 integrate 시점**이며, 결정 이후 본 로드맵 §5·§6 또는 §7 PV-* 작업 카드에 반영한다. v1에서는 assessment-autograde 로드맵 안의 학생·학부모 뷰(`/aura-web/gradebook`)를 먼저 구현하되, 학부모 RLS는 `ParentChildLink.status='active'` + `GradebookEntry.releasedAt IS NOT NULL` 이중 조건으로 §5 매트릭스 원칙을 승계한다.
