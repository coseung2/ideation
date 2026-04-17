# Phase 5 — 신규 plan 목록

> task_id: `2026-04-16-performance-assessment-autograde`
> seed: `seed_0badf1e571bc` (ambiguity 0.10, interview_20260415_224854)
> 작성: 2026-04-16 phase5 integrator

---

## 신규 plan

### `plans/assessment-autograde-roadmap.md` (신규 1개)

수행평가 자동채점 파이프라인 v1 로드맵. seed.yaml + decisions.md를 살아있는 설계 문서로 번역.

**수록 내용**:
- 메타: 작성일 2026-04-16, seed `seed_0badf1e571bc`, interview `interview_20260415_224854`, 관련 plan 5개(assignment-board·parent-viewer·canva-publisher-receiver·tablet-performance·seeds-index) 교차링크.
- 핵심 명제: 5 primitive 고정 (교사 출제·학생 응시·자동채점·교사 확정·성적 송신).
- §1 Prisma 최종 확정 스키마: `Board.layout="assessment"` + 7 신규 엔티티(AssessmentTemplate·AssessmentQuestion·AssessmentSubmission·AssessmentAnswer·GradebookEntry·ProctorEvent·FeatureFlag) + Classroom.gradebookReleasePolicy·schoolManagedDevices 확장. seed.yaml `ontology_schema` 필드 전부 반영. `isLocked/lockedReason` 영속 컬럼 포함.
- §2 사용자 흐름 6개 섹션: 교사 출제 / 학생 응시 / 자동채점 / 교사 확정 / 학부모 열람(parent-viewer 승계) / 부정행위 감독 L1 강화.
- §3 태블릿 성능 예산 체크리스트: iframe 0 + 문항 ±1 lazy + OCR 클라우드 오프로드 전용 + S-Pen 800×400 고정 px.
- §4 부정행위 방지 L1~L4 레이어: L1 자동 화면이탈 잠금(U3 확정), L2 교사 대시보드, L3 재응시 락·SW 방어, L4 Knox Kiosk 플래그만.
- §5 송신 채널 D1: 공통 Supabase + RLS 3분화 + Realtime p95<500ms + PGMQ `grading_retry` exponential backoff.
- §6 Tier 정책: Pro 전용 + `FeatureFlag.assessmentTierGate` 런칭 플래그(개발·베타 false → 런칭 true).
- §7 리스크 표 12건(R1~R12).
- §8 파킹 항목 요약 (ESSAY UI·자동 제출 A·C·Knox 실 MDM·Capacitor·OneRoster·LTI·쌤기부·카메라 감독·키스트로크 등).
- §9 v1 타임라인 8~10주(W1~W10 마일스톤).
- §10 수용 기준: seed.yaml acceptance_criteria 전체 반영(특히 U1~U5 강조).
- §11 작업 분할 AA-1 ~ AA-10.
- §12 새로 드러난 분기(12.1 학부모 탭 편입 v1.5 미결 / 12.2 생기부 문장 / 12.3 Pro tier 학급 과금).
- 변경 로그 2026-04-16 (seed_0badf1e571bc).
