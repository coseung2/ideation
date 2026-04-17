# Aura-board Breakout Room 보드 로드맵

> 작성일: 2026-04-12
> Seed: `seed_bb1d4eb1c442` (task `2026-04-12-breakout-room-board`)
> Interview: `interview_20260412_102210` (ambiguity 0.147)
> 전제 로드맵:
> - `ideation/plans/tablet-performance-roadmap.md` (T0-① 섹션 격리 뷰 = 본 기능 구현 전제)
> - `ideation/plans/drawing-board-library-roadmap.md` (AssetAttachment 복사 패턴 준거)
> - Seed 2 Tier 정책 (`seeds-index.md` Seed 2) 승계

---

## 0. 핵심 명제

> **Aura-board의 모둠 협력학습 축 = "갤럭시 탭 S6 Lite 30명에서도 렉 없이, 모둠 간 완전 독립된 작업 공간을 교사가 템플릿으로 개설·배포·회수한다."**

Padlet Breakout rooms의 "브로드캐스트 + 클라이언트 필터" 실패 사례를 반복하지 않고, 이미 확보된 **Section 엔티티 + T0-① 섹션 격리 뷰 인프라**를 1:1로 재활용해 구현한다.

---

## 1. 설계 전제 (Phase 3 확정 결정)

### 1.1 엔티티 최소화 — 신규 2종, Section 재활용

| 결정 | 내용 |
|---|---|
| Section 엔티티 | **재활용 유지**. `BreakoutGroup` 신설 금지. `Section.accessToken`은 이미 "Breakout view teacher-rotatable access token"으로 정의됨. |
| 신규 엔티티 1 | `BreakoutTemplate` — 시스템/교사 커스텀/학교 공용 템플릿 카탈로그 |
| 신규 엔티티 2 | `BreakoutAssignment` — 수업 인스턴스(Board + 템플릿 + 배포·열람·상태) |
| teacher-pool 섹션 | **보드 레벨 단일 섹션**으로만 존재 (모둠 수와 독립). 전 모둠 공유 자원 풀. |
| 라벨 | i18n 계층에서 `layout=="breakout"`일 때 "모둠 N"으로 표기 (DB 용어는 `section` 유지) |
| Jigsaw "한 학생 두 모둠 소속" | `BreakoutAssignment.role = "expert" | "home"` 플래그로 처리. 별도 엔티티 불필요. |

### 1.2 배포모드 3종 · 열람모드 2종

| 축 | 값 | 설명 |
|---|---|---|
| deployMode | `link-fixed` | 모둠별 물리 링크 배포(학생이 링크 = 모둠 결정). 이동 개념 없음. |
| deployMode | `self-select` | 학생이 모둠 목록에서 초기 1회 선택. 변경은 교사 승인. |
| deployMode | `teacher-assign` | 교사가 배정. 학생 자발적 이동 불가(신고 버튼만). |
| visibility | `own-only` | 학생은 자기 모둠만. (기본값) |
| visibility | `peek-others` | 학생도 타 모둠 열람. (갤러리 워크·발표 준비 전용) |
| **교사 권한** | 전 모둠 전체 접근 | 가시성 모드와 직교한 상위 RBAC (`viewSection`) — 어떤 설정에서도 보장. |

### 1.3 모둠 기본값 / 상한

| 항목 | 값 | 근거 |
|---|---|---|
| defaultGroupCount | **4** | 소규모 학급·특별실 대응 |
| defaultGroupCapacity | 6 (soft) | 협업 품질 임계 |
| maxGroupCount | **10** | T0 성능 예산 내 |
| maxGroupCapacity | 6 | 협업·성능 밸런스 |

### 1.4 복제 방식 — 복사(독립)

- 보드 개설 시 `BreakoutTemplate.structure.sectionsPerGroup[]`를 모둠 수(N)만큼 복제 → Section N개 INSERT
- 모둠 섹션 카드는 **모둠 간 완전 독립** (1모둠 수정이 2모둠에 전파 X)
- **예외 — `role="teacher-pool"` 공유 섹션**: 보드 레벨 단일. 자연스럽게 전 모둠 공유.
- **"모든 모둠에 이 카드 복제" 단발 액션**: 교사 UI의 명시적 버튼 — 서버가 N모둠 섹션에 INSERT 반복(한 번만). 되돌리기 별도.
- **템플릿 원본 수정 역전파 X**: `BreakoutTemplate.structure` 수정은 기존 파생 Board에 영향 없음. `version` 필드로 버전 관리. 새 Board 개설 시점에만 최신 반영.
- drawing-board-library-roadmap의 `AssetAttachment` 복사 패턴과 의식적으로 일관되게 유지.

### 1.5 Tier 매트릭스

| 항목 | Free | Pro |
|---|---|---|
| 브레이크아웃 레이아웃 | ✅ 공통 기본 | ✅ 공통 기본 |
| 반 수 상한 | 5반 (Seed 2 승계) | 무제한 |
| 시스템 템플릿 접근 | **3종** (KWL · 브레인스토밍 · 아이스브레이커) | **8종 전체** |
| 교사 커스텀 템플릿 저장 | **3개** 상한 | 무제한 |
| 학교 공용 템플릿 등록 (`scope="school"`) | ❌ | ✅ |
| 쿼터 초과 시 | 안내 모달 + 업그레이드 CTA (Seed 2 정책 승계, 자동 결제 X) |

Gating: `BreakoutTemplate.requiresPro=true` 플래그.

### 1.6 v1 템플릿 카탈로그 (7+1종, v2 파킹 1종)

| # | 템플릿 | Tier | recommendedVisibility | 근거 |
|---|---|---|---|---|
| 1 | KWL 차트 | Free | own-only | 전 학년 보편 |
| 2 | 브레인스토밍 | Free | own-only | 발산 오염 방지 |
| 3 | 아이스브레이커 | Free | own-only | 모둠 친밀도 |
| 4 | 찬반 토론 | Pro | own-only | 논지 독립 형성 |
| 5 | Jigsaw | Pro | own-only | expert/home 단계 독립 |
| 6 | 모둠 발표 준비 | Pro | peek-others | 상호 학습 가치 |
| 7 | 갤러리 워크 | Pro | peek-others | 상호 관람이 목적 |
| 8 | 6색 모자 | Pro | own-only | 모자별 역할 독립 (Pro 예비) |
| — | 월드카페 | **v2 파킹** | peek-others | 호스트 지정·시간 로테이션 운영 복잡도 초과 |

---

## 2. 데이터 모델 (Prisma 초안)

```prisma
model BreakoutTemplate {
  id                      String   @id @default(cuid())
  name                    String
  tier                    String   // "free" | "pro"
  requiresPro             Boolean  @default(false)
  scope                   String   @default("system") // "system" | "teacher" | "school"
  ownerId                 String?  // teacher 커스텀인 경우
  structure               Json     // { teacherPool?: SectionSpec, sectionsPerGroup: SectionSpec[] }
  recommendedVisibility   String   // "own-only" | "peek-others"
  defaultGroupCount       Int      @default(4)
  defaultGroupCapacity    Int      @default(6)
  maxGroupCount           Int      @default(10)
  version                 Int      @default(1)
  createdAt               DateTime @default(now())
  updatedAt               DateTime @updatedAt

  owner        User?               @relation(fields: [ownerId], references: [id], onDelete: SetNull)
  assignments  BreakoutAssignment[]

  @@index([scope, tier])
  @@index([ownerId])
}

model BreakoutAssignment {
  id                  String   @id @default(cuid())
  boardId             String   @unique
  templateId          String
  deployMode          String   // "link-fixed" | "self-select" | "teacher-assign"
  groupCount          Int
  groupCapacity       Int
  visibilityOverride  String   // "own-only" | "peek-others"
  status              String   @default("draft") // "draft" | "live" | "closed" | "archived"
  isPublic            Boolean  @default(true)    // 반 공개 기본
  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  board     Board            @relation(fields: [boardId], references: [id], onDelete: Cascade)
  template  BreakoutTemplate @relation(fields: [templateId], references: [id])
  members   BreakoutMembership[]

  @@index([templateId])
  @@index([status])
}

model BreakoutMembership {
  id           String @id @default(cuid())
  assignmentId String
  sectionId    String   // Section FK — 모둠 = 섹션 재활용
  studentId    String
  role         String?  // Jigsaw 용: "expert" | "home" | null
  assignedAt   DateTime @default(now())
  assignedBy   String   // "self" | teacherUserId

  assignment BreakoutAssignment @relation(fields: [assignmentId], references: [id], onDelete: Cascade)

  @@unique([sectionId, studentId])   // 한 섹션에 학생 1회
  @@index([assignmentId])
  @@index([studentId])
}
```

**Section 연결**: `Section.role` 값 확장 — `"student-copy" | "teacher-pool" | "gallery"` 등. `Section.accessToken`은 `link-fixed` 배포 시 모둠별 URL 생성에 사용.

**Card 이동 정책 (Phase 3 Q5)**: `Card.sectionId`는 생성 시 고정. 학생이 이동(재배정)돼도 이전 모둠에 쓴 카드는 원래 섹션에 남는다(sketch R6 정책).

---

## 3. 작업 분할 (BR-1 ~ BR-9)

| # | 작업 | 의존 | 공수 |
|---|---|---|---|
| **BR-1** | 스키마 마이그레이션 — `BreakoutTemplate` / `BreakoutAssignment` / `BreakoutMembership` + `Section.role`/`accessToken` 확장 | — | 1.5일 |
| **BR-2** | 시스템 템플릿 8종 seed (`data/breakout-template-seed.json`) + `npm run seed:breakout` 멱등 스크립트 | BR-1 | 2일 |
| **BR-3** | 교사 개설 플로우 UI — 템플릿 피커 → 모둠 수/정원 → deployMode/visibilityOverride 선택 → Draft 생성 | BR-1, BR-2 | 3일 |
| **BR-4** | 모둠 섹션 복제 엔진 — `BreakoutTemplate.structure.sectionsPerGroup[]` × N 복제, teacher-pool 단일 생성, Membership 초기 INSERT | BR-3 | 3일 |
| **BR-5** | 배포모드 3종 구현 — `link-fixed`(accessToken 발급) / `self-select`(모둠 목록 + capacity 표시) / `teacher-assign`(드래그 배정 UI) | BR-4, **T0-①** | 4일 |
| **BR-6** | 열람모드 2종 + 교사 전체 접근 RBAC — `own-only`/`peek-others`를 `viewSection` 권한에 매핑, 교사 상위 집합 보장 | BR-4, **T0-①** | 2일 |
| **BR-7** | "모든 모둠에 이 카드 복제" 단발 액션 — 교사 UI 버튼 + 확인 모달 + 서버 N모둠 INSERT 반복 + 실패 rollback | BR-4 | 2일 |
| **BR-8** | 교사 커스텀 템플릿 저장 — 기존 Board → "템플릿으로 저장" 플로우, 구조 JSON 직렬화, Free 3개/Pro 무제한 쿼터 게이트 | BR-2, BR-3 | 3일 |
| **BR-9** | 학교 공용 템플릿 등록(`scope="school"`, Pro 전용) + 관리자 승인 큐 | BR-8 | 2일 |

**추가 소작업 (파킹 후보, 본 로드맵 변경 로그에 기록)**:
- BR-A "모든 모둠에 복제" **되돌리기(undo)** — 단발 액션 취소 히스토리
- BR-B 신고 버튼 — `teacher-assign` 모드 학생용 "배정 이상" 신고
- BR-C 학생 셀프 모둠 이동 (v2)
- BR-D 월드카페 템플릿 (v2, 호스트 지정 + 시간 로테이션)

---

## 4. 의존 로드맵과의 상호 참조

### 4.1 tablet-performance-roadmap T0-①

T0-① "섹션 격리 Breakout 뷰"는 본 기능의 **구현 전제**. BR-5/BR-6은 T0-①의 `src/app/board/[id]/s/[sectionId]/page.tsx` + `src/lib/realtime.ts` 채널 키(`board:${bid}:section:${sid}`) + `viewSection` RBAC 인프라에 직접 의존한다.

> T0-①의 `Section.accessToken`이 본 로드맵의 `link-fixed` 배포 모드 토큰과 같은 필드다. 중복 마이그레이션 금지.

### 4.2 drawing-board-library-roadmap

- **복사 패턴 재사용**: `AssetAttachment`의 "카드에 붙을 땐 복사 기본(제출물 안정성) + 라이브러리는 역참조 유지" 원칙을 본 로드맵의 템플릿 복제 방식이 승계한다(§1.4).
- **템플릿 선택 모달 패턴 재사용**: drawing-board의 "학생이 '템플릿 고르기' 모달에서 1개 선택(빈 캔버스 옵션 포함)" UX를 교사 개설 플로우(BR-3)의 템플릿 피커에 재사용 — 동일 컴포넌트 추출 검토.

### 4.3 Seed 2 Tier 매트릭스

§1.5 Tier 매트릭스는 Seed 2(`seed_8967b77d7759`)의 정책(Free 5반 / Pro ₩9,900 / 쿼터 초과 시 안내 모달)과 1:1 호환. 가격·반 수 변경은 Seed 2에서 일괄 관리.

---

## 5. 수용 기준 (seed.yaml acceptance_criteria 그대로)

- [ ] BreakoutTemplate에 id·name·tier·structure·recommendedVisibility·defaultGroupCount·defaultGroupCapacity 필드 존재
- [ ] BreakoutAssignment에 id·boardId·templateId·deployMode·groupCount·groupCapacity·visibilityOverride·status 필드 존재
- [ ] 배포모드 3종(link-fixed · self-select · teacher-assign) 모두 동작
- [ ] 열람모드 2종(own-only · peek-others) 학생 기준 적용, 교사 항상 전체 접근
- [ ] 모둠 기본값 4모둠 · 정원 6 · 상한 10모둠이 BreakoutAssignment 생성 시 기본 적용
- [ ] v1 시스템 템플릿 7종 + Pro 예비 1종(6색 모자) 총 8종 등록
- [ ] Free 3종 접근 가능, 나머지 5종 Pro 전용
- [ ] recommendedVisibility: own-only 5종(KWL · 브레인스토밍 · 찬반 토론 · Jigsaw · 아이스브레이커) · peek-others 2종(갤러리 워크 · 모둠 발표 준비)
- [ ] 한 모둠 섹션 수정이 타 모둠에 전파 안 됨
- [ ] 교사 UI "모든 모둠에 이 카드 복제" 단발 액션 버튼 존재
- [ ] 템플릿 원본 수정이 기존 Board에 역전파 X
- [ ] 반 공개(public) 기본값 적용
- [ ] **T0 성능 게이트 통과** (tablet-performance-roadmap §11): 갤탭 S6 Lite 30명·모둠 10개·섹션당 카드 50장 기준 TTI < 3s

---

## 6. 리스크 & 완화

| 리스크 | 완화 |
|---|---|
| 모둠 10개 × 섹션당 카드 50장 렌더 폭발 | T0-① 섹션 격리 뷰 강제. 교사 통합 뷰는 "섹션 목록 + 요약", 상세는 섹션별 진입. |
| Jigsaw 시 한 학생 두 섹션 소속 → Membership unique 위반 | `BreakoutMembership.role`(`expert`/`home`)로 구분, unique는 `(sectionId, studentId)` 유지. |
| 템플릿 버전 업 시 기존 Board 기대 혼란 | version 필드 표시 UI + "템플릿 업데이트됨 — 새 Board에만 반영" 안내. |
| teacher-pool 섹션이 N모둠에 broadcast되어 성능 저하 | teacher-pool은 **보드 레벨 단일 섹션** — WS 채널 분리로 구독 학생만 수신. |
| "모든 모둠에 복제" 중 일부 실패 시 partial state | 트랜잭션 + rollback 또는 결과 보고 모달(성공/실패 모둠 명시). |

---

## 7. 파킹 (v2+)

- 월드카페 템플릿 (호스트 학생 지정 + 시간 단위 로테이션)
- 학생 셀프 모둠 이동 (저작권 꼬임 + WS 채널 재구독 비용)
- "모든 모둠에 복제" 되돌리기 UX
- 교사 커스텀 템플릿 공유 마켓 (Pro 간 커뮤니티)
- Jigsaw expert → home 자동 전환 워크플로우

---

## 8. 학부모 열람 범위 (Seed 7 통합, 2026-04-12)

Breakout Room 모듈의 학부모 열람은 **Seed 7 `seed_37b35654542f`** (`plans/parent-viewer-roadmap.md`)의 **자녀 범위 매트릭스 §5**로 일원화됐다. 본 로드맵의 `visibility` 축(own-only/peek-others)과 직교하며, 학부모는 별도 축으로 취급된다.

| 축 | 값 |
|---|---|
| 엔티티 | `BreakoutAssignment` · `BreakoutMembership` · `Section` · `Card` |
| 학부모 열람 | **자녀 본인 세션 결과·제출물만**. peek-others 설정이어도 `/parent/*`에서는 타 모둠 노출 **X** |
| 서버 필터 | `BreakoutMembership.studentId ∈ parent.children` → 해당 `sectionId` + 같은 `assignmentId`의 자녀 본인 Card만 반환 |
| teacher-pool 섹션 | 학부모 뷰에서 **항상 제외** (`Section.role == "teacher-pool"` 필터링) |
| Jigsaw expert/home | 자녀가 멤버인 섹션만 — expert 섹션도 자녀가 expert로 속해 있으면 열람, 아니면 비노출 |
| visibilityOverride 영향 | 없음 — 학부모 열람은 visibility 설정과 **직교**. 교사 `peek-others`여도 학부모는 자녀 모둠만 |
| "모든 모둠에 복제" 영향 | 복제된 카드도 각 섹션의 `sectionId` 기준 필터링 — 자녀 섹션에 있는 복제본만 열람 |
| 진입점 | `/parent/child/[id]/breakout` — 자녀 참여 세션 요약 리스트 + 섹션 카드 읽기 전용 |
| v1 | 순수 read-only. 학부모 댓글·응원 X |

구현 작업은 Seed 7의 **PV-7 (자녀 범위 서버 필터)** 에서 처리한다. 본 로드맵의 BR-5 (deployMode)·BR-6 (visibility) 구현은 학생·교사 관점만 담당하며, 학부모 관점은 Seed 7에서 별도 `/parent/*` 엔드포인트로 다룬다.

> 교사의 전체 접근 RBAC(§1.2)와 학부모의 자녀 귀속 필터는 **두 개의 직교 RBAC 축** — 절대 혼동 금지. 학부모는 교사가 아니며, `BoardMember.role="parent"`는 `viewSection` 중 `studentId ∈ parent.children` 교집합만 허용한다.

---

## 9. 변경 로그

| 날짜 | 시드 ID | 변경 |
|---|---|---|
| 2026-04-12 | `seed_bb1d4eb1c442` | 초안 생성 — 신규 엔티티 2종, 템플릿 8종, Free 3/Pro 5, 복사 방식, own-only 기본, v1 학생 이동 불허 확정. BR-1~BR-9 작업 분할. |
| 2026-04-12 | `seed_37b35654542f` (Seed 7) | §8 학부모 열람 범위 절 신규 추가 — Seed 7 매트릭스(`parent-viewer-roadmap.md#5`) 참조. `BreakoutMembership.studentId ∈ parent.children` 서버 필터, teacher-pool 제외, peek-others 무관 자녀 모둠만 열람 규칙 확정. 구현은 Seed 7 PV-7에서 처리. |
