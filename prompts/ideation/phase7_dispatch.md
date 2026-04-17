# Phase 7 — Dispatch

정교화된 산출물을 적절한 destination INBOX에 배송.

## 입력

- `tasks/{task_id}/phase0/request.json`
- `tasks/{task_id}/phase4/seed.yaml`
- `tasks/{task_id}/phase5/updated_docs.md`, `new_docs.md`
- `tasks/{task_id}/phase6/padlet_phase0_request.json`, `handoff_note.md`

## 출력

destination별 분기 — `destinations/_registry.md` 참조.

**padlet INBOX** (기본):
```
destinations/padlet/INBOX/{task_id}/
├── MANIFEST.md
├── request.json
├── handoff_note.md
├── seed.yaml
├── decisions.md
└── context_links.md
```

**parking**: `destinations/parking/{task_id}.md`
**archive**: `destinations/archive/{task_id}/summary.md`
**research-vault**: `destinations/research-vault/{task_id}/report.md`

## 에이전트

`agents/dispatcher.md`

## 절차 (오케스트레이터 관점)

1. phase6 완료 확인 (handoff_note.md·request.json 존재)
2. `Agent` 도구로 `dispatcher` 에이전트 호출
3. dispatcher가 scope·topic·seed 기반으로 destination 결정
4. 판정 모호하면 dispatcher가 AskUserQuestion으로 사용자 확정
5. 산출물 복사·MANIFEST 생성
6. 오케스트레이터가 최종 보고 (destination 경로·파일 수)

## 검증 게이트 (배송 검증)

- MANIFEST.md 존재 + 필수 필드 완비
- destination별 요구 파일 모두 존재
- INBOX 경로 충돌 없음 (동일 task_id 폴더 기존 없음)

## 핸드오프

task 종결. `seeds-index.md`에 배송 경로 기록은 phase5 integrator가 이미 반영. 누락 시 dispatcher가 경고.

## scope 스킵 규칙

- `scope == "parking"`: phase2~6 스킵하고 phase7만 실행 (파킹 전용)
- `scope == "research_only"`: phase3~6 스킵하고 phase1 exploration.md 기반으로 phase7 research-vault 배송

## 종결 조건

- destination INBOX에 배송 성공 → task 완료
- 배송 실패 (충돌·입력 누락) → 이전 phase 재실행
