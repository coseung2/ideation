# Phase 2 — Parent/Viewer Access sketch

- task_id: `2026-04-12-parent-viewer-access`
- 기준 단말 (학부모): **스마트폰 세로 320~430px**, iOS Safari · Android Chrome, 3G~LTE 변동
- 기준 단말 (교사·학생): 기존 갤럭시 탭 S6 Lite 승계 (이 작업에서 변경 없음)
- 전제: **phase1 1순위 하이브리드 고정** — ClassDojo 단기 코드 + Seesaw 자녀-스코프 필터 + Canvas Observer RBAC + 하이클래스/Seesaw Family 모바일 UX
- 선행 결정 승계:
  - Seed 1 (그림보드): 학부모는 자녀 전체 열람, `StudentAsset.isSharedToClass` 상위
  - Seed 4 (식물관찰일지): 학부모는 자녀 1인 전체 열람, 관찰 `isPrivate` 상위
  - Seed 3 (행사 보드): 학부모는 자녀 Submission·영상·결과 열람
  - Seed 6 (Breakout): 학부모는 자녀 모둠 Section 열람 (편집 X)
  - `BoardMember.role = "parent"` 신설 (memory 메모 확정)
  - 학부모는 별도 인증 스택, 학생 QR/textCode와 분리

---

## 1. 전제 고정 (phase1 권고 승계)

| 축 | 고정 결정 | 출처 |
|---|---|---|
| 인증·초대 | 교사 발급 8~9자리 Parent Code, **48h 만료 + 최대 3회 사용** (phase3에서 4회/30일 vs 3회/48h 최종 논의) | ClassDojo |
| 매핑 모델 | N:M (복수 보호자 ↔ 복수 자녀) | Seesaw·Canvas·ClassDojo |
| RBAC 역할 | `BoardMember.role = "parent"` — read-only, 쓰기·댓글·공유 전면 차단 | Canvas Observer |
| 스코프 필터 | 서버 미들웨어·RLS 수준에서 `studentId ∈ parent.children` 강제 | Seesaw |
| 학부모 계정 | 별도 계정·별도 인증 스택 (이메일·전화·카카오 중 택 1). 학생 QR과 분리 | 통합 관례 |
| 모바일 UX | `/parent/*` 라우트에 390px 세로 전용 레이아웃. 태블릿 레이아웃 상속 금지. PWA 우선 | 하이클래스·Seesaw Family |
| 알림 v1 | 인앱 배지 + **주간 요약**만 (실시간 push는 v2+) | 보수적 기본값, phase3 확정 |
| 개인정보 | 자녀 본명·사진 기본 노출, 동명이인 구분은 학년·반으로만. 타 학생 노출 불가 | Seed 1·4 승계 |

**성능 예산 (학부모 스마트폰 신규 추가)**:
- 자녀 피드 초기 TTI < 2s (LTE), < 3s (3G)
- 이미지 썸네일 한 장 < 200KB (스마트폰 기준 더 타이트)
- 첫 뷰포트 네트워크 요청 < 500KB
- iframe 학부모 뷰에서 **원천 금지** (Canva·Drawpile 모두 썸네일 프록시만)
- 동시 열람 상한: 학부모 1인당 자녀 수 **≤ 3 (다자녀 상한, phase3 확정 필요)**

---

## 2. Prisma 스키마 초안 — 신규/수정

### 2.1 신규 엔티티 (4개)

```prisma
// 학부모 계정 — User와 분리된 별도 스택.
// 이메일/전화 단일 유니크 식별자. OAuth는 카카오만 v1 지원 검토(phase3).
model Parent {
  id              String    @id @default(cuid())
  // 식별자는 셋 중 최소 1개 필수 — 앱 레벨 zod로 검증.
  email           String?   @unique
  phoneHash       String?   @unique  // 평문 저장 금지 (개보법)
  kakaoSubHash    String?   @unique  // 카카오 sub 해시
  displayName     String
  emailVerified   DateTime?
  phoneVerifiedAt DateTime?
  locale          String    @default("ko-KR")
  createdAt       DateTime  @default(now())
  revokedAt       DateTime? // soft delete — 양육권 변경 시 즉시 철회

  links    ParentChildLink[]
  sessions ParentSession[]

  @@index([email])
  @@index([phoneHash])
}

// 학부모 ↔ 학생 N:M 매핑. 상태·발급자·만료 추적.
model ParentChildLink {
  id         String    @id @default(cuid())
  parentId   String
  studentId  String
  // "pending" → "active" → "revoked" (app zod validation)
  status     String    @default("pending")
  // 관계 명칭 (선택): "mother" | "father" | "guardian" | "other"
  relation   String?
  issuedById String    // 발급 교사 User.id (감사 로그)
  activatedAt DateTime?
  revokedAt   DateTime?
  revokedReason String? // "custody_change" | "self_withdraw" | "teacher_action"
  createdAt   DateTime  @default(now())

  parent    Parent  @relation(fields: [parentId], references: [id], onDelete: Cascade)
  student   Student @relation(fields: [studentId], references: [id], onDelete: Cascade)

  @@unique([parentId, studentId])
  @@index([studentId])
  @@index([parentId, status])
  @@index([issuedById])
}

// 교사가 발급하는 단기 Parent Code.
// 8~9자리 alphanumeric, tokenHash로만 저장, 클립보드·종이 배포용 plain은 발급 직후 1회만 노출.
model ParentInviteCode {
  id            String    @id @default(cuid())
  studentId     String
  issuedById    String    // 교사 User.id
  codeHash      String    @unique
  // 48h 기본 만료 (phase3에서 확정)
  expiresAt     DateTime
  // 최대 사용 횟수. 초과 시 소프트 만료.
  maxUses       Int       @default(3)
  usedCount     Int       @default(0)
  consumedByParentIds String @default("[]") // JSON array - 감사 기록
  revokedAt     DateTime?
  createdAt     DateTime  @default(now())

  student Student @relation(fields: [studentId], references: [id], onDelete: Cascade)

  @@index([studentId])
  @@index([expiresAt])
}

// 학부모 세션 (User.Session과 분리된 네임스페이스).
// NextAuth 확장 대신 별도 테이블 — 학생 QR·교사 OAuth와 인증 스택 격리.
model ParentSession {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  parentId    String
  expires      DateTime
  userAgent    String?
  createdAt    DateTime @default(now())

  parent Parent @relation(fields: [parentId], references: [id], onDelete: Cascade)

  @@index([parentId])
}
```

### 2.2 기존 엔티티 수정 (2곳만)

```prisma
// BoardMember.role 값 집합 확장 (스키마 변경 없음, app zod만).
// "owner" | "editor" | "viewer" | "parent"  ← 신규
//   parent는 row 존재 시 무조건 read-only + studentId 스코프 (join으로 강제)

// Student 역참조만 추가 (마이그레이션 1건).
model Student {
  // ... 기존 필드 유지
  parentLinks ParentChildLink[]
  inviteCodes ParentInviteCode[]
}
```

**마이그레이션 영향 평가**:
- 신규 테이블 4개만, 기존 컬럼 변경 0건 → 무중단 배포 가능
- `BoardMember.role`은 문자열 유니언이라 DB 마이그레이션 불필요 (app zod만 업데이트)
- Postgres RLS 정책은 **추가** 형태 (기존 정책 수정 X)

---

## 3. RBAC 확장 설계

### 3.1 역할·권한 매트릭스

| Action | owner | editor | viewer | **parent** |
|---|---|---|---|---|
| 카드/애셋 조회 (자녀 스코프) | ✅ | ✅ | ✅ | ✅ (자녀 것만) |
| 카드/애셋 조회 (타 학생) | ✅ | ✅ | ✅ (공개분) | ❌ |
| 카드 생성·수정·삭제 | ✅ | ✅ | ❌ | ❌ |
| 댓글·리액션 | ✅ | ✅ | ❌(설정) | ❌ (v1 고정) |
| 공유 링크 발급 | ✅ | ❌ | ❌ | ❌ |
| 교사 승인·피드백 | ✅ | ❌ | ❌ | ❌ |
| 보드 설정 변경 | ✅ | ❌ | ❌ | ❌ |

### 3.2 서버 쿼리 강제 원칙

`src/lib/rbac.ts`에 새 함수 `assertParentScope(parentId, studentId)` 추가. 모든 학부모 라우트 API는 이 함수를 첫 줄에서 호출 → 실패 시 403.

추가로 Prisma middleware (또는 Postgres RLS)에 "session.role == 'parent' 면 studentId IN (SELECT studentId FROM ParentChildLink WHERE parentId = session.parentId AND status = 'active')" 강제 필터.

### 3.3 feature별 스코프 규칙 (cross-cutting 일관)

| Feature | 학부모 조회 범위 | isPrivate/isPublic 상위? |
|---|---|---|
| 그림보드 라이브러리 (Seed 1) | 자녀의 StudentAsset 전체 (isSharedToClass 무관) | ✅ 상위 |
| 식물관찰일지 (Seed 4) | 자녀의 StudentPlant 1건 + 전 PlantObservation | ✅ 상위 (observation isPrivate 무시) |
| 행사 신청 보드 (Seed 3) | 자녀가 제출한 Submission·videoUrl·SubmissionReview feedback·결과 | ✅ announceMode 무관 |
| Breakout (Seed 6) | 자녀가 속한 Section 카드만. 교사-pool Section 제외 | ✅ peek-others 무관 |
| 숙제 제출 (Assignment) | 자녀의 Submission + 교사 피드백/평가 | ✅ |
| Quiz | **v1 제외** — 실시간 세션 구조라 사후 열람만 제공(점수 요약 페이지) | v2 확장 |

---

## 4. 역할별 사용자 흐름

### 4.1 교사 — 학부모 초대 발급

1. `/classroom/[id]/students/[sid]/invite-parent` 진입
2. 기존 활성 링크 목록 + "새 Parent Code 발급" 버튼
3. 클릭 → 서버가 8자리 코드 생성, `ParentInviteCode` 저장 (codeHash, expiresAt = now+48h, maxUses=3)
4. 모달에 **plain code 1회 노출** + 카카오톡/문자 공유 버튼 + 종이 인쇄용 PDF (학생 이름 + 코드 + QR)
5. 학생이 종이/메시지로 학부모에게 전달

### 4.2 학부모 — 최초 연결

1. 스마트폰에서 `parent.aura-board.app` 접속 (PWA 설치 유도 배너)
2. **이메일 or 전화번호 or 카카오** 중 택 1로 계정 생성 (OTP 인증)
3. "자녀 연결하기" → 8자리 Parent Code 입력
4. 서버 검증: 코드 유효 + 미만료 + 사용횟수 < max → `ParentChildLink` 생성 (status="active"), `usedCount++`
5. 자녀 피드로 리다이렉트 (자녀 이름·학급·썸네일 타임라인)
6. 다자녀: "자녀 추가" 버튼 → 2단계 반복

### 4.3 학부모 — 일상 열람 (스마트폰)

1. 홈 = 자녀별 카드 스택 (썸네일 + 최근 활동 배지)
2. 자녀 탭 → 탭 메뉴: [그림보드] [식물일지] [행사] [숙제]
3. 각 탭 세로 스크롤, 썸네일 1열 (Vercel Image Optimization, lazy load)
4. 상세 탭 시 풀스크린 시트 (이미지 확대, 메모 표시, 교사 피드백 표시)
5. 편집 UI 완전 제거 (저장·댓글·공유 버튼 렌더 안 함)
6. 주간 요약 배지: "이번 주 새 관찰 3건, 그림 2장"

### 4.4 교사 — 학부모 철회

1. `/classroom/[id]/students/[sid]/parents` → 활성 보호자 리스트
2. "연결 해제" 버튼 → `ParentChildLink.status = "revoked"`, `revokedReason = "teacher_action"`
3. 해당 학부모 세션 즉시 무효화 (Prisma middleware가 status 체크)
4. 학부모가 다시 접속 시 "해제됨" 안내 + 재발급 문의 연락처

### 4.5 학생 — 코드 재요청 (오프라인 대체)

- 학생이 부모에게 코드 전달 실패 시, 교사에게 재발급 요청. 학생 UI에는 학부모 기능 노출 안 함 (인증 스택 완전 분리)

---

## 5. 모바일 성능 체크리스트 (tablet-performance-roadmap §2 준거 + 스마트폰 확장)

| 지표 | 학부모 스마트폰 목표 | 비고 |
|---|---|---|
| TTI 자녀 피드 | < 2s (LTE) / < 3s (3G) | 태블릿 30카드 기준보다 더 타이트 |
| 세로 스크롤 프레임 | 60fps | IntersectionObserver 가상화 필수 |
| 첫 뷰포트 바이트 | < 500KB (이미지 포함) | srcset w=320 최소 해상도 |
| 이미지 썸네일 개당 | < 200KB | Vercel Image 자동 WebP/AVIF |
| iframe 동시 마운트 | **0** | 학부모 뷰는 iframe 금지, 프록시 썸네일만 |
| WebSocket | **비활성** | 학부모 실시간 불필요, SWR 폴링 (v1 60초) |
| 메모리 30분 사용 후 | < 200MB | 스마트폰 OS 백그라운드 KILL 대응 |
| PWA 오프라인 | 마지막 피드 idb-keyval 캐시 | 지하철·엘리베이터 대응 |

**신규 게이트 (QA phase9 추가)**:
- [ ] 390px 세로 뷰포트 Lighthouse Mobile Score ≥ 90
- [ ] 3G fast 에뮬레이션 TTI < 3s
- [ ] iframe 마운트 0건 검증 (DOM snapshot)
- [ ] 학부모 세션에서 타 학생 데이터 0건 유출 (네트워크 패널 assertion)

---

## 6. tier·과금 연계 (Seed 2 승계)

| 축 | 제안 (phase3 최종 확정) |
|---|---|
| Free 교사 | 학부모 초대 **자녀당 2명까지** (부·모 가정 기본) |
| Pro 교사 | 학부모 초대 **자녀당 5명까지** (조부모·보호자 포함) |
| 학부모 1인당 자녀 수 상한 | **5명** (다자녀 현실 상한) — phase3 확정 |
| 유료 게이팅 vs 기본 제공 | **기본 제공 권장** — 학부모 소외는 교육 공공성 훼손. 단, 발급 한도만 tier 연계 |
| 알림 push (v2) | Pro 전용 후보 |

이유: 학부모 열람이 Pro 전용이면 Free 반 자녀 가정이 차별받아 교사 도입 저항. 발급 한도만 tier gating.

---

## 7. 리스크 표

| 리스크 | 영향 | 완화 |
|---|---|---|
| **양육권 분쟁 시 접근 철회 지연** | 높음 — 법적·윤리 이슈 | 교사 1클릭 revoke + `revokedAt` 즉시 세션 무효화. 감사 로그 `issuedById`·`revokedReason` 필수 |
| **타 학생 개인정보 유출** | 매우 높음 — 학부모가 반 다른 아이 작품 볼 수 있으면 치명 | Prisma middleware 하드코드 필터, E2E 테스트 필수, 게이트에 "타 학생 데이터 0건" assertion |
| **스마트폰 저사양 단말 렉** | 중간 — 갤S8·저가 Android 대응 필요 | iframe 금지, 이미지 200KB 상한, 가상화 스크롤, PWA 오프라인 캐시 |
| **학부모 앱 설치 거부** | 중간 — 한국 학부모 저장공간·개보법 우려 | PWA 우선 (설치 없이 동작), 카카오톡 웹뷰 호환, 이메일 fallback |
| **동명이인 자녀 연결 오류** | 중간 | Parent Code는 studentId 1:1 바인딩. 입력 후 "학급 + 이름 + 학년" 확인 2단계 |
| **단기 코드 유출·재사용** | 낮음-중 | 48h 만료 + maxUses 3 + codeHash 저장 + 사용 기록 감사 |
| **학부모 계정 탈취** | 높음 | OTP 재인증, PhoneHash 저장, 세션 만료 14일, 이상 로그인 교사 알림 |
| **다자녀 상한 과다 발급** | 낮음 | Parent당 active link ≤ 5 (앱 레벨 enforce) |
| **v1 알림 기대 불일치** | 낮음 | UI에 "주간 요약 기준" 명시 배너, push는 v2 로드맵 공표 |
| **교사 실수로 전체 노출** | 중간 | 학부모는 BoardMember row 없이도 ParentChildLink만으로 스코프 쿼리 — 교사 실수 경로 자체 제거 |

---

## 8. Canva·기존 기능 시너지

| 기존 기능 | 학부모 뷰 처리 |
|---|---|
| Canva oEmbed 카드 (P0-①) | iframe 대신 썸네일 프록시만 (읽기 전용이라 interactive 불필요) |
| Canva Autofill 결과물 (P1-③) | 교사 발송 개인화 자료는 자녀 대상만 학부모 열람 허용 |
| Drawpile 그림보드 (Seed 1) | ORA iframe 비노출, PNG 썸네일만. 라이브러리 전체(비공개 포함) 표시 |
| Quiz (Seed 5 Quiz) | v1 비노출. v2에서 점수 요약 리포트만 |
| 섹션 accessToken (T0-①) | 학부모는 accessToken 무시, ParentChildLink 스코프로 자녀 Section만 조회 |
| 행사 Submission (Seed 3) | 자녀 제출분만. 타 신청자 목록·득점 비노출 |

---

## 9. 미결 질문 (phase3 인터뷰 재료) — 7개

1. **페어링 방식 조합**: phase1은 Parent Code만 제안. QR(학생이 탭에서 부모폰으로 스캔) 병행할지? 교사 승인 단계 추가(pending→active)할지, 즉시 active? → 권장 **자동 active + 교사 revoke 권한**, phase3 확인.
2. **알림 v1 범위**: 인앱 배지만 / 주간 이메일 요약 / 카카오톡 알림톡 중 어디까지? → Seed 2 tier 정책과 결합. 기본값 제안: 인앱 배지 + 주간 이메일.
3. **학부모 1인당 자녀 수 상한**: 5명? 무제한? 교사당 학부모 수 상한과의 관계. → 권장 5명.
4. **학부모 코멘트 기능**: v1은 read-only 추천이지만, 교사 일방향 "확인했어요" 스탬프만 허용할지 (하이클래스 수신확인 UX). → 권장 v2 파킹, v1은 순수 read-only.
5. **스마트폰 전용 뷰 분기**: 반응형 단일 라우트 vs `/parent/*` 별도 라우트 vs 완전 별도 PWA 도메인. → 권장 별도 라우트 `/parent/*` + PWA, 390px 전용 레이아웃.
6. **학생 개인정보 노출 기본값**: 자녀 본명·얼굴 사진 기본 노출 여부. 다른 학부모(부+모)가 동일 자녀 볼 때 격리 보장. 특히 조부모 연결 시 노출 범위. → phase3 확인.
7. **tier 연계**: 학부모 기능은 Free 기본 제공 / 발급 한도만 tier gating 제안. 유료 전용으로 돌릴지? → 권장 기본 제공 + 발급 한도 gating.

---

## 10. 검증 게이트 체크

- [x] 전제 결정 (phase1 하이브리드 고정) 명시
- [x] Prisma 신규 엔티티 ≥ 1 (신규 4개, 수정 1곳) — 10개 미만 (에스컬레이션 없음)
- [x] 역할별 흐름 ≥ 1 (교사·학부모·학생 3 + 교사 revoke = 5 흐름)
- [x] 태블릿 성능 체크리스트 (tablet-performance-roadmap §2 준거, 스마트폰 확장 추가)
- [x] Canva/tier/기존 기능 시너지 매트릭스
- [x] 리스크 표 ≥ 5 (10개 수록)
- [x] 미결 질문 7개 (≤ 7 권장 충족)
- [x] 기존 Board·Card·Section·Classroom·Student 재사용 우선 (수정 최소)
- [x] padlet 폴더 쓰기 없음 (문서만 작성)

---

## 11. phase3 인터뷰 제안 순서

1. 자녀 스코프 격리 E2E 검증 규약 (리스크 #2 방지)
2. 페어링 방식 최종 확정 (미결 #1)
3. 알림 v1 스코프 (미결 #2)
4. tier 연계 (미결 #7)
5. 나머지 미결 일괄
