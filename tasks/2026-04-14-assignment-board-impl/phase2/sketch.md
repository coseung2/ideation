# Phase 2 — Sketch: Roster-Bound Assignment Collection Board

- **task_id**: `2026-04-14-assignment-board-impl`
- **전제**: phase1 하이브리드 권고 고정 (북극성 5 primitive + 공개 레퍼런스 1:1 매핑)
- **기준 단말**: 갤럭시 탭 S6 Lite (Chrome Android, S-Pen), 학급 N=30

---

## 1. 전제 결정 (phase1 고정점 재확인)

### 1.1 북극성 (고정)

1. **학급 로스터 기반 카드 자동 생성** — Classroom → Student N명 → N개 AssignmentSlot 카드 자동 인스턴스화
2. **제출/미제출 시각 구분** — 카드 표면 색상 + 배지 (채움/공란/옅은 경고/중간 상태)
3. **카드 클릭 → 학생 제출물 뷰** — 전체화면 모달 (태블릿 세로폭 제약으로 사이드패널은 제외)
4. **교사 가이드 영역 + 학생 영역 분리** — 보드 상단 owner-only 안내 영역, 하단 student 카드 그리드
5. **번호순 5×6 정형 격자** — `(row, col) = (⌈n/5⌉, ((n-1) mod 5)+1)` 결정적 배치, 자유 드래그 금지

### 1.2 phase1 "핵심 결정 후보 8건"에 대한 에이전트 자율 결정

phase1이 남긴 후보에 대해 기존 스키마·primitive 정합성·태블릿 제약·선행 seed(Breakout BR-1 — `Board.layout` 확장 방식) 관례에 따라 다음과 같이 초벌 결정 (phase3에서 사용자 확인 가능):

| 후보 | 초벌 결정 | 근거 |
|---|---|---|
| ① 신규 BoardType vs 기존 Board 확장 | **기존 Board 확장** — `Board.layout = "assignment"` (이미 schema 주석에 예약됨) | schema.prisma 163 라인 주석 `"assignment"` 명시. 보드 은유 통일. Breakout(BR-1)과 동일 패턴 |
| ② AssignmentCard 신규 vs Card 재사용 | **AssignmentSlot 신규 엔티티 + 기존 Card 재사용(slot의 `cardId` FK)** | 1:1 slot 제약·ownerStudentId·status는 Card에 섞기엔 범용성 침해. slot은 "자리(고정)" + card는 "내용물(1회 작성)"로 분리 |
| ③ 제출물 저장 Submission 재사용 | **Submission 재사용 + status enum 확장** | 이미 schema 299라인 주석에 `"submitted" \| "reviewed" \| "returned"` 예약. 스키마 충돌 없음. slot ↔ Submission 연결은 slot에 `submissionId` FK |
| ④ Classroom/Roster 연결 | **기존 `Classroom` + `Student` 재사용** — `Board.classroomId` 이미 존재. Assignment 생성 시 Student 전원 조회해 slot 인스턴스화 | Seed 7-v2 ClassInviteCode 체계 무간섭. 신규 엔티티 0개 |
| ⑤ 교사 가이드 영역 | **기존 Section 재사용 + `role="guide"` 필드** — 보드 상단 고정 1개 `SECTION(role=guide)`에 일반 Card 배치 | Breakout이 Section을 유사하게 재활용한 선례. 신규 엔티티 0개 |
| ⑥ 5×6 격자 고정 | **결정적** — `Student.number`로 좌표 산출. v1 교사 커스텀 좌석 배치 금지 | 북극성 5 "자리표" 은유 고정. 커스텀은 v2 파킹 |
| ⑦ 카드 클릭 뷰 모달 vs 사이드패널 | **전체화면 모달** | 탭 S6 Lite 세로모드 1200×800 수준, 사이드패널은 양쪽 다 좁아짐. 모달이 lazy-load 언마운트도 명확 |
| ⑧ 썸네일 파이프라인 | **서버 리사이즈 단일 저해상도 (160×120 WebP) + `loading="lazy"` + IntersectionObserver** | tablet-performance-roadmap §2·§8 직결. 기존 T0-④ 파이프라인 재사용 |

---

## 2. Prisma 초안 (기존 스키마 확장)

### 2.1 신규 엔티티: `AssignmentSlot`

```prisma
// 과제 수거 보드의 학생별 "자리" — 1 Board × 1 Student 유니크.
// Board 생성 시 classroomId의 Student 전원만큼 자동 인스턴스화.
// Student 추가/제거 시 slot 자동 동기화 (v1 교사 수동 재동기화 버튼 + v2 자동 트리거).
// 카드 자체는 일반 Card 재사용 — slot은 "자리(고정·1:1)" + card는 "내용물(학생이 작성)".
model AssignmentSlot {
  id              String   @id @default(cuid())
  boardId         String
  studentId       String
  // 1~30 (= Student.number 미러링, 5×6 격자 좌표 산출용)
  slotNumber      Int
  // Card FK — 학생이 제출 시 생성. 미제출이면 null.
  cardId          String?  @unique
  // Submission FK — 1:1. v1 생성 시점 = 카드 생성 시점.
  submissionId    String?  @unique
  // status 전이: "assigned" → "viewed"(옵션, Teams식) → "submitted" → "returned" → "reviewed"
  // Moodle 이원 상태 중 submissionStatus 축.
  submissionStatus String  @default("assigned")
  // Moodle 이원 상태 중 gradingStatus 축 (owner가 리뷰한 이후 갱신).
  gradingStatus    String  @default("not_graded") // "not_graded" | "graded" | "released"
  viewedAt        DateTime? // 학생이 자기 카드에 처음 진입한 시각
  submittedAt     DateTime?
  returnedAt      DateTime? // 교사가 "다시 하기"로 되돌린 시각
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  board      Board       @relation(fields: [boardId], references: [id], onDelete: Cascade)
  student    Student     @relation(fields: [studentId], references: [id], onDelete: Cascade)
  card       Card?       @relation(fields: [cardId], references: [id], onDelete: SetNull)
  submission Submission? @relation(fields: [submissionId], references: [id], onDelete: SetNull)

  @@unique([boardId, studentId])      // 1인 1slot 강제
  @@unique([boardId, slotNumber])     // 격자 좌표 유일성
  @@index([boardId])
  @@index([studentId])
  @@index([boardId, submissionStatus]) // 미제출 필터
}
```

### 2.2 `Board` 확장 (신규 필드)

```prisma
model Board {
  // ... 기존 유지 ...
  // Assignment 전용 메타 (layout = "assignment"일 때만 유효)
  assignmentDueAt     DateTime?  // 마감. Canvas missing policy 참고 — 마감 후 상태 자동 전이 옵션
  assignmentAllowLate Boolean    @default(true)
  assignmentGuideText String     @default("") // 교사 가이드 (Section 대신 Board 필드로 둘지는 결정 ⑤ 참고)
  // layout 값에 "assignment" 추가 (기존 163라인 주석 실현)

  // 관계
  assignmentSlots AssignmentSlot[]
}
```

### 2.3 `Section` 확장 (결정 ⑤ 채택 시)

```prisma
model Section {
  // ... 기존 유지 ...
  // "guide" = 교사 가이드 섹션 (owner-only 쓰기), null = 일반 섹션
  role String? // nullable, 기본값 null로 BR-1 이후 확장
}
```

**대안**: `Board.assignmentGuideText` 단일 필드만 쓰고 Section 건드리지 않음 — v1은 텍스트+이미지 간단만 허용하는 경우 유력. **미결 질문 Q1에서 결정**.

### 2.4 `Submission` 재사용 (신규 필드 없음)

기존 schema.prisma 299라인 주석이 이미 `"submitted" | "reviewed" | "returned"` 예약. 본 과제에서 `reviewed` = gradingStatus=graded 시점 set, `returned` = 교사가 되돌림. 필드 추가 불필요.

### 2.5 신규/수정 엔티티 집계

- **신규 1개**: `AssignmentSlot`
- **수정 2개**: `Board`(필드 3개 + 관계 1개), `Section`(필드 1개 — 대안 채택 시 0개)
- **재사용**: `Classroom`, `Student`, `Card`, `Submission`, `BoardMember` 전부

sketch-architect escalation 기준 "필요 데이터 모델이 10개 이상 신규" 크게 하회. 기존 패턴과 정합.

---

## 3. 역할별 사용자 흐름

### 3.1 교사(owner) — 과제 보드 생성 ~ 독려

```
[대시보드] "과제 보드 만들기"
  → 1) 학급 선택 (Classroom 드롭다운, 이미 teacherId로 필터링됨)
  → 2) 제목 + 마감일 + 가이드 텍스트 입력
  → 3) "생성" 클릭
    ├─ 서버: Board(layout="assignment", classroomId) 생성
    ├─ 서버: Student WHERE classroomId 전원 조회 (N명)
    └─ 서버: N개 AssignmentSlot 트랜잭션 일괄 insert (slotNumber=student.number)
  → 4) 보드 뷰 진입: 상단 가이드 영역 + 하단 5×6 격자 (30 slot 중 N개 활성, 나머지 비활성 플레이스홀더)

[보드 뷰 · 교사]
  → 미제출 필터 버튼: `WHERE submissionStatus = "assigned"` slot만 하이라이트 (나머지 디밍)
  → "미제출 독려" 버튼: 필터된 학생 전원에게 인앱 알림 발송 (v1 인앱만, 푸시·이메일은 v2)
  → 카드 클릭: 전체화면 모달에 Submission.content + attachments 표시 + "반려(returned)" / "리뷰 완료(reviewed)" 액션

[학생 추가·제거 동기화]
  → v1: 교사가 "Roster 동기화" 버튼 수동 클릭 시 slot 재계산
  → v2: Student CRUD 트리거로 자동
```

### 3.2 학생(editor) — 내 카드 제출

```
[학급 로그인 후 보드 접속]
  → 보드 뷰 진입: 본인 slotNumber 카드에 "내 자리" 하이라이트 + 자동 스크롤
  → (본인 slot이 아닌 다른 카드는 탭해도 읽기 전용으로 열림. v1은 본인 카드만 쓰기 가능)
  → 본인 카드 탭 → 전체화면 모달 (작성 모드)
    ├─ 처음 진입 시: AssignmentSlot.viewedAt 갱신 (submissionStatus: "assigned" → "viewed")
    ├─ 텍스트 / 이미지 / 첨부 / 링크 입력 (Card 콘텐츠 포맷 재사용)
    └─ "제출" 버튼 → Card + Submission 생성·연결, submissionStatus = "submitted", submittedAt = now()
  → 제출 후: 카드 표면이 채워진 썸네일 + 제출 완료 배지로 전환
  → "수정" 재제출: 교사가 returned 로 되돌린 경우만 허용 (v1), 또는 마감 전 자유 수정 (미결 Q2)
```

### 3.3 학부모(viewer) — 본 보드 명시적 스코프

parent-viewer-roadmap.md v2 "자녀 범위 매트릭스" 편입 필요.

- **원칙**: 학부모는 **본인 자녀의 AssignmentSlot + 해당 slot의 Card/Submission만** 열람 (active link + 자녀 studentId 일치 시).
- **격리**: 타 학생 slot·card는 API 필터링으로 차단. 5×6 격자 자체를 보여주지 않고 **자녀 전용 단일 뷰**(= 내 자녀 제출물 카드 1장 + 교사 가이드 텍스트)로 축약. DOM 마스킹 불필요 (애초에 미노출).
- **교사 가이드**: owner-only 쓰기, viewer-readable — 마감·가이드 텍스트는 자녀 학부모 열람 허용.
- **matrix 뷰 제외**: MEMORY "editor·viewer·태블릿 전부 제외" 원칙 승계. 학부모 모바일 PWA에서도 슬롯 1장만.

### 3.4 태블릿(editor+학생) 별도 플로우 주의

editor 권한 자체는 데스크톱/태블릿 공통이지만, 작성 UX는 태블릿 S-Pen 기준 최적화:
- 썸네일 격자는 CSS Grid + order, 가상화 없음 (§4 참조)
- 작성 모달 진입 후에만 캔버스/첨부 업로드 UI 마운트
- 모달 닫기 = 캔버스 언마운트 (메모리 환원)

---

## 4. 태블릿 성능 체크리스트 (tablet-performance-roadmap §2·§3·§8 기준)

기준 단말 **갤럭시 탭 S6 Lite · Chrome Android · 학교 Wi-Fi 50 Mbps 공유**. 30 slot 동시 렌더 가정.

- [ ] **초기 TTI < 3s** — 보드 진입부터 5×6 격자 인터랙티브까지 (§2 예산)
- [ ] **DOM 카드 노드 ≤ 30** — virtualization 없이도 통과 가능. CSS Grid + `order: {slotNumber}` 결정적 배치
- [ ] **카드당 DOM 자식 ≤ 6** — 이름·번호·상태 배지·썸네일 1장·아이콘·클릭 hitbox만
- [ ] **썸네일 파이프라인** — 서버 리사이즈 160×120 WebP 단일 장, 원본 응답 금지 (T0-④ 재사용)
- [ ] **`loading="lazy"` + IntersectionObserver** — 뷰포트 밖 slot(주로 4~6행) 디코딩 지연
- [ ] **카드 표면 S-Pen 캔버스 금지** — 필기·드로잉은 클릭 모달 내부에서만 마운트, 닫기 시 언마운트
- [ ] **상태 토글은 순수 CSS** — `data-submission-status` 속성 + CSS 선택자로 색상 전환, React 리렌더 회피
- [ ] **WebSocket 채널 분리** — `board:${id}:assignment` 단일 채널, slot별 분리는 과도. 메시지 < 200B (slotId + status 델타)
- [ ] **모달 지연 마운트** — 카드 클릭 시에만 Submission/Card 상세 fetch + 렌더. 모달 닫기 시 즉시 언마운트
- [ ] **iframe 금지** — Canva 임베드는 v1 불허. 썸네일만(§4 iframe 폭발 위험 회피). v2에서 "라이브 보기" 버튼으로 단일 iframe 허용
- [ ] **메모리 1시간 후 < 500MB** — 기존 전역 예산 승계
- [ ] **드래그 비활성** — 격자 고정이므로 드래그 이벤트 바인딩 자체를 제거 (DraggableCard 대신 StaticSlotCard 컴포넌트 신규)
- [ ] **padlet feature phase9 QA 게이트 통과 필수** — 실측 TTI·프레임·메모리 증빙

### 예상 성능 여유

DOM 30 × 자식 6 = 180 노드 + 썸네일 지연 디코딩. 탭 S6 Lite 4GB RAM 기준 초기 레이아웃·스크롤 비용은 Breakout 섹션 뷰(§5 T0-①)와 유사 수준으로 평가, 여유 있음. 단 **미결 Q3**: 학급 N=40 이상으로 확대 시 가상화 도입 기준.

---

## 5. Canva 시너지 — `canva-assignment-pdf-merge` 접점

로컬 스킬 `canva-assignment-pdf-merge/SKILL.md`는 Canva 디자인 제목 규칙 `완료-{과제명}-{번호}-{이름}`으로 완료본을 수집·PDF 병합한다. 본 AssignmentSlot 설계와 접점:

| 접점 | 연결 방식 |
|---|---|
| **슬롯 → Canva 제목 규칙 매핑** | 학생이 Canva에서 작성 후 Content Publisher 앱으로 게시 시 자동 제목 생성 (`완료-{Board.title}-{Student.number}-{Student.name}`) → `Card.canvaDesignId`에 저장 → slot 연결 |
| **병합 PDF export** | 교사가 "완료본 병합" 버튼 클릭 → 현 보드 slot 중 `submissionStatus="submitted"` + `Card.canvaDesignId IS NOT NULL` 필터 → 기존 스킬로 PDF 병합 |
| **학생 저마찰 플로우** | Canva Apps SDK OAuth(기존 OAuthClient "canva") 체계로 학생이 Canva 내에서 바로 slot에 게시. 신규 인증 플로우 없음 (known_constraints 충족) |
| **썸네일** | Canva `/thumbnail` 프록시(T0-④ 참고)로 단일 저해상도 이미지를 slot 카드에 로드 — 성능 예산 준수 |

**v1 범위에서 Canva 연동은 선택**: Canva 제목 규칙·병합 스킬은 기존 사용 중이므로 slot에 `canvaDesignId` 파이프라인 연결만으로 시너지 확보. Canva 미사용 제출(텍스트·일반 이미지 첨부)도 정상 동작.

---

## 6. 리스크 표

| # | 리스크 | 영향 | 완화 |
|---|---|---|---|
| R1 | **Roster 변경 시 slot 싱크 불일치** — 학년 중 전학·추가 발생 시 기존 slot이 고아가 되거나 번호 충돌 | 중 (교사 혼란, 평가 누락) | v1: 교사 수동 "Roster 동기화" 버튼. 동기화 시 기존 slot 보존·새 번호 slot 추가·삭제된 학생 slot은 `status="orphaned"`로 마킹(삭제 금지, 기록 보존). v2: 자동 트리거 |
| R2 | **5×6 고정 격자가 N>30 학급에서 깨짐** | 중 (대학급 사용 불가) | v1 명시적 제한: N≤30. N=31~40은 자동 5×8로 확장(미결 Q3). N≥41은 v1 불가, 분반 가이드 |
| R3 | **Submission 재사용으로 event-signup과 status enum 충돌** | 중 (마이그레이션 혼란) | status는 `layout`별로 해석 분기. 주석 이미 이원화됨(299라인). zod 검증에서 layout별 허용 enum 체크 강제 |
| R4 | **태블릿에서 학생 30명 동시 제출 시 WebSocket 폭발** | 고 (렉·탈락) | 채널 단일(`board:${id}:assignment`) + 델타 페이로드 <200B + 100ms 디바운싱. 동시 쓰기 충돌은 slot 단위 1:1이므로 경합 없음 |
| R5 | **학부모 DOM 누출** — 격자 뷰를 viewer에 잘못 노출 시 타 학생 이름·번호 노출 | 고 (PII 유출) | viewer는 격자 자체를 렌더하지 않음. API에서 parent 토큰 시 `AssignmentSlot WHERE studentId IN (parent의 자녀)` 강제. RLS 정책 3중(API·DOM·RLS) parent-viewer §1.2 승계 |
| R6 | **재제출 정책 모호** — returned 후 수정과 마감 전 자유 수정 경계 불분명 | 저 (UX 혼란) | 미결 Q2에서 정책 확정. 기본 가설: 마감 전 자유 수정 + returned는 마감 후 특수 경로 |
| R7 | **Canva 제목 규칙 깨짐** — 학생이 수동 제목 편집 시 병합 스킬 실패 | 저 (교사 수작업) | Content Publisher 앱이 저장 시점에 제목 강제. 학생 자유 편집 허용 시 "제출" 액션이 제목 자동 교정 |
| R8 | **owner+데스크톱 전용 matrix 뷰** 요건이 v1에 흘러감 | 중 (태블릿·editor에 노출) | v1은 **matrix 뷰 자체 제외**. 미제출 필터만 제공 (격자 내 하이라이트). matrix는 v2 owner+데스크톱 전용 별도 라우트 |

---

## 7. 미결 질문 (phase3 인터뷰 대상)

총 6건 (≤7 권장 한도 내).

- **Q1. 교사 가이드 영역 모델링** — `Board.assignmentGuideText` 단일 텍스트 필드 vs `Section(role="guide")` + 일반 Card 재사용? v1 범위에서 텍스트+이미지 1장 정도면 전자로 충분. 가이드에 동영상·첨부·여러 카드가 필요하면 후자.
- **Q2. 재제출 정책** — 마감 전 학생 자유 수정 허용? returned 상태에서만 수정 허용? 두 경로 병존? 교사 입장에서 "한 번 낸 건 건드리지 마" 원칙이 강하면 returned 경로만.
- **Q3. 학급 N 상한과 격자 확장** — v1에서 N>30 학급 허용? 허용 시 5×8(N=40)까지 자동 확장 vs N≤30 하드 제한? 탭 S6 Lite 성능 여유가 있어도 UX 번호 기억의 한계.
- **Q4. 미제출 독려 채널** — v1 인앱 배지 전용 vs 학급 코드 기반 학부모 이메일 경유 알림(parent-viewer §1.3 주간 이메일에 얹기)? 학생 본인에게 닿는 채널은 인앱이 유일 (이메일 없음).
- **Q5. 학생 카드의 타 학생 카드 열람 권한** — editor 학생이 다른 학생의 제출물을 "읽기 전용"으로 볼 수 있는가? Seesaw는 비공개 기본, Classroom도 비공개. 단 "갤러리 워크" 유형 과제를 위해 owner-only 토글 필요성 검토.
- **Q6. `slotNumber`와 `Student.number` 동기화 정책** — `Student.number`가 변경되면 slot 좌표가 움직이는가(이동) vs 생성 시점 값 스냅샷(고정)? 학년 중 번호 재부여는 드물지만 예외 처리 명확화 필요.

---

## 8. phase1 요구사항 달성 확인

| phase1 핵심 결정 과제 | sketch 대응 |
|---|---|
| 신규 BoardType vs 플래그 | §1.2 ① `Board.layout="assignment"` 확장 채택 |
| Card 재사용 vs AssignmentCard 신규 | §1.2 ② `AssignmentSlot` 신규 + Card 재사용 (하이브리드) |
| Submission 재사용 판단 | §1.2 ③ 재사용, enum 주석 이미 예약됨 |
| Classroom/Roster 연결 | §1.2 ④ 기존 Classroom·Student 재사용, 신규 엔티티 0 |
| 교사 가이드 영역 | §1.2 ⑤ Section role 확장 vs Board 필드 — Q1 미결 |
| 태블릿 성능 | §4 체크리스트 13항목 + §2 예산 준수 |
| 권한 (editor 자기 카드만) | §3.2 본인 slot 쓰기, §3.3 viewer 자녀 slot만 |

---

## 9. 다음 단계 (phase3 interview 진입 조건)

- 검증 게이트: 데이터 모델 초안 ✅ / 사용자 흐름 ≥ 3 역할 ✅ / 미결 질문 ≥ 1 ✅ (6건)
- 인터뷰 주제: 위 Q1~Q6 + (사용자 확인 필요시) §1.2 ①~⑧ 초벌 결정 재검토
- 에이전트 자율 답변 가능: Q1·Q2·Q3·Q5·Q6 (기존 primitive·레퍼런스·태블릿 예산에서 추론 가능)
- 사용자 결정 권장: Q4 (알림 채널·학부모 연계 정책)
