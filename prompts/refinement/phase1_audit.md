# Phase 1 (refinement) — Audit

기존 결정·문서 감사. 변경 요인 정리.

## 입력

`tasks/{task_id}-refine/phase0/target.json`

## 출력

`tasks/{task_id}-refine/phase1/audit.md`

```markdown
# Audit — {topic}

## 대상 시드
- `seed_91bcd4e99efb` — 그림보드×라이브러리 (ambiguity 0.177, 2026-04-12)

## 기존 확정 결정 요약

| # | 결정 | 근거 | 유효성 |
|---|---|---|---|
| 1 | Drawpile 채택 | 이비스페인트 대체 | ✅ 유지 |
| 2 | iPad 9th 기준 | 초기 학교 감각 | ❌ 갤럭시 탭으로 변경됨 |
...

## 변경 요인

- iPad → 갤럭시 탭 S6 Lite: Apple Pencil → S-Pen, Safari → Chrome Android
- 교사 인터뷰에서 세로 스크롤 선호 확인

## 연쇄 영향

- 성능 예산 수치: 그대로 유지 가능(TTI 3s, 드래그 60fps) but 웹 런타임 특성 차이 반영 필요
- S-Pen Pointer Events 방식 (Apple Pencil과 호환 가능)
- ...

## 뒤집힐 후보 결정

인터뷰가 확정해야 할 것들.
```

## 절차

1. `target.json`의 각 seed_id로 Ouroboros DB에서 시드 YAML 조회(참조만)
   - 실무상 `seeds-index.md` + 관련 plan 문서로 대체 가능
2. 각 확정 결정을 표로 나열
3. 변경 요인(change_trigger 확장) 자세히 기술
4. 연쇄 영향 분석 — 어떤 결정이 뒤집힐 가능성 있는가
5. phase3 인터뷰에서 재검토할 항목 초벌 목록화

## 자율 진행 지침

- 기존 시드의 acceptance_criteria와 현재 상황 차이를 구체 근거로 기록
- 유효성 판정에서 "✅ 유지" 항목은 이번 재검토 범위 밖이라고 명시 (인터뷰 축소)

## 검증 게이트

- 대상 시드별로 확정 결정 요약 존재
- 변경 요인 ≥ 1개
- 뒤집힐 후보 결정 ≥ 1개 (0개면 refinement 불필요 → task 종결)

## 핸드오프

`audit.md` phase2에 전달.
