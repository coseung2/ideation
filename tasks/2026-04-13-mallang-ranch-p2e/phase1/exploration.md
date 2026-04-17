# Phase 1 — Exploration: 말랑말랑목장 P2E 재해석

> task_id: `2026-04-13-mallang-ranch-p2e`
> 생성일: 2026-04-13
> 에이전트: explorer

## 요약

말랑말랑목장 감성(저압력·귀여운 사육·수집)을 잇는 온체인 게임의 레퍼런스는 2026년 현재 **Pixels(Ronin)·Sunflower Land(Base/Ronin 멀티체인)** 2강 구도로 굳어졌다. Axie Infinity는 고전적 반면교사(투기적 전투 P2E → SLP 폭락 → 85~90% 가치 감소)로, 2026년 V2 종료 발표까지 이어졌다. 한국은 2026년에도 **현금화 P2E 등급거부 기조 유지**이며, 위메이드·넥슨 전례처럼 **글로벌 170개국 런치 + 국내 VPN 차단**이 사실상 표준. 1인~소규모 개발이 실제로 올라탈 만한 지속가능 루프는 **Sunflower Land 계열(커뮤니티 소유 토큰·팀 리저브 없음·일일 인플레 제어 + Gas-free L2)**. 단, 원작 "말랑말랑목장"은 공개 문헌이 거의 없어 **IP 직접 차용 불가 — 감성만 차용한 오리지널 브랜드**로 가야 한다.

---

## 비교표

| 후보 | 카테고리 | 게임 루프 | 토크노믹스 | 체인 | 2026 상태 | 규제 | 개발 규모 | 라이선스/IP |
|---|---|---|---|---|---|---|---|---|
| **1. Pixels** | A. 경영/수집 | 농장·펫·커뮤니티 퀘스트, 저압력 일일 접속 | 이중(PIXEL + vPIXEL 스펜드온리), 20~50% 출금 페널티, Chapter 2.5로 일일 인플레 **-84%**, 월 28M PIXEL 캡 | Ronin (이전 Polygon에서 마이그) | **1M+ DAU**(2026-03), Chapter 3 Bountyfall 라이브 | 글로벌 서비스, 한국 현금화 차단 통상 우회 | 초기 팀 중규모(수십명), Sky Mavis 퍼블리셔 배경 | 자체 IP |
| **2. Sunflower Land** | A. 경영/수집 + C. 농장 | 농사·요리·자원 수집·플레이어 거래, 계절 업데이트 | **커뮤니티 토큰(팀 리저브 0)**, 2025-04 SFL→FLOWER 스왑, gas-free 인게임 TX | **멀티체인**: Base 시작 → Ronin · TON · Arbitrum · Solana 확장 | 600K+ 팜, 2026 Project II(클리커) 런칭 예정 | 글로벌, 지역 제한 느슨(Thought Farm Pty Ltd/호주) | **오픈소스 · 인디 출발** (팀 소규모 추정) | 자체 IP, 커뮤니티 기여 모델 |
| **3. Axie Infinity** (반면교사) | B. 전투·투기 | Axie 번식·배틀·아레나, 고압력 | AXS+SLP 이중, 무한 SLP 발행 → 붕괴. 2026-01 bAXS(본디드) 도입, SLP Origins 발행 중단, 일일 -30% | Ronin (자체 사이드체인 원조) | V2 "Classic" **2026-06-24 종료**, Origin/Atia's Legacy(MMO)로 피봇 | 글로벌, 한국 미서비스 | 대규모(Sky Mavis 300+명) | 자체 IP |
| **4. Farmers World** | A. 경영 + C. 농장 웹3 | 광산·농장·어업 수집, 도구 내구도 소모 | 3토큰(FWW·FWF·FWG), 싱크 부족으로 자원가 폭락, 커뮤니티 주도 번 필요 | WAX | 150K 활성(2024), 2025년 이후 **업데이트·커뮤니케이션 정체** | 글로벌 | 소규모 팀(익명에 가까움) | 자체 IP |
| **5. Heroes of Mavia** | B. 전투·기지 | Clash of Clans 계 기지 업글·PvP, 모바일 F2P | Ruby 2.0 마이그(2025 Q2), Nexira DAEP로 확장, 스테이킹·PvP 페이즈2 | Base | 로드맵 진행, 모바일 F2P 유지 | 글로벌 F2P | 중규모(Skrice Studios) | 자체 IP |
| **6. Big Time** | B. 전투·수집(MMORPG) | 시간여행 던전·장비 제작 | BIGTIME 캡 5B, **팀/투자자 배정 0%(페어런치)**, 호글래스 차지 드랍, 연간 -76% | 자체 네트워크 | 2025 F2P 전환, 누적 매출 $100M+, Binance 상장 | 글로벌 | 대규모 | 자체 IP |
| **7. Cookie Run: Kingdom** (대조군) | D. 무토큰 F2P | 킹덤 건설·가챠·컬렉션·캠페인 | **순수 IAP 가챠**, P2E·토큰 0, 30일 리텐션 한국 21% | — (온체인 없음) | 2025 리뉴얼 +37.5% 매출, 누적 $500M+ | 한국 정상 서비스(청불 아님) | 대규모(Devsisters) | 자체 IP |
| **8. Hay Day** (대조군) | D. 무토큰 F2P | 말 그대로 농장 경영, 장기 리텐션 벤치마크 | Supercell IAP 표준 | — | 10년+ 장수 | 글로벌·한국 정상 | 대규모 | 자체 IP |

---

## 후보별 상세

### 1. Pixels (Ronin)
- **장점**
  - 캐주얼 농장 루프에 **대규모 DAU(1M+)** 유지 — 감성 타겟과 직접 일치하는 유일 사례
  - Chapter 2.5에서 **일일 인플레 -84%·20~50% 출금 페널티·vPIXEL 스펜드온리 토큰** 도입 → 장기 생존형 설계의 현시점 교과서
  - Ronin 게임 전용 네트워크로 가스비 사실상 제로, 월렛 온보딩(Ronin Wallet) 내재화
- **단점**
  - 팀·자본 규모가 수십명대 + Animoca/Sky Mavis 퍼블리셔 라인 — **1인 개발이 동일 스케일 복제 불가**
  - 토큰 가치 유지를 위해 20~50% 출금 페널티·스테이킹 의존 — 유저 입장 "진짜 인출"하려면 큰 비용
  - Ronin 밸리데이터 요건 **250K RON 스테이크** — 자체 L2 띄우려면 사실상 금융 자본 필요
- **본 프로젝트와 연관성**: 루프·감성은 완벽한 1차 벤치마크. 단 개발 규모는 Sunflower 쪽이 현실적. Pixels는 **"성공한 상위 스케일의 목표 모델"**로 관찰하고, 토크노믹스 메커니즘(스펜드온리 토큰·출금 페널티·월 발행 캡)은 **메커니즘 단위로 이식**.

### 2. Sunflower Land (Base/Ronin/멀티체인)
- **장점**
  - **팀 리저브 0% · 커뮤니티 완전 소유 토큰(FLOWER)** — 지속가능 토크노믹스의 2026 최전선 사례
  - 인디 발로 시작해 600K+ 팜으로 성장 — **1인~소규모 입장에서 복제 가능성이 가장 높은 유일한 레퍼런스**
  - Gas-free 인게임 TX(sidechain/L2 추상화) + 멀티체인(Base→Ronin→TON→Arbitrum→Solana) 확장성
  - 오픈소스 컬처로 스마트컨트랙트·게임 로직 레퍼런스 확보 용이
- **단점**
  - UI가 픽셀아트 웹 기반으로 **"말랑말랑"한 귀여움 감성과는 결이 다름** (스타듀밸리 쪽) — 아트 디렉션은 차별화 필요
  - FLOWER가 완전 커뮤니티 소유라 **운영팀의 현금흐름 설계가 더 어렵다** (NFT 프라이머리 세일·인게임 SKU 판매에 의존)
  - 다체인 확장은 1인이 따라가기 버거움 — MVP는 단일 체인 고정 필수
- **본 프로젝트와 연관성**: **1순위 참조 모델**. 운영 철학(커뮤니티 소유·팀 리저브 0·오픈소스)·런치 체인(Base 단독 MVP)·수익 구조(NFT 프라이머리 + 인게임 SKU)·gas-free UX 모두 벤치마킹.

### 3. Axie Infinity (반면교사)
- **장점**
  - P2E 시장 개척자 — 토크노믹스 실패의 **공개된 포렌식 자료가 가장 풍부**
  - 2026-01 bAXS(본디드)·SLP 발행 중단으로 "늦게라도 피봇하면 이런 모양" 사례
- **단점**
  - **SLP 무한 발행 + 신규 유저 자본 유입 의존** → 2022년 $620M 해킹과 함께 폭락
  - V2 "Classic" 2026-06-24 종료 — P2E-first 게임의 수명 상한 표시
- **본 프로젝트와 연관성**: **절대 하지 말 것 리스트**. (a) 플레이 = 토큰 민팅 직결 금지, (b) 신규 유저 유입=수익원 구조 금지, (c) 스콜라십/길드 기반 매크로 농장화 경계. 반대로 bAXS의 **비전송·본디드 모델**은 Sunflower 철학과 교차 검증.

### 4. Farmers World (WAX)
- **장점**: 저사양 브라우저 + 3토큰 리소스 루프로 한때 150K 활성 — 캐주얼 농장 P2E가 시장 있음을 증명
- **단점**: **싱크 설계 실패**(자원가 폭락·번 캠페인 의존)·2025년 이후 업데이트 정체 → 장기 운영 실패 사례
- **본 프로젝트와 연관성**: 3토큰 분리는 이식할 만하나 **싱크 설계 실패의 해부 자료**로 더 가치 있음. WAX 체인은 2026 시점 신규 개발에 권장하지 않음.

### 5. Heroes of Mavia (Base)
- **장점**: 모바일 F2P Clash of Clans 공식으로 P2E 안착, Ruby 2.0 마이그로 토큰 갱신 선례
- **단점**: 카테고리 B(전투·기지)라 말랑 감성과 **정면 상충**. 사행성 전투 루프
- **본 프로젝트와 연관성**: 참조 가치 낮음. 단 Base L2 선택 근거·모바일 배포 경험은 참고.

### 6. Big Time
- **장점**: **팀/투자자 배정 0%(페어런치)** — 토큰 공정성의 또 다른 교과서. 2025 F2P 전환으로 유저 확보 급증
- **단점**: 카테고리 B(MMORPG 던전), 연간 -76% 토큰 가치 — 감성·지속성 모두 본 프로젝트와 거리
- **본 프로젝트와 연관성**: 페어런치 모델만 벤치마킹 (Sunflower도 동일 철학)

### 7~8. Cookie Run: Kingdom / Hay Day (대조군 — P2E가 꼭 필요한가?)
- **요지**: 2026년에도 **순수 F2P + IAP 가챠/SKU**가 목장·농장 감성의 **최강 수익 모델**임이 실증되고 있다. Cookie Run 2025 리뉴얼 +37.5% 매출, Hay Day 10년+ 장수.
- **함의**: 본 프로젝트가 온체인을 쓰는 **정당성은 "실제 자산 소유·2차 거래"에 국한**되어야 한다. 단순 "돈 벌리는 농장"이면 Cookie Run 쪽이 훨씬 효율적. → **NFT=소유권(동물·토지 캐릭터), 토큰=커뮤니티 거버넌스 + 소규모 유틸**에 국한, **핵심 수익은 IAP/NFT 프라이머리 세일** 설계 쪽이 1인 개발에 유리.

---

## 체인·인프라 옵션

| 체인 | 장점 | 단점 | 1인 개발 진입성 |
|---|---|---|---|
| **Ronin** | 게임 DAU 1위, Ronin Wallet·마켓플레이스 내재, 1M+ 유저에 검증된 UX | **자체 L2 배포는 밸리데이터 250K RON 스테이크 필요**(수십만~백만 USD), 퍼블리셔 관계 필요 | 낮음 — "메인 Ronin EVM 위 컨트랙트 배포"는 가능하나 Pixels 급 수준으로 지원받기는 어려움 |
| **Base** (Coinbase L2) | 가스 매우 저렴($0.001~0.01 대역), EVM 호환, Coinbase 온램프 내재, **Sunflower Land 이전 후 성장 실증** | 게임 전용이 아님 → 게임 퍼블리셔·마켓 없음 | **높음** — Sunflower·기타 인디 다수 사용, 퍼미션리스 배포 |
| **Immutable zkEVM** | 게임 전용 L2, sub-$0.01 fee, 2-sec block, 300+ 게임 파이프라인, 퍼블리셔 지원 | **게임 등재는 Immutable 큐레이션 필요**(신청·심사) | 중간 — 신청 절차는 있으나 소규모도 통과 사례 있음. Immutable Passport(월렛 추상화)가 큰 이점 |
| **Polygon zkEVM** | EVM 1.0 호환 | **2026 sunsetting 발표** — 신규 개발 금지 | 권장 안 함 |
| **Solana / TON** | 모바일 친화, 가스 저렴 | EVM 아님 → 학습곡선, 한국 유저 친숙도 낮음 | 중간 |

### MVP 권고 체인
- **1순위: Base** — 가스·온램프·EVM·퍼미션리스·Sunflower 전례. 1인 개발자 기본값.
- **2순위: Immutable zkEVM** — 게임 전용 + Passport 월렛 추상화(이메일 로그인 + 자산 지갑)가 "캐주얼 이용자가 피로 없이 진입"이라는 동기와 직결. 심사 통과 가능성 선조사 필요.
- **파킹: Ronin** — v2에서 DAU 확보 뒤 멀티체인 확장 옵션(Sunflower가 Base→Ronin 경로 밟은 것처럼).

---

## 규제 환경

### 한국 P2E 등급분류 정책 (2026 기준)
- **기조 유지**: 게임물관리위원회는 2024~2026년 동안 **가상자산 환전·경품 환금성 게임을 사행성 사유로 등급분류 거부** 중. 게임산업진흥법 제28조(경품 제공 금지)·제32조(환전 알선 금지) 근거.
- **2025년 변동**: 게임법 전면개정안(민주당 의원 대표발의)이 **경품금지 조항을 아케이드게임에 한정**하려 했으나, 게임위 폐지·게임진흥원 이관 논의로 **2026년 현재 여전히 입법 계류**. 일부 규제 완화 여지 열림 / 실효 시점 불투명.
- **2025-08 개정 예고**: 등급분류규정 제17조(사행성 확인 사항) 개정 예고 → **심사 기준 명문화**는 진행 중이나 P2E 본체 허용은 아님.
- **실무 결론**: 2026 MVP 시점 **국내 P2E 등급분류 신청은 실패 전제**. 글로벌 우선 런치 + 국내 VPN 우회 묵인이 유일한 유효 전략.

### 글로벌 런치 + 국내 차단의 현실성
- **전례**:
  - 위메이드 미르4 글로벌 / 이미르(170개국, **한국·중국 제외**) — 동접 유지, IP 기반 지역 차단 + VPN 우회 묵인. 운영 성립.
  - 넥슨 NXPC, 카카오게임즈 등도 글로벌 법인으로 P2E 분리 운영.
- **시사점**: **국적에 따른 IP 차단 + 가상자산 페이아웃 지갑에 한국 KYC 거주지 제한**을 설계하면 법적 회색지대에서 운영 가능. 단, 국내 수사·과태료 리스크는 0이 아니며, **거주지 증명 기반 토큰 페이아웃 차단 + 앱스토어 심사 분리**가 필요.
- **앱스토어**: Apple App Store·Google Play 한국 계정에는 P2E 앱 등재 불가 관행 지속. **웹앱(PWA) + 글로벌 스토어** 조합 필요.

### 원작 IP 리스크 — 말랑말랑목장
- 공개 리서치 결과 "말랑말랑목장(2013)"에 대한 **나무위키·한국어 위키 등 공개 레퍼런스가 현재 조회되지 않음** (유사어 "말랑이 온라인(2021, Whoyaho)" 검색됨 — 별개 게임). 이는 두 가지 중 하나를 의미:
  - (a) 사용자의 기억 속 게임이 비공식·단명 SNS 게임이었거나
  - (b) 명칭이 달라 IP 추적이 추가로 필요함
- **법적 권고**: 원작 명칭·캐릭터·에셋은 **직접 차용 금지**. "2010년대 초반 카카오/싸이월드 캐주얼 목장 감성"을 **감성·장르 레벨로만 참조**하고, 브랜드·캐릭터·월드는 **오리지널 IP로 신규 설계**. 한국 저작권법상 "게임 장르·일반적 시스템"은 보호 대상이 아니나 캐릭터 디자인·상표는 엄격 보호.
- **네이밍**: "말랑말랑목장"은 내부 코드네임으로만 사용. 외부 브랜드는 phase2 sketch에서 신규 작명.

---

## 1순위 권고

**Sunflower Land 운영 모델(커뮤니티 소유 토큰·팀 리저브 0·오픈소스)을 1차 참조로, Pixels의 토크노믹스 통제 메커니즘(스펜드온리 세컨드 토큰·출금 페널티·월 발행 캡)을 이식한 하이브리드**를 권고한다. 체인은 **Base 단일 L2 MVP**(Immutable zkEVM 심사 병행).

근거는 프로젝트 제약과 다음과 같이 연결된다:
- **1인~소규모 개발 제약** → Pixels·Big Time·Mavia 같은 중대규모 팀 모델은 스케일이 맞지 않는다. 반면 Sunflower Land는 **인디 발(發) → 600K+ 팜 성장**으로, 동일 규모 복제 가능성이 공개 사례 중 유일.
- **한국 규제 제약** → 현금화 P2E 국내 등급 불가. **글로벌 런치 + 국내 VPN 우회 묵인**이 위메이드 전례로 성립. Base + PWA + 글로벌 스토어 조합이 심사 우회에 최적.
- **지속가능 토크노믹스 제약** → Axie 붕괴(무한 SLP 발행 + 신규 유저 의존) 반대편. **팀 리저브 0(Sunflower) + 일일 발행 캡·출금 페널티·스펜드온리 세컨드 토큰(Pixels Chapter 2.5) + 핵심 수익은 NFT 프라이머리·IAP(Cookie Run 대조군)** 3중 안전장치.
- **말랑말랑 감성 제약** → 카테고리 A(경영·수집), 카테고리 B(전투·투기) 배제. 7~8번 대조군은 **P2E 전면 도입의 정당성을 NFT 소유권·2차 거래로 축소**해야 함을 시사 — 토큰은 커뮤니티 거버넌스·소규모 유틸에 국한, 핵심 현금흐름은 IAP와 NFT 프라이머리 세일로.
- **원작 IP 제약** → 감성만 차용, 오리지널 브랜드 신규 설계. phase2에서 네이밍·아트 디렉션 확정.

---

## 참조

- [Pixels CEO Reflects on First Year on Ronin (NFT Plazas)](https://nftplazas.com/pixels-2025-goals/)
- [Pixels Strategic Updates to Improve Token Utility (GAM3S.GG)](https://gam3s.gg/news/pixels-updates-token-utility/)
- [Sunflower Land to Launch $FLOWER Token April 2025 (GAM3S.GG)](https://gam3s.gg/news/sunflower-land-flower-token/)
- [Sunflower Land Expands to Ronin Ahead of FLOWER Token Launch (JuiceNews)](https://juicenews.io/article/sunflower-land-expands-ronin-flower-token/)
- [Sunflower Land Project II 2026 Announcement (ChainPlay)](https://chainplay.gg/blog/a-new-chapter-for-sunflower-land-project-ii-announced-for-2026/)
- [Sunflower Land official site (docs fetched)](https://docs.sunflower-land.com/)
- [Playing, earning, crashing, and grinding: Axie infinity (Sage 2025 학술논문)](https://journals.sagepub.com/doi/10.1177/20539517251357296)
- [Axie Infinity Pivots Tokenomics and Web3 Strategy in 2026 (Ainvest)](https://www.ainvest.com/news/axie-infinity-pivots-tokenomics-web3-strategy-2026-2601/)
- [Farmers World Overview (NFT Evening)](https://nftevening.com/farmers-world-everything-you-need-to-know/)
- [Heroes of Mavia 2025 Roadmap (ChainPlay)](https://chainplay.gg/blog/heroes-of-mavia-q1-2025-launches-mpex-begins-nexira-development/)
- [Big Time 2025 Guide (Gate.com)](https://www.gate.com/learn/articles/a-comprehensive-guide-to-the-popular-blockchain-game-bigtime/853)
- [Ronin zkEVM with Polygon CDK (BlockchainGamerBiz)](https://www.blockchaingamer.biz/news/32770/ronin-zkevm-polygon-cdk/)
- [Immutable zkEVM chain settings (Thirdweb)](https://thirdweb.com/immutable-zkevm)
- [Gaming Crypto Analysis IMX vs RON Late 2025 (CryptoScopeLab)](https://cryptoscopelab.com/gaming-crypto-analysis-imx-vs-ron-review/)
- [게임물관리위원회 등급분류규정 개정 예고 2025-08 (김·장)](https://www.kimchang.com/ko/insights/detail.kc?sch_section=4&idx=32735)
- [존폐 기로 게임위·P2E 규제 약화 우려 (다음뉴스, 2025-10)](https://v.daum.net/v/20251013104025358)
- [위믹스 국내 상폐 P2E 역설 (뉴데일리, 2025-06)](https://biz.newdaily.co.kr/site/data/html/2025/06/02/2025060200124.html)
- [위메이드 이미르 글로벌 170개국 (시사저널e, 2025)](https://www.sisajournal-e.com/news/articleView.html?idxno=416048)
- [Devsisters Cookie Run 25% 성장 2025 (Seoul Economic)](https://en.sedaily.com/news/2026/02/09/devsisters-posts-25-percent-revenue-growth-on-cookie-run)
- [말랑이 온라인(2021, Whoyaho) — 동명이게임 확인용 (나무위키)](https://namu.wiki/w/%EB%A7%90%EB%9E%91%EC%9D%B4%20%EC%98%A8%EB%9D%BC%EC%9D%B8)
