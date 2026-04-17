# Ideation Pipeline — Index

새 아이디어·기능·통합 방향 모색 → 시드 생성 → padlet 인수인계 절차.

## Phase 순서

| # | Phase | 사양 파일 | 에이전트 계약서 | 사용 스킬 |
|---|---|---|---|---|
| 0 | 아이디어 캡처 | `phase0_capture.md` | `agents/capture-analyst.md` | — |
| 1 | 탐색·리서치 | `phase1_explore.md` | `agents/explorer.md` | `WebSearch`, `WebFetch` |
| ⚙ | **탐색 검증** | | | |
| 2 | 스케치 | `phase2_sketch.md` | `agents/sketch-architect.md` | — |
| ⚙ | **스케치 검증** | | | |
| 3 | 인터뷰 | `phase3_interview.md` | `agents/interview-facilitator.md` | `/ouroboros:interview` |
| 4 | 시드 | `phase4_seed.md` | `agents/seed-generator.md` | `/ouroboros:seed` |
| ⚙ | **시드 검증** (ambiguity ≤ 0.2) | | | |
| 5 | 문서 통합 | `phase5_integrate.md` | `agents/integrator.md` | — |
| 6 | 인수인계 | `phase6_handoff.md` | `agents/handoff-writer.md` | — |
| ⚙ | **인수인계 검증** | | | |
| 7 | 배송 | `phase7_dispatch.md` | `agents/dispatcher.md` | — |
| ⚙ | **배송 검증** (MANIFEST + 필수 파일) | | | |

**오케스트레이터 역할**: 각 phase는 해당 에이전트 계약서(agents/*.md 전문)를 `Agent` 도구에 prompt로 전달하여 호출. 호출 프로토콜은 `CLAUDE.md` 참조.

**Destination 라우팅**: phase7 dispatcher가 `destinations/_registry.md` 규칙에 따라 padlet INBOX / parking / archive / research-vault 중 배송지 결정.

## task 디렉토리

```
tasks/{YYYY-MM-DD-slug}/
├── phase0/request.json           # topic, motivation, audience
├── phase1/exploration.md         # 비교·벤치마크·레퍼런스
├── phase2/sketch.md              # 데이터 모델 초안·흐름·리스크·미결
├── phase3/
│   ├── session_id.txt            # Ouroboros interview session ID
│   └── decisions.md              # 인터뷰 답변 요약
├── phase4/
│   ├── seed_id.txt               # seed ID
│   └── seed.yaml                 # 시드 복사본 (백업·감사용)
├── phase5/
│   ├── updated_docs.md           # 갱신한 plans/ 문서 목록
│   └── new_docs.md               # 신규 생성한 문서 목록
└── phase6/
    ├── padlet_phase0_request.json  # padlet 파이프라인 진입 템플릿
    └── handoff_note.md             # 인수 에이전트에게 줄 프롬프트
```

## 스킵 규칙

- `scope == "parking"` → phase2~6 스킵 후 `ideas-parking-lot.md` 섹션 추가로 종결
- `scope == "research_only"` → phase3~6 스킵 후 `research/`에 보고서 저장
- `scope == "quick_decision"` (미결 1~2개만) → phase1·2 간소화 가능, phase3는 필수

스킵 사용 시 `tasks/{task_id}/SKIP_{PHASE}.md`로 사유 기록.

## 검증 게이트 (자동, 매 phase 후)

1. **탐색 검증** (phase1 후): `exploration.md`에 비교 대상 ≥ 3개 + 각 장단점 기술
2. **스케치 검증** (phase2 후): 데이터 모델 초안 + 사용자 흐름 + 미결 질문 ≥ 1개
3. **시드 검증** (phase4 후): Ouroboros ambiguity ≤ 0.2
4. **인수인계 검증** (phase6 후): `padlet_phase0_request.json` + `handoff_note.md` 동시 존재

실패 시 해당 phase 재실행. 3회 연속 실패 시 사용자에게 보고.

## 핸드오프 원칙

- 각 phase는 바로 앞 phase의 산출물만 입력
- 다운스트림이 업스트림 누락을 추정으로 보정 금지 — 해당 phase 재실행
- 살아있는 문서(`plans/`, `research/`, `data/`) 갱신은 **phase5에서만** 수행
- padlet 쪽 쓰기는 **하지 않음** (하네스 전반 원칙)

## 자율 진행 전제

- phase3 인터뷰 답변은 에이전트가 직접(배경·기존 결정·상식 기반)
- 사용자 확정이 필요한 **수치·핵심 방향**만 AskUserQuestion (가격·tier 한도·사용자 경험 규범 등)
- 중대한 재방향(시드 폐기·큰 스코프 변경) 전 사용자 확인

## 결정 기록 원칙

- 인터뷰 중 나온 결정은 모두 `phase3/decisions.md`에 남겨 이후 phase에서 재인용
- phase5 통합 시 `plans/`에 추가된 결정은 `seeds-index.md`의 해당 seed 섹션에도 반영
