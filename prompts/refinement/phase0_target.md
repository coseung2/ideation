# Phase 0 (refinement) — Target

재검토할 기존 시드·주제 지정.

## 입력

사용자 자연어 프롬프트 (예: "기준 단말이 갤럭시 탭으로 바뀌었으니 태블릿 로드맵 재검토").

## 출력

`tasks/{task_id}-refine/phase0/target.json`

```json
{
  "type": "refinement",
  "slug": "tablet-baseline-update",
  "task_id": "2026-04-12-tablet-baseline-update-refine",
  "target_seed_ids": ["seed_91bcd4e99efb"],
  "target_docs": [
    "plans/tablet-performance-roadmap.md",
    "plans/drawing-board-library-roadmap.md"
  ],
  "change_trigger": "기준 단말 iPad 9th → 갤럭시 탭 S6 Lite 변경",
  "affected_decisions": ["성능 예산", "S-Pen 입력 처리", "Chrome Android 최적화 방향"],
  "created_at": "2026-04-12T00:00:00+09:00"
}
```

## 필드 규칙

- `target_seed_ids`: 재검토할 이전 시드들. 최소 1개. 복수면 모두 parent_seed_id로 연결
- `target_docs`: 갱신 대상 `plans/`·`research/` 문서
- `change_trigger`: 왜 재검토가 필요한가 한 문장
- `affected_decisions`: 이번 재검토로 뒤집힐 수 있는 결정들 예상 목록

## 절차

1. 사용자 프롬프트에서 재검토 대상 식별
2. `seeds-index.md`에서 관련 seed_id 찾아 `target_seed_ids`에 기록
3. 관련 plan·research 문서 경로 나열
4. `change_trigger`가 불명확하면 사용자에게 1질문

## 자율 진행 지침

- 사용자가 명시적으로 시드 ID를 주지 않아도 `seeds-index.md` 기반으로 추정 가능
- 단 추정에 의존할 때는 `target.json`에 주석으로 추정 근거 남김

## 검증 게이트

- `target_seed_ids` ≥ 1
- `change_trigger` 비어있지 않음
- `target_docs` ≥ 1

## 핸드오프

`target.json` 한 파일 phase1에 전달.
