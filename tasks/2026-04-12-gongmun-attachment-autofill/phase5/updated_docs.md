# Phase 5 — Updated Plan Documents

- **task_id**: `2026-04-12-gongmun-attachment-autofill`
- **seed_id**: `seed_3f953e443fa1`
- **updated_at**: 2026-04-12

## 갱신

### 1. `ideation/plans/seeds-index.md`

**변경 1 — 전체 Seed 표 갱신**:
- 헤더 "전체 Seed 8개" → "전체 Seed 9개 (Aura-board 8개 + 외부 프로젝트 1개)"
- 표에 `Destination` 컬럼 신규 추가
- Seed 1~8 `Destination`을 `padlet` / `padlet + aura-canva-app`으로 표기
- **Seed 9 행 신규 추가**:
  - 주제: 공문 붙임파일 자동 생성 CLI (gongmun-assistant v1)
  - Seed ID: `seed_3f953e443fa1`
  - Interview ID: `interview_20260412_131559`
  - Ambiguity: 0.177
  - 연관 plan: `gongmun-assistant-v1-roadmap.md` (신규)
  - Destination: **gongmun-assistant (외부 신규 프로젝트)**

**변경 2 — Seed 9 핵심 결정 표 블록 신규 삽입** (Seed 8 블록 바로 뒤):
- 프로젝트 경계 문단 (Aura-board와 완전 독립 명시)
- 핵심 결정 요약 표 (D1~D8에 해당하는 13행): 실행 모드·입력 경로·템플릿 관리·슬롯 규약·LLM provider·개인정보 경계·CSV 자동 삭제·config 저장소·배포·v1 범위·v2+ 파킹·기반 스킬·종료 코드·리스크 대응
- 8단계 파이프라인 한 줄 요약
- 작업 GM-1~GM-10 참조

**변경 3 — 인접 시드 의존성 다이어그램 갱신**:
- 기존 Seed 1~8 의존성 그래프 하단에 박스로 Seed 9 완전 독립 구역 추가
- "ideation/ 와 코드·DB·인증·배포 채널 전부 분리 / Seed 1~8 어느 것과도 의존성 화살표 없음 / destination: gongmun-assistant / 기반 스킬: hwpx-master / 공통 성능 예산·Aura-board Tier·RBAC·RLS 정책 모두 미적용" 5항목 명시

---

### 2. `ideation/plans/phase0-requests.md`

**변경 — 파일 말미 "사용 지침" 앞에 gongmun-assistant 섹션 신규 삽입**:
- 상단 주의: "외부 프로젝트 주의 — 이 블록들은 Aura-board/padlet과 무관. `gongmun-assistant/tasks/...`에 복사. `destination: gongmun-assistant`"
- **GM-1 ~ GM-10 총 10개 JSON 블록** 추가:
  - GM-1: 프로젝트 스캐폴드 + 레이어 경계 정적 검사
  - GM-2: hwpx_io (python-hwpx 래퍼 + linesegarray 제거)
  - GM-3: llm provider ABC + Anthropic + Ollama + privacy 스캐너
  - GM-4: 템플릿 catalog + 시스템 번들 6개 + 사이드카 YAML
  - GM-5: core.table_fill 결정론 + LLM 호출 0회 정적 격리
  - GM-6: core.pipeline Stage 1-8 + narrative_fill
  - GM-7: config 스키마 + 첫 실행 셋업 마법사
  - GM-8: CLI typer + 대화형 + gongmun.bat
  - GM-9: validate.py + E2E + 한컴 수동 QA
  - GM-10: PyPI 배포 + 설치·템플릿 제작 가이드

**사용 지침 갱신**:
- 항목 1에 대상 프로젝트별 경로 분기 명시 (padlet/tasks vs gongmun-assistant/tasks)
- 항목 3에 `context_refs` (복수형) 케이스 추가
- 항목 6 신규: `destination` 필드 있는 블록은 Aura-board 외부이므로 padlet 저장소에 올리지 않도록 주의

---

## 상충·승격 검토

- **이전 Seed와의 상충 없음** — Seed 9는 Aura-board 외부 완전 독립 프로젝트.
- **superseded 관계 없음** — 기존 Seed 1~8 문서 불변.
- **정책 승계 없음** — Seed 2 Tier·Seed 7 Parent 매트릭스·공통 성능 예산 모두 미적용 (다이어그램 박스에 명시).

## 링크 검증

- `gongmun-assistant-v1-roadmap.md` → `phase2/sketch.md`, `phase3/decisions.md`, `phase4/seed.yaml` 링크 유효 (상대 경로 `ideation/tasks/.../`).
- `seeds-index.md` → `gongmun-assistant-v1-roadmap.md` 링크 신규 추가.
- `phase0-requests.md`의 GM-* 블록 context_refs → roadmap §·phase 문서 섹션 앵커 유효.
- 기반 스킬 참조 `gongmun-assistant/skills/hwpx-master.SKILL.md`은 외부 프로젝트 내부 경로이므로 ideation 외부 링크로 표기.
