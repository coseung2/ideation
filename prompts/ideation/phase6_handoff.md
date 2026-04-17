# Phase 6 — Handoff

padlet 파이프라인 진입 자료 생성. 다른 에이전트가 이어받을 수 있도록.

## 입력

- `tasks/{task_id}/phase4/seed.yaml`
- `tasks/{task_id}/phase5/updated_docs.md`·`new_docs.md`

## 출력

- `tasks/{task_id}/phase6/padlet_phase0_request.json`
- `tasks/{task_id}/phase6/handoff_note.md`

## padlet_phase0_request.json 형식

padlet의 `prompts/feature/phase0_analyst.md` 포맷을 따른다.

```json
{
  "type": "feature",
  "slug": "drawing-board-library",
  "task_id": "2026-04-12-drawing-board-library",
  "change_type": "new_feature",
  "motivation": "이비스페인트 → 갤러리 → 패들렛 업로드 반복 마찰 제거",
  "user_story": "학생으로서 그림보드에서 저장한 작품을 다른 보드 카드에 원클릭으로 붙일 수 있다",
  "success_metric": "라이브러리 → 카드 삽입 동작 3초 내 완료, 갤럭시 탭 S6 Lite 기준 TTI < 3s 유지",
  "affected_surfaces": ["/board/:id (drawing layout)", "/board/:id (column/submission)", "/library"],
  "context_refs": {
    "seed": "ideation/tasks/2026-04-12-drawing-board-library/phase4/seed.yaml",
    "plan": "ideation/plans/drawing-board-library-roadmap.md",
    "performance_budget": "ideation/plans/tablet-performance-roadmap.md",
    "decisions": "ideation/tasks/2026-04-12-drawing-board-library/phase3/decisions.md"
  },
  "created_at": "2026-04-12T00:00:00+09:00"
}
```

## handoff_note.md 형식

다른 에이전트(padlet에서 작업할 Claude Code 세션)에게 그대로 던질 수 있는 프롬프트.

```markdown
# Handoff Note — {topic}

## 배경

{motivation + 배경 2~3 문장}

## 참조 문서 필수 독해

아래 순서로 반드시 읽고 작업:
1. `ideation/plans/seeds-index.md` — 전체 결정 맥락
2. `ideation/plans/tablet-performance-roadmap.md` — 강제 성능 예산
3. `ideation/plans/{이번 주제 roadmap}.md` — 세부 결정
4. `ideation/tasks/{task_id}/phase3/decisions.md` — 확정된 미결 답변

## 기준 단말·제약

- 갤럭시 탭 S6 Lite (Chrome Android, S-Pen) — 성능 예산 강제
- padlet은 feature 파이프라인으로 진입 (`padlet/prompts/feature/_index.md`)
- GPL 격리(예: Drawpile 포크 별도 레포) 등 라이선스 규칙 준수

## 이번 작업

{seed.goal}

## 수용 기준 (seed에서 복사)

- [ ] {acceptance_criteria 1}
- [ ] {acceptance_criteria 2}
...

## 주의

- padlet 프로젝트의 검증 게이트(`prompts/feature/_index.md`) 전 항목 통과 필수
- 구현 중 결정이 달라져야 한다면 **구현 전 ideation에 재인터뷰 요청** — 현장 임의 결정 금지
- `ideation/` 문서는 **읽기 참조용**, 이 폴더를 수정해야 하면 ideation 에이전트에게 요청
```

## 절차

1. seed.yaml에서 goal·acceptance_criteria 추출
2. request.json의 `user_story`·`success_metric`을 seed 기반으로 작성 (padlet phase0 포맷 준수)
3. 관련 문서 경로를 `context_refs`에 나열
4. handoff_note.md 작성 (위 템플릿)
5. 두 파일 모두 task 폴더 저장
6. `plans/phase0-requests.md` 에도 블록 추가 (phase5에서 시작한 작업 여기서 최종 확정)

## 자율 진행 지침

- `user_story`는 seed의 goal에서 기계적으로 1문장 유도
- `success_metric`은 seed.acceptance_criteria 중 가장 측정 가능한 것 하나 선택
- `affected_surfaces`는 sketch.md·roadmap의 변경 지점에서 경로 추출

## 검증 게이트 (인수인계 검증)

- `padlet_phase0_request.json` 존재 + 필드 완비
- `handoff_note.md` 존재 + 참조 문서 ≥ 4개
- seed.yaml의 acceptance_criteria 전부가 handoff_note에 체크리스트로 복사됨

## 종결

- 해당 task 완료로 처리
- 사용자에게 최종 보고: "{topic} 시드 생성 완료. padlet 쪽에 던질 프롬프트는 `phase6/handoff_note.md`, request는 `phase6/padlet_phase0_request.json`"
