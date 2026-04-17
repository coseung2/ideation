# Phase 3 — Interview

Ouroboros 소크라틱 인터뷰로 미결 질문 해소.

## 입력

- `tasks/{task_id}/phase2/sketch.md` (미결 질문 목록 포함)
- 관련 `plans/` 문서 (인터뷰 초기 컨텍스트용)

## 출력

- `tasks/{task_id}/phase3/session_id.txt` — Ouroboros interview session ID
- `tasks/{task_id}/phase3/decisions.md` — 확정 결정 요약

```markdown
# Decisions — {topic}

Interview: {session_id}
Ambiguity: 0.XX

## 확정된 결정

| 미결 | 결정 | 근거 |
|---|---|---|
| 이어그리기 | 덮어쓰기 | 용량 절약, 카드 사본 불변 |
| 공유 범위 | 반 공개 기본 + 개별 비공개 토글 | 갤러리 문화 + 프라이버시 |
...

## 파킹된 항목 (v2+)

- ...
- ...

## 새로 드러난 분기

인터뷰 중 기존 sketch에 없었던 큰 이슈가 등장했다면 기록.
예: "Tier 수익 모델 분기 — 별도 인터뷰 주제로 분리"
```

## 사용 스킬

- `/ouroboros:interview` — 새 세션 시작, initial_context는 `sketch.md` 전문 + 관련 plan 문서 요약

## 절차

1. Ouroboros MCP `ouroboros_interview` 호출 (initial_context에 sketch.md 전문 + 관련 plan 요약 + 이미 확정된 상위 결정 전부 넣음)
2. MCP가 주는 각 질문에 대해 **에이전트가 직접 답변** (기본 방침)
   - 답변 근거: 기존 plans·memory·상식·업계 베스트 프랙티스
   - 수치·가격·가치 판단 등 **사용자 확정이 꼭 필요한 건** AskUserQuestion 경로로 라우팅
3. Ouroboros가 `ambiguity ≤ 0.2` 신호 줄 때까지 반복
4. 완료 후 `decisions.md` 작성

## 자율 답변 지침

**에이전트가 답할 수 있는 것** (기본):
- 스키마·아키텍처 패턴 (기존 padlet 스키마·CLAUDE.md 규칙에서 유도)
- 보안·권한 정책 (기존 RBAC 패턴 준수)
- 성능·태블릿 제약 대응 (`tablet-performance-roadmap.md` 준수)
- 기존에 결정된 정책의 일관 적용 (반 공개 기본·매트릭스 데스크톱 전용 등)

**사용자 판정이 필요한 것** (AskUserQuestion):
- 가격·tier 한도 수치
- 새로운 수익·배포 전략
- 사용자 경험 규범 선택 (예: 학생 주체 vs 교사 주체)
- 이전 결정을 뒤집는 큰 방향 전환

## 분기 처리

인터뷰 중 새로운 큰 주제가 드러나면(예: Tier 수익 모델):
- **현재 세션에 편입 금지** — 원래 topic에 집중
- `decisions.md`의 "새로 드러난 분기"에 기록
- phase5 통합 단계에서 후속 task 필요 여부 결정

## 검증 게이트 (시드 검증, phase4에서 판정)

phase3 자체 게이트는 ambiguity ≤ 0.2 도달 여부. 미달 시 인터뷰 계속.

## 스킵

`sketch.md 미결 질문 == 0`이면 phase3 전체 스킵 가능 (단, phase2에서 명시 스킵 플래그 필요).

## 핸드오프

`session_id.txt` + `decisions.md` phase4에 전달.
