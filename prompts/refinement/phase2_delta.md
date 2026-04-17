# Phase 2 (refinement) — Delta

변경 필요 결정 목록 확정. 인터뷰 질문 도출.

## 입력

`tasks/{task_id}-refine/phase1/audit.md`

## 출력

`tasks/{task_id}-refine/phase2/delta.md`

```markdown
# Delta — {topic}

## 변경 필요 결정

| # | 기존 결정 | 제안 변경 | 근거 | 사용자 확정 필요? |
|---|---|---|---|---|
| 1 | 기준 iPad 9th | 갤럭시 탭 S6 Lite | 학교 현장 | ❌ (이미 확정) |
| 2 | iframe 3개 LRU | 동일 수치 유지하되 Chrome Android 전용 검증 필요 | 다른 런타임 | ⚠️ 검증 후 |
| 3 | Apple Pencil 전제 | S-Pen Pointer Events 기준으로 변경 | 플랫폼 | ⚠️ |

## 유지되는 결정

변경 없음 — 이번 재검토 범위 밖:
- Drawpile 채택
- GPL 격리 원칙
- 반 공개 기본값
- ...

## 인터뷰 질문 (phase3 재료)

1. S-Pen 필압 처리 방식 상세?
2. Chrome Android 메모리 상한(Exynos 9611 4GB) 아래 iframe LRU 수치 유효한가?
3. ...
```

## 절차

1. `audit.md`의 "뒤집힐 후보 결정"과 "변경 요인"을 매트릭스로 정리
2. 각 제안 변경에 근거 + 사용자 확정 필요 여부 표시
3. "유지되는 결정"을 명시해 인터뷰에서 다시 건드리지 않도록 잠금
4. 인터뷰 질문 추출 (phase3 입력)

## 자율 진행 지침

- 근거가 명확한 변경(기준 단말 같은)은 에이전트가 "확정" 처리
- 수치·UX 규범 변경은 **사용자 확정 필요** 표시 → phase3에서 AskUserQuestion
- 유지 결정은 phase3에 아예 안 들어가도록 인터뷰 `initial_context`에 "이미 확정" 명시

## 검증 게이트

- 변경 필요 결정 ≥ 1개
- 각 변경에 근거 존재
- 유지되는 결정 목록 존재 (인터뷰 범위 잠금용)
- 인터뷰 질문 ≥ 1개

## 핸드오프

`delta.md` phase3에 전달 (ideation phase3와 공유). phase3 인터뷰의 `initial_context`는 `delta.md` 전문 + "유지 결정 목록"을 잠금 메타로 포함.
