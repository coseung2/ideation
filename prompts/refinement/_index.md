# Refinement Pipeline — Index

기존 시드·설계를 재검토(환경 변경·새 제약·결정 번복 반영).

> Ideation과 다른 점: phase0에서 "새 아이디어 캡처" 대신 **기존 시드·문서 감사**로 시작. phase2 sketch 대신 **delta 문서**를 작성. 이후 phase3~6은 ideation과 공유.

## Phase 순서

| # | Phase | 파일 | 비고 |
|---|---|---|---|
| 0 | 타겟 식별 (target) | `phase0_target.md` | 재검토할 기존 시드·주제 지정 |
| 1 | 감사 (audit) | `phase1_audit.md` | 기존 결정·문서 읽고 변경 요인 식별 |
| 2 | Delta (delta) | `phase2_delta.md` | 변경돼야 할 결정만 추린 목록 |
| 3 | Interview | → `ideation/phase3_interview.md` 공유 | |
| 4 | Seed | → `ideation/phase4_seed.md` 공유 | parent_seed_id 연결 |
| 5 | Integrate | → `ideation/phase5_integrate.md` 공유 + supersede 표시 | |
| 6 | Handoff | → `ideation/phase6_handoff.md` 공유 | 변경된 결정을 명시 |

## 원칙

- **새 시드는 이전 시드의 parent_seed_id를 명시** (Ouroboros 시드 체인)
- **살아있는 문서**(`plans/`)는 변경 로그 섹션을 하단에 추가
- **수퍼시드된 시드**는 `seeds-index.md`에서 "superseded by seed_X" 표시

## 트리거 예시

- 기준 단말 변경 (iPad → 갤럭시 탭 S6 Lite)
- 새 제약 추가 (태블릿 성능 예산)
- 가격 정책 변경
- 신규 의존성 (Drawpile 포크 결정 등)
- 사용자 경험 재설계 (가로 노선도 → 세로 타임라인)

## task 디렉토리

```
tasks/{YYYY-MM-DD-slug}-refine/
├── phase0/target.json          # 재검토 타겟 시드 ID·주제
├── phase1/audit.md             # 기존 결정 vs 현재 상황 대조
├── phase2/delta.md             # 변경 필요 항목 목록
├── phase3~6/                   # ideation 공유
```

## 검증 게이트

- **감사 검증** (phase1 후): 기존 시드 ≥ 1개 식별 + 변경 요인 ≥ 1개
- **Delta 검증** (phase2 후): 변경 항목 ≥ 1개 + 각 변경의 근거 명시
- 이후 ideation 파이프라인 게이트 동일

## 스킵 금지

refinement는 phase0~2를 스킵하지 않음 (변경 근거가 핵심).
