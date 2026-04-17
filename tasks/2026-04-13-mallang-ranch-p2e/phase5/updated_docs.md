# Phase 5 — Updated docs (mallang-ranch-p2e)

> task_id: `2026-04-13-mallang-ranch-p2e`
> 생성일: 2026-04-13
> 에이전트: integrator
> 입력: phase4 seed.yaml (메인 + supplemental_overlay) + phase3 decisions.md (D1~D6 / U1~U3 / N1~N7)

---

## 갱신한 살아있는 문서 3건

### 1. `plans/seeds-index.md`

- **헤더 카운트** "전체 Seed 9개 (Aura-board 8개 + 외부 프로젝트 1개)" → "전체 Seed 10개 (Aura-board 8개 + 외부 프로젝트 2개)"
- **Seed 마스터 표**에 행 10 추가:
  - `# 10 | 코지 P2E 목장 게임 (mallang-ranch-p2e) | seed_f85d1cb8a245 | interview_20260413_070940 | 0.132 | mallang-ranch-p2e-roadmap.md (신규) | mallang-ranch (외부 신규 P2E)`
- **Seed 10 상세 섹션 신규** (Seed 9 다음, 인접 시드 의존성 다이어그램 직전): 핵심 결정 표 24행 — 체인·클라이언트·동물·희귀도·랜치·교배 시스템·cooldown·hybrid·토큰 구조·TGE 트리거·Land 1차 분양·컨트랙트·감사·세일 분배·거버넌스·no team reserve·지역·언어·법인·proof of humanity·오픈소스·MVP 기간.
- **인접 시드 의존성 다이어그램 footer**에 Seed 10 독립 박스 추가 (Seed 9 박스 다음). "★ 완전 독립 — 외부 신규 P2E 프로젝트 ★" + 세부 8줄 (destination·체인·게임 엔진·정책 미적용·코드네임 등).

### 2. `plans/phase0-requests.md`

- 마지막 섹션(GM-10 블록) 다음, "사용 지침" 직전에 **mallang-ranch-p2e 안내 섹션** 추가:
  - padlet 파이프라인 진입 JSON 미수록 명시
  - 외부 destination(`../mallang-ranch/INBOX/`)으로 dispatcher가 직접 배송
  - 작업 분해는 외부 저장소 자체 task tracker에서 관리
  - 핵심 작업 묶음 참조: `plans/mallang-ranch-p2e-roadmap.md` §7
- "사용 지침" 1번에 mallang-ranch 라우팅 안내 1줄 추가.

### 3. `ideas-parking-lot.md`

- 기존 "게임 제작 보드" 섹션 유지.
- 신규 섹션 2건 추가 (N2·N3):
  - **mallang-ranch-p2e — Apple App Store iOS 네이티브 빌드 (v2)** — N2 파킹. 꺼낼 트리거·예상 공수 2~3개월·Sunflower/Pixels 전례 참조.
  - **mallang-ranch-p2e — Immutable zkEVM Passport 심사 결과 대응 (followup)** — N3 followup. 꺼낼 트리거·리서치 항목 3개·심사 결과별 행동 분기.

---

## 갱신 검증

| 항목 | 검증 |
|---|---|
| seeds-index 표 양식 일관 | 기존 9행과 동일 7열 (`# / 주제 / Seed ID / Interview ID / Ambiguity / 연관 plan / Destination`) |
| Seed 10 상세 섹션 양식 일관 | Seed 6·7·8·9 패턴 따라 "핵심 결정 요약" 표 + 상세 설계 링크 |
| 의존성 다이어그램 박스 양식 | Seed 9 박스와 동일 폭·구분선 |
| phase0-requests "사용 지침" 양식 | 기존 번호 매김 유지, 라우팅 안내 1줄 추가 |
| ideas-parking-lot 섹션 양식 | 기존 게임 제작 보드 섹션과 동일 (`## 제목` + 인용 + 본문 + 트리거 + 참고) |

모든 갱신은 기존 행·섹션 삭제 없이 추가 only. 양식 보존.
