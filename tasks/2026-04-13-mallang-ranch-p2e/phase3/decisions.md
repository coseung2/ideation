# Phase 3 — Decisions: Mallang Ranch P2E

> task_id: `2026-04-13-mallang-ranch-p2e`
> 생성일: 2026-04-13
> 에이전트: interview-facilitator
> Ouroboros session: `interview_20260413_070940`
> ambiguity 최종값: **0.13** (목표 ≤ 0.20 달성)

---

## 확정 결정 (에이전트 자율 답변)

### D1. MVP 스코프 (동물/희귀도/랜치 크기)

- **동물 5종**: 소(cow) · 닭(chicken) · 양(sheep) · 토끼(rabbit) · 알파카(alpaca)
- **희귀도 3단계**: common / rare / epic (legendary 제거)
- **랜치 3크기**: trial 6x6 / small 10x10 / large 16x16
- **개발 타겟**: Season 0 MVP까지 **9개월**

### D2. 토큰 출시 시점 (확정 세부)

- **Season 0 (런치~런치+12개월)**: 토큰 미발행. $FEED는 오프체인 포인트로만 가동, $RANCH는 미존재.
- **TGE 트리거**: 누적 DAU 5만 달성 + 2차 마켓 월 거래량 $200K 이상 2개월 연속 충족 시 Season 2 시작점에 $RANCH TGE.
- **토큰 스왑 전례 참조**: Sunflower SFL→FLOWER (2년차 교체) 스타일. TGE 시 얼리 어답터 에어드랍은 활동 기반(플레이 시간·길드 기여·NFT 홀딩 가중).
- **TGE 전까지 $RANCH 언급·프로미스 금지** (SEC Howey 테스트 회피).

### D3. 초기 NFT 프라이머리 세일

- **총 Land 발행 캡**: Small 4,000 + Large 1,000 = **합계 5,000 Land**
- **가격대** (Base ETH 기준, ETH $3,000 가정):
  - Small Land: 고정가 0.03 ETH (~$90), WL 0.02 ETH (~$60)
  - Large Land: 고정가 0.15 ETH (~$450), WL 0.10 ETH (~$300)
- **세일 구조**: **WL 48시간 우선 → 3주 갭 → 퍼블릭 고정가**. Dutch 옥션 배제.
- **WL 모집**: Discord·X 초기 커뮤니티 1,000명 선수집 후 기여도·랜덤 혼합으로 2,000 WL 슬롯 선정.
- **예상 매출 천장**: Small 4,000×$75 + Large 1,000×$375 ≒ **$675K**
- **Starter Pack** (세일 이후): Land Small + Common Animal x2 번들 $15~30, Coinbase Onramp 신용카드 결제 유도.

### D4. 컨트랙트 수 상한

- **6개 기본안 채택** (마켓 Reservoir 위임 안 배제):
  1. `AnimalNFT` (ERC-721)
  2. `LandNFT` (ERC-721, 공급 캡 5,000 enforce)
  3. `ItemNFT` (ERC-1155, 희귀 아이템·한정 코스튬)
  4. `RanchToken` (ERC-20, Season 2 배포 전까지 drafting만)
  5. `FeedClaim` (오라클 서명 기반 $FEED→$RANCH 스왑, 락업·페널티·월 캡 enforce)
  6. `Marketplace` (EIP-712 오더북 + Seaport 결제 위임형, 로열티 enforce)
- **근거**: Reservoir 위임으로 5개로 줄이면 로열티 enforce·2차 수수료 수금을 제3자 인프라에 의존해야 함. 1인 감사 $30K 예산 내 6개 표준 컨트랙트(OpenZeppelin 베이스)는 Spearbit·Code4rena 경량 감사 범위 안.
- **감사 예산 상한**: $40K (기본 $30K + 재감사 예비 $10K).

### D5. 수익 분배 비율

- **프라이머리 세일 수익**: **팀 70% / 커뮤니티 트레저리 30%**
  - 팀 $472K: 감사 $40K + 아트 외주 $60K + 초기 운영 6~9개월 $130K + 2년 지속 운영 $192K + 예비비 $50K
  - 트레저리 $202K: 커뮤니티 이벤트·시즌 에어드랍·버그 바운티·$RANCH TGE 시 DEX 유동성 시드.
- **2차 로열티 5%**: **팀 50% / 트레저리 50%** (프라이머리와 다른 비율 — 2차는 지속 운영 재원이므로 트레저리 지분 확대)
- **거버넌스 구조**:
  - Season 0: Gnosis Safe 2-of-3 멀티시그 (개발자 + 파트너 + 아트디렉터 or 법률자문)
  - Season 2 $RANCH TGE 후: 트레저리 출금은 Snapshot 투표 + Timelock 7일 멀티시그 실행으로 승격.

### D6. 지역·규제 운영 (재확인)

- 글로벌 170개국 + 한국 IP 차단 + 한국 KYC 거주지 페이아웃 차단 (phase2 전제 유지).
- 앱스토어: **PWA 웹앱 우선**. Google Play 글로벌 계정 등재는 v2 타진. Apple App Store는 토큰·NFT 제거 F2P 빌드 분기 가능성만 열어두고 MVP는 웹.
- 법인: **Cayman Foundation** 설립 검토 (Sunflower Land 호주 Thought Farm Pty와 달리 토큰 거버넌스 회피용). Wyoming DAO LLC는 미국 세무 리스크로 후순위.

---

## 에이전트 답변 근거 (자율 결정 트레이스)

| 결정 | 근거 유형 | 참조 |
|---|---|---|
| 동물 5종·희귀도 3·랜치 3 | 9개월 1인 공수 | Sunflower 인디 6~8개월 + 외주 오버헤드 30% |
| Season 0 노토큰 | 규제·지속가능성 | phase2 전제 8 + Howey 회피 |
| Land 5,000 캡 | 콜드 스타트 앵커 | Pixels 초기 Land 수천~1만 관행 |
| 가격 $60~$450 | 캐주얼 진입장벽 | Sunflower $30~50, Pixels $100~500 중간 |
| WL→퍼블릭 2단계 | 1인 운영 단순성 | Dutch 옥션 가격 디스커버리 리스크 |
| 컨트랙트 6개 | 로열티 자체 enforce 필요 | OpenZeppelin 표준만 사용 |
| 프라이머리 70/30 | 1인 캐시플로우 | 85/15 시그널링 모순, 50/50 운영비 부족 |
| 2차 50/50 | 지속 운영 재원 | Sunflower·Big Time 2차 커뮤니티 지분 확대 관행 |

---

## 사용자 확정 필요 (오케스트레이터에게 에스컬레이션)

Ouroboros는 ambiguity 0.13에 조기 종결했으나, 계약서상 **창작·주관 영역**은 인터뷰 자율 답변 금지 항목이다. 오케스트레이터가 사용자에게 확인받아야 할 2건:

### E1. 오리지널 브랜드명·아트 디렉션

**질문**: 외부 브랜드 네이밍과 아트 스타일을 무엇으로 할 것인가? (원작 "말랑말랑목장"은 코드네임 전용, 외부 사용 금지)

**에이전트 디폴트 제안 (세 개의 후보 조합)**:

| 옵션 | 브랜드명 | 아트 스타일 | 감성 포지셔닝 |
|---|---|---|---|
| **A (권고)** | **Moochi Ranch** | 파스텔 2D 치비 (카카오 감성 복원) | 아시아 캐주얼 유저 + 글로벌 kawaii 접근. 상표 검색 필요 (US/JP/EU). |
| B | **Fluffy Meadow** | 픽셀아트 (Sunflower 레거시 호환) | Web3 네이티브 친숙도. 차별화 약함. |
| C | **Pompom Farm** | 3D 로우폴리 | 모바일 퍼포먼스 부담·1인 공수 과다. 비추. |
| D (자유 서술) | 사용자 지정 | 사용자 지정 | — |

**에이전트 권고**: **옵션 A (Moochi Ranch + 파스텔 2D 치비)**. 근거: (a) 타겟 1차 30~40대 한국 유저의 카카오 감성과 직접 대응, (b) 글로벌 kawaii 트렌드(Hello Kitty·Pusheen·Line Friends) 호환, (c) 픽셀아트보다 차별화가 뚜렷하고 3D보다 1인 공수 경제적, (d) "Moochi"는 일본어 떡(餅/mochi) 연상으로 말랑 감성 직역.

### E2. 한국어 UI 제공 여부

**질문**: 한국 IP 차단 전제 하에서 한국어 UI를 제공할 것인가?

**옵션**:

| 옵션 | 장점 | 단점 |
|---|---|---|
| **A (권고)** | 한국어 미제공 / 영어·일본어·베트남어·태국어 (동남아 Web3 유저) | 한국 서비스 의도 없음을 규제당국에 명확 시그널링 | 해외 교민·VPN 유저 UX 손상 |
| B | 한국어 제공하되 한국 IP 차단 | 교민·일본계 한국 유저 UX | 규제당국에 "한국 타겟" 시그널로 해석 리스크 |
| C | 영어 단일 런치, 유저 베이스 확보 후 한국어 추가 검토 | 중립·1인 공수 최소 | 한국어권 커뮤니티 초기 형성 지연 |

**에이전트 권고**: **옵션 A (한국어 미제공 + 동남아 언어 우선)**. 근거: (a) 위메이드 이미르 전례 — 국내 서비스 의도 부재를 언어 선택으로 증명, (b) Pixels·Sunflower는 동남아 Web3 유저(필리핀·베트남·태국) 비중이 높아 ROI 좋음, (c) 글로벌 런치 후 1년간 해외 한국어권 유저 유의미 수요 확인되면 v2에서 추가. 옵션 C는 1인 공수엔 더 가벼우나 초기 시즌 다언어 커뮤니티 형성이 리텐션에 직결되어 일본어·동남아 3개 언어는 런치 시점 필수.

---

## 새로 드러난 분기 (세션 편입 금지·향후 파킹)

### N1. Cayman Foundation 설립 시점

phase2 리스크 4번에 "DAO 법인 Wyoming/Cayman 검토"가 있었으나, Season 0 노토큰 기간에 Foundation 먼저 설립할지, TGE 직전 설립할지가 새 분기. 설립 비용 $20~40K. 권장: **TGE 6개월 전 설립**, Season 0는 개발자 개인사업자 or 일반 법인(싱가포르·BVI Ltd)으로 시작.

### N2. Apple App Store iOS 네이티브 빌드 분기

F2P 전용(토큰/NFT 제거) iOS 빌드를 만들지 여부는 개발 공수 2~3개월 추가 필요. 1인 9개월 타겟엔 과도. v2 파킹 확정.

### N3. Immutable zkEVM 심사 결과 대응

심사 통과 시 Passport 월렛 추상화를 Base에도 쓸 수 있는지(크로스체인 가능성) 추가 리서치. 실패 시 Base + Coinbase Smart Wallet fallback. phase4 시드에서 기본은 Base + Coinbase Smart Wallet, Immutable Passport는 "심사 통과 시 2순위" 조건부 조항으로 기록.

### N4. 봇·매크로 방지 Proof of Humanity 선택

리스크 3번 완화에 World ID or Gitcoin Passport 선택 보너스 언급했으나, 어느 쪽을 채택할지 세부 결정. 권장: Gitcoin Passport (Web3 네이티브·무료·Base 연동). World ID는 iris scan 진입장벽.

---

## ambiguity 최종값

**0.13** (Ouroboros 세션 종결 기준). 목표 ≤ 0.20 달성. phase4 seed generation 진입 가능.

---

## 사용자 최종 답변 (2026-04-13 오케스트레이터 라운드)

### U1. 아트 파이프라인·스코프 → **메이저 피벗: 교배 메커닉 도입**

- **아트 도구**: Godot(클라이언트) + Blender(3D 모델링) 본인 직접 작업. 외주 $60K 절감 가능, 단 시간 비용 ↑.
- **스코프 해법**: 사용자 제안 — *"몇개만 개발해두고 교배시 혼종으로 나오는건 알아서 되지 않나?"*
- **확정**:
  - 베이스 동물 **5종 유지** (cow·chicken·sheep·rabbit·alpaca)
  - 희귀도 **2단계로 축소** (common·rare; epic은 교배 결과 emergent)
  - **교배(Breeding) 시스템 추가** — 두 동물 부모 → 자손 동물 (트레잇 절차적 합성)
    - 외형: 부모 색상·패턴·체형 가중 평균 + 랜덤 mutation (Blender shader 파라미터 조합으로 런타임 생성, 별도 모델링 불필요)
    - 능력치: 부모 stat 가중 평균 + 분산
    - 희귀도 emergence: 부모 trait 조합이 사전 정의 패턴과 매치되면 새 hybrid breed 발견 (collection 보상)
- **공수 영향**:
  - 모델링 부담: 5종 × 1 base + 의상·악세서리 5~10종 (≈ 4~6주)
  - 교배 시스템 코드: 절차적 trait DNA + Blender shader 파라미터 매핑 (≈ 4주)
  - 9개월 MVP 안에 가능 (외주 없이도) → **MVP 기간 9개월 유지**
- **새 게임플레이 루프**: 일일 돌봄·수확 → 교배로 hybrid 발견 → 컬렉션 완성 욕구 → 리텐션 동력 추가
- **토크노믹스 영향**:
  - 교배 비용으로 $FEED 싱크 추가 (인플레 완화)
  - 발견 hybrid는 NFT로 민팅 가능 (2차 마켓 깊이 ↑)
  - Cooldown·breeding limit으로 봇 매크로 방지 (Axie 무한 교배 인플레 반면교사)

### U2. 한국어 UI → **옵션 A 채택**

한국어 미제공, 영어·일본어·동남아 3개 언어. 위메이드 이미르 전례로 규제 시그널 최소화.

### U3. 브랜드명 → **코드네임 유지**

`mallang-ranch-p2e` 코드네임으로 phase4~7 진행. 실제 마케팅 브랜드는 사용자가 추후 직접 결정. 시드·핸드오프 문서 모두 코드네임 사용, 브랜드 자리는 `{TBD_BRAND_NAME}` 플레이스홀더.

---

## 새 분기 추가 (U1 파생)

### N5. 교배 시스템 온체인/오프체인 경계

- 옵션 a: trait DNA 전체 온체인 저장 (CryptoKitties식). 가스비 ↑·검증 가능성 ↑.
- 옵션 b: trait DNA 오프체인, 결과 NFT만 온체인 (Axie식). 가스비 ↓·중앙 신뢰 필요.
- 옵션 c: 하이브리드 — DNA hash만 온체인, full DNA 오프체인 reveal (zk-style).
- **권장**: 옵션 b (Sunflower 오프체인 우선 원칙과 정합). 시드에 명시.

### N6. 교배 cooldown·limit

- 권장: 동물당 평생 교배 횟수 7회 한계, cooldown 72시간, 각 교배마다 cooldown 24h씩 증가 (CryptoKitties siring cap 패턴).
- $FEED 비용: 1회 100 $FEED + 양쪽 부모 희귀도 합산에 비례 가중.

### N7. Hybrid breed 발견 보상

- 길드·전체 첫 발견자에게 NFT 트로피 + $FEED 보너스 (디스커버리 보드 게이미피케이션).
- 사전 정의 hybrid 패턴 ≈ 30~50종 (커뮤니티가 모두 발견하는 데 6~12개월 소요 예상 → 시즌 1·2 컨텐츠).
