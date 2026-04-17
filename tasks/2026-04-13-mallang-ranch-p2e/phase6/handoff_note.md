# Handoff Note — mallang-ranch-p2e (외부 신규 프로젝트)

> task_id: `2026-04-13-mallang-ranch-p2e`
> 생성일: 2026-04-13
> 에이전트: handoff-writer (ideation phase 6)
> destination: **외부 신규 프로젝트 `mallang-ranch/` (미생성 상태)**
> 다음 단계: phase7 dispatcher → `../mallang-ranch/INBOX/` 생성 + README 작성 + 본 문서/JSON 배송

---

## 1. 배경

**mallang-ranch-p2e**(코드네임)는 1인 개발자가 Base L2 위에 9개월 Season 0 MVP로 런치하는 코지 P2E 목장 게임이다. 원작 "말랑말랑목장" 감성을 레퍼런스로만 삼고 IP·아트·명칭은 전부 오리지널로 대체하며, Sunflower Land의 커뮤니티 신뢰 철학(no team reserve · no pre-mine · Season 0 노토큰)을 계승한다. NFT 1차 분양(Land 5,000캡 $675K 천장) + 2차 로열티 5%로 자립하고, Season 2 TGE 트리거(DAU 50K + 월 거래량 $200K, 2개월 연속) 충족 시에만 $RANCH ERC-20을 발행해 SEC Howey 리스크를 회피한다. 교배 시스템을 추가해 solo dev의 아트 공수(epic/legendary 직접 모델링) 부담을 줄이는 동시에 하이브리드 발견 리텐션 루프를 확보한다.

---

## 2. 참조 문서 필수 독해 순서

신규 프로젝트 착수 세션이 시작되면 반드시 아래 순서대로 읽는다.

1. **`tasks/2026-04-13-mallang-ranch-p2e/phase4/seed.yaml`** — Ouroboros 시드 원본 + supplemental_overlay (사용자 U1·U2·U3 최종 답변과 N5·N6·N7 파생 분기 반영). ambiguity 0.132로 PASS. 모든 의사결정의 최상위 기준.
2. **`plans/mallang-ranch-p2e-roadmap.md`** — 살아있는 로드맵 §0~§10. 9개월 M0~M9 마일스톤·토크노믹스 3단계·6개 컨트랙트 구조·교배 시스템 세부 스펙·리스크 R1~R9 완화안.
3. **`tasks/2026-04-13-mallang-ranch-p2e/phase3/decisions.md`** — D1~D6 자율 결정의 근거 트레이스 + U1~U3 사용자 확정 + N1~N7 분기. 의사결정 근거를 다시 따라가야 할 때 이 문서가 1차 레퍼런스.
4. **`tasks/2026-04-13-mallang-ranch-p2e/phase1/exploration.md`** — 레퍼런스 P2E 게임(Sunflower Land·Pixels·Axie·Big Time 등) 비교 분석. 설계 선택의 선행 사례 리서치.
5. **`data/mallang-ranch-seed.json`** — 동물 5종·희귀도·랜치 3크기·교배 시스템·hybrid pattern skeleton·컨트랙트 메타 등 구조화 시드 데이터. 코드 초기 시드로 직접 로드 가능한 스켈레톤.
6. (선택) `tasks/2026-04-13-mallang-ranch-p2e/phase2/sketch.md` — 초기 도메인 스케치·사용자 흐름·미결 질문 초안 (phase3에서 대부분 해소됨).
7. (선택) `tasks/2026-04-13-mallang-ranch-p2e/phase5/updated_docs.md` · `new_docs.md` — phase5 integrator가 수행한 살아있는 문서 갱신 기록.

---

## 3. 기준 환경·제약

| 항목 | 값 |
|---|---|
| 팀 규모 | **1인 개발** (solo dev, no outsourcing) |
| 아트 파이프라인 | Godot 클라이언트 + Blender 모델링 셀프 (외주 $0 — U1 확정) |
| MVP 기간 | **9개월** (M0~M9, Season 0 런치까지) |
| 체인 | **Base L2 only** (Immutable zkEVM Passport는 심사 통과 시 2순위 조건부) |
| 배포 플랫폼 | PWA 웹앱 우선 · Google Play v2 · Apple iOS 네이티브 F2P 빌드는 파킹 |
| 지역 | **글로벌 170개국, 한국 IP/KYC 차단** (규제 회피) |
| 언어 | **영어·일본어·베트남어·태국어 4개** / **한국어 미제공** (U2 확정, 이중 시그널) |
| 브랜드명 | **`mallang-ranch-p2e` 코드네임 유지** / 외부 브랜드명은 `{TBD_BRAND_NAME}` (U3 사용자 추후 결정) |
| 토큰 정책 | Season 0 노토큰 · Season 2 TGE 트리거 충족 시에만 $RANCH 발행 · no team reserve · no pre-mine |
| 감사 예산 | **$40K 상한** (30K base + 10K re-audit) |
| 컨트랙트 수 | **6개** (OpenZeppelin 베이스, Marketplace 자체 deploy, 로열티 enforce) |
| 오픈소스 | 코드베이스 전체 공개 (Sunflower Land 철학 정합) |
| 법인 | Season 0: 개인사업자 or 싱가포르/BVI Ltd · **Cayman Foundation은 TGE 6개월 전 설립** |

---

## 4. 이번 작업 (seed.goal)

Base L2 위에서 1인 개발자가 9개월 안에 Season 0 MVP를 런치하는 코지 P2E 목장 게임 **mallang-ranch-p2e** 의 **외부 신규 저장소 부트스트래핑**. 구체 범위: (a) git repo 초기화 + 모노레포 구조(contracts/·client/·server/·assets/·docs/) 설계, (b) Foundry 기반 6개 스마트 컨트랙트 골격(AnimalNFT·LandNFT·ItemNFT·RanchToken·FeedClaim·Marketplace) 스텁 + OpenZeppelin 의존성 고정, (c) Godot 4.x 프로젝트 골격 + 한국어 제외 4언어 i18n 스켈레톤, (d) Blender → Godot 에셋 파이프라인 초안(5종 base mesh + shader 파라미터 절차적 외형 시스템), (e) Cayman Foundation/싱가포르 BVI Ltd 설립 옵션 법률 자문 RFP 초안, (f) Discord·X 커뮤니티 런칭 계획 초안(WL 2,000 슬롯 선정 기준 포함). 런치 전까지 $RANCH 토큰은 코드·문서 어디에도 프로미스/언급 금지.

---

## 5. 수용 기준 체크리스트

seed.acceptance_criteria(메인) + supplemental_overlay.added_acceptance_criteria 전부를 1:1 매칭한다. 체크리스트 ID는 phase3 결정 ID(D1~D6, U1~U3, N5~N7)와 정합.

### D 시리즈 — 메인 시드 수용 기준

- [ ] **D1** — Season 0 런치 시점에 동물 5종(cow/chicken/sheep/rabbit/alpaca), 희귀도 3종(common/rare/epic), 랜치 3크기(trial/small/large) 체계가 가동한다. *(주의: overlay에 의해 런치 시점 희귀도는 2단(common/rare)로 축소되고 epic은 교배 emergent로만 등장. 아래 U1 항목 참조.)*
- [ ] **D2** — Land NFT 1차 분양이 총 공급 5,000(Small 4,000 + Large 1,000)으로 WL 48h 우선 → 퍼블릭 고정가 2단계 구조로 체결된다.
- [ ] **D3** — Small Land 퍼블릭 0.03 ETH / WL 0.02 ETH, Large Land 퍼블릭 0.15 ETH / WL 0.10 ETH로 가격 설정된다.
- [ ] **D4** — 1차 분양 수익이 팀 70% / 커뮤니티 트레저리 30% 분배 계약으로 자동 스플릿된다.
- [ ] **D5** — 2차 로열티 5%가 팀 50% / 트레저리 50% 분배된다.
- [ ] **D6** — 트레저리가 Gnosis Safe 2-of-3 멀티시그(Season 0)로 거버넌스되며, $RANCH TGE 이후 Snapshot 온체인 투표 + 7일 Timelock으로 승격된다.
- [ ] **D7** — $RANCH TGE는 DAU 50,000 달성 이전에는 트리거되지 않는다. *(overlay에 의해 거래량 조건 추가 — 아래 N 시리즈 참조.)*
- [ ] **D8** — 1차 분양 예상 매출 천장 $675K로 $280K 비용 + $400K 트레저리 시드 커버 구조를 시뮬레이션으로 검증한다.
- [ ] **D9** — 스마트 컨트랙트 구조가 AnimalNFT·LandNFT·ItemNFT·RanchToken·FeedClaim·Marketplace로 구현된다 *(overlay에 의해 5→6, Marketplace 자체 deploy로 변경)*.
- [ ] **D10** — Triple token 구조(Animal/Land/Rare Item NFT + $RANCH ERC-20 + $FEED spend-only)가 가동한다.
- [ ] **D11** — 10개 도메인 엔티티(animal · land · rare_item · ranch_token · feed_token · primary_sale · revenue_split · treasury · season · governance)가 모델링된다. *(overlay에 의해 +3개: breeding_event · hybrid_pattern · trait_dna.)*
- [ ] **D12** — 개발 phase가 3mo contracts+audit → 3mo game loop+art integration → 3mo closed alpha+bugfix+sale prep로 진행된다.

### U 시리즈 — 사용자 라운드 확정

- [ ] **U1** — 희귀도가 2단(common/rare) 베이스로 축소되고 epic+ 등급은 **교배 결과 emergent로만** 등장한다 (base catalog에 epic 슬롯 미존재).
- [ ] **U1-breeding** — 교배 MVP 런치: 두 부모 동물 → 가중 평균 + mutation + sometimes_rare_emergence 트레잇 자손 + off-chain DNA + on-chain NFT 결과 민팅.
- [ ] **U1-art** — 아트 파이프라인이 Godot 클라이언트 + Blender 모델링 셀프 구조로 가동 (외주 $0).
- [ ] **U2** — 런치 언어는 영어·일본어·베트남어·태국어 4개이며 **한국어는 제공하지 않는다**.
- [ ] **U3** — 모든 산출물에 코드네임 `mallang-ranch-p2e` 사용, 외부 브랜드 자리는 `{TBD_BRAND_NAME}` 플레이스홀더로 유지 (사용자 직접 결정 대기).

### N 시리즈 — 파생 분기 확정

- [ ] **N5** — 교배 시스템에서 trait DNA는 오프체인(서버) 저장, 결과 NFT만 온체인 민팅(metadata hash 포함). Option B(Axie-style) 채택.
- [ ] **N6** — 교배 cooldown: 초기 72시간 + 교배마다 24시간 증가, 동물당 평생 최대 7회 교배. 비용은 100 $FEED base + 부모 희귀도 합산 가중 modifier.
- [ ] **N7** — 사전 정의 hybrid 패턴 30~50종 카탈로그 구축 + 첫 발견자에게 NFT 트로피(초기 non-tradeable) + $FEED 보너스 보상.

### 추가 수용 기준 (overlay)

- [ ] **AC-TGE** — $RANCH TGE 트리거가 **DAU ≥ 50,000 AND 월 2차 거래량 ≥ $200K, 2개월 연속** 충족 조건으로 구현된다.
- [ ] **AC-Audit** — 감사 예산 $40K 상한(30K base + 10K re-audit)을 초과하지 않는다.
- [ ] **AC-Legal** — Season 0는 개인사업자/싱가포르 BVI Ltd로 시작하고, Cayman Foundation 설립은 TGE 6개월 전 완료한다.
- [ ] **AC-Korea** — 한국 IP geo-block + 한국 거주지 KYC 페이아웃 차단이 양방향으로 가동한다.
- [ ] **AC-Bot** — Gitcoin Passport 기반 Proof of Humanity 통합으로 봇·매크로 방지 레이어가 가동한다.

---

## 6. 주의사항

1. **코드네임 엄수** — 모든 산출물(코드·문서·커밋·이슈)에서 `mallang-ranch-p2e`만 사용. 외부 브랜드명은 사용자가 직접 결정할 때까지 `{TBD_BRAND_NAME}` 플레이스홀더로 유지. 에이전트가 임의로 "Moochi Ranch" 등 후보 이름을 최종 산출물에 쓰지 말 것.
2. **한국 서비스 절대 금지** — P2E 등급분류 거부 전제. 한국 IP/KYC 차단 + 한국어 UI 미제공의 이중 시그널을 법무 자문으로 검증. 마케팅 채널에서도 한국어 콘텐츠·한국 커뮤니티 공식 운영 금지.
3. **원작 IP 직접 차용 금지** — "말랑말랑목장" 명칭·캐릭터 디자인·아트에셋을 그대로 쓰지 말 것. 감성 레퍼런스만 허용(파스텔·코지·kawaii 방향성 참고). 모든 에셋은 Blender 오리지널 제작.
4. **$RANCH 토큰 프로미스 금지 (SEC Howey)** — Season 0 전체 기간 $RANCH 토큰의 가격·배포·에어드랍·수익 기대를 일체 언급·프로미스·암시하지 말 것. 마케팅 카피·백서·Discord 공지·X 게시물 전부 해당. 법무 검토 필수. `RanchToken.sol`은 drafting만, Season 2 TGE 트리거 충족 전까지 배포 금지.
5. **아트 공수 선형 — 스코프 고정** — 5종 base × 1 mesh + 코스튬 5~10종으로 절대 상한. 교배 자손 외형은 Blender shader 파라미터 조합으로 런타임 생성 (별도 모델링 추가 금지). 동물 >5 · 희귀도 베이스 >2 · 랜치 >3 추가 시 scope review gate 트리거.
6. **감사 예산 $40K 상한** — Spearbit / Code4rena / Sherlock 경량 감사 범위 안에서 6개 컨트랙트 소화. 초과 시 컨트랙트 축소 또는 기능 지연 결정.
7. **다음 단계 시작점 — 외부 신규 저장소 부트스트래핑**:
   - (a) Cayman Foundation vs 싱가포르 BVI Ltd 법률 자문 RFP 초안 검토
   - (b) `mallang-ranch/` git repo 초기화 + 모노레포 디렉토리 구조 확정
   - (c) Godot 4.x 프로젝트 골격 + 4언어 i18n 스켈레톤
   - (d) Foundry 기반 6개 스마트 컨트랙트 스텁 + OpenZeppelin 의존성 고정
   - (e) Blender → Godot 에셋 파이프라인 초안 (shader 파라미터 스키마 선정의)
   - (f) Discord·X 커뮤니티 런칭 계획 초안 (WL 2,000 모집 기준)

---

## 7. 다음 phase 실행 에이전트 프롬프트 예시

아래 블록을 그대로 신규 세션(또는 dispatcher가 외부 프로젝트 착수 에이전트에 전달)에 붙여 쓰면 된다.

```text
당신은 mallang-ranch-p2e 외부 신규 저장소 부트스트래핑 에이전트다. 본 작업은 Aura-board(padlet) 피처 파이프라인이 아닌, 완전 신규 git 저장소(`mallang-ranch/`)를 0부터 초기화하는 단계다.

## 배경
1인 개발자가 Base L2 위에 9개월 Season 0 MVP로 런치하는 코지 P2E 목장 게임. 원작 "말랑말랑목장" 감성 레퍼런스만 차용하고 IP·아트·브랜드명은 오리지널. Sunflower Land 철학 계승(no team reserve · Season 0 노토큰 · 오픈소스).

## 필수 독해 (이 순서)
1. tasks/2026-04-13-mallang-ranch-p2e/phase4/seed.yaml (메인 + supplemental_overlay)
2. plans/mallang-ranch-p2e-roadmap.md (§0~§10)
3. tasks/2026-04-13-mallang-ranch-p2e/phase3/decisions.md (D/U/N 결정)
4. tasks/2026-04-13-mallang-ranch-p2e/phase1/exploration.md (레퍼런스 비교)
5. data/mallang-ranch-seed.json (동물/랜치/교배 스켈레톤)
6. tasks/2026-04-13-mallang-ranch-p2e/phase6/handoff_note.md (본 문서)

## 제약
- 1인 개발, 9개월 Season 0, Base L2 only, PWA 우선
- 코드네임 mallang-ranch-p2e 엄수 · 외부 브랜드는 {TBD_BRAND_NAME}
- 한국 IP/KYC 차단 · 한국어 UI 미제공 · 4개 언어(EN/JA/VI/TH)
- $RANCH 토큰 프로미스·언급 금지 (SEC Howey) — Season 2 TGE 트리거 충족 전까지
- 원작 IP 직접 차용 금지 (명칭·아트 전부 오리지널)
- 감사 예산 $40K 상한 · 컨트랙트 6개 · 외주 $0

## 수용 기준 (본 handoff_note.md §5 전체)
D1~D12 + U1~U3 + N5~N7 + AC-TGE·Audit·Legal·Korea·Bot 전부 1:1 커버.

## 첫 세션 산출물
1. 모노레포 디렉토리 구조 제안 (contracts/ · client/ · server/ · assets/ · docs/ · legal/)
2. Foundry init + OpenZeppelin 의존성 고정 + 6개 컨트랙트 스텁 (AnimalNFT·LandNFT·ItemNFT·RanchToken·FeedClaim·Marketplace) — 함수 시그니처만, 로직 구현은 다음 세션
3. Godot 4.x 프로젝트 골격 + 4언어 i18n 스켈레톤
4. README (코드네임·한국 서비스 금지·토큰 언급 금지·라이선스 오픈소스 명시)
5. 법무 자문 RFP 초안 (Cayman Foundation vs 싱가포르 BVI Ltd)

## 금기
- 임의 브랜드 이름 최종 산출물 고정 금지
- $RANCH 토큰 배포·프로미스·언급 금지
- 원작 IP 명칭·아트 직접 차용 금지
- 한국어 UI·한국 커뮤니티 공식 운영 금지
- 스코프 상한(5종/2단/3크기) 초과 금지 (초과 시 scope review gate 필수)
```

---

## 8. 검증 메타

| 항목 | 값 |
|---|---|
| seed.acceptance_criteria 커버리지 | D1~D12 메인 12개 + U1~U3 · N5~N7 overlay 6개 + AC 5개 = **총 23개** 체크리스트, seed.yaml `acceptance_criteria` + `supplemental_overlay.added_acceptance_criteria`와 1:1 매칭 |
| JSON 유효성 | `project_phase0_request.json` strict JSON (trailing comma 없음, UTF-8, 2-space indent) |
| 코드네임 일관성 | 본 문서 + JSON 전체에서 `mallang-ranch-p2e` 사용 · 외부 브랜드 자리 `{TBD_BRAND_NAME}` 플레이스홀더 유지 |
| 참조 문서 독해 순서 | 5개 이상(seed.yaml → roadmap → decisions → exploration → data/seed.json) + 선택 2개 추가 |
| destination 정합성 | `mallang-ranch (외부 신규 — 미생성 상태)` 명시 · phase7 dispatcher가 `../mallang-ranch/INBOX/` 생성 책임 |
