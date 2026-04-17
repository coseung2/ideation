# Phase 5 — 갱신된 plan 목록

> task_id: `2026-04-16-performance-assessment-autograde`
> seed: `seed_0badf1e571bc` (ambiguity 0.10, interview_20260415_224854)
> 작성: 2026-04-16 phase5 integrator
> 원칙: 기존 plan은 **추가만**, 기존 본문 수정 금지. 변경 로그·관련 주제·최신 제약·메모 섹션을 하단에 추가하는 방식.

---

## 갱신 plan 목록 (총 6개)

### 1. `plans/seeds-index.md`

- 헤더 카운트: "전체 Seed 12개 (Aura-board 9개 + …)" → **"전체 Seed 13개 (Aura-board 10개 + refinement 1개 + 외부 프로젝트 2개)"**로 증가 반영.
- 표에 행 추가: `| 12 | 수행평가 자동채점 파이프라인 (assessment-autograde) | seed_0badf1e571bc | interview_20260415_224854 | 0.10 | assessment-autograde-roadmap.md (신규) | padlet |`.
- 상세 섹션 "🎯 Seed 12" 추가: 19행 결정표(Board 확장·7 신규 엔티티·MCQ 결정론·SHORT Gemini Flash·D1 인프라·L1 자동 잠금·`isLocked` 영속·teacher_manual 릴리스·Pro+tierGate 플래그·UI 2곳 운영·태블릿 응시 예산·AI 제안 비공개·동의서·감사 로그·재응시 락·SW·Knox 플래그만·타임라인 8~10주) + 새로 드러난 분기 3.1~3.5.
- 인접 시드 의존성 그래프에 Seed 12 화살표 7개 추가 (assignment-board 상위 확장 / Supabase 공유 / 릴리스 이중 조건 연계 / tierGate 플래그 패턴 / tablet-performance §2b / matrix_desktop_only 승계).
- 공통 성능 예산 주석에 "Seed 12 응시 화면은 iframe 0" 단서 추가.

### 2. `plans/assignment-board-roadmap.md`

- 하단에 **"관련 주제"** 섹션 신설: assessment-autograde-roadmap.md 교차링크 + "**채점·성적 송신 레이어는 본 assignment-board가 아닌 assessment-autograde가 담당한다**" 한 줄 명시. assignment-board는 제출·반려·Roster 배부에 집중, 수행평가식 자동채점·MCQ/SHORT LLM·GradebookEntry·릴리스는 assessment-autograde의 범위로 위임.
- 기존 본문(§0~§12·변경 로그) 수정 없음.

### 3. `plans/parent-viewer-roadmap.md`

- §11 변경 로그에 2026-04-16 행 추가: "수행평가 성적 탭 편입 미결 메모 추가(§관련 주제). §5 매트릭스·§7 PV-* 작업 카드에 영향 없음. v1은 assessment-autograde 로드맵 안의 별도 뷰로 진행, 탭 편입은 v1.5 integrate 시점에 결정."
- 문서 최하단에 **"관련 주제"** 섹션 신설: assessment-autograde-roadmap.md 교차링크 + **수행평가 성적 탭 편입 미결 (분기 3.1)** 메모. "parent-viewer v2 자녀 단일 뷰에 성적 탭 편입 vs 별도 route 신설" 두 옵션 병기, **결정권은 v1.5 integrate 시점**. v1 합의는 "동일 Aura 웹앱·동일 RLS·동일 PWA 셸"까지. 학부모 RLS는 `ParentChildLink.status='active'` + `GradebookEntry.releasedAt IS NOT NULL` 이중 조건으로 §5 매트릭스 원칙 승계.
- 기존 §0~§10 본문 수정 없음.

### 4. `plans/tablet-performance-roadmap.md`

- 변경 로그에 2026-04-16 행 추가 (seed_0badf1e571bc).
- **§2b 최신 제약 섹션 신설** (§2a 패턴 따름): "수행평가 응시 화면(assessment-autograde Seed 12)" 표제 + iframe **0** 강화 + OCR **클라우드 오프로드 전용**(Tesseract.js 금지) + S-Pen 캔버스 **800×400 고정 px** 60fps + 문항 ±1 lazy + 드래프트 이중화 + Realtime 구독 학생=자기 submission만.
- 기존 §2 일반 예산, §2a 격자 예산, §3 이하 본문 수정 없음.

### 5. `plans/phase0-requests.md`

- "## 수행평가 자동채점 파이프라인 (assessment-autograde-roadmap.md, Seed 12)" 섹션을 "과제 배부 보드" 다음 / "mallang-ranch-p2e" 이전에 신설.
- 블록 2종 수록:
  - **AA-1** Prisma 마이그레이션 (7 신규 엔티티 + Classroom·FeatureFlag 확장 + RLS 3분화) — 10개 acceptance.
  - **AA-2~AA-10** 통합 assessment-autograde feature 본체 — 27개 acceptance (U1~U5 전부 커버: tierGate 플래그 / MCQ LLM 호출 0 / SHORT Gemini Flash 프롬프트 구조 / partialCredit 이진 / 자동 화면이탈 잠금 + isLocked 영속 + 새로고침 복원 + 해제 2경로 + teacher_resume ProctorEvent + 시간 계속 흐름 + MCQ·SHORT 2종 드롭다운). `context_refs`에 assessment-autograde-roadmap.md · tablet-performance §2b · assignment-board-roadmap.md · parent-viewer-roadmap.md · phase1~4 산출물 포함. destination padlet 명시. slug `performance-assessment-autograde`.
- 기존 다른 블록·사용 지침 §1~5 수정 없음.

### 6. `ideas-parking-lot.md`

- "수행평가 자동채점 파이프라인 (assessment-autograde, Seed 12) — v1.5/v2/v3+ 파킹" 섹션을 최하단 placeholder 주석 위에 신설. decisions.md §2 파킹 목록 정제하여 11건 수록:
  - ESSAY UI (v1.5 베타 플래그)
  - 자동 잠금·자동 제출 A·C (v1.5 요청 시)
  - 부정행위 임계값 교사 조정 UI (v1.5)
  - Knox Kiosk 실 MDM 연동 (v2+ Enterprise)
  - Capacitor 네이티브 셸 + ML Kit Digital Ink (v2)
  - OneRoster 1.2 Gradebook · LTI 1.3 AGS (v2+ 외부 연동)
  - 쌤기부-style 생기부 문장 AI 생성 (v1.5 별 task)
  - 카메라 기반 AI 원격 감독 (파킹, 우선순위 낮음)
  - 키스트로크 biometrics (v3+ 중·고등)
  - 교사 현장 감독용 모바일 대시보드 (별 task)
  - 이메일·문자 학부모 알림 (v1.5)
  - Upstage/CLOVA OCR fallback (LLM 비용 폭주 시)

---

## 검증

- [x] seed.yaml `ontology_schema` 필드 전부 신규 roadmap §1에 반영 (AssessmentTemplate·Question·Submission·Answer·GradebookEntry·ProctorEvent·Classroom.gradebookReleasePolicy·FeatureFlag.assessmentTierGate + 신규 `isLocked/lockedReason` + `ProctorEventType.teacher_resume`)
- [x] seed.yaml `acceptance_criteria` 전부 신규 roadmap §10 + phase0-requests AA-2~AA-10 acceptance에 반영 (특히 U1·U2·U3·U5 강조)
- [x] seed.yaml `constraints` 전부 반영 (Supabase 공유·RLS 3분화·MCQ+SHORT 2종·matrix owner+데스크톱·MCQ 결정론·SHORT Gemini Flash·isLocked 영속·시간 계속 흐름·자동 제출 없음·tierGate·teacher_manual·IndexedDB+Supabase 이중화·PGMQ backoff·동의서·Realtime p95<500ms·갤탭 S6 Lite·OCR 클라우드 오프로드·타임라인 8~10주)
- [x] 기존 plan 본문 훼손 없음 (추가만, Markdown 상대경로 `./`)
- [x] 변경 로그·날짜·seed_id 신규 plan 하단 명시
- [x] seeds-index 시드 7 supersede 전례 따라 Seed 12 행 형식 유지
- [x] phase0-requests 블록에 context_refs 포함 (assessment-autograde·assignment-board·parent-viewer 요건 충족)
- [x] parent-viewer 하단 분기 3.1 메모 + 결정권 v1.5 명시
- [x] tablet-performance 최신 제약 섹션 신설(§2b)
- [x] ideas-parking-lot decisions §2 파킹 11건 이관
