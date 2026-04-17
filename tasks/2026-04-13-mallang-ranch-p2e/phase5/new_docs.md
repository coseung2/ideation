# Phase 5 — New docs (mallang-ranch-p2e)

> task_id: `2026-04-13-mallang-ranch-p2e`
> 생성일: 2026-04-13
> 에이전트: integrator

---

## 신규 생성 plan 2건

### 1. `plans/mallang-ranch-p2e-roadmap.md` — 신규 로드맵 (외부 신규 P2E 프로젝트)

10개 섹션 구성 (계약서 명시 §1~§10 전부 충족):

| § | 섹션 | 핵심 내용 |
|---|---|---|
| 0 | 핵심 명제 | 1인 9개월 Base L2 Season 0. 한국 IP 차단 + 한국어 미제공 (이중 시그널). 베이스 5종 직접 모델링 + 희귀 변종은 교배 emergent |
| 1 | 1순위 모델·체인·규제 전제 | Base L2 + Coinbase Smart Wallet (Immutable Passport 조건부 2순위) + Godot/Blender 1인 + Sunflower 철학 + Cayman TGE-6mo + Gitcoin Passport |
| 2 | 핵심 게임 메커닉 | 일일 루프 4단계 + 베이스 동물 5종 + 희귀도 2단계 + 랜치 3크기 + **교배 시스템 상세** (N5/N6/N7 결정 모두 포함) |
| 3 | 토크노믹스 | Triple token 구조 + Season 정의 3단 + **TGE 트리거 (DAU 50K AND volume $200K AND 2개월 연속)** + Howey 회피 |
| 4 | NFT 자산 설계 | Animal/Land(캡 5,000)/Item/Hybrid Trophy + Starter Pack |
| 5 | 컨트랙트 구조 | **6개** (overlay) — AnimalNFT/LandNFT/ItemNFT/RanchToken/FeedClaim/Marketplace 자체 deploy. 감사 $40K 상한 |
| 6 | 수익·분배·거버넌스 | 1차 70/30 + 2차 5% 50/50 + 거버넌스 단계 (Safe 2-of-3 → Snapshot+Timelock) + No team reserve |
| 7 | 9개월 MVP 마일스톤 | M0–M3 (컨트랙트+감사) / M3–M6 (게임 루프+아트) / M6–M9 (closed alpha+세일 준비+launch) |
| 8 | 리스크·완화 | R1~R9 — 솔로 번아웃·세일 흥행·봇·한국 규제·Howey·취약점·스코프·법인·hybrid 분포 |
| 9 | 미해결 분기 | N1·N2·N3 잔존 (N4·N5·N6·N7은 결정 완료) |
| 10 | 변경 로그 | 2026-04-13 초안 — 메인 시드 + supplemental_overlay 병합 4건 명시 |

### 2. `data/mallang-ranch-seed.json` — skeleton 시드 데이터

JSON 스키마 `mallang-ranch-seed.v1`. 필드:

- `branding` — codename + `{TBD_BRAND_NAME}` placeholder
- `base_animals` — 5종 (cow/chicken/sheep/rabbit/alpaca) — slug, 한·영문명, primaryProduct, harvestCycleHours, baseStats(yield/stamina/fertility), appearancePalette
- `rarity_tiers` — 4단(common/rare/epic/legendary). epic·legendary는 `available_at_launch:false` + `source: ["breeding_emergent_only"]` + note에 U1 피벗 명시
- `ranch_sizes` — trial(미분양)/small(4,000캡)/large(1,000캡), 각 ETH 가격
- `land_sale` — 캡 5,000 + 구조·WL 슬롯·예상 매출 천장 $675K + 분배율
- `breeding_system` — N5/N6/N7 결정 전부 (off-chain DNA + on-chain NFT, 72h cooldown +24h/breed, lifetime 7, 100 FEED base, 발견자 보상 2종)
- `hybrid_patterns_skeleton` — `target_count_range: [30, 50]` + 패턴당 8필드 + placeholder 3개 (구체 trait/외형/스탯은 Godot/Blender 통합 시점에 채움)
- `tokenomics` — RANCH (TGE 트리거 객체 + team_reserve_pct=0 + Sunflower 철학) / FEED (off_chain_point, non-transferable)
- `contracts` — 6개 (LandNFT supply_cap_enforced + Marketplace self_deployed + RanchToken deploy_at Season_2)
- `audit` — $40K 상한 breakdown + 후보 firm
- `localization_at_launch` / `localization_excluded` (한국어 명시)
- `proof_of_humanity` Gitcoin Passport
- `legal_entity_track` Season 0 → Cayman TGE-6mo

---

## 신규 생성 사항 검증

| 항목 | 검증 |
|---|---|
| 신규 plan 섹션 11개 | 계약서 명시 11개(0~10) 전부 작성 — 누락 없음 |
| 메인 seed + supplemental_overlay 병합 | 희귀도 3→2, 컨트랙트 5→6, 교배 시스템 추가, Korean UI 제외, 코드네임 유지, TGE 트리거 거래량 추가 — 모두 overlay 우선으로 반영 |
| acceptance_criteria 13개 모두 plan 변환 | §2 (5종/2단/3크기/교배 MVP) + §3 (TGE 트리거) + §4 (Land 5000/가격/세일 구조) + §5 (6개 컨트랙트) + §6 (분배·거버넌스) + §7 (9개월 마일스톤) + §1 (Cayman TGE-6mo) — 전부 커버 |
| ontology_schema 10+3개 모두 plan 변환 | animal·land·rare_item → §4 / ranch_token·feed_token → §3 / primary_sale·revenue_split·treasury → §6 / season·governance → §3·§6 / breeding_event·hybrid_pattern·trait_dna → §2.4 |
| 데이터 placeholder 명시 | hybrid_patterns_skeleton 3개 항목에 placeholder:true + note (Godot/Blender 통합 시점에 채움) — 계약서 "skeleton 수준" 준수 |
| Aura-board 의존성 | 명시적으로 "완전 독립" 표기 — seeds-index 의존성 다이어그램 footer 박스 |

---

## 신규 destination INBOX는 본 phase에서 생성하지 않음

- `../mallang-ranch/INBOX/` 생성과 README 작성은 **phase6 handoff-writer + phase7 dispatcher**의 책임.
- destinations/_registry.md 갱신(외부 프로젝트 표에 mallang-ranch 행 추가)도 dispatcher 단계에서 수행.
- integrator는 ideation 내부 살아있는 문서·시드 데이터 skeleton만 작성.
