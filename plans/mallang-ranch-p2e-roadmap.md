# mallang-ranch-p2e 로드맵 (코지 P2E 목장 게임 · 외부 신규 프로젝트)

> 작성일: 2026-04-13
> Seed: `seed_f85d1cb8a245` (task `2026-04-13-mallang-ranch-p2e`)
> Interview: `interview_20260413_070940` (ambiguity 0.132, 자동 종료)
> **Destination**: `mallang-ranch/` — **ideation 외부 신규 프로젝트** (Aura-board 무관)
> 코드네임: `mallang-ranch-p2e` (외부 마케팅 브랜드는 `{TBD_BRAND_NAME}` — 사용자 추후 결정)
> 전제 문서:
> - `ideation/tasks/2026-04-13-mallang-ranch-p2e/phase1/exploration.md` (체인·런타임·Sunflower/Pixels 비교)
> - `ideation/tasks/2026-04-13-mallang-ranch-p2e/phase2/sketch.md` (도메인·세일·컨트랙트 초안)
> - `ideation/tasks/2026-04-13-mallang-ranch-p2e/phase3/decisions.md` (D1~D6 + U1~U3 + N1~N7)
> - `ideation/tasks/2026-04-13-mallang-ranch-p2e/phase4/seed.yaml` (메인 + supplemental_overlay)

---

## 0. 핵심 명제

> **1인 개발자가 9개월 안에 Base L2 위에서 코지 P2E 목장 게임 Season 0를 출시한다. Land NFT 5,000장 1차 판매로 9개월+2년 운영비를 충당하고, $RANCH 토큰은 DAU 5만 + 월 거래량 $200K 2개월 연속 충족 시점에만 발행한다. 한국 IP/KYC는 차단하되 한국어 UI는 제공하지 않는다(규제 시그널 최소화). 베이스 동물 5종은 직접 모델링하고, 희귀 변종은 교배 시스템의 emergent 결과로만 등장시켜 1인 아트 공수를 절감한다.**

본 프로젝트는 Aura-board(`padlet/`)와 **완전 독립**이며, ideation 산출물은 설계 문서만이다. 실제 코드는 외부 저장소 `mallang-ranch/`에 위치한다.

---

## 1. 1순위 모델·체인·규제 전제 (phase1 권고 승계)

| 축 | 채택 | 사유 |
|---|---|---|
| 체인 | **Base L2** (Coinbase) | 가스 저렴·EVM 표준·OnchainKit·Coinbase Smart Wallet 통합. 커스텀 L2 배제(1인 운영 불가) |
| 월렛 추상화 | Coinbase Smart Wallet (기본) + Immutable Passport (심사 통과 시 2순위 조건부) | 가스 free + 패스키 인증 |
| 게임 클라이언트 | **Godot** (PWA HTML5 export) | Apple App Store iOS 빌드는 v2 파킹 |
| 모델링 | **Blender** (1인 직접) | 외주 $0 — $60K 절감분을 교배 시스템 개발·예비비로 재배치 |
| 영감 모델 | **Sunflower Land 철학** (no team reserve, off-chain priority) + Pixels Land 분양 구조 | 신뢰 있는 탈중앙 시그널링 |
| 지역 | 글로벌 170개국, **한국 IP 차단 + 한국 KYC 거주지 페이아웃 차단** | 위메이드 이미르 전례 |
| 언어 (런치) | **English / Japanese / Vietnamese / Thai** (Korean 미제공) | U2 — Pixels·Sunflower SE Asia Web3 유저 비중 + 규제 시그널 |
| 법인 | Season 0: 개인사업자 or 싱가포르 BVI Ltd → **TGE 6개월 전 Cayman Foundation 설립** | N1 |
| Proof of Humanity | **Gitcoin Passport** (Web3 native, free, Base 연동) | World ID iris scan 진입장벽 후순위 |
| 오픈소스 | 코드 전체 공개 | 신뢰 + 기여 유도 |

---

## 2. 핵심 게임 메커닉

### 2.1 일일 루프 (Season 0 MVP)

1. 동물 돌봄 (먹이·청소) → $FEED 소비
2. 수확 (우유·달걀·양털·당근·알파카털) → 시장 출하 → $FEED 획득
3. 랜드 꾸미기 (코스튬·악세서리)
4. **교배** → hybrid 발견 → 컬렉션 보드 등록

### 2.2 베이스 동물 (5종, 희귀도 2단계)

| 종 | 영문 slug | 기본 산물 |
|---|---|---|
| 소 | cow | 우유 |
| 닭 | chicken | 달걀 |
| 양 | sheep | 양털 |
| 토끼 | rabbit | 당근(채집 대행) |
| 알파카 | alpaca | 알파카털 |

희귀도: **common / rare** (2단계). epic·legendary는 **교배 emergent 결과로만** 등장 — 베이스 카탈로그에 슬롯 미존재. (U1 피벗)

### 2.3 랜치 크기 3단계

| 티어 | 그리드 | 동물 수용 |
|---|---|---|
| trial | 6×6 | 체험용, 제한적 |
| small | 10×10 | 메인 진입 |
| large | 16×16 | 길드·고래 유저 |

### 2.4 교배 시스템 (MVP_REQUIRED)

| 항목 | 결정 |
|---|---|
| 부모 수 | 2명 |
| trait 상속 | parent weighted average + random mutation + sometimes_rare_emergence |
| 외형 | 부모 색·패턴·체형 가중 평균 + Blender shader 파라미터 조합으로 **런타임 외형 생성** (별도 모델링 불필요) |
| 능력치 | 부모 stat 가중 평균 + 분산 |
| 희귀도 emergence | trait 조합이 사전 정의 hybrid 패턴과 매치되면 새 hybrid breed 발견 (collection 보상) |
| **온체인/오프체인 경계 (N5)** | **Option B (Axie-style)**: trait DNA = off-chain server / 결과 NFT = on-chain mint with metadata hash. Sunflower 오프체인 우선 정합, 가스비 ↓ |
| Cooldown (N6) | 초기 72시간 + 매 교배마다 24h씩 증가. **lifetime 7회 cap / 동물** (CryptoKitties siring cap 패턴) |
| 비용 | 100 $FEED base + 부모 희귀도 합산 가중 |
| Hybrid 패턴 카탈로그 (N7) | **30~50종** 사전 정의 (Season 1·2 컨텐츠, 커뮤니티 6~12개월 발견 추정) |
| 발견 보상 | 첫 발견자 NFT 트로피 (commemorative, 초기 non-tradeable) + $FEED 보너스 |

**토크노믹스 영향**:
- $FEED sink 추가 → 인플레 완화
- hybrid NFT 추가 발행 → 2차 마켓 깊이 ↑
- cooldown·breeding limit으로 봇 매크로 방지 (Axie 무한 교배 인플레 반면교사)

---

## 3. 토크노믹스

### 3.1 Triple token 구조

| 토큰 | 형식 | 역할 | 발행 시점 |
|---|---|---|---|
| **NFT 자산** (Animal·Land·Item) | ERC-721 / ERC-1155 | 소유권·거래 | Season 0 launch |
| **$FEED** | 비전송 인게임 포인트 (오프체인) | 게임플레이 currency | Season 0 launch (오프체인 포인트로만 가동) |
| **$RANCH** | ERC-20 거버넌스·유틸리티 | 거버넌스·트레저리 분배 | **TGE 트리거 충족 후** Season 2 시작점 |

### 3.2 Season 정의

| Season | 기간 (대략) | 토큰 | 거버넌스 |
|---|---|---|---|
| Season 0 | launch ~ launch+12mo | $FEED 오프체인만, $RANCH 미존재 | Gnosis Safe 2-of-3 멀티시그 (개발자 + 파트너 + 아트디렉터/법률자문) |
| Season 1 | TGE 전 운영 기간 | 동일 | 동일 |
| Season 2+ | TGE 이후 | $RANCH 활성 | Snapshot 투표 + Timelock 7일 멀티시그 실행 |

### 3.3 TGE 트리거 (덮어쓰기 — overlay)

**조건**: DAU 50K **AND** 2차 마켓 월 거래량 $200K **AND** 둘 다 2개월 연속 충족 → Season 2 시작점에 $RANCH TGE.

(MCP 자동 시드는 DAU만 기재되었으나 phase3 D2에서 volume 트리거가 추가됨. supplemental_overlay 우선.)

### 3.4 에어드랍 weighting

활동 기반: 플레이 시간 + 길드 기여 + NFT 홀딩. Sunflower SFL→FLOWER (2년차 교체) 스타일 참조.

### 3.5 Howey 회피

- Season 0~1에서 **$RANCH 언급·프로미스 금지** (마케팅·문서·디스코드 모두).
- 토큰 가치 상승 약속 표현 금지.
- 에어드랍 사전 공지 금지.

---

## 4. NFT 자산 설계

### 4.1 Animal NFT (ERC-721)

- species: cow / chicken / sheep / rabbit / alpaca
- rarity: common / rare (베이스). epic+ 는 hybrid에서만.
- on-chain attribute: species_id, rarity, dna_hash, generation, parent_a_id?, parent_b_id?, breed_count_consumed
- off-chain: full DNA (color_genes, pattern_genes, stat_genes, mutation_seed, parent_lineage)

### 4.2 Land NFT (ERC-721, 캡 5,000)

- size: trial / small / large
- 1차 분양: trial 비분양(체험), Small 4,000 + Large 1,000 = 5,000 캡 enforce
- 가격 (Base ETH 기준, ETH $3,000 가정):
  - Small: public 0.03 ETH (~$90) / WL 0.02 ETH (~$60)
  - Large: public 0.15 ETH (~$450) / WL 0.10 ETH (~$300)
- 세일 구조: **WL 48h 우선 → 3주 갭 → 퍼블릭 고정가** (Dutch 옥션 배제)
- WL 모집: Discord·X 초기 1,000명 → 기여도+랜덤 혼합 2,000 슬롯
- **예상 매출 천장**: 4,000×$75 + 1,000×$375 ≒ **$675K**

### 4.3 Item NFT (ERC-1155)

- 희귀 코스튬·악세서리·도구 (한정 발행)
- 별도 카탈로그는 Season 0 런치 시 5~10종 시드, 이후 시즌마다 추가

### 4.4 Hybrid Trophy NFT (Animal NFT 파생)

- 첫 발견자에게 commemorative trophy (초기 non-tradeable)
- 30~50 hybrid 패턴 × 첫 발견자 1명 = 평생 30~50개 trophy

### 4.5 Starter Pack (세일 이후)

- Land Small + Common Animal × 2 번들 $15~30
- Coinbase Onramp 신용카드 결제 유도 (지갑·가스 진입장벽 ↓)

---

## 5. 컨트랙트 구조 (6개, overlay 확정)

| # | 컨트랙트 | 역할 |
|---|---|---|
| 1 | `AnimalNFT` | ERC-721. species/rarity/dna_hash/lineage 메타 |
| 2 | `LandNFT` | ERC-721. **공급 캡 5,000 enforce**, 세일 단계 enforce |
| 3 | `ItemNFT` | ERC-1155. 코스튬·악세서리·한정 아이템 |
| 4 | `RanchToken` | ERC-20. **Season 2 배포 전까지 drafting만**, 미배포 |
| 5 | `FeedClaim` | 오라클 서명 기반 $FEED → $RANCH 스왑. 락업·페널티·월 캡 enforce. (Season 2부터 활성) |
| 6 | `Marketplace` | **자체 deploy** EIP-712 오더북 + Seaport 결제 위임형. **로열티 enforce 자체 보장** |

**근거** (D4, overlay):
- Reservoir 위임으로 5개로 줄이면 로열티 enforce·2차 수수료 수금이 제3자 인프라 의존 → 1인 운영 risk.
- OpenZeppelin 표준 베이스 → Spearbit·Code4rena 경량 감사 $40K 예산 내 가능.

**감사 예산 상한**: $40K (기본 $30K + 재감사 예비 $10K).

---

## 6. 수익·분배·거버넌스

### 6.1 1차 세일 분배 (70/30)

| 분배 | 금액 (예상 $675K 기준) | 사용처 |
|---|---|---|
| **팀 70% = $472K** | audit $40K + art_outsource $0 (solo Blender) + initial_ops 6~9mo $130K + sustained_ops 2yr $192K + reserve $50K + **breeding_dev_marginal $60K** | overlay: 아트 외주 $60K 절감 → 교배 시스템 개발·예비비로 재배치 |
| **트레저리 30% = $202K** | community events + season airdrops + bug bounty + DEX liquidity seed (TGE 시) | — |

### 6.2 2차 로열티 (5%, 50/50)

| 분배 | 사용처 |
|---|---|
| 팀 50% | 지속 운영 |
| 트레저리 50% | 2차는 지속 운영 재원 — 트레저리 지분 확대 |

### 6.3 거버넌스 단계

| 단계 | 구조 |
|---|---|
| Season 0 (런치~TGE 전) | Gnosis Safe 2-of-3 멀티시그 (개발자 + 파트너 + 아트디렉터/법률자문) |
| Season 2+ ($RANCH TGE 후) | Snapshot 투표 + Timelock 7일 + 멀티시그 실행 |

### 6.4 No team reserve (Sunflower 철학)

- 팀 토큰 적립금 0%
- 팀 보유 토큰은 본인 활동(에어드랍)으로만 획득
- 신뢰 시그널링

---

## 7. 9개월 MVP 마일스톤

| 페이즈 | 기간 | 주요 산출 |
|---|---|---|
| **M0–M3** | 1~3개월 | 6개 컨트랙트 작성 + 단위 테스트 + Spearbit/Code4rena 1차 감사 + Land/Animal/Item ABI 확정 |
| **M3–M6** | 4~6개월 | Godot 클라이언트 + 게임 루프 (돌봄·수확·교배) + Blender 베이스 모델 5종 + shader 파라미터 시스템 + 오프체인 trait DNA 서버 + hybrid 패턴 카탈로그 30~50종 시드 |
| **M6–M9** | 7~9개월 | Closed Alpha (WL 1,000명 초청) + 버그 픽스 + Marketplace 통합 + Coinbase Onramp + WL 48h → 퍼블릭 세일 인프라 + 세일 진행 + Season 0 launch |

---

## 8. 리스크·완화

| # | 리스크 | 완화 |
|---|---|---|
| R1 | 솔로 개발자 번아웃 | scope freeze (5종/2단계/3크기), 외주 $60K 절감을 reserve로, M6 후 closed alpha로 전환해 부담 분산 |
| R2 | 1차 세일 흥행 실패 | WL 우선·고정가로 Dutch 옥션 가격 디스커버리 리스크 회피. Discord·X 초기 1,000명 코어 커뮤니티 사전 형성 |
| R3 | 봇·매크로 farming | Gitcoin Passport + breeding cooldown(72h+24h/breed) + lifetime 7 cap |
| R4 | 한국 규제 신고 | 한국 IP 차단 + 한국 KYC 페이아웃 차단 + **한국어 UI 미제공** (이중 시그널) |
| R5 | Howey 테스트 | Season 0~1 $RANCH 언급·프로미스 금지. TGE는 DAU+거래량 객관 트리거로만 발동 |
| R6 | 컨트랙트 취약점 | 6개 OpenZeppelin 표준 베이스. $30K 1차 감사 + $10K 재감사 예비 |
| R7 | 스코프 크리프 | 동물 5 / 희귀도 2 / 랜치 3 enforce. 위반 시 scope review gate (exit_conditions.scope_breach) |
| R8 | Cayman Foundation 설립 지연 | TGE 6개월 전 트리거. Season 0는 개인사업자/싱가포르 BVI Ltd로 시작 가능 |
| R9 | hybrid 패턴 너무 적음/많음 | 30~50종 범위 — 6~12개월 커뮤니티 발견 분포 시뮬레이션으로 조정 |

---

## 9. 미해결 분기 (잔존)

phase3 N1~N4 + 사용자 라운드 N5~N7 중 **결정 완료**: N4(Gitcoin Passport), N5·N6·N7. **파킹**: N1(Cayman 시점), N2(iOS 네이티브 v2), N3(Immutable zkEVM 심사 대응).

| ID | 미결 | 후속 |
|---|---|---|
| N1 | Cayman Foundation 설립 시점 정확 일자 | TGE 6개월 전 — 운영 진행 중 재확인 |
| N2 | Apple App Store iOS 네이티브 빌드 | v2 파킹 (`ideas-parking-lot.md`) |
| N3 | Immutable zkEVM Passport 심사 결과 | research-vault followup investigation |

---

## 10. 변경 로그

- **2026-04-13**: 초안 작성 — `seed_f85d1cb8a245` 기반. 메인 시드 + supplemental_overlay 병합 (희귀도 3→2, 컨트랙트 5→6, breeding 시스템 추가, Korean UI 제외, 코드네임 유지, TGE 트리거에 거래량 추가).
