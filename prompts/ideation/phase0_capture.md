# Phase 0 — Capture

사용자 아이디어를 측정 가능한 주제로 캡처.

## 입력

사용자 자연어 프롬프트.

## 출력

`tasks/{task_id}/phase0/request.json`

```json
{
  "type": "ideation",
  "slug": "drawing-board-library",
  "task_id": "2026-04-12-drawing-board-library",
  "scope": "quick_decision",
  "topic": "Aura-board에 미술 수업용 그림보드 + 학생 개인 라이브러리 추가",
  "motivation": "이비스페인트 → 갤러리 → 패들렛 업로드 반복 마찰 제거",
  "target_user": "초·중 교사 + 갤럭시 탭 S6 Lite 쓰는 학생",
  "expected_outcome": "학생이 그림 저장 → 다른 보드에서 원클릭 재사용",
  "known_constraints": ["태블릿 성능", "padlet 읽기 전용"],
  "related_docs": ["plans/tablet-performance-roadmap.md"],
  "created_at": "2026-04-12T00:00:00+09:00"
}
```

## 필드 규칙

- `scope`: `parking` | `research_only` | `quick_decision` | `full_exploration`
- `topic`: 한 문장, 대상 시스템(Aura-board 또는 하위 기능) 명시
- `motivation`: "왜 지금" — 해결하려는 페인 포인트 구체
- `target_user`: 교사·학생·학부모·관리자 중 누구
- `expected_outcome`: 측정 가능하거나 관찰 가능한 결과 한 문장

## 절차

1. 사용자 프롬프트에서 `topic`, `motivation`, `target_user` 추출
2. `scope` 판단 — 파킹만인지, 리서치만인지, 인터뷰까지 갈 것인지
3. 기존 `plans/`·`research/`에 관련 문서 있으면 `related_docs`에 링크
4. 정보 부족 시 사용자에게 1~2개 질문 (3개 이상 필요하면 task 보류)

## 자율 진행 지침

- `scope`는 에이전트 판단 가능. 사용자 한 마디로 명시한 경우(예: "파킹해줘" → `parking`) 따름.
- `motivation`·`target_user`는 대화 맥락에서 유추해도 됨. 유추 근거를 `request.json`에 주석으로 남김.

## 금지

- 추상적 outcome ("더 좋은 UX")
- target_user 미상
- 기존 문서 무시하고 중복 제안

## 핸드오프

`request.json` 한 파일만 phase1에 전달. `scope == "parking"`이면 phase2로 건너뛰지 말고 `ideas-parking-lot.md` 직행 후 종결.
