# Phase 4 — Seed

Ouroboros 시드 생성. Ambiguity 검증.

## 입력

- `tasks/{task_id}/phase3/session_id.txt`
- `tasks/{task_id}/phase3/decisions.md`

## 출력

- `tasks/{task_id}/phase4/seed_id.txt`
- `tasks/{task_id}/phase4/seed.yaml` — 시드 YAML 복사본 (감사·참조용)

## 사용 스킬

- `/ouroboros:seed` — session_id로 시드 생성

## 절차

1. `ouroboros_generate_seed` MCP 호출 (session_id 전달)
2. 반환된 seed_id · ambiguity 확인
3. seed YAML 전문을 `phase4/seed.yaml`에 저장
4. seed_id를 `phase4/seed_id.txt`에 저장
5. 결과 사용자에게 요약 보고

## 검증 게이트 (시드 검증)

- **`ambiguity ≤ 0.2`** (Ouroboros 자동 판정)
- seed.yaml의 `goal`·`constraints`·`acceptance_criteria`·`ontology_schema` 필드 비어있지 않음
- `interview_id`가 phase3의 session_id와 일치

미달 시 phase3 재실행 (인터뷰 추가 라운드).

## 에러 처리

- `is_error=true` + `recoverable=true`: 최대 2회 재시도
- 계속 실패 시 phase3 재실행 신호

## 자율 진행 지침

- 시드 생성 자체는 MCP에 의해 자동. 에이전트는 결과만 확인.
- 시드 내용 검토 시 `decisions.md`와 대조, **누락된 결정이 있으면 phase3 재실행 권고**.

## 핸드오프

`seed_id.txt` + `seed.yaml` phase5에 전달.
