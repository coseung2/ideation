# Phase 2 — Breakout Room 보드 스케치

> task_id: `2026-04-12-breakout-room-board`
> 작성일: 2026-04-12
> 전제: phase1 1순위 권고(하이브리드 자체 구현) 고정
> 참조: `phase1/exploration.md`, `plans/tablet-performance-roadmap.md` §5 T0-①, `plans/event-signup-roadmap.md`, `plans/drawing-board-library-roadmap.md`, `padlet/prisma/schema.prisma`

---

## 0. 전제 결정 (phase1 권고 고정)

- **UX 모델**: Padlet Breakout rooms (섹션 = 모둠, 모둠별 독립 링크)
- **구현 모델**: Miro Breakout frames 사상 — 템플릿 저장 시 모둠 섹션 구조까지 직렬화
- **탐색 UX**: FigJam 교육 템플릿 갤러리 (카테고리·미리보기·"이 템플릿으로 시작")
- **배포 UX**: Zoom Self-Select 토글 + QR·accessToken 링크 일괄 발급
- **성능 축**: tablet-performance-roadmap T0-① 섹션 격리 뷰 승계 (섹션별 WS 채널, 섹션 스코프 쿼리, iframe LRU 3개)
- **스코프**: 자체 구현. 외부 SaaS 의존 없음. GPL 격리 불필요(순수 자체 코드).

---

## 1. 데이터 모델 초안 (Prisma)

> 설계 원칙: 기존 `Board`·`Section`·`Card`·`Classroom`·`Student`·`BoardMember` 최대 재사용. 신규 엔티티는 "템플릿 카탈로그"와 "모둠별 토큰"만 추가.

### 1-A. `Board` 확장 — `layout = "breakout"` 신설

```prisma
model Board {
  // ... 기존 필드 유지 ...
  layout      String @default("freeform")
    // freeform | grid | stream | columns | assignment | quiz | plant-roadmap | drawing | event-signup | "breakout" ⭐신규

  // Breakout 메타 (layout == "breakout"일 때만 의미)
  breakoutTemplateId  String?   // 개설 시 선택한 템플릿 스냅샷 원본
  breakoutGroupCount  Int?      // 모둠 수 (생성 시점 캡처, 이후 가감 가능)
  breakoutMode        String?   // "teacher-assign" | "self-select" | "link-fixed"
                                // Zoom 3모드 ↔ Aura 매핑
  breakoutVisibility  String?   // "own-only" | "peek-others" | "teacher-tour"
                                // 학생이 타 모둠을 볼 수 있는 범위
  breakoutTextCode    String?   @unique
                                // Nearpod 5자 코드 (모둠 선택 랜딩 진입용, 선택 기능)

  breakoutTemplate    BreakoutTemplate? @relation(fields: [breakoutTemplateId], references: [id], onDelete: SetNull)

  @@index([breakoutTemplateId])
}
```

### 1-B. `Section` 재사용 + Breakout 메타 덧붙임

> 결정 초안: **Section 재활용**. 별도 `BreakoutGroup` 모델 신설은 **하지 않음**(미결 Q2로 최종 확인).
> 근거:
> - T0-① 섹션 격리 뷰가 이미 `Section.accessToken`을 "Breakout view teacher-rotatable access token"으로 정의해 둠 (schema.prisma line 129~133 주석).
> - 섹션별 WS 채널 (`board:${id}:section:${id}`)·섹션별 rbac (`viewSection`)·섹션별 라우트 (`/b/:slug/s/:sectionId`)가 모두 이미 Section을 단위로 상정.
> - 새 엔티티를 만들면 `BreakoutGroup.sectionId` 1:1 중복만 남고 쿼리 경로가 두 배 길어짐.
> - "모둠은 UX 용어, Section은 스키마 용어" — 라벨만 i18n 계층에서 "모둠 N"으로 표기하면 충분.

```prisma
model Section {
  id          String  @id @default(cuid())
  boardId     String
  title       String          // "1모둠", "찬성팀" 등 (템플릿에서 자동 설정, 교사 수정 가능)
  order       Int     @default(0)

  // 기존 (T0-①에서 이미 추가됨):
  accessToken String? @unique  // 모둠별 공유 링크 해시 — 교사 재발급 가능

  // ⭐ Breakout 전용 메타 (신규 컬럼)
  role        String?  // "group" | "teacher-pool" | "showcase" | "staging"
                       // 템플릿 내부 섹션 역할 태그 (예: Jigsaw 템플릿은 "expert"·"home" 두 역할)
  capacity    Int?     // 모둠 정원(soft limit, self-select 모드에서 사용)
  lockedUntil DateTime? // 템플릿 지정 단계 전까지 잠금 (예: "발표 섹션"은 토의 이후 해금)
  color       String?  // 모둠별 색상 (학생 구분 시각화)

  board Board  @relation(fields: [boardId], references: [id], onDelete: Cascade)
  cards Card[] @relation("SectionCards")
  assignments BreakoutAssignment[]   // ⭐ 신규 역참조

  @@index([boardId])
}
```

### 1-C. 신규 엔티티 ① `BreakoutTemplate` — 템플릿 카탈로그

```prisma
// 교사 공통(시스템 시드) + 학교 공용 + 교사 개인 모두 한 테이블
model BreakoutTemplate {
  id            String   @id @default(cuid())
  ownerId       String?                     // null이면 시스템 시드(공통 템플릿)
  scope         String   @default("private") // "system" | "school" | "private"
                                              // system = 모든 교사 사용 가능, school = 해당 학교만, private = 소유 교사만
  schoolId      String?                     // scope=="school"일 때
  slug          String                      // "kwl", "pros-cons", "jigsaw" 등 i18n 키
  title         String                      // "KWL 차트"
  subtitle      String?                     // "아는 것 / 배우고 싶은 것 / 배운 것"
  description   String   @db.Text
  thumbnailUrl  String?                     // 갤러리 미리보기 (Canva 썸네일 프록시 경유)
  category      String                      // "discussion" | "brainstorm" | "organize" | "present" | "icebreaker" | "debate"
  gradeBand     String?                     // "elementary" | "middle" | "high" | "all"
  estimatedMinutes Int?                     // 수업 진행 예상 시간
  requiresPro   Boolean  @default(false)    // Free tier 제한
  defaultGroupCount Int  @default(4)        // 기본 모둠 수 (교사 override 가능)

  // ⭐ 구조 정의 (모둠 당 섹션 구조를 JSON으로 직렬화)
  // {
  //   sectionsPerGroup: [
  //     { role: "group", titleTemplate: "{n}모둠", cards: [...] },
  //     { role: "showcase", titleTemplate: "{n}모둠 발표", lockedUntil: "stage2" }
  //   ],
  //   sharedSections: [  // 모둠 공통(전체 학생 공유) 섹션
  //     { role: "teacher-pool", title: "자료실", cards: [{ type:"link", url:"..." }] }
  //   ]
  // }
  structure     String   @db.Text            // JSON

  isPublished   Boolean  @default(true)
  version       Int      @default(1)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  owner         User?    @relation(fields: [ownerId], references: [id], onDelete: SetNull)
  boards        Board[]                      // 이 템플릿에서 파생된 Breakout 보드들

  @@index([ownerId])
  @@index([scope, category])
  @@index([schoolId])
}
```

> 구조 JSON은 drawing-board-library-roadmap의 "ORA 템플릿" 사상(한 템플릿 안에 여러 배포 단위) + event-signup-roadmap의 `customQuestions JSON` 저장 패턴 조합.

### 1-D. 신규 엔티티 ② `BreakoutAssignment` — 학생↔모둠 배정

> self-select 모드에서 학생이 모둠을 고른 이력, teacher-assign 모드에서 교사가 미리 배정한 결과 저장.
> 링크 고정(link-fixed) 모드에서는 레코드 없이 accessToken으로 진입 → 학생 세션이 그 섹션을 본다.

```prisma
model BreakoutAssignment {
  id           String   @id @default(cuid())
  sectionId    String                         // 배정된 모둠 = Section
  studentId    String?                        // Student 계정 (학급 내부)
  userId       String?                        // 또는 User (교사 참관·보조)
  role         String   @default("member")    // "member" | "leader" | "observer"
  assignedAt   DateTime @default(now())
  assignedBy   String?                        // 교사 userId (self-select이면 null)

  section      Section  @relation(fields: [sectionId], references: [id], onDelete: Cascade)
  student      Student? @relation(fields: [studentId], references: [id], onDelete: Cascade)

  @@unique([sectionId, studentId])
  @@index([sectionId])
  @@index([studentId])
}
```

> 학생 이동 허용(미결 Q5) 여부에 따라 `@@unique([sectionId, studentId])` 유지 vs `@@index`만 남기는 선택.
> 당분간은 **학생 1명 = 한 모둠(unique)** 초안, 교사가 드래그로 재배정 시 UPDATE.

### 1-E. 기존 엔티티 영향 없음

- `Classroom`, `Student`, `User`, `BoardMember`, `Card` **컬럼 추가 없음**.
- QR·textCode 인증 인프라(`Student.qrToken`/`textCode`) 그대로 재사용 → 학생은 이미 학급 세션이 있으므로 Breakout 진입 시 신규 로그인 없음.
- `Submission`은 Breakout에서 사용 안 함(자유 보드형 — 카드로 기여).

### 1-F. 엔티티 카운트

| 종류 | 개수 |
|---|---|
| 신규 엔티티 | 2 (`BreakoutTemplate`, `BreakoutAssignment`) |
| 수정 엔티티 | 2 (`Board` +5 컬럼, `Section` +4 컬럼) |
| 재사용(변경 없음) | 5 (`Student`·`Classroom`·`User`·`Card`·`BoardMember`) |

에스컬레이션 임계치(10개 이상 신규) 미달 → sketch 범위 내.

---

## 2. 사용자 흐름

### 2-A. 교사 흐름 — "3분 이내 개설·배포"

```
1. [대시보드] "새 보드" → 레이아웃 카드에서 "브레이크아웃(모둠 활동)" 선택
2. [템플릿 갤러리 모달]
   - 카테고리 탭: 토의 / 브레인스토밍 / 정리 / 발표 / 아이스브레이커 / 찬반
   - 공통 템플릿 우선 노출, "내 템플릿"·"학교 템플릿" 탭 분리
   - 각 카드: 썸네일 + 제목 + 예상 시간 + Free/Pro 뱃지
   - "이 템플릿으로 시작" (FigJam 패턴)
   - 옵션: "빈 Breakout으로 시작"
3. [모둠 설정]
   - 모둠 수: 슬라이더 2–10 (기본값 = 템플릿 defaultGroupCount)
   - 모둠명 규칙: "1모둠, 2모둠…" / "A, B, C…" / "주제별(자유 입력)"
   - 배포 모드 선택:
     ① 링크 고정(link-fixed) — 모둠 링크를 교사가 나눠줌 (기본값)
     ② 학생 자율 선택(self-select) — 공통 랜딩에서 학생이 모둠 선택
     ③ 교사 배정(teacher-assign) — 학생 목록 드래그 배정
   - 상호 열람 범위: "자기 모둠만" / "다른 모둠 구경 가능" / "교사만 전체 순회"
4. [생성 트리거]
   서버 트랜잭션:
     - Board(layout="breakout") 1건
     - Section N건 (템플릿 structure에 따라 모둠당 1개 또는 복수)
     - 공유 섹션(teacher-pool 등) 별도 1건
     - 모둠 섹션마다 accessToken 자동 생성
     - 템플릿 structure.cards[] 를 각 섹션에 **복사** (참조 아님 — 모둠별 편집 독립성 보장)
5. [배포 화면]
   - 모둠별 카드 그리드, 각 카드에:
     · 섹션 진입 링크 `/b/:slug/s/:sectionId?t=:accessToken`
     · QR PNG 다운로드
     · "링크 복사"
     · "토큰 재발급"(예전 QR 무효)
   - 상단 액션: "전체 링크 ZIP 다운로드", "한 장 PDF 생성(모둠별 QR 4×8)"
   - self-select 모드면 상단에 "공통 진입 링크 + 5자 코드" 1개 표시
6. [수업 중 교사 뷰]
   - `/b/:slug` = 통합 뷰: 섹션 목록 + 섹션별 카드 수·최근 활동 요약만 렌더 (성능 예산 보호)
   - 섹션 카드 탭 → 해당 섹션 상세 (학생과 동일한 뷰 + 편집 권한)
   - 라이브 모드 토글: iframe·실시간 커서 활성화 (기본 OFF)
```

### 2-B. 학생 흐름 — "한 번의 탭으로 모둠 진입"

**Case A: link-fixed 모드 (교사가 모둠 링크 배포)**
```
1. 교사가 QR/링크/5자코드를 칠판·프린트로 공유
2. 학생: 태블릿 홈 → 학급 페이지 → QR 스캔 또는 코드 입력
   (기존 Student.qrToken/textCode 세션 승계, 새 로그인 없음)
3. `/b/:slug/s/:sectionId?t=:accessToken` 직접 진입
   → 서버가 accessToken 검증 → 해당 섹션 카드만 SSR
4. 자기 모둠 섹션만 로드 (다른 섹션 WS 이벤트 수신 0건)
5. 카드 작성·이미지 첨부·Canva 카드 삽입(iframe LRU 3)
6. 상단 탭 "다른 모둠 보기"는 breakoutVisibility에 따라 표시/숨김
```

**Case B: self-select 모드**
```
1. 학생이 공통 랜딩 진입 `/b/:slug`
2. 모둠 카드 그리드 표시 (각 모둠 현재 인원·정원·짧은 설명)
3. "여기 참여" 탭 → BreakoutAssignment 생성 → 해당 섹션으로 리다이렉트
4. (재진입 시) 이전 배정 기억 → 자동으로 본인 모둠 섹션으로
```

**Case C: teacher-assign 모드**
```
1. 교사가 사전에 학생 드래그 배정 (BreakoutAssignment 미리 생성)
2. 학생이 공통 링크 진입 → 서버가 assignment 조회 → 본인 섹션으로 리다이렉트
3. 오배정 신고 버튼 (교사에게 알림)
```

(학부모 흐름은 Breakout의 범위 밖 — 모둠 활동은 실시간 수업 내부 도구. 필요 시 교사 통합 뷰를 viewer 권한으로 열람 가능하나 이 스케치에서는 기본 미지원.)

---

## 3. 태블릿 성능 체크리스트 (갤탭 S6 Lite × 30대)

> 전제: tablet-performance-roadmap §2 성능 예산 전부 승계. 아래는 Breakout 고유 추가 체크.

| # | 항목 | 수용 기준 | 근거 |
|---|---|---|---|
| P1 | 섹션 격리 로드 | `/s/:sectionId` 진입 시 **해당 섹션 카드만** 서버에서 수신 (Network 탭 검증) | T0-① |
| P2 | 초기 TTI (모둠 섹션 50카드) | < 3s | roadmap §2 |
| P3 | 섹션 전환 시 이전 DOM·iframe 완전 언마운트 | 메모리 프로파일 후 직전 섹션 누수 < 5MB | T0-① |
| P4 | WS 채널 격리 | 다른 모둠 카드 변경 이벤트 수신 0건 (DevTools WS 프레임) | T0-① |
| P5 | iframe 동시 마운트 | Breakout 섹션 내에서도 LRU 3개 상한 준수 | T0-② |
| P6 | 교사 통합 뷰 | 섹션 N개여도 카드 본문 페치 금지 — 섹션별 "카드 수·최근 활동 요약"만 | neu |
| P7 | 라이브 모드 토글 | 기본 OFF. 토글 ON 시에만 실시간 커서·iframe 활성 | T0-② |
| P8 | 템플릿 복사 트랜잭션 | 모둠 10개 × 섹션 2개 × 카드 5개 = 100건 INSERT, 서버 < 800ms | neu |
| P9 | QR 일괄 생성 | 클라 측 생성, 10모둠 PDF 조합 < 2s (Web Worker 활용) | neu |
| P10 | WS 메시지 크기 | 카드 이동 이벤트 < 200B, 100ms 디바운싱 | T0-③ |
| P11 | 템플릿 썸네일 | Canva 이미지 프록시 경유, < 100KB/장, `loading=lazy` | T0-④ |
| P12 | self-select 랜딩 | 공통 랜딩은 섹션 카드 없이 모둠 메타만 표시 (인원·정원) — 전체 Card 페치 금지 | T0-① |
| P13 | 모둠 수 상한 | UI에서 10개 초과 불가 (성능 예산·교실 현실) | neu |
| P14 | 오프라인 후퇴 | 섹션별 SWR + idb-keyval 캐시, 학교 Wi-Fi 불안정 대비 | roadmap §3 |

**미달 시 기능 배포 금지** — padlet feature 파이프라인 phase9 QA 매트릭스에 위 14항 자동 삽입.

---

## 4. Canva·tier·기존 기능 시너지

| 항목 | 시너지 |
|---|---|
| **템플릿 썸네일** | 교사 템플릿 썸네일을 Canva Autofill(P1-③)로 생성 — 교사가 "KWL 차트" 슬라이드를 Canva에서 디자인 후 썸네일 자동 추출 |
| **Canva oEmbed 카드 (P0-①)** | 템플릿 structure.cards[]에 Canva 자료 URL 포함 가능 (예: "찬반 토론 템플릿"에 교사가 준비한 Canva 자료 카드 2장) — iframe LRU 3으로 보호 |
| **Free tier** | `BreakoutTemplate.requiresPro=true` 플래그로 일부 고급 템플릿(예: Jigsaw 다단계)을 Pro 한정. 공통 기본 템플릿 5개는 Free 무제한 |
| **Free tier 보드 수** | 기존 Free 1반 제약 하에서 Breakout 보드는 일반 Board로 카운트됨. 별도 제한 없음 |
| **그림보드 (drawing layout)** | Breakout 섹션 중 하나를 "그림 모둠 작품"용으로 쓸 때 `StudentAsset` 라이브러리 재사용 — 각 모둠 섹션에서 `AssetAttachment`로 그림 첨부 |
| **Section 재사용 일관성** | 기존 freeform·columns 레이아웃의 Section 관리 UI 그대로 활용 — 교사 학습 곡선 최소 |
| **accessToken 패턴** | event-signup의 `Board.accessToken` 패턴과 동일 메커니즘을 `Section` 단위로 복제 — 재사용·감사·재발급 UX 통일 |
| **Nearpod 5자 코드** | `Board.breakoutTextCode`로 학생 진입 마찰 감소 (QR 스캔 실패 대비), 기존 `Quiz.roomCode` 구현 패턴 재사용 |
| **Zoom Self-Select** | breakoutMode 토글로 "링크 고정 vs 학생 자율 vs 교사 배정" 3모드 한 UI — 학년·상황별 전환 |

---

## 5. 리스크 표

| # | 리스크 | 영향 | 완화 |
|---|---|---|---|
| R1 | 템플릿 복사 트랜잭션 폭주 (10모둠 × 다단계 템플릿) | 개설 응답 3s 초과 → "3분 이내 개설" 목표 실패 | 모둠 수 상한 10, 카드 상한 템플릿당 50, 서버는 `$transaction` batch + 백그라운드 QR 생성 |
| R2 | 모둠 간 이벤트 크로스토크 | 프라이버시·성능 양측 위반 | WS 토픽 `board:${bid}:section:${sid}` 강제. 서버에서 subscription 시 sectionId 소유 검증 (rbac viewSection) |
| R3 | accessToken 유출 (학생 간 공유) | 다른 모둠 무단 열람 | 링크 토큰 + 학생 세션 이중 체크. breakoutVisibility="own-only"일 땐 assignment 매칭 강제 |
| R4 | self-select 모드에서 인기 모둠 몰림 | 일부 모둠 정원 초과, 다른 모둠 빈 상태 | `Section.capacity` soft limit + UI에 실시간 인원 표시 + 정원 초과 시 "대기" 배지 |
| R5 | Section 재활용으로 인한 UX 혼란 | "모둠"과 "섹션"이 같은 것인지 교사가 혼동 | i18n: layout="breakout"일 땐 UI에서 "모둠"으로 일관 표기. 관리자 뷰(교사 설정)에서만 "섹션" 용어 병기 |
| R6 | 학생 이동 허용 시 카드 저작권 꼬임 | 학생 A가 1모둠에서 쓴 카드가 2모둠으로 따라감? | v1은 **학생이 모둠을 옮겨도 카드는 원래 섹션에 남음** (Card.sectionId 고정). 이동 시 새 섹션에서 처음부터 기여 |
| R7 | 교사 통합 뷰 과부하 | 통합 뷰가 모든 섹션 카드 받으면 성능 예산 위반 | 통합 뷰는 "섹션 요약 카드"만 렌더. 상세는 섹션 진입 시 lazy 로드 (P6 체크리스트) |
| R8 | 템플릿 structure JSON 스키마 폭주 | 템플릿마다 필드 다르면 파싱·버전 관리 지옥 | Zod 스키마로 structure 검증. `BreakoutTemplate.version` 증가 시 마이그레이션 스크립트 강제 |
| R9 | Canva iframe 카드를 템플릿에 과다 포함 | 학생 섹션 로드 시 iframe 폭발 | 템플릿 검증 시 iframe 카드 수 ≤ 3 강제. 초과 시 "라이브 모드 전환 후 순차 재생" 안내 |
| R10 | Free tier 악용 (시스템 템플릿 남용으로 대량 보드 생성) | DB·스토리지 남용 | 기존 Free 1반 제약 + 월 보드 개설 수 상한(이미 존재하면 승계, 없으면 tier 문서에 추가 요청) |

---

## 6. 미결 질문 (phase3 인터뷰 재료)

> 에이전트 즉결 불가 — 사용자(교사 당사자) 결정이 질적 차이를 만드는 항목만 남김.

| # | 질문 | 왜 미결인가 | sketch 초안 기본값 |
|---|---|---|---|
| Q1 | 기본 제공(시스템) 템플릿 종류는? 예시 후보: KWL 차트 / 브레인스토밍 / 찬반 토론 / 모둠 발표 준비 / Jigsaw / 아이스브레이커 / 피라미드 토의 / 갤러리 워크 | 교사 수업 시나리오 의존, 초등·중등·고등 학년대별 수요 차이 큼 | v1에 5개: KWL·브레인스토밍·찬반 토론·모둠 발표·아이스브레이커 |
| Q2 | Section 재활용 vs `BreakoutGroup` 신규 엔티티 | 현재 초안은 Section 재활용(1-B 근거). 하지만 Jigsaw처럼 "한 학생이 두 모둠 소속" 시나리오가 확정되면 별도 엔티티가 깔끔 | Section 재활용 유지 |
| Q3 | 모둠 인원·개수 기본값 | 학급 크기(25–35명)에 따라 4–5명 × 6–8모둠이 표준이나 고정값 필요 | 기본 모둠 수 = 4, 모둠당 정원 = 6 (soft) |
| Q4 | 모둠 간 상호 열람 기본값 | 교육적으로 "갤러리 워크"는 허용, "찬반 토론"은 금지 — 템플릿별로 다름 | 템플릿별 추천값 + 교사 override. 기본은 "자기 모둠만" |
| Q5 | 학생이 모둠 이동 가능 여부 | self-select 모드에서만 허용? teacher-assign 모드에선 신청만 받을지? | v1은 self-select 모드에서만 1회 이동 허용, teacher-assign은 이동 불가 |
| Q6 | 템플릿 구조 복제 방식: 복사 vs 참조 | drawing-board-library-roadmap의 "AssetAttachment는 복사"와 같은 결 — 독립성 vs 일괄 수정 | **복사 기본**. 학생 모둠의 카드는 모둠별 독립(모둠끼리 서로 영향 없음). 교사 공통 자료실 섹션(`teacher-pool`)만 예외로 전 모둠 공유(이건 단일 섹션·보드 전체 공유라 자연스럽게 공유) |
| Q7 | Free tier에서 템플릿 몇 개까지? 시스템 템플릿은 Free? Pro 고급만 유료? | 수익 모델과 직결 | 시스템 템플릿 5개 Free 무제한, 교사 개인 템플릿 저장은 Free 3개까지·Pro 무제한, `requiresPro=true` 고급 템플릿은 Pro 전용 |

---

## 7. 검증 게이트 자가 점검

| 게이트 | 상태 |
|---|---|
| 데이터 모델 초안 | ✅ 신규 2개(`BreakoutTemplate`, `BreakoutAssignment`) + 수정 2개(`Board`, `Section`) |
| 사용자 흐름 ≥ 1 역할 | ✅ 교사(§2-A) + 학생 3모드(§2-B) |
| 태블릿 성능 체크리스트 | ✅ 14항 (P1–P14) |
| 미결 질문 ≥ 1 | ✅ 7개 (Q1–Q7) |
| 미결 질문 ≤ 10 (스코프 적절) | ✅ |
| 신규 엔티티 < 10 (에스컬레이션 임계 미달) | ✅ |

---

## 8. 다음 phase 입력 노트 (phase3 interview-facilitator용)

- 인터뷰 핵심 타깃: Q1 (템플릿 세트) + Q3 (인원·개수 기본값) + Q7 (tier 연결) — 제품 수익·교사 UX에 직결
- Q2, Q5, Q6은 에이전트가 phase3에서 배경 지식으로 자답 가능 (이미 초안에 방향성 제시)
- ambiguity ≤ 0.2 목표 달성 가능 — 핵심 모델·흐름·성능이 고정됐고, 미결 7개는 모두 파라미터 레벨
