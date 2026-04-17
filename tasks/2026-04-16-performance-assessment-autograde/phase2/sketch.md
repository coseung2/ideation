# Sketch — 수행평가 자동채점 파이프라인 (Aura-board → Aura 웹앱)

> task_id: `2026-04-16-performance-assessment-autograde`
> 작성: 2026-04-16 (phase2 / sketch-architect)
> 입력: phase0 request.json + phase1 exploration.md (축 A~F)
> 전제: phase1 권고 **1안 "웹 네이티브 + LLM Vision + 공통 DB + 다층 방어"** 고정. 2안은 D축 분리 시 폴백.
> 기준 단말: 갤럭시 탭 S6 Lite (Snapdragon 720G, 4GB RAM, Chrome Android, S-Pen)

---

## 1. 전제 결정 (phase1 권고 반영)

| 축 | 1순위 | 근거 1줄 |
|---|---|---|
| **A** 객관식 엔진 | **SurveyJS Form Library (MIT)** | MIT·React 네이티브 번들·iframe 0·한국어 OK, tablet-performance §2a "iframe v1 금지" 부합 |
| **B** 입력·OCR | **tldraw/perfect-freehand 캔버스 + Gemini 2.5 Flash Vision 통합 OCR** + 키보드 토글 | S-Pen 활용, OCR+채점 1 round-trip, 네트워크 단절 시 IndexedDB 큐잉 |
| **C** 서술형 채점 | **Gemini 2.5 Flash** + Sentence-Transformers 전처리 + Promptfoo 회귀 + 프로바이더 추상화 | 1M 컨텍스트 학급 일괄·$0.15/$0.60, OpenAI·Claude 교체 가능 |
| **D** 성적 송신 | **공통 Supabase + RLS + Realtime Postgres Changes** | 두 앱 같은 Supabase 전제, ms 레벨 실시간, parent-viewer v2 RLS 재사용 |
| **E** 성적부 UI | **Classroom 표 구조 + Canvas 루브릭 popover + 하이클래스 학부모 뷰**, matrix는 owner+데스크톱 전용 | 국내 교사·학부모 UX 기대치 정렬 + 태블릿 격리 방침(MEMORY matrix_desktop_only) |
| **F** 부정행위 방지 | **L1(Fullscreen/Visibility API) + L2(Supabase Realtime 교사 대시보드) + L3(문항풀 랜덤·셔플·시간제한) + L4(Knox Kiosk 옵션 토글)** | BYOD+갤탭+교사 현장 감독 환경. 카메라(F7)·키스트로크(F9) 파킹 |

**즉결 결정 (sketch 단계에서 자율 확정, phase3 인터뷰 대상 아님)**:
- `Board.layout` 값 이름: **`"assessment"`** (assignment과 병렬 어휘).
- 신규 Prisma 엔티티 접두사: **`Assessment*`** (교사가 "수행평가"라 부르는 현장 어휘).
- 확정 2단계(confirmedAt + releasedAt) Moodle 스타일 채택. 학부모는 `releasedAt` 이후 열람.
- 프로바이더 추상화 레이어 경로: `src/lib/grading/providers/{gemini,openai,claude}.ts` (축 C에서 못박음).
- AI 제안은 항상 "제안" 상태로만 생성, 교사 확정 없이 학부모·학생에게 노출 금지 (known_constraint 준수).
- L1 이벤트 유형 enum은 고정: `visibility_hidden | focus_lost | fullscreen_exit | paste_blocked | copy_blocked | contextmenu_blocked | window_open_blocked | idle_over_threshold` (8종).

---

## 2. 데이터 모델 초안 (Prisma)

> 기존 Board·Card·Section·Classroom·Student·AssignmentSlot·Parent·ParentChildLink 재사용.
> 신규 7개 모델 + `Board.layout="assessment"` 확장 + 1개 설정 플래그.
> 열·관계·enum은 방향 제시이며 정식 마이그레이션 스펙 아님.

### 2.1 기존 모델 확장

```prisma
// Board — layout 열거 확장. 기존 assignment과 병렬.
// layout: "freeform" | "grid" | "stream" | "columns" | "assignment" | "assessment" | "quiz" | ...
model Board {
  // ... 기존 필드 유지 ...

  // ── Assessment layout (AS-1) ────────────────────────────
  assessmentTemplateId String?              @unique
  assessmentTemplate   AssessmentTemplate?  @relation(fields: [assessmentTemplateId], references: [id])
  assessments          AssessmentSubmission[] // 역관계 (학생별 응시 인스턴스)
}

// Classroom — L4 Knox 토글 플래그 (phase1 축 F 결정)
model Classroom {
  // ... 기존 필드 ...
  schoolManagedDevices Boolean @default(false) // L4: Knox Kiosk 토글 (school 단말 배포 시만 on)
  gradebookReleasePolicy String @default("teacher_manual") // "teacher_manual" | "on_confirm" — 학부모 공개 시점 정책
}

// AssignmentSlot — 기존 수거 레이어 재사용.
// 수행평가는 AssignmentSlot을 "응시 슬롯(seat)"으로 사용하지 않고, 별도 AssessmentSubmission으로 관리.
// 단, "수행평가 과제 게시물을 수거함으로 합치고 싶다" 옵션을 위해 FK로 느슨 연결.
model AssignmentSlot {
  // ... 기존 필드 ...
  assessmentSubmissionId String?                @unique
  assessmentSubmission   AssessmentSubmission?  @relation(fields: [assessmentSubmissionId], references: [id], onDelete: SetNull)
}
```

### 2.2 신규: AssessmentTemplate (출제 메타)

```prisma
model AssessmentTemplate {
  id            String   @id @default(cuid())
  boardId       String   @unique // 1 Board = 1 Assessment (v1 제약)
  title         String
  description   String   @default("")
  subject       String?  // "국어"|"수학"|"사회"|"과학"|"영어"|기타
  totalPoints   Int      @default(100)
  // 출제·응시·채점 라이프사이클
  status        String   @default("draft") // "draft" | "published" | "closed"
  publishedAt   DateTime?
  closedAt      DateTime?
  // 응시 제어 (축 F L3)
  durationMin   Int?     // 시간 제한 (null = 무제한)
  randomizePool Boolean  @default(false) // 문항풀에서 학생별 랜덤 N개
  randomizeCount Int?    // randomizePool=true일 때 학생당 추출 수
  shuffleChoices Boolean @default(true) // 보기 셔플
  shuffleOrder  Boolean  @default(false) // 문항 순서 셔플
  // 응시 환경 (축 F L1·L4)
  enforceFullscreen Boolean @default(true)
  blockCopy     Boolean  @default(true)
  blockPaste    Boolean  @default(true)
  idleThresholdSec Int   @default(120) // 무활동 간주
  kioskMode     Boolean  @default(false) // L4 Knox Kiosk (Classroom.schoolManagedDevices=true일 때만 적용)
  // 루브릭 원본 (서술형 문항에서 참조)
  rubricText    String   @default("") // 교사가 쓰는 자연어 루브릭, LLM 프롬프트에 주입
  // 채점 LLM 프로바이더 선택
  gradingProvider String @default("gemini-flash") // "gemini-flash"|"gpt-5-mini"|"claude-haiku-4.5"
  // Aura 웹앱 성적부 연동
  gradebookCategory String? // "수행평가"|"지필평가"|"형성평가" 등 교사 자유 텍스트
  weight         Float    @default(1.0) // 가중치 (E2 Canvas 참조)

  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  createdById   String

  board        Board                  @relation(fields: [boardId], references: [id], onDelete: Cascade)
  createdBy    User                   @relation(fields: [createdById], references: [id])
  questions    AssessmentQuestion[]
  submissions  AssessmentSubmission[]

  @@index([boardId])
  @@index([status, publishedAt])
}
```

### 2.3 신규: AssessmentQuestion (문항)

```prisma
model AssessmentQuestion {
  id            String   @id @default(cuid())
  templateId    String
  order         Int      @default(0)
  // 문항 유형
  kind          AssessmentQuestionKind // enum 아래 참조
  stem          String   // 문항 지문 (마크다운 허용)
  imageUrl      String?  // 보조 이미지
  points        Int      @default(1)

  // kind별 payload (JSON). 클라이언트엔 sanitized(정답 제거)본만 전송.
  //  MCQ:     { choices: [{id,label,text}], correctChoiceIds: [string], allowMultiple: boolean }
  //  OX:      { correctAnswer: "O"|"X" }
  //  NUMERIC: { expected: number, tolerance: number, unit?: string }
  //  SHORT:   { modelAnswers: string[], keywords: string[], partialCredit: boolean }
  //  ESSAY:   { rubricOverride?: string, maxChars?: int, minChars?: int }
  payload       Json
  // 루브릭 (서술형·논술형에서 추가). template.rubricText를 베이스로 문항별 override.
  rubric        String?

  // 문항 생성 AI 보조 메타 (phase1 미결, 일단 보관만)
  aiGeneratedBy String? // null | "gemini-flash" | "gpt-5-mini"
  aiSourcePrompt String?

  template      AssessmentTemplate  @relation(fields: [templateId], references: [id], onDelete: Cascade)
  answers       AssessmentAnswer[]

  @@index([templateId])
}

enum AssessmentQuestionKind {
  MCQ       // 객관식 선다 (단일·복수 선택 payload로 구분)
  OX        // T/F
  NUMERIC   // 주관식 숫자 (허용 오차)
  SHORT     // 단답·서술 (키워드 + 모범답안)
  ESSAY     // 논술·장문 서술 (루브릭 기반 LLM 채점)
}
```

### 2.4 신규: AssessmentSubmission (학생 응시 인스턴스)

```prisma
model AssessmentSubmission {
  id            String   @id @default(cuid())
  templateId    String
  studentId     String
  // 상태머신: in_progress → submitted → grading → reviewed → confirmed → released
  status        AssessmentSubmissionStatus @default(in_progress)

  // 시간
  startedAt     DateTime @default(now())
  submittedAt   DateTime?
  // 자동채점(AI 제안) 완료 시각
  gradedAt      DateTime?
  // 교사 확정·릴리스 (E축 이원화)
  confirmedAt   DateTime?
  confirmedById String?
  releasedAt    DateTime?
  releasedById  String?

  // 점수 (확정 후 집계)
  totalScore    Float?
  suggestedScore Float?   // AI 제안 합산 (교사가 오버라이드 가능)
  suggestedFeedback String? // LLM 전체 총평

  // 축 F L1 부정행위 이벤트 로그 (분리 엔티티 ProctorEvent로 정규화. 요약만 캐시)
  proctorEventCount Int   @default(0)
  proctorFlag   String?   // null | "warning" | "critical" — 3회 이상 이탈 시 세팅

  // 네트워크 단절 대응 (리스크)
  localDraftHash String? // IndexedDB 동기화용 해시

  // 응시 환경 캡처 (감사 로그)
  userAgent     String?
  classroomIpHash String?

  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  template      AssessmentTemplate @relation(fields: [templateId], references: [id], onDelete: Cascade)
  student       Student            @relation(fields: [studentId], references: [id], onDelete: Restrict)
  confirmedBy   User?              @relation("AssessmentConfirmedBy", fields: [confirmedById], references: [id])
  releasedBy    User?              @relation("AssessmentReleasedBy",  fields: [releasedById],  references: [id])
  answers       AssessmentAnswer[]
  proctorEvents ProctorEvent[]
  gradebookEntry GradebookEntry?   // 확정 후 1:1 복제 레코드
  board         Board              @relation(fields: [boardId], references: [id], onDelete: Cascade)
  boardId       String
  assignmentSlot AssignmentSlot?   // 역관계 (선택적으로 기존 수거 레이어와 연동)

  @@unique([templateId, studentId]) // 학생당 1회 응시 (재시험은 별도 템플릿·별도 레코드)
  @@index([studentId])
  @@index([status, updatedAt])
}

enum AssessmentSubmissionStatus {
  in_progress  // 학생 응시 중
  submitted    // 학생 제출 완료, 자동채점 대기
  grading      // 자동채점 진행 중 (LLM 호출 인플라이트)
  reviewed     // AI 제안 생성 완료, 교사 검토 대기
  confirmed    // 교사 확정 (점수 결정), 미공개
  released     // 학부모·학생 공개
}
```

### 2.5 신규: AssessmentAnswer (문항별 답안)

```prisma
model AssessmentAnswer {
  id            String   @id @default(cuid())
  submissionId  String
  questionId    String

  // 답안 payload (유형별)
  //  MCQ:     { selectedChoiceIds: [string] }
  //  OX:      { value: "O"|"X" }
  //  NUMERIC: { value: number, unit?: string }
  //  SHORT/ESSAY:
  //            { textAnswer?: string,           // 키보드 입력
  //              inkImageUrl?: string,          // S-Pen 필기 PNG (Supabase Storage)
  //              inkStrokes?: Json,             // tldraw/perfect-freehand 스트로크 원본 (재쓰기·재채점용)
  //              ocrText?: string,              // LLM Vision이 읽어낸 텍스트 (학생 편집 가능 프리뷰)
  //              inputMode: "keyboard"|"ink"|"mixed" }
  payload       Json

  // 자동채점 결과 (suggestion)
  autoScore     Float?   // 0~points
  autoScoreMax  Float?   // = question.points
  autoCorrect   Boolean? // MCQ·OX·NUMERIC은 boolean, SHORT/ESSAY는 null (부분점수만)
  autoFeedback  String?  // LLM 근거 요약
  autoGradedAt  DateTime?
  autoProviderId String? // "gemini-flash" 등 (프로바이더 추상화)
  autoRawResponse Json?  // 프롬프트·응답 원본 (감사·재채점용)

  // 교사 확정 (오버라이드)
  finalScore    Float?
  finalFeedback String?
  overriddenAt  DateTime?

  // 제출 순간 snapshot — 사후 문항 수정 무관하게 고정
  questionSnapshot Json // stem·payload(sanitized) 복사본

  submission    AssessmentSubmission @relation(fields: [submissionId], references: [id], onDelete: Cascade)
  question      AssessmentQuestion   @relation(fields: [questionId], references: [id], onDelete: Restrict)

  @@unique([submissionId, questionId])
  @@index([submissionId])
}
```

### 2.6 신규: GradebookEntry (Aura 웹앱 성적탭용)

```prisma
// 교사가 "확정(confirm)" 버튼을 누르는 순간 AssessmentSubmission → GradebookEntry로 denormalize 복제.
// Aura 웹앱은 이 테이블만 읽어 성적부 렌더 (AssessmentSubmission은 Aura-board의 작업대, GradebookEntry는 성적부 영속 레코드).
// D1 공통 Supabase + RLS 전제이므로 Aura 웹앱은 직접 쿼리 (Realtime 구독 포함).
model GradebookEntry {
  id              String   @id @default(cuid())
  submissionId    String   @unique
  studentId       String
  classroomId     String   // denormalized for RLS·인덱스

  // 성적부 표시 필드 (denormalize)
  assessmentTitle String
  assessmentCategory String? // template.gradebookCategory
  subject         String?
  totalPoints     Int
  totalScore      Float
  percentile      Float?    // 선택: 학급 내 %

  // 상태
  confirmedAt     DateTime
  confirmedById   String
  releasedAt      DateTime? // null이면 학부모·학생에게 숨김 (E축 하이클래스 뷰 게이트)

  // 학부모·학생 노출 제어
  visibleToStudent Boolean  @default(true)
  visibleToParent  Boolean  @default(false) // releasedAt 세팅될 때 true

  // 루브릭별 세부 점수 (E2 Canvas popover 원본)
  rubricBreakdown Json?    // [{criterion, score, max, feedback}]

  // 교사 총평
  teacherComment  String?

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  submission      AssessmentSubmission @relation(fields: [submissionId], references: [id], onDelete: Cascade)
  student         Student              @relation(fields: [studentId], references: [id], onDelete: Restrict)
  classroom       Classroom            @relation(fields: [classroomId], references: [id], onDelete: Cascade)
  confirmedBy     User                 @relation(fields: [confirmedById], references: [id])

  @@index([classroomId, releasedAt])
  @@index([studentId])
}
```

### 2.7 신규: ProctorEvent (축 F L1 부정행위 로그)

```prisma
model ProctorEvent {
  id            String   @id @default(cuid())
  submissionId  String
  type          ProctorEventType
  ts            DateTime @default(now())
  durationMs    Int?     // visibility_hidden의 경우 감춰진 시간
  meta          Json?    // UA, 화면 크기, 스크롤 위치 등 보조 컨텍스트

  submission    AssessmentSubmission @relation(fields: [submissionId], references: [id], onDelete: Cascade)

  @@index([submissionId, ts])
  @@index([type])
}

enum ProctorEventType {
  visibility_hidden       // Page Visibility API
  focus_lost              // window blur
  fullscreen_exit         // fullscreenchange
  paste_blocked           // onpaste preventDefault
  copy_blocked            // oncopy preventDefault
  contextmenu_blocked     // oncontextmenu preventDefault
  window_open_blocked     // window.open 시도 차단
  idle_over_threshold     // idleThresholdSec 초과 무입력
}
```

### 2.8 관계 요약 다이어그램 (텍스트)

```
Classroom 1 ── N Student
Classroom 1 ── N Board
Board 1 ── 1 AssessmentTemplate (layout="assessment"일 때만)
AssessmentTemplate 1 ── N AssessmentQuestion
AssessmentTemplate 1 ── N AssessmentSubmission
Student 1 ── N AssessmentSubmission (각 template당 1회)
AssessmentSubmission 1 ── N AssessmentAnswer
AssessmentSubmission 1 ── N ProctorEvent
AssessmentSubmission 1 ── 1 GradebookEntry (confirm 시 생성)
AssessmentSubmission 0..1 ── 0..1 AssignmentSlot (선택적 연동)
Student 1 ── N ParentChildLink (기존) — 학부모는 릴리스된 GradebookEntry만 조회
Classroom 1 ── N GradebookEntry (RLS 주체: teacherId = classroom.teacherId)
```

---

## 3. 사용자 흐름

### 3.1 교사 출제 흐름

1. 교사가 Aura-board 내 `보드 생성` → `레이아웃: 수행평가` 선택 → `Board` 생성 후 `AssessmentTemplate` 자동 초기화.
2. 템플릿 메타 입력 (제목·교과·만점·카테고리·가중치·시간제한·응시 환경 옵션).
3. 문항 추가 — 유형 드롭다운(`MCQ/OX/NUMERIC/SHORT/ESSAY`) 선택 후 **심플 폼**(SurveyJS Form Library 렌더러 기반 자체 폼)으로 지문·정답·배점 입력.
4. 서술형(SHORT/ESSAY)엔 **루브릭 텍스트** 입력 (자연어, LLM 프롬프트에 그대로 주입).
5. (선택) AI 문항 생성 보조 — 교과서 단원 붙여넣기 → Gemini Flash로 초안 N개 생성 → 교사 편집·삭제 후 확정.
6. `미리보기` 모드에서 학생 화면 시뮬레이션 (정답 표시 off 상태 렌더).
7. `게시(publish)` 버튼 → `template.status="published"`, `publishedAt` 기록 → 학급 학생 모두에게 실시간 Realtime push.

### 3.2 학생 응시 흐름

1. 학생이 교실 Aura-board 접속 → "수행평가 [제목]" 배너 카드 탭.
2. 응시 안내 화면 → "시작" 터치 시 **`element.requestFullscreen()`** 호출 (축 F L1). 실패 시(브라우저 거부) 경고 후 진행 차단.
3. `AssessmentSubmission` row 생성 (`status="in_progress"`, `startedAt` 기록) + 타이머 시작.
4. 문항별 렌더링 — **lazy load** (화면에 보이는 ± 1개만 DOM 마운트, 성능 예산 준수).
   - MCQ/OX: 터치 한 번 선택, 로컬 state + 300ms debounce 서버 autosave.
   - NUMERIC: 숫자 키패드.
   - SHORT/ESSAY: **탭 전환 UI — [필기] | [키보드]**.
     - 필기: tldraw/perfect-freehand 캔버스 (고정 픽셀, 60fps). S-Pen 호버·필압 지원.
     - 키보드: 기본 `<textarea>` (최대 문자수 제한).
5. 모든 입력은 **IndexedDB 로컬 draft + Supabase autosave** 이중 저장 (네트워크 단절 대응).
6. L1 이벤트 감지 — Page Visibility·blur·fullscreen exit·paste/copy/contextmenu/window.open 이벤트를 `ProctorEvent`로 실시간 insert. `proctorEventCount` 3회 도달 시 학생 화면에 "감독 교사에게 통보됨" 배너 표시.
7. 학생 `제출` 터치 → S-Pen 필기는 PNG+stroke JSON 업로드(Supabase Storage) → `AssessmentSubmission.status="submitted"`, `submittedAt` 기록 → 전체화면 해제.
8. "채점 중" 로딩 상태(~수초) 후 학생에겐 **객관식 즉시 채점 결과만** 표시 (서술형은 "교사 확인 중").

### 3.3 자동채점 흐름

1. 서버가 `status="submitted"` 트리거(Supabase Realtime or Edge Function)로 채점 job 큐잉.
2. `status="grading"` 전환.
3. **결정론 채점 레인** — MCQ·OX·NUMERIC: 서버에서 정답과 비교, `autoScore`·`autoCorrect` 즉시 세팅. p95 < 100ms.
4. **LLM 채점 레인** — SHORT·ESSAY:
   - a. Sentence-Transformers(KR-SBERT) 전처리 — 공백·무관 답안 자동 0점 필터 (비용 절감).
   - b. 통과 답안은 `gradingProvider` (기본 Gemini 2.5 Flash)로 전송. 프롬프트: `{template.rubricText + question.rubric + modelAnswers + (inkImageUrl if ink else textAnswer)}`.
   - c. LLM 응답 JSON 파싱 → `autoScore`·`autoFeedback`·`autoRawResponse`·`ocrText`(필기일 때) 저장.
   - d. 판독 불가 케이스 → `autoCorrect=null, autoFeedback="판독 불가 — 교사 확인 요망"`.
5. 모든 문항 채점 완료 시 `AssessmentSubmission.suggestedScore` 합산·`suggestedFeedback` 생성·`gradedAt` 세팅 → `status="reviewed"`.
6. 교사 대시보드에 Realtime 배지(채점 완료 N/M).

### 3.4 교사 확정 흐름

1. 교사가 Aura-board `수행평가 > [제목] > 채점 대기` 탭 열기. **owner+데스크톱 전용** 매트릭스 뷰(학생 행 × 문항 열).
2. 각 셀 클릭 → Canvas 스타일 popover — AI 제안 점수·피드백·LLM 근거·원본 필기 이미지·OCR 텍스트.
3. 교사가 `finalScore` 오버라이드 or 그대로 수용. 개별 문항·전체 일괄 처리 둘 다 지원.
4. `확정(confirm)` 버튼 → `AssessmentSubmission.status="confirmed"`, `confirmedAt`·`confirmedById` 기록 + `GradebookEntry` row INSERT (denormalize 복제).
5. (이원화 E축) 교사가 추가로 `릴리스(release)` 버튼 누르면 `releasedAt` 세팅 → 학부모·학생에게 공개. `Classroom.gradebookReleasePolicy="on_confirm"`이면 confirm과 동시에 자동 릴리스.

### 3.5 학부모 열람 흐름 (parent-viewer v2 승계)

1. 학부모가 Aura 웹앱 로그인 → 자녀 단일 프로필 화면 (하이클래스 스타일).
2. `성적` 탭 → `GradebookEntry where studentId IN (linked children) AND releasedAt IS NOT NULL` — Supabase RLS 자동 필터.
3. 카드 리스트 뷰(타임라인) — 최근 수행평가 N개 → 카드 탭 시 상세 (총점·루브릭 breakdown·교사 총평).
4. **matrix/grid 뷰는 미노출** (모바일 PWA + viewer 권한 = 차단).
5. 학부모에겐 AI 제안 원본·오버라이드 이력·필기 이미지 **비공개** (교사 확정본만 노출).

### 3.6 부정행위 감독 흐름 (L2 교사 대시보드)

1. 교사가 응시 중 `보드 > 실시간 감독` 탭 열기 (**owner+데스크톱 전용**).
2. Supabase Realtime 구독으로 `ProctorEvent` 신규 row 실시간 수신.
3. 학생 그리드 — 각 학생 배지:
   - 🟢 정상 (이벤트 0~1)
   - 🟠 주의 (이벤트 2 또는 무활동 임박)
   - 🔴 경고 (이벤트 3+ 또는 critical flag)
4. 배지 클릭 → 이벤트 타임라인 (timestamp + 유형 + duration).
5. 교사 액션은 **수동 대면 확인** 전용. 자동 답안 락·자동 제출 강제는 **v1에서 미구현** (false positive 리스크, 축 F 결론).

---

## 4. 태블릿 성능 체크리스트 (`tablet-performance-roadmap.md` §2·§2a 기준)

- [ ] **초기 TTI (30문항 응시 화면) < 3s** — 문항 lazy 마운트, sanitized JSON 프리페치, SurveyJS core만 먼저 로드.
- [ ] **S-Pen 필기 캔버스 60fps** — tldraw/perfect-freehand 고정 픽셀(예: 800×400), 오프스크린 캔버스 비활성, 입력 이벤트 throttle 없이 직결.
- [ ] **iframe ≤ 1** — 수행평가 응시 화면 **자체 iframe 0**. Canva oEmbed 카드 혼재 시 최대 1개.
- [ ] **OCR는 클라우드 오프로드** — Tesseract.js 등 온디바이스 OCR **금지**. 제출 시 PNG 업로드 → 서버 LLM Vision 호출.
- [ ] **문항 렌더링 lazy** — 가상 스크롤 또는 "현재 문항 ± 1개"만 DOM 마운트. MCQ 30문항 × 4보기 = 120 DOM 일괄 렌더 금지.
- [ ] **제출물 청크 업로드** — S-Pen PNG은 문항별 개별 업로드(문항 이동 시 background flush). 제출 버튼에서 한꺼번에 올리지 않음.
- [ ] **IndexedDB 오프라인 드래프트** — `idb-keyval`(기존 tablet-performance §3 재사용)로 답안·필기 스트로크 로컬 저장. 재접속 시 diff 동기화.
- [ ] **Realtime 구독 스코프** — 학생 탭은 자기 `AssessmentSubmission`만, 교사 대시보드는 `templateId` 스코프. 보드 전체 ChangeFeed 금지.
- [ ] **델타 페이로드** — 답안 autosave는 변경 문항만 PATCH. 전체 submission 재전송 금지.
- [ ] **드래그 중 프레임 60fps** — 응시 화면엔 드래그 없음(해당 없음). 교사 대시보드 매트릭스는 **데스크톱 전용**이므로 태블릿 예산 미포함.
- [ ] **DevTools 감지 휴리스틱은 선택**: 성능 비용 적으나 false positive 많음. v1은 비활성, v1.5 옵션.

---

## 5. Aura-board ↔ Aura 웹앱 송신 채널 (축 D 1순위 구체화)

### 5.1 기본 경로 — 공통 Supabase + RLS + Realtime Postgres Changes (D1)

전제: **Aura-board와 Aura 웹앱은 동일 Supabase 프로젝트**를 공유한다 (미결 질문 Q1에서 최종 확인, 분리 시 5.2 폴백).

1. 교사 `확정` → Aura-board 서버 액션이 `GradebookEntry` INSERT.
2. Aura 웹앱이 `supabase.channel('public:GradebookEntry')` 구독 — RLS 정책이 아래를 자동 적용:
   - **teacher**: `classroomId IN (SELECT id FROM Classroom WHERE teacherId = auth.uid())`
   - **student**: `studentId = auth.student_id() AND visibleToStudent = true`
   - **parent**: `studentId IN (SELECT studentId FROM ParentChildLink WHERE parentId = auth.parent_id() AND status='active') AND releasedAt IS NOT NULL`
3. 레이턴시 목표: p95 **< 500ms** (확정 클릭 → 성적탭 표시).

### 5.2 폴백 — PAT 웹훅 (D2, canva-publisher-receiver 포크)

Supabase 분리 배포일 경우 `ExternalAccessToken` + `/api/external/grades` 엔드포인트로 폴백. canva-publisher-receiver-roadmap 패턴 승계 — rate limit·3-stage migration·HMAC 서명 재사용. phase3에서 Q1 결정 후 택일.

### 5.3 실패 시 재시도 큐 (PGMQ)

- 채점·송신 pipeline의 각 단계 실패(LLM 타임아웃·Supabase 레이턴시·네트워크 끊김)는 `pgmq.grading_retry` 큐에 enqueue.
- 워커가 SKIP LOCKED로 5회 재시도, exponential backoff (2·4·8·16·32s).
- 5회 실패 시 `AssessmentSubmission.status="grading"` 유지 + 교사 대시보드에 `retry_exhausted` 배지 → 수동 재채점 버튼.

---

## 6. 부정행위 방지 구현 레이어 (축 F L1~L4 구체화)

### 6.1 L1 — UX 신호 (무공수 웹 표준)

| 이벤트 | 브라우저 API | 코드 훅 | 로깅 엔티티 |
|---|---|---|---|
| 탭 전환·앱 전환·스크린락 | Page Visibility API | `document.addEventListener('visibilitychange', fn)` — `visibilityState === 'hidden'` 시 start, `visible` 복귀 시 end. 로그에 `durationMs` 기록 | `ProctorEvent(type=visibility_hidden)` |
| 포커스 손실 | Window focus/blur | `window.addEventListener('blur'/'focus', fn)` | `ProctorEvent(type=focus_lost)` |
| 풀스크린 해제 | Fullscreen API | `document.addEventListener('fullscreenchange', fn)` — `document.fullscreenElement === null` 시 기록 + 재진입 다이얼로그 | `ProctorEvent(type=fullscreen_exit)` |
| 붙여넣기 | Clipboard events | `<element>.onpaste = e => { e.preventDefault(); log(); }` | `ProctorEvent(type=paste_blocked)` |
| 복사 | `oncopy = e => { e.preventDefault(); log(); }` | `ProctorEvent(type=copy_blocked)` |
| 우클릭/롱프레스 컨텍스트 | `oncontextmenu = e => { e.preventDefault(); log(); }` | `ProctorEvent(type=contextmenu_blocked)` |
| 새 창 개방 | `window.open` 오버라이드 | 응시 컴포넌트 mount 시 `window.open = (...args) => { log(); return null; }` (unmount 시 원복) | `ProctorEvent(type=window_open_blocked)` |
| 무활동 | `setInterval` + 입력 이벤트 lastInput | `idleThresholdSec` 초 동안 pointer/key 입력 없음 시 fire | `ProctorEvent(type=idle_over_threshold)` |

로깅 경로: 응시 컴포넌트 → `/api/assessment/[submissionId]/proctor-events` (debounce 2s, 배치 payload) → Supabase insert → 교사 Realtime.

### 6.2 L2 — 교사 실시간 대시보드 (owner+데스크톱 전용)

- 경로: `/teacher/assessments/[templateId]/proctor` (Aura-board 내부).
- 구성: 학생 그리드(학급 인원 × 응시 상태). 각 셀은 `AssessmentSubmission.proctorFlag`·`proctorEventCount`를 Realtime 구독.
- 상태 배지: 🟢/🟠/🔴 (임계값 기본 0 / 1~2 / 3+, 교사 설정 가능).
- 클릭 시 `ProctorEvent` 타임라인 drawer.
- 자동 제재 금지 — 교사 대면 확인 전용 (MEMORY autonomous_mode·false positive 리스크).

### 6.3 L3 — 문항 설계 방어 (서버측)

| 방어 | 구현 위치 | 상세 |
|---|---|---|
| 정답 서버 보유 | `AssessmentQuestion.payload` 서버 보관, 클라이언트엔 sanitized(정답 제거) JSON만 | `/api/assessment/[id]/questions?role=student` 응답에서 `correctChoiceIds`·`correctAnswer`·`expected`·`modelAnswers` 필드 stripped |
| 문항풀 랜덤화 | `template.randomizePool=true` 시 서버가 학생 seed 기반 N개 추출 | 학생별 `seededShuffle(templateId + studentId)` |
| 보기 순서 셔플 | `template.shuffleChoices=true` | MCQ choices 배열을 학생별 seed로 셔플. 서버는 원본·매핑 테이블 보유. `AssessmentAnswer.questionSnapshot`에 학생이 본 순서 고정 기록 |
| 제출 후 재응시 락 | `AssessmentSubmission.status='submitted'` 이후 PATCH 거부 | 서버 가드 + 클라이언트 UI 비활성 |
| 시간 제한 강제 | 서버가 `startedAt + durationMin` 초과 시 제출 거부·자동 제출 | Edge Function cron 또는 제출 요청 시 검증 |
| Service Worker 화이트리스트 | 응시 중 외부 도메인 fetch 차단 | 응시 페이지에서 SW 등록 — `fetch` 핸들러가 `origin === location.origin` 또는 Supabase·Storage만 통과. 미통과 시 `ProctorEvent(type=window_open_blocked)` 유사 로그 |

### 6.4 L4 — Knox Kiosk (학교 단말 옵션)

- 조건: `Classroom.schoolManagedDevices = true` + `AssessmentTemplate.kioskMode = true` 동시 충족 시 활성.
- 구현: v1에선 **UI 플래그만 노출 + 안내 문서**(교사에게 "학교 관리 단말에서 Knox Kiosk 프로필 적용하세요" 가이드). 실 MDM 연동은 v2+.
- 프로파일: Samsung Knox Manage → Single App Kiosk → Aura Board PWA를 홈 앱으로 고정 + 홈·최근앱 버튼 차단 + 알림 셔터 차단.
- BYOD 기본 환경에선 **비활성** (MEMORY·phase1 축 F 결론).

---

## 7. Tier·권한 매트릭스

### 7.1 Tier 한도 (즉결은 하지 않음 — 미결 Q1 후보)

| 기능 | Free | Pro | 비고 |
|---|---|---|---|
| AssessmentTemplate 생성 | **월 N회 제한 (Q2)** | 무제한 | v1 sketch 기본값: Free 월 3회 제안, phase3 확정 |
| LLM 채점(Gemini Flash) | Free는 월 200문항 한도 | Pro 월 5000문항 | 비용 상한 Q3 |
| 학급 응시 인원 | 30명까지 | 무제한 | Aura-board 기본값 승계 |
| GradebookEntry 보관 | 1학기 | 학년 단위 + export | 개인정보 최소보관 원칙 |
| 교사 대시보드 L2 | 기본 제공 | 기본 제공 | 모두 제공 |
| L4 Knox Kiosk 토글 | 비노출 | 노출 | Enterprise tier 승격 검토 (미결 Q7) |

### 7.2 성적 RLS 권한 표

| 역할 | 읽기 | 쓰기 | 비고 |
|---|---|---|---|
| **teacher** (owner) | 자기 Classroom 전체 GradebookEntry·AssessmentSubmission·ProctorEvent | 확정·릴리스·오버라이드 | 담임만, 다른 반 교차 조회 금지 |
| **student** (editor) | 자기 `studentId` GradebookEntry (visibleToStudent=true, releasedAt or confirmed) + 자기 AssessmentSubmission | 응시 중 AssessmentAnswer write only, 제출 후 불가 | AI 제안 원본 비공개 |
| **parent** (viewer) | 연결된 자녀의 `releasedAt IS NOT NULL` GradebookEntry만 | 없음 | matrix 미노출, 자녀 단일 뷰만 |
| **admin** (시스템) | 감사 목적 전체 | 데이터 수정 금지 | 접근 로그 기록 |

---

## 8. 리스크 표

| # | 리스크 | 영향 | 완화 |
|---|---|---|---|
| R1 | 서술형 LLM 오채점 (학생 이의제기·민원) | 신뢰 하락·법적 분쟁 | 교사 확정(confirm) 필수. AI 점수는 "제안"으로만 저장. `AssessmentAnswer.autoRawResponse`에 프롬프트·응답 보관해 이의 제기 시 교사 재검토 |
| R2 | OCR(한글 손글씨) 정확도 불균형 | 서술형 점수 왜곡 | 학생에 **재쓰기·키보드 전환 토글** 제공. OCR 결과 **편집 가능 프리뷰**(`ocrText` 필드)로 학생이 응시 중 교정 가능. 판독 불가는 명시 플래그 → 교사 수동 검토 |
| R3 | 네트워크 단절 중 답안 손실 | 응시 중단·학생 피해 | IndexedDB 로컬 드래프트 + Supabase autosave 이중화. 재접속 시 diff 머지. `AssessmentSubmission.localDraftHash`로 무결성 검증 |
| R4 | 부정행위 탐지 false positive (알림 배너로 visibility 순간 hidden 등) | 학생·학부모 민원, 교사 오판 | **로그는 기록만, 자동 처분 금지**. L2 대시보드 표시 + 교사 대면 확인 필수. 임계값 교사 조정 가능 |
| R5 | 학부모 민원 (점수 열람 범위·시점) | 학교 도입 저항 | parent-viewer v2 RLS 재사용 + 교사 릴리스 버튼(`releasedAt`)로 공개 타이밍 제어. 정책은 `Classroom.gradebookReleasePolicy`로 학급별 설정 |
| R6 | 갤탭 S6 Lite 렉 (대용량 응시 화면) | 응시 실패·제출 못함 | 핵심 경로 iframe ≤ 1, 문항 lazy, S-Pen 캔버스 고정 픽셀 60fps. 30문항 기준 TTI < 3s 검증 게이트 |
| R7 | 개인정보(성적·필기) 저장 | 개인정보보호법 위반 리스크 | 최소 보관(1학기~학년 단위 + export 후 파기), Supabase at-rest 암호화, 감사 로그 `accessId`. LLM 국외 이전 동의서 학기초 일괄 수령 |
| R8 | LLM 비용 폭주 (학급 수 × 문항 수 × 재채점) | 운영비 적자 | Pro tier 월 상한 + Sentence-Transformers 전처리 필터 + Gemini Flash (최저가) 기본. 비용 대시보드는 관리자용 별도 페이지 |
| R9 | DB 공유 전제 붕괴 (D1→D2 전환) | 대규모 재설계 | phase3 Q1에서 확정. 폴백 경로 PAT 웹훅(D2)을 sketch 단계에서 미리 문서화하여 전환 가능성 확보 |

---

## 9. 미결 질문 (phase3 인터뷰 재료)

> 6개 — 에이전트가 즉결 불가하고 사용자 방침이 필요한 핵심 결정점만.

1. **Q1 (D축 인프라 관계)**: Aura-board와 Aura 웹앱은 **동일 Supabase 프로젝트**를 공유하는가, **완전 분리된 Supabase**인가? → D1(Realtime+RLS) vs D2(PAT 웹훅) 결정. 답변에 따라 5.1↔5.2 경로 선택.
2. **Q2 (Tier 스코프)**: 수행평가 기능은 **Pro tier 전용**인가, Free에서도 **월 N회(예: 3회) 허용**인가? 또한 LLM 채점 월 상한 설정값?
3. **Q3 (LLM 벤더·비용 상한)**: 서술형 채점 1순위 벤더는 **Gemini 2.5 Flash**(sketch 기본) 유지인가? **월 비용 상한**(예: 교사당 $X, 학급당 $Y)은? 민감 답변(사회·도덕·문학)을 Claude Haiku 4.5로 라우팅하는 옵션 포함 여부?
4. **Q4 (학부모 공개 시점)**: `GradebookEntry` 학부모 공개 시점은 **"교사 확정 후 즉시(`on_confirm`)"** 기본인가, **"교사가 릴리스 버튼 따로 눌러야"** 기본인가? `Classroom.gradebookReleasePolicy` 기본값 확정.
5. **Q5 (성적탭 UI 위치)**: Aura 웹앱 성적탭은 **신규 route** (`/aura-web/gradebook`)인가, **기존 assignment-board 탭에 성적 컬럼 추가** 방식인가, 아니면 **Aura 웹앱 별도 페이지 + Aura-board 내부 교사용 매트릭스 별도 2곳 운영**인가?
6. **Q6 (v1 문항 유형 스코프)**: v1 배포에 포함할 문항 유형은? **OMR(MCQ·OX) 전용** / **객관식+NUMERIC** / **서술형 SHORT 포함** / **ESSAY까지 전부 포함**. 스코프가 크면 v1 3개월, 전부 포함 시 6~8주 추가.

> **부록 (자율 결정 후보 — 사용자 이견 없으면 sketch 기본값 유지)**:
> - AI 문항 생성 보조(교과서 붙여넣기 → Gemini 초안): v1 포함 기본. 불필요 시 phase3에서 삭제.
> - 부정행위 임계값 기본(🟠=1~2, 🔴=3+): 교사 조정 가능. 기본값 유지 권고.
> - L4 Knox Kiosk: v1은 플래그만, 실 MDM 연동은 v2 파킹.

---

## 10. 부록 — 권고 1안 ↔ 2안 전환 지점

| 지점 | 1안 (기본) | 2안 (분리 배포 폴백) |
|---|---|---|
| D 채널 | 공통 Supabase + Realtime | `/api/external/grades` PAT 웹훅 |
| L2 대시보드 | Realtime 구독 즉시 | 웹훅 이벤트 polling(~5s) |
| OCR | Gemini Vision 통합 | CLOVA OCR → 별도 LLM 채점 2-step |
| 개발 공수 | 6~8주 + L1·L2·L3 1~1.5주 | 동일 범위지만 보안 30% 절감 대신 지연·OCR 2-step 추가 |

**전환 스위치**: `Classroom.schoolManagedDevices` 플래그와 독립. 배포 시점 환경변수 `AURA_WEB_SUPABASE_MODE=shared|separate`로 결정.

---

Phase 2 완료. 엔티티 7개, 흐름 6개, 미결 6개. 축 F L1~L4 레이어 스케치 포함. 다음: phase3 interview.
