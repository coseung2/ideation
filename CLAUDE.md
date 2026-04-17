# Ideation — 아이디어 정교화 하네스

이 폴더는 **Aura-board**(상위 `padlet/`)의 아이디어·설계 문서를 정교화하는 브레인스토밍 공간이다. 구현은 여기서 하지 않고, 결정만 내린 뒤 `padlet/` 피처 파이프라인으로 인수인계한다.

> 본 하네스는 padlet 하네스(`padlet/CLAUDE.md`)를 레퍼런스로 구조화됨. 단 톤은 정식 스펙 산출이 아닌 **아이디어 정교화** 수준으로 유지.

## 원칙

1. **실구현은 여기서 하지 않는다** — `padlet/`는 읽기 전용 참조만.
2. **자율 진행 기본** — 중대한 방향 전환만 사용자 확인, 나머지는 에이전트 판단.
3. **결정물은 시드** — Ouroboros `ooo interview` → `ooo seed`가 의사결정 종결 수단.
4. **살아있는 문서는 `plans/`** — 시드와 별개로 로드맵·카탈로그·레퍼런스 유지보수.
5. **인수인계는 phase0 request.json** — padlet 파이프라인 진입 포맷 준수.
6. **오케스트레이터 ≠ 실행자** — 메인 세션은 오케스트레이션만, 각 phase는 전문 에이전트 계약서 기반으로 `Agent` 도구로 서브에이전트 호출.

## 오케스트레이터 프로토콜

이 대화 세션(메인 Claude Code)은 **오케스트레이터**다. 직접 리서치·쓰기·편집을 수행하지 않고, 각 phase를 전담하는 **서브에이전트**(`agents/*.md` 계약서 기반)를 `Agent` 도구로 호출한다.

### 에이전트 카탈로그

`agents/_registry.md` 참조. phase ↔ 에이전트 매핑:

| 파이프라인 | Phase | 에이전트 |
|---|---|---|
| ideation | 0 capture | capture-analyst |
| ideation | 1 explore | explorer |
| ideation | 2 sketch | sketch-architect |
| ideation | 3 interview | interview-facilitator |
| ideation | 4 seed | seed-generator |
| ideation | 5 integrate | integrator |
| ideation | 6 handoff | handoff-writer |
| ideation | **7 dispatch** | **dispatcher** |
| refinement | 0 target | target-identifier |
| refinement | 1 audit | auditor |
| refinement | 2 delta | delta-analyst |
| refinement | 3~7 | (ideation의 해당 에이전트 재사용) |

### Destination 라우팅 (v2)

phase7 dispatcher가 아래 목적지 중 하나로 배송:

**외부 프로젝트 INBOX** (`../{project}/INBOX/`):
- **padlet** — Aura-board 교실 학습 플랫폼
- **aura** — AURA 대시보드
- **class** — 교사 수업 기획·콘텐츠 제작
- **library** — Reading Creation Platform
- **trading-journal** — 암호화폐 매매일지
- **퀀트기획서** — 바이비트 자동매매 봇
- **free** — 공모전 출품작 묶음 (target_subproject 필수)
- **note** — 교과서 → 학생 공책정리 PDF 자동생성 (자체 8-phase 파이프라인)

**ideation 내부** (`destinations/`):
- **parking** — `scope == "parking"`
- **archive** — 종결·폐기·수퍼시드
- **research-vault** — `scope == "research_only"`
- **internal** (fallback) — 외부·내부 라우팅 모두 실패 시 로컬 보관. 나중에 재라우팅 가능

라우팅 규칙 전체는 `destinations/_registry.md` 참조.

### 외부 프로젝트 INBOX 쓰기 예외

원칙은 **ideation 외부 프로젝트 읽기 전용**이지만, dispatcher 에이전트는 **INBOX/ 경로에 한해 쓰기 예외 권한**을 가진다. 각 INBOX에는 `README.md`가 있어 출처·소비 방법·삭제 권한을 자립적으로 설명한다.

### 호출 포맷

```
Agent({
  description: "Phase {N} — {agent-name}",
  subagent_type: "general-purpose",
  prompt: "<agents/{agent-name}.md 전문>\n\n---\n런타임 입력:\n- task_id: ...\n- 입력 파일: ...\n- 추가 맥락: ..."
})
```

### 오케스트레이터가 직접 하는 일 (서브에이전트에게 위임하지 않음)

- task 폴더 생성 (`tasks/{YYYY-MM-DD-slug}/`)
- 검증 게이트 판정 (산출 파일 존재·필드 완비·ambiguity ≤ 0.2)
- phase 간 파일 경로 전달
- 재시도 결정 (실패 시 최대 3회 같은 에이전트 재호출)
- 사용자 커뮤니케이션 (진행 보고·방향 확정 질문)
- 에스컬레이션 처리 (에이전트가 유보한 항목을 사용자에게 질문)

### 오케스트레이터가 하지 않는 일

- 직접 WebSearch·Read·Edit·Write (에이전트 작업을 대신 수행)
- 에이전트 산출물 수정 (재호출로 해결)
- 의사결정 (결정은 에이전트의 자율 답변 또는 사용자 확정)

## 파이프라인 분기

| type | 트리거 | 인덱스 |
|---|---|---|
| `ideation` | 새 아이디어·기능·통합 방향 모색 | `prompts/ideation/_index.md` |
| `refinement` | 기존 시드·설계 재검토(기준 변화·새 제약 반영) | `prompts/refinement/_index.md` |

## 작업 시작 순서

1. 사용자 프롬프트에서 `type` 결정 (모호하면 1회 질문)
2. `tasks/{YYYY-MM-DD-slug}/phase0/request.json`에 `type`과 `topic` 기록
3. 해당 파이프라인의 `_index.md`를 따라 순차 진행
4. 각 phase 산출물은 `tasks/{task_id}/phase{N}/`에 저장
5. 최종 출력은 `plans/`·`research/`·`data/`에 통합 반영 + `seeds-index.md` 갱신

## 디렉토리

```
ideation/
├── CLAUDE.md                    # 이 파일. 오케스트레이션 루트
├── INDEX.md                     # 전체 산출물 인덱스
├── prompts/                     # 파이프라인 phase 사양
│   ├── ideation/
│   │   ├── _index.md
│   │   └── phase0~6_*.md
│   └── refinement/
│       ├── _index.md
│       └── phase0~2_*.md
├── agents/                      # 서브에이전트 계약서
│   ├── _registry.md
│   ├── capture-analyst.md
│   ├── explorer.md
│   ├── sketch-architect.md
│   ├── interview-facilitator.md
│   ├── seed-generator.md
│   ├── integrator.md
│   ├── handoff-writer.md
│   ├── dispatcher.md
│   ├── target-identifier.md
│   ├── auditor.md
│   └── delta-analyst.md
├── destinations/                # 내부 배송지 (외부는 각 프로젝트 폴더의 INBOX/)
│   ├── _registry.md             # 외부+내부 라우팅 레지스트리
│   ├── parking/                 # 나중에 꺼낼
│   ├── archive/                 # 종결·폐기
│   ├── research-vault/          # 완결된 리서치 보고서
│   └── internal/                # Fallback — 라우팅 실패 시 로컬 보관
├── tasks/                       # 작업 단위 산출물 (감사 이력)
│   └── {YYYY-MM-DD-slug}/
├── plans/                       # 살아있는 설계 문서
│   ├── implementation-roadmap.md (Canva 통합)
│   ├── tablet-performance-roadmap.md
│   ├── drawing-board-library-roadmap.md
│   ├── plant-catalog.md
│   ├── plant-journal-roadmap.md
│   ├── event-signup-roadmap.md
│   ├── phase0-requests.md
│   └── seeds-index.md
├── research/
│   └── canva-developer-aura-research.md
├── data/
│   └── plant-species-seed.json
├── ideas-parking-lot.md         # 지금은 안 하지만 나중에 꺼낼 아이디어
└── canva-assignment-pdf-merge/  # 동작 중인 Canva 스킬(과제 PDF 병합)
```

## 검증 게이트 (자동)

| 게이트 | 위치 | 통과 조건 |
|---|---|---|
| **탐색 검증** | ideation phase1 직후 | `exploration.md`에 비교 대상 ≥ 3개 + 각 장단점 기술 |
| **스케치 검증** | ideation phase2 직후 | `sketch.md`에 데이터 모델 초안 + 사용자 흐름 + 미결 질문 ≥ 1개 |
| **시드 검증** | ideation phase4 직후 | Ouroboros ambiguity ≤ 0.2 |
| **인수인계 검증** | ideation phase6 직후 | `padlet_phase0_request.json` + `handoff_note.md` 동시 존재 |

실패 시 해당 phase 재실행. 3회 연속 실패 시 오케스트레이터가 사용자에게 보고.

## 핸드오프 원칙

1. 각 phase는 `tasks/{task_id}/phase{N}/`에 산출물 저장
2. 다음 phase는 직전 phase 산출물만 입력
3. 살아있는 문서(`plans/`, `research/`) 업데이트는 phase5(integrate)에서만 수행
4. padlet으로의 인수인계는 phase6(handoff)에서만 수행

## 자율 진행 방침

- 사용자가 "내가 승인 없이 쭉 진행해"라고 지시한 상태가 기본.
- Ouroboros 인터뷰에서 질문은 **에이전트가 직접 답변** (배경·기존 결정·상식 기반). 사용자 최종 결정이 필요한 수치·방향 전환만 AskUserQuestion.
- 파일 생성·수정(ideation 내부) 자유. padlet은 읽기 전용.
- 중대한 재방향(시드 폐기, 큰 스코프 변경)은 사용자 확인 후 진행.

## 스킵 규칙

- **`scope == "parking"`** (파킹만 해두는 아이디어): phase2~6 스킵, `ideas-parking-lot.md`에 섹션 추가로 종료
- **`scope == "research_only"`** (리서치만): phase3~6 스킵, `research/`에 보고서만 기록
- 스킵 사용 시 `tasks/{task_id}/SKIP_{PHASE}.md`로 사유 기록

## gstack / Ouroboros 스킬 통합

각 phase 파일의 `## 사용 스킬` 섹션에 명시. 주요 스킬:
- `/ouroboros:interview` — phase3
- `/ouroboros:seed` — phase4
- `WebSearch` / `WebFetch` — phase1

## 메모리 참조

사용자 방침·프로젝트 상수는 `/home/coseung2/.claude/projects/-mnt-c-Users-----Desktop-Obsidian-Vault-ideation/memory/MEMORY.md` 에 저장됨. 하네스 실행 시 해당 메모리가 자동 로드되어 태블릿 기준(갤럭시 탭 S6 Lite)·자율 진행·품질 우선 등 방침이 적용된다.
