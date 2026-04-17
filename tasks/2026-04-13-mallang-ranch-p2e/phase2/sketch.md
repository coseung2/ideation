# Phase 2 — Sketch: Mallang Ranch P2E (코드네임, 외부 브랜드 미정)

> task_id: `2026-04-13-mallang-ranch-p2e`
> 생성일: 2026-04-13
> 에이전트: sketch-architect
> 입력: phase0/request.json, phase1/exploration.md

---

## 전제 (phase1 권고 고정, 재논의 금지)

1. **1순위 운영 모델**: Sunflower Land (커뮤니티 소유 토큰·팀 리저브 0·오픈소스·gas-free 인게임 TX) **+** Pixels Chapter 2.5 토크노믹스 통제 메커니즘(스펜드온리 세컨드 토큰·출금 페널티·월 발행 캡)을 이식한 하이브리드.
2. **체인**: **Base L2 단일 MVP**. Immutable zkEVM는 심사 병행(Passport 월렛 추상화 획득 시 2순위 채택 가능). Ronin·Solana·TON은 v2 이후 파킹.
3. **IP**: 원작 "말랑말랑목장" **직접 차용 금지**. 감성·장르 레벨만 참조, 브랜드·캐릭터·월드는 오리지널 신규 설계. 외부 작명은 phase3 인터뷰에서 확정.
4. **지역**: 글로벌 170개국 런치, **한국 IP 차단 + 한국 KYC 거주지 페이아웃 차단** (위메이드 이미르 전례). 앱스토어는 한국 계정 미등재, 글로벌 계정 + PWA.
5. **수익 구조 우선순위**: (1) NFT 프라이머리 세일 + (2) 인게임 IAP/SKU + (3) 2차 마켓 수수료 ≫ 토큰 소각·페이아웃. 토큰은 **거버넌스·소규모 유틸 한정** (Cookie Run/Hay Day 대조군 교훈).
6. **개발 규모**: 1인~초소규모. 커스텀 L2·밸리데이터 금지. Base 위 컨트랙트 배포 + 오프체인 서버 + 인디 스케일 오픈소스 생태계 활용.

---

## 데이터 모델 초안

### 온체인 자산 (Base L2, EVM)

| 자산 클래스 | 표준 | 온체인화 여부 | 이유 |
|---|---|---|---|
| **Animal NFT** (고유 동물 개체) | ERC-721 | **온체인 필수** | 감정적 애착·2차 거래의 핵심 소유물. Sunflower Bumpkin·Pixels Pet 동일 포지션 |
| **Land NFT** (목장 토지 플롯) | ERC-721 | **온체인 필수** | 프라이머리 세일의 주 수익원. 공급 캡 설정으로 토크노믹스 앵커 역할 |
| **Rare Item NFT** (희귀 도구·장식·한정 코스튬) | ERC-1155 | **온체인** (선별적) | 희귀도·시즌 한정 품만 온체인. 소량 발행·수집 가치 |
| **Common Item/Seed/Feed** (소비 자원) | — (오프체인 DB) | **오프체인** | 높은 TPS·저가치. 온체인화 시 가스·UX 낭비 |
| **Harvest/Produce** (우유·계란·양털 등 수확물) | — (오프체인) | **오프체인 기본**, 선택적 on-chain "Bottling" | 평시는 오프체인 가상 재화. 유저가 "병입(bottle)" 선택 시 ERC-1155로 온체인화 → 2차 거래 가능. Sunflower SFL 인게임→토큰 스왑 메커니즘 차용 |
| **$RANCH (가칭, 가상자산 토큰)** | ERC-20 | **온체인** | 거버넌스 + 소규모 유틸. 총공급 제한·팀 리저브 0 |
| **$FEED (가칭, 스펜드온리 인게임 토큰)** | **비전송 내부 원장** (오프체인) or soulbound ERC-20 | 하이브리드 | Pixels vPIXEL 대응. 플레이 보상 = 인게임 소비 가능, 출금·P2P 전송 불가 |

**핵심 결정(즉결)**:
- **3중 토큰 구조**: (a) Animal/Land/Rare Item NFT — 소유권, (b) $RANCH ERC-20 — 거버넌스·유틸 일부, (c) $FEED 스펜드온리 — 플레이 보상·인게임 경제. Pixels 2.5 공식 + Sunflower 팀 리저브 0 철학 결합.
- **토큰 출시 시점**: MVP(Season 0)는 **토큰 없이 NFT 프라이머리 + $FEED 오프체인 포인트로 가동**. 커뮤니티·DAU가 5만+ 누적 시 $RANCH 프리시즌→TGE. Sunflower가 SFL→FLOWER를 2년차에 교체한 전례.
- **출금 페널티**: $FEED → $RANCH 스왑은 **플레이 시간 14일 락업 + 20~50% 페널티 슬로프** (Pixels Chapter 2.5). 바이패스 가능한 스테이킹 트랙 제공(장기 홀더 우대).

### 오프체인 게임 상태 스키마 (PostgreSQL/Supabase 가정, 약식)

```
Player {
  id: uuid PK
  wallet_address: string UNIQUE
  passport_sub: string NULL       # Immutable Passport / 소셜 로그인 서브
  display_name: string
  country_code: string            # KR이면 페이아웃 차단 플래그
  created_at: timestamp
  last_login_at: timestamp
  feed_balance: bigint            # 오프체인 $FEED 잔고
  experience: int
  tutorial_completed: bool
}

Ranch {
  id: uuid PK
  owner_player_id: uuid FK → Player.id
  land_nft_token_id: string UNIQUE NULL   # 무료 체험 랜치는 NULL, NFT 구매 시 연결
  tier: enum('trial','owned')             # 체험(임대) vs 소유
  size_w: int                              # e.g. 6x6 (trial), 10x10 (Small), 16x16 (Large)
  size_h: int
  layout_json: jsonb                       # 구조물 배치
  updated_at: timestamp
}

AnimalInstance {
  id: uuid PK
  ranch_id: uuid FK → Ranch.id
  nft_token_id: string UNIQUE NULL         # Trial 동물은 NULL, NFT화 시 연결
  species: enum('cow','chicken','sheep','rabbit','alpaca', ...)  # MVP 5종
  rarity: enum('common','rare','epic','legendary')               # MVP 3~4단계
  name: string                              # 플레이어 애칭
  level: int                                # 돌봄에 따른 성장
  affection: int                            # 0~100
  hunger: int                               # 0~100 (주기적 감소)
  cleanliness: int
  birth_timestamp: timestamp
  last_fed_at: timestamp
  last_harvested_at: timestamp
  traits_json: jsonb                        # 시드 기반 생성 고유 특성
}

Harvest {
  id: uuid PK
  animal_instance_id: uuid FK
  produce_type: enum('milk','egg','wool','carrot', ...)
  quantity: int
  quality: enum('normal','premium','gold')
  harvested_at: timestamp
  consumed: bool                            # 가공·판매·bottling로 소비 여부
  bottled_nft_token_id: string NULL         # 온체인 bottling 시
}

Quest {
  id: uuid PK
  player_id: uuid FK
  template_id: string                       # QuestTemplate 참조
  state: enum('available','in_progress','completed','claimed')
  progress_json: jsonb
  started_at: timestamp
  completed_at: timestamp NULL
  reward_feed: bigint
  reward_xp: int
  reward_item_sku: string NULL
}

Wallet {                                    # 오프체인 지갑 캐시·사이닝 세션
  id: uuid PK
  player_id: uuid FK
  chain: enum('base','immutable')
  address: string
  provider: enum('passport','metamask','coinbase','wallet_connect')
  linked_at: timestamp
}

MarketListing {                             # 인게임 2차 마켓 오더북 (온체인 실제 결제는 Seaport/Reservoir 위임)
  id: uuid PK
  nft_contract: string
  token_id: string
  seller_address: string
  price_wei: string
  currency: enum('eth','usdc','ranch')
  listed_at: timestamp
  expires_at: timestamp
  signature: string                         # EIP-712 오프체인 서명
  filled: bool
}

GuildMembership {                           # v1 길드·채팅 (선택적)
  id: uuid PK
  guild_id: uuid FK
  player_id: uuid FK
  role: enum('owner','officer','member')
  joined_at: timestamp
}

EconomyDailyStat {                          # 토크노믹스 모니터링 (Axie 반면교사)
  date: date PK
  feed_minted: bigint
  feed_burned: bigint
  ranch_circulating: bigint
  nft_primary_sales_usd: numeric
  nft_secondary_volume_usd: numeric
  dau: int
  churn_rate: numeric
}
```

**엔티티 수: 10개** (Player, Ranch, AnimalInstance, Harvest, Quest, Wallet, MarketListing, GuildMembership, EconomyDailyStat + 토큰/NFT 온체인 클래스 제외 순수 오프체인 기준 9 + 시즌/이벤트 메타 1 예비 = 10).

### 온·오프체인 동기화 지점

| 트리거 | 방향 | 설명 |
|---|---|---|
| **NFT 프라이머리 세일 민팅** | off→on | 유저 결제(Base ETH/USDC) → Land/Animal NFT mint → 인덱서가 AnimalInstance/Ranch row 생성 |
| **2차 마켓 거래 체결** | on→off | Seaport/Reservoir 주문 체결 이벤트 수신 → owner 변경 반영 |
| **동물 돌봄·수확·레벨업** | off만 | 온체인 쓰기 없음. 모든 상태는 오프체인. 가스·UX 고려 |
| **희귀도 업그레이드** (희귀 단계 상승 시 NFT metadata 갱신) | off→on | 유저가 재료 소각 + 서버 검증 → 컨트랙트 `upgrade(tokenId, newRarity)` 호출. 배치 처리 (일 1회) |
| **Harvest → Bottle NFT** | off→on | 유저가 bottling 선택 시 ERC-1155 민팅. 선택적, 수수료 부담 유저 |
| **$FEED → $RANCH 스왑** | off→on | 락업·페널티 적용 후 오라클 서명으로 ERC-20 민팅. 월 발행 캡 컨트랙트 enforce |
| **시즌 리워드 정산** | off→on | 시즌 종료 시 상위 랭커에 한정 NFT 에어드랍 (Merkle drop) |
| **거버넌스 투표** | on only | $RANCH 홀더 Snapshot off-chain 서명 투표. 실행은 멀티시그 |

동기화 전략: **오프체인 우선, 온체인은 경제적 유의미한 이벤트만**. Sunflower의 gas-free 인게임 TX 철학 + Pixels의 오라클 기반 민팅 혼합.

---

## 사용자 흐름

### 신규 유저 (지갑 없음 → 첫 수확 → NFT 구매 유도)

1. **랜딩** (PWA 웹앱) → "Try Free Ranch" CTA. 한국 IP면 "Global Store" 안내(다른 지역 우회 가이드 없음, 단순 접근 차단 공지).
2. **계정 생성**: Immutable Passport 소셜 로그인(Google/Apple/이메일 OTP). 지갑은 백그라운드로 자동 생성·관리. 로그인 → 튜토리얼 진입 ≤ 15초 목표.
3. **무료 체험 랜치**: 6x6 플롯, 공통 등급 동물 1~2마리 임대 지급(NFT 아님). 7일 제한.
4. **첫 수확 튜토리얼**: 먹이 주기 → 24시간(가속 2시간) → 우유/계란 수확 → 가공/판매 → $FEED 포인트 획득.
5. **NFT 구매 유도**: 체험 3일차 또는 레벨 5 달성 시 "Starter Pack" 오퍼 (Land Small + Common Animal x2, 약 $15~30, Coinbase 온램프로 신용카드 결제 가능).
6. **지갑 승격**: 소셜 로그인 세션을 외부 지갑(Metamask/Coinbase Wallet)으로 export 가능(Passport 키 export 기능).

### 리텐션 유저 (일일 접속 → 돌봄 → 길드·이벤트)

1. **일일 접속 보상**: 접속 시 $FEED 소량 + 주간 연속 보너스(7일 스탬프).
2. **돌봄 루프**: 동물 먹이·청소·쓰다듬기(3~5분 세션) → 애정도·생산량 영향. 오프라인 감소(Hay Day 공식).
3. **수확·가공**: 원재료 → 가공품(치즈·빵 등) → 판매. 가공이 마진 차이.
4. **길드/채팅**: 길드 단위 주간 목표. Discord·Telegram과 Webhook 연동.
5. **주간 이벤트**: 시즌 테마(봄·할로윈 등) → 한정 동물·코스튬 → Rare NFT 드랍.

### 경제 참여자 (NFT 마켓 거래 → 2차 수수료 → P2P)

1. **인게임 마켓**: Reservoir API 임베드 or 자체 오더북. ETH/USDC/$RANCH 결제.
2. **2차 거래 수수료**: 판매자 2.5% + 로열티 5% → (로열티 50% 팀 운영비, 50% 커뮤니티 트레저리).
3. **희귀도 크래프팅**: 같은 종 동일 등급 3마리 + $RANCH 소량 → 1단계 상위 등급(공급 축소·번 싱크).
4. **임대/스콜라십 (v2 파킹)**: Axie 교훈으로 MVP엔 미도입. 길드 내 공유 편의 기능만.

### 캐시아웃 유저 (토큰/NFT 매도 → 스테이블 환전, 한국 불가)

1. **NFT 매도**: 2차 마켓 → ETH/USDC 수령 → Coinbase 오프램프 → 현지 은행(한국 거주자 불가).
2. **$RANCH 매도**: CEX 상장 시(Season 3+) 외부 DEX·CEX 매도. MVP엔 $RANCH 미발행.
3. **$FEED**: 외부 출금 **불가**. 인게임 소비만.
4. **KYC 체크**: 페이아웃 시 Passport KYC 거주국 확인 → KR/CN 차단.

---

## 성능·UX 체크리스트 (웹·모바일 하이브리드)

| 항목 | 목표 | 근거 |
|---|---|---|
| **모바일 브라우저 FPS** | 60fps (중저사양 안드로이드 포함) | 캐주얼 타깃·저압력 UX 필수. 에셋은 스프라이트 기반 2D·총 용량 ≤ 50MB 초기 로드 |
| **초기 진입~지갑 생성** | ≤ 15초 (Passport 소셜 로그인 기준) | Immutable Passport 공식 벤치 8~12초 + 네트워크 버퍼 |
| **온체인 TX 체감 지연** | ≤ 3초 confirm, ≤ 12초 finality (Base L2) | Base 블록타임 2초 × 1 confirm |
| **가스비 체감** | 건당 < $0.02 | Base 평균 $0.001~0.01 대역 |
| **오프라인 → 온라인 복귀** | 돌봄 상태 보존, 놓친 생산은 타임스탬프 기반 누적(상한 캡) | Hay Day 공식 |
| **번역·지역화** | 영어(글로벌 기본) + 일본어·베트남어·태국어(동남아 Web3 유저) + **한국어 선택 미제공**(한국 서비스 의도 없음 신호) | phase3 재논의 가능 — 해외 교민 배려 여부 |
| **접근성** | WCAG AA(컬러 콘트라스트·포커스·자막) | PWA 웹 기반 필수 |
| **오프라인/PWA** | Service Worker 캐시, 오프라인 시 "마지막 동기화 시점" 배지 | 네트워크 불안정 모바일 |
| **월렛 UX** | 가스 추상화(스폰서드 TX, Paymaster) + 유저에게 "수수료 무료" 표시 | Sunflower gas-free 철학 복제 |

---

## 시너지·분배 채널

### 타 P2E·온체인 게임 인터옵
- **월렛 공용**: Passport·Metamask·Coinbase Wallet 모두 지원. Base에서 다른 Base 게임(Zerebro·Base Gods·Sunflower Land Base 레거시)과 동일 지갑 공유.
- **에셋 공유 가능성 (v2 이후 파킹)**: 타 P2E 동물 NFT wrap→ 본 게임 전시 전용(스탯 적용 불가) 기능. MVP는 과도함.
- **Soneium/Story Protocol 등 IP 체인 연동**: v2 이후 오리지널 캐릭터 IP 자체를 온체인 IP로 등록 옵션.

### 커뮤니티 도구
- **Discord**: 길드 채널 자동 생성(Webhook), 역할 연동(지갑 홀딩 기반 Collab.land).
- **X(Twitter)**: 수확·희귀 동물 획득 공유 카드 자동 생성, 공식 계정 주간 리더보드.
- **Telegram**: 동남아 유저 대상 봇 + mini-app (TON 연동은 v2).
- **Reddit r/CryptoGaming**: 커뮤니티 AMA, Sunflower·Pixels 서브레딧 크로스 참여.

### 결제·스토어 하이브리드
- **웹 결제**: Coinbase Onramp / Stripe 신용카드 → USDC → NFT/$RANCH. KYC·지역 차단 내재.
- **앱스토어**: **Google Play 글로벌 계정** 등록 가능성 타진(Pixels·Sunflower 전례 제한적). **Apple App Store는 P2E·가상자산 정책 엄격** → 웹앱(PWA) 중심, iOS 앱은 v2 이후.
- **IAP vs 웹 결제**:
  - 토큰·NFT 관련 결제는 **웹 결제만** (앱스토어 수수료 30% 회피 + 정책 위반 회피).
  - 코스메틱·시간 단축(타임 스킵) 등 **순수 오프체인 유틸**은 앱스토어 IAP로 분리 판매.
- **수수료 구조**: 2차 마켓 7.5%(판매자 2.5% + 로열티 5%), 프라이머리 세일 100% 팀→그중 30% 커뮤니티 트레저리 적립(Big Time 페어런치 변형).

---

## 리스크 표

| # | 리스크 | 영향 | 완화 |
|---|---|---|---|
| 1 | **토크노믹스 붕괴** (Axie SLP형 무한 발행·투기 이탈) | 치명적 — 프로젝트 수명 단절 | 팀 리저브 0, 월 발행 캡, $FEED 스펜드온리, 14일 락업·20~50% 출금 페널티, EconomyDailyStat 주간 공개. Season 0는 토큰 미발행 |
| 2 | **스마트컨트랙트 익스플로잇** (재진입·오버플로·오라클 조작) | 치명적 — 자금 탈취·평판 파괴 | OpenZeppelin 표준 라이브러리만 사용, 자체 커스텀 로직 최소화. 런치 전 Spearbit/Code4rena 경량 감사(예산 $20~40k). Timelock 7d + Gnosis Safe 2-of-3 멀티시그. Bug bounty $5k~. 출금 일일 캡 Circuit Breaker |
| 3 | **봇 농장·매크로** (스콜라십 농장 모델로 장기 붕괴 — Axie 필리핀 전례) | 높음 — 토큰 가치·리더보드 오염 | 계정당 Land 보유 상한, $FEED/시간 상한, Passport KYC 기반 디바이스 fingerprint, Proof of Humanity(World ID or Gitcoin Passport) 선택 검증 보너스. 스콜라십 공식 지원 미제공 |
| 4 | **규제 리스크** (한국 게임위 등급거부·수사 / 미국 SEC 증권성 판단) | 높음 — 영업 중단·소송 | 한국 IP·KYC 차단, 한국 앱스토어 미등재. **미국 SEC 대비**: $RANCH는 거버넌스 전용, 수익 약속·스테이킹 이자 없음. Howey 테스트 회피 메모 법률 검토(Anderson Kill·Aaron 변호사 자문 예산). DAO 법인(Wyoming DAO LLC or Cayman Foundation) 설립 검토 |
| 5 | **IP 침해** (말랑말랑목장 원작 주장 or 에셋 유사성 클레임) | 중간 — 경고장·스토어 테이크다운 | 원작 명·캐릭터·월드 미사용, 감성만 차용. 아트는 오리지널 디자이너 work-for-hire 계약. 상표 출원(캐릭터·브랜드명 US/JP/EU, 한국 출원 보류). Farmers World·Hay Day류 장르 관습은 차용 허용 |
| 6 | **유저 번아웃** (일일 의무감·FOMO·P2E 피로) | 중간 — 리텐션 하락 | **저압력 철학 명시**: 오프라인 누적 캡, 일일 접속 보상은 "누락해도 보상 반환 티켓" 제공, 이벤트 비강제 참여. Cookie Run/Hay Day의 "놓쳐도 괜찮은" UX. 푸시 알림 기본 OFF |
| 7 | **1인 개발자 번아웃·키퍼슨 리스크** | 치명적 — 프로젝트 좌초 | **스코프 캡**: MVP는 동물 5종·랜치 3크기·퀘스트 템플릿 20개 고정. 오픈소스화(Sunflower 전례)로 커뮤니티 기여자 수혈. 런치 전 최소 공동 개발자 1명 확보 또는 콘트랙트 감사·아트·커뮤니티 매니저 아웃소싱. 트레저리 멀티시그 필수(단일 실패 지점 방지) |
| 8 | **체인 장애** (Base sequencer 다운 · L1 장애) | 중간 — 거래 중단 | Base 공식 fallback 문서 따름(Force Inclusion via L1). 장애 시 오프체인 게임 플레이는 계속 가능하도록 설계. SLA 공지 페이지 |
| 9 | **스토어 BAN** (Apple·Google P2E 정책 위반) | 높음 — 배포 채널 상실 | 기본은 PWA 웹앱. 네이티브 앱은 **토큰·NFT 기능 제거 버전** 별도 빌드(F2P only) 고려. Apple 정책 위반 회피: 인앱에서 토큰 관련 표시·결제 없음, 외부 웹 링크만 |
| 10 | **브리지·온램프 해킹** (Coinbase Onramp·브리지 compromise) | 중간 — 유저 자금 손실 | 공식 Coinbase Onramp·Stripe 직결만 사용(서드파티 브리지 미통합). Warn 배너 "비공식 링크 주의". 핫월렛 최소 잔고 정책 |
| 11 | **DB·오프체인 상태 소실** (서버 장애로 동물 상태·$FEED 잔고 유실) | 중간 — 신뢰 붕괴 | Supabase/Postgres 자동 백업 + 일일 Merkle 스냅샷을 IPFS/Arweave에 공개 게시. 유저가 자신의 오프체인 잔고를 언제든 검증 가능 (Pixels의 트랜스페어런시 대시보드 차용) |

**선정 11종** (최소 7종 요건 초과).

---

## 미결 질문 (phase3 인터뷰용)

1. **MVP 스코프 확정**: 동물 5종(소·닭·양·토끼·알파카) / 희귀도 4단계(common·rare·epic·legendary) / 랜치 크기 3종(6x6 trial, 10x10 small, 16x16 large) 기본안 OK? 아니면 축소(3종·3단계·2크기)? 1인 개발 공수 대비 어느 선인가?
2. **토큰 출시 시점**: 런치 시 $RANCH TGE vs Season 0 NFT·IAP만 + 2년차 TGE(Sunflower 전례)? 후자 권고하나 **마케팅 화제성 vs 지속가능성** 트레이드오프를 사용자가 결정해야 함.
3. **초기 NFT 프라이머리 세일 규모·가격대**: 총 Land 발행량 캡(예: Small 5,000 + Large 1,000) / 가격($30~500 대역) / 세일 구조(Dutch 옥션 vs 고정가 WL vs 공개세일)?
4. **컨트랙트 수 상한**: 1인 감당 가능한 컨트랙트는 몇 개? 제안 기본(AnimalNFT·LandNFT·ItemNFT·RanchToken·FeedClaim·Marketplace = 6개) OK, 아니면 마켓은 Reservoir 위임해서 5개로 축소?
5. **오리지널 브랜드명·아트 디렉션**: "Mallang Ranch"는 코드네임. 외부 브랜드명은? (후보: "Fluffy Meadow" · "Moochi Ranch" · "Pompom Farm" · "Yumiyum Ranch" 등 플레이스홀더). 아트 스타일은 (a) 파스텔 2D 치비 (카카오 감성 복원) vs (b) 픽셀아트 (Sunflower 레거시 호환) vs (c) 3D 로우폴리 중 어디?
6. **수익 분배 비율**: 프라이머리 세일 수익의 (a) 팀 운영 70% + 커뮤니티 트레저리 30% vs (b) 50/50 vs (c) Big Time형 85/15? 2차 로열티 5% 중 팀·트레저리 분배는?
7. **한국어 UI 제공 여부**: 한국 서비스 의도 없음을 신호로 **한국어 미제공**이 기본안이나, 해외 교민·VPN 유저 UX 배려 vs 규제 리스크 명확화 중 어느 우선?

**7개 — 권장 범위(3~7) 상한**.

---

## 요약

**오리지널 브랜드 기반 Base L2 P2E 목장 게임**. Sunflower Land 운영 철학(팀 리저브 0·오픈소스·gas-free) + Pixels Chapter 2.5 토크노믹스 통제(스펜드온리 $FEED·$RANCH 거버넌스·월 발행 캡·출금 페널티) + Cookie Run 대조군 교훈(핵심 수익은 NFT 프라이머리·IAP, 토큰은 보조). 엔티티 10개, 온체인 자산 3종(Animal/Land/Rare Item) + 토큰 2종($RANCH ERC-20, $FEED 스펜드온리 하이브리드). Season 0 MVP는 토큰 없이 NFT·IAP만으로 가동, DAU 5만 누적 시 TGE. 리스크 11종 식별·완화안 매핑. phase3 인터뷰에서 MVP 스코프·가격·브랜드명·분배 비율·한국어 UI 7개 확정 필요.
