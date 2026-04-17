# 과제 배부 보드 — 설계 노트 (v1)

> 작성일: 2026-04-14
> 관련: `tablet-performance-roadmap.md`, `event-signup-roadmap.md`, `parent-viewer-roadmap.md`, `implementation-roadmap.md`
> 동기: Padlet에 없는 **학급 로스터 기반 과제 수거 보드** — 학생 번호순 5×6 정형 격자, Seesaw 썸네일 + Moodle 이원 상태 배지, Classroom식 제출물 모달.
> 구조: `Board.layout = "assignment"` + 신규 `AssignmentSlot` 엔티티 1종
> 시드: **`seed_38c34e91bf28`** (ambiguity 0.083, interview_20260414_131412)

---

## 0. 핵심 명제

> **북극성 5 primitive 고정**: (1) 학급 로스터 기반 카드 자동 생성 · (2) 제출/미제출 시각 구분 · (3) 카드 클릭 → 전체화면 모달 · (4) 교사 가이드 영역 + 학생 영역 분리 · (5) 번호순 5×6 정형 격자

---

## 1. Board 확장 (`layout = "assignment"`)

```prisma
model Board {
  // ... 기존 유지 ...
  layout      String @default("freeform")   // + "assignment" (Breakout "breakout" / event-signup "event-signup"과 동일 패턴)

  // Assignment 전용 메타 (layout = "assignment"일 때만 유효)
  assignmentDueAt         DateTime?
  assignmentAllowLate     Boolean  @default(true)
  // 교사 가이드 — 단일 텍스트 필드 확정 (Q1 결정: Section role 확장 폐기)
  assignmentGuideText     String   @default("")

  assignmentSlots AssignmentSlot[]
}
```

- `Section.role` 필드는 도입하지 않음 (Q1 결정). 동영상·여러 카드 수요는 v2에서 Section role 확장으로 승격.
- `Board.galleryMode` / `galleryReleasedAt`은 **v2 스키마 예약** (Q5). v1 스키마 미포함.

## 2. 신규 엔티티 `AssignmentSlot`

```prisma
// 과제 수거 보드의 학생별 "자리" — 1 Board × 1 Student 유니크.
// Board 생성 시 classroomId의 Student 전원(N≤30)만큼 자동 인스턴스화.
// v1: 학생 추가/삭제 시 교사 "Roster 동기화" 버튼 수동 트리거. v2: 자동.
// 카드 자체는 일반 Card 재사용 — slot은 "자리(고정·1:1)" + card는 "내용물".
model AssignmentSlot {
  id               String   @id @default(cuid())
  boardId          String
  studentId        String
  // 1~30. 생성 시점 Student.number 스냅샷 (Q6 결정: Student.number 변경해도 불변)
  slotNumber       Int
  // Card FK — 학생이 제출 시 생성. 미제출이면 null.
  cardId           String?  @unique
  // Submission FK — 1:1. 학생이 제출 시점에 생성.
  submissionId     String?  @unique
  // submissionStatus 전이:
  //   assigned → viewed(옵션) → submitted → returned → submitted(재제출) → reviewed
  //   추가 값 "orphaned" — Q6: 학생 소프트 삭제 시 dimmed 읽기 전용
  submissionStatus String   @default("assigned")
  // Moodle 이원 상태 중 gradingStatus 축 (owner 리뷰 시 갱신)
  gradingStatus    String   @default("not_graded") // "not_graded" | "graded" | "released"
  viewedAt         DateTime?
  submittedAt      DateTime?
  returnedAt       DateTime?
  // Q7 결정: 반려 사유 1줄(≤200자) 필수. Submission.feedback 재사용 가능 시 생략 가능하지만
  //         slot 차원 메타(감사·빠른 조회)로 여기도 보관.
  returnReason     String?  @db.VarChar(200)
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  board      Board       @relation(fields: [boardId], references: [id], onDelete: Cascade)
  student    Student     @relation(fields: [studentId], references: [id], onDelete: Cascade)
  card       Card?       @relation(fields: [cardId], references: [id], onDelete: SetNull)
  submission Submission? @relation(fields: [submissionId], references: [id], onDelete: SetNull)

  @@unique([boardId, studentId])       // 1인 1slot 강제
  @@unique([boardId, slotNumber])      // 격자 좌표 유일성
  @@index([boardId])
  @@index([studentId])
  @@index([boardId, submissionStatus]) // 미제출 필터
}
```

## 3. 상태 머신 (확정)

### 3.1 submissionStatus 전이

```
assigned ─(학생 첫 모달 진입)─> viewed ─(제출)─> submitted ─(교사 반려)─> returned ─(학생 재제출)─> submitted
                              └(제출)──────────> submitted ─(교사 리뷰 완료)─> reviewed
                                                                ↓ Q6
                                                           orphaned (학생 소프트 삭제 시)
```

### 3.2 재제출 정책 매트릭스 (Q2 확정)

| 조건 | 학생 재제출 동작 |
|---|---|
| `gradingStatus="not_graded"` & 마감 전 | Card 내용 in-place 갱신, `Submission.updatedAt` 갱신. 이력 보존 無 (v1) |
| `gradingStatus="not_graded"` & 마감 후 | `assignmentAllowLate=true`면 허용, `false`면 차단 |
| `gradingStatus="graded"` 또는 `"released"` | 재제출 버튼 비활성 — 교사 `returned` 액션 필요 |
| `submissionStatus="returned"` | 학생 수정 허용. 제출 시 `returned → submitted` 재전이, `gradingStatus` 즉시 `not_graded`로 리셋 |

### 3.3 v1 히스토리 모델

- **덮어쓰기**. `SubmissionHistory` 엔티티는 **v2 파킹**.
- v1 유일 히스토리: `Submission.updatedAt`.

---

## 4. UI 엔트리포인트 (확정)

| 액션 | 주체 | 엔트리포인트 | 비고 |
|---|---|---|---|
| 보드 생성 | owner | `/(teacher)/boards/new?layout=assignment` | 학급 선택 → N≤30 검증 → N개 slot 일괄 insert |
| 가이드 편집 | owner | 보드 뷰 상단 owner-only 텍스트 영역 | `Board.assignmentGuideText` 단일 필드 |
| 제출 열람 | owner | **카드 클릭 → 전체화면 모달 전용** | 격자 뷰 롱탭·우클릭 컨텍스트 메뉴 도입 안 함 |
| 반려(`returned`) | owner | **모달 내부 "반려" 버튼** | Q7: 반려 사유 1줄(≤200자) 필수 |
| 리뷰 완료(`reviewed`) | owner | 모달 내부 "리뷰 완료" 버튼 | `gradingStatus=graded` 동반 갱신 |
| 미제출 독려 | owner | 보드 상단 "미제출 필터" → "일괄 독려" | Q4: **인앱 배지만**. 학부모 이메일 경유 v1 제외 |
| 제출 | editor | 본인 slot 카드 → 모달 (작성 모드) | 타 학생 slot은 읽기 전용 (Q5: v1 비공개 고정) |
| 재제출 | editor | 본인 slot 모달 "수정" 버튼 | §3.2 매트릭스로 게이팅 |
| 반려 사유 확인 | editor | 모달 재진입 시 **상단 고정 배너** | Q7: 배너 미닫음 정책. `returned` 카드는 격자에서 `"!"` 배지 |

**태블릿 S-Pen 인터랙션 일관성**: 카드 표면 S-Pen 금지, 모달 내부에서만 필기/캔버스 마운트. 카드 `onTap` 핸들러 단일화(모달 오픈).

---

## 5. 학급 N 정책 (Q3 확정)

- **v1 하드 제한: N≤30**. `Classroom`의 Student 수 > 30이면 보드 생성 **차단** + "분반 또는 v2 대기" 안내.
- 5×8 (N=40) 자동 확장은 v2 파킹.
- 근거: "자리표 인지 모델"이 5×6 위에서 설계됨. 격자 가변화 시 학생 공간 기억 학습 비용 증가. 태블릿 DOM 예산은 여유 있으나 UX 의미가 깨짐.

---

## 6. 역할별 사용자 흐름

### 6.1 교사(owner)
1. 대시보드 → "과제 보드 만들기" → 학급 선택 (이미 teacherId로 필터링됨).
2. 제목 · 마감일 · 가이드 텍스트(= `Board.assignmentGuideText`) 입력.
3. "생성" → 서버: `Board(layout="assignment")` + N개 `AssignmentSlot` 트랜잭션 일괄 insert (slotNumber = Student.number 스냅샷).
4. 보드 뷰: 상단 가이드 + 하단 5×6 격자(30 slot 중 N개 활성, 나머지 비활성 플레이스홀더).
5. 미제출 필터 → 일괄 독려(인앱 알림).
6. 카드 클릭 → 전체화면 모달에서 리뷰 완료 / 반려(사유 필수).
7. Roster 동기화: 학기 중 전학·추가 발생 시 수동 버튼. 기존 slot 보존, 신규 slot 추가, 삭제된 학생은 `submissionStatus="orphaned"`로 dim.

### 6.2 학생(editor)
1. 학급 로그인 → 보드 접속 → 본인 slotNumber 카드 하이라이트 + 자동 스크롤.
2. 본인 카드 탭 → 전체화면 모달 (작성 모드). 첫 진입 시 `viewedAt` 갱신 (`assigned → viewed`).
3. 텍스트 / 이미지 / 첨부 / 링크 입력(Card 포맷 재사용) → "제출" → `submissionStatus=submitted`.
4. 재제출: §3.2 매트릭스로 게이팅.
5. `returned` 재진입 시 반려 사유 상단 고정 배너 노출.
6. 타 학생 slot은 **v1 비공개 고정** (Q5). 읽기조차 불가.

### 6.3 학부모(viewer)
- `parent-viewer-roadmap.md` v2 (`seed_6d7077aac472`) **자녀 범위 매트릭스 §5**에 편입.
- 학부모는 본인 자녀의 `AssignmentSlot` + 해당 slot의 Card/Submission·returnReason만 열람. 격자 자체 미노출, **자녀 전용 단일 뷰**로 축약.
- 교사 가이드 텍스트는 viewer-readable.
- matrix/grid 뷰: **owner + 데스크톱 전용**. editor·viewer·태블릿 전부 제외.

---

## 7. 태블릿 성능 체크리스트 (기준: 갤럭시 탭 S6 Lite · Chrome Android)

30-카드 5×6 정형 격자 성능 예산은 `tablet-performance-roadmap.md §2a` (신규)에 공식 등재. 본 로드맵 항목별 게이트:

- [ ] **초기 TTI < 3s** (§2 예산 승계)
- [ ] **DOM 카드 노드 ≤ 30** · 카드당 자식 ≤ 6 (이름·번호·상태 배지·썸네일·아이콘·hitbox)
- [ ] 썸네일 **서버 리사이즈 160×120 WebP 단일 장** + `loading="lazy"` + IntersectionObserver (T0-④ 재사용)
- [ ] 카드 표면 S-Pen 캔버스 금지 — 필기는 모달 내부에서만 마운트
- [ ] 상태 토글은 순수 CSS (`data-submission-status`) → React 리렌더 회피
- [ ] WebSocket 채널 `board:${id}:assignment` 단일, slot별 분리 금지. 메시지 < 200B (slotId + status 델타). 100ms 디바운싱
- [ ] 모달 지연 마운트 · 닫기 시 즉시 언마운트
- [ ] iframe 금지 (v1). v2에서 "라이브 보기" 단일 iframe 허용 검토
- [ ] 드래그 비활성 — `DraggableCard` 대신 `StaticSlotCard` 컴포넌트 신규
- [ ] 메모리 1시간 후 < 500MB (전역 예산 승계)

---

## 8. 관련 로드맵 관계

### 8.1 `event-signup-roadmap.md` (Seed 3, `seed_43fdf181262f`)
- **Submission 엔티티 공유**. v1 충돌 없음:
  - event-signup: `Submission.status` ∈ `{submitted, approved, rejected, waitlist, withdrawn, pending_approval}`
  - assignment: `AssignmentSlot.submissionStatus` ∈ `{assigned, viewed, submitted, returned, reviewed, orphaned}` — **slot 레벨 필드**로 분리되어 `Submission.status`와 네임스페이스 충돌 없음.
- 재사용 합의: `Submission`은 콘텐츠 컨테이너(content·linkUrl·fileUrl·feedback·updatedAt). Assignment는 별도 slot 엔티티가 1:1로 감싸고 상태를 들고 있음.
- Zod 검증에서 `Board.layout` 기반 분기로 `Submission.status` 허용 값 제한(R3 완화).

### 8.2 `tablet-performance-roadmap.md`
- §2 성능 예산에 **§2a 30-카드 5×6 정형 격자 게이트**를 신규 추가 (본 파일 §7 항목을 전역 QA에 편입).

### 8.3 `parent-viewer-roadmap.md` (Seed 7-v2)
- 자녀 범위 매트릭스에 `AssignmentSlot` 행 추가 필요 (PV-7 서버 필터에서 처리). 본 로드맵은 owner/editor 로직만 구현.

### 8.4 `implementation-roadmap.md` (Canva 통합) + `canva-assignment-pdf-merge/`
- 학생이 Canva Content Publisher 앱으로 게시 시 제목 규칙 `완료-{Board.title}-{Student.number}-{Student.name}` 자동 생성 → `Card.canvaDesignId`에 저장 → slot 연결.
- 교사 "완료본 병합" 버튼: slot 중 `submissionStatus="submitted"` + `Card.canvaDesignId IS NOT NULL` 필터 → 기존 `canva-assignment-pdf-merge` 스킬로 PDF 병합.
- **v1 범위에서 Canva 연동은 선택** — 미사용 제출도 정상 동작.

---

## 9. 작업 분할 (AB-1 ~ AB-10)

| 단계 | 내용 |
|---|---|
| AB-1 | Prisma: `Board.layout "assignment"` 확장 + 3 필드 + `AssignmentSlot` 엔티티 마이그레이션 |
| AB-2 | 보드 생성 서버 액션 — 학급 검증(N≤30) + slot 트랜잭션 일괄 insert |
| AB-3 | 보드 뷰(owner) — 상단 `assignmentGuideText` 영역 + 하단 5×6 `StaticSlotCard` 격자 (CSS Grid + `order: slotNumber`) |
| AB-4 | `StaticSlotCard` 컴포넌트 — 썸네일 lazy + `data-submission-status` CSS 토글 |
| AB-5 | 전체화면 모달(owner·editor 공용) — 지연 마운트 + 반려·리뷰 완료·재제출 버튼 + 반려 사유 배너 |
| AB-6 | 상태 머신 서버 검증 — §3.2 재제출 매트릭스 zod 가드 |
| AB-7 | 미제출 필터 + 일괄 독려 인앱 알림 (기존 알림 인프라 재사용) |
| AB-8 | RBAC — `viewAssignmentSlot`(owner all / editor own-slot / parent child-slot) · `reviewAssignmentSlot`(owner only) |
| AB-9 | Roster 동기화 버튼 (수동) — 신규 slot 추가 + 삭제된 학생 `orphaned` 마킹 |
| AB-10 | 학부모 자녀 뷰 필터 (`parent-viewer-roadmap.md` PV-7 협업 — 본 로드맵에서는 서버 쿼리만 제공) |

Canva 시너지(제목 규칙 + PDF 병합)는 v1.5 후속, `implementation-roadmap.md` P0-② 후속 작업으로 처리.

---

## 10. 수용 기준 핵심

- [ ] 교사가 학급 로스터(N≤30)로 보드 생성 시 N개 slot이 5×6 격자에 **Student.number 순서대로** 결정적 배치
- [ ] 제출/미제출 카드가 한눈에 시각 구분 (썸네일 채움 + 상태 배지)
- [ ] 카드 클릭 → 전체화면 모달 (사이드패널 **없음**)
- [ ] 교사 가이드 영역 owner-only 상단 · 학생 slot 격자 하단 레이아웃
- [ ] 상태 머신 §3.1 + 재제출 매트릭스 §3.2 서버 검증 통과
- [ ] 반려 액션은 모달 내부에서만 가능 · 사유 ≤200자 필수
- [ ] 학생이 반려 사유를 모달 상단 배너로 확인
- [ ] `returned` 카드에 격자 뷰 `"!"` 배지
- [ ] 타 학생 slot 읽기 **차단** (API + DOM + RLS 3중)
- [ ] 미제출 독려 인앱 배지 발송 · 외부 이메일 채널 없음
- [ ] 썸네일 160×120 WebP 서버 리사이즈 + IntersectionObserver lazy
- [ ] `assignmentAllowLate=false`에서 마감 후 제출 차단 검증
- [ ] matrix/grid 뷰는 owner + 데스크톱 전용, editor·viewer·태블릿 전부 제외

---

## 11. v2+ 파킹

- `Section(role="guide")` + Card 재사용으로 가이드 영역 확장 (동영상·여러 카드)
- `SubmissionHistory` 엔티티 승격 — 재제출 이력 보존
- 5×8 / N>30 자동 확장 (자동 vs 교사 커스텀 좌석)
- 학부모 이메일·푸시 독려 채널 (parent-viewer-roadmap §1.3 통합)
- 갤러리 워크 모드 (`Board.galleryMode`, 교사 공개 액션 기반)
- 풀 코멘트 시스템 (스레드·멘션·읽음)
- Roster 자동 동기화 (Student CRUD 트리거)
- matrix 뷰(owner + 데스크톱) 별도 라우트

---

## 12. 리스크

| # | 리스크 | 영향 | 완화 |
|---|---|---|---|
| R1 | Roster 변경 시 slot 싱크 불일치 | 중 | 생성 시점 스냅샷(Q6) + 수동 동기화 버튼 + `orphaned` 상태 보존 |
| R2 | Submission 재사용으로 event-signup과 status enum 충돌 | 중 | slot의 `submissionStatus` 필드로 네임스페이스 분리. layout별 zod 분기 |
| R3 | 학부모 DOM 누출 (타 학생 격자 노출) | 고 | viewer는 격자 자체 미렌더, 자녀 단일 뷰로 축약. API/RLS 3중 |
| R4 | `graded` 이후 학생이 임의 덮어쓰기 | 고 | §3.2 매트릭스 서버 가드. 재제출 버튼 DOM 비활성 + API 403 |
| R5 | 반려 사유 누락 | 저 | 서버 zod `returnReason.length ≥ 1 && ≤ 200`로 강제 |
| R6 | 태블릿 동시 제출 30 WebSocket 폭발 | 고 | 단일 채널 + 델타 페이로드 + 100ms 디바운싱 |
| R7 | matrix 뷰 요건이 v1에 흘러감 | 중 | v1은 matrix 제외, v2에서 별도 라우트 |

---

## 변경 로그

| 날짜 | 시드 ID | 변경 |
|---|---|---|
| 2026-04-14 | `seed_38c34e91bf28` | 신규 로드맵 생성. 7건 확정 결정(Q1~Q7) + AssignmentSlot 스키마 + AB-1~AB-10 작업 분할. |

---

## 관련 주제

- [`./assessment-autograde-roadmap.md`](./assessment-autograde-roadmap.md) — 수행평가 자동채점 파이프라인 (Seed 12, `seed_0badf1e571bc`, 2026-04-16). **채점·성적 송신 레이어는 본 assignment-board가 아닌 assessment-autograde가 담당한다.** assignment-board는 제출·반려·Roster 기반 배부 레이어에 집중하며, 수행평가식 자동채점·MCQ/SHORT LLM·GradebookEntry·성적 릴리스는 assessment-autograde의 범위다.
