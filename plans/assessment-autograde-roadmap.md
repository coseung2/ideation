# Aura-board 수행평가 자동채점 파이프라인 로드맵 (v1)

> 작성일: 2026-04-16
> Seed: `seed_0badf1e571bc` (task `2026-04-16-performance-assessment-autograde`, ambiguity 0.10)
> Interview: `interview_20260415_224854` (보충 세션, 이전 `interview_20260415_223155` ambiguity 0.15 종결)
> 관련 plan:
> - `./assignment-board-roadmap.md` (채점·성적 송신 레이어 상위 확장 — 본 로드맵이 담당)
> - `./parent-viewer-roadmap.md` (성적 학부모 열람 정책 승계, 탭 편입 여부는 §12.1 분기 3.1 미결)
> - `./canva-publisher-receiver-roadmap.md` (송신 채널 비교 레퍼런스만, 본 v1은 Supabase 공통 채널로 분기)
> - `./tablet-performance-roadmap.md` (응시 화면 iframe 0, OCR 클라우드 오프로드, S-Pen 800×400 고정 px)
> - `./seeds-index.md` (Seed 12 등재)
> - `./phase0-requests.md` (padlet 진입 블록)

---

## 0. 핵심 명제

> **북극성 5 primitive 고정**: (1) **교사 출제** (AssessmentTemplate + AssessmentQuestion, MCQ/SHORT 2종) · (2) **학생 응시** (AssessmentSubmission + AssessmentAnswer, 드래프트 IndexedDB + Supabase autosave 이중화) · (3) **자동채점** (MCQ 서버 결정론 매칭 + SHORT Gemini 2.5 Flash LLM + PGMQ 재시도) · (4) **교사 확정** (GradebookEntry finalScore + 릴리스 버튼) · (5) **성적 송신** (공통 Supabase + RLS 3분화 + Realtime p95 < 500ms, `/aura-web/gradebook` 학생·학부모 뷰 + `/teacher/assessments/[id]/gradebook` 교사 매트릭스)

v1은 MCQ + SHORT 2종 고정. OX/NUMERIC은 v1.5 파킹, ESSAY는 v2 파킹. 매트릭스 뷰는 owner + 데스크톱 전용. 부정행위 정책은 **알림 + 자동 화면이탈 잠금(L1 강화)** 하이브리드 — `isLocked/lockedReason`을 Supabase에 영속화해 새로고침 우회 차단. Pro 전용 + `FeatureFlag.assessmentTierGate` 런칭 플래그(개발·베타 false).

---

## 1. Prisma 최종 확정 스키마

### 1.1 Board 확장 (assignment-board 패턴 승계)

```prisma
model Board {
  // ... 기존 유지 ...
  layout      String @default("freeform")   // + "assessment" (freeform/event-signup/breakout/assignment/assessment)
  // assessment 전용 메타는 AssessmentTemplate에서 관리 (Board는 진입 셸 역할)
}
```

### 1.2 신규 엔티티 7종

```prisma
// (1) 평가 템플릿 — 1 Classroom × N AssessmentTemplate
model AssessmentTemplate {
  id               String   @id @default(cuid())
  classroomId      String
  boardId          String?                                  // Board(layout="assessment") 진입 셸 참조
  title            String
  rubricText       String   @default("")                    // 공통 자연어 루브릭 (Gemini 프롬프트 주입)
  durationMin      Int                                       // 제한 시간(분). endAt = startedAt + durationMin 고정
  gradingProvider  String   @default("gemini-flash")        // 프로바이더 추상화, 기본 Gemini 2.5 Flash
  kioskMode        Boolean  @default(false)                 // v1은 UI 플래그만 (실 MDM 연동 없음)
  tierGate         Boolean  @default(false)                 // 런칭 플래그. FeatureFlag.assessmentTierGate와 연동
  createdById      String                                    // 교사
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  classroom  Classroom              @relation(fields: [classroomId], references: [id], onDelete: Cascade)
  questions  AssessmentQuestion[]
  submissions AssessmentSubmission[]

  @@index([classroomId])
  @@index([boardId])
}

// (2) 평가 문항 — v1 허용 kind = MCQ, SHORT (Zod validation gate)
model AssessmentQuestion {
  id          String   @id @default(cuid())
  templateId  String
  order       Int                                              // 응시 순서 (lazy 마운트 ±1)
  kind        String                                           // enum: "MCQ" | "OX" | "NUMERIC" | "SHORT" | "ESSAY". v1 create API는 MCQ·SHORT만 허용
  prompt      String                                           // 문제 본문
  payload     Json                                             // kind별 데이터
  //   MCQ:   { choices: [{id, text}], correctChoiceIds: string[] }
  //   SHORT: { modelAnswers: string[], keywords: string[], partialCredit: boolean }
  //   OX/NUMERIC/ESSAY: schema-only, v1 create 차단
  rubric      String   @default("")                           // 문항별 자연어 루브릭 override
  maxScore    Int      @default(1)
  aiGeneratedBy   String?                                      // "gemini-flash" 등 (AI 생성 보조)
  aiSourcePrompt  String?                                      // 교과서 paste 원문
  createdAt   DateTime @default(now())

  template AssessmentTemplate @relation(fields: [templateId], references: [id], onDelete: Cascade)
  answers  AssessmentAnswer[]

  @@unique([templateId, order])
  @@index([templateId])
}

// (3) 학생 제출 (시험 응시 레코드)
model AssessmentSubmission {
  id            String   @id @default(cuid())
  templateId    String
  studentId     String
  status        String   @default("in_progress")              // "in_progress" | "submitted" | "retry_exhausted"
  startedAt     DateTime
  endAt         DateTime                                       // = startedAt + durationMin 고정, 정지 없음
  submittedAt   DateTime?                                      // 학생 명시 제출 or 서버 cron 자동 전이
  isLocked      Boolean  @default(false)                       // 화면이탈 잠금 상태 (Supabase 영속화)
  lockedReason  String?                                         // "visibility_hidden" | "fullscreen_exit" | "focus_lost" | "teacher_manual" | null
  localDraftHash String?                                        // IndexedDB 드래프트 무결성 검증
  createdAt     DateTime @default(now())

  template AssessmentTemplate @relation(fields: [templateId], references: [id], onDelete: Cascade)
  answers  AssessmentAnswer[]
  gradebookEntry GradebookEntry?
  proctorEvents  ProctorEvent[]

  @@unique([templateId, studentId])                             // 1 학생 × 1 템플릿 × 1 제출
  @@index([templateId])
  @@index([studentId])
  @@index([status])
  @@index([endAt])                                              // cron 시간초과 스캔
}

// (4) 개별 답안
model AssessmentAnswer {
  id              String   @id @default(cuid())
  submissionId    String
  questionId      String
  payload         Json                                            // MCQ: { selectedChoiceIds: string[] }, SHORT: { textAnswer?: string, inkImageUrl?: string }
  autoRawResponse String?                                         // LLM 원본 응답 (감사용)
  autoScore       Int?                                            // 자동 채점 점수 (MCQ/SHORT)
  manualScore     Int?                                            // 교사 수동 보정 (null이면 autoScore 사용)
  updatedAt       DateTime @updatedAt

  submission AssessmentSubmission @relation(fields: [submissionId], references: [id], onDelete: Cascade)
  question   AssessmentQuestion   @relation(fields: [questionId], references: [id], onDelete: Cascade)

  @@unique([submissionId, questionId])
  @@index([submissionId])
}

// (5) 성적부 항목 — 교사 확정 + 릴리스 게이트
model GradebookEntry {
  id            String   @id @default(cuid())
  submissionId  String   @unique
  finalScore    Int                                             // 교사 확정 점수 (autoScore·manualScore 집계 결과 교사 확인)
  visibleToStudent Boolean @default(true)                       // 학생 뷰 필터 플래그 (릴리스 후 진정한 노출은 releasedAt으로 통제)
  releasedAt    DateTime?                                        // 릴리스 버튼 누른 시각. null이면 학생·학부모 비노출 (teacher_manual 정책)
  releasedById  String?
  createdById   String                                            // 확정 교사
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  submission AssessmentSubmission @relation(fields: [submissionId], references: [id], onDelete: Cascade)

  @@index([releasedAt])
  @@index([visibleToStudent])
}

// (6) 감독 이벤트 — 전수 기록 (자동 상태전이 없음, 알림만)
model ProctorEvent {
  id           String   @id @default(cuid())
  submissionId String
  type         String                                             // "visibility_hidden" | "fullscreen_exit" | "focus_lost" | "teacher_resume"
  durationMs   Int                                                // 이벤트 지속 시간(ms). 잠금 기간 보존용
  createdAt    DateTime @default(now())

  submission AssessmentSubmission @relation(fields: [submissionId], references: [id], onDelete: Cascade)

  @@index([submissionId])
  @@index([type])
}

// (7) FeatureFlag (tierGate 런칭 플래그 전용. env var + DB row 이중 방어)
model FeatureFlag {
  key       String   @id                                          // "assessmentTierGate"
  enabled   Boolean  @default(false)
  updatedAt DateTime @updatedAt
  updatedBy String?
}
```

### 1.3 기존 엔티티 확장 필드

```prisma
model Classroom {
  // ... 기존 유지 ...
  gradebookReleasePolicy String @default("teacher_manual")     // "teacher_manual" 고정. 학급 단위 오버라이드는 v2
  schoolManagedDevices   Boolean @default(false)                // kioskMode 플래그 연동 (v1 UI 표시만)
}

model AssignmentSlot {
  // ... 기존 유지 ...
  // assessment 연동은 v2. v1에서 AssessmentSubmission은 AssignmentSlot과 독립 경로.
}
```

---

## 2. 사용자 흐름 (6개 섹션)

### 2.1 교사 출제 흐름

1. `/teacher/assessments/new` 진입 → Classroom 선택 → AssessmentTemplate 생성 (title · durationMin · rubricText).
2. 문항 추가 UI — **드롭다운은 MCQ·SHORT 2종만** 노출 (U5). create API Zod gate가 `kind IN ("MCQ","SHORT")` 검증.
3. **MCQ 입력**: 보기 N개 + correctChoiceIds 체크박스 선택. LLM 호출 없음.
4. **SHORT 입력**: 4필드 폼
   - (필수, ≥1건) 모범답안 예시 → `payload.modelAnswers[]`
   - (선택) 필수 키워드(쉼표 구분) → `payload.keywords[]`
   - (체크박스) 부분점수 허용 → `payload.partialCredit`
   - (선택) 문항별 자연어 루브릭 override → `AssessmentQuestion.rubric`
5. AI 문항 생성 보조(선택): 교과서 단원 paste → Gemini Flash 초안 N개 → 교사 편집·확정. `aiGeneratedBy`·`aiSourcePrompt` 보관.
6. "게시" → Board(layout="assessment") 생성 or 기존 진입 셸 연결. Classroom 전체 학생에게 기본 공개.

### 2.2 학생 응시 흐름

1. 로그인 → 학급 보드 → 수행평가 카드 진입 → `/assessment/[templateId]/take`.
2. "응시 시작" 클릭 → 서버 AssessmentSubmission INSERT (`startedAt = now()`, `endAt = startedAt + durationMin`).
3. 문항 ±1개 lazy 마운트 (30문항 × 4보기 일괄 렌더 금지 — Snapdragon 720G TTI < 3s 보장).
4. 답안 입력 시 IndexedDB 로컬 draft + Supabase autosave 300ms debounce. `localDraftHash` 무결성 검증. 문항별 background flush (일괄 업로드 금지).
5. **S-Pen 캔버스** (SHORT inkImageUrl 전용): tldraw/perfect-freehand 고정 픽셀 **800×400**, 60fps, throttle 없음. 동의서 미제출 학생은 손글씨 비활성(키보드만).
6. 남은 시간 카운트다운 표시. 잠금 시 dim + "잠금 중 — 시간은 계속 흐릅니다" 고지(§2.6 참조).
7. "제출" 클릭 → `status="submitted"`. 이후 PATCH 거부(재응시 락 L3).
8. 시간 초과 시 서버 cron이 `in_progress → submitted` 자동 전이 (객관적 사실, 부정행위 아님).

### 2.3 자동채점 흐름

1. `AssessmentSubmission.status = "submitted"` 트리거.
2. 각 `AssessmentAnswer`를 순회:
   - **MCQ**: 서버가 `question.payload.correctChoiceIds` ↔ `answer.payload.selectedChoiceIds` **결정론 매칭**. 전부 일치 → `autoScore = maxScore`, 아니면 0. **LLM 호출 금지 가드**(accept criteria).
   - **SHORT**: KR-SBERT 코사인 유사도 < 0.25 또는 10자 미만 → 자동 0점(LLM 스킵). 그 외 Gemini 2.5 Flash 호출 — 프롬프트 = `template.rubricText + question.rubric + modelAnswers + keywords + (inkImageUrl via Gemini Vision OR textAnswer)`. `partialCredit=false` 시 이진 처리(전부맞음/전부틀림), 기본은 부분점수. `autoRawResponse`에 원본 응답 저장.
3. 채점 실패 → PGMQ `grading_retry` 큐 enqueue. exponential backoff 2·4·8·16·32s, 5회 실패 시 `status="retry_exhausted"` + 교사 매트릭스 배지.
4. AI 제안(autoScore)은 교사 확정 전 학생·학부모에게 **완전 비공개**.

### 2.4 교사 확정 흐름

1. `/teacher/assessments/[id]/gradebook` (owner + 데스크톱 전용. 학생 × 문항 > 30×30 시 react-virtual).
2. autoScore 열람 + 수동 보정(`manualScore`) 가능. `retry_exhausted` 배지는 수동 채점 경로로 전환.
3. "성적 확정" → GradebookEntry INSERT (`finalScore` 계산, `releasedAt=null` 유지).
4. "릴리스" 버튼 클릭 → `GradebookEntry.releasedAt = now()` SET → Supabase Realtime broadcast → 학생·학부모 `/aura-web/gradebook` 즉시 표시(p95 < 500ms).

### 2.5 학부모 열람 흐름 (parent-viewer 승계)

- **v1.5 integrate 시점 미결** (본 로드맵 §12.1): parent-viewer v2의 "성적" **탭 편입** vs `/aura-web/gradebook` **별도 route 신설**.
- 현재 합의: Aura 웹앱 내부 · 동일 RLS · 동일 PWA 셸. UI 트리 결정은 v1.5 integrate에서.
- `ParentChildLink.status='active'` + `GradebookEntry.releasedAt IS NOT NULL` 이중 조건만 노출.

### 2.6 부정행위 감독 흐름 (U3 자동 화면이탈 잠금 하이브리드)

1. 클라이언트 Page Visibility API + `fullscreenchange` + `focusout` 구독. 이벤트 발생 → 답안 영역 즉시 disable.
2. `POST /api/assessment/[submissionId]/lock` (debounce 불가, 즉시 호출) → 서버가 `isLocked=true`, `lockedReason` 기록 + ProctorEvent INSERT. 네트워크 단절 시 클라이언트 로컬 `isLocked=true` 유지(안전 측), 재접속 시 서버 state가 ground truth.
3. 교사 대시보드에 Realtime broadcast → 배지 표시. 자동 제출·자동 처벌·자동 상태 전이 없음.
4. **해제 경로 2종**:
   - 학생 "돌아왔습니다" 모달 클릭 → `POST /api/assessment/[submissionId]/unlock?by=student` → `isLocked=false, lockedReason=null`.
   - 교사 "재개 승인" → `POST /api/assessment/[submissionId]/unlock?by=teacher&teacherId=X` → 동일 갱신 + ProctorEvent(`type="teacher_resume"`).
5. RLS: unlock API는 teacher(Classroom owner) 또는 본인 student만 호출 가능 (타 학생 unlock 차단).
6. 학생 페이지 mount 시 submission row 로드 → `isLocked=true`면 자동으로 잠금 UI 복원(새로고침 우회 차단).
7. Realtime 구독: 학생 탭이 자기 submission row `isLocked` 필드 변화 수신 → 교사 재개 승인 즉시 UI 해제.
8. **시간 계속 흐름**: `endAt = startedAt + durationMin` 고정, 잠금 중에도 타이머 진행. UI dim + "잠금 중 — 시간은 계속 흐릅니다" 고지. 피해 구제는 교사 개별(v1.5 `endAt` 연장 검토 또는 finalScore 보정).

---

## 3. 성능 예산 체크리스트 (태블릿 — 갤럭시 탭 S6 Lite)

`tablet-performance-roadmap.md §2` 공통 예산 위에 응시 화면 전용 제약:

- [ ] 응시 화면 **iframe 0** (가상화·LRU 포함 일체 금지)
- [ ] 현재 문항 **±1개만 DOM 마운트** (30문항 일괄 렌더 금지, Snapdragon 720G TTI < 3s)
- [ ] 썸네일·이미지 T0-④ 파이프라인(160×120 WebP + lazy)
- [ ] S-Pen 캔버스: tldraw/perfect-freehand **고정 픽셀 800×400**, 60fps, 입력 throttle 없음
- [ ] OCR **클라우드 오프로드 전용** — Tesseract.js·온디바이스 금지. `inkImageUrl` PNG → Gemini Vision 1-round-trip
- [ ] 드래프트: IndexedDB + Supabase autosave 300ms debounce, `localDraftHash` 무결성
- [ ] 문항별 background flush (제출 버튼 일괄 업로드 금지 — 네트워크 단절 대응)
- [ ] Realtime 구독 스코프: 학생=자기 submission, 교사 proctor 대시보드=templateId. 보드 전체 ChangeFeed 금지
- [ ] 갤럭시 탭 S6 Lite Chrome 실측 응시 TTI < 3s, 1시간 사용 후 메모리 < 500MB

매트릭스 뷰는 owner+데스크톱 전용으로 태블릿 예산 적용 대상 아님(`project_matrix_desktop_only` 방침).

---

## 4. 부정행위 방지 L1~L4 레이어 구현 가이드

| 레이어 | v1 구현 | 근거 |
|---|---|---|
| **L1 UX 신호 (강화)** | Page Visibility API + fullscreenchange + focusout 구독 → 답안 영역 disable + 서버 lock API 즉시 호출 + Realtime 배지 + "돌아왔습니다" 모달/교사 재개 승인 2경로 해제 | U3 확정(decisions §6·§7.2.3) |
| **L2 교사 대시보드** | `/teacher/assessments/[id]/gradebook` 실시간 proctor 배지(🟢 0~1 / 🟠 2 / 🔴 3+ 이벤트). Realtime p95 < 500ms | sketch §L2, v1 고정 임계값 |
| **L3 재응시 락·SW 방어** | `status="submitted"` 이후 PATCH 거부. Service Worker 외부 도메인 fetch 화이트리스트(same-origin + Supabase + Storage). 이탈 시 `window_open_blocked` 유사 로그 | sketch §L3 |
| **L4 Knox Kiosk (플래그만)** | `Classroom.schoolManagedDevices && AssessmentTemplate.kioskMode` UI 플래그 + 가이드 문서. **실 MDM 연동 0** — v2+ Enterprise | U3 BYOD 기본 환경, v1 스코프 제한 |

**자동 제출·자동 처벌·자동 상태전이 없음.** 서버 cron 자동 전이는 시간초과 1건만 예외(객관적 사실).

---

## 5. 송신 채널 D1(공통 Supabase + RLS + Realtime + PGMQ)

**채택: D1 공통 Supabase 단일 프로젝트 공유** (Aura-board ↔ Aura 웹앱).
- 근거: parent-viewer v2가 이미 공통 Supabase 전제. Realtime p95 < 500ms 요구는 D1만 충족. 분리 시 RLS 이중 관리·PAT 웹훅 2~3주 추가.
- canva-publisher-receiver의 PAT + `/api/external/cards` 패턴은 **외부 앱(Canva) 수신** 전용이며 본 로드맵의 내부 Aura-board ↔ Aura 웹앱 성적 송신에는 **사용하지 않음** (비교 레퍼런스로만 유지).

**RLS 3분화**:
- teacher: `Classroom owner` (owner_id = auth.teacher_id())
- student: `AssessmentSubmission.studentId = auth.student_id()` + `GradebookEntry.visibleToStudent = true`
- parent: `ParentChildLink.status='active'` + `GradebookEntry.releasedAt IS NOT NULL`

**Realtime 구독**:
- 학생 탭: 자기 submission row의 `isLocked` 필드 변경 + GradebookEntry `releasedAt` 변경
- 교사 proctor 대시보드: templateId 스코프 submissions + ProctorEvent
- 학부모: GradebookEntry `releasedAt` IS NOT NULL 이벤트 필터

**PGMQ `grading_retry` 큐**: exponential backoff 2·4·8·16·32s, 5회 실패 → `retry_exhausted` 배지.

---

## 6. Tier 정책

| 축 | Free | Pro |
|---|---|---|
| AssessmentTemplate 생성 | ❌ (tierGate true 시) | ✅ |
| 응시 (학생) | tierGate off 기간 전체 공개 | tierGate on 시 Pro만 |
| 개발·베타 기간 | `FeatureFlag.assessmentTierGate = false` → **전 사용자 공개** | 동일 |
| 런칭 | tierGate `true` 전환 → Pro만 접근 | ✅ |
| 감사 로그 보관 | 1학기 | 학년 + CSV/PDF export |

**이중 방어**: 발급 시점(create API) + 수신 시점(take API) 모두 `user.tier === "pro" OR FeatureFlag.assessmentTierGate === false` 검증. canva-publisher-receiver R7(Free 강등 우회) 패턴 승계.

---

## 7. 리스크 표

| # | 리스크 | 영향 | 완화 |
|---|---|---|---|
| R1 | MCQ LLM 우발 호출 (비용 폭주) | 고 | create API Zod gate + 채점 라우터에 kind=MCQ 분기 가드 + Promptfoo 회귀에서 MCQ 경로 LLM 호출 0건 단언 |
| R2 | SHORT LLM 오채점 이의제기 | 고 | `autoRawResponse` 감사 보존, 교사 수동 보정(`manualScore`) 경로, KR-SBERT 전처리로 극단 답안 스킵 |
| R3 | 화면이탈 잠금 새로고침 우회 | 고 | `AssessmentSubmission.isLocked` Supabase 영속화 + mount 시 자동 복원 + 네트워크 단절 안전 측 락 |
| R4 | 드래프트 유실 (태블릿 배터리 부족·네트워크 단절) | 중 | IndexedDB + Supabase autosave 이중화, `localDraftHash` 무결성, 재접속 diff 머지 |
| R5 | 태블릿 응시 화면 렉 | 고 | iframe 0 + 문항 ±1 lazy + OCR 클라우드 오프로드 + 고정 캔버스 800×400 |
| R6 | PGMQ 장애로 채점 큐 적체 | 중 | 5회 backoff + `retry_exhausted` 배지 + 교사 수동 재채점 경로 + healthz 모니터 |
| R7 | Free 강등 우회(tier gate) | 중 | 발급+수신 2중 tier 재검증 |
| R8 | SHORT LLM 비용 폭주 | 중 | KR-SBERT 코사인 < 0.25 또는 10자 미만 자동 0점(LLM 스킵) + Pro tier 월 채점 상한 예측 대시보드(`/admin/llm-cost`) |
| R9 | unlock API 타 학생 우회 | 고 | RLS: teacher=Classroom owner OR student=본인만. 타 학생 unlock → 403 |
| R10 | 학부모 타이밍 오해(미릴리스 표시) | 저 | releasedAt IS NULL 응답에서 GradebookEntry 완전 비포함. 학부모 UI에 "아직 공개되지 않음" 표기 |
| R11 | 부정행위 오탐(false positive) → 학부모 민원 | 고 | 자동 제출·자동 처벌 없음. ProctorEvent는 알림만. 교사 개별 구제 경로 |
| R12 | 감사 로그 개인정보 누적 | 중 | Free=1학기/Pro=학년 보관 후 Supabase cron 자동 파기 |

---

## 8. 파킹 항목 요약

| 항목 | 파킹 사유 | 재등장 시점 |
|---|---|---|
| ESSAY UI 활성 | OCR 판독 불가·LLM 오채점 파일럿 필요 | v1.5 또는 v2 베타 플래그 |
| 자동 잠금·자동 제출(정책 A·C) | 오탐 리스크 큼 | v1.5 요청 시 C(임계값 복합) |
| 부정행위 임계값 교사 조정 UI | v1 고정값 단순화 | v1.5 |
| Knox Kiosk 실 MDM 연동 | BYOD 기본 | v2+ Enterprise |
| Capacitor 네이티브 셸 + ML Kit Digital Ink | 순수 웹 유지 | v2 |
| OneRoster 1.2 Gradebook Service | 외부 SIS 연동 | v2+ |
| LTI 1.3 AGS | 외부 LMS 연동 시 | — |
| 쌤기부-style 생기부 문장 AI 생성 | 수행평가 → 생기부 초안 파이프라인 | v1.5 별 task |
| 카메라 기반 AI 원격 감독(F7) | 초중등 프라이버시·학부모 민원 | 파킹 (ideas-parking-lot) |
| 키스트로크 biometrics(F9) | 초등 표본 부족 | v3+ 고부담 평가 |
| Upstage/CLOVA OCR fallback | Gemini Vision 충분 | LLM 비용 폭주 시 |
| 이메일·문자 학부모 알림 | PWA 푸시 우선 | v1.5 |
| 교사 현장 감독용 모바일 대시보드 | owner+데스크톱 전용 원칙 | 별 task 필요 |

---

## 9. v1 타임라인 (8~10주)

| 주차 | 마일스톤 |
|---|---|
| W1 | Prisma 마이그레이션(7 엔티티 + FeatureFlag) + RLS 3분화 정책 |
| W2 | 교사 출제 UI (AssessmentTemplate·AssessmentQuestion MCQ·SHORT 4필드) + Zod gate |
| W3 | 학생 응시 UI (lazy 마운트 + IndexedDB/Supabase autosave + S-Pen 800×400) |
| W4 | MCQ 결정론 매칭 + SHORT Gemini 2.5 Flash 프로바이더(추상화 유지) + KR-SBERT 전처리 |
| W5 | PGMQ grading_retry 큐 + exponential backoff + retry_exhausted 배지 |
| W6 | L1 자동 화면이탈 잠금 (isLocked/lockedReason 영속, lock/unlock API, Realtime 동기화, 타이머 계속 흐름) |
| W7 | 교사 매트릭스 뷰 + react-virtual(30×30↑) + 수동 보정 + GradebookEntry 확정·릴리스 버튼 |
| W8 | 학생 뷰 `/aura-web/gradebook` (releasedAt 노출 게이트) + 학부모 연계 준비 |
| W9 (추가 0.5~1주) | Promptfoo 회귀 + `/admin/llm-cost` 대시보드 + L1~L3 보안 E2E |
| W10 | 동의서 수령 플로우 + 감사 로그 보관 정책 + 베타 배포 (tierGate false 유지) |

**Scope 축소(MCQ+SHORT)**로 하한 8주 달성 가능. 잠금 API 0.5주 + Promptfoo 0.5주 고려해 8~10주 범위.

---

## 10. 수용 기준 (seed.yaml acceptance_criteria 전체 반영)

- [ ] AssessmentTemplate 생성 UI에서 MCQ·SHORT 2종만 문항 유형 드롭다운 노출 (OX·NUMERIC·ESSAY 차단, Zod gate)
- [ ] MCQ 채점 시 `correctChoiceIds` ↔ `selectedChoiceIds` 서버 결정론적 매칭, **LLM 호출 없음**
- [ ] SHORT 채점 시 Gemini 2.5 Flash 호출, payload(modelAnswers+keywords+partialCredit) + rubric 전체 주입, 부분점수 산출
- [ ] `partialCredit=false` 시 이진(전부맞음/전부틀림) 채점
- [ ] 화면이탈(visibility/fullscreen/focus) 감지 시 즉시 `isLocked=true` + 교사 대시보드 배지 Realtime 푸시
- [ ] `isLocked` 상태가 `AssessmentSubmission` DB에 영속화되어 새로고침 후에도 잠금 UI 복원
- [ ] 학생 "돌아왔습니다" 모달 → unlock API → 잠금 해제
- [ ] 교사 "재개 승인" → unlock API + Realtime broadcast → 학생 UI 즉시 해제
- [ ] 잠금 중 타이머 계속 진행, UI dim + "잠금 중 — 시간은 계속 흐릅니다" 텍스트
- [ ] `ProctorEvent` 전수 기록 (type, durationMs 포함)
- [ ] `FeatureFlag.assessmentTierGate=false` 시 전 사용자 접근, `true` 시 Pro만
- [ ] 교사 릴리스 클릭 전까지 학생·학부모 GradebookEntry 비노출
- [ ] 채점 실패 5회 시 `retry_exhausted` 배지 표시 + 수동 재채점 경로
- [ ] SHORT 교사 UI 4필드 폼(모범답안/키워드/부분점수/루브릭)
- [ ] RLS: teacher=Classroom owner, student=자기 visibleToStudent, parent=ParentChildLink active + releasedAt IS NOT NULL
- [ ] unlock API RLS: teacher 또는 본인 student만 호출 가능
- [ ] 감사 로그 보관 Free=1학기, Pro=학년
- [ ] 동의서 미제출 학생 손글씨 입력 비활성화
- [ ] 태블릿 응시 화면 iframe 0 (DOM snapshot 검증)
- [ ] S-Pen 캔버스 800×400 고정 픽셀, 60fps
- [ ] OCR 클라우드 오프로드 전용 (Tesseract.js import 0)

---

## 11. 작업 분할 (AA-1 ~ AA-10)

| 단계 | 내용 |
|---|---|
| AA-1 | Prisma 마이그레이션 — 7 신규 엔티티 + Classroom·FeatureFlag 확장 + RLS 3분화 |
| AA-2 | AssessmentTemplate·Question create API + Zod gate(MCQ·SHORT만) + AI 문항 생성 보조(Gemini Flash 초안) |
| AA-3 | 학생 응시 UI — lazy 문항 마운트 ±1 + IndexedDB/Supabase autosave 이중화 + S-Pen 고정 캔버스 |
| AA-4 | 자동채점 — MCQ 결정론 매칭 + SHORT Gemini 2.5 Flash 프로바이더(추상화) + KR-SBERT 전처리 |
| AA-5 | PGMQ `grading_retry` 큐 + exponential backoff + retry_exhausted 배지 + healthz |
| AA-6 | L1 자동 화면이탈 잠금 — `isLocked/lockedReason` 영속, `/api/assessment/[submissionId]/lock`·unlock?by=student|teacher, Realtime 동기화, mount 시 자동 복원 |
| AA-7 | 교사 매트릭스 뷰(owner+데스크톱) + react-virtual + autoScore 열람 + manualScore 수동 보정 + GradebookEntry 확정 |
| AA-8 | 성적 릴리스 — `releasedAt` SET + Supabase Realtime broadcast → 학생·학부모 뷰 `p95 < 500ms` 표시 |
| AA-9 | 동의서 수령 플로우(LLM 국외 이전·손글씨·성적 열람 3건 묶음) + 감사 로그 보관 cron |
| AA-10 | Promptfoo SHORT 채점 회귀 + `/admin/llm-cost` 비용 대시보드 + L1~L3 E2E 보안 게이트 |

---

## 12. 새로 드러난 분기 (integrate 기록용)

### 12.1 학부모 열람 경로 중복 (분기 3.1)

- parent-viewer v2(seed_6d7077aac472)는 이미 Aura 웹앱 내 "자녀 단일 뷰"를 소유.
- 본 로드맵의 GradebookEntry 학부모 뷰를 **parent-viewer v2 안의 "성적" 탭** 편입할지, 별도 route 신설할지 미결.
- **결정권: v1.5 integrate 시점**. 현 v1은 "parent-viewer v2와 동일 Aura 웹앱·동일 RLS·동일 PWA 셸"로만 합의.
- `parent-viewer-roadmap.md` 하단에 "수행평가 성적 탭 편입 미결" 메모 추가됨.

### 12.2 생기부 문장 자동 생성 후속

- 쌤기부 벤치마크(E8). 수행평가 → 교사 확정 → LLM 문장 초안 → NEIS export. 별 task로 분리. 현 스코프 이탈.

### 12.3 Pro tier 학급 인원 과금 모델

- U1 "Pro 전용" 잠정 확정 시 학급 인원·LLM 채점 월 상한 구조가 Free/Pro 단순 이분법으로 표현 어려움. Team/School tier 신설 여부 별 task 후보.

---

## 변경 로그

| 날짜 | 시드 ID | 변경 |
|---|---|---|
| 2026-04-16 | `seed_0badf1e571bc` | 신규 로드맵 생성. 5 primitive 고정(출제·응시·자동채점·교사확정·성적송신). Prisma 7 신규 엔티티 + Classroom·FeatureFlag 확장. MCQ 결정론 + SHORT Gemini 2.5 Flash. L1 자동 화면이탈 잠금(isLocked/lockedReason 영속, 해제 2경로, 타이머 계속 흐름). Pro 전용 + `FeatureFlag.assessmentTierGate` 런칭 플래그. teacher_manual 릴리스 정책. D1 공통 Supabase + RLS 3분화 + Realtime p95<500ms. 타임라인 8~10주. AA-1~AA-10 작업 분할. interview_20260415_224854 기반, ambiguity 0.10. |
